# Deployment Guide

## 1. Cloud Provider and Service

The application is deployed on **Microsoft Azure** using an **Azure Virtual Machine (VM)**.

### Why Azure?

Azure was selected because it provided the cloud infrastructure and services required for the project while offering a **free trial with sufficient resources to support the deployment without additional infrastructure costs**.

The Azure Virtual Machine provides the flexibility required to run the Dockerized application and configure the server environment. Azure services were also used for:

- **Network Security Groups (NSGs)** to control inbound traffic.
- **Microsoft Entra ID and Azure RBAC** for access control.
- **GitHub Actions OIDC** for secure CI/CD authentication without storing long-lived Azure credentials.

## 2. Virtual Machine Setup

### 2.1 Create the Virtual Machine

An Ubuntu 24.04 LTS virtual machine was created in Microsoft Azure.

The VM was configured with:

- **Name:** `myVM`
- **Resource group:** `AMALITECH`
- **Region:** South Africa North
- **OS:** Ubuntu 24.04 LTS

After creation, the VM was accessed through SSH for initial server configuration.

### 2.2 Configure Network Access

An Azure Network Security Group (NSG) was configured to control inbound traffic.

The following rules were created:

| Protocol | Port | Source | Purpose |
|---|---:|---|---|
| TCP | 80 | Any | Public HTTP traffic |
| TCP | 22 | Administrator's IP only | SSH administration |

Port `3000` was not exposed through the NSG because the application is accessed through Nginx.

The SSH rule was restricted to the administrator's public IP using a `/32` CIDR:

```text
<ADMIN_PUBLIC_IP>/32
```

### 2.3 Prepare the Server

After connecting to the VM through SSH, the system packages were updated:

```bash
sudo apt update
sudo apt upgrade -y
```

## 3. Docker Installation and Application Deployment

### 3.1 Install Docker

Docker was installed on the Ubuntu virtual machine using Docker's official APT repository.

First, the required packages were installed:

```bash
sudo apt update
sudo apt install -y ca-certificates curl
```

Docker's official GPG key was added:

```bash
sudo install -m 0755 -d /etc/apt/keyrings

sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  -o /etc/apt/keyrings/docker.asc

sudo chmod a+r /etc/apt/keyrings/docker.asc
```

The Docker APT repository was then configured:

```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

Docker Engine and the required components were installed:

```bash
sudo apt update

sudo apt install -y docker-ce docker-ce-cli containerd.io \
  docker-buildx-plugin docker-compose-plugin
```

The installation was verified:

```bash
docker --version
```

The Docker service was checked with:

```bash
sudo systemctl status docker
```

Docker was configured to start automatically when the VM boots:

```bash
sudo systemctl enable --now docker
```

### 3.2 Pull the Application Image

The application was packaged as a Docker image and published to Docker Hub through the CI/CD pipeline.

The image can be pulled onto the VM using:

```bash
docker pull <DOCKERHUB_USERNAME>/amalitech-deployready:<IMAGE_TAG>
```

For example:

```bash
docker pull mbaduko/amalitech-deployready:<IMAGE_TAG>
```

The downloaded image can be verified with:

```bash
docker images
```

### 3.3 Configure the Application Environment

Application environment variables were stored on the VM in a `.env` file rather than being included in the Docker image.

The environment file was stored at:

```text
/home/azureuser/amalitech-deployready/.env
```

The file was passed to the container at runtime using Docker's `--env-file` option.

### 3.4 Run the Application Container

The application container was started with:

```bash
docker run -d \
  --name amalitech-deployready \
  -p 127.0.0.1:3000:3000 \
  --env-file /home/azureuser/amalitech-deployready/.env \
  <DOCKERHUB_USERNAME>/amalitech-deployready:<IMAGE_TAG>
```

The application port was bound to `127.0.0.1` so that port 3000 is not directly accessible from the internet. Nginx handles public HTTP traffic on port 80 and forwards requests to the application.

### 3.5 Check if the Container is Running

To check the status of the running containers:

```bash
docker ps
```

The expected output should contain the `amalitech-deployready` container with a status similar to:

```text
STATUS
Up ...
```

For more detailed information about the container:

```bash
docker inspect amalitech-deployready
```

The application can also be tested directly from the VM:

```bash
curl http://localhost:3000
```

### 3.6 View Application Logs

To view the application's current container logs:

```bash
docker logs amalitech-deployready
```

To follow the logs in real time:

```bash
docker logs -f amalitech-deployready
```

The `-f` option keeps the command running and displays new log entries as they are produced.

## 4. Nginx Reverse Proxy Setup

Nginx was installed and configured on the VM to serve as a reverse proxy, forwarding public HTTP traffic on port 80 to the application running on `127.0.0.1:3000`.

### 4.1 Install Nginx

Nginx was installed using the Ubuntu package manager:

```bash
sudo apt install -y nginx
```

The Nginx service was enabled to start on boot:

```bash
sudo systemctl enable --now nginx
```

### 4.2 Configure the Reverse Proxy

A new Nginx server block was created to proxy requests to the application container:

```bash
sudo nano /etc/nginx/sites-available/amalitech-deployready
```

The following configuration was added:

```nginx
server {
    listen 80;
    server_name _;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
    }
}
```

The configuration was enabled by creating a symbolic link:

```bash
sudo ln -s /etc/nginx/sites-available/amalitech-deployready /etc/nginx/sites-enabled/
```

The default Nginx site was removed to prevent conflicts:

```bash
sudo rm /etc/nginx/sites-enabled/default
```

### 4.3 Test and Reload Nginx

The Nginx configuration was tested for syntax errors:

```bash
sudo nginx -t
```

If the test passes, Nginx was reloaded to apply the changes:

```bash
sudo systemctl reload nginx
```

### 4.4 Verify the Reverse Proxy

The application was accessed through the VM's public IP address on port 80 from a web browser:

```text
http://<VM_PUBLIC_IP>
```

Nginx forwarding was also verified directly from the VM:

```bash
curl http://localhost:80
```

## 5. CI/CD Pipeline

The CI/CD pipeline was implemented using **GitHub Actions** and is defined in `.github/workflows/deploy.yml`. It automates testing, building, and deploying the application on every push to the `main` branch.

### 5.1 Pipeline Overview

The pipeline consists of three stages:

| Stage | Description |
|---|---|
| **Test** | Install dependencies and run the test suite. The pipeline stops if any test fails. |
| **Build** | Build a multi-platform Docker image (`linux/amd64`, `linux/arm64`) and push it to Docker Hub, tagged with the Git commit SHA. |
| **Deploy** | Authenticate with Azure using OIDC, then pull the new image on the VM and restart the container. |

### 5.2 Trigger

The pipeline runs automatically when changes are pushed to the `main` branch. It can also be triggered manually through the GitHub Actions UI using `workflow_dispatch`.

### 5.3 Docker Image Tagging

Each build produces a Docker image tagged with the full Git commit SHA:

```text
<DOCKERHUB_USERNAME>/amalitech-deployready:<COMMIT_SHA>
```

This provides a unique, immutable identifier for every deployed version, making it straightforward to roll back to a previous release if needed.

### 5.4 Azure Authentication (OIDC)

The pipeline authenticates with Azure using **OpenID Connect (OIDC)** through the `azure/login@v2` action. This approach eliminates the need to store long-lived Azure credentials as GitHub secrets.

The following secrets are required in the GitHub repository:

| Secret | Description |
|---|---|
| `DOCKERHUB_NAME` | Docker Hub username |
| `DOCKERHUB_TOKEN` | Docker Hub access token |
| `AZURE_CLIENT_ID` | Microsoft Entra ID application client ID |
| `AZURE_TENANT_ID` | Microsoft Entra ID tenant ID |
| `AZURE_SUBSCRIPTION_ID` | Azure subscription ID |

The OIDC configuration requires the following GitHub Actions permissions:

```yaml
permissions:
  id-token: write
  contents: read
```

### 5.5 Deployment to the VM

After authentication, the pipeline uses the Azure CLI (`az vm run-command invoke`) to execute commands directly on the VM. This approach does not require SSH keys to be stored as secrets — Azure handles authentication through the OIDC token.

The deployment script:

1. **Pulls** the new Docker image from Docker Hub
2. **Stops** the existing container (if running)
3. **Removes** the old container
4. **Starts** a new container with the updated image

```bash
az vm run-command invoke \
  --resource-group AMALITECH \
  --name myVM \
  --command-id RunShellScript \
  --scripts '
    docker pull <DOCKERHUB_USERNAME>/amalitech-deployready:<COMMIT_SHA>
    docker stop amalitech-deployready || true
    docker rm amalitech-deployready || true
    docker run -d \
      --name amalitech-deployready \
      -p 127.0.0.1:3000:3000 \
      --env-file /home/azureuser/amalitech-deployready/.env \
      <DOCKERHUB_USERNAME>/amalitech-deployready:<COMMIT_SHA>
  '
```

### 5.6 Manual Deployment

To trigger a deployment manually without pushing a commit:

1. Navigate to the repository on GitHub
2. Go to **Actions** → **Build, Push & Deploy**
3. Click **Run workflow** and select the `main` branch
4. Click **Run workflow**

### 5.7 Rollback

To roll back to a previous version, trigger the pipeline manually and pass the desired commit SHA as the image tag, or SSH into the VM and run:

```bash
docker stop amalitech-deployready
docker rm amalitech-deployready

docker run -d \
  --name amalitech-deployready \
  -p 127.0.0.1:3000:3000 \
  --env-file /home/azureuser/amalitech-deployready/.env \
  <DOCKERHUB_USERNAME>/amalitech-deployready:<DESIRED_COMMIT_SHA>
```
