# Deployment Report

## 1. Cloud Provider and Service

The application is deployed on **Microsoft Azure** using an **Azure Virtual Machine (VM)**.

Azure was selected because its free trial provided sufficient resources to run the application without additional infrastructure costs. The Azure VM service also offered the flexibility needed to run the application using Docker, configure Nginx as a reverse proxy, control network access using Azure Network Security Groups, and integrate GitHub Actions with Azure using OIDC and Azure RBAC for secure automated deployments.

## 2. Virtual Machine Setup

An Ubuntu 24.04 LTS virtual machine was created on Microsoft Azure with the following configuration:

- **VM name:** `myVM`
- **Resource group:** `AMALITECH`
- **Region:** South Africa North
- **Operating system:** Ubuntu 24.04 LTS

An SSH key was configured during VM creation to provide secure administrative access.

### 2.1 Network Security

An Azure Network Security Group (NSG) was configured and associated with the VM. The following inbound rules were set up:

| Port | Source | Purpose |
|---|---|---|
| 80 | Any | Public HTTP traffic |
| 22 | Administrator's public IP only | SSH administration |

Port 3000 was not exposed publicly because the application is accessed through Nginx. The SSH rule was restricted using `/32`, limiting access to a single IP address.

### 2.2 Server Access

After creating the VM, an SSH connection was established using the configured private key:

```bash
ssh -i <path-to-private-key> azureuser@<VM-public-ip>
```

## 3. Docker Installation and Application Deployment

### 3.1 Docker Installation

Docker was installed on the Ubuntu VM using Docker's official APT repository. The required packages were installed first:

```bash
sudo apt update
sudo apt install ca-certificates curl
```

Docker's official GPG key was added:

```bash
sudo install -m 0755 -d /etc/apt/keyrings

sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  -o /etc/apt/keyrings/docker.asc

sudo chmod a+r /etc/apt/keyrings/docker.asc
```

The Docker APT repository was configured:

```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

Docker Engine and its components were installed:

```bash
sudo apt update

sudo apt install docker-ce docker-ce-cli containerd.io \
  docker-buildx-plugin docker-compose-plugin
```

The installation was verified and Docker was enabled to start on boot:

```bash
docker --version
sudo systemctl status docker
```

### 3.2 Application Image

The application image was built and published to Docker Hub by the CI/CD pipeline. The image was pulled onto the VM using:

```bash
docker pull <DOCKERHUB_USERNAME>/amalitech-deployready:<IMAGE_TAG>
```

The application container was started with:

```bash
docker run -d \
  --name amalitech-deployready \
  -p 127.0.0.1:3000:3000 \
  --env-file /home/azureuser/amalitech-deployready/.env \
  <DOCKERHUB_USERNAME>/amalitech-deployready:<IMAGE_TAG>
```

The application was bound to `127.0.0.1:3000` rather than `0.0.0.0:3000`, which prevents direct external access to port 3000. Public HTTP traffic enters on port 80 and is forwarded through Nginx to the application.

### 3.3 Nginx Reverse Proxy

Nginx was installed and enabled as a system service:

```bash
sudo apt update
sudo apt install nginx

sudo systemctl enable nginx
sudo systemctl start nginx
```

A reverse proxy configuration was created at `/etc/nginx/sites-available/amalitech-deployready`:

```nginx
server {
    listen 80;
    listen [::]:80;

    server_name _;

    location / {
        proxy_pass http://127.0.0.1:3000;

        proxy_http_version 1.1;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

The configuration was enabled and the default Nginx site was removed to prevent a conflicting default server:

```bash
sudo ln -s \
  /etc/nginx/sites-available/amalitech-deployready \
  /etc/nginx/sites-enabled/

sudo rm /etc/nginx/sites-enabled/default
```

The configuration was validated and Nginx was reloaded:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

The resulting request flow is:

```text
Internet
    |
    | HTTP :80
    v
  Nginx
    |
    | proxy_pass
    v
127.0.0.1:3000
    |
    v
Docker Container
    |
    v
Application
```

## 4. Container Verification

After deploying the container, its status was verified using:

```bash
docker ps
```

The application appeared with a status similar to:

```text
CONTAINER ID   IMAGE                                      STATUS          PORTS
xxxxxxxxxxxx   mbaduko/amalitech-deployready:<TAG>      Up 2 minutes    127.0.0.1:3000->3000/tcp
```

Additional verification was performed using:

```bash
docker inspect amalitech-deployready
curl http://localhost:3000
curl http://localhost
```

A successful response from all three commands confirmed that the application was running and that Nginx was forwarding requests correctly.

## 5. Application Logs

Application logs were viewed using:

```bash
docker logs amalitech-deployready
```

To follow logs in real time:

```bash
docker logs -f amalitech-deployready
```

Recent log entries were displayed using:

```bash
docker logs --tail 100 amalitech-deployready
```

Logs from a specific time window could also be retrieved:

```bash
docker logs --since 1h amalitech-deployready
```

If the application was not responding, the following commands were used to diagnose the issue:

```bash
docker ps -a
docker logs --tail 100 amalitech-deployready
```

These commands helped identify issues such as application startup failures, missing environment variables, or runtime errors.

## 6. Azure RBAC and GitHub OIDC

The deployment pipeline uses GitHub Actions OIDC to authenticate with Azure instead of storing an Azure password or client secret in GitHub. GitHub obtains a short-lived OIDC token and Azure verifies it through a federated identity credential.

The authorization flow:

```text
GitHub Actions
      |
      | OIDC token
      v
Microsoft Entra ID
      |
      | Federated identity
      v
Azure Service Principal
      |
      | RBAC
      v
myVM
      |
      | Run Command
      v
Docker deployment
```

### 6.1 Azure App Registration

An application registration was created in Microsoft Entra ID to represent the GitHub Actions deployment identity. The application's Client ID was recorded for use by GitHub Actions.

The Azure Tenant ID and Subscription ID were retrieved:

```bash
az account show \
  --query "{subscriptionId:id,tenantId:tenantId,name:name}" \
  -o table
```

These values were stored as GitHub Actions secrets:

- `AZURE_CLIENT_ID`
- `AZURE_TENANT_ID`
- `AZURE_SUBSCRIPTION_ID`

### 6.2 GitHub OIDC Federation

A federated identity credential was created for the Azure application registration. The credential was configured to trust tokens issued by `https://token.actions.githubusercontent.com` with the audience `api://AzureADTokenExchange`.

The subject was restricted to the repository's main branch:

```text
repo:Mbaduko/AmaliTech-DEG-Project-based-challenges:ref:refs/heads/main
```

This ensures the Azure identity can only be used by the specified repository and branch.

### 6.3 Custom Azure RBAC Role

A custom Azure RBAC role named `GitHub Actions VM Deploy` was created with a single permission:

```text
Microsoft.Compute/virtualMachines/runCommand/action
```

This permission allows GitHub Actions to execute deployment commands on the VM through Azure's VM Run Command functionality.

### 6.4 Role Assignment

The custom role was assigned to the service principal at the VM scope rather than at the subscription or resource-group level:

- **Role:** `GitHub Actions VM Deploy`
- **Scope:** `/subscriptions/<subscription-id>/resourceGroups/AMALITECH/providers/Microsoft.Compute/virtualMachines/myVM`

The assignment was verified with:

```bash
az role assignment list \
  --assignee "$APP_ID" \
  --all \
  --query "[].{Role:roleDefinitionName,Scope:scope}" \
  -o table
```

The result confirmed that the identity has access only to `myVM`. The role definition was also verified:

```bash
az role definition list \
  --name "GitHub Actions VM Deploy" \
  --query "[].{Name:roleName,Actions:permissions[0].actions,NotActions:permissions[0].notActions}" \
  -o json
```

This follows the principle of least privilege by granting no more access than necessary.

### 6.5 GitHub Actions Configuration

The workflow was configured to request an OIDC token:

```yaml
permissions:
  id-token: write
  contents: read
```

Azure authentication was configured using:

```yaml
- name: Azure Login
  uses: azure/login@v2
  with:
    client-id: ${{ secrets.AZURE_CLIENT_ID }}
    tenant-id: ${{ secrets.AZURE_TENANT_ID }}
    subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
```

No Azure client secret is required.

### 6.6 Deployment via Azure Run Command

After authenticating with Azure, GitHub Actions invoked a command on the VM:

```bash
az vm run-command invoke \
  --resource-group AMALITECH \
  --name myVM \
  --command-id RunShellScript \
  --scripts '
    docker pull <DOCKERHUB_USERNAME>/amalitech-deployready:<IMAGE_TAG>

    docker stop amalitech-deployready || true
    docker rm amalitech-deployready || true

    docker run -d \
      --name amalitech-deployready \
      -p 127.0.0.1:3000:3000 \
      --env-file /home/azureuser/amalitech-deployready/.env \
      <DOCKERHUB_USERNAME>/amalitech-deployready:<IMAGE_TAG>
  '
```

### 6.7 Verification

The OIDC and RBAC configuration was tested by invoking a simple command:

```bash
az vm run-command invoke \
  --resource-group AMALITECH \
  --name myVM \
  --command-id RunShellScript \
  --scripts "echo 'GitHub OIDC successfully reached myVM'"
```

The successful response confirmed that GitHub Actions could authenticate to Azure through OIDC and execute a command on the VM. The deployment was then tested by checking the running Docker container:

```bash
docker ps
```

This confirmed that the pipeline could successfully pull and run the new Docker image.

### Security Result

The deployment authentication no longer depends on SSH credentials stored in GitHub:

```text
GitHub OIDC
    |
    v
Microsoft Entra ID
    |
    v
Federated Identity Credential
    |
    v
Azure RBAC
    |
    v
myVM only
    |
    v
Run Command only
```

The GitHub deployment identity has no access to the rest of the Azure subscription and does not require a long-lived Azure client secret.

## 7. CI/CD Pipeline

The CI/CD pipeline was implemented in `.github/workflows/deploy.yml` using GitHub Actions. It automates the full build and deployment process on every push to the `main` branch.

### 7.1 Pipeline Stages

| Step | Action |
|---|---|
| 1 | Checkout repository |
| 2 | Set up Node.js |
| 3 | Install dependencies with `npm ci` |
| 4 | Run tests |
| 5 | Log in to Docker Hub |
| 6 | Build the Docker image (multi-platform: `linux/amd64`, `linux/arm64`) |
| 7 | Push the image to Docker Hub |
| 8 | Authenticate to Azure using GitHub OIDC |
| 9 | Execute deployment commands on `myVM` via Azure VM Run Command |
| 10 | Pull the new Docker image on the VM |
| 11 | Stop and remove the previous container |
| 12 | Start the new container using the updated image |

### 7.2 Image Tagging

Every build produces a Docker image tagged with the Git commit SHA:

```text
<DOCKERHUB_USERNAME>/amalitech-deployready:<COMMIT_SHA>
```

This provides a unique, immutable identifier for every deployed version, mapping directly to a source code commit.

### 7.3 Trigger

The pipeline runs automatically on pushes to `main`. It can also be triggered manually through the GitHub Actions UI using `workflow_dispatch`.

## 8. SSH Security

SSH was used exclusively for administrator access and initial VM configuration. It is **not** used by the GitHub Actions pipeline for deployment.

The NSG restricts port 22 to the administrator's IP using `/32`, limiting SSH access to a single source address. GitHub Actions uses Azure OIDC and Run Command for deployment, eliminating the need to store SSH private keys as GitHub secrets. Previous SSH-based deployment credentials were removed from the repository after migrating to the OIDC approach.

## 9. Deployment and Update Process

Pushing code to the `main` branch triggers the GitHub Actions workflow, which builds a new Docker image, pushes it to Docker Hub, and deploys it to the VM.

The deployment sequence:

```text
Code push to main
-> Tests run
-> Docker image built (tagged with commit SHA)
-> Image pushed to Docker Hub
-> GitHub Actions authenticates to Azure via OIDC
-> Azure Run Command executes on myVM
-> Docker pulls the new image
-> Previous container stopped and removed
-> New container started with updated image
-> Nginx serves the updated application
```

A deployment can also be triggered manually from the GitHub Actions UI without pushing a commit.