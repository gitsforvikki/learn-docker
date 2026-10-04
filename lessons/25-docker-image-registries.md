# Lesson 25 — Docker Image Registries

## 1. What Is a Docker Image Registry?

A Docker image registry is a service that stores and distributes container images.

Mental model:

~~~text
Developer
   │
   │ docker build
   ▼
Docker Image
   │
   │ docker push
   ▼
Image Registry
   │
   │ docker pull
   ▼
Server / CI/CD
   │
   ▼
Container
~~~

A registry lets CI/CD build an image once and lets deployment systems pull that exact artifact.

---

## 2. Why Registries Matter

Without a registry:

~~~text
Source Code → build separately on each server
~~~

With a registry:

~~~text
Source Code
   ↓
CI/CD builds image
   ↓
Push image
   ↓
Registry
   ↓
Deployment pulls image
   ↓
Container
~~~

This reduces environment differences and makes deployments reproducible.

---

## 3. Common Registries

Important examples:

- Docker Hub
- GitHub Container Registry (GHCR)
- Amazon Elastic Container Registry (ECR)
- Google Artifact Registry
- Azure Container Registry
- GitLab Container Registry
- Private/self-hosted registries

The core workflow is:

~~~text
login → tag → push → pull → run
~~~

---

## 4. Registry, Repository, Image and Tag

Example:

~~~text
ghcr.io/company/codebuddy-backend:1.0.0
│       │       │                 │
│       │       │                 └── Tag
│       │       └──────────────────── Repository
│       └──────────────────────────── Namespace/owner
└──────────────────────────────────── Registry
~~~

General image reference:

~~~text
[registry/]namespace/repository:tag
~~~

Docker Hub example:

~~~text
username/codebuddy-backend:1.0.0
~~~

---

## 5. Build an Image

~~~bash
docker build -t codebuddy-backend:1.0.0 .
docker images
~~~

The image now exists locally.

---

## 6. Tag an Image

To prepare an image for a registry:

~~~bash
docker tag codebuddy-backend:1.0.0 username/codebuddy-backend:1.0.0
~~~

GHCR example:

~~~bash
docker tag codebuddy-backend:1.0.0 ghcr.io/username/codebuddy-backend:1.0.0
~~~

Tagging does not rebuild the image. It creates another reference to the same image content.

---

## 7. Login

Docker Hub:

~~~bash
docker login
~~~

Specific registry:

~~~bash
docker login ghcr.io
~~~

For CI/CD, use protected credentials/secrets.

Never put registry passwords or access tokens in Dockerfiles, Compose files committed to Git, or source code.

---

## 8. Push an Image

~~~bash
docker push username/codebuddy-backend:1.0.0
~~~

GHCR:

~~~bash
docker push ghcr.io/username/codebuddy-backend:1.0.0
~~~

During a push, Docker uploads required image layers and the image manifest.

Existing layers can be reused, which makes distribution more efficient.

---

## 9. Pull an Image

On another machine:

~~~bash
docker pull username/codebuddy-backend:1.0.0
docker run username/codebuddy-backend:1.0.0
~~~

The deployment server does not need the application source code to run the image.

---

## 10. Public vs Private Images

### Public

Anyone allowed to access the repository can generally pull the image.

Useful for open-source projects and public examples.

### Private

Authentication is required.

Useful for company applications, proprietary software, and production services.

---

## 11. Image Tags

Examples:

~~~text
codebuddy-backend:latest
codebuddy-backend:1.0.0
codebuddy-backend:1.1.0
codebuddy-backend:2026-10-05
codebuddy-backend:a81ecfe
~~~

A tag is a reference and can be changed to point to different image content.

Therefore:

~~~text
latest ≠ permanently fixed version
~~~

For production, prefer release versions or commit-SHA tags.

---

## 12. Why latest Is Risky

Suppose:

~~~text
Today:
latest → version A

Tomorrow:
latest → version B
~~~

A later pull can therefore produce different content.

This makes rollbacks and debugging harder.

Prefer:

~~~text
1.0.0
1.1.0
1.2.0
~~~

or:

~~~text
git-a81ecfe
git-b7c21d4
~~~

For highly controlled deployments, pin the image digest.

---

## 13. Image Digests

An image digest identifies image content.

Example:

~~~text
sha256:abc123...
~~~

Image references can use:

~~~text
codebuddy-backend:1.0.0
codebuddy-backend@sha256:...
~~~

Tags are convenient for humans. Digests provide exact content-based identification.

---

## 14. Tag vs Digest

| Tag | Digest |
|---|---|
| Human-friendly | Content-based |
| Can move | Identifies exact content |
| Easy to use | Strong reproducibility |
| 1.0.0 | sha256:... |
| Good release label | Good exact deployment reference |

A production workflow can use a release tag for readability and record/pin the digest for exact deployment.

---

## 15. Image Layers and Registries

Docker images contain layers:

~~~text
Image
 ├── Layer 1
 ├── Layer 2
 ├── Layer 3
 └── Layer 4
~~~

A registry stores these layers and the image manifest.

If another image already uses a layer, that content can be reused instead of uploaded again.

---

## 16. Registry Does Not Store Running Containers

A registry stores images.

It does not store the runtime state of a container.

~~~text
Image
   ↓
Registry
   ↓
Pull
   ↓
Container
~~~

Container-specific runtime state such as writable filesystem changes, running processes, and temporary state is not what gets published as an image.

Persistent application data should use appropriate storage such as Docker volumes or managed databases.

---

## 17. Registry Workflow with CI/CD

Typical production pipeline:

~~~text
Developer pushes Git commit
          ↓
CI/CD starts
          ↓
Run tests
          ↓
Build Docker image
          ↓
Tag with version / commit SHA
          ↓
Login to registry
          ↓
Push image
          ↓
Deploy exact image
          ↓
Health checks
~~~

Clean separation:

~~~text
Git → source control
Registry → image artifact storage
Server → runtime
~~~

---

## 18. CodeBuddy Production Image Flow

CodeBuddy has separate frontend and backend repositories.

Conceptually:

~~~text
Frontend repo
    ↓
Build frontend image
    ↓
Registry
    ↓
Deployment server

Backend repo
    ↓
Build backend image
    ↓
Registry
    ↓
Deployment server
~~~

Runtime:

~~~text
Frontend container
       ↓
Backend container
       ↓
MongoDB
       ↓
Persistent storage
~~~

The registry stores the frontend/backend images; it does not replace the database or persistent storage.

---

## 19. Registry Does Not Deploy Your Application

A registry's responsibility is primarily image storage and distribution.

This is not automatically:

~~~text
docker push → application deployed
~~~

Deployment requires another system/process, such as:

- Docker on a server
- Docker Compose
- Kubernetes
- Jenkins
- GitHub Actions
- a cloud container platform

Mental model:

~~~text
Git
 ↓
CI/CD
 ↓
Registry
 ↓
Deployment platform
 ↓
Container
~~~

---

## 20. Useful Commands

### Build

~~~bash
docker build -t app:1.0.0 .
~~~

### Tag

~~~bash
docker tag app:1.0.0 username/app:1.0.0
~~~

### Login

~~~bash
docker login
~~~

### Push

~~~bash
docker push username/app:1.0.0
~~~

### Pull

~~~bash
docker pull username/app:1.0.0
~~~

### Inspect

~~~bash
docker image inspect username/app:1.0.0
~~~

### Run

~~~bash
docker run username/app:1.0.0
~~~

---

## 21. Common Mistakes

### Mistake 1 — Using latest everywhere

Production becomes less predictable.

### Mistake 2 — Hard-coding registry credentials

Use secure CI/CD credentials instead.

### Mistake 3 — Confusing registry with deployment

A registry stores images; another system runs them.

### Mistake 4 — Rebuilding directly on production

Prefer building in CI/CD and deploying the resulting image.

### Mistake 5 — Deploying only an unversioned image

Use a version or commit-based tag for traceability.

### Mistake 6 — Assuming an image contains persistent data

Database data belongs in persistent storage, not image layers.

### Mistake 7 — Forgetting private registry authentication

A deployment server must authenticate before pulling private images.

---

## 22. Best Practices

1. Use a trusted registry.
2. Keep production repositories private when appropriate.
3. Never commit registry credentials.
4. Prefer immutable version or commit tags.
5. Avoid relying on latest for production.
6. Build images in CI/CD.
7. Scan images for vulnerabilities.
8. Keep runtime images small.
9. Retain important release images for rollback.
10. Use image digests when exact reproducibility is required.
11. Separate source control, image storage, and runtime responsibilities.
12. Use least-privilege registry credentials.

---

# Interview Questions

### 1. What is a Docker registry?

A service that stores and distributes Docker images.

### 2. What is the difference between Docker Hub and a private registry?

Docker Hub is a hosted registry service with public and private repositories. A private registry restricts access to authorized users or systems.

### 3. What is the difference between an image tag and digest?

A tag is a human-friendly reference that can move. A digest identifies exact image content.

### 4. Why is latest not recommended for production?

Because it can point to different image content over time, making deployments and rollbacks less predictable.

### 5. What happens during docker push?

Docker uploads required image layers and the image manifest to the registry.

### 6. Can you push a running container to a registry?

No. Registries distribute images. A running container is a runtime instance created from an image.

### 7. Why tag images with a Git commit SHA?

It creates a traceable relationship between the deployed image and the source commit that produced it.

### 8. Where should registry credentials be stored in CI/CD?

As protected CI/CD secrets or credentials, not in source code.

### 9. Does pushing an image deploy it?

No. The image must be pulled and run by a deployment system.

### 10. Why build once and deploy the same image?

It reduces environment differences and ensures the tested artifact is the artifact deployed to production.

---

# Quick Revision

~~~text
Source Code
    ↓
docker build
    ↓
Docker Image
    ↓
docker tag
    ↓
docker push
    ↓
Registry
    ↓
docker pull
    ↓
Server / Kubernetes / Compose
    ↓
Container
~~~

### Core commands

~~~bash
docker build -t app:1.0.0 .
docker tag app:1.0.0 username/app:1.0.0
docker login
docker push username/app:1.0.0
docker pull username/app:1.0.0
docker image inspect username/app:1.0.0
~~~

---

# ⭐ Must Remember

1. **A registry stores and distributes Docker images.**
2. **Build → tag → push → pull → run is the basic registry workflow.**
3. **Tags can move; digests identify exact image content.**
4. **Avoid latest as the only production identifier.**
5. **Use version or commit-SHA tags for traceability.**
6. **Never store registry credentials in Git.**
7. **A registry stores images, not running containers.**
8. **A registry does not itself deploy the application.**
9. **Build once and deploy the same tested image.**
10. **Registry + CI/CD + deployment platform form the production image delivery pipeline.**
