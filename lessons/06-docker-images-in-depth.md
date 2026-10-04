# Lesson 06 — Docker Images in Depth

## 1. What Is a Docker Image?

A Docker image is a **read-only, immutable template** used to create containers.

An image contains the files, dependencies, configuration, and metadata needed to run an application.

Think of it as:

**Image → Template**  
**Container → Running instance of that template**

One image can be used to create many containers.

---

## 2. How Images Are Built

Docker images are usually built from a **Dockerfile**.

A Dockerfile describes the steps required to create an image:

```text
Dockerfile
   ↓
docker build
   ↓
Docker Image
   ↓
docker run
   ↓
Container
```

For example, a Node.js image may contain:

- Base OS/runtime files
- Node.js
- Application dependencies
- Application source code
- Configuration needed to start the application

---

## 3. Image Layers

Docker images are made of **layers**.

Each Dockerfile instruction that changes the filesystem generally creates a new image layer.

Example:

```dockerfile
FROM node:22-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
```

Conceptually:

```text
Layer 4 → Application source
Layer 3 → Installed dependencies
Layer 2 → Working directory / metadata changes
Layer 1 → Node.js base image
```

### Why layers matter

Layers provide:

- **Reusability** — unchanged layers can be shared.
- **Caching** — Docker can reuse previously built layers.
- **Smaller storage usage** — shared layers do not need to be duplicated.
- **Faster builds** — unchanged steps can use cache.

This is one reason Docker builds can become much faster when Dockerfile instructions are ordered correctly.

---

## 4. Read-Only Image vs Writable Container Layer

An image itself is read-only.

When Docker creates a container from an image, it adds a **thin writable layer** on top of the image layers.

```text
Container
┌──────────────────────┐
│ Writable layer       │
├──────────────────────┤
│ Image layer          │
│ Image layer          │
│ Image layer          │
└──────────────────────┘
```

Changes made inside the container are normally written to the container's writable layer.

If the container is deleted, data stored only in that writable layer is lost.

For persistent data, use **volumes or bind mounts**.

---

## 5. Image ID, Repository and Tag

A Docker image can be identified using:

```text
repository:tag
```

Example:

```text
nginx:1.27
```

Here:

- `nginx` → repository/image name
- `1.27` → tag

If no tag is specified, Docker commonly uses:

```text
latest
```

Example:

```bash
docker pull nginx
```

is effectively requesting:

```text
nginx:latest
```

### Important

`latest` does **not** mean "newest image" in a technical sense. It is simply a tag named `latest`.

For production, using an explicit version tag or digest gives more predictable deployments.

---

## 6. Image Repository and Registry

These terms are related but different.

### Image Repository

A repository stores versions/tags of a particular image.

Example:

```text
nginx
```

### Registry

A registry is a service that stores and distributes images.

Examples include:

- Docker Hub
- GitHub Container Registry
- Amazon Elastic Container Registry (ECR)
- Google Artifact Registry

Typical flow:

```text
Dockerfile
   ↓
docker build
   ↓
Local Image
   ↓
docker push
   ↓
Registry
   ↓
docker pull
   ↓
Another Machine
```

---

## 7. Image Naming

A fully qualified image name can look like:

```text
registry.example.com/team/my-app:1.0
```

Parts:

```text
registry.example.com / team / my-app : 1.0
       registry          namespace   repository  tag
```

For Docker Hub, an image may look like:

```text
username/my-app:1.0
```

---

## 8. Important Image Commands

### List local images

```bash
docker image ls
```

or:

```bash
docker images
```

### Pull an image

```bash
docker pull nginx:1.27
```

### Inspect an image

```bash
docker image inspect nginx:1.27
```

This provides detailed metadata such as:

- Image configuration
- Architecture
- Environment
- Entrypoint/CMD
- Root filesystem layers
- Created information

### View image history

```bash
docker image history nginx:1.27
```

This helps understand the layers and commands that contributed to the image.

### Remove an image

```bash
docker image rm nginx:1.27
```

The image cannot be removed if it is still required by existing containers unless the relevant containers are removed first.

---

## 9. Image IDs and Digests

A tag such as:

```text
my-app:1.0
```

is a human-friendly reference.

Docker also uses immutable content-based identifiers.

A **digest** identifies an image by its content:

```text
image@sha256:<digest>
```

Example:

```bash
docker pull nginx@sha256:<digest>
```

### Tag vs Digest

- **Tag** → convenient, can be moved to point to another image.
- **Digest** → content-addressed and immutable for that image content.

For reproducible production deployments, digests provide stronger guarantees than mutable tags.

---

## 10. Image vs Container

| Image | Container |
|---|---|
| Read-only template | Runtime instance |
| Used to create containers | Created from an image |
| Contains application filesystem | Adds writable runtime layer |
| Can be shared/reused | Has its own runtime state |
| Not running | Can be running or stopped |

Important relationship:

```text
One Image
   ↓
Container 1
Container 2
Container 3
```

Deleting a container does not normally delete the image from which it was created.

---

## 11. Image Lifecycle

A common image workflow is:

```text
Build
  ↓
Tag
  ↓
Test
  ↓
Push to Registry
  ↓
Pull on Deployment Machine
  ↓
Run Container
```

Example:

```bash
docker build -t my-app:1.0 .
docker run my-app:1.0
docker tag my-app:1.0 username/my-app:1.0
docker push username/my-app:1.0
```

---

## 12. Common Mistakes

### Mistake 1: Thinking an image is a running application

An image is a template. A container is the runtime instance.

### Mistake 2: Assuming container changes modify the image

Changes made inside a running container do not modify the original image.

### Mistake 3: Treating `latest` as a guaranteed newest version

`latest` is only a tag.

### Mistake 4: Storing important application data in the container layer

The writable container layer is not a replacement for persistent storage.

### Mistake 5: Using unnecessary large images

Larger images increase storage, transfer time, and potentially the attack surface. Image optimization is covered later.

---

## 13. Interview Questions

### Q1. What is a Docker image?

A Docker image is an immutable, read-only template containing the filesystem, dependencies, configuration, and metadata required to create a container.

### Q2. What are Docker image layers?

Images are composed of filesystem layers. Layers can be cached and shared, which improves build speed and storage efficiency.

### Q3. Why are Docker images read-only?

Immutability makes images reusable and predictable. Container-specific changes are stored in the container's writable layer instead.

### Q4. What happens when a container is created from an image?

Docker uses the image's read-only layers and adds a writable container layer on top.

### Q5. What is the difference between a tag and a digest?

A tag is a human-friendly reference that can change. A digest is a content-based immutable identifier for a specific image.

### Q6. What is the difference between an image repository and a registry?

A repository organizes versions/tags of an image, while a registry is the service that stores and distributes image repositories.

### Q7. Why are image layers useful?

They enable caching, reuse, sharing, faster builds, and reduced storage usage.

### Q8. Does deleting a container delete its image?

No. Containers and images are separate Docker objects.

---

## Quick Revision

```text
Image        → Read-only template
Container    → Runtime instance of an image
Layers       → Reusable filesystem building blocks
Tag          → Human-friendly image reference
Digest       → Content-based immutable image identifier
Repository   → Collection of image versions/tags
Registry     → Service that stores/distributes images
Writable     → Container-specific changes
Volume       → Persistent data outside container lifecycle
```

### Must Remember

1. **Images are read-only; containers add a writable layer.**
2. **Images are built from layers.**
3. **Layers enable caching and reuse.**
4. **Tags can change; digests identify specific image content.**
5. **`latest` is a tag, not a guarantee of the newest version.**
6. **Do not depend on the container writable layer for persistent data.**
7. **Image → container is the fundamental Docker relationship.**
