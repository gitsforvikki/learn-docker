# Lesson 02 — Docker Architecture

## 1. Docker Architecture

Docker follows a **client-server architecture**.

The main components are:

- **Docker CLI (Client)** — sends commands
- **Docker Daemon (`dockerd`)** — performs Docker operations
- **Docker Engine** — core technology used to build and run containers
- **Docker Registry** — stores and distributes Docker images

### Basic flow

```text
Docker CLI
    ↓
Docker API
    ↓
Docker Daemon (dockerd)
    ↓
Images / Containers / Networks / Volumes
    ↓
Application
```

## 2. Docker CLI

The **Docker CLI** is the command-line interface used to interact with Docker.

Examples:

```bash
docker images
docker ps
docker pull nginx
docker run nginx
```

The CLI sends the requested operation to the Docker daemon through the Docker API.

## 3. Docker Daemon (`dockerd`)

The **Docker daemon** is the background service responsible for managing Docker objects and operations.

It manages:

- Images
- Containers
- Networks
- Volumes

For example, when you run:

```bash
docker run nginx
```

the CLI sends the request to `dockerd`, and the daemon handles the required work.

## 4. Docker Engine

**Docker Engine** is the core Docker technology used to build and run containers.

> Docker CLI is how you interact with Docker; `dockerd` is the daemon that performs and manages Docker operations.

## 5. Docker Registry

A **Docker registry** stores and distributes Docker images.

Examples include:

- Docker Hub
- Private company registries
- Cloud container registries

When an image is not available locally, Docker can pull it from a registry.

## 6. What Happens with `docker run nginx`?

The simplified flow is:

```text
Docker CLI
    ↓
Docker Daemon
    ↓
Is nginx image available locally?
    │
    ├── Yes → Create container
    │
    └── No
         ↓
      Pull image from Registry
         ↓
      Store image locally
         ↓
      Create container
         ↓
      Start container
         ↓
      Nginx runs
```

## 7. `docker pull` vs `docker run`

### `docker pull`

Downloads an image.

```bash
docker pull nginx
```

It **does not create or start a container**.

### `docker run`

Uses an image to create and start a container.

```bash
docker run nginx
```

If the image is not available locally, Docker first pulls it from a registry.

## 8. Interview Questions

### Q1. What are the main components of Docker architecture?

**Answer:** Docker CLI, Docker daemon, Docker Engine, and Docker registries.

### Q2. What does `dockerd` do?

**Answer:** `dockerd` is the background Docker daemon that manages images, containers, networks, volumes, and Docker operations.

### Q3. Does Docker CLI directly manage containers?

**Answer:** No. The CLI sends commands through the Docker API to the Docker daemon, which performs the operations.

### Q4. What happens when you run `docker run nginx`?

**Answer:** Docker checks for the image locally. If it is absent, it pulls the image from a registry, stores it locally, creates a container, and starts it.

### Q5. What is a Docker registry?

**Answer:** A registry is a storage and distribution system for Docker images.

### Q6. Does `docker pull` create a container?

**Answer:** No. It only downloads the image.

## ⭐ Quick Revision

```text
CLI
 ↓
Docker API
 ↓
dockerd
 ↓
Images / Containers / Networks / Volumes
 ↓
Application
```

- **CLI** → sends commands
- **Docker API** → communication layer
- **`dockerd`** → performs/manages Docker operations
- **Registry** → stores/distributes images
- **Image** → used to create containers
- **`docker pull`** → downloads image
- **`docker run`** → creates + starts container
