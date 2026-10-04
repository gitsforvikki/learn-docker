# Lesson 05 — Running Your First Container

## 1. `docker run`

```bash
docker run IMAGE
```

Example:

```bash
docker run nginx
```

Docker checks for the image locally, pulls it if necessary, creates a container, and starts it.

## 2. Detached Mode

```bash
docker run -d nginx
```

`-d` runs the container in the background.

## 3. Name a Container

```bash
docker run -d --name my-nginx nginx
```

`--name` gives the container a predictable name.

## 4. Port Mapping

```bash
docker run -d --name my-nginx -p 8080:80 nginx
```

```text
Host :8080
   ↓
Container :80
   ↓
Nginx
```

`-p HOST_PORT:CONTAINER_PORT` publishes a container port on the host.

## 5. Container Lifecycle

```bash
docker ps
docker logs my-nginx
docker port my-nginx
docker stop my-nginx
docker start my-nginx
docker rm my-nginx
```

`docker run` creates and starts a new container. `docker start` starts an existing stopped container.

## 6. Why Can a Container Exit Immediately?

A container stays running while its **main process** is running.

```bash
docker run ubuntu
```

This can exit immediately because the container's main process finishes.

> Container lifecycle is tied to its main process.

## 7. Important `docker run` Options

| Option | Purpose |
|---|---|
| `-d` | Detached/background mode |
| `--name` | Custom container name |
| `-p HOST:CONTAINER` | Port publishing |
| `-e` | Environment variable |
| `--rm` | Remove container after exit |
| `-it` | Interactive terminal |

Example:

```bash
docker run -it ubuntu bash
```

## Interview Questions

### What does `docker run` do?
**Answer:** It creates and starts a container from an image, pulling the image first if it is not available locally.

### What does `-d` mean?
**Answer:** Detached mode; the container runs in the background.

### What does `-p 8080:80` mean?
**Answer:** It maps host port 8080 to container port 80.

### Why can `docker run ubuntu` exit?
**Answer:** The container's main process finishes, so the container stops.

### What is `docker start` vs `docker run`?
**Answer:** `docker run` creates and starts a new container; `docker start` starts an existing stopped container.

## ⭐ Quick Revision

```text
docker run IMAGE
       ↓
find/pull image
       ↓
create container
       ↓
start container
```

- `docker run` → create + start
- `-d` → detached
- `--name` → custom name
- `-p HOST:CONTAINER` → port mapping
- `-e` → environment variable
- `--rm` → remove after exit
- `-it` → interactive terminal
- Container lifetime depends on its main process