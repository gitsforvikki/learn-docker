# Lesson 35 — Docker Swarm & Kubernetes

## 1. Why Orchestration?

Running one Docker container is easy:

```bash
docker run -d my-api
```

But production applications often need:

- multiple replicas of an application
- automatic restart when a container fails
- service discovery
- load balancing
- rolling updates
- scaling
- secrets and configuration management
- scheduling workloads across multiple machines

Managing these manually becomes difficult.

**Container orchestration** is the practice of automatically deploying, scheduling, networking, scaling, updating, and recovering containers across one or more machines.

Two important orchestration technologies to understand are:

- **Docker Swarm** — Docker's native orchestration system
- **Kubernetes** — the most widely adopted container orchestration platform

> Docker runs containers. An orchestrator manages containers at scale.

---

# 2. Docker Swarm

Docker Swarm is Docker's built-in orchestration mode.

A Swarm is a cluster of Docker hosts that work together as one logical system.

## Swarm architecture

A Swarm normally contains:

- **Manager nodes** — maintain cluster state and schedule services
- **Worker nodes** — run assigned tasks/containers

A node can also be both a manager and worker.

### Mental model

```text
                    Docker Swarm
                         |
             +-----------+-----------+
             |                       |
        Manager Node            Worker Node
        cluster state           runs tasks
        scheduling
             |
        Worker Node
        runs tasks
```

---

# 3. Swarm Service

In normal Docker:

```bash
docker run my-api
```

You directly create a container.

In Swarm, you normally define a **service**:

```bash
docker service create --name api --publish 8080:3000 my-api
```

A service describes the desired state.

For example:

> Run 3 replicas of `my-api`.

The Swarm manager continuously works to maintain that state.

---

# 4. Swarm Replicas

Create a service with three replicas:

```bash
docker service create \
  --name api \
  --replicas 3 \
  --publish 8080:3000 \
  my-api
```

Check services:

```bash
docker service ls
```

Check tasks:

```bash
docker service ps api
```

Scale:

```bash
docker service scale api=5
```

If a container fails, Swarm can create another task to maintain the desired replica count.

---

# 5. Swarm Overlay Network

A Swarm can connect containers running on different Docker hosts through an **overlay network**.

Create one:

```bash
docker network create \
  --driver overlay \
  app-network
```

Services attached to the same overlay network can communicate using service names.

Example:

```text
Frontend service
      |
      | HTTP
      v
Backend service
      |
      | MongoDB protocol
      v
Mongo service
```

The important idea is:

> Overlay networking allows container-to-container communication across Swarm nodes.

---

# 6. Swarm Ingress / Routing Mesh

Swarm supports an ingress routing mesh.

Suppose a service publishes:

```bash
--publish 8080:3000
```

A request arriving at a Swarm node can be routed to an appropriate service task, even if that task is running on another node.

This makes exposing replicated services easier.

---

# 7. Deploying a Stack in Swarm

A Compose-style file can describe a multi-service Swarm application.

Example:

```yaml
services:
  api:
    image: my-api:1.0
    ports:
      - "8080:3000"
    deploy:
      replicas: 3

  frontend:
    image: my-frontend:1.0
    ports:
      - "80:80"
    deploy:
      replicas: 2
```

Deploy:

```bash
docker stack deploy -c compose.yaml codebuddy
```

Useful commands:

```bash
docker stack ls
docker stack services codebuddy
docker stack ps codebuddy
docker stack rm codebuddy
```

---

# 8. Swarm Rolling Updates

Swarm can update replicas gradually instead of replacing everything at once.

Example:

```bash
docker service update \
  --image my-api:2.0 \
  api
```

This supports safer deployments and reduces downtime.

---

# 9. Swarm Secrets and Configs

Swarm provides dedicated mechanisms for sensitive and non-sensitive configuration.

### Secret

Used for sensitive data such as:

- database passwords
- API credentials
- private keys

Example:

```bash
echo "super-secret-password" | docker secret create db_password -
```

### Config

Used for non-secret configuration such as:

- application configuration
- Nginx configuration
- feature settings

The important distinction:

> Secrets are for sensitive values. Configs are for ordinary configuration.

---

# 10. Kubernetes

Kubernetes is a container orchestration platform designed to manage containerized applications across clusters.

Instead of manually managing individual containers, you declare what the application should look like.

Kubernetes then works continuously to make the actual cluster match that desired state.

---

# 11. Kubernetes Cluster

A Kubernetes cluster contains:

- **Control plane**
- **Worker nodes**

```text
                 Kubernetes Cluster
                         |
          +--------------+--------------+
          |                             |
     Control Plane                  Worker Nodes
          |                       +-----+-----+
          |                       |           |
      API Server                 Node       Node
      Scheduler                 Pods       Pods
      Controllers
      etcd
```

---

# 12. Kubernetes Control Plane

## API Server

The **API server** is the main entry point to the Kubernetes cluster.

Tools such as `kubectl` communicate with the API server.

```text
kubectl
   |
   v
API Server
```

## Scheduler

The scheduler decides which worker node should run a newly created Pod.

It considers available resources and scheduling constraints.

## Controller Manager

Controllers continuously compare:

- desired state
- actual state

and take action when they differ.

This is the foundation of Kubernetes' reconciliation model.

## etcd

`etcd` is the distributed key-value store used by Kubernetes to store cluster state and configuration.

High-level view:

```text
API Server
    |
   etcd
    |
Cluster state
```

You normally interact with Kubernetes through the API rather than directly editing etcd.

---

# 13. Kubernetes Worker Node

A worker node runs application workloads.

Important components include:

### Kubelet

The kubelet runs on each worker node and makes sure the Pods assigned to that node are running correctly.

### Container Runtime

The container runtime actually runs containers.

Modern Kubernetes commonly uses runtimes such as **containerd** or **CRI-O**.

### Networking Components

Kubernetes networking allows Pods and Services to communicate across the cluster.

---

# 14. Pod — The Basic Kubernetes Workload Unit

A **Pod** is the smallest deployable unit in Kubernetes.

A Pod normally contains one application container, but it can contain multiple tightly coupled containers.

Example:

```text
Pod
 |
 +-- Container: Node.js API
```

Multiple containers in the same Pod share networking and can share mounted storage.

### Important

Do not think:

> Pod = container

Instead:

> A Pod is a Kubernetes execution unit that contains one or more containers.

---

# 15. Deployment

A Deployment manages replicated application Pods.

Example:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
        - name: api
          image: my-api:1.0
          ports:
            - containerPort: 3000
```

Apply it:

```bash
kubectl apply -f deployment.yaml
```

Check it:

```bash
kubectl get deployments
kubectl get pods
```

A Deployment provides:

- replica management
- rolling updates
- replacement of failed Pods
- desired-state management

---

# 16. Kubernetes Service

Pods are replaceable and their IP addresses can change.

A **Service** provides a stable network endpoint for a group of Pods.

Example:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api
spec:
  selector:
    app: api
  ports:
    - port: 80
      targetPort: 3000
  type: ClusterIP
```

Apply:

```bash
kubectl apply -f service.yaml
```

The Service selects Pods using labels:

```text
             Service
           api:80
              |
       +------+------+
       |      |      |
     Pod    Pod    Pod
   :3000  :3000  :3000
```

---

# 17. Kubernetes Service Types

## ClusterIP

Default type.

Used for internal cluster communication.

```text
Frontend Pod
     |
     v
api Service
     |
     v
Backend Pods
```

## NodePort

Exposes a service through a port on each node.

Useful for basic external access, but generally not the preferred production entry point.

## LoadBalancer

Requests an external load balancer, when supported by the infrastructure/cloud provider.

Typical production flow:

```text
Internet
   |
Load Balancer
   |
Kubernetes Service
   |
Pods
```

---

# 18. ConfigMap

A ConfigMap stores non-sensitive configuration.

Example:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: api-config
data:
  NODE_ENV: production
  LOG_LEVEL: info
```

Applications can consume ConfigMap values as:

- environment variables
- mounted files

---

# 19. Secret

Kubernetes Secrets are intended for sensitive configuration such as:

- database credentials
- API keys
- tokens

Example:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: api-secret
type: Opaque
stringData:
  DATABASE_PASSWORD: change-me
```

Important security point:

> A Kubernetes Secret should not be treated as automatically equivalent to a fully encrypted password vault. Production environments should use appropriate secret-management and encryption controls.

Never commit real credentials into Git.

---

# 20. Namespace

A Namespace logically separates resources inside a Kubernetes cluster.

Example:

```text
Cluster
 |
 +-- development
 |    +-- frontend
 |    +-- api
 |
 +-- production
      +-- frontend
      +-- api
```

Useful command:

```bash
kubectl get pods -n production
```

Namespaces are useful for organization, access control, quotas, and environment separation.

---

# 21. Desired State and Reconciliation

This is one of the most important Kubernetes concepts.

You declare:

```text
Desired:
3 API Pods
```

Suppose one Pod crashes:

```text
Actual:
2 API Pods
```

Kubernetes controllers detect the difference and create another Pod.

```text
Desired State
     |
     v
Kubernetes Controllers
     |
     v
Actual Cluster State
     |
     | if different
     v
Take corrective action
```

This is called **reconciliation**.

### Interview definition

> Kubernetes is declarative: you describe the desired state, and controllers continuously reconcile the actual state toward it.

---

# 22. Important kubectl Commands

Check cluster:

```bash
kubectl cluster-info
kubectl get nodes
```

Inspect workloads:

```bash
kubectl get pods
kubectl get deployments
kubectl get services
```

More detail:

```bash
kubectl describe pod <pod-name>
kubectl describe deployment <deployment-name>
```

View logs:

```bash
kubectl logs <pod-name>
```

Execute a command:

```bash
kubectl exec -it <pod-name> -- sh
```

Apply configuration:

```bash
kubectl apply -f deployment.yaml
```

Delete resources:

```bash
kubectl delete -f deployment.yaml
```

Scale:

```bash
kubectl scale deployment api --replicas=5
```

Check rollout:

```bash
kubectl rollout status deployment/api
```

---

# 23. Docker Compose vs Swarm vs Kubernetes

| Feature | Docker Compose | Docker Swarm | Kubernetes |
|---|---|---|---|
| Main purpose | Multi-container app development | Docker-native orchestration | Production orchestration |
| Typical scope | One host | Cluster | Cluster |
| Scaling | Basic | Built-in | Built-in |
| Self-healing | Limited | Yes | Yes |
| Rolling updates | Limited | Yes | Yes |
| Service discovery | Yes | Yes | Yes |
| Secrets | Environment/files | Native secrets | Secrets + integrations |
| Complexity | Low | Medium | High |
| Learning curve | Easy | Moderate | High |
| Best use | Local development | Simple Docker orchestration | Large/production clusters |

### Practical rule

- **Compose** → develop and run a multi-container application locally.
- **Swarm** → understand Docker-native orchestration and simpler cluster deployments.
- **Kubernetes** → learn for modern production orchestration and larger ecosystems.

---

# 24. CodeBuddy → Kubernetes Mapping

For CodeBuddy:

```text
CodeBuddy Frontend
        |
        v
Frontend Deployment
        |
Frontend Service
        |
        v
Backend Service
        |
        v
Backend Deployment
        |
        +------> MongoDB
        |
        +------> Redis (if introduced later)
```

A possible mapping:

| CodeBuddy component | Kubernetes object |
|---|---|
| React frontend | Deployment |
| Frontend network endpoint | Service |
| Node/Express backend | Deployment |
| Backend endpoint | Service |
| MongoDB | Stateful workload/service or managed database |
| Environment configuration | ConfigMap |
| JWT/payment/database secrets | Secret or external secret manager |
| Multiple environments | Namespaces |
| External HTTP entry point | LoadBalancer / Ingress or Gateway |

For a real production system, a managed MongoDB service may be preferable to running MongoDB inside the same Kubernetes cluster unless there is a specific operational reason to self-host it.

---

# 25. Docker Image → Kubernetes Deployment Flow

Kubernetes does not replace Docker image building.

A typical flow is:

```text
Source Code
    |
    v
Dockerfile
    |
    v
Docker Image
    |
    v
Container Registry
    |
    v
Kubernetes Deployment
    |
    v
Pods
    |
    v
Service
    |
    v
Users
```

For CodeBuddy:

```text
CodeBuddy Backend
      |
      v
Docker Build
      |
      v
Registry
      |
      v
Kubernetes
      |
      +--> Backend Pods
      |
      +--> Service
      |
      +--> ConfigMap / Secrets
```

This connects the previous Docker lessons directly to orchestration.

---

# 26. What Kubernetes Does NOT Replace

Kubernetes does not remove the need to understand Docker fundamentals.

You still need to understand:

- Docker images
- Dockerfiles
- image registries
- container ports
- environment variables
- networking
- volumes
- health checks
- resource limits
- security

Kubernetes consumes container images and orchestrates workloads built from them.

---

# 27. Common Mistakes

### Mistake 1 — Thinking Kubernetes is a Docker replacement

Kubernetes is an orchestration platform. It manages containerized workloads.

### Mistake 2 — Treating a Pod as exactly the same as a container

A Pod can contain one or more containers.

### Mistake 3 — Using Pod IPs directly

Pods are replaceable.

Use a Kubernetes Service for stable application-to-application communication.

### Mistake 4 — Putting secrets directly in Git

Never commit real credentials.

### Mistake 5 — Running everything as one huge Pod

Separate independently scalable application components into appropriate workloads.

### Mistake 6 — Learning Kubernetes before Docker fundamentals

Without understanding images, containers, networking, volumes, and registries, Kubernetes becomes much harder to understand.

### Mistake 7 — Treating `latest` as a reliable deployment version

Prefer meaningful version tags and, where appropriate, immutable image digests.

---

# 28. Interview Questions

### Q1. What is container orchestration?

It is the automated management of containerized workloads, including scheduling, scaling, networking, updates, and recovery.

### Q2. What is Docker Swarm?

Docker Swarm is Docker's native container orchestration system.

### Q3. What is Kubernetes?

Kubernetes is a declarative container orchestration platform used to automate deployment, scaling, networking, and lifecycle management of containerized applications.

### Q4. What is a Kubernetes Pod?

A Pod is the smallest deployable unit in Kubernetes and contains one or more containers that share networking and storage context.

### Q5. Why do we need a Kubernetes Service?

Pods are ephemeral and their IPs can change. A Service provides a stable network endpoint and routes traffic to matching Pods.

### Q6. What is a Deployment?

A Deployment manages a desired number of replicated Pods and supports controlled updates and replacement of failed Pods.

### Q7. What is the difference between ConfigMap and Secret?

ConfigMap stores ordinary configuration. Secret is intended for sensitive configuration.

### Q8. What is the Kubernetes control plane?

It is the set of components responsible for managing cluster state, scheduling, API access, and reconciliation.

### Q9. What is etcd?

A distributed key-value store that Kubernetes uses to persist cluster state.

### Q10. What is reconciliation?

The continuous process of comparing desired state with actual state and taking corrective action.

### Q11. Docker Compose vs Kubernetes?

Compose is primarily for defining and running multi-container applications, especially during development. Kubernetes is a full orchestration platform for managing workloads across clusters.

### Q12. Why is Docker still important when using Kubernetes?

Kubernetes runs containerized workloads built from container images. Docker knowledge is fundamental for building, packaging, debugging, and securing those images.

---

# 29. Quick Revision

```text
Docker
  |
  +-- Build images
  +-- Run containers
  +-- Network containers
  +-- Store data
  |
  v
Compose
  |
  +-- Run multiple services
  +-- Development workflow
  |
  v
Orchestration
  |
  +-- Swarm
  |     +-- Manager
  |     +-- Worker
  |     +-- Service
  |     +-- Replica
  |     +-- Overlay network
  |     +-- Stack
  |
  +-- Kubernetes
        +-- Control Plane
        +-- Worker Node
        +-- Pod
        +-- Deployment
        +-- Service
        +-- ConfigMap
        +-- Secret
        +-- Namespace
        +-- Reconciliation
```

### ⭐ Must Remember

1. **Compose = multi-container application workflow, mainly local/development.**
2. **Swarm = Docker-native orchestration.**
3. **Kubernetes = full-featured container orchestration platform.**
4. **Pod = smallest Kubernetes deployable unit.**
5. **Deployment = manages replicated Pods and updates.**
6. **Service = stable network endpoint for Pods.**
7. **ConfigMap = non-sensitive configuration.**
8. **Secret = sensitive configuration mechanism; secure it properly.**
9. **Kubernetes follows desired state + reconciliation.**
10. **Kubernetes uses container images; Docker fundamentals remain essential.**
11. **Do not use Pod IPs as stable application endpoints.**
12. **For CodeBuddy, Docker builds the images; a registry stores them; Kubernetes can deploy and manage the resulting workloads.**
