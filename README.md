# DropMate – Cloud-Native Delivery Management System

**Live App:** https://dropmate-frontend-ovkf3.ondigitalocean.app/

DropMate is a cloud-native delivery management system providing **real-time tracking**, **driver coordination**, and **persistent storage**, deployed on **Kubernetes (DOKS)** with a React web frontend.

---

# 1. Team Information

| Name        | Student Number | Email                       | Role                              |
|------------|----------------|-----------------------------|-----------------------------------|
| Qiwen Lin  | 1012495104     | qw.lin@mail.utoronto.ca     | Backend & Architecture Lead       |
| Liz Zhu    | 1011844943     | liz.zhu@mail.utoronto.ca    | Data & Infrastructure Engineer    |
| Zongyan Yao| 1005200836     | zongyan.yao@mail.utoronto.ca| Frontend & UI/UX                  |
| Yuhao Cao  | 1004145329     | davidyh.cao@mail.utoronto.ca| Observability & QA Engineer       |

---

# 2. Motivation, Problem & Objectives

### Problem
Local small businesses often rely on manual tools (spreadsheets, group chats, phone calls) for deliveries, resulting in:
- No real-time order visibility
- Slow driver coordination
- No operational data
- High dependency on expensive marketplaces

### Motivation
We aimed to build a self-hostable, low-cost, and cloud-native logistics system that gives businesses:
- Full ownership over their data
- A modern real-time tracking experience
- Scalable backend infrastructure
- Hands-on demonstration of Kubernetes, microservices, and DevOps pipelines

### Objectives
Demonstrate containerization, orchestration, and observability. Provide a working microservice system with:
- Shipment & driver management  
- Real-time location tracking  
- Persistent PostgreSQL storage  
- Kubernetes orchestration with autoscaling  
- React dashboard for customers and dispatchers

---

# 3. Features

### 3.1 Delivery Features
- Create, assign, and update shipments
- Driver location tracking via REST
- Real-time WebSocket updates
- Customer tracking interface 
- Event timeline per shipment

### 3.2 System Features
- Fully containerized microservices 
- Kubernetes Deployments + Autoscaling (HPA)
- PostgreSQL StatefulSet with persistent volumes
- Redis caching for live driver locations
- Ingress-based HTTPS routing
- Rolling updates / zero downtime deployment

### 3.3 Web Frontend
- Shipment list view
- Shipment tracking page
- Live driver map (Google Maps API)
- Real-time updates via Socket.io

---

# 4. User Guide

### Demo Login Credentials
| Role | Email | Password |
|------|--------|-----------|
| Customer | admin@dropmate.com | test123 |
| Driver   | driverTest@dropmate.com | test123 |

### Demo Video  
[Demo Video Link](https://youtu.be/hpUit9Y83YA?si=r5JnPqBDB8Wenugw)

### Main User Flows
- Customer: Log in → Track package → View live driver location
- Dispatcher: Log in → Create shipment → Assign driver → Monitor status
- Driver: Log in → Receive shipment → Send location updates
  
---

# 5. Architecture & Technical Stack

## System Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           Internet / Users                               │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                    ┌───────────────┴───────────────┐
                    │                               │
                    ▼                               ▼
        ┌───────────────────────┐       ┌──────────────────────┐
        │   Frontend (React)    │       │   Mobile Clients     │
        │  dropmate-frontend    │       │  (iOS/Android)       │
        │  ───────────────────  │       │                      │
        │  DO App Platform      │       └──────────────────────┘
        │  HTTPS enabled        │                   │
        └───────────────────────┘                   │
                    │                               │
                    │ HTTPS API Calls               │ HTTPS API Calls
                    │                               │
                    └───────────────┬───────────────┘
                                    ▼
        ┌───────────────────────────────────────────────────────┐
        │         Nginx Ingress Controller                      │
        │         ─────────────────────────                     │
        │  • Single LoadBalancer                                │
        │  • SSL/TLS Termination                                │
        │  • Domain-based routing                               │
        │  • Automatic HTTPS redirect                           │
        └───────────────────────────────────────────────────────┘
                    │
        ┌───────────┼───────────────────────┐
        │           │                       │
        ▼           ▼                       ▼
┌─────────────┐ ┌──────────────┐ ┌────────────────────┐
│   Ingress   │ │   Ingress    │ │     Ingress        │
│ api.drop... │ │ location.... │ │  notify.drop...    │
└─────────────┘ └──────────────┘ └────────────────────┘
        │           │                       │
        ▼           ▼                       ▼
┌─────────────┐ ┌──────────────┐ ┌────────────────────┐
│  Service    │ │   Service    │ │     Service        │
│  core-api   │ │  location-   │ │  notification-     │
│  (ClusterIP)│ │  service     │ │  service           │
└─────────────┘ └──────────────┘ └────────────────────┘
        │           │                       │
        │ Auto-scale 1-10 pods              │
        ▼           ▼                       ▼
┌─────────────────────────────────────────────────────┐
│              Kubernetes Pod Layer                   │
│  ┌────────┐ ┌────────┐ ┌────────┐  ┌────────┐       │
│  │Core-API│ │Core-API│ │Location│  │ Notify │       │
│  │  Pod 1 │ │  Pod 2 │ │  Pod 1 │  │  Pod 1 │       │
│  └────────┘ └────────┘ └────────┘  └────────┘       │
│       │          │          │            │          │
└───────┼──────────┼──────────┼────────────┼──────────┘
        │          │          │            │
        └──────────┴──────────┴────────────┴───────────┐
                    │                                  │
                    ▼                                  ▼
        ┌───────────────────────┐         ┌──────────────────────┐
        │   PostgreSQL          │         │   Email Service      │
        │   (Managed Service)   │         │   (SendGrid)         │
        │   + Redis Cache       │         │                      │
        └───────────────────────┘         └──────────────────────┘
```


## Technical Stack

### Backend
- **Node.js**  
- Express REST APIs  
- Socket.io for real-time streaming  
- Redis for messaging  
- PostgreSQL for persistence  

### Frontend
- **React 19 + Vite**  
- Socket.io client  
- Google Maps API  

### Cloud Infrastructure
- **DigitalOcean Kubernetes (DOKS)**  
- NGINX Ingress + TLS  
- Horizontal Pod Autoscaler  
- cert-manager for SSL  
- Kubernetes Secrets, ConfigMaps  

### DevOps
- Docker  
- kubectl + doctl  
---

# 6. Development Guide

### Backend Local Setup
Follow the instructions in the [dropmate-backend](https://github.com/kevinlin29/dropmate-backend) repository.

### Frontend Local Setup
Follow the instructions in the [dropmate-frontend](https://github.com/kevinlin29/dropmate-frontend) repository.

### Kubernetes Deployment
Follow the deployment instructions in the [dropmate-backend](https://github.com/kevinlin29/dropmate-backend/) repository.


# 7. Individual Contributions

### Note for Grading
Our team collaborated using a single Linux machine for development.  
Because of this setup, most commits appear under one user account.  
However, all team members contributed equally to the project, and we collectively acknowledge and agree on this representation.

### Qiwen Lin
- Kubernetes & microservice architecture  
- Core API development  
- Ingress + TLS setup  

### Liz Zhu
- PostgreSQL schema design  
- Backend DB queries  
- Persistent storage testing 

### Zongyan Yao
- React UI development  
- Live map integration  
- Frontend–backend connectivity 

### Yuhao Cao
- Notification + scaling-monitor services  
- SendGrid alerting  
- QA and scaling tests  

---

# 8. Conclusion & Lessons Learned

### Key Insights
- Kubernetes is powerful but requires discipline in configuration  
- Clear microservice boundaries simplify scaling and debugging  
- Observability tooling dramatically reduced time spent diagnosing issues  
- Frontend–backend coordination is crucial in real-time systems  

### Final Remarks
DropMate evolved from a class project into a realistic cloud-native platform demonstrating:

- Containerization  
- Microservice architecture  
- Stateful and stateless workloads  
- Real-time communication  
- Horizontal scaling  
- Secure TLS-enabled deployment  

This project provided hands-on experience with designing, deploying, and operating a real distributed system in production-like conditions.

---
