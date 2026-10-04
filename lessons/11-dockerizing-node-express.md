# Lesson 11 — Dockerizing Node.js + Express

## 1. What Does Dockerizing an Application Mean?

Dockerizing an application means packaging the application and its runtime dependencies into a Docker image so it can run consistently in a container.

For a Node.js + Express API:

```text
Node.js + Express Application
          ↓
      Dockerfile
          ↓
      docker build
          ↓
      Docker Image
          ↓
       docker run
          ↓
   Express Container
```

The goal is not simply to "put the code in Docker." The container should contain everything required to run the application while keeping the image reproducible, efficient, and suitable for deployment.

---

## 2. Example Project Structure

A typical Express API may look like:

```text
backend/
├── src/
│   └── server.js
├── package.json
├── package-lock.json
├── Dockerfile
└── .dockerignore
```

Example `package.json`:

```json
{
  "scripts": {
    "start": "node src/server.js"
  }
}
```

Example Express server:

```js
const express = require("express");

const app = express();
const PORT = process.env.PORT || 3000;

app.get("/health", (req, res) => {
  res.json({ status: "ok" });
});

app.listen(PORT, "0.0.0.0", () => {
  console.log(`API running on port ${PORT}`);
});
```

### Important: listen on 0.0.0.0

Inside a container, the server should normally listen on:

```text
0.0.0.0
```

rather than only:

```text
localhost / 127.0.0.1
```

Listening only on localhost can make the application inaccessible through Docker's published port.

---

## 3. Create the Dockerfile

A basic production-oriented Dockerfile:

```dockerfile
FROM node:22-alpine

WORKDIR /app

COPY package*.json ./

RUN npm ci --omit=dev

COPY . .

ENV NODE_ENV=production

EXPOSE 3000

USER node

CMD ["npm", "start"]
```

### What each instruction does

- `FROM` → Node.js runtime
- `WORKDIR` → application directory
- `COPY package*.json` → dependency manifests
- `RUN npm ci --omit=dev` → install production dependencies
- `COPY . .` → application source
- `ENV` → production environment
- `EXPOSE` → documents the container port
- `USER node` → run without root privileges
- `CMD` → starts Express

If the application requires development dependencies to build something, use a multi-stage approach rather than removing dependencies blindly.

---

## 4. Create .dockerignore

For a Node.js backend:

```text
node_modules
.git
.env
npm-debug.log
coverage
Dockerfile*
.dockerignore
```

The exact list depends on the project.

### Why exclude .env?

Environment files can contain secrets such as:

- Database passwords
- JWT secrets
- API keys
- Third-party credentials

These should normally be supplied at runtime instead of being baked into the image.

---

## 5. Build the Image

From the backend project directory:

```bash
docker build -t my-api:1.0 .
```

Check the image:

```bash
docker image ls
```

Inspect it:

```bash
docker image inspect my-api:1.0
```

---

## 6. Run the Express Container

Run:

```bash
docker run -d \
  --name my-api \
  -p 3000:3000 \
  my-api:1.0
```

The port mapping is:

```text
Host 3000 → Container 3000
```

Then test:

```bash
curl http://localhost:3000/health
```

Expected response:

```json
{"status":"ok"}
```

---

## 7. Container Environment Variables

Application configuration should generally come from the environment rather than being hard-coded.

Example:

```bash
docker run -d \
  --name my-api \
  -p 3000:3000 \
  -e PORT=3000 \
  -e NODE_ENV=production \
  -e DATABASE_URL="..." \
  my-api:1.0
```

In Node.js:

```js
const databaseUrl = process.env.DATABASE_URL;
```

### Important

Do not put sensitive values directly into the Dockerfile.

For local development, an environment file can be supplied with:

```bash
docker run --env-file .env my-api:1.0
```

For production, use the deployment platform's environment/secrets mechanism rather than committing a secret-containing `.env` file to Git.

---

## 8. Container Port vs Host Port

Suppose Express listens on port 3000 inside the container.

You can publish it to another host port:

```bash
docker run -p 8080:3000 my-api:1.0
```

Now:

```text
Host:      8080
Container: 3000
```

The application still listens on 3000 inside the container.

Request:

```text
http://localhost:8080
        ↓
Host port 8080
        ↓
Container port 3000
        ↓
Express
```

---

## 9. Container Networking

Containers have their own network namespace.

When an API is connected to another container, it should normally communicate using the **container/service name and container port**, not the host's published port.

For example:

```text
API container
    ↓
database container
```

The API might connect to:

```text
mongodb://mongo:27017/app
```

where `mongo` is the Docker network/service name.

This becomes especially important with Docker Compose and is covered further in the networking and Compose lessons.

---

## 10. Development vs Production

A development container often needs:

- Source-code mounting
- Development dependencies
- Automatic reload
- Debugging tools

A production container should generally:

- Contain only required runtime dependencies
- Use a stable image
- Avoid unnecessary source/build tools
- Run as a non-root user
- Receive configuration from the environment
- Have predictable startup behavior

Do not automatically use the same Dockerfile workflow for both development and production.

---

## 11. Development Example with Nodemon

A development image might use:

```dockerfile
FROM node:22-alpine

WORKDIR /app

COPY package*.json ./
RUN npm install

COPY . .

EXPOSE 3000

CMD ["npm", "run", "dev"]
```

A development workflow can mount the source directory so code changes are immediately visible inside the container.

The exact bind-mount setup is covered later with Docker volumes and Compose.

---

## 12. Debugging a Node.js Container

### Check running containers

```bash
docker ps
```

### Check all containers

```bash
docker ps -a
```

### Read application logs

```bash
docker logs my-api
```

Follow logs:

```bash
docker logs -f my-api
```

### Execute a shell

```bash
docker exec -it my-api sh
```

### Inspect configuration

```bash
docker inspect my-api
```

### Check processes

```bash
docker top my-api
```

These commands help distinguish application errors from Docker configuration or networking problems.

---

## 13. Common Problems

### Problem 1: Container starts and immediately exits

Check:

```bash
docker ps -a
docker logs my-api
```

Usually the main application process has exited or failed.

### Problem 2: API works inside the container but not from the host

Check:

- Express is listening on `0.0.0.0`
- Correct `-p HOST:CONTAINER` mapping
- Correct application port
- Container is running

### Problem 3: Environment variable is undefined

Check how it was supplied:

```bash
docker run -e KEY=value ...
```

or:

```bash
docker run --env-file .env ...
```

Also verify the variable name matches `process.env.KEY`.

### Problem 4: Database connection fails

Do not assume `localhost` refers to the host or another container.

Inside the API container:

```text
localhost → the API container itself
```

When another container provides the database, use its Docker network/service name.

### Problem 5: Permission errors

Check the selected `USER` and ownership/permissions of files copied into the image.

---

## 14. Production-Oriented Improvements

A production Node.js container should generally consider:

1. Small and appropriate base image.
2. `.dockerignore`.
3. Dependency caching.
4. Production dependencies only where appropriate.
5. Non-root user.
6. Runtime environment variables.
7. No secrets baked into the image.
8. Health checking.
9. Predictable image tags/digests.
10. Minimal runtime contents.

A later lesson covers Docker health checks in more depth.

---

## 15. Complete Example

### Dockerfile

```dockerfile
FROM node:22-alpine

WORKDIR /app

COPY package*.json ./
RUN npm ci --omit=dev

COPY . .

ENV NODE_ENV=production

EXPOSE 3000

USER node

CMD ["npm", "start"]
```

### .dockerignore

```text
node_modules
.git
.env
coverage
npm-debug.log
```

### Build

```bash
docker build -t my-api:1.0 .
```

### Run

```bash
docker run -d \
  --name my-api \
  -p 3000:3000 \
  -e NODE_ENV=production \
  my-api:1.0
```

### Verify

```bash
docker ps
docker logs my-api
curl http://localhost:3000/health
```

---

## 16. Interview Questions

### Q1. How do you Dockerize a Node.js application?

Create a Dockerfile, select a suitable Node.js base image, copy dependency manifests, install dependencies, copy source code, expose/document the application port, define the startup command, build the image, and run a container.

### Q2. Why should a Node.js server listen on 0.0.0.0 inside a container?

Because the application must accept connections arriving through the container's network interface. Binding only to localhost can prevent access through Docker's published port.

### Q3. Why copy package.json before source code?

It allows Docker to cache the dependency installation layer when only application source code changes.

### Q4. Why should node_modules usually be in .dockerignore?

Host-installed dependencies can be large and platform-specific. Dependencies should normally be installed in the image's target environment.

### Q5. How do containers communicate with each other?

When connected to the same Docker network, containers can normally communicate using container/service names and the destination container's internal port.

### Q6. Why should secrets not be copied into a Node.js image?

The image can be stored, inspected, cached, and distributed. Secrets baked into it can therefore be exposed.

### Q7. Why does a container exit immediately after starting?

The container's lifecycle is tied to its main process. If the main process exits, the container stops.

### Q8. What is the difference between a container port and a host port?

The container port is where the application listens inside the container. The host port is the port published on the Docker host and mapped to the container port.

---

## Quick Revision

```text
Node.js + Express
       ↓
Dockerfile
       ↓
docker build
       ↓
Node API Image
       ↓
docker run -p
       ↓
Express Container
       ↓
Host Request
```

### Must Remember

1. **Dockerizing = packaging the application and runtime dependencies into an image.**
2. **Node/Express should normally listen on 0.0.0.0 inside a container.**
3. **`-p HOST:CONTAINER` publishes the container port to the host.**
4. **Use environment variables for runtime configuration.**
5. **Never bake secrets into the image.**
6. **Container-to-container communication normally uses Docker network/service names and internal ports.**
7. **The container stops when its main process exits.**
8. **Use `.dockerignore` and avoid copying host `node_modules`.**
9. **Production images should contain only what is required to run the API.**
