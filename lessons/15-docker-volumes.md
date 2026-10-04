# Lesson 15 — Docker Volumes & Persistent Storage

## 1. Why Storage Matters

A container's writable layer is temporary. When a container is removed, data written only inside that container is lost.

This creates a problem for stateful applications:

- Databases need persistent data.
- Uploaded files may need to survive container recreation.
- Application-generated files may need to remain available.
- Containers should be replaceable without losing important data.

Docker provides storage mechanisms that let data live outside the container's writable layer.

**Core idea:**

`Container filesystem` = temporary runtime data

`Volume / bind mount` = data that can survive container recreation

---

## 2. Container Writable Layer

When Docker starts a container from an image, Docker adds a writable layer on top of the read-only image layers.

Example:

```
Image
├── Application code
├── Dependencies
└── Base OS files
        ↓
Container writable layer
├── logs
├── temporary files
└── runtime changes
```

If the container is deleted, its writable layer is deleted too.

Therefore:

> **Never rely on the container writable layer for important persistent data.**

---

## 3. Docker Volume

A **Docker volume** is storage managed by Docker and stored outside the container's writable layer.

Typical flow:

```
Container
    │
    │ mounts
    ▼
Docker Volume
    │
    ▼
Persistent data
```

The container can be removed and recreated while the volume remains.

Volumes are the preferred Docker storage mechanism for many containerized stateful workloads.

---

## 4. Create and Inspect a Volume

Create:

```bash
docker volume create app-data
```

List volumes:

```bash
docker volume ls
```

Inspect:

```bash
docker volume inspect app-data
```

Remove:

```bash
docker volume rm app-data
```

Remove unused volumes:

```bash
docker volume prune
```

Be careful with `docker volume prune`: unused volumes may contain data you still need.

---

## 5. Mount a Volume into a Container

Example with Nginx:

```bash
docker volume create nginx-data

docker run -d \
  --name nginx-volume \
  -p 8080:80 \
  --mount source=nginx-data,target=/usr/share/nginx/html \
  nginx
```

Short syntax:

```bash
docker run -d \
  --name nginx-volume \
  -v nginx-data:/usr/share/nginx/html \
  nginx
```

Both mount the Docker volume at:

```
/usr/share/nginx/html
```

inside the container.

### Recommended syntax

For learning and production configuration, `--mount` is often clearer because source and target are explicit.

---

## 6. Volume Persistence Example

Start a container and write data:

```bash
docker volume create demo-data

docker run --rm \
  --mount source=demo-data,target=/data \
  alpine \
  sh -c "echo 'Hello Docker' > /data/message.txt"
```

The container is removed because of `--rm`, but the volume remains.

Create another container using the same volume:

```bash
docker run --rm \
  --mount source=demo-data,target=/data \
  alpine \
  cat /data/message.txt
```

Output:

```
Hello Docker
```

This demonstrates the key property of volumes:

**container lifecycle ≠ volume lifecycle**

---

## 7. Named Volumes vs Anonymous Volumes

### Named volume

You explicitly provide a name:

```bash
docker run -v app-data:/data alpine
```

Advantages:

- Easy to identify.
- Easy to reuse.
- Easier to back up/manage.
- Good for application data.

### Anonymous volume

Docker creates the volume automatically:

```bash
docker run -v /data alpine
```

Anonymous volumes can be useful for temporary or image-defined storage, but named volumes are usually easier to manage when persistence matters.

---

## 8. Volumes with Databases

Databases are a common use case.

Example PostgreSQL:

```bash
docker volume create postgres-data

docker run -d \
  --name postgres-db \
  -e POSTGRES_PASSWORD=secret \
  --mount source=postgres-data,target=/var/lib/postgresql/data \
  postgres
```

The database files are stored in the volume rather than only inside the container.

If the container is recreated with the same volume, the database data can remain available.

### Important

A volume provides persistence, but it is **not automatically a complete backup strategy**.

You still need:

- backups
- restore procedures
- appropriate retention
- disaster recovery planning

---

## 9. Volumes in Docker Compose

Compose makes named volumes convenient.

Example:

```yaml
services:
  db:
    image: postgres
    environment:
      POSTGRES_PASSWORD: secret
    volumes:
      - postgres-data:/var/lib/postgresql/data

volumes:
  postgres-data:
```

The top-level `volumes` section declares the named volume.

The service mounts it into the database container.

---

## 10. Volume vs Container Filesystem

| Feature | Container writable layer | Docker volume |
|---|---|---|
| Survives container removal | No | Yes |
| Managed by Docker | Container lifecycle | Yes |
| Good for database data | No | Yes |
| Easy to reuse | No | Yes |
| Persistent application data | No | Yes |
| Independent of container lifecycle | No | Yes |

---

## 11. Volume vs Bind Mount

Both will be covered in more detail in Lesson 16.

### Volume

```bash
-v app-data:/data
```

Docker manages the storage location.

### Bind mount

```bash
-v /home/user/project:/app
```

You explicitly choose a host filesystem path.

A useful rule:

- **Volume → persistent application/container data**
- **Bind mount → direct host ↔ container file sharing**, especially useful during development

---

## 12. Read-Only Volume Mounts

A volume can be mounted read-only:

```bash
docker run --rm \
  --mount source=app-data,target=/data,readonly \
  alpine
```

Short form:

```bash
docker run --rm \
  -v app-data:/data:ro \
  alpine
```

Read-only mounts reduce the chance that a container accidentally modifies important data.

---

## 13. Mounting Over Existing Container Data

Suppose the image contains:

```
/app/data/default.txt
```

and you mount a volume at:

```
/app/data
```

The mounted volume becomes the filesystem visible at that location.

Therefore, mounting storage can hide files that existed in the underlying image at that path.

This is an important debugging concept.

---

## 14. Common Mistakes

### Mistake 1 — Storing database data only in the container

Bad:

```
docker run postgres
```

without understanding where persistent database data lives.

Better:

Use a persistent volume and a proper backup strategy.

### Mistake 2 — Assuming volumes are backups

A volume protects against container deletion, but not every failure.

### Mistake 3 — Deleting volumes carelessly

Commands such as:

```bash
docker volume prune
```

can remove unused volumes.

### Mistake 4 — Using bind mounts when a Docker-managed volume is more appropriate

For production application data, a named volume is often simpler and less host-path dependent.

### Mistake 5 — Confusing volume persistence with database replication

A volume keeps filesystem data available. It does not automatically provide high availability, replication, or disaster recovery.

---

## 15. Practical Mental Model

Think of Docker storage in three layers:

```
Image
  ↓
Container writable layer
  ↓
Temporary runtime changes

Volume
  ↓
Persistent application data
```

The important separation is:

```
Container = replaceable
Volume    = persistent
```

This is one of the most important Docker concepts for real applications.

---

## 16. Interview Questions

### Q1. What is a Docker volume?

A Docker volume is Docker-managed persistent storage that exists independently of a container's writable layer.

### Q2. Why are volumes needed?

Because the container writable layer is tied to the container lifecycle. Important data should survive container recreation.

### Q3. Does deleting a container delete its named volume?

No. A named volume normally remains until it is explicitly removed.

### Q4. Are Docker volumes backups?

No. Volumes provide persistent storage, not a complete backup/disaster-recovery solution.

### Q5. What is the difference between a volume and a bind mount?

A volume is managed by Docker, while a bind mount maps a specific host filesystem path into the container.

### Q6. Can multiple containers use the same volume?

Yes, provided the application and access pattern support sharing the same data safely.

### Q7. How do you list Docker volumes?

```bash
docker volume ls
```

### Q8. How do you inspect a volume?

```bash
docker volume inspect <volume>
```

### Q9. What does `--mount` do?

It explicitly defines a mount source, target, and options for attaching storage to a container.

---

## 17. Quick Revision

- Container writable storage is temporary.
- Important data should not depend on a container's writable layer.
- Docker volumes provide persistent Docker-managed storage.
- Named volumes are easy to identify and reuse.
- `docker volume create` creates a volume.
- `docker volume ls` lists volumes.
- `docker volume inspect` shows volume details.
- `docker volume rm` removes a volume.
- `docker volume prune` removes unused volumes.
- `--mount` and `-v` can mount volumes.
- Volumes survive container deletion.
- A volume is not the same as a backup.
- Databases commonly use persistent volumes.
- Docker Compose can declare named volumes.
- Read-only mounts can protect data.
- Bind mounts and volumes solve different problems.

## Must Remember

> **Containers are replaceable; persistent data should live outside the container writable layer.**

> **Docker volume persistence protects data from container replacement, but you still need backups for real production systems.**
