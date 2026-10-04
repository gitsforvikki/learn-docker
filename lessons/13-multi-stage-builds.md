# Lesson 13 — Multi-Stage Builds

## 1. What Is a Multi-Stage Build?

A multi-stage Docker build uses multiple FROM instructions in one Dockerfile. Each FROM starts a separate build stage.

The common production pattern is:

Build environment → Build artifact → Minimal runtime environment

The builder stage contains everything required to build the application. The final stage contains only what is required to run it.

## 2. Why Use Multi-Stage Builds?

A single-stage image can contain:

- source code
- development dependencies
- compilers and build tools
- package-manager caches
- temporary build files

This can make the production image larger and expose unnecessary components.

Multi-stage builds help create:

- smaller images
- faster image transfer and deployment
- cleaner production images
- fewer unnecessary runtime components
- better security posture

## 3. Basic Structure

    # Build stage
    FROM node:22-alpine AS builder

    WORKDIR /app
    COPY package*.json ./
    RUN npm ci
    COPY . .
    RUN npm run build

    # Runtime stage
    FROM nginx:alpine

    COPY --from=builder /app/dist /usr/share/nginx/html

    EXPOSE 80
    CMD ["nginx", "-g", "daemon off;"]

The important concepts are:

- FROM node:22-alpine AS builder creates a named build stage.
- FROM nginx:alpine starts a new runtime stage.
- COPY --from=builder copies only the required files from the builder.
- The final image is based on the last stage.

## 4. Build Stage vs Runtime Stage

| Build Stage | Runtime Stage |
|---|---|
| Contains build tools | Contains runtime requirements |
| May contain dev dependencies | Usually production dependencies only |
| Contains source code | Contains required runtime files |
| Usually larger | Usually smaller |
| Used while building | Used when the container runs |

A later stage does not automatically contain the filesystem of an earlier stage.

## 5. Named Stages

Use AS to name a stage:

    FROM node:22-alpine AS builder

Then reference it:

    COPY --from=builder /app/dist /usr/share/nginx/html

Named stages are preferred over numeric stage references because they are easier to understand and maintain.

## 6. React Production Example

For a Vite React application, a common production Dockerfile is:

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

Build:

    docker build -t my-react-app .

Run:

    docker run -d --name react-app -p 8080:80 my-react-app

Here the host uses port 8080 while Nginx listens on port 80 inside the container.

For Create React App, the build directory is commonly build instead of dist. Always verify the actual output directory of the project.

## 7. Node.js Production Example

Multi-stage builds can also be used for Node.js applications:

    FROM node:22-alpine AS builder

    WORKDIR /app
    COPY package*.json ./
    RUN npm ci
    COPY . .
    RUN npm run build

    FROM node:22-alpine

    WORKDIR /app
    COPY package*.json ./
    RUN npm ci --omit=dev
    COPY --from=builder /app/dist ./dist

    ENV NODE_ENV=production
    USER node
    EXPOSE 3000
    CMD ["node", "dist/server.js"]

The exact build output depends on the framework and project configuration.

## 8. More Than Two Stages

A Dockerfile can contain several stages.

Example:

    FROM node:22-alpine AS dependencies
    WORKDIR /app
    COPY package*.json ./
    RUN npm ci

    FROM dependencies AS builder
    COPY . .
    RUN npm run build

    FROM nginx:alpine AS production
    COPY --from=builder /app/dist /usr/share/nginx/html
    CMD ["nginx", "-g", "daemon off;"]

Common stages include:

- dependencies
- development
- testing
- builder
- production

## 9. Copying Between Stages

General syntax:

    COPY --from=<stage> <source> <destination>

Example:

    COPY --from=builder /app/dist /usr/share/nginx/html

You can technically reference a stage by number, such as:

    COPY --from=0 /app/dist /usr/share/nginx/html

Named stages are recommended because changing the order of stages will not make the reference confusing or fragile.

## 10. Stage Isolation

Consider:

    FROM node:22-alpine AS builder
    WORKDIR /app
    COPY . .
    RUN npm run build

    FROM nginx:alpine

The Nginx stage does not automatically contain /app.

You must explicitly copy what is required:

    COPY --from=builder /app/dist /usr/share/nginx/html

This isolation is fundamental to multi-stage builds.

## 11. Image Size

The builder may contain hundreds of megabytes of dependencies and build tools, while the final runtime image may need only the generated application files.

Inspect images with:

    docker images

Inspect detailed metadata with:

    docker image inspect my-react-app

The goal is not simply the smallest possible image. The final image must still contain everything required to run correctly.

## 12. Security Benefits

Multi-stage builds can reduce unnecessary components in the production image.

The final image should generally not contain:

- compilers
- test tools
- unnecessary development dependencies
- package-manager caches
- build-only files

However, multi-stage builds are not a complete security solution. You should still use:

- minimal base images
- non-root users where practical
- updated dependencies
- vulnerability scanning
- .dockerignore
- proper secret management

## 13. Secrets

Never hard-code secrets into Dockerfiles.

Avoid patterns such as:

    ENV DATABASE_PASSWORD=secret123

and:

    ARG API_KEY=secret123

Use appropriate build-secret or runtime-secret mechanisms instead.

A multi-stage build does not make hard-coded secrets safe.

## 14. Building a Specific Stage

Use --target to build a particular stage:

    docker build --target builder -t my-app-builder .

This is useful for:

- debugging
- development
- testing
- CI pipelines

For example, a Dockerfile may contain development, builder, and production stages, while CI chooses the required target.

## 15. Single-Stage vs Multi-Stage

Single-stage:

    Source
      ↓
    Build dependencies
      ↓
    Build application
      ↓
    Same image runs application

Multi-stage:

    Source
      ↓
    Builder stage
      ↓
    Build application
      ↓
    Required artifacts
      ↓
    Minimal runtime stage
      ↓
    Production container

Multi-stage builds are normally preferred for production applications when the build environment differs from the runtime environment.

## 16. Common Mistakes

### Mistake 1 — Forgetting COPY --from

Starting a runtime stage is not enough. You must copy the build output.

### Mistake 2 — Wrong build directory

Vite commonly produces dist, while Create React App commonly produces build. Next.js has its own production output patterns.

### Mistake 3 — Installing development dependencies in production

For Node.js production images, npm ci --omit=dev is commonly appropriate when dev dependencies are not needed at runtime.

### Mistake 4 — Copying the entire builder filesystem

Avoid copying the complete builder environment when only a small artifact is required.

Prefer:

    COPY --from=builder /app/dist /usr/share/nginx/html

### Mistake 5 — Making the image small at the cost of correctness

Do not remove files or dependencies that the application actually needs at runtime.

## 17. Best Practices

1. Separate build and runtime environments.
2. Use named stages.
3. Copy only required artifacts.
4. Use a minimal appropriate runtime image.
5. Keep build dependencies out of production.
6. Use .dockerignore.
7. Use production dependencies only when appropriate.
8. Run as a non-root user where practical.
9. Never hard-code secrets.
10. Test the final runtime image independently.
11. Inspect the final image size and contents.
12. Keep base images and dependencies maintained.

## 18. CI/CD Flow

A typical production pipeline is:

    Developer
       ↓
    Git push
       ↓
    CI pipeline
       ↓
    Docker build
       ↓
    Builder stage
       ↓
    Application artifact
       ↓
    Production stage
       ↓
    Final Docker image
       ↓
    Container registry
       ↓
    Deployment

For CodeBuddy, the same pattern can later be used with Jenkins, Docker, a registry, and Kubernetes.

## 19. Interview Questions

### Q1. What is a multi-stage Docker build?

A technique that uses multiple build stages to separate the build environment from the production runtime environment, allowing only required artifacts to be included in the final image.

### Q2. Why use multi-stage builds?

Main reasons are smaller images, fewer unnecessary runtime dependencies, faster transfers, cleaner production images, and a better security posture.

### Q3. What does COPY --from do?

It copies files from the filesystem of another build stage into the current stage.

### Q4. Does the final stage automatically contain previous stages?

No. Stages are isolated. Required files must be explicitly copied.

### Q5. Can a Dockerfile have more than two stages?

Yes. It can have as many useful stages as the build requires.

### Q6. What does AS builder do?

It gives a stage a name so later instructions can reference it.

### Q7. What is --target used for?

It tells Docker to build a specific stage rather than the final stage.

### Q8. Is multi-stage building only for frontend applications?

No. It is useful for frontend applications, Node.js, Go, Java, Rust, and many other technologies.

## 20. Quick Revision

    Multi-stage build
          ↓
    Multiple FROM instructions
          ↓
    Separate build and runtime environments
          ↓
    Build application
          ↓
    Copy required artifacts
          ↓
    Minimal runtime image
          ↓
    Production container

### Must Remember

- FROM ... AS builder creates a named stage.
- COPY --from=builder copies files from another stage.
- Later stages do not automatically inherit earlier filesystem contents.
- The last stage normally becomes the final image.
- --target can build a specific stage.
- Build dependencies should stay out of the production image.
- Multi-stage builds are a major production Docker pattern.

## One-Line Interview Answer

Multi-stage Docker builds separate the build environment from the production runtime environment, allowing only required application artifacts to be copied into a smaller and cleaner final image.
