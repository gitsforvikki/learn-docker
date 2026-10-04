# Lesson 07 — Dockerfile Fundamentals

## 1. What Is a Dockerfile?

A **Dockerfile** is a text file containing instructions that Docker uses to build a Docker image.

It defines:

- The base image
- Application files
- Dependencies
- Environment variables
- Working directory
- Network ports
- Default startup command

Basic flow:

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

A Dockerfile is therefore the **recipe for creating an image**.

---

## 2. Basic Dockerfile Structure

Example:

```dockerfile
FROM node:22-alpine

WORKDIR /app

COPY package*.json ./

RUN npm ci

COPY . .

EXPOSE 3000

CMD ["npm", "start"]
```

The instructions are processed in order.

---

## 3. FROM

`FROM` specifies the base image.

```dockerfile
FROM node:22-alpine
```

Every normal Dockerfile starts with a `FROM` instruction.

Examples:

```dockerfile
FROM node:22-alpine
FROM python:3.13-slim
FROM nginx:alpine
```

### Why the base image matters

The base image provides the starting filesystem and runtime needed by the application.

A smaller, appropriate base image can reduce:

- Image size
- Build time
- Network transfer
- Attack surface

---

## 4. WORKDIR

`WORKDIR` sets the working directory for subsequent instructions.

```dockerfile
WORKDIR /app
```

After this, commands such as `RUN`, `COPY`, and `CMD` operate relative to `/app` where applicable.

Prefer `WORKDIR` over repeatedly using `cd`.

---

## 5. COPY

`COPY` copies files from the **build context** into the image.

```dockerfile
COPY package*.json ./
COPY . .
```

For example:

```text
Project directory
       ↓ COPY
Docker image filesystem
```

The source path is relative to the build context, not an arbitrary location on the host.

---

## 6. RUN

`RUN` executes a command **while the image is being built**.

Example:

```dockerfile
RUN npm ci
```

The result of the command becomes part of the image layer.

Important distinction:

```text
RUN → image build time
CMD → container runtime
```

Example:

```dockerfile
RUN npm ci
CMD ["npm", "start"]
```

Dependencies are installed while building the image; the application starts when a container runs.

---

## 7. CMD

`CMD` defines the default command to run when a container starts.

```dockerfile
CMD ["npm", "start"]
```

It does **not** execute during `docker build`.

A Dockerfile can have multiple CMD instructions, but only the **last CMD** takes effect.

CMD can also provide default arguments for an ENTRYPOINT. The detailed CMD vs ENTRYPOINT comparison is covered in Lesson 09.

---

## 8. EXPOSE

`EXPOSE` documents the port on which the application listens inside the container.

```dockerfile
EXPOSE 3000
```

Important:

**EXPOSE does not publish the port to the host.**

To make the application reachable from the host, use port publishing:

```bash
docker run -p 3000:3000 my-app
```

Think:

```text
EXPOSE → documentation/metadata
-p      → actual host-to-container port publishing
```

---

## 9. ENV

`ENV` defines environment variables in the image/container environment.

```dockerfile
ENV NODE_ENV=production
```

Multiple variables can be defined:

```dockerfile
ENV NODE_ENV=production
ENV PORT=3000
```

However, **do not put secrets such as passwords, API keys, or private tokens directly in a Dockerfile**. Values written with `ENV` can become part of the image configuration/history.

Secrets should be supplied using appropriate runtime secret/configuration mechanisms.

---

## 10. ARG

`ARG` defines a variable available during **image build time**.

```dockerfile
ARG NODE_VERSION=22
FROM node:${NODE_VERSION}-alpine
```

Build with:

```bash
docker build --build-arg NODE_VERSION=22 -t my-app .
```

### ARG vs ENV

| ARG | ENV |
|---|---|
| Build-time variable | Environment variable |
| Available while building | Available to the resulting container |
| Set with `--build-arg` | Can be supplied/overridden at runtime |
| Not intended for runtime configuration | Used for runtime configuration |

Do not use `ARG` as a secure secret store; build arguments can be exposed through image build metadata/history depending on how they are used.

---

## 11. USER

`USER` specifies the user that subsequent commands and the container's default process run as.

Example:

```dockerfile
USER node
```

Running applications as a non-root user is an important security practice when the application does not require root privileges.

---

## 12. ADD

`ADD` can copy files into the image and has additional behavior such as extracting local tar archives and supporting certain URL sources.

Example:

```dockerfile
ADD app.tar.gz /app/
```

For ordinary file copying, prefer:

```dockerfile
COPY . .
```

because its behavior is simpler and more explicit.

---

## 13. LABEL

`LABEL` adds metadata to an image.

Example:

```dockerfile
LABEL maintainer="team@example.com"
LABEL version="1.0"
```

Labels can be useful for identifying or organizing images.

---

## 14. SHELL

`SHELL` changes the default shell used by shell-form commands such as `RUN`.

It is more commonly relevant when building Windows-based images or when a specific shell is required.

For normal Linux Dockerfiles, it is usually unnecessary.

---

## 15. Docker Build Context

When you run:

```bash
docker build -t my-app .
```

the final `.` means the **current directory is the build context**.

Docker sends the build context to the builder so instructions such as `COPY` can access files from it.

Example:

```text
project/
├── Dockerfile
├── package.json
├── src/
└── node_modules/
```

If the context is `project/`, Dockerfile instructions can copy files available inside that context.

Files that should not be sent should be excluded with a `.dockerignore` file.

---

## 16. .dockerignore

A `.dockerignore` file prevents unnecessary files from being included in the build context.

Typical Node.js example:

```text
node_modules
.git
.next
.env
npm-debug.log
```

Why it matters:

- Smaller build context
- Faster builds
- Less unnecessary data sent to the builder
- Prevents accidental inclusion of files such as local environment files

A `.dockerignore` file is especially important for JavaScript/Node.js projects because `node_modules` can be very large.

---

## 17. Instruction vs Build-Time vs Runtime

| Instruction | Main purpose | Stage |
|---|---|---|
| `FROM` | Select base image | Build |
| `WORKDIR` | Set working directory | Build/Runtime metadata |
| `COPY` | Copy files | Build |
| `RUN` | Execute build command | Build |
| `ARG` | Build-time variable | Build |
| `ENV` | Environment variable | Build/Runtime |
| `EXPOSE` | Document container port | Metadata |
| `USER` | Select user | Build/Runtime |
| `CMD` | Default startup command | Runtime |
| `ENTRYPOINT` | Main executable | Runtime |
| `LABEL` | Image metadata | Build |

---

## 18. Complete Node.js Example

```dockerfile
FROM node:22-alpine

WORKDIR /app

COPY package*.json ./

RUN npm ci

COPY . .

ENV NODE_ENV=production

EXPOSE 3000

USER node

CMD ["npm", "start"]
```

Build:

```bash
docker build -t my-node-app:1.0 .
```

Run:

```bash
docker run -p 3000:3000 my-node-app:1.0
```

The important sequence is:

```text
FROM
  ↓
WORKDIR
  ↓
COPY package files
  ↓
RUN npm ci
  ↓
COPY source
  ↓
ENV / EXPOSE / USER
  ↓
CMD
```

---

## 19. Common Mistakes

### Mistake 1: Confusing RUN and CMD

```dockerfile
RUN npm start
```

would start the application during image building, which is not the normal pattern.

Use:

```dockerfile
CMD ["npm", "start"]
```

to start it when the container runs.

### Mistake 2: Assuming EXPOSE publishes ports

It does not.

Use:

```bash
docker run -p 3000:3000 my-app
```

### Mistake 3: Copying everything without .dockerignore

This can send large or sensitive files into the build context.

### Mistake 4: Putting secrets in ENV or ARG

Dockerfile instructions are not a secure secret-management mechanism.

### Mistake 5: Running unnecessarily as root

Use a non-root user when possible.

### Mistake 6: Using ADD when COPY is sufficient

Prefer `COPY` for normal file copying.

---

## 20. Interview Questions

### Q1. What is a Dockerfile?

A Dockerfile is a text file containing instructions used to build a Docker image.

### Q2. What is the difference between RUN and CMD?

`RUN` executes during image build and creates image-layer changes. `CMD` defines the default command executed when a container starts.

### Q3. Does EXPOSE publish a port?

No. `EXPOSE` documents the intended container port. Port publishing is done with `-p` or `-P`.

### Q4. What is the difference between ARG and ENV?

`ARG` is primarily for build-time variables. `ENV` defines environment variables available to the resulting image/container.

### Q5. Why use .dockerignore?

It keeps unnecessary files out of the build context, improving build efficiency and reducing the chance of accidentally including sensitive or unwanted files.

### Q6. Why is COPY generally preferred over ADD?

`COPY` has simpler and more predictable file-copy behavior. `ADD` has additional features that are often unnecessary.

### Q7. What does WORKDIR do?

It sets the working directory for subsequent Dockerfile instructions and the container's default working directory.

### Q8. Can a Dockerfile contain multiple CMD instructions?

Yes, but only the last CMD takes effect.

### Q9. Why should applications avoid running as root?

A non-root process reduces the potential impact if the application is compromised.

---

## Quick Revision

```text
Dockerfile  → Recipe for building an image
FROM        → Base image
WORKDIR     → Working directory
COPY        → Copy files into image
RUN         → Execute during build
CMD         → Default runtime command
EXPOSE      → Documents container port
ENV         → Runtime environment configuration
ARG         → Build-time variable
USER        → Run as specified user
.dockerignore → Excludes files from build context
```

### Must Remember

1. **Dockerfile = instructions for building an image.**
2. **RUN happens during build; CMD happens when the container starts.**
3. **EXPOSE does not publish a port.**
4. **Use `-p HOST:CONTAINER` to publish ports.**
5. **Use `.dockerignore` to control the build context.**
6. **Do not store secrets directly in Dockerfile instructions.**
7. **Prefer COPY for normal file copying.**
8. **Use a non-root USER when practical.**
