# Lesson 14 — Development vs Production Dockerfiles

## 1. Why Use Different Docker Configurations?

Development and production have different requirements.

### Development
The goal is:

- fast development
- hot reload
- debugging
- source-code mounting
- development dependencies
- easy iteration

### Production
The goal is:

- small image
- predictable builds
- security
- production dependencies only
- no unnecessary development tools
- reliable startup
- reproducible deployment

Trying to use exactly the same Docker setup for both can make development inconvenient or production unnecessarily large.

## 2. Development vs Production

| Development | Production |
|---|---|
| Hot reload | Stable application process |
| Source mounted as volume | Application copied into image |
| Dev dependencies | Production dependencies |
| Debug tools | Minimal runtime |
| Frequent rebuilds | Reproducible image |
| Easy debugging | Security and reliability |
| Often bind mounts | Immutable image |

The important principle is:

> Development containers optimize the developer experience; production images optimize reliability, security, and efficiency.

## 3. Development Dockerfile Example

For a Node.js application:

    FROM node:22-alpine

    WORKDIR /app

    COPY package*.json ./
    RUN npm install

    COPY . .

    EXPOSE 3000

    CMD ["npm", "run", "dev"]

This image is intended for development.

It may contain:

- nodemon
- TypeScript tooling
- test dependencies
- linters
- other development packages

That is acceptable when the image is used only for development.

## 4. Development with Bind Mounts

A common development workflow is:

    docker run --rm -it \
      -p 3000:3000 \
      -v "$PWD:/app" \
      -v /app/node_modules \
      my-node-dev

The bind mount:

    -v "$PWD:/app"

maps the host project directory into the container.

Now source-code changes made on the host can be visible inside the container.

The anonymous volume:

    -v /app/node_modules

helps prevent the host's node_modules from replacing the container's Linux-compatible dependencies.

## 5. Why Hot Reload Matters

Without hot reload, a developer may need to:

    edit code
      ↓
    rebuild image
      ↓
    stop container
      ↓
    start container
      ↓
    test again

With a development setup:

    edit code
      ↓
    file change detected
      ↓
    development server reloads
      ↓
    test immediately

Tools such as nodemon, Vite, and Next.js development mode are designed for this workflow.

## 6. Production Dockerfile

A production Node.js image should avoid unnecessary development dependencies.

Example:

    FROM node:22-alpine

    WORKDIR /app

    COPY package*.json ./
    RUN npm ci --omit=dev

    COPY . .

    ENV NODE_ENV=production

    USER node

    EXPOSE 3000

    CMD ["node", "server.js"]

Important characteristics:

- deterministic dependency installation
- production dependencies only
- production environment
- non-root user
- no development server
- application packaged inside the image

The exact Dockerfile depends on the application.

## 7. npm install vs npm ci

For production builds, npm ci is usually preferred when package-lock.json is committed.

Development:

    npm install

Production:

    npm ci --omit=dev

Why?

npm ci is designed for clean, reproducible installation from the lockfile and is commonly used in CI/CD and production image builds.

## 8. Development Should Usually Not Copy node_modules

Use a .dockerignore file:

    node_modules
    .next
    dist
    build
    coverage
    .git
    .env*
    npm-debug.log*

This prevents unnecessary host files from entering the Docker build context.

It also avoids accidentally copying host-installed dependencies into the image.

## 9. Environment Variables

Development often uses local configuration:

    NODE_ENV=development
    API_URL=http://localhost:3000

Production should use production configuration:

    NODE_ENV=production
    API_URL=https://api.example.com

Do not put secrets directly into a Dockerfile.

For example, avoid:

    ENV DATABASE_PASSWORD=supersecret

Instead, provide sensitive values through appropriate runtime secret/environment mechanisms.

## 10. Port Differences

A common mistake is confusing host and container ports.

Example:

    docker run -p 8080:3000 my-app

Means:

    host:8080 → container:3000

The application inside the container still listens on port 3000.

The host port can be different between development and production without changing the application port.

## 11. Development vs Production Dependencies

Development dependencies can include:

- nodemon
- Jest
- Vitest
- ESLint
- TypeScript tooling
- testing utilities

Production usually needs only what the application requires to execute.

For Node.js:

    npm ci --omit=dev

helps keep development-only dependencies out of the runtime image.

Important: some frameworks or applications require packages normally classified as development dependencies at runtime. Always verify the application's actual requirements rather than blindly removing packages.

## 12. Multi-Stage Production Dockerfile

Development and production can also be handled with multi-stage builds.

Example:

    FROM node:22-alpine AS builder

    WORKDIR /app

    COPY package*.json ./
    RUN npm ci

    COPY . .
    RUN npm run build

    FROM node:22-alpine AS production

    WORKDIR /app

    COPY package*.json ./
    RUN npm ci --omit=dev

    COPY --from=builder /app/dist ./dist

    ENV NODE_ENV=production

    USER node

    EXPOSE 3000

    CMD ["node", "dist/server.js"]

This separates:

    build environment
          ↓
    production runtime

and keeps build-only files out of the final image.

## 13. React Production Setup

A React SPA normally needs a build step and a static web server.

Typical flow:

    React source
       ↓
    Node builder
       ↓
    npm run build
       ↓
    dist/
       ↓
    Nginx runtime

Example:

    FROM node:22-alpine AS builder

    WORKDIR /app
    COPY package*.json ./
    RUN npm ci
    COPY . .
    RUN npm run build

    FROM nginx:alpine

    COPY --from=builder /app/dist /usr/share/nginx/html

    EXPOSE 80

    CMD ["nginx", "-g", "daemon off;"]

The final production image does not need Node.js if the React application is purely static.

## 14. Next.js Production Setup

Next.js can require a Node.js runtime depending on how the application is deployed.

A common production approach is to use standalone output.

In next.config.ts:

    const nextConfig = {
      output: "standalone",
    };

Then a production image can copy the standalone output and required static assets.

The exact Dockerfile depends on the Next.js application and whether it uses:

- server rendering
- static generation
- API/route handlers
- image optimization
- standalone output

The key principle remains:

> Build once, then run only what production requires.

## 15. One Dockerfile vs Separate Dockerfiles

There are several valid strategies.

### Strategy A — Separate Dockerfiles

Example:

    Dockerfile.dev
    Dockerfile

Build development:

    docker build -f Dockerfile.dev -t my-app:dev .

Build production:

    docker build -f Dockerfile -t my-app:prod .

This is simple and easy to understand.

### Strategy B — One Multi-Stage Dockerfile

Example stages:

    development
    builder
    production

Then select a stage with:

    docker build --target development -t my-app:dev .

or:

    docker build --target production -t my-app:prod .

This can reduce duplication.

### Strategy C — One Production Dockerfile + Compose for Development

A common team approach is:

- Dockerfile for production
- docker-compose.yml / compose.yaml for development
- bind mounts
- development command
- local environment variables

The best choice depends on project complexity and team workflow.

## 16. Development with Docker Compose

A typical development service may look like:

    services:
      app:
        build:
          context: .
          dockerfile: Dockerfile.dev
        ports:
          - "3000:3000"
        volumes:
          - .:/app
          - /app/node_modules
        environment:
          NODE_ENV: development
        command: npm run dev

This gives developers:

- source-code synchronization
- hot reload
- isolated dependencies
- consistent runtime environment

Production Compose configuration should normally avoid development bind mounts and development commands.

## 17. Common Mistakes

### Mistake 1 — Running npm run dev in production

Development servers are intended for development.

Use the application's production build/start process.

### Mistake 2 — Mounting source code in production

Bind mounts can make the deployed container dependent on the host filesystem.

Production images should normally contain the application itself.

### Mistake 3 — Installing all development dependencies in production

This increases image size and adds unnecessary components.

### Mistake 4 — Copying node_modules from the host

Host dependencies may be incompatible with the container OS or architecture.

### Mistake 5 — Putting secrets in Dockerfiles

Dockerfiles can become part of source control and image history.

Use runtime configuration and proper secret-management mechanisms.

### Mistake 6 — Using localhost for container-to-container communication

Inside a container, localhost refers to that same container.

For example, if an API container connects to MongoDB in another Compose service, use the service name:

    mongodb://mongo:27017/mydb

not:

    mongodb://localhost:27017/mydb

### Mistake 7 — Forgetting application binding

For servers that need to be reachable outside the container, listen on:

    0.0.0.0

rather than only:

    localhost

## 18. Best Practices

### Development

- optimize for fast feedback
- use hot reload
- use bind mounts carefully
- keep development dependencies available
- use Compose when multiple services are needed
- do not store secrets in images

### Production

- build a production image
- use npm ci when appropriate
- install production dependencies only
- use multi-stage builds when useful
- run as non-root where practical
- avoid source bind mounts
- use explicit production commands
- keep images minimal
- scan and update dependencies
- test the final image

## 19. Typical Full-Stack Workflow

For a React/Next.js frontend and Node.js backend:

Development:

    Source code
        ↓
    Docker Compose
        ↓
    Frontend dev server
        +
    Backend dev server
        +
    Database
        ↓
    Hot reload

Production:

    Git push
        ↓
    CI/CD
        ↓
    Build frontend image
        +
    Build backend image
        ↓
    Test
        ↓
    Push images to registry
        ↓
    Deploy
        ↓
    Production containers

This is the foundation for the Jenkins and Kubernetes workflows you will learn later.

## 20. Interview Questions

### Q1. What is the main difference between development and production Dockerfiles?

Development Dockerfiles optimize for debugging, hot reload, and fast iteration. Production Dockerfiles optimize for small, secure, reproducible, and reliable runtime images.

### Q2. Why should we not use npm run dev in production?

Development servers are intended for development and usually include unnecessary tooling and less production-oriented behavior.

### Q3. Why use npm ci in production builds?

It installs dependencies from the lockfile in a clean and reproducible way, making it suitable for CI/CD and production image builds.

### Q4. Why are bind mounts common in development but avoided in production?

They make source-code changes immediately available during development, but production should normally run from an immutable image containing the application.

### Q5. Can the same Dockerfile support both development and production?

Yes. Multi-stage Dockerfiles can define development, builder, and production stages, and --target can select a stage.

### Q6. Why use Nginx for a React SPA?

After building, a React SPA consists of static files. Nginx can efficiently serve those files without requiring Node.js at runtime.

### Q7. What should the final production image contain?

Only the runtime, application artifacts, production dependencies, configuration needed to run, and other files genuinely required by the application.

## 21. Quick Revision

    DEVELOPMENT
    ────────────
    Source mount
        ↓
    Dev dependencies
        ↓
    Hot reload
        ↓
    Fast feedback

    PRODUCTION
    ──────────
    Build
        ↓
    Production dependencies
        ↓
    Minimal runtime
        ↓
    Immutable image
        ↓
    Deployment

### Must Remember

- Development optimizes for developer experience.
- Production optimizes for reliability, security, and efficiency.
- Do not use development servers for production.
- Use production dependencies where appropriate.
- Bind mounts are mainly a development technique.
- Do not copy host node_modules into images.
- Never hard-code secrets.
- Container-to-container communication uses service/container names, not localhost.
- Multi-stage builds are useful for separating build and runtime environments.
- The final production image should contain only what it actually needs.

## One-Line Interview Answer

Development Docker setups optimize for fast iteration and debugging, while production Docker images optimize for reproducibility, security, minimal size, and reliable runtime behavior.
