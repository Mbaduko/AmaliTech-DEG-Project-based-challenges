# DeployReady

A lightweight analytics API service built with **Node.js** and **Express**, deployed on **Microsoft Azure** using **Docker**, **Nginx**, and **GitHub Actions CI/CD**.

## About the Application

DeployReady provides a small REST API with three endpoints:

- `/health` — Reports the application status. Useful for monitoring and uptime checks.
- `/metrics` — Returns server metrics including uptime, memory usage, and the running Node.js version.
- `/data` — Accepts a JSON payload and echoes it back. Designed as a simple data ingestion endpoint.

The application uses **Jest** and **Supertest** for testing, and the entire stack is containerized with Docker for consistent deployment across environments.

## Live Application

**Public URL:** [Here](http://4.168.235.204)

Test the endpoints:

```bash
curl http://<VM_PUBLIC_IP>/health
curl http://<VM_PUBLIC_IP>/metrics
curl -X POST http://<VM_PUBLIC_IP>/data -H "Content-Type: application/json" -d '{"key": "value"}'
```

## Running with Docker Compose

Navigate to the app directory:

```bash
cd dev-ops/DeployReady/app
```

Build and start the container:

```bash
docker compose up --build -d
```

The application will be available at `http://localhost:3000`.

View logs:

```bash
docker compose logs -f
```

Stop and remove the container:

```bash
docker compose down
```

## Running with Docker (without Compose)

Navigate to the app directory:

```bash
cd dev-ops/DeployReady/app
```

Build the image:

```bash
docker build -t amalitech-deployready .
```

Run the container:

```bash
docker run -d \
  --name amalitech-deployready \
  -p 3000:3000 \
  amalitech-deployready
```

The application will be available at `http://localhost:3000`.

## Running without Docker

Navigate to the app directory:

```bash
cd dev-ops/DeployReady/app
```

Install dependencies:

```bash
npm install
```

Start the server:

```bash
npm start
```

The server starts on port 3000 by default.

## Testing

```bash
cd dev-ops/DeployReady/app
npm test
```

## Project Files

| File | Purpose |
|---|---|
| `app/index.js` | Application entry point |
| `app/Dockerfile` | Docker image definition |
| `app/docker-compose.yml` | Compose configuration for local development |
| `app/.env.example` | Environment variable template |
| `app/DEPLOYMENT.md` | Full deployment report |
| `.github/workflows/deploy.yml` | CI/CD pipeline |
