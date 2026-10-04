# Lesson 36 — CodeBuddy Production Flow

## 1. Purpose

This lesson combines the Docker concepts from the course into one realistic production-oriented flow for CodeBuddy:

- React frontend
- Node.js + Express backend
- MongoDB
- Socket.IO
- optional Redis
- Docker
- Docker Compose
- Container Registry
- CI/CD
- Kubernetes

The goal is to understand how source code becomes a running production application.

## 2. CodeBuddy Architecture

A production-oriented architecture can look like:

    Internet
       |
       v
    DNS / Load Balancer / Gateway
       |
       +------------------+
       |                  |
       v                  v
    Frontend           Backend
    React/Nginx        Node/Express
                           |
                    +------+------+
                    |             |
                    v             v
                 MongoDB        Redis
                              (if needed)

The frontend and backend should have independent deployment boundaries.

## 3. Docker Images

Backend:

    Source Code
        |
        v
    Dockerfile
        |
        v
    Backend Image

Frontend:

    Source Code
        |
        v
    Multi-stage Dockerfile
        |
        v
    Static files + Nginx image

Important production principles:

- use small trusted base images
- install only required runtime dependencies
- use multi-stage builds for frontend applications
- use .dockerignore
- do not bake secrets into images
- run containers as non-root where practical
- pin important image and dependency versions

## 4. Local Development with Compose

Compose can run the complete local stack:

    services
      |
      +-- frontend
      +-- backend
      +-- mongodb
      +-- redis (optional)

Example relationship:

    Browser
       |
       | localhost:3000
       v
    Backend container
       |
       | mongodb:27017
       v
    MongoDB container

The backend must use the Compose service name, such as mongodb, rather than localhost for container-to-container communication.

## 5. Persistent Database

Containers are replaceable. Database data must therefore use persistent storage.

Local Compose example:

    volumes:
      mongo-data:

    services:
      mongodb:
        image: mongo:8
        volumes:
          - mongo-data:/data/db

For production, evaluate a managed MongoDB service or a properly designed stateful deployment with backups, replication, monitoring, and recovery procedures.

## 6. Configuration and Secrets

Backend configuration can include:

    NODE_ENV
    PORT
    MONGO_URI
    JWT_SECRET
    RAZORPAY_KEY_ID
    RAZORPAY_KEY_SECRET
    FRONTEND_URL
    REDIS_URL

Configuration should come from the runtime environment.

Never commit real credentials to Git or copy them into Docker images.

For production, use a proper secret-management mechanism such as Kubernetes Secrets with appropriate cluster security or an external secret manager.

## 7. Frontend Environment Variables

Anything delivered to browser JavaScript is public.

For example, a Vite variable such as:

    VITE_API_URL

can be public configuration.

It must never contain:

    JWT_SECRET
    DATABASE_PASSWORD
    RAZORPAY_KEY_SECRET

Rule:

> If the browser can receive it, the user can inspect it.

## 8. Image Registry

After building images, push them to a registry:

    Dockerfile
       |
       v
    docker build
       |
       v
    Versioned image
       |
       v
    Container Registry
       |
       v
    Deployment platform

Example image names:

    codebuddy/backend:1.0.0
    codebuddy/frontend:1.0.0

Prefer meaningful version tags and, when appropriate, immutable image digests instead of relying only on latest.

## 9. CI/CD

A typical pipeline is:

    Git Push
       |
       v
    CI/CD
       |
       +--> Install dependencies
       +--> Lint
       +--> Test
       +--> Build
       +--> Docker Build
       +--> Security Scan
       +--> Push Image
       |
       v
    Deploy
       |
       v
    Health Check

This can be implemented with Jenkins, GitHub Actions, GitLab CI, or another CI/CD system.

Jenkins is an automation/orchestration tool for the pipeline; it does not need to contain the application itself.

## 10. Kubernetes Deployment

After images are pushed:

    Container Registry
          |
          v
    Kubernetes Deployment
          |
          v
    Backend Pods

A Deployment can maintain three backend replicas:

    Deployment
       |
       +--> Pod 1
       +--> Pod 2
       +--> Pod 3

If one Pod fails, Kubernetes can create another to restore the desired state.

## 11. Kubernetes Service

Pods are replaceable and their IP addresses can change.

A Service gives the backend a stable endpoint:

    Frontend
       |
       v
    backend Service
       |
       +--> Pod 1
       +--> Pod 2
       +--> Pod 3

Other workloads should communicate with the Service rather than hard-coding Pod IPs.

## 12. Frontend Deployment

The frontend can also run as replicated Pods:

    Frontend Deployment
       |
       +--> Frontend Pod
       +--> Frontend Pod

A Service or external Gateway/Load Balancer can expose the frontend.

## 13. Socket.IO Scaling

CodeBuddy uses Socket.IO, so horizontal backend scaling needs additional planning.

With multiple backend replicas:

    Browser A ---> Backend Pod 1
    Browser B ---> Backend Pod 2

One instance may not automatically know about connections handled by another instance.

Depending on the architecture, you may need:

- a shared Socket.IO adapter
- Redis for cross-instance event propagation
- appropriate connection/session strategy
- WebSocket support in the load balancer or Gateway

This is an important production concern for CodeBuddy.

## 14. MongoDB in Production

Backend Pods are usually designed to be replaceable. MongoDB is stateful.

A common production architecture is:

    Kubernetes
       |
       v
    Backend Pods
       |
       v
    Managed MongoDB

A self-hosted MongoDB deployment requires proper persistent storage, backups, replication, security, monitoring, and recovery planning.

For learning, MongoDB with Docker Compose is excellent. Production database operations require much more than simply running a MongoDB container.

## 15. Health Checks

CodeBuddy should expose a lightweight endpoint such as:

    GET /health

Possible response:

    { "status": "ok" }

Docker and Kubernetes can use health information.

Kubernetes commonly provides:

- liveness probe — is the application alive?
- readiness probe — is it ready to receive traffic?
- startup probe — has startup completed?

Readiness is especially important during deployments because traffic should not be sent to an application that has not finished starting.

## 16. Resource Management

Production workloads should define appropriate CPU and memory requests/limits.

Requests help Kubernetes schedule workloads.

Limits restrict resource consumption.

Values should be based on measurements and load testing, not arbitrary numbers.

## 17. Rolling Deployment

Suppose CodeBuddy currently runs version 1.0.0:

    v1.0.0
      |
      +-- Pod 1
      +-- Pod 2
      +-- Pod 3

A new version 1.1.0 can be rolled out gradually:

    v1.0.0 -> v1.1.0
    v1.0.0 -> v1.1.0
    v1.0.0 -> v1.1.0

This reduces the risk of taking every replica offline simultaneously.

## 18. Rollback

If the new release is unhealthy, Kubernetes can roll back the Deployment.

Useful commands:

    kubectl rollout status deployment/codebuddy-backend

    kubectl rollout history deployment/codebuddy-backend

    kubectl rollout undo deployment/codebuddy-backend

Versioned images make rollback much easier to understand and reproduce.

## 19. External Traffic

A typical production path is:

    User
      |
      v
    DNS
      |
      v
    Load Balancer / Gateway
      |
      +----------------+
      |                |
      v                v
    Frontend        Backend
    Service         Service
      |                |
      v                v
    Frontend Pods   Backend Pods

The external layer may provide:

- TLS termination
- host/path routing
- load balancing
- security policies

## 20. Observability

Production CodeBuddy needs visibility into:

- application errors
- request latency
- CPU and memory
- container restarts
- Pod health
- database health
- WebSocket connections
- deployment status

Logs should be collected centrally rather than relying only on logs inside an individual container.

## 21. Security Checklist

### Images

- trusted/minimal base images
- version pinning
- dependency/image scanning
- no secrets in images
- non-root runtime where practical

### Containers

- avoid privileged containers
- minimize capabilities
- resource limits
- read-only filesystem where practical

### Application

- input validation
- authentication and authorization
- secure cookies
- HTTPS
- secure payment credentials
- appropriate security headers

### Infrastructure

- restricted network access
- protected Docker socket
- Kubernetes RBAC
- protected cluster credentials
- secret management
- backups

## 22. Complete CodeBuddy Production Flow

The complete flow is:

    Developer
        |
        v
    Git Push
        |
        v
    CI/CD
        |
        +--> Test
        +--> Build
        +--> Docker Build
        +--> Security Scan
        +--> Push Images
        |
        v
    Container Registry
        |
        v
    Kubernetes
        |
        +--> Frontend Deployment
        |       |
        |       +--> Frontend Pods
        |
        +--> Backend Deployment
        |       |
        |       +--> Backend Pods
        |
        +--> Services
        |
        +--> ConfigMap / Secrets
        |
        +--> Health Probes
        |
        +--> Resource Limits
        |
        v
    Database / Redis
        |
        v
    Users

## 23. Local → Production Progression

### Stage 1 — Normal application

    Frontend
    Backend
    MongoDB

### Stage 2 — Docker

    Frontend Container
    Backend Container
    MongoDB Container

### Stage 3 — Compose

    docker compose up

Use Compose for repeatable local development.

### Stage 4 — Registry

    Docker Build
        |
        v
    Registry

### Stage 5 — CI/CD

    Git Push
        |
        v
    Jenkins / GitHub Actions
        |
        v
    Test + Build + Push

### Stage 6 — Kubernetes

    Registry
       |
       v
    Kubernetes
       |
       +--> Frontend
       +--> Backend
       +--> Services
       +--> Config
       +--> Secrets

### Stage 7 — Production hardening

Add:

- HTTPS
- DNS/domain
- monitoring
- centralized logging
- backups
- health probes
- resource limits
- security scanning
- secret management
- rollback strategy

## 24. Docker Course Connection

The course now connects as:

    01-05  Docker Fundamentals
        |
        v
    06-10  Images + Dockerfiles
        |
        v
    11-14  Dockerizing Applications
        |
        v
    15-18  Storage + Networking + Configuration
        |
        v
    19-24  Docker Compose
        |
        v
    25-28  Registry + CI/CD + Deployment
        |
        v
    30-35  Advanced Docker + Orchestration
        |
        v
    36     CodeBuddy Production Flow

The goal is now to be able to explain:

> How does a real application move from source code to a production containerized deployment?

## 25. Interview Questions

### Q1. Explain the complete Docker deployment flow.

Developer pushes code to Git. CI/CD tests the application, builds Docker images, scans them, and pushes versioned images to a registry. The deployment platform pulls those images and runs them as containers or Kubernetes Pods. Services provide networking, configuration and secrets are injected at runtime, health checks verify the application, and monitoring provides operational visibility.

### Q2. Why use Docker Compose?

To define and run multiple related services consistently, especially during local development and testing.

### Q3. Why use a container registry?

To store versioned container images so deployment environments can pull the exact application artifact they need.

### Q4. Why should MongoDB data not exist only inside a container?

Containers are replaceable. Persistent database data requires persistent storage, backups, and a recovery strategy.

### Q5. Why is localhost often wrong inside a container?

Inside a container, localhost refers to that same container. Other services should normally be reached through their Docker/Compose/Kubernetes service name.

### Q6. Why does CodeBuddy need special consideration for Socket.IO scaling?

Different backend replicas can handle different client connections. Shared coordination, such as a Redis adapter, may be required for events to propagate between instances.

### Q7. Why use multi-stage builds?

To keep build-time dependencies out of the final runtime image, reducing size and attack surface.

### Q8. What is Kubernetes responsible for?

Kubernetes manages containerized workloads including scheduling, replicas, networking, updates, recovery, configuration, and desired-state reconciliation.

## ⭐ Must Remember

1. Docker packages applications into portable images.
2. Compose is excellent for local multi-container development.
3. A registry stores versioned deployment artifacts.
4. CI/CD automates test → build → scan → push → deploy.
5. Kubernetes orchestrates containerized workloads at cluster scale.
6. Deployment manages Pods; Service provides stable networking.
7. Containers are replaceable; persistent data needs persistent storage.
8. Service names are used for container-to-container communication.
9. Browser networking and container networking are different.
10. Frontend public environment variables are not secrets.
11. Backend secrets must never be baked into images or committed to Git.
12. Health and readiness checks are important for reliable deployments.
13. Socket.IO requires additional planning when backend replicas scale horizontally.
14. Production needs observability, security, backups, and rollback.
15. The core flow is:

    Code
      ↓
    Git
      ↓
    CI/CD
      ↓
    Test
      ↓
    Docker Build
      ↓
    Security Scan
      ↓
    Container Registry
      ↓
    Kubernetes
      ↓
    Pods + Services
      ↓
    Users

## Course Completion

Lesson 36 completes the Docker course by connecting the individual concepts to one realistic CodeBuddy production architecture.

The next step is hands-on implementation: build the CodeBuddy Dockerfiles, Compose setup, registry workflow, and Kubernetes manifests.
