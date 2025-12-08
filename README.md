# DropMate – Cloud-Native Local Delivery Management System

**Course:** ECE1779 – Introduction to Cloud Computing (Fall 2025)  
**Project Type:** Kubernetes-orchestrated microservice backend with React web frontend

DropMate is a cloud-native delivery management platform that lets small businesses manage parcels, drivers, and customers with **real-time tracking**, **persistent storage**, and **automated scaling**.  
The system is deployed on **DigitalOcean Kubernetes (DOKS)** with a **React web dashboard** for dispatchers and customers.

---

## 1. Team Information

| Name        | Student Number | Email                  | Role                           |
|------------|----------------|------------------------|--------------------------------|
| Qiwen Lin  | 1012495104     | Qw.lin@mail.utoronto.ca | Backend & Architecture Lead     |
| Liz Zhu    | 1011844943      | Liz.zhu@mail.utoronto.ca | Frontend & UX Lead             |
| Zongyan Yao  | 1005200836      | zongyan.yao@mail.utoronto.ca | Data & Infrastructure Engineer |
| David Cao  | #########      | \<fill-in\>@mail.utoronto.ca | Observability & QA Engineer    |

---

## 2. Motivation

Many local businesses (cafés, small retailers, groceries) now offer delivery but still coordinate it using:

- Ad-hoc phone calls with drivers  
- Spreadsheets or chat groups for orders  
- No real-time visibility for customers  
- No historical data to improve operations  

Meanwhile, existing third-party platforms (e.g., marketplaces) provide good tracking but:

- Charge high commissions  
- Do not expose operational data  
- Lock businesses into their ecosystem  

Our motivation was to build a system that a small business could **host themselves** to:

- Track deliveries in real time (like a modern marketplace app)  
- Own their **data and infrastructure**  
- Keep cloud costs predictable and under their control  

From a course perspective, we wanted a project that exercises the full cloud stack:

- Containerized microservices with **Docker**  
- Orchestration and scaling with **Kubernetes on DOKS**  
- **Stateful storage** (PostgreSQL) managed as a cluster resource  
- **Real-time communication** (Socket.io, Redis)  

DropMate is the concrete implementation of these goals.

---

## 3. Objectives

### 3.1 Functional Objectives

- Build a backend that supports the core delivery workflow:
  - Create and manage shipments (origin, destination, status).
  - Register drivers and assign deliveries.
  - Track driver locations and expose a “live location” API.
  - Stream shipment updates to connected clients in real time.
- Build a web dashboard that:
  - Shows a list of shipments and their status.
  - Lets users search/track by tracking number.
  - Displays live driver location and status on a map.

### 3.2 Cloud / System Objectives

- **Containerization**
  - Package each backend service into its own Docker image.
  - Provide a `docker-compose` setup for local development.

- **Kubernetes (Chosen over Swarm)**
  - Deploy all services on **DigitalOcean Kubernetes (DOKS)**.
  - Use Deployments for stateless services and a StatefulSet for PostgreSQL.
  - Expose services via a single NGINX Ingress + LoadBalancer.
  - Configure Horizontal Pod Autoscalers (HPA) for core services.

- **Stateful & Cache Layers**
  - Use PostgreSQL with persistent volumes for durable data.
  - Use Redis for:
    - Caching hot data (e.g., latest locations).
    - Lightweight pub/sub for real-time updates.

- **Reliability & Observability**
  - Implement health endpoints and probes for all services.
  - Monitor resource usage and scaling behavior.
  - Add a scaling-monitor service that sends email alerts when thresholds are crossed.

- **Automation**
  - Use GitHub Actions to:
    - Build and test backend services.
    - Build and push Docker images to DigitalOcean Container Registry.
    - Optionally apply Kubernetes manifests as part of a deployment pipeline.

### 3.3 Learning Objectives

- Apply cloud computing concepts (containerization, orchestration, autoscaling) to a realistic use case.
- Gain hands-on experience with:
  - Kubernetes networking (Services, Ingress, DNS, TLS).
  - Managing stateful workloads (Postgres StatefulSet).
  - Integrating a React frontend with a real-time backend.
  - Operating and debugging a live cluster.


---

## 4. Technical Stack

### 4.1 Backend & Microservices

- **Language / Runtime**
  - Node.js (JavaScript)

- **Services**
  - **core-api**
    - REST API for shipments, drivers, and tracking.
    - Socket.io WebSocket server for shipment events.
    - Integrates with PostgreSQL and Redis.
  - **location-service**
    - Receives driver GPS updates (REST).
    - Stores latest locations in Redis.
    - Provides APIs for querying driver position.
  - **notification-service**
    - Subscribes to internal events (Redis pub/sub).
    - Broadcasts shipment/driver updates to WebSocket clients.
  - **scaling-monitor-service**
    - Talks to the Kubernetes API.
    - Monitors HPA status and pod counts.
    - Sends alerts via SendGrid when thresholds or anomalies occur.

- **Infrastructure Components**
  - **PostgreSQL**
    - Deployed as a StatefulSet with PVCs.
    - Stores shipments, drivers, and event history.
  - **Redis**
    - Deployed as a standard Deployment + Service.
    - Used for caching and pub/sub.

- **External Integrations**
  - **Firebase Admin SDK** – verify ID tokens (when auth is enabled).
  - **SendGrid** – send scaling/health alerts.
  - (Optional) **Expo Push** – push notifications if extended to mobile.

### 4.2 Kubernetes & Cloud Platform

- **Cloud Provider**
  - DigitalOcean Kubernetes (DOKS)
  - Cluster: `dropmate-cluster`, region `nyc1`
  - Node pool: 2 × `s-2vcpu-4gb` (scalable to 3+ nodes)

- **Key K8s Resources**
  - Namespace: `dropmate`
  - Deployments: core-api, location-service, notification-service, scaling-monitor-service, Redis.
  - StatefulSet: PostgreSQL with persistent volume (≥ 10 GB).
  - Services:
    - ClusterIP for internal service-to-service traffic.
    - NGINX Ingress for external HTTP routing.
  - Ingress:
    - Single LoadBalancer fronting NGINX Ingress controller.
    - Routes domains like:
      - `api.dropmate.ca` → core-api
      - `location.dropmate.ca` → location-service
      - `notify.dropmate.ca` → notification-service
    - TLS certificates via cert-manager + Let’s Encrypt.

- **Configuration & Secrets**
  - Kubernetes Secrets for:
    - DB credentials, JWT secret.
    - Firebase, SendGrid, Redis URLs.
  - ConfigMaps for non-secret configuration.

### 4.3 Frontend (React Web Dashboard)

- **Framework**
  - React 19 + Vite

- **Libraries**
  - **Socket.io client** – subscribe to shipment and driver updates.
  - **React Router (or equivalent)** – for navigating between views.
  - **Google Maps JavaScript API** – for displaying maps and markers.
  - **Axios or Fetch** – for calling backend APIs.

- **Frontend Responsibilities**
  - Show an overview of all shipments and their statuses.
  - Provide a tracking view for a single shipment.
  - Render a map with live driver location.
  - Reflect real-time changes via WebSocket events.

### 4.4 Tooling

- **Containerization**
  - Dockerfiles in each backend service directory.
  - `docker-compose.yml` for local development (Postgres, Redis, services).
- **CLI Tools**
  - `docker`, `docker-compose`
  - `kubectl`
  - `doctl` (DigitalOcean CLI)

---

## 5. Features

This section summarizes what DropMate implements from both an application and cloud standpoint.

### 5.1 Microservice Architecture

- System decomposed into **four backend services** (core-api, location, notification, scaling-monitor) plus **Postgres** and **Redis**.
- Each service:
  - Has its own Docker image.
  - Has its own Kubernetes Deployment (or StatefulSet for Postgres).
  - Can be scaled independently with `kubectl scale` or HPA rules.

This design demonstrates service separation, independent scaling, and clear ownership of responsibilities.

### 5.2 Real-Time Delivery Tracking

- Drivers periodically send location updates to the **location-service**.
- Location data is stored in Redis and exposed via HTTP APIs.
- The **notification-service** subscribes to internal events and uses Socket.io to:
  - Emit `shipment_updated` and `shipment_assigned` events.
  - Push live updates to connected web clients.
- The React frontend:
  - Opens a WebSocket connection.
  - Updates shipment cards and map markers as events arrive.

This satisfies the requirement for a real-time, cloud-connected application.

### 5.3 Persistent & Durable Storage

- **PostgreSQL** as the main data store:
  - Tables for shipments, drivers, and status events.
  - Deployed via a **StatefulSet** with attached persistent volume.
- Data survives:
  - Pod restarts.
  - Rolling deployments.

In local development, the same schema is used with Docker Compose. In the cluster, storage is backed by DigitalOcean block storage.

### 5.4 Scalability & High Availability

- Stateless services (core-api, location-service, notification-service, scaling-monitor) are deployed as Deployments with:
  - Multiple replicas (e.g., 2–4).
  - Readiness and liveness probes.
- **Horizontal Pod Autoscalers (HPA)**:
  - Monitor CPU/memory and scale replicas up/down.
- Helper scripts:
  - `scale-up.sh` – increase replica counts for load testing.
  - `scale-down.sh` – reduce replicas to save cost.

This demonstrates core cloud concepts: elasticity and fault tolerance.

### 5.5 Monitoring, Health Checks & Alerts

- Each service exposes `/health`:
  - Used by Kubernetes liveness and readiness probes.
  - Also used for quick manual checks (`curl /health`).
- `scaling-monitor-service`:
  - Queries Kubernetes API about HPA and pod states.
  - Sends email alerts (via SendGrid) when:
    - Services reach high replica counts.
    - Or anomalies are detected.
- Operators can:
  - Use `kubectl get pods`, `kubectl logs`, and `kubectl top` to inspect cluster health.
  - Confirm that autoscaling and failover behave as expected.

### 5.6 Security & Networking

- **Network Access**
  - Internal services are exposed as ClusterIP only.
  - Public access is routed through NGINX Ingress + LoadBalancer.
- **TLS**
  - HTTPS endpoints (e.g., `https://api.dropmate.ca`) via cert-manager + Let’s Encrypt.
- **Secrets Management**
  - Kubernetes Secrets for:
    - `DATABASE_URL`, `REDIS_URL`, JWT secret.
    - Firebase and SendGrid credentials.
- **Authentication (optional, when enabled)**
  - Firebase Admin SDK verifies ID tokens passed from clients.
  - Authenticated endpoints can restrict access to shipment data.

### 5.7 Web Frontend Capabilities

- **Shipment List View**
  - Lists all known shipments.
  - Status tags (e.g., pending, in transit, delivered).
  - Ability to filter or search by tracking number.
- **Tracking Detail View**
  - Shows detailed information about a single shipment.
  - Timeline of status changes.
- **Live Map View**
  - Uses Google Maps to show origin, destination, and current driver location.
  - Updates marker position as WebSocket events arrive.
- **Error & Loading States**
  - Displays loading indicators while fetching data.
  - Shows user-friendly messages when the backend is unavailable.

The frontend is intentionally kept lightweight but demonstrates core integration patterns with a cloud backend.


---

## 6. User Guide

<!-- Intentionally left for course demo-specific instructions (screenshots, step-by-step usage). -->
## Deployed URL
https://dropmate-frontend-ovkf3.ondigitalocean.app/

With the following test credentials
| Role     | Email                     | Password |
|----------|---------------------------|----------|
| Customer | admin@dropmate.com        | test123  |
| Driver   | driverTest@dropmate.com   | test123  |

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

## Note for Grading

Our team collaborated using a single Linux machine for development.  
Because of this setup, most commits appear under one user account.  
However, all team members contributed equally to the project, and we collectively acknowledge and agree on this representation.

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
