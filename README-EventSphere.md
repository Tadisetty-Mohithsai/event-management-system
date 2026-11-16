# EventSphere — Cloud-Native Event Management & Ticketing Platform

EventSphere is a full-stack event discovery and ticket-booking application. Users can explore live music events, filter by city and category, create an account, reserve tickets, and view QR-code tickets. Organizers and administrators have additional event-management and ticket check-in capabilities.

The project also includes containerization, Kubernetes deployment manifests, a Helm chart, a GitHub Actions workflow, and Prometheus/Grafana monitoring.

## Table of contents

- [Features](#features)
- [Technology stack](#technology-stack)
- [Architecture](#architecture)
- [Project structure](#project-structure)
- [Prerequisites](#prerequisites)
- [Run with Docker Compose](#run-with-docker-compose-recommended)
- [Demo accounts](#demo-accounts)
- [Run in local development mode](#run-in-local-development-mode)
- [API endpoints](#api-endpoints)
- [Monitoring](#monitoring)
- [Kubernetes deployment](#kubernetes-deployment)
- [Helm deployment](#helm-deployment)
- [CI/CD](#cicd)
- [Troubleshooting](#troubleshooting)
- [Security notes](#security-notes)
- [Future enhancements](#future-enhancements)

## Features

### Event discovery
- Modern, responsive event-discovery interface.
- Browse a curated collection of music events.
- Search by event title, artist, venue, or city.
- Filter events by category and city.
- View event descriptions, venue, date, availability, and ticket price.

### User accounts and ticket booking
- User registration and login using JWT authentication.
- Role-based access for attendees, organizers, and administrators.
- Reserve between 1 and 8 tickets per booking.
- Transactional booking and ticket-inventory updates in PostgreSQL.
- Unique ticket reference for each booking.
- QR-code ticket generation.
- View previously booked tickets.

### Organizer and administrator capabilities
- Create and publish events.
- View an administrative summary of events, bookings, tickets, revenue, and check-ins.
- Validate tickets through the check-in endpoint.
- Restrict organizer/admin operations using role-based authorization.

### Cloud-native and DevOps capabilities
- Dockerfiles for frontend and backend.
- Docker Compose stack for the application, PostgreSQL, Prometheus, and Grafana.
- Kubernetes manifests for namespace, database, backend, frontend, and secrets.
- Helm chart with configurable images, replicas, resources, and secrets.
- GitHub Actions workflow for dependency installation, syntax checks, frontend build, and container builds.
- Prometheus application metrics and a provisioned Grafana dashboard.

> **Payment note:** The current project records ticket bookings and calculated totals. It does not integrate a real payment gateway or process actual payments.

## Technology stack

| Layer | Technology |
|---|---|
| Frontend | React 18 |
| Frontend tooling | Vite 6 |
| Icons | Lucide React |
| Backend | Node.js, Express |
| Authentication | JSON Web Tokens (JWT) |
| Password hashing | bcryptjs |
| Database | PostgreSQL 16 |
| Validation | Zod |
| QR tickets | qrcode |
| API security | Helmet, CORS, express-rate-limit |
| Containers | Docker, Docker Compose |
| Orchestration | Kubernetes |
| Packaging/deployment | Helm |
| CI/CD | GitHub Actions |
| Monitoring | Prometheus, Grafana |
| Metrics library | prom-client |

## Architecture

```text
Browser
   |
   v
React + Vite frontend (served by Nginx)
   |
   | HTTP / JSON API
   v
Node.js + Express backend
   |             |
   |             +---- JWT authentication and role checks
   |
   +---- PostgreSQL database
   |
   +---- /metrics endpoint
              |
              v
          Prometheus
              |
              v
           Grafana
```

In Docker Compose, the frontend is exposed on port `8089`, the API on `4000`, PostgreSQL is available to the other containers on the Compose network, Prometheus on `9090`, and Grafana on `3001`.

## Project structure

```text
EventSphere/
├── .env.example
├── .gitignore
├── docker-compose.yml
├── EXPO-PRESENTATION.md
├── running commands.txt
│
├── backend/
│   ├── Dockerfile
│   ├── package.json
│   ├── .env.example
│   ├── db/
│   │   └── init.sql
│   └── src/
│       └── server.js
│
├── frontend/
│   ├── Dockerfile
│   ├── index.html
│   ├── nginx.conf
│   ├── package.json
│   ├── vite.config.js
│   └── src/
│       ├── main.jsx
│       └── style.css
│
├── k8s/
│   ├── namespace.yaml
│   ├── postgres.yaml
│   ├── backend.yaml
│   ├── frontend.yaml
│   └── secret.example.yaml
│
├── helm/
│   └── eventsphere/
│       ├── Chart.yaml
│       ├── values.yaml
│       └── templates/
│
├── monitoring/
│   ├── prometheus.yml
│   └── grafana/
│       ├── dashboards/
│       └── provisioning/
│
└── .github/
    └── workflows/
        └── ci.yml
```

## Prerequisites

### Recommended: Docker Compose

- Docker Desktop installed and running.
- Docker Compose v2 (`docker compose`).

### For local development without Docker

- Node.js 20 or later.
- npm.
- PostgreSQL 16, either installed locally or started as a Docker container.

### For Kubernetes deployment

- A Kubernetes cluster (for example, Minikube, Docker Desktop Kubernetes, or a cloud cluster).
- `kubectl`.
- Helm 3.
- Access to a container registry if deploying custom images.

## Run with Docker Compose (recommended)

Docker Compose starts the complete application stack.

### 1. Extract the project and open a terminal

Change directory to the folder containing `docker-compose.yml`:

```powershell
cd path\to\EventSphere
```

### 2. Configure environment variables

Create your local environment file:

```powershell
Copy-Item .env.example .env
```

Open `.env` and set strong local values for `POSTGRES_PASSWORD`, `JWT_SECRET`, and `GRAFANA_ADMIN_PASSWORD`.

The example values are for local demonstration only. Do not use them in a public deployment.

### 3. Build and start all services

```powershell
docker compose up --build -d
```

### 4. Check container status

```powershell
docker compose ps
```

### 5. Open the application

| Service | URL |
|---|---|
| EventSphere frontend | http://localhost:8089 |
| Backend health check | http://localhost:4000/health |
| Prometheus | http://localhost:9090 |
| Grafana | http://localhost:3001 |

The frontend port is **8089** in the supplied `docker-compose.yml`.

### 6. View logs

```powershell
docker compose logs --tail=100
```

To follow logs for a particular service:

```powershell
docker compose logs -f api
docker compose logs -f web
docker compose logs -f db
```

### 7. Stop the application

```powershell
docker compose down
```

To also delete the persisted PostgreSQL and Grafana volumes (this permanently removes their stored data):

```powershell
docker compose down -v
```

## Demo accounts

When `SEED_DEMO=true`, the backend creates these demonstration accounts if they do not already exist:

| Role | Email | Password |
|---|---|---|
| Organizer | `organizer@eventsphere.local` | `ExpoDemo123!` |
| Attendee | `attendee@eventsphere.local` | `ExpoDemo123!` |

The organizer account can access organizer/admin-protected operations. The attendee account can browse events and make bookings.

Change or disable demo credentials before deploying the application publicly.

## Run in local development mode

The frontend and backend can be run separately for development. PostgreSQL must be running and accessible to the backend.

### 1. Start PostgreSQL

The simplest option is to start only the database service from the project root:

```powershell
docker compose up -d db
```

### 2. Configure and start the backend

Open a terminal:

```powershell
cd backend
Copy-Item .env.example .env
npm install
npm run dev
```

Check `backend/.env` and make sure `DATABASE_URL` points to the PostgreSQL instance. For the database started by Compose, the host port may need to be published or the backend can be run as a Compose service.

The backend defaults to port `4000`.

### 3. Configure and start the frontend

Open a second terminal:

```powershell
cd frontend
npm install
```

Create `frontend/.env` with:

```env
VITE_API_URL=http://localhost:4000/api
```

Then start Vite:

```powershell
npm run dev
```

Open the local URL printed by Vite (normally `http://localhost:5173`).

For local development, ensure the backend `CORS_ORIGIN` includes the frontend origin, for example `http://localhost:5173`.

### Available npm scripts

Backend (`backend/`):

| Command | Purpose |
|---|---|
| `npm install` | Install backend dependencies |
| `npm run dev` | Start backend with Node watch mode |
| `npm start` | Start backend |
| `npm test` | Run Node.js test runner |

Frontend (`frontend/`):

| Command | Purpose |
|---|---|
| `npm install` | Install frontend dependencies |
| `npm run dev` | Start Vite development server |
| `npm run build` | Build frontend for production |
| `npm run preview` | Preview the production build |

## API endpoints

The backend base URL in Docker Compose is `http://localhost:4000`.

| Method | Endpoint | Authentication | Description |
|---|---|---|---|
| `GET` | `/health` | No | API and database health |
| `GET` | `/metrics` | No | Prometheus metrics |
| `GET` | `/api/events` | No | List/search/filter events |
| `GET` | `/api/events/:id` | No | Get event details |
| `POST` | `/api/auth/register` | No | Register a user |
| `POST` | `/api/auth/login` | No | Log in |
| `GET` | `/api/auth/me` | Bearer token | Get current user |
| `POST` | `/api/bookings` | Bearer token | Book tickets |
| `GET` | `/api/bookings/mine` | Bearer token | List current user's bookings |
| `POST` | `/api/bookings/:id/check-in` | Organizer/admin | Check in a ticket |
| `POST` | `/api/events` | Organizer/admin | Create an event |
| `GET` | `/api/admin/summary` | Organizer/admin | Get admin summary |

### Example registration request

```json
{
  "name": "Demo User",
  "email": "demo@example.com",
  "password": "StrongPassword123"
}
```

### Example login request

```json
{
  "email": "demo@example.com",
  "password": "StrongPassword123"
}
```

### Example booking request

Send a Bearer token in the `Authorization` header:

```json
{
  "eventId": 1,
  "quantity": 2
}
```

The booking response includes booking details and a QR code. The API validates ticket quantity and remaining availability and updates the booking and event inventory in a database transaction.

## Monitoring

EventSphere includes Prometheus and Grafana in the Docker Compose stack.

### Prometheus

Open:

```text
http://localhost:9090
```

Prometheus scrapes the backend metrics endpoint at:

```text
http://api:4000/metrics
```

The backend exposes default Node.js process metrics and application counters, including HTTP request and successful booking counters.

### Grafana

Open:

```text
http://localhost:3001
```

The supplied Compose configuration uses username `admin`. The password is configured by `GRAFANA_ADMIN_PASSWORD` in `.env`; if it is not set, the Compose default is `admin`.

The project provisions a Prometheus data source and an EventSphere dashboard. Change the default password and restrict access before any public deployment.

## Kubernetes deployment

The `k8s/` directory contains Kubernetes resources for the namespace, PostgreSQL, backend, frontend, and a secret template.

### 1. Build container images

From the project root:

```bash
docker build -t YOUR_REGISTRY/eventsphere-api:1.0.0 ./backend
docker build -t YOUR_REGISTRY/eventsphere-web:1.0.0 ./frontend
```

### 2. Push images to your registry

```bash
docker push YOUR_REGISTRY/eventsphere-api:1.0.0
docker push YOUR_REGISTRY/eventsphere-web:1.0.0
```

### 3. Update Kubernetes configuration

Before applying the manifests:

- Update image names in `k8s/backend.yaml` and `k8s/frontend.yaml`.
- Replace all example secret values in `k8s/secret.example.yaml`.
- Review database storage and service configuration.
- Configure registry credentials if the cluster needs to authenticate to your image registry.

Do not commit real credentials to Git.

### 4. Apply the manifests

```bash
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/secret.example.yaml
kubectl apply -f k8s/postgres.yaml
kubectl apply -f k8s/backend.yaml
kubectl apply -f k8s/frontend.yaml
```

The secret manifest is a template and must be updated with secure values before use.

### 5. Check deployment status

```bash
kubectl get pods -n eventsphere
kubectl get services -n eventsphere
kubectl get deployments -n eventsphere
```

### 6. Access the frontend locally

```bash
kubectl port-forward svc/eventsphere-frontend 8080:80 -n eventsphere
```

Then open:

```text
http://localhost:8080
```

For a public deployment, configure an Ingress or load balancer, TLS, persistent storage, backups, and appropriate network policies.

## Helm deployment

A Helm chart is provided at `helm/eventsphere`.

Before installing:

1. Edit `helm/eventsphere/values.yaml`.
2. Replace the placeholder frontend and backend image names.
3. Set strong database and JWT secret values.
4. Review replica counts, resource requests/limits, and storage settings.

Install or upgrade:

```bash
helm upgrade --install eventsphere ./helm/eventsphere --namespace eventsphere --create-namespace
```

Check the release:

```bash
helm list -n eventsphere
kubectl get all -n eventsphere
```

## CI/CD

The GitHub Actions workflow is located at:

```text
.github/workflows/ci.yml
```

It runs on pushes to `main` and `master`, pull requests, and manual workflow dispatch.

The workflow:
- Checks out the repository.
- Sets up Node.js 20.
- Installs backend dependencies.
- Checks backend JavaScript syntax.
- Installs frontend dependencies.
- Builds the frontend.
- Builds backend and frontend Docker images.

The image-publishing job is guarded by a repository variable and runs only for pushes to `main` when `PUBLISH_IMAGES` is set to `true`. Configure the registry, image names, and required repository settings before enabling publishing.

## Troubleshooting

### Frontend does not open
- Confirm Docker Desktop is running.
- Run `docker compose ps`.
- Check frontend logs with `docker compose logs web`.
- Confirm port `8089` is not already in use.

### API health check fails
- Check `docker compose logs api`.
- Confirm PostgreSQL is healthy with `docker compose ps`.
- Check database connection settings in `.env`.
- Test `http://localhost:4000/health`.

### Database connection errors
- Wait for the database health check to pass.
- Verify `POSTGRES_DB`, `POSTGRES_USER`, and `POSTGRES_PASSWORD` match the backend `DATABASE_URL`.
- If you changed database credentials after the volume was initialized, the existing PostgreSQL volume may still contain the old credentials. Do not delete volumes unless you are willing to lose persisted data.

### Login or registration fails
- Check API logs.
- Confirm the frontend is sending requests to the correct API URL.
- For local Vite development, set `VITE_API_URL=http://localhost:4000/api`.
- Confirm backend CORS settings include the frontend origin.

### Prometheus has no data
- Confirm the API container is healthy.
- Open `http://localhost:4000/metrics`.
- Check the Prometheus target status at `http://localhost:9090/targets`.

### Port is already in use
Change the host-side port mapping in `docker-compose.yml` or stop the process already using the required port.

## Security notes

This project is an educational/demo foundation and is not a production-certified ticketing or payment platform.

Before deploying publicly:

- Replace all default/example passwords and secrets.
- Use a long, random `JWT_SECRET`.
- Store secrets in a secret manager or Kubernetes Secrets, not in source control.
- Enable HTTPS/TLS.
- Restrict CORS to trusted frontend origins.
- Restrict access to Grafana, Prometheus, and operational endpoints.
- Configure PostgreSQL backups and persistent storage.
- Review authorization and operational logging.
- Add payment-provider integration only with appropriate server-side verification if real payments are required.

## Future enhancements

Potential improvements include:
- Payment gateway integration with verified payment status.
- Email confirmations and downloadable tickets.
- Event organizer dashboard in the frontend.
- QR scanning interface for event check-in.
- Event recommendations based on user interests.
- Automated backend and frontend tests.
- Kubernetes Ingress, autoscaling, and production-grade secrets management.

## License

No license is specified in the supplied project. Add a `LICENSE` file before distributing EventSphere as an open-source project.
