# Lesson 19 — What is Docker Compose?

## 1. Concept / Definition

**Docker Compose** is a tool for defining and running **multi-container Docker applications** using a YAML configuration file.

Instead of starting every container manually with long `docker run` commands, Compose lets us describe the application in one file and manage the complete stack with a small set of commands.

A typical full-stack application may contain:

- Frontend
- Backend API
- Database
- Cache such as Redis
- Reverse proxy

Compose defines these services, their networks, volumes, environment variables, ports, dependencies, and build configuration.

### The core idea

Without Compose:

```text
docker build ...
docker run ...
docker network create ...
docker run backend ...
docker run database ...
docker run frontend ...
```

With Compose:

```text
compose.yaml
     ↓
docker compose up
     ↓
Frontend + Backend + Database + Network + Volumes
```

---

## 2. Why Docker Compose is Needed

Managing multiple containers manually becomes difficult because we must repeatedly configure:

- Container names
- Ports
- Networks
- Environment variables
- Volumes
- Images/build contexts
- Startup dependencies

Compose stores this configuration as code.

### Main benefits

- One file describes the application stack.
- One command can start the stack.
- Services automatically share a Compose network.
- Service names can be used for container-to-container communication.
- Volumes and environment configuration can be declared.
- The same configuration can be used by the whole development team.
- Easy to start and stop the complete application.

> **Important:** Docker Compose is primarily an orchestration/development tool for multi-container applications. It is not the same thing as Kubernetes.

---

## 3. Compose File

The modern Compose specification normally uses:

```text
compose.yaml
```

You may also see:

```text
compose.yml
docker-compose.yml
docker-compose.yaml
```

For new projects, prefer **`compose.yaml`**.

Modern Docker Compose commands use:

```bash
docker compose <command>
```

not the older standalone:

```bash
docker-compose <command>
```

---

## 4. Basic Compose Example

Suppose we have a Node.js API and MongoDB.

### Project structure

```text
my-app/
├── server/
│   ├── package.json
│   ├── src/
│   └── Dockerfile
└── compose.yaml
```

### compose.yaml

```yaml
services:
  api:
    build: ./server
    ports:
      - "3000:3000"
    environment:
      PORT: 3000
      MONGO_URL: mongodb://mongo:27017/myapp
    depends_on:
      - mongo

  mongo:
    image: mongo:8
    volumes:
      - mongo-data:/data/db

volumes:
  mongo-data:
```

This describes two services:

- `api`
- `mongo`

Compose creates the required network and volume.

---

## 5. Understanding the Main Compose Keys

### services

Defines the containers that make up the application.

```yaml
services:
  api:
    ...
  mongo:
    ...
```

Each service normally results in a container.

---

### image

Use an existing Docker image.

```yaml
services:
  mongo:
    image: mongo:8
```

Compose pulls the image if it is not available locally.

---

### build

Build an image from a Dockerfile.

```yaml
services:
  api:
    build: ./server
```

Equivalent conceptually to:

```bash
docker build ./server
```

You can also specify a Dockerfile explicitly:

```yaml
build:
  context: ./server
  dockerfile: Dockerfile
```

---

### ports

Publishes a container port to the host.

```yaml
ports:
  - "3000:3000"
```

Meaning:

```text
Host port 3000 → Container port 3000
```

Important:

- Host/browser → use the published host port.
- Container → container communication normally uses the service name and container port.

---

### environment

Defines environment variables inside the service container.

```yaml
environment:
  NODE_ENV: production
  PORT: 3000
```

For secrets, avoid hard-coding sensitive values in a committed Compose file.

---

### volumes

Mounts persistent storage.

```yaml
volumes:
  - mongo-data:/data/db
```

The named volume is declared at the top level:

```yaml
volumes:
  mongo-data:
```

---

### depends_on

Expresses a startup dependency.

```yaml
depends_on:
  - mongo
```

It does **not** by itself mean that MongoDB is fully ready to accept connections.

For real readiness handling, use health checks and application retry logic where appropriate.

---

## 6. Important Compose Commands

### Start the application

```bash
docker compose up
```

Builds images when necessary and starts the services.

### Start in background

```bash
docker compose up -d
```

### Build images

```bash
docker compose build
```

### Build and start

```bash
docker compose up --build
```

### List services/containers

```bash
docker compose ps
```

### View logs

```bash
docker compose logs
```

Specific service:

```bash
docker compose logs api
```

Follow logs:

```bash
docker compose logs -f api
```

### Stop services

```bash
docker compose stop
```

### Stop and remove containers/network

```bash
docker compose down
```

### Stop and remove containers plus volumes

```bash
docker compose down -v
```

⚠️ `down -v` removes Compose-managed volumes and can delete persistent database data.

---

## 7. The Most Important Networking Concept

Compose normally creates a project-specific network for the services.

For example:

```text
                 Compose Network
                       │
          ┌────────────┼────────────┐
          ↓            ↓            ↓
       frontend       api         mongo
          │            │            │
          └────────────┘            │
                       └─────────────┘
```

The backend can connect to MongoDB using:

```text
mongodb://mongo:27017/myapp
```

**not**:

```text
mongodb://localhost:27017/myapp
```

Inside the `api` container:

- `localhost` means the API container itself.
- `mongo` resolves to the MongoDB service.
- `27017` is MongoDB's container port.

This is one of the most important Docker Compose concepts.

---

## 8. Port Mapping vs Internal Communication

Consider:

```yaml
mongo:
  image: mongo:8
  ports:
    - "27018:27017"
```

From the host:

```text
localhost:27018
```

From another Compose service:

```text
mongo:27017
```

The host mapping does **not** change the internal container port.

### Rule

```text
Browser/Host → published host port
Container → service name + container port
```

---

## 9. Compose Lifecycle

A useful mental model:

```text
compose.yaml
      ↓
docker compose build
      ↓
Docker images
      ↓
docker compose up
      ↓
Containers + Network + Volumes
      ↓
Application running
      ↓
docker compose down
      ↓
Containers/network removed
      ↓
Named volumes remain unless -v is used
```

This explains why application configuration belongs in Compose rather than a collection of manually typed commands.

---

## 10. Compose and Dockerfile Work Together

Compose does **not** replace Dockerfiles.

Dockerfile answers:

> **How do I build this service's image?**

Compose answers:

> **How do all these services run together?**

Example:

```text
Dockerfile
    ↓
Build API image

compose.yaml
    ↓
Run API + MongoDB + Redis + frontend
```

This distinction is frequently asked in interviews.

---

## 11. Using a .env File

Compose supports environment-based configuration.

Example:

```text
.env
```

```env
API_PORT=3000
MONGO_PORT=27017
```

Then:

```yaml
services:
  api:
    ports:
      - "${API_PORT}:3000"
```

The `.env` file is commonly useful for Compose variable substitution.

Do not assume that every `.env` value is automatically a secret. Treat sensitive values carefully and avoid committing secrets to Git.

---

## 12. Example: Node.js + MongoDB

```yaml
services:
  api:
    build:
      context: ./server
    ports:
      - "3000:3000"
    environment:
      NODE_ENV: production
      PORT: 3000
      MONGO_URL: mongodb://mongo:27017/codebuddy
    depends_on:
      - mongo

  mongo:
    image: mongo:8
    volumes:
      - mongo-data:/data/db

volumes:
  mongo-data:
```

Application flow:

```text
Browser
   │
   │ localhost:3000
   ↓
API container
   │
   │ mongo:27017
   ↓
MongoDB container
   │
   ↓
mongo-data volume
```

This is the basic architecture we will build on in the next Compose lessons.

---

## 13. Common Mistakes

### Mistake 1 — Using localhost between containers

Wrong:

```text
mongodb://localhost:27017/db
```

Correct:

```text
mongodb://mongo:27017/db
```

---

### Mistake 2 — Assuming depends_on means ready

```yaml
depends_on:
  - mongo
```

controls startup ordering, but does not guarantee application-level readiness.

---

### Mistake 3 — Accidentally deleting database data

Be careful with:

```bash
docker compose down -v
```

---

### Mistake 4 — Exposing every internal service to the host

A database does not necessarily need:

```yaml
ports:
  - "27017:27017"
```

If only the API needs MongoDB, keep MongoDB internal to the Compose network unless host access is actually required.

---

### Mistake 5 — Putting secrets directly in Git

Avoid:

```yaml
environment:
  DATABASE_PASSWORD: super-secret-password
```

Use appropriate environment/secret management instead.

---

## 14. Best Practices

- Prefer `compose.yaml` for new projects.
- Keep services logically separated.
- Use service names for internal communication.
- Do not use hard-coded container IP addresses.
- Use named volumes for persistent database data.
- Keep databases internal unless host access is required.
- Use health checks when readiness matters.
- Keep secrets out of Git.
- Use explicit image tags rather than relying blindly on `latest`.
- Keep Dockerfiles responsible for image construction and Compose responsible for service orchestration.
- Use `docker compose config` to validate the resolved Compose configuration.

### Validate configuration

```bash
docker compose config
```

This is very useful before starting a complex stack.

---

## 15. Common Debugging Commands

Check running services:

```bash
docker compose ps
```

View logs:

```bash
docker compose logs -f
```

Inspect containers:

```bash
docker ps
```

Inspect networks:

```bash
docker network ls
```

Inspect volumes:

```bash
docker volume ls
```

Open a shell inside a service:

```bash
docker compose exec api sh
```

From there, you can test connectivity to another service.

---

## 16. Interview Questions

### Q1. What is Docker Compose?

Docker Compose is a tool for defining and running multi-container Docker applications using a declarative YAML configuration.

### Q2. What is the difference between Dockerfile and Compose?

**Dockerfile:** defines how an image is built.

**Compose:** defines how multiple services/containers are configured and run together.

### Q3. How do containers communicate in Compose?

They normally communicate through the Compose network using **service names** as DNS names and the target container's internal port.

### Q4. Does depends_on guarantee that a service is ready?

No. It primarily controls dependency/startup ordering. Readiness should be handled with health checks and/or application retry logic.

### Q5. What does `docker compose down -v` do?

It removes the Compose-created containers and networks and also removes the associated Compose-managed volumes.

### Q6. Why should we avoid localhost between Compose services?

Because `localhost` inside a container refers to that same container, not another service.

### Q7. Can Compose build images?

Yes. A service can use `build` to build an image from a Dockerfile.

### Q8. Why use a named volume for MongoDB/PostgreSQL?

Containers are replaceable, but database data must survive container recreation. A named volume provides persistent storage.

---

## 17. Quick Revision

```text
Dockerfile → builds an image
Compose    → runs/configures multiple services

services   → application containers
image      → use an existing image
build      → build an image
ports      → host ↔ container publishing
environment → runtime configuration
volumes    → persistent storage
depends_on → startup dependency/order

Container → Container:
service-name:container-port

Host → Container:
localhost:published-host-port
```

### Essential commands

```bash
docker compose up
docker compose up -d
docker compose up --build
docker compose ps
docker compose logs -f
docker compose build
docker compose exec <service> sh
docker compose stop
docker compose down
docker compose down -v
docker compose config
```

---

## ⭐ Must Remember

1. **Compose manages multi-container applications.**
2. **Dockerfile builds an image; Compose runs/configures services.**
3. **Containers in Compose normally communicate using service names.**
4. **Container-to-container communication uses the container port, not the published host port.**
5. **`localhost` inside a container means that container itself.**
6. **`depends_on` does not automatically mean the dependency is ready.**
7. **Named volumes keep database data persistent across container recreation.**
8. **Be very careful with `docker compose down -v`.**
9. **Do not commit secrets to Compose files or Git.**
10. **Use `docker compose config` to validate the configuration.**

---

## 🎯 Interview Summary

If asked to explain Docker Compose in an interview:

> Docker Compose is a declarative tool for defining and running multi-container applications. We describe services, images/build contexts, ports, networks, volumes, and configuration in a Compose YAML file. Compose creates the application network so services can communicate using service names. Dockerfiles define how individual images are built, while Compose defines how those services run together. For persistent data, we use named volumes, and for production-style readiness we use health checks rather than relying only on `depends_on`.
