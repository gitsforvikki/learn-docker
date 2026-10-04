# Lesson 10 — Docker Image Optimization

## 1. Why Optimize Docker Images?

Docker image optimization means making an image:

- Smaller
- Faster to build
- Faster to transfer
- Faster to deploy
- Easier to maintain
- Less exposed to unnecessary security vulnerabilities

A smaller image is not automatically a better image, but unnecessary files, packages, and build tools should be avoided.

---

## 2. Main Optimization Areas

The most important areas are:

```text
Base image
   ↓
Build context
   ↓
Dockerfile layers
   ↓
Build cache
   ↓
Dependencies
   ↓
Multi-stage builds
   ↓
Runtime image
```

---

## 3. Choose an Appropriate Base Image

Example:

```dockerfile
FROM node:22
```

This may contain more components than an application needs.

A smaller alternative may be:

```dockerfile
FROM node:22-alpine
```

Other commonly used minimal variants include:

```text
-alpine
-slim
```

### Important

Do not choose a base image only because it is smaller.

The image must:

- Support your application
- Contain required system libraries
- Work correctly with native dependencies
- Be maintained and appropriate for production

**Correctness comes before size.**

---

## 4. Use .dockerignore

A `.dockerignore` file prevents unnecessary files from entering the build context.

Example:

```text
node_modules
.git
.next
.env
coverage
*.log
```

Benefits:

- Smaller build context
- Faster builds
- Less data transferred to the builder
- Lower chance of accidentally including sensitive files

For Node.js projects, excluding `node_modules` is especially important.

---

## 5. Reduce Unnecessary Layers and Instructions

Docker images are built in layers.

Avoid creating unnecessary filesystem changes.

For example, package installation and cleanup can sometimes be performed in the same `RUN` instruction:

```dockerfile
RUN apk add --no-cache curl
```

The important principle is not "always minimize the number of lines." It is:

> Avoid unnecessary data remaining in image layers.

Modern Docker build tools can optimize builds significantly, so layer count alone should not be treated as the only optimization metric.

---

## 6. Use Build Cache Effectively

Good instruction ordering allows Docker to reuse cached layers.

For Node.js:

```dockerfile
COPY package*.json ./
RUN npm ci

COPY . .
```

This is generally better than:

```dockerfile
COPY . .
RUN npm ci
```

Why?

If only application source changes:

```text
package.json unchanged
       ↓
npm ci layer can be reused
       ↓
only later source-related steps rebuild
```

This makes development and CI builds faster.

---

## 7. Multi-Stage Builds

Multi-stage builds are one of the most important Docker optimization techniques.

Example:

```dockerfile
FROM node:22-alpine AS builder

WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .
RUN npm run build

FROM node:22-alpine

WORKDIR /app

COPY --from=builder /app/package*.json ./
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/dist ./dist

CMD ["node", "dist/server.js"]
```

The first stage contains build requirements.

The final stage contains only what is required to run the application.

Conceptually:

```text
Builder image
├── source
├── dependencies
├── compiler/build tools
└── temporary files
          ↓
       build
          ↓
Final image
├── runtime
├── production dependencies
└── application output
```

### Benefits

- Smaller final image
- Fewer unnecessary tools
- Reduced attack surface
- Cleaner runtime environment

---

## 8. Install Only Required Dependencies

Production containers usually should not contain unnecessary development dependencies.

For Node.js applications, depending on the application and package manager:

```bash
npm ci --omit=dev
```

can install production dependencies only.

The exact approach should match the application's build requirements.

For applications that need development dependencies to build, install them in the builder stage and keep only the required runtime dependencies in the final stage.

---

## 9. Avoid Copying Unnecessary Files

Do not blindly copy the entire project if the runtime does not need everything.

Instead of:

```dockerfile
COPY . .
```

a final runtime stage can selectively copy required artifacts:

```dockerfile
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/package.json ./
```

The exact files depend on the application architecture.

This is particularly useful with compiled applications and frontend/server builds.

---

## 10. Don't Put Secrets in Images

Never place secrets directly into:

- Dockerfile
- `ENV`
- `ARG`
- Source code copied into the image

Example of what to avoid:

```dockerfile
ENV DATABASE_PASSWORD=secret123
```

Once a secret is included in an image, removing the Dockerfile line later does not automatically remove it from already-built image history/layers.

Use runtime environment configuration or a proper secrets mechanism instead.

---

## 11. Run as a Non-Root User

A container should not run as root unless root privileges are actually required.

Example:

```dockerfile
USER node
```

Running as a non-root user can reduce the impact of an application compromise.

The exact user and permissions depend on the base image and application.

---

## 12. Pin Important Versions

Avoid relying blindly on floating tags.

Example:

```dockerfile
FROM node:22-alpine
```

is more predictable than:

```dockerfile
FROM node:latest
```

For highly reproducible builds, image digests can provide an even stronger guarantee:

```dockerfile
FROM node:22-alpine@sha256:<digest>
```

There is a trade-off: digest pinning improves reproducibility but requires deliberate dependency update management.

---

## 13. Keep the Runtime Image Focused

A production runtime image should contain what the application needs to run, not everything used to develop or build it.

Avoid unnecessary:

- Compilers
- Debugging tools
- Source files not required at runtime
- Test files
- Documentation
- Package caches
- Development dependencies

Multi-stage builds are often the cleanest way to achieve this.

---

## 14. Combine Optimization With Correctness

Image optimization is not about making the Dockerfile as short as possible.

For example, switching to an extremely minimal image can break an application if it requires system libraries that the image does not provide.

A good optimization process is:

```text
Working application
      ↓
Measure image size/build time
      ↓
Remove unnecessary content
      ↓
Test application
      ↓
Check security
      ↓
Measure again
```

Always verify that optimization does not change application behavior.

---

## 15. Example — Optimized Node.js Image

A common production pattern is:

```dockerfile
FROM node:22-alpine AS builder

WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .
RUN npm run build

FROM node:22-alpine

WORKDIR /app

ENV NODE_ENV=production

COPY package*.json ./
RUN npm ci --omit=dev

COPY --from=builder /app/dist ./dist

USER node

EXPOSE 3000

CMD ["node", "dist/server.js"]
```

This separates:

```text
Builder
→ install/build dependencies
→ compile application

Final image
→ runtime
→ production dependencies
→ built application
```

The exact Dockerfile should be adapted to the application's framework and build output.

---

## 16. Inspect Image Size

List images:

```bash
docker image ls
```

View image history:

```bash
docker image history my-app:1.0
```

Inspect detailed metadata:

```bash
docker image inspect my-app:1.0
```

These commands help identify unexpectedly large images and understand how the image was constructed.

---

## 17. BuildKit and Modern Docker Builds

Modern Docker builds use BuildKit by default in current Docker setups.

BuildKit provides advanced build capabilities such as:

- Better caching
- Parallel build work where possible
- Multi-stage builds
- Improved build output
- Advanced cache/export features

For normal Docker usage, the important point is:

> Modern Docker image builds are designed around efficient layer caching and BuildKit-based build functionality.

---

## 18. Common Optimization Mistakes

### Mistake 1: Choosing the smallest possible image without testing

A smaller image is useless if required libraries are missing.

### Mistake 2: Copying node_modules from the host

Host dependencies may be platform-specific and unnecessarily large.

Use the package manager inside the image/build stage instead.

### Mistake 3: Including development tools in the final image

Use multi-stage builds to keep build-only tools out of production.

### Mistake 4: Keeping package caches unnecessarily

Clean package-manager caches when they would otherwise remain in the final image.

### Mistake 5: Storing secrets in the image

Images can be inspected and reused. Secrets should not be baked into them.

### Mistake 6: Using --no-cache for every build

This makes builds slower and removes useful cache reuse.

### Mistake 7: Optimizing only for image size

Build speed, security, maintainability, reproducibility, and application correctness also matter.

---

## 19. Interview Questions

### Q1. How can you reduce Docker image size?

Use an appropriate base image, .dockerignore, multi-stage builds, production-only dependencies, selective copying, and removal of unnecessary files and packages.

### Q2. Why are multi-stage builds useful?

They separate build dependencies from runtime dependencies and allow only required artifacts to be copied into the final image.

### Q3. Why is .dockerignore important?

It prevents unnecessary files from entering the build context and can improve build performance and reduce accidental inclusion of sensitive files.

### Q4. Why should you avoid putting secrets in Docker images?

Images and their metadata/layers can be inspected, cached, stored, and distributed. Secrets baked into an image can therefore be exposed and are difficult to remove safely.

### Q5. Why use a non-root user?

It reduces the privileges available to the application and can limit the impact of a container compromise.

### Q6. Is Alpine always the best base image?

No. It is often small, but compatibility and application requirements must be considered.

### Q7. How can Docker build time be improved?

Use effective layer ordering, build caching, a good .dockerignore, and multi-stage builds where appropriate.

### Q8. What is the goal of a production runtime image?

To contain the minimum practical set of runtime components required to run the application reliably and securely.

---

## Quick Revision

```text
Image Optimization
       ↓
Appropriate base image
       ↓
.dockerignore
       ↓
Good layer/cache strategy
       ↓
Multi-stage build
       ↓
Production dependencies only
       ↓
No secrets
       ↓
Non-root runtime
       ↓
Focused final image
```

### Must Remember

1. **Optimize for size, build speed, security, reproducibility, and correctness.**
2. **Use .dockerignore to control the build context.**
3. **Order Dockerfile instructions to maximize cache reuse.**
4. **Use multi-stage builds to keep build tools out of the runtime image.**
5. **Install only the dependencies required by the runtime.**
6. **Never bake secrets into Docker images.**
7. **Run as a non-root user when practical.**
8. **Do not assume Alpine is always the best choice.**
9. **Use versioned or digest-pinned base images when reproducibility matters.**
10. **Measure and test optimization changes instead of optimizing blindly.**
