# Lesson 03 — Containers vs Images

## Image

A Docker image is a **read-only, layered package/template** used to create containers.

## Container

A container is a **runtime instance of an image**. It has a writable layer on top of the image layers.

```text
Image (read-only)
       ↓
Container
+ writable layer
```

One image can create multiple containers.

## Important Differences

| Image | Container |
|---|---|
| Read-only template | Runtime instance |
| Used to create containers | Created from an image |
| Immutable/layered | Has a writable runtime layer |
| Stored/distributed | Has a lifecycle |

Deleting a container does **not** delete its image.

Data stored only in the container writable layer is removed when the container is removed. Persistent data should use volumes or external storage.

## Important Commands

```bash
docker images
docker ps
docker ps -a
docker run nginx
docker stop <container>
docker start <container>
docker rm <container>
docker rm -f <container>
```

- `docker images` → list local images
- `docker ps` → running containers
- `docker ps -a` → all containers
- `docker run` → create and start a container from an image
- `docker stop` → stop a running container
- `docker start` → start a stopped container
- `docker rm` → remove a container

## Interview Questions

### What is the difference between an image and a container?
**Answer:** An image is a read-only layered template; a container is a runtime instance of that image with a writable layer.

### Can multiple containers use the same image?
**Answer:** Yes. Multiple containers can be created from the same image.

### What happens to data in the container writable layer when the container is removed?
**Answer:** It is removed. Persistent data should be stored using a volume or external storage.

### Does removing a container remove its image?
**Answer:** No. Containers and images are separate Docker objects.

## ⭐ Quick Revision

**Image = read-only template. Container = runtime instance of an image.**

One image → many containers.

Container writable-layer data is **not persistent storage**.