# Lesson 24 — Docker Compose Development Workflow

## 1. What Is a Compose Development Workflow?

A Docker Compose development workflow is a repeatable way to run a multi-container application during development.

Typical flow:

```text
Write code
   ↓
Compose starts services
   ↓
Application containers run
   ↓
Source code is mounted for development
   ↓
Code changes are detected
   ↓
Application reloads/restarts
   ↓
Test/debug
   ↓
Rebuild only when dependencies or image configuration change
```

For a full-stack application:

```text
Browser
   ↓
Frontend container
   ↓
Backend container
   ↓
Database container
```

Docker Compose gives the project one reproducible development environment.

---

## 2. Development vs Production

Development and production have different priorities.

| Development | Production |
|---|---|
| Fast code changes | Small, optimized images |
| Bind mounts | Immutable images |
| Hot reload | Production server |
| Debugging tools | Minimal dependencies |
| Source code mounted | Source baked into image |
| Frequent rebuilds | Controlled deployments |
| Local database container | Managed/production database |

Do not blindly use a development Compose setup in production.

---

## 3. Typical Development Compose File

Example:

```yaml
services:
  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile.dev
    ports:
      - "3000:3000"
    volumes:
      - ./backend:/app
      - backend_node_modules:/app/node_modules
    environment:
      NODE_ENV: development
      MONGO_URI: mongodb://mongo:27017/app
    depends_on:
      mongo:
        condition: service_healthy

  mongo:
    image: mongo:8
    volumes:
      - mongo_data:/data/db
    healthcheck:
      test: ["CMD", "mongosh", "--eval", "db.adminCommand('ping')"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  backend_node_modules:
  mongo_data:
```

Important idea:

```text
./backend on host
       │
       │ bind mount
       ▼
/app inside container
```

---

## 4. Why Bind Mount Source Code?

During development, you want code changes to appear immediately inside the container.

Without a bind mount:

```text
Change code
   ↓
Rebuild image
   ↓
Restart container
```

With a bind mount:

```text
Change code
   ↓
File appears inside container
   ↓
Dev server detects change
   ↓
Hot reload/restart
```

This makes development much faster.

---

## 5. The node_modules Problem

A common Node.js development setup is:

```yaml
volumes:
  - ./backend:/app
  - backend_node_modules:/app/node_modules
```

Why?

The bind mount:

```yaml
- ./backend:/app
```

can hide the `node_modules` directory that was created inside the image.

The second mount keeps dependencies in a Docker-managed volume:

```yaml
- backend_node_modules:/app/node_modules
```

Mental model:

```text
Host source code ───────► /app
Docker volume ──────────► /app/node_modules
```

This pattern is especially useful for Node.js development.

---

## 6. Development Dockerfile

A development Dockerfile can contain development dependencies and a file-watching command.

Example:

```dockerfile
FROM node:22-alpine

WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .

EXPOSE 3000

CMD ["npm", "run", "dev"]
```

This is intentionally different from a production Dockerfile.

Production commonly uses:

```dockerfile
RUN npm ci --omit=dev
```

while development normally needs dev dependencies.

---

## 7. Start the Development Environment

Start services:

```bash
docker compose up
```

Run in background:

```bash
docker compose up -d
```

Build and start:

```bash
docker compose up --build
```

Start only selected services:

```bash
docker compose up backend mongo
```

---

## 8. Watch Logs

All services:

```bash
docker compose logs
```

Follow logs:

```bash
docker compose logs -f
```

Specific service:

```bash
docker compose logs -f backend
```

Last 100 lines:

```bash
docker compose logs --tail=100 backend
```

Logs are one of the first places to look when a service fails.

---

## 9. Check Running Services

```bash
docker compose ps
```

This shows:

- service/container status
- published ports
- container names
- health status when configured

For deeper inspection:

```bash
docker compose ps -a
docker compose top backend
```

---

## 10. Execute Commands Inside a Service

Open a shell:

```bash
docker compose exec backend sh
```

Run a command directly:

```bash
docker compose exec backend npm --version
```

For MongoDB:

```bash
docker compose exec mongo mongosh
```

Use `exec` for a running container.

If the service is not running, use:

```bash
docker compose run --rm backend sh
```

---

## 11. Rebuild When Necessary

You normally do not need to rebuild for every source-code change when using bind mounts.

Rebuild when changing things such as:

- Dockerfile
- base image
- installed dependencies
- system packages
- image-level environment/configuration

Build:

```bash
docker compose build
```

Build one service:

```bash
docker compose build backend
```

Build without cache:

```bash
docker compose build --no-cache backend
```

Then restart:

```bash
docker compose up -d
```

---

## 12. Dependency Changes

Suppose you add a package:

```bash
npm install express
```

Depending on how the development environment is configured, update the image/dependencies as needed.

A common clean workflow is:

```bash
docker compose build backend
docker compose up -d backend
```

If dependencies are stored in a named volume, remember that the volume may preserve old dependencies.

When diagnosing stale dependencies, inspect:

```bash
docker compose exec backend ls node_modules
```

---

## 13. Restart vs Rebuild

These operations are different.

### Restart

```bash
docker compose restart backend
```

Restarts the existing container.

### Recreate

```bash
docker compose up -d --force-recreate backend
```

Creates a new container from the existing image/configuration.

### Rebuild

```bash
docker compose build backend
```

Creates a new image.

### Full development reset

When the environment is badly out of sync:

```bash
docker compose down
docker compose up --build
```

Be careful with:

```bash
docker compose down -v
```

because it removes Compose-managed volumes and can delete local database data.

---

## 14. Environment Variables

Keep environment-specific configuration outside the image when possible.

Example:

```yaml
services:
  backend:
    environment:
      NODE_ENV: development
      PORT: 3000
      MONGO_URI: mongodb://mongo:27017/app
```

Or:

```yaml
services:
  backend:
    env_file:
      - .env
```

Do not commit real secrets to Git.

For development, use safe local/test credentials.

---

## 15. Frontend Development

A React/Vite development service may look like:

```yaml
services:
  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile.dev
    ports:
      - "5173:5173"
    volumes:
      - ./frontend:/app
      - frontend_node_modules:/app/node_modules
    environment:
      VITE_API_URL: http://localhost:3000
    depends_on:
      - backend

volumes:
  frontend_node_modules:
```

Important:

The browser is outside Docker.

Therefore browser-accessible URLs usually use published host ports:

```text
Browser → http://localhost:3000
```

Container-to-container communication uses service names:

```text
frontend container → http://backend:3000
backend container  → mongodb://mongo:27017
```

This distinction is critical.

---

## 16. Hot Reload

Hot reload depends on the development server and filesystem event behavior.

Typical flow:

```text
Host editor
   ↓
Bind mount
   ↓
Container filesystem
   ↓
Vite / Next.js / nodemon
   ↓
Reload or restart
```

Some environments may need polling if filesystem events are not detected correctly.

For example, tools may support settings such as:

```text
CHOKIDAR_USEPOLLING=true
```

Use polling only when necessary because it can consume more CPU.

---

## 17. Health Checks

`depends_on` controls startup ordering, but startup ordering is not the same as application readiness.

Better development setup:

```yaml
depends_on:
  mongo:
    condition: service_healthy
```

with:

```yaml
healthcheck:
  test: ["CMD", "mongosh", "--eval", "db.adminCommand('ping')"]
  interval: 10s
  timeout: 5s
  retries: 5
```

Mental model:

```text
Container started ≠ Service ready
```

Your application should still handle temporary dependency failures gracefully.

---

## 18. Useful Development Commands

### Start

```bash
docker compose up -d
```

### Stop containers

```bash
docker compose stop
```

### Start stopped containers

```bash
docker compose start
```

### Stop and remove containers/network

```bash
docker compose down
```

### Rebuild

```bash
docker compose build
```

### Rebuild and start

```bash
docker compose up --build
```

### Logs

```bash
docker compose logs -f
```

### Shell

```bash
docker compose exec backend sh
```

### List services

```bash
docker compose ps
```

---

## 19. Development Troubleshooting Workflow

When something fails, use this order:

### Step 1 — Check service state

```bash
docker compose ps
```

### Step 2 — Read logs

```bash
docker compose logs backend
```

### Step 3 — Check environment

```bash
docker compose exec backend env
```

### Step 4 — Check filesystem

```bash
docker compose exec backend ls -la
```

### Step 5 — Check dependencies

```bash
docker compose exec backend npm ls
```

### Step 6 — Check networking

```bash
docker compose exec backend getent hosts mongo
```

### Step 7 — Check database connectivity

Use the application's database client or database CLI from inside the container.

### Step 8 — Rebuild only if required

```bash
docker compose up --build
```

This avoids unnecessarily rebuilding everything.

---

## 20. Common Mistakes

### Mistake 1 — Using localhost for another container

Wrong:

```text
backend → mongodb://localhost:27017
```

Correct:

```text
backend → mongodb://mongo:27017
```

### Mistake 2 — Using container port from the host

If:

```yaml
ports:
  - "8080:3000"
```

then:

- host/browser → `localhost:8080`
- container-to-container → service-name:3000

### Mistake 3 — Rebuilding after every source change

With bind mounts and hot reload, source changes normally do not require image rebuilds.

### Mistake 4 — Running production Dockerfiles for development

Production images may not include dev dependencies or hot-reload tooling.

### Mistake 5 — Deleting volumes accidentally

Avoid:

```bash
docker compose down -v
```

unless you intentionally want to remove the local persistent data.

### Mistake 6 — Committing secrets

Never commit real passwords, API keys, tokens, or production credentials.

---

## 21. Recommended Development Architecture

For a typical full-stack application:

```text
                    ┌───────────────┐
                    │    Browser    │
                    └───────┬───────┘
                            │
                    localhost:5173
                            │
                    ┌───────▼───────┐
                    │   Frontend    │
                    │  dev server   │
                    └───────┬───────┘
                            │
                       backend:3000
                            │
                    ┌───────▼───────┐
                    │    Backend    │
                    │ Node/Express  │
                    └───────┬───────┘
                            │
                        mongo:27017
                            │
                    ┌───────▼───────┐
                    │    MongoDB    │
                    │ named volume  │
                    └───────────────┘
```

This is a strong local development architecture for a project such as CodeBuddy.

---

## 22. Development Workflow for CodeBuddy

A practical CodeBuddy workflow can be:

```text
1. Start Compose
       ↓
2. Frontend + Backend + MongoDB start
       ↓
3. Open frontend in browser
       ↓
4. Edit React code
       ↓
5. Hot reload
       ↓
6. Edit Express code
       ↓
7. Nodemon restarts backend
       ↓
8. Backend connects to mongo:27017
       ↓
9. Test API / Socket.IO features
       ↓
10. Inspect logs when needed
       ↓
11. Rebuild only after dependency/Dockerfile changes
       ↓
12. Stop with docker compose down
```

---

## 23. Best Practices

1. Keep development and production configurations separate.
2. Use bind mounts for source code during development.
3. Keep `node_modules` in a Docker volume when appropriate.
4. Use service names for container-to-container communication.
5. Use health checks for important dependencies.
6. Do not use `localhost` to reach another container.
7. Avoid unnecessary image rebuilds.
8. Keep secrets out of Git.
9. Use named volumes for local database persistence.
10. Use `docker compose logs -f` during debugging.
11. Use `docker compose exec` to inspect running containers.
12. Do not run `down -v` casually.

---

# Interview Questions

### 1. Why use Docker Compose for development?

Compose defines multiple services, networks, volumes, and configuration in one reproducible file.

### 2. Why use bind mounts during development?

They allow host source-code changes to appear inside the container without rebuilding the image.

### 3. Why mount node_modules separately?

The source bind mount can hide the image's `node_modules`; a Docker volume keeps dependencies inside the container environment.

### 4. When should you rebuild a Compose service?

When Dockerfile instructions, base images, installed dependencies, or other image-level configuration changes.

### 5. What is the difference between restart and rebuild?

Restart starts the existing container again. Rebuild creates a new image from the Dockerfile.

### 6. Why should a backend use `mongo:27017` instead of `localhost:27017`?

Because `mongo` is the Compose service hostname, while `localhost` inside the backend container refers to the backend container itself.

### 7. Does `depends_on` guarantee that a database is ready?

No. It can control startup ordering, and with health-check conditions it can wait for a reported healthy state, but applications should still handle connection failures.

### 8. Why should development and production Compose setups differ?

Development prioritizes fast iteration and debugging; production prioritizes security, immutability, reliability, and optimized images.

### 9. What does `docker compose down -v` do?

It removes Compose-created containers/networks and also removes associated named volumes declared by the Compose project. This can delete local database data.

### 10. What is the difference between `docker compose exec` and `docker compose run`?

`exec` runs a command in an existing running service container. `run` creates a new one-off container for the service.

---

# Quick Revision

```text
Compose Development
        │
        ├── compose.yaml
        │
        ├── Bind mounts → source code
        │
        ├── Named volume → node_modules / database
        │
        ├── Hot reload → fast development
        │
        ├── Service names → container DNS
        │
        ├── Health checks → readiness
        │
        ├── logs → debugging
        │
        ├── exec → inspect running container
        │
        └── rebuild → Dockerfile/dependency changes
```

### Most important commands

```bash
docker compose up -d
docker compose up --build
docker compose ps
docker compose logs -f
docker compose exec backend sh
docker compose build
docker compose restart backend
docker compose down
```

---

# ⭐ Must Remember

1. **Bind mounts are mainly for development source-code sharing.**
2. **Service names are used for container-to-container communication.**
3. **localhost inside a container means that same container.**
4. **Host/browser uses published host ports.**
5. **Container-to-container traffic uses container ports.**
6. **Source-code changes normally do not require image rebuilds with bind mounts.**
7. **Dockerfile/dependency changes usually require a rebuild.**
8. **`down -v` can delete local database data.**
9. **Container started does not always mean application ready.**
10. **Development Dockerfiles and production Dockerfiles have different goals.**
