# Lesson 04 — Docker Installation & Basic Commands

## 1. Verify Docker

```bash
docker --version
docker version
docker info
```

- `docker --version` → CLI version
- `docker version` → client/server version information
- `docker info` → Docker environment information

## 2. Image Commands

```bash
docker pull nginx
docker image ls
docker image inspect nginx
docker history nginx
```

## 3. Container Commands

```bash
docker ps
docker ps -a
docker stop <container>
docker start <container>
docker restart <container>
docker rm <container>
```

## 4. Logs and Shell Access

```bash
docker logs <container>
docker logs -f <container>
docker exec -it <container> sh
```

`docker logs` shows the container application's stdout/stderr. `docker exec` runs a command inside a running container.

## 5. Inspect

```bash
docker inspect <container>
```

Use it to inspect configuration and runtime information such as state, ports, networks, mounts, and environment.

## 6. Networks and Volumes

```bash
docker network ls
docker network inspect <network>
docker volume ls
docker volume inspect <volume>
```

## 7. Port Publishing

```text
-p HOST_PORT:CONTAINER_PORT
```

Example:

```bash
docker run -p 5000:3000 my-api
```

`EXPOSE 3000` documents the intended container port; it does **not** publish the port. `-p` actually publishes it.

## ⭐ Quick Revision

- `docker ps` → running containers
- `docker ps -a` → all containers
- `docker logs` → container stdout/stderr
- `docker exec` → execute inside a running container
- `docker inspect` → detailed configuration/state
- `-p HOST:CONTAINER` → publish a container port
- `EXPOSE` ≠ port publishing
