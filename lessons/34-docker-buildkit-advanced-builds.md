# Lesson 34 — Docker BuildKit & Advanced Builds

## 1. What Is BuildKit?

BuildKit is Docker's modern build engine. It improves Docker image builds with better caching, parallel execution, efficient build context handling, secret/SSH mounts, and advanced build features.

Think: Dockerfile + build context → BuildKit → image layers

BuildKit is important for faster, safer, and more reproducible production builds.

## 2. Why BuildKit Matters

BuildKit improves builds through:
- Better layer caching
- Parallel build stages where possible
- Improved build output
- Cache import/export
- Secret mounts
- SSH mounts
- Multiple build stages and targets
- Efficient handling of build contexts

## 3. Buildx

Buildx is Docker's extended build interface built around BuildKit.

    docker buildx version
    docker buildx ls
    docker buildx create --name mybuilder --use
    docker buildx inspect
    docker buildx build -t my-app:1.0 .

## 4. Dockerfile Syntax

BuildKit enables advanced Dockerfile features.

    # syntax=docker/dockerfile:1

Keep the syntax directive current enough for the features your Dockerfile requires.

## 5. Build Cache

Docker can reuse previously built layers when relevant inputs have not changed.

Good ordering:

    COPY package*.json ./
    RUN npm ci
    COPY . .

If only application source changes, the dependency layer can often remain cached.

## 6. Cache Mounts

BuildKit can mount a persistent cache during a build step without placing the cache itself into the final image.

    RUN --mount=type=cache,target=/root/.npm npm ci

This can make repeated dependency installation faster. The cache is a build optimization, not application data.

## 7. Build Secrets

Never place credentials in Dockerfile ARG or ENV just to use them during a build.

BuildKit supports temporary secret mounts:

    RUN --mount=type=secret,id=npmrc,target=/root/.npmrc npm ci

Provide the secret:

    docker build --secret id=npmrc,src=$HOME/.npmrc -t private-app:1.0 .

The goal is to make the secret available to the build step without intentionally storing it in the resulting image.

## 8. SSH Mounts

Private Git dependencies can require SSH authentication.

BuildKit can forward an SSH agent without copying the private key into the image.

    RUN --mount=type=ssh git clone git@github.com:example/private-repo.git
    docker build --ssh default -t private-app:1.0 .

## 9. Multi-Stage Builds

Multi-stage builds separate build tools from the runtime image.

    FROM node:22-alpine AS builder
    WORKDIR /app
    COPY package*.json ./
    RUN npm ci
    COPY . .
    RUN npm run build

    FROM node:22-alpine
    WORKDIR /app
    COPY --from=builder /app/dist ./dist
    CMD ["node", "dist/server.js"]

The final image does not need all build tools. See Lesson 13 for fundamentals.

## 10. Build Targets

A Dockerfile can contain named stages that can be selected as build targets.

    FROM node:22-alpine AS development
    ...
    FROM node:22-alpine AS production
    ...

Build a specific target:

    docker build --target development -t my-app:dev .
    docker build --target production -t my-app:prod .

Useful when development and production need different build environments.

## 11. Cache Import and Export

CI systems often start with an empty local build cache. BuildKit can import and export cache data so repeated CI builds can reuse previous work.

Conceptually: CI build A → cache export → registry/cache storage → CI build B → cache import

Exact cache backends depend on the Buildx/Docker environment.

## 12. Build and Push

    docker buildx build --platform linux/amd64 -t registry.example.com/my-app:1.0 --push .

Multiple platforms:

    docker buildx build \
      --platform linux/amd64,linux/arm64 \
      -t registry.example.com/my-app:1.0 \
      --push .

Multi-platform builds are useful when the same image must run on different CPU architectures.

## 13. Local Image vs Registry Output

    docker buildx build -t my-app:1.0 .
    docker buildx build -t my-app:1.0 --load .
    docker buildx build -t registry.example.com/my-app:1.0 --push .

Use --load when a compatible result is needed in the local Docker image store. Use --push to send the result directly to a registry.

## 14. Build Context Optimization

The build context is the set of files made available to the build. Keep it small.

Example .dockerignore:

    node_modules
    .git
    .env
    coverage
    dist
    logs

A smaller context reduces transfer and accidental inclusion of unnecessary files.

## 15. Named Build Contexts

BuildKit can provide additional named build contexts.

    docker build --build-context docs=./docs .

This is useful when a build needs separate source trees or generated assets.

## 16. Reproducible Builds

Important practices:
- Lock dependencies
- Pin important base image versions
- Prefer immutable image digests for critical production builds
- Control build arguments
- Avoid downloading unpinned dependencies
- Record source/version information

BuildKit improves build mechanics, but reproducibility still depends on the Dockerfile and dependencies.

## 17. Provenance and SBOM

Two important concepts:
- Provenance: information about the build process and source
- SBOM: Software Bill of Materials describing software components

These improve supply-chain visibility and security analysis.

## 18. Build Checks and Linting

Useful checks include:
- Dockerfile syntax
- Dockerfile linting such as Hadolint
- Vulnerability scanning
- Dependency auditing
- Tests in the build or CI pipeline

A successful docker build does not prove an application is production-ready.

## 19. Advanced Dockerfile Example

    # syntax=docker/dockerfile:1
    FROM node:22-alpine AS deps
    WORKDIR /app
    COPY package*.json ./
    RUN --mount=type=cache,target=/root/.npm npm ci

    FROM node:22-alpine AS build
    WORKDIR /app
    COPY --from=deps /app/node_modules ./node_modules
    COPY . .
    RUN npm run build

    FROM node:22-alpine AS production
    WORKDIR /app
    ENV NODE_ENV=production
    COPY --from=deps /app/node_modules ./node_modules
    COPY --from=build /app/dist ./dist
    USER node
    CMD ["node", "dist/server.js"]

This combines caching, multiple stages, and a non-root runtime.

## 20. BuildKit in CI/CD

Production flow:

    Git push
       ↓
    CI starts
       ↓
    Buildx / BuildKit
       ↓
    Restore build cache
       ↓
    Build + test
       ↓
    Scan image
       ↓
    Generate metadata/attestations if required
       ↓
    Push image
       ↓
    Deploy immutable image

This is especially useful for Jenkins and other CI systems.

## 21. Common Mistakes

1. Putting secrets in ARG or ENV during builds.
2. Copying the entire repository before dependency installation and destroying cache efficiency.
3. Sending a huge build context.
4. Assuming cache always makes a build reproducible.
5. Building for the wrong CPU architecture.
6. Forgetting --push in a registry build.
7. Forgetting --load when a local Buildx result is needed.
8. Keeping compilers and build tools in the production image.
9. Using mutable tags for critical production deployments without considering immutability.
10. Assuming BuildKit itself makes an insecure Dockerfile secure.

## 22. Best Practices

1. Use BuildKit/Buildx for advanced builds.
2. Keep the build context small.
3. Order Dockerfile layers for effective caching.
4. Use cache mounts for expensive package-manager operations.
5. Use secret mounts for build-time credentials.
6. Use SSH mounts instead of copying private keys.
7. Use multi-stage builds.
8. Build only the required target.
9. Use multi-platform builds when needed.
10. Export/import cache in CI where useful.
11. Scan and test images before deployment.
12. Prefer immutable image references for critical production releases.
13. Generate provenance/SBOM information when required.

# Interview Questions

### 1. What is BuildKit?
BuildKit is Docker's modern build engine that provides improved caching, parallelism, advanced mounts, multi-platform builds, and other advanced build features.

### 2. What is Buildx?
Buildx is Docker's extended build interface for BuildKit-based builds.

### 3. Why use cache mounts?
They allow package-manager caches to persist between builds without becoming part of the final image.

### 4. How do BuildKit secret mounts improve security?
They make secrets available only to a build step instead of intentionally embedding them in image layers.

### 5. What is --load?
It loads a Buildx build result into the local Docker image store when the output is compatible with that store.

### 6. What is --push?
It pushes the build result directly to a container registry.

### 7. Why use multi-platform builds?
To produce images that can run on different CPU architectures such as amd64 and arm64.

### 8. Why is .dockerignore important?
It reduces build context size and prevents unnecessary or sensitive files from being sent to the builder.

### 9. What is an SBOM?
A Software Bill of Materials that lists software components contained in an artifact.

### 10. Does BuildKit make an application secure automatically?
No. BuildKit provides security-related features, but the Dockerfile, dependencies, secrets, runtime configuration, image, and host still need proper security controls.

# Quick Revision

BuildKit = modern Docker build engine
Buildx = extended CLI/build interface

Key features:
    Cache → faster builds
    Secret mounts → safer build secrets
    SSH mounts → private Git access
    Multi-stage → smaller runtime images
    Multi-platform → amd64/arm64
    Cache export/import → faster CI
    Provenance/SBOM → supply-chain visibility

# ⭐ Must Remember

1. BuildKit is the modern Docker build engine.
2. Buildx provides the advanced build interface.
3. Layer ordering strongly affects cache efficiency.
4. Use cache mounts for expensive dependency downloads.
5. Never bake build secrets into image layers.
6. Use SSH mounts instead of copying private SSH keys.
7. Use multi-stage builds to keep build tools out of production images.
8. --load loads a Buildx result locally; --push sends it to a registry.
9. Multi-platform builds commonly target linux/amd64 and linux/arm64.
10. Keep .dockerignore effective so the build context stays small.
11. Cache improves speed, not security or reproducibility by itself.
12. Provenance and SBOMs improve supply-chain visibility.
13. A strong CI pipeline can combine BuildKit → test → scan → push → deploy.