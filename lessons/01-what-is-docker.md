# Lesson 01 — What is Docker?

## 1. What is Docker?

**Docker is a platform for building, packaging, distributing, and running applications in containers.**

The main idea is to package an application together with the things it needs to run—such as its runtime, dependencies, and configuration—so it behaves consistently across different environments.

Docker helps solve the classic:

> **“It works on my machine.”**

Instead of installing and configuring the application separately on every machine, we package the application and its required environment into a container.

---

## 2. Why Do We Need Docker?

Without containerization, an application may depend on:

- A specific programming language/runtime version
- Specific package or library versions
- Operating-system-level dependencies
- Environment configuration
- System tools
- Different development, testing, and production setups

This can cause an application to work on a developer's machine but fail in testing or production.

### Docker's solution

Docker provides a consistent way to package and run applications.

A simplified model is:

```text
Application
    +
Runtime
    +
Dependencies
    +
Configuration
    ↓
Docker Image
    ↓
Container
```

The same image can then be used across development, testing, staging, production, and cloud environments.

---

## 3. What is a Container?

A **container** is an isolated running environment for an application.

It contains the application and the dependencies required by that application while remaining isolated from other processes and applications.

For example:

```text
Host Machine
│
├── Container A
│   └── Node.js application
│
├── Container B
│   └── MongoDB
│
└── Container C
    └── Redis
```

Each container can run a different service with its own required environment.

---

## 4. Docker Image vs Container

A very important mental model:

> **Image = read-only template/package**  
> **Container = running instance of an image**

Example:

```text
Docker Image
     │
     ├── Container 1
     ├── Container 2
     └── Container 3
```

One image can be used to create multiple containers.

An image is not the running application itself. It is the packaged blueprint from which containers are created.

---

## 5. Docker vs Virtual Machines

Docker containers and virtual machines provide isolation, but they work differently.

### Virtual Machine

```text
Physical/Cloud Machine
│
├── Hypervisor
│   │
│   ├── VM 1
│   │   └── Guest OS
│   │       └── Application
│   │
│   └── VM 2
│       └── Guest OS
│           └── Application
```

Each VM includes a complete guest operating system.

### Docker Containers

```text
Host Operating System
│
├── Docker Engine
│   ├── Container 1 → Application
│   ├── Container 2 → Application
│   └── Container 3 → Application
```

Containers generally share the host kernel rather than carrying a complete guest OS for every application.

### Main difference

| Virtual Machine | Docker Container |
|---|---|
| Includes a complete guest OS | Shares the host kernel |
| Generally heavier | Generally lightweight |
| Usually slower to start | Usually starts quickly |
| Strong OS-level isolation | Process-level isolation |
| Higher resource overhead | Lower resource overhead |

**Important:** Containers are not simply “small virtual machines.” They use OS-level mechanisms for process isolation and resource management.

---

## 6. Important Docker Terminology

### Docker

The platform/tooling used to build, package, distribute, and run containerized applications.

### Image

A read-only packaged template containing the application and its required environment.

### Container

A running instance created from an image.

### Dockerfile

A text file containing instructions for building a Docker image.

### Docker Engine

The underlying technology that builds and runs containers.

These concepts will be explored in detail in later lessons.

---

## 7. Basic Docker Workflow

A typical Docker workflow looks like this:

```text
Write Application
       ↓
Create Dockerfile
       ↓
Build Docker Image
       ↓
Store / Distribute Image
       ↓
Run Container
       ↓
Application Runs
```

For example:

```bash
docker build -t my-app .
docker run my-app
```

The exact Dockerfile and command options will be covered in later lessons.

---

## 8. Why Companies Use Docker

Docker is useful because it provides:

### 1. Consistency

The same packaged application environment can be used across development, testing, and production.

### 2. Portability

The image can be moved between compatible environments such as local machines, servers, and cloud infrastructure.

### 3. Isolation

Applications and their processes can be isolated from one another.

### 4. Reproducible deployments

A specific image can represent a specific version of an application and its dependencies.

### 5. Efficient resource usage

Because containers share the host kernel instead of running a complete guest OS for every application, they generally require fewer resources than VMs.

### 6. Faster application startup

Containers generally start much faster than full virtual machines.

---

## 9. Docker in a Full-Stack Application

A real application can be split into multiple containers.

For example:

```text
                    Docker Host
                        │
        ┌───────────────┼───────────────┐
        │               │               │
        ▼               ▼               ▼
   Frontend          Backend         Database
   Next.js           Node.js         MongoDB
   Container         Container       Container
```

Each service can have its own image and container.

Later, **Docker Compose** can be used to manage multiple related containers as one application stack.

---

## 10. Common Misconceptions

### Docker is not a programming language

Docker is a containerization platform/tooling ecosystem.

### A container is not an image

An image is the packaged template; a container is a running instance of that image.

### Docker is not the same as a VM

Containers use a different isolation model and generally do not require a complete guest OS per application.

### Docker does not automatically make an application secure

Containerization provides isolation, but application, image, host, secrets, permissions, and network security still need to be handled correctly.

---

## 11. Common Beginner Mistakes

- Thinking Docker and virtual machines are exactly the same
- Confusing images with containers
- Thinking a container always contains a complete operating system
- Treating Docker as only a deployment tool instead of understanding the build/package/run lifecycle
- Assuming containerization alone guarantees security
- Putting application data that must survive container replacement only inside the container's writable layer

---

## 12. Interview Questions

### Q1. What is Docker?

**Answer:** Docker is a platform for building, packaging, distributing, and running applications in isolated containers.

### Q2. Why is Docker used?

**Answer:** Docker provides consistency, portability, isolation, reproducible deployments, and efficient application packaging across environments.

### Q3. What problem does Docker solve?

**Answer:** It helps solve environment inconsistencies such as “it works on my machine” by packaging an application together with its runtime and dependencies.

### Q4. What is the difference between an image and a container?

**Answer:** An image is a read-only packaged template, while a container is a running instance created from that image.

### Q5. Docker vs VM?

**Answer:** A VM normally includes a complete guest operating system, while containers generally share the host kernel and use process-level isolation, making containers lighter and faster to start.

### Q6. Can multiple containers be created from one image?

**Answer:** Yes. One image can be used to create multiple containers.

---

## 13. Quick Revision

```text
Docker
  ↓
Build + Package + Distribute + Run
  ↓
Application + Dependencies
  ↓
Docker Image
  ↓
Running Container
```

### ⭐ Must Remember

- **Docker = platform/tooling for containers**
- **Image = read-only template/package**
- **Container = running instance of an image**
- Containers generally share the **host kernel**
- Docker helps provide **consistency, portability, isolation, and reproducibility**
- Docker is different from a VM
- One image can create multiple containers

### 🎯 Interview Summary

> **Docker packages an application and its dependencies into a portable image that can be used to create isolated, reproducible containers. Compared with traditional virtual machines, containers generally use fewer resources because they share the host kernel rather than running a complete guest operating system for each application.**

---

## What's Next?

**Lesson 02 — Docker Architecture**

The next lesson explains how Docker Engine, the CLI, images, containers, and the Docker daemon work together.
