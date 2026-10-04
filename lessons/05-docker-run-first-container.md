# Lesson 05 — Running Your First Container

## 1. What is `docker run`?

`docker run` is the main command used to create and start a **new container from an image**.

A useful mental model is:

```text
Image
  ↓
docker run
  ↓
Create container
  ↓
Start container
  ↓
Running application
```

If the image is not available locally, Docker pulls it from a configured registry first.

```bash
docker run nginx
```

### Important theory

An **image is a reusable template**, while a **container is a running instance created from that image**.

Every time you run:

```bash
docker run nginx
```

Docker creates a **new container**. It does not restart an existing stopped container.

---

## 2. Detached Mode — `-d`

Normally, Docker can attach your terminal to the container's output.

For server applications, you usually want the application to keep running in the background:

```bash
docker run -d nginx
```

`-d` means **detached mode**.

The terminal is returned to you while the container continues running in the background.

Check it with:

```bash
docker ps
```

---

## 3. Giving a Container a Name

Docker can generate a random container name, but predictable names are easier to manage.

```bash
docker run -d --name my-nginx nginx
```

Now you can use:

```bash
docker logs my-nginx
docker stop my-nginx
docker start my-nginx
```

### Important

Container names must be unique. You cannot create another container with the same name until the existing one is removed or renamed.

---

## 4. Publishing Ports — `-p`

A process running inside a container is isolated from the host's network by default.

If Nginx listens on port `80` **inside the container**, that does not automatically mean you can access it through port `80` on your host.

Use:

```bash
docker run -d --name my-nginx -p 8080:80 nginx
```

The format is:

```text
-p HOST_PORT:CONTAINER_PORT
```

Flow:

```text
Browser
   ↓
localhost:8080
   ↓
Host port 8080
   ↓
Container port 80
   ↓
Nginx
```

So:

- `8080` → host port
- `80` → container port

You can choose a different host port while the application continues listening on its original container port.

---

## 5. `EXPOSE` vs `-p`

These are commonly confused.

### `EXPOSE`

`EXPOSE 80` in a Dockerfile documents that the application expects to use port `80`.

It does **not** make the port accessible from the host.

### `-p`

```bash
-p 8080:80
```

Actually publishes/maps the container port to a host port.

> **Interview point:** `EXPOSE` documents a port; `-p` publishes a port.

---

## 6. Container Lifecycle

A typical flow is:

```text
Image
  ↓
docker run
  ↓
Created + Running
  ↓
docker stop
  ↓
Stopped
  ↓
docker start
  ↓
Running again
  ↓
docker rm
  ↓
Removed
```

Useful commands:

```bash
docker ps
docker ps -a
docker stop my-nginx
docker start my-nginx
docker restart my-nginx
docker rm my-nginx
```

### `docker run` vs `docker start`

This distinction is important:

- `docker run` → creates **and** starts a new container.
- `docker start` → starts an **existing stopped** container.

---

## 7. Why Can a Container Exit Immediately?

A container is not a virtual machine that automatically stays alive.

A container keeps running while its **main process (PID 1)** is running.

For example:

```bash
docker run ubuntu
```

Ubuntu's default command can finish immediately, so the main process exits and the container stops.

Check the stopped container with:

```bash
docker ps -a
```

### Key concept

> **Container lifetime is tied to its main process.**

This is one of the most important Docker fundamentals.

---

## 8. Interactive Containers — `-it`

For command-line interaction:

```bash
docker run -it ubuntu bash
```

- `-i` → keeps standard input open
- `-t` → allocates a terminal (TTY)

Together, `-it` gives you an interactive terminal inside the container.

---

## 9. Automatically Remove Temporary Containers — `--rm`

For short-lived or testing containers:

```bash
docker run --rm ubuntu echo "Hello"
```

After the main process exits, Docker automatically removes the container.

Without `--rm`, the stopped container remains and can be viewed with:

```bash
docker ps -a
```

---

## 10. Environment Variables — `-e`

You can pass configuration into a container when starting it:

```bash
docker run -e NODE_ENV=production my-api
```

Inside the container, the application can read `NODE_ENV`.

This is commonly used for configuration such as environment mode, API URLs, feature flags, and database settings.

For sensitive values, prefer proper secret-management mechanisms instead of exposing secrets unnecessarily in commands or images.

---

## 11. Important `docker run` Options

| Option | Purpose |
|---|---|
| `-d` | Run in detached/background mode |
| `--name` | Assign a custom container name |
| `-p HOST:CONTAINER` | Publish a container port |
| `-e KEY=VALUE` | Set an environment variable |
| `--rm` | Automatically remove container after exit |
| `-it` | Interactive terminal |

---

## Interview Questions

### What does `docker run` do?

**Answer:** It creates and starts a new container from an image. If the image is unavailable locally, Docker pulls it first.

### Does `docker run` restart an existing container?

**Answer:** No. It creates a new container. Use `docker start` to start an existing stopped container.

### What does `-d` mean?

**Answer:** Detached mode. The container runs in the background.

### What does `-p 8080:80` mean?

**Answer:** It publishes host port `8080` to container port `80`.

### Why can `docker run ubuntu` exit immediately?

**Answer:** A container runs while its main process is running. When that process exits, the container stops.

### What is the difference between `EXPOSE` and `-p`?

**Answer:** `EXPOSE` documents the intended container port; `-p` actually publishes the port to the host.

### What is `-it` used for?

**Answer:** It provides an interactive terminal, commonly used for shell access to containers.

---

## ⭐ Quick Revision

```text
docker run IMAGE
      ↓
Check local image
      ↓
Pull if needed
      ↓
Create container
      ↓
Start main process
      ↓
Container runs while main process runs
```

- Image = reusable template
- Container = runtime instance of an image
- `docker run` = create + start a new container
- `docker start` = start an existing stopped container
- `-d` = detached mode
- `--name` = predictable container name
- `-p HOST:CONTAINER` = publish port
- `EXPOSE` ≠ port publishing
- `-it` = interactive terminal
- `--rm` = remove container after exit
- Container lifetime depends on its main process

## 🎯 Interview Summary

The most important idea in this lesson is understanding what happens when `docker run` is executed:

**Docker uses an image to create a new container, starts its main process, and the container remains running only while that main process is running.**

