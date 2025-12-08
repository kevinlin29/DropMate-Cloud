# DropMate – Cloud-Native Local Delivery Management System

**Course:** ECE XXXX – Cloud / Container Orchestration Project  
**Project Type:** Kubernetes-orchestrated microservice backend with React web frontend

DropMate is a cloud-native delivery management platform that lets small businesses manage parcels, drivers, and customers with **real-time tracking**, **persistent storage**, and **automated scaling**.  
The system is deployed on **DigitalOcean Kubernetes (DOKS)** with a **React web dashboard** for dispatchers and customers.

---

## 1. Team Information

| Name        | Student Number | Email                  | Role                           |
|------------|----------------|------------------------|--------------------------------|
| Qiwen Lin  | 1012495104     | Qw.lin@mail.utoronto.ca | Backend & Architecture Lead     |
| Liz Zhu    | #########      | \<fill-in\>@mail.utoronto.ca | Frontend & UX Lead             |
| Zongyan Yao  | 1005200836      | zongyan.yao@mail.utoronto.ca | Data & Infrastructure Engineer |
| David Cao  | #########      | \<fill-in\>@mail.utoronto.ca | Observability & QA Engineer    |

> **GitHub Repositories**
> - Backend: `https://github.com/kevinlin29/dropmate-backend`
> - Frontend: `https://github.com/kevinlin29/dropmate-frontend`

---

## 2. Motivation

Local businesses (cafés, small shops, local groceries) often rely on **manual coordination** for deliveries:

- Orders tracked in spreadsheets or group chats  
- Drivers coordinated via ad-hoc phone calls  
- Customers get vague ETAs or no tracking at all  
- No historical data for operations analysis  
- Third-party platforms (e.g., marketplaces) are expensive and opaque

At the same time, cloud platforms like **DigitalOcean Kubernetes** and modern tools like **Docker, React, and Socket.io** make it feasible for a small team to build their own logistics backend with:

- **Real-time delivery visibility** comparable to big platforms  
- **Low operational cost** via autoscaling  
- **Full data ownership** in a self-managed Postgres database  
- **Automated deployments, logging, and monitoring**

Our motivation was twofold:

1. **Solve a realistic problem** – building a system that local businesses could actually self-host to manage their deliveries.
2. **Practice end-to-end cloud engineering** – from microservice design, Dockerization, and Kubernetes on DOKS, to observability, CI/CD, and a modern web frontend.

DropMate is the result: a small but complete cloud-native system that could be deployed, scaled, and operated in a real environment.

---

## 3. Objectives

### Functional Objectives

- Provide a **backend API** for:
  - Creating and managing shipments/orders
  - Managing drivers and their availability
  - Tracking real-time driver locations
  - Emitting events when shipments are updated
- Provide a **web frontend** for:
  - Viewing all shipments and their status
  - Tracking packages and live driver locations on a map
  - Interacting with the backend in real time (WebSockets)

### Cloud / Systems Objectives

- Package all components into **Docker containers**.
- Orchestrate the system with **Kubernetes (DOKS)**:
  - Use Deployments, StatefulSets, Services, and Ingress.
  - Demonstrate **horizontal scaling** (HPA) and failover.
- Use **stateful storage**:
  - PostgreSQL for operational data
  - Persistent volumes for durability
- Use **Redis** for:
  - Caching
  - Lightweight messaging and pub/sub
- Implement **observability**:
  - Health checks, logs, and basic metrics
- Implement **CI/CD**:
  - Build and push container images
  - Automate deployment to Kubernetes via scripts / GitHub Actions

### Non-Functional Objectives

- Keep the system **modular** and **extensible** (each microservice independently deployable).
- Maintain **clear documentation** for setup and operations.
- Design something realistic enough that it could be extended into a production system.

---

## 4. Technical Stack

### 4.1 Backend (Cloud & Microservices)

- **Language & Runtime**
  - Node.js (TypeScript / JavaScript)
- **Microservices**
  - `core-api` – REST API + WebSocket gateway (Socket.io)
  - `location-service` – Driver GPS ingestion & geospatial logic
  - `notification-service` – Real-time notification / WebSocket fan-out
  - `scaling-monitor-service` – Observes HPA metrics and sends alerts

- **Infrastructure**
  - **Kubernetes (DigitalOcean Kubernetes)**
    - 2× `s-2vcpu-4gb` nodes (current setup)
    - Namespaced resources (`dropmate` namespace)
  - **Ingress**
    - NGINX Ingress controller
    - Single DigitalOcean LoadBalancer
    - TLS via cert-manager + Let’s Encrypt
  - **Database**
    - PostgreSQL as a **StatefulSet**
    - Persistent volume (10+ GB)
  - **Cache / Messaging**
    - Redis (cache + pub/sub)

- **External Services**
  - Firebase Admin SDK – authentication & token verification
  - SendGrid – email alerts (e.g. scaling / health notifications)
  - Expo Push – (optional) mobile push notification integration

- **Containerization**
  - Dockerfiles per service
  - Local dev via `docker-compose.yml` (Postgres + Redis + services)

- **Kubernetes Manifests**
  - `k8s/digitalocean/`:
    - `03-postgres.yaml` – StatefulSet + PVC
    - `04-redis.yaml` – Redis Deployment/Service
    - `05-core-api.yaml`, `06-location-service.yaml`, `07-notification-service.yaml`
    - `08-ingress.yaml` – domain routing (api/location/notify)
    - `10-scaling-monitor-rbac.yaml`, `11-scaling-monitor-service.yaml`

### 4.2 Frontend (React Web Dashboard)

- **Framework**
  - React 19 + Vite
- **Key Libraries**
  - Socket.io client – for real-time shipment updates
  - React hooks + context – for UI and state management
  - React Router (or simple router) – for navigation between views
  - Google Maps JavaScript API – map rendering and address helpers
- **Features**
  - Responsive layout with CSS Grid
  - Tracking page to:
    - Enter tracking numbers
    - View shipment list and statuses
    - View driver location on a map (poll/real-time)
  - Status timeline per shipment

### 4.3 DevOps & Tooling

- **CI/CD**
  - GitHub Actions:
    - Build & test services
    - Build Docker images
    - Push to DigitalOcean Container Registry
    - Optional deploy step applying K8s manifests
- **CLI Tools**
  - `doctl` – DigitalOcean CLI
  - `kubectl` – Kubernetes management
  - `docker` / `docker-compose`

---

## 5. Features

This section focuses on **backend + web frontend** features relevant to cloud and distributed systems.

### 5.1 System-Level Features

#### Microservice Architecture

- Multiple independently deployable services:
  - `core-api`, `location-service`, `notification-service`, `scaling-monitor-service`
- Clear separation of concerns:
  - `core-api` handles HTTP/REST, core business logic, and bridges to WebSockets.
  - `location-service` focuses on driver coordinates and location updates.
  - `notification-service` handles real-time outbound events to connected clients.
- Shared infrastructure via Postgres and Redis, but each service can be scaled independently.

#### Real-Time Delivery Tracking

- **Drivers** send GPS updates to the backend (via REST/Socket).
- **Location service** stores the latest coordinates in Redis.
- **Notification service** broadcasts updates to web clients over WebSockets.
- **React frontend** subscribes to WebSocket events and updates the UI instantly:
  - Live location marker on the map
  - Live status changes on shipment cards

#### Persistent & Durable Storage

- Postgres database stores:
  - Shipments (IDs, origin/destination, status, timestamps)
  - Driver profiles and assignments
  - Basic user/customer data (depending on scope)
- Implemented as a Kubernetes **StatefulSet** with persistent volumes:
  - Survives pod restarts
  - Verified by restarting database pods and confirming data remains

#### Autoscaling & High Availability

- DigitalOcean Kubernetes cluster with:
  - 2 worker nodes (scalable to 3+)
  - NGINX Ingress with a single external LoadBalancer
- Horizontal Pod Autoscaler (HPA) for stateless services:
  - e.g. `core-api` can scale from 1 to N replicas based on CPU/memory
- Sample helper scripts:
  - `scale-up.sh` – scale services up (e.g. 4 replicas)
  - `scale-down.sh` – scale services down (e.g. 1 replica)

#### Monitoring & Alerts

- Health endpoints for all services (e.g. `/health`):
  - Used by Kubernetes liveness/readiness probes
- `scaling-monitor-service`:
  - Watches HPA metrics / pod states via the Kubernetes API
  - Sends alerts via SendGrid (e.g. if scaling events or anomalies occur)
- Basic monitoring via:
  - `kubectl top pods/nodes`
  - `kubectl logs` and label-based queries

#### Security

- Secrets handled via:
  - Kubernetes Secrets (`01-secrets.yaml`)
  - Environment variables managed by setup script
- Authentication:
  - Firebase Admin SDK for verifying ID tokens (when integrated)
  - JWT secrets stored in K8s Secrets
- TLS:
  - NGINX Ingress with cert-manager + Let’s Encrypt (HTTPS endpoints like `https://api.dropmate.ca`)

---

### 5.2 Backend API Features

Typical endpoints (simplified):

- `GET /health` – service health check
- `GET /api/shipments` – list shipments
- `GET /api/shipments/:id` – get shipment by ID
- `GET /api/shipments/track/:trackingNumber` – track shipment by tracking number
- `POST /api/shipments` – create a new shipment
- `GET /api/shipments/:id/location` – latest driver location
- `GET /api/shipments/:id/events` – status timeline

Real-time events (WebSocket):

- `shipment_updated` – broadcast when status or location changes
- `shipment_assigned` – fired when a driver is assigned

---

### 5.3 React Frontend Features

- **Responsive layout**
  - Works on desktop and smaller viewports
  - Uses CSS Grid and media queries
- **Shipment dashboard**
  - List of shipments with their status (in transit, delivered, etc.)
  - Search / filter by tracking number
- **Tracking view**
  - Input a tracking number
  - See current state and timeline
- **Live map**
  - Displays driver location (polled or real-time)
  - Uses Google Maps JavaScript API
- **WebSocket integration**
  - Connects to notification service via Socket.io
  - Updates UI in real time without page refresh

---

## 6. User Guide

<!-- Intentionally left for course demo-specific instructions (screenshots, step-by-step usage). -->

---

## 7. Development Guide

<!-- Intentionally left for course demo-specific local setup instructions. 
     Backend & frontend already have their own detailed docs in /docs and respective repos. -->

---

## 8. Deployment Information

<!-- Intentionally left blank for final deployment URL & cluster details summary. -->

---

## 9. Individual Contributions

**Qiwen Lin**
- Designed overall **system architecture** (microservices, K8s topology on DOKS).
- Implemented core of `core-api` service (shipments API, WebSocket integration).
- Wrote and refined Kubernetes manifests for core services and ingress.
- Led DigitalOcean deployment (cluster creation, registry setup, DNS, TLS).

**Liz Zhu**
- Designed and implemented the **React web frontend**:
  - Tracking UI, shipment list, and map layout.
  - Socket.io client integration for real-time updates.
- Implemented responsive layout and front-end error states.
- Connected frontend to backend APIs and WebSockets in both local and cloud environments.

**Zongyan Yao**
- Designed PostgreSQL schemas (shipments, drivers, events) and migration flow.
- Implemented database integration in `core-api`:
  - Query patterns, status updates, and timeline events.
- Helped configure persistent volumes and verified Postgres StatefulSet behavior.
- Assisted with Docker Compose setup for local development.

**David Cao**
- Built `notification-service` and `scaling-monitor-service`:
  - WebSocket broadcast logic
  - HPA monitoring and email alert integration via SendGrid.
- Set up and tested log/metrics workflows (using `kubectl` tools and cluster metrics).
- Wrote and ran test scripts (e.g. `test-shipment-flow.js`) to simulate end-to-end flows.
- Assisted with QA, including verifying scaling up/down and pod restart behavior.

> Note: Individual contributions are also reflected in the Git commit history of the backend and frontend repositories.

---

## 10. Lessons Learned and Concluding Remarks

### 10.1 Lessons Learned

#### 1. Cloud Architecture Is Mostly About Boundaries

Splitting the system into multiple services forced us to be explicit about **contracts**:

- API definitions, event names, and DB ownership needed to be clearly specified early.
- Even in a small project, having a separate `location-service` and `notification-service` made it clear where responsibilities started and ended.
- This also made scaling decisions easier (e.g. scale `core-api` separately from `notification-service`).

We learned that good boundaries reduce coupling and make Kubernetes manifests much easier to reason about.

#### 2. Kubernetes Is Powerful but Demands Discipline

Kubernetes gave us:

- Rolling updates, automatic restarts, and service discovery “for free”.
- The ability to scale services with one command or an HPA rule.

But it also introduced:

- Extra YAML complexity (Secrets, ConfigMaps, Ingress, RBAC, etc.).
- New classes of bugs (e.g. misconfigured environment variables, missing probes).

We learned to adopt a **layered approach**:

1. Get everything running locally with `docker-compose`.
2. Move the same images and environment assumptions onto the cluster.
3. Gradually add probes, HPA, and ingress rather than everything at once.

#### 3. Observability and Test Scripts Are Not Optional

Having simple tools like:

- `/health` endpoints,
- `test-shipment-flow.js` to simulate a delivery,
- and a `scale-up.sh` script

made it much easier to debug. Instead of guessing whether something worked, we could:

- Call `curl /health` for each service.
- Run a script that created a shipment and watched it move through statuses.
- Watch pods scale via `kubectl get hpa` during load.

We learned that **small automation scripts** can save hours of manual testing.

#### 4. Frontend and Backend Need to Evolve Together

Because the React frontend depended on:

- The exact shape of APIs (`/api/shipments`, etc.),
- Specific WebSocket event names (`shipment_updated`, `shipment_assigned`),

any backend change had an immediate impact. Maintaining a simple, shared mental model of the API and avoiding frequent breaking changes helped us keep the frontend and backend in sync.

We learned that communication and consistent contracts are as important as the code itself.

---

### 10.2 Concluding Remarks

DropMate started as a course project but evolved into a **miniature real-world cloud system**:

- A Kubernetes cluster running multiple microservices
- A stateful Postgres database with persistent volumes
- Redis for caching and messaging
- A React dashboard that shows real-time delivery updates
- CI/CD scripts and automation for deployment and scaling

We met our main goals:

- Demonstrated **containerization and orchestration** using Docker and Kubernetes.
- Implemented **real-time tracking** with WebSockets and live maps.
- Achieved **durable state** via StatefulSets and Postgres.
- Practiced **DevOps workflows** from local development to cloud deployment.

If we had more time, potential extensions would include:

- A richer analytics service for delivery metrics (average delivery time, driver utilization).
- More advanced routing/dispatch logic (e.g., integrating with external routing APIs).
- Better service observability (Prometheus, Grafana dashboards, distributed tracing).
- A public “self-serve” onboarding flow for businesses to start using DropMate.

Overall, this project gave us hands-on experience with the full lifecycle of a cloud-native system and helped us understand how real services are built, deployed, and operated at scale.

---
