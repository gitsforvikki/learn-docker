# Lesson 16 — Bind Mounts & tmpfs

## 1. What is a Bind Mount?

A **bind mount** maps a specific directory or file from the Docker host directly into a container.

Unlike a Docker volume, Docker does not manage the host path.

```
Host filesystem
/home/user/project
        │
        │ bind mount
        ▼
Container
/app
```

**Core idea:**
- source = host path
- target = container path

## 2. Why Use Bind Mounts?

Bind mounts are especially useful when the host and container need to share files directly.

Common use cases:
- Local development
- Hot reload
- Sharing source code with a development container
- Sharing configuration files
- Testing local files inside a container

For production application data, Docker-managed volumes are often a better default because they are less dependent on the host directory structure.

## 3. Basic Bind Mount Syntax

Using `--mount`:

```bash
docker run -d \
  --name nginx-bind \
  --mount type=bind,source=/home/user/site,target=/usr/share/nginx/html \
  nginx
```

Using short `-v` syntax:

```bash
docker run -d \
  --name nginx-bind \
  -v /home/user/site:/usr/share/nginx/html \
  nginx
```

## 4. `--mount` vs `-v`

`--mount` is explicit and easier to read because the mount type, source, target, and options are named.

`-v` is shorter and very common in existing Docker commands and Compose files.

Understand both. Prefer `--mount` when clarity matters.

## 5. Practical Example

Create a host directory:

```bash
mkdir -p ~/docker-bind-demo
echo "Hello from the host" > ~/docker-bind-demo/index.html
```

Run Nginx:

```bash
docker run -d \
  --name nginx-bind-demo \
  -p 8080:80 \
  --mount type=bind,source=$HOME/docker-bind-demo,target=/usr/share/nginx/html,readonly \
  nginx
```

Open `http://localhost:8080`.

If you modify the host file, the container can see the changed file without rebuilding the image. This is why bind mounts are useful during development.

## 6. Read-Only Bind Mounts

A bind mount can be read-only:

```bash
docker run --rm \
  --mount type=bind,source=$HOME/config,target=/app/config,readonly \
  alpine
```

Short form:

```bash
docker run --rm \
  -v $HOME/config:/app/config:ro \
  alpine
```

Read-only mounts are useful when the container only needs to consume host data.

## 7. Bind Mounting a Single File

You can mount a specific file instead of an entire directory:

```bash
docker run --rm \
  --mount type=bind,source=$HOME/app/config.json,target=/app/config.json,readonly \
  node:22-alpine
```

Use this carefully for sensitive configuration. Consider file permissions, backups, and source control.

## 8. Bind Mounts for Development

A development container can mount the project source:

```bash
docker run --rm -it \
  --mount type=bind,source="$PWD",target=/app \
  node:22-alpine \
  sh
```

The host source is immediately visible inside `/app`.

For real Node.js projects, `node_modules` may need separate handling so host and container dependencies do not conflict.

## 9. Important: Bind Mounts Can Hide Image Files

Suppose the image contains `/app/package.json`, `/app/node_modules`, and `/app/src`.

If you mount a host directory at `/app`, the mounted directory becomes what the container sees at that path.

Files that existed in the image at that location are hidden while the mount is active. They are not necessarily deleted from the image.

This is a common debugging issue.

## 10. File Ownership and Permissions

Bind mounts use the host filesystem directly, so Linux ownership and permissions matter.

Common errors include:

```text
Permission denied
EACCES
Cannot write file
```

A container process running as a non-root user may not have permission to write to a host directory owned by another UID/GID.

## 11. Bind Mounts with Docker Compose

Example:

```yaml
services:
  api:
    build: .
    volumes:
      - ./src:/app/src
```

A common development pattern is:

```yaml
services:
  api:
    build: .
    volumes:
      - .:/app
```

Be careful with broad mounts such as `.:/app`, because they can hide files copied into the image and expose more host files than necessary.

## 12. Docker Volume vs Bind Mount

| Feature | Docker Volume | Bind Mount |
|---|---|---|
| Storage managed by Docker | Yes | No |
| Uses explicit host path | No | Yes |
| Good for persistent app data | Yes | Sometimes |
| Good for local development | Sometimes | Yes |
| Host filesystem dependency | Low | High |
| Direct host file access | No | Yes |

Mental model:

```
Volume:      Container → Docker-managed storage
Bind mount:  Container → specific host path
```

## 13. What is tmpfs?

**tmpfs** is a memory-backed temporary filesystem.

Example:

```bash
docker run -d \
  --name tmpfs-demo \
  --mount type=tmpfs,target=/app/tmp \
  nginx
```

Short form:

```bash
docker run -d \
  --tmpfs /app/tmp \
  nginx
```

tmpfs data is temporary and should not be used as persistent application storage.

## 14. Why Use tmpfs?

tmpfs is useful for temporary data that should not be written to persistent storage.

Examples:
- Temporary processing files
- Scratch space
- Short-lived application data
- Data that only needs to exist during the container lifetime

Because tmpfs uses memory, it should not be treated as unlimited storage.

## 15. tmpfs Size

You can limit tmpfs size:

```bash
docker run --rm \
  --mount type=tmpfs,target=/app/tmp,tmpfs-size=64m \
  alpine
```

Advanced tmpfs options depend on Docker and the host operating system.

## 16. Three Main Docker Storage Choices

### Volume
`Volume = Docker-managed persistent storage.`

Use it when application data must persist independently of the container.

### Bind mount
`Bind mount = direct host filesystem access.`

Use it when the container needs specific host files or directories, especially during development.

### tmpfs
`tmpfs = temporary memory-backed storage.`

Use it for short-lived data that does not need persistence.

## 17. Common Mistakes

### Mistake 1 — Using bind mounts everywhere
Bind mounts tightly couple containers to host directory structures.

### Mistake 2 — Accidentally hiding application files
Mounting `.:/app` can hide files copied into `/app` during the image build.

### Mistake 3 — Ignoring permissions
Host UID/GID and container user permissions can cause `EACCES` errors.

### Mistake 4 — Writing important data to tmpfs
tmpfs is temporary and memory-backed.

### Mistake 5 — Exposing too much host data
Mount the smallest host path needed.

### Mistake 6 — Assuming `ro` means secret
Read-only means the container cannot write through that mount. It does not encrypt or hide the data.

## 18. Best Practices

- Prefer named volumes for Docker-managed persistent application data.
- Use bind mounts when direct host filesystem access is actually required.
- Use read-only mounts when write access is unnecessary.
- Mount the smallest host path needed.
- Avoid mounting sensitive host directories.
- Understand UID/GID and Linux permissions.
- Use tmpfs only for genuinely temporary data.
- Do not depend on tmpfs for persistence.
- Be careful with `.:/app` because it can hide image contents.
- Keep production containers independent of unnecessary host filesystem paths.

## 19. Interview Questions

### Q1. What is a bind mount?
A bind mount maps a specific host filesystem path into a container.

### Q2. How is a bind mount different from a volume?
A volume is managed by Docker, while a bind mount directly references a host path.

### Q3. Why are bind mounts common in development?
They allow host source-code changes to appear inside the container immediately, enabling workflows such as hot reload.

### Q4. Can a bind mount be read-only?
Yes. Use `readonly` with `--mount` or `:ro` with `-v`.

### Q5. What is tmpfs?
tmpfs is a memory-backed temporary filesystem used for short-lived data.

### Q6. Does tmpfs persist after a container is removed?
No. It is temporary storage.

### Q7. What happens when a mount covers a directory containing image files?
The mounted filesystem hides the underlying image files at that location while the mount is active.

### Q8. Why can bind mounts cause permission problems?
Because the container interacts directly with the host filesystem, so host ownership and permissions affect access.

### Q9. Which storage should generally be used for persistent database data?
A Docker volume is usually the better default unless infrastructure requirements specifically call for a host path.

## 20. Quick Revision

- Bind mount = specific host path mapped into a container.
- Volume = Docker-managed persistent storage.
- tmpfs = temporary memory-backed storage.
- `--mount` is explicit and readable.
- `-v` is shorter and common.
- Bind mounts are very useful for development.
- Bind mounts can create host/container permission problems.
- A mount can hide files that already exist in the image at that path.
- Read-only mounts reduce accidental modification.
- tmpfs data is not persistent.
- Do not use tmpfs for important persistent application data.
- Avoid unnecessarily broad host mounts.

## Must Remember

> **Volume = Docker-managed persistence.**

> **Bind mount = direct host filesystem access.**

> **tmpfs = temporary memory-backed storage.**

## Interview Summary

- **Need persistent Docker-managed application data? → Volume**
- **Need live host ↔ container file sharing? → Bind mount**
- **Need temporary memory-backed storage? → tmpfs**