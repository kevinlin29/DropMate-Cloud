
# DropMate — Cloud-Native Delivery Management

**Live App:** https://dropmate-frontend-ovkf3.ondigitalocean.app/

DropMate is a cloud-native delivery management system built as a set of small, focused microservices running on Kubernetes. It provides real-time driver tracking, WebSocket-based notifications, and a React dashboard for customers and drivers. The project demonstrates containerization, orchestration, observability, and production-ready deployment patterns suitable for small-to-medium delivery operations.

Quick highlights:
- 4 microservices (Core API, Location, Notification, Scaling Monitor)
- React frontend with Google Maps and Socket.IO for real-time updates
- PostgreSQL 16 with monthly partitioning for time-series events
- Redis for pub/sub and caching
- Kubernetes (DigitalOcean Kubernetes) with HPA, NGINX Ingress and cert-manager

---

## Contents
- Overview
- Features
- Architecture
- Tech stack
- Development (local) and Kubernetes deployment
- Demo credentials & video
- Team & contributions

---

## Overview of DropMate

DropMate separates responsibilities into specialized services so each can scale and be operated independently:

- Core API (business logic, auth, shipment management)
- Location Service (high-frequency GPS ingestion, location history)
- Notification Service (WebSocket distribution via Socket.IO)
- Scaling Monitor (observability of HPA events and email alerts)

The system uses PostgreSQL as the single source of truth and Redis pub/sub to propagate events between services and connected clients.

---

## Key Features

- Real-time driver location (30s device updates, live UI markers)
- Redis pub/sub broadcasting to clients via Notification Service
- WebSocket-based notifications and room-based subscriptions
- Partitioned time-series storage for driver location and shipment events
- Google Maps & Directions integration for route visualization and ETA
- Role-based access control with Firebase Authentication (customer/driver/admin)

---

## Driver features — Claiming orders & updating deliveries

This section describes the expected behaviour, API interactions, WebSocket events and data changes when drivers claim available shipments and update delivery progress.

1) Claiming an order (race-safe flow)

- UI: Drivers see an **Available Packages** grid with a `Claim` button for each unassigned shipment.
- Frontend action: clicking `Claim` sends a request to the Core API to claim/assign the shipment to the current driver.
- Suggested endpoint: `POST /api/shipments/:id/assign-driver` with body `{ "driverId": "<driverId>" }` or `POST /api/shipments/:id/claim` (server infers driver from auth token).
- Server-side logic:
	- Verify the caller is a driver and is currently `available`.
	- Use a database transaction and optimistic locking (e.g. `SELECT ... FOR UPDATE` or WHERE `driver_id IS NULL AND status = 'pending'`) to atomically set `driver_id`, update `status` → `assigned`, and insert a `shipment_events` record.
	- If another driver already claimed the shipment, return `409 Conflict` with the current assignment.
	- Publish events to Redis channels: `shipment:{shipmentId}:events` and `driver:{driverId}:events` for interested subscribers.
	- Emit WebSocket events via Notification Service (e.g. `shipment_status_updated` and `driver_assigned`) so UIs update in real time.

Example (PowerShell - using token):

```powershell
curl -X POST "http://localhost:8080/api/shipments/123/assign-driver" -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" -d '{"driverId":"drv_42"}'
```

2) Starting a delivery / update deliveries

- When a driver starts a delivery (e.g. taps **Start**), the frontend calls the Core API to transition the shipment to `in_transit`.
- Suggested endpoint: `PATCH /api/shipments/:id/status` with body `{ "status": "in_transit" }`.
- Server-side actions:
	- Validate the caller is the assigned driver for the shipment.
	- Update `shipments.status = 'in_transit'`, set `drivers.status = 'on_delivery'` and insert a `shipment_events` record with type `picked_up` or `in_transit`.
	- Optionally record the pickup coordinates and timestamp in `shipment_events`.
	- Trigger automatic GPS tracking (frontend begins periodic location updates to Location Service) while in `in_transit`.
	- Publish `shipment_status_updated` and `driver_status_updated` events via Redis → Notification Service.

3) Updating delivery progress and granular package status

- Drivers can update `package_status` for fine-grained state such as `out_for_delivery`, `attempt_failed`, `delivered`, or `exception`.
- Suggested endpoint: `PATCH /api/shipments/:id/package-status` with `{ "package_status": "delivered", "metadata": { ... } }`.
- Server-side:
	- Validate actor and shipment assignment.
	- Insert `shipment_events` rows for audit and timeline rendering (include metadata like signature, photos, notes, GPS coords).
	- When `delivered`, set `shipments.status = 'delivered'` and `drivers.status = 'available'`.
	- Send push notifications to the customer using Expo token(s) stored in `push_tokens` and emit WebSocket events for live updates.

4) Notifications & audit

- Every status change should create an append-only `shipment_events` record for audit and timeline display.
- Events published to Redis should include a minimal payload: `{ "shipmentId": "123", "event": "in_transit", "driverId": "drv_42", "occurred_at": "...", "meta": {...} }`.
- Notification Service pattern-subscribes to `shipment:*` and `driver:*` channels and broadcasts to connected clients in the matching rooms.

5) Concurrency & failure handling

- Claim collisions: return `409 Conflict` and include the current assignment so the client can refresh.
- Partial failures: operations that update multiple resources (shipment + driver + events) must run in a DB transaction. If publishing to Redis fails, proceed but log and retry the publish asynchronously (do not leave DB in an inconsistent state).
- Idempotency: clients should treat repeated calls (retries) as idempotent where possible—use unique event IDs or detect duplicate transitions in the API.

6) Sample WebSocket events emitted

- `shipment_status_updated` — { shipmentId, status, package_status, occurred_at }
- `driver_location_updated` — { driverId, lat, lng, accuracy, occurred_at }
- `shipment_event_created` — { shipmentId, eventType, metadata }

7) Security & permissions

- Only authenticated drivers can call claim/update endpoints for driver actions; Core API validates Firebase token and enforces role checks.
- Admins can override assignments via admin endpoints (not exposed to drivers).

These additions provide a clear contract for frontend flows, backend behaviour, WebSocket notifications, and database changes for claim/update delivery features. If you want, I can also:

- Generate example request/response JSONs for all endpoints mentioned.
- Add a small `ENV_TEMPLATE.md` describing backend env variables (including secrets used by compose).


## Architecture (summary)

Clients (React frontend / mobile) ↔ NGINX Ingress (TLS) ↔ Microservices (Core API, Location, Notification, Scaling Monitor)

- PostgreSQL StatefulSet for persistent data
- Redis for pub/sub and fast caches
- HPA for Core, Location and Notification services

Example event flows:

- GPS update: Driver → Location Service → PostgreSQL + Redis → Notification Service → Frontend
- Delivery update: Driver → Core API → PostgreSQL → WebSocket → Frontend

---

## Technical Stack

- Frontend: React (Vite), Google Maps, Socket.IO client, Firebase
- Backend: Node.js, Express, Socket.IO, Firebase Admin SDK
- Data: PostgreSQL 16 (partitioning), Redis 7
- Infrastructure: Docker, Kubernetes (DOKS), NGINX Ingress, cert-manager
- DevOps: kubectl, doctl, DigitalOcean Container Registry

---

## Development Guide

> **Note for Local Development:** This repository is currently configured for **production deployment on DigitalOcean Kubernetes (DOKS)**. While Docker Compose files are included for local development, the environment configurations (`.env` files) point to production cluster endpoints rather than `localhost`. Local deployment may require modifying these configuration files to use local service URLs (`http://localhost:8080/api` instead of production IPs).

Prerequisites: Node.js 18+, Docker, PostgreSQL 16, Redis 7, kubectl, doctl


Backend local (overview):
1. See `dropmate-backend` for service source and scripts.
2. Run SQL migrations in `dropmate-backend/schema/` (01-08).
3. Create `.env` with `DATABASE_URL`, `REDIS_URL`, Firebase credentials, etc.
4. Start each service with `npm run dev` in its folder.

Docker Compose (backend)

A `docker-compose.yml` is provided at `dropmate-backend/docker-compose.yml` to run the backend stack locally for development or testing. It composes the database and infrastructure required by the services and also builds the service images from local Dockerfiles so mounted source code can be used for quick iteration.

Included services (summary):

- `db` (Postgres 16)
	- Ports: `5432:5432`
	- Persistent volume: `db_data`
	- SQL files in `./schema` are mounted to `/docker-entrypoint-initdb.d/schema` and will be executed on first init
	- Healthcheck uses `pg_isready`
- `redis` (Redis 7)
	- Ports: `6379:6379`
	- Healthcheck uses `redis-cli ping`
- `core-api` (built from `services/core-api`)
	- Ports: `8080:8080`
	- Uses `env_file` at `services/core-api/.env` plus additional environment entries defined in compose
	- Mounts source folder into the container for live code iteration (`./services/core-api/src:/app/src`)
	- Depends on `db` (with healthcheck)
- `location-service` (built from `services/location-service`)
	- Ports: `8081:8081`
	- Depends on `db` and `redis`
	- Mounts source folder for development
- `notification-service` (built from `services/notification-service`)
	- Ports: `8082:8082`
	- Depends on `redis`
	- Mounts source folder for development
- `pgadmin` (optional, dpage/pgadmin4)
	- Ports: `5050:80` (UI at http://localhost:5050)

Key notes about environment and secrets

- The compose file sets example environment variables for local testing (e.g. `POSTGRES_PASSWORD: admin123`, `JWT_SECRET: supersecret`). Replace those with secure values for any real deployment.
- Sensitive values (Firebase private key, Expo token) are referenced using host environment interpolation (e.g. `${FIREBASE_PRIVATE_KEY}`) — set these in your shell or in an env file that is not checked into git.
- `core-api` also reads `services/core-api/.env` via `env_file`. Check that file for service-specific variables.

Quick start (PowerShell):

```powershell
cd dropmate-backend
docker compose up --build
```

Run in detached mode:

```powershell
docker compose up --build -d
```

Stop and remove containers (preserves volumes):

```powershell
docker compose down
```

Stop and remove containers and volumes (data removed):

```powershell
docker compose down -v
```

Useful commands

- View logs for all services (follow):

```powershell
docker compose logs -f
```

- View logs for a single service (e.g. core-api):

```powershell
docker compose logs -f core-api
```

- Run a shell inside a running service container:

```powershell
docker compose exec core-api sh
# or for bash if available
docker compose exec core-api bash
```

- Rebuild a single service after code changes:

```powershell
docker compose up --build -d core-api
```

Database initialization and migrations

- The compose mounts `./schema` into the Postgres init directory so SQL in that folder runs on first database creation. If you need to re-run migrations against an existing database, connect with `psql` or use your migration tool of choice.
- Example: connect to Postgres from the host using psql (if installed):

```powershell
psql "postgresql://postgres:admin123@localhost:5432/dropmate"
```

Security & best practices

- Never commit secrets — add `.env` and service `.env` files to `.gitignore`.
- For multi-developer teams prefer using an `.env.local` or a secrets manager and document required variables (I can generate an `ENV_TEMPLATE.md` if you want).
- The compose setup is intended for local development. Production deployments use the Kubernetes manifests in `dropmate-backend/k8s/digitalocean/`.

Troubleshooting

- If a container fails to start, check `docker compose logs <service>` and the healthcheck status with `docker ps`.
- If ports conflict with services on your machine, change the host-side port mapping in `docker-compose.yml` or stop the conflicting service.
- If Postgres-mounted schema files do not appear to run, ensure the DB volume is fresh (remove with `docker compose down -v`) so init scripts run on a fresh data directory.

Frontend local (overview):
1. See `dropmate-frontend` for the React app.
2. Install: `npm install` and set `.env.local` with API URLs, Google Maps key and Firebase config.
3. Start dev server: `npm run dev` (default port `5173`).

---

## Kubernetes Deployment (summary)

Full manifests and scripts are under `dropmate-backend/k8s/digitalocean/`.

Typical steps:

1. Create DOKS cluster and save kubeconfig: `doctl kubernetes cluster kubeconfig save <cluster-id>`
2. Create namespace and apply configmaps/secrets from `k8s/digitalocean/`.
3. Apply manifests in order: Postgres → Redis → Core API → Location → Notification → Ingress → Scaling Monitor.

Example apply sequence:

```powershell
kubectl apply -f k8s/digitalocean/02-configmaps.yaml
kubectl apply -f k8s/digitalocean/03-postgres.yaml
kubectl apply -f k8s/digitalocean/04-redis.yaml
kubectl apply -f k8s/digitalocean/05-core-api.yaml
kubectl apply -f k8s/digitalocean/06-location-service.yaml
kubectl apply -f k8s/digitalocean/07-notification-service.yaml
kubectl apply -f k8s/digitalocean/08-ingress.yaml
kubectl apply -f k8s/digitalocean/10-scaling-monitor-rbac.yaml
kubectl apply -f k8s/digitalocean/11-scaling-monitor-service.yaml
```

Verify with: `kubectl get pods,svc,hpa -n dropmate`

---

## Demo credentials

Role | Email | Password
---|---:|---
Customer | admin@dropmate.com | test123
Driver | driverTest@dropmate.com | test123

---

## Video Demo

[Watch Demo Video](https://youtu.be/hpUit9Y83YA?si=r5JnPqBDB8Wenugw)

---

## Team & Contributions

| Name | Student Number | Email | Role |
|---|---:|---|---|
| Qiwen Lin | 1012495104 | qw.lin@mail.utoronto.ca | Backend & Architecture Lead |
| Liz Zhu | 1011844943 | liz.zhu@mail.utoronto.ca | Data & Infrastructure Engineer |
| Zongyan Yao | 1005200836 | zongyan.yao@mail.utoronto.ca | Frontend & UI/UX |
| Yuhao Cao | 1004145329 | davidyh.cao@mail.utoronto.ca | Observability & QA Engineer |

### Individual contributions

- Qiwen Lin: Kubernetes architecture, Core API, NGINX ingress, PostgreSQL partitioning, Firebase Admin
- Liz Zhu: DB schema & migrations, indexing & partitioning, PV/StatefulSet testing, query tuning
- Zongyan Yao: React frontend, Google Maps & Directions integration, UI/UX, address autocomplete
- Yuhao Cao: Notification Service (Socket.IO), Scaling Monitor (HPA watcher + SendGrid), QA & load testing

*(Note: commits may appear under one user due to shared development environment; all members contributed equally.)*

---

## Where to look in the repo

- `dropmate-backend/` — backend services, Dockerfiles, k8s manifests and database schema
- `dropmate-frontend/` — React app, components, Firebase integration

