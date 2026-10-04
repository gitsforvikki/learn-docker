# Lesson 08 — Building Your Own Docker Image

## 1. What Does "Building an Image" Mean?

Building a Docker image means converting a **Dockerfile + build context** into a reusable Docker image.

Basic flow:

```text
Project Files
     +
Dockerfile
     +
.dockerignore
     ↓
docker build
     ↓
Docker Image
     ↓
docker run
     ↓
Container
```

The image contains the application and everything required to run it.

---

## 2. Example Application

Consider a simple Node.js application:

```text
my-app/
├── package.json
├── package-lock.json
├── server.js
├── Dockerfile
└── .dockerignore
```

Example `server.js`:

```js
const http = require("http");

const PORT = process.env.PORT || 3000;

const server = http.createServer((req, res) => {
  res.end("Hello from Docker!");
});

server.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});
```

---

## 3. Create the Dockerfile

Create a file named exactly:

```text
Dockerfile
```

Example:

```dockerfile
FROM node:22-alpine

WORKDIR /app

COPY package*.json ./

RUN npm ci

COPY . .

ENV NODE_ENV=production

EXPOSE 3000

CMD ["node", "server.js"]
```

### Why this order?

The dependency files are copied before the application source:

```dockerfile
COPY package*.json ./
RUN npm ci
COPY . .
```

If only source code changes, Docker can often reuse the dependency installation layer during the next build.

This is an important Docker build-cache optimization.

---

## 4. Create .dockerignore

Create:

```text
.dockerignore
```

Example:

```text
node_modules
.git
.env
npm-debug.log
```

This prevents unnecessary files from being sent as part of the build context.

---

## 5. Build the Image

From the project directory:

```bash
docker build -t my-node-app:1.0 .
```

Breakdown:

- `docker build` → builds an image
- `-t my-node-app:1.0` → assigns repository and tag
- `.` → current directory is the build context

Result:

```text
Dockerfile + context
        ↓
   Docker builder
        ↓
my-node-app:1.0
```

---

## 6. Verify the Image

List images:

```bash
docker image ls
```

You should see:

```text
my-node-app    1.0
```

Inspect the image:

```bash
docker image inspect my-node-app:1.0
```

View its build history:

```bash
docker image history my-node-app:1.0
```

---

## 7. Run the Image

Start a container:

```bash
docker run -d --name my-node-container -p 3000:3000 my-node-app:1.0
```

Breakdown:

```text
-d                    → detached mode
--name                → container name
-p 3000:3000          → host:container port
my-node-app:1.0       → image
```

Check the container:

```bash
docker ps
```

View logs:

```bash
docker logs my-node-container
```

The application should be reachable on the published host port.

---

## 8. Test the Container

Use:

```bash
curl http://localhost:3000
```

Expected response:

```text
Hello from Docker!
```

The complete flow is:

```text
Dockerfile
   ↓
docker build
   ↓
Image
   ↓
docker run
   ↓
Container
   ↓
Application process
   ↓
Port 3000
```

---

## 9. Understanding the Build Cache

Docker builds images in layers.

Suppose the Dockerfile contains:

```dockerfile
FROM node:22-alpine

WORKDIR /app

COPY package*.json ./

RUN npm ci

COPY . .

CMD ["node", "server.js"]
```

If only `server.js` changes:

```text
FROM                 → cached
WORKDIR              → cached
COPY package*.json   → cached
RUN npm ci           → cached
COPY . .             → rebuilt
CMD                  → reused/rebuilt as needed
```

Docker can reuse unchanged layers.

### Why this matters

Good Dockerfile ordering can make repeated builds significantly faster.

A common pattern for Node.js applications is:

```dockerfile
COPY package*.json ./
RUN npm ci
COPY . .
```

rather than:

```dockerfile
COPY . .
RUN npm ci
```

because application source files usually change more frequently than dependency files.

---

## 10. Rebuilding After a Code Change

Change `server.js`, then build again:

```bash
docker build -t my-node-app:1.0 .
```

Docker checks each build step and reuses valid cached layers.

If a layer changes, subsequent dependent layers may also need to be rebuilt.

---

## 11. Build Without Cache

To force Docker to rebuild without using the normal build cache:

```bash
docker build --no-cache -t my-node-app:1.0 .
```

Use this when troubleshooting cache-related build behavior or when you intentionally need a clean rebuild.

Do not use `--no-cache` routinely because it makes builds slower.

---

## 12. Build With a Different Tag

Tags are useful for versioning.

```bash
docker build -t my-node-app:1.1 .
```

Now the same application can have different image references:

```text
my-node-app:1.0
my-node-app:1.1
```

In real projects, meaningful version tags help identify exactly what should be deployed.

---

## 13. Tag an Existing Image

You can assign another tag to an existing image:

```bash
docker tag my-node-app:1.0 my-node-app:latest
```

You can also tag it for a registry:

```bash
docker tag my-node-app:1.0 username/my-node-app:1.0
```

The tag operation does not rebuild the image.

---

## 14. Build Context Is Important

When you run:

```bash
docker build -t my-node-app .
```

the `.` determines the build context.

Dockerfile instructions such as:

```dockerfile
COPY . .
```

can only access files available within that context.

For example:

```text
project/
├── Dockerfile
├── src/
└── package.json
```

Running the build from `project/` gives Docker access to those files.

Files outside the context cannot normally be copied into the image with `COPY`.

---

## 15. Build Context vs Dockerfile Location

The Dockerfile and build context are related but are not necessarily the same path.

Example:

```bash
docker build -f docker/Dockerfile -t my-app .
```

Here:

- `-f docker/Dockerfile` → selects the Dockerfile
- `.` → build context

This distinction is important in larger projects and monorepos.

---

## 16. Multi-Stage Builds

A multi-stage Dockerfile uses multiple `FROM` instructions to separate the **build environment** from the **final runtime image**.

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

Conceptually:

```text
Builder stage
   ↓
compile / build
   ↓
Only required output
   ↓
Final runtime stage
```

### Why multi-stage builds matter

They can:

- Keep build tools out of the final image
- Reduce final image size
- Reduce unnecessary dependencies
- Improve security by minimizing the runtime environment

Multi-stage builds are especially important for compiled applications and frontend builds.

---

## 17. Build Arguments

A build can accept configurable values with `ARG`.

Dockerfile:

```dockerfile
ARG NODE_VERSION=22
FROM node:${NODE_VERSION}-alpine
```

Build:

```bash
docker build --build-arg NODE_VERSION=22 -t my-node-app:1.0 .
```

Remember:

```text
ARG → build time
ENV → runtime environment
```

Do not use build arguments as a secure mechanism for secrets.

---

## 18. Image Build vs Container Run

These are different operations.

### Build

```bash
docker build -t my-app:1.0 .
```

Creates an image.

### Run

```bash
docker run my-app:1.0
```

Creates and starts a container from the image.

Think:

```text
docker build → Image
docker run   → Container
```

---

## 19. Useful Troubleshooting Commands

### See running containers

```bash
docker ps
```

### See all containers

```bash
docker ps -a
```

### Check logs

```bash
docker logs <container>
```

### Open a shell in a running container

```bash
docker exec -it <container> sh
```

### Inspect container configuration

```bash
docker inspect <container>
```

### Inspect image configuration

```bash
docker image inspect <image>
```

These commands help determine whether the problem is in the image, container configuration, application process, networking, or runtime environment.

---

## 20. Common Mistakes

### Mistake 1: Building from the wrong directory

The build context controls which files are available to `COPY`.

### Mistake 2: Forgetting .dockerignore

Large or sensitive files may unnecessarily enter the build context.

### Mistake 3: Poor Dockerfile instruction order

Copying the entire project before installing dependencies can invalidate the dependency cache whenever source code changes.

### Mistake 4: Confusing image and container

`docker build` creates an image; `docker run` creates and starts a container.

### Mistake 5: Using --no-cache for every build

It removes the performance benefit of build caching.

### Mistake 6: Putting development dependencies and build tools in the final production image unnecessarily

Multi-stage builds can keep the final runtime image smaller and cleaner.

---

## 21. Interview Questions

### Q1. What happens when you run docker build?

Docker reads the Dockerfile and build context, executes the build instructions, creates filesystem layers, and produces a Docker image.

### Q2. What does the dot in docker build -t my-app . mean?

It specifies the current directory as the build context.

### Q3. Why copy package.json before copying the source code?

It allows Docker to reuse the dependency installation layer when application source files change but dependency files remain unchanged.

### Q4. What is Docker build cache?

Docker can reuse previously built layers when the corresponding build step and its inputs have not changed.

### Q5. What is the difference between docker build and docker run?

`docker build` creates an image. `docker run` creates and starts a container from an image.

### Q6. What is a multi-stage build?

A Dockerfile technique using multiple build stages so that only the required artifacts are copied into the final runtime image.

### Q7. Can the Dockerfile be stored in a different location from the build context?

Yes. The `-f` option selects the Dockerfile while the final path argument specifies the build context.

### Q8. Does docker tag create another copy of the image?

No. It creates another reference/tag for the same image content.

---

## Quick Revision

```text
Dockerfile + Context
        ↓
   docker build
        ↓
      Image
        ↓
    docker run
        ↓
    Container
```

### Must Remember

1. **`docker build` creates an image; `docker run` creates and starts a container.**
2. **The final argument of `docker build` specifies the build context.**
3. **Docker builds images in layers and can reuse cached layers.**
4. **Copy dependency files before source code when that improves cache reuse.**
5. **Use `.dockerignore` to keep unnecessary files out of the build context.**
6. **`--no-cache` forces a clean build but should not be used unnecessarily.**
7. **`-f` selects the Dockerfile; the final path specifies the build context.**
8. **Multi-stage builds separate build-time requirements from the final runtime image.**
9. **Tags are references; tagging does not rebuild or duplicate the image content.**
