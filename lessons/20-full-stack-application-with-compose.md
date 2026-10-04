# Lesson 20 — Build a Full-Stack Application with Compose

## 1. What We Are Building

Docker Compose becomes especially useful when an application contains multiple services.

A typical full-stack application is:

~~~text
Browser
   |
   v
Frontend
   |
   v
Backend API
   |
   v
Database
~~~

Each component can run in its own container while Compose manages the complete application.

Example stack:

- React or Next.js frontend
- Node.js + Express backend
- MongoDB database
- Docker network
- Persistent database volume

---

## 2. Why Separate Containers?

A common beginner approach is putting the entire application into one container.

A better architecture separates responsibilities:

~~~text
+-------------+
|  Frontend   |
+------+------+ 
       |
       v
+-------------+
|   Backend   |
+------+------+
       |
       v
+-------------+
|  MongoDB    |
+-------------+
~~~

Benefits:

- Each service has a clear responsibility.
- Services can be built and updated independently.
- Dependencies remain isolated.
- Individual services can be scaled.
- Failures are easier to identify.
- The architecture is closer to real production systems.

---

## 3. Example Project Structure

~~~text
fullstack-app/
├── frontend/
│   ├── package.json
│   ├── src/
│   ├── Dockerfile
│   └── .dockerignore
│
├── backend/
│   ├── package.json
│   ├── src/
│   ├── Dockerfile
│   └── .dockerignore
│
└── compose.yaml
~~~

The database can use the official MongoDB image, so it does not need its own Dockerfile.

---

## 4. Backend Example

Suppose the Express API listens on port 3000.

~~~js
const express = require("express");

const app = express();

app.use(express.json());

app.get("/api/health", (req, res) => {
  res.json({ status: "ok" });
});

const PORT = process.env.PORT || 3000;

app.listen(PORT, "0.0.0.0", () => {
  console.log("API running on port " + PORT);
});
~~~

### Important

A containerized server should normally listen on:

~~~text
0.0.0.0
~~~

rather than only localhost.

This allows traffic to reach the application through the container network and published ports.

---

## 5. Backend Dockerfile

~~~dockerfile
FROM node:22-alpine

WORKDIR /app

COPY package*.json ./

RUN npm ci --omit=dev

COPY . .

ENV NODE_ENV=production
ENV PORT=3000

EXPOSE 3000

USER node

CMD ["node", "src/server.js"]
~~~

The Dockerfile defines how the backend image is built.

Compose defines how this backend runs with the other services.

---

## 6. Frontend Dockerfile

For a React/Vite production build, the frontend can be built with Node and served using Nginx.

~~~dockerfile
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
~~~

The final image contains the production frontend and Nginx rather than the Node build dependencies.

Next.js can use a different production Dockerfile because a Next.js application may require a Node runtime.

---

## 7. Compose File

Now we define the complete application.

~~~yaml
services:
  frontend:
    build: ./frontend
    ports:
      - "8080:80"
    depends_on:
      - backend

  backend:
    build: ./backend
    ports:
      - "3000:3000"
    environment:
      PORT: 3000
      MONGO_URL: mongodb://mongo:27017/myapp
    depends_on:
      - mongo

  mongo:
    image: mongo:8
    volumes:
      - mongo-data:/data/db

volumes:
  mongo-data:
~~~

One Compose file now describes:

- Frontend
- Backend
- MongoDB
- Network
- Persistent database storage

---

## 8. Application Architecture

After running:

~~~bash
docker compose up --build
~~~

the architecture is approximately:

~~~text
                    Host
                     |
          +----------+----------+
          |                     |
     localhost:8080       localhost:3000
          |                     |
          v                     v
     +----------+          +----------+
     | frontend | -------> | backend  |
     +----------+          +----+-----+
                                 |
                                 | mongo:27017
                                 v
                            +---------+
                            |  mongo  |
                            +----+----+
                                 |
                                 v
                            mongo-data
~~~

The user normally accesses the frontend.

The frontend communicates with the backend.

The backend communicates with MongoDB.

---

## 9. Most Important Concept: Browser vs Container Networking

This is one of the most important full-stack Docker concepts.

### Container to container

The backend connects to MongoDB using:

~~~text
mongodb://mongo:27017/myapp
~~~

The service name mongo is resolved through the Compose network.

### Host browser to backend

The browser can use:

~~~text
http://localhost:3000
~~~

because port 3000 was published from the backend container.

### Key distinction

~~~text
Container -> Container
service-name:container-port

Browser -> Host-published container
localhost:host-port
~~~

A browser normally cannot resolve the Docker-internal hostname backend.

---

## 10. Frontend API Configuration

Browser-based React code runs outside the Docker network.

Therefore, this usually does not work in browser JavaScript:

~~~text
http://backend:3000
~~~

For a simple local setup, the browser can call:

~~~text
http://localhost:3000
~~~

For Vite:

~~~env
VITE_API_URL=http://localhost:3000
~~~

Example:

~~~js
fetch(import.meta.env.VITE_API_URL + "/api/health");
~~~

### Important mental model

~~~text
Docker network:
backend -> mongo:27017

Host/browser:
browser -> localhost:3000
~~~

Always identify where the request originates before choosing the hostname.

---

## 11. Backend to MongoDB

The backend should use the MongoDB service name:

~~~text
mongodb://mongo:27017/myapp
~~~

Do not use:

~~~text
mongodb://localhost:27017/myapp
~~~

Inside the backend container:

~~~text
localhost = backend container
mongo     = MongoDB container
~~~

Docker Compose provides DNS resolution for service names.

---

## 12. Database Persistence

MongoDB stores its data at:

~~~text
/data/db
~~~

Compose mounts a named volume:

~~~yaml
volumes:
  - mongo-data:/data/db
~~~

and declares it at the top level:

~~~yaml
volumes:
  mongo-data:
~~~

The idea is:

~~~text
MongoDB container
       |
       v
    /data/db
       |
       v
 mongo-data volume
~~~

If the MongoDB container is recreated, the data can remain in the named volume.

---

## 13. Start the Full Stack

From the project root:

~~~bash
docker compose up --build
~~~

Background mode:

~~~bash
docker compose up --build -d
~~~

Check services:

~~~bash
docker compose ps
~~~

Expected services:

~~~text
frontend
backend
mongo
~~~

---

## 14. Test Each Layer

### Test the backend

Open:

~~~text
http://localhost:3000/api/health
~~~

Expected response:

~~~json
{
  "status": "ok"
}
~~~

### Test the frontend

Open:

~~~text
http://localhost:8080
~~~

### Check backend logs

~~~bash
docker compose logs -f backend
~~~

### Enter the backend container

~~~bash
docker compose exec backend sh
~~~

This is useful for debugging configuration and network problems.

---

## 15. Code Changes and Rebuilds

If application source files were copied into the image, changing files on the host does not automatically modify the existing image.

Rebuild:

~~~bash
docker compose up --build
~~~

For development, bind mounts and hot reload provide a better workflow. That topic is covered later.

---

## 16. Scaling a Service

Compose can run multiple instances of a service in supported workflows.

Example:

~~~bash
docker compose up --scale backend=3
~~~

Conceptually:

~~~text
        backend
       /   |   \
      v    v    v
 backend-1 backend-2 backend-3
~~~

However, if the backend publishes a fixed host port:

~~~yaml
ports:
  - "3000:3000"
~~~

multiple replicas cannot all bind the same host port.

This demonstrates an important difference:

- Internal service communication can involve multiple containers.
- Host port publishing must be designed for the number of replicas.

For larger production environments, Kubernetes provides more advanced service discovery and load balancing.

---

## 17. Adding Redis

A real application may have additional services.

~~~yaml
services:
  backend:
    build: ./backend
    environment:
      REDIS_URL: redis://redis:6379

  redis:
    image: redis:8
~~~

The backend connects to:

~~~text
redis://redis:6379
~~~

The same service-name principle applies to MongoDB, PostgreSQL, Redis, RabbitMQ, and other Compose services.

---

## 18. Full-Stack Networking Mental Model

Always ask: Who is making the request?

### Browser to frontend

~~~text
http://localhost:8080
~~~

### Browser to backend

~~~text
http://localhost:3000
~~~

### Container to container

~~~text
http://backend:3000
~~~

### Backend to MongoDB

~~~text
mongodb://mongo:27017/myapp
~~~

The correct hostname depends on the request origin.

---

## 19. Common Mistakes

### Mistake 1 — Backend uses localhost for MongoDB

Wrong:

~~~text
mongodb://localhost:27017/myapp
~~~

Correct:

~~~text
mongodb://mongo:27017/myapp
~~~

### Mistake 2 — Browser uses the Docker service name

Usually wrong:

~~~text
http://backend:3000
~~~

Use a browser-reachable address such as:

~~~text
http://localhost:3000
~~~

or use a reverse proxy architecture.

### Mistake 3 — Backend listens only on localhost

Wrong:

~~~js
app.listen(3000, "127.0.0.1");
~~~

Better:

~~~js
app.listen(3000, "0.0.0.0");
~~~

### Mistake 4 — Database data disappears

Use a named volume:

~~~yaml
volumes:
  - mongo-data:/data/db
~~~

### Mistake 5 — Assuming depends_on means readiness

depends_on controls dependency startup ordering, but it does not automatically guarantee that MongoDB is ready to accept connections.

Use health checks and application retry or connection handling when required.

### Mistake 6 — Changing code without rebuilding

If source is copied into the image:

~~~bash
docker compose up --build
~~~

is required after changes unless a development bind-mount workflow is configured.

---

## 20. Debugging Checklist

Debug the application layer by layer.

### 1. Check containers

~~~bash
docker compose ps
~~~

### 2. Check logs

~~~bash
docker compose logs backend
docker compose logs frontend
docker compose logs mongo
~~~

### 3. Check networks

~~~bash
docker network ls
~~~

### 4. Validate Compose configuration

~~~bash
docker compose config
~~~

### 5. Enter a container

~~~bash
docker compose exec backend sh
~~~

### 6. Check environment variables

~~~bash
docker compose exec backend env
~~~

### 7. Check volumes

~~~bash
docker volume ls
~~~

Do not randomly change configuration. Identify which connection in the chain is failing.

---

## 21. Production Architecture

A larger production architecture may look like:

~~~text
Internet
   |
   v
Reverse Proxy / Load Balancer
   |
   +------------+
   |            |
   v            v
Frontend      Backend
                |
          +-----+-----+
          |           |
          v           v
       Database      Redis
~~~

Docker Compose is excellent for local development, testing, learning, and smaller deployments.

For larger environments requiring advanced scheduling, self-healing, autoscaling, and distributed orchestration, Kubernetes is more appropriate.

---

## 22. CodeBuddy Connection

The CodeBuddy architecture can follow the same pattern:

~~~text
React Frontend
      |
      v
Node + Express API
      |
      +-------> MongoDB
      |
      +-------> Other services
~~~

Later, CodeBuddy can be extended with:

- Frontend Dockerfile
- Backend Dockerfile
- Compose networking
- MongoDB persistent volume
- Environment configuration
- Redis if needed
- Reverse proxy
- CI/CD
- Kubernetes

This lesson provides the foundation for the later CodeBuddy production workflow.

---

## 23. Interview Questions

### Q1. How would you Dockerize a full-stack application?

Create separate images and containers for the frontend, backend, and supporting services. Use Docker Compose to define builds, networking, ports, volumes, environment variables, and dependencies.

### Q2. How does the backend connect to MongoDB in Compose?

Using the Compose service name and MongoDB internal port:

~~~text
mongodb://mongo:27017/database
~~~

### Q3. Why can browser JavaScript normally not use the Docker hostname backend?

Because backend is a Docker-network DNS name. Browser JavaScript runs outside that Docker network, so it normally needs a host-reachable URL such as localhost:3000 or a reverse-proxy URL.

### Q4. Why use separate frontend and backend containers?

They have different responsibilities, dependencies, build processes, and scaling requirements.

### Q5. How do you persist MongoDB data?

Use a named Docker volume mounted at MongoDB's data directory.

### Q6. What happens when docker compose down is executed?

Compose removes the application containers and networks. Named volumes are normally preserved unless volume removal is explicitly requested.

### Q7. What does docker compose up --build do?

It rebuilds the required images before starting the services.

### Q8. What is the difference between service communication and browser communication?

Service communication occurs inside the Docker network and uses service names. Browser communication occurs outside Docker and normally uses a published host port or reverse proxy.

---

## 24. Quick Revision

~~~text
Frontend container
       |
       v
Backend container
       |
       v
Database container
       |
       v
Named volume
~~~

### Key rules

~~~text
Container -> Container
service-name:container-port

Browser -> Docker-published service
localhost:host-port

Database persistence
named volume

Image construction
Dockerfile

Multi-service runtime configuration
Compose
~~~

### Essential commands

~~~bash
docker compose up
docker compose up --build
docker compose up -d
docker compose ps
docker compose logs -f
docker compose exec <service> sh
docker compose config
docker compose down
~~~

---

## ⭐ Must Remember

1. **Separate application responsibilities into services.**
2. **Dockerfiles build individual service images; Compose runs the complete stack.**
3. **Container-to-container communication uses Compose service names.**
4. **Browser requests normally use host-published ports, not Docker service names.**
5. **localhost inside a container means that container itself.**
6. **Backend containers should normally listen on 0.0.0.0.**
7. **Use named volumes for persistent database data.**
8. **depends_on does not guarantee application readiness.**
9. **Rebuild images when code is copied into the image and no development mount is configured.**
10. **Debug the stack layer by layer: frontend -> backend -> database/network.**

---

## 🎯 Interview Summary

A strong interview explanation:

> I would separate the frontend, backend, and database into independent containers. Dockerfiles define how the frontend and backend images are built, while Docker Compose defines how the services run together. Compose provides a network where services communicate through service names, so the backend can connect to MongoDB using mongo:27017. The browser is outside the Docker network, so browser-side API calls normally use a published host port or a reverse proxy. Database data is persisted with a named volume. For production readiness, I would also add health checks, secure configuration, a reverse proxy, and appropriate orchestration depending on scale.
