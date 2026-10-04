# Lesson 22 — Compose Networking

## 1. What Is Compose Networking?

Docker Compose automatically creates a network for an application when services are started.

This allows services in the same Compose project to communicate.

Example:

~~~text
             Compose Network
                    |
       +------------+------------+
       |            |            |
       v            v            v
   frontend      backend      database
~~~

The most important rule is:

**Use the Compose service name as the hostname for service-to-service communication.**

Examples:

~~~text
backend -> database:5432
backend -> redis:6379
frontend -> backend:3000
~~~

---

## 2. Why Networking Matters

A multi-container application is useful only when its services can communicate.

~~~text
Frontend
   |
   | HTTP
   v
Backend
   |
   | Database protocol
   v
Database
~~~

Each service can remain isolated in its own container while Docker provides the communication path.

---

## 3. Compose Creates a Default Network

Suppose the Compose file contains:

~~~yaml
services:
  frontend:
    image: nginx:alpine

  backend:
    image: my-backend

  mongo:
    image: mongo:8
~~~

When you run:

~~~bash
docker compose up -d
~~~

Compose normally creates a project network similar to:

~~~text
projectname_default
~~~

All services are attached to that network unless configured otherwise.

---

## 4. Service Names Become Hostnames

Suppose:

~~~yaml
services:
  backend:
    image: my-backend

  mongo:
    image: mongo:8
~~~

The backend can connect to MongoDB using:

~~~text
mongo:27017
~~~

It does not need the MongoDB container IP.

Conceptually:

~~~text
mongo
  |
  v
Docker DNS
  |
  v
MongoDB container IP
~~~

This is called **service discovery**.

---

## 5. Why Container IPs Should Not Be Hard-Coded

Container IP addresses can change when containers are recreated.

Example:

~~~text
Old MongoDB container
172.20.0.5

Container recreated

New MongoDB container
172.20.0.8
~~~

If the backend stored the old IP, the connection could break.

Using:

~~~text
mongo:27017
~~~

allows Docker to resolve the current container address.

### Rule

**Use service names, not container IP addresses.**

---

## 6. Container Port vs Host Port

Consider:

~~~yaml
services:
  backend:
    ports:
      - "3000:3000"

  mongo:
    ports:
      - "27018:27017"
~~~

There are two different networking paths.

### Host to backend

~~~text
localhost:3000
~~~

### Backend to MongoDB

~~~text
mongo:27017
~~~

The backend does not need to use:

~~~text
mongo:27018
~~~

The published host port 27018 is for access through the host.

The internal MongoDB port remains 27017.

### Must remember

~~~text
Host access:
localhost:published-host-port

Container access:
service-name:container-port
~~~

---

## 7. Browser Networking vs Docker Networking

This is one of the most important concepts.

A browser running on your host is not automatically part of the Compose network.

Therefore:

~~~text
Browser -> backend
localhost:3000
~~~

while:

~~~text
Backend container -> MongoDB container
mongo:27017
~~~

A server-side frontend container can communicate with the backend using:

~~~text
backend:3000
~~~

### Mental model

~~~text
                    Host
                     |
                  Browser
                     |
               localhost:3000
                     |
                     v
                +---------+
                | backend |
                +----+----+
                     |
                 mongo:27017
                     |
                     v
                +---------+
                |  mongo  |
                +---------+
~~~

---

## 8. Creating a Custom Network

You can explicitly define a network.

~~~yaml
services:
  backend:
    image: my-backend
    networks:
      - app-network

  mongo:
    image: mongo:8
    networks:
      - app-network

networks:
  app-network:
~~~

Both services can communicate through app-network.

---

## 9. Why Use Custom Networks?

Custom networks provide explicit control over service communication.

Benefits:

- Better architecture visibility
- Network isolation
- Separation of public and private services
- Multiple independent application networks
- More controlled communication

Example:

~~~text
Public Network
frontend <----> backend

Private Network
backend <----> database
~~~

---

## 10. Multiple Networks

A service can belong to more than one network.

~~~yaml
services:
  frontend:
    networks:
      - public

  backend:
    networks:
      - public
      - private

  database:
    networks:
      - private

networks:
  public:
  private:
~~~

Architecture:

~~~text
              public
frontend ---------------- backend
                              |
                            private
                              |
                              v
                           database
~~~

This creates a useful isolation boundary.

- frontend can reach backend.
- backend can reach database.
- frontend cannot directly reach database.

---

## 11. Internal Networks

A network can be marked as internal:

~~~yaml
networks:
  private:
    internal: true
~~~

This is useful for isolated backend/database communication that should not provide normal external connectivity through that network.

---

## 12. External Networks

Compose can connect services to a network that already exists.

Create it:

~~~bash
docker network create shared-network
~~~

Then:

~~~yaml
networks:
  shared-network:
    external: true
~~~

A service can use it:

~~~yaml
services:
  backend:
    image: my-backend
    networks:
      - shared-network
~~~

This can be useful when multiple Compose projects need to communicate through a shared Docker network.

---

## 13. Network Aliases

A service can have additional DNS names on a network.

~~~yaml
services:
  backend:
    image: my-backend
    networks:
      app-network:
        aliases:
          - api

networks:
  app-network:
~~~

Other services on that network can then use either the service name or the alias.

---

## 14. Ports Do Not Define Internal Service Networking

Consider:

~~~yaml
services:
  backend:
    ports:
      - "3000:3000"

  database:
    image: postgres:18
~~~

The backend can communicate with the database if both services share a network.

The database does not need:

~~~yaml
ports:
  - "5432:5432"
~~~

unless the host itself needs direct database access.

### Better architecture

~~~text
Browser
   |
   v
Backend
   |
   v
Database
~~~

Only expose services that actually need host access.

---

## 15. expose

A service can declare a port using expose:

~~~yaml
services:
  backend:
    expose:
      - "3000"
~~~

It does not publish the port to the host.

Another service on the same network can communicate using:

~~~text
backend:3000
~~~

In modern Compose networking, explicit expose is often unnecessary because services on the same network can already communicate through their internal ports.

---

## 16. DNS Resolution

Compose provides DNS-based service discovery.

Suppose:

~~~yaml
services:
  api:
    image: my-api

  redis:
    image: redis:8
~~~

The API can connect to:

~~~text
redis:6379
~~~

Conceptually:

~~~text
api
 |
 | DNS query: redis
 v
Docker DNS
 |
 v
Redis container IP
 |
 v
Redis:6379
~~~

The application does not need to know the IP address.

---

## 17. Inspecting Networks

List Docker networks:

~~~bash
docker network ls
~~~

Inspect a network:

~~~bash
docker network inspect <network-name>
~~~

For example:

~~~bash
docker network inspect myapp_default
~~~

This can show connected containers and network configuration.

---

## 18. Testing Connectivity

Enter a container:

~~~bash
docker compose exec backend sh
~~~

Then test the target service if suitable networking tools are installed.

For example:

~~~bash
wget -qO- http://frontend:80
~~~

or:

~~~bash
wget -qO- http://backend:3000/api/health
~~~

Minimal images may not contain ping, curl, or netcat, so do not assume those commands are available.

---

## 19. Network Isolation and Security

Example:

~~~yaml
services:
  frontend:
    networks:
      - public

  backend:
    networks:
      - public
      - private

  database:
    networks:
      - private

networks:
  public:
  private:
    internal: true
~~~

Flow:

~~~text
Browser
   |
   v
Frontend
   |
   v
Backend
   |
   v
Database
~~~

The database is isolated from the public network.

Good practices:

- Do not publish database ports unless required.
- Keep databases on private networks.
- Use multiple networks when isolation is needed.
- Avoid hard-coded container IPs.
- Expose only required host ports.
- Combine network isolation with authentication and encryption.
- Do not assume Docker networking alone makes a service secure.

---

## 20. PostgreSQL Example

~~~yaml
services:
  backend:
    build: ./backend
    environment:
      DATABASE_URL: postgresql://postgres:password@database:5432/app
    networks:
      - private

  database:
    image: postgres:18
    networks:
      - private
    volumes:
      - postgres-data:/var/lib/postgresql/data

networks:
  private:

volumes:
  postgres-data:
~~~

The backend uses:

~~~text
database:5432
~~~

not localhost:5432.

---

## 21. Redis Example

~~~yaml
services:
  backend:
    environment:
      REDIS_URL: redis://redis:6379
    networks:
      - private

  redis:
    image: redis:8
    networks:
      - private

networks:
  private:
~~~

The backend connects to:

~~~text
redis:6379
~~~

No host port is necessary unless external Redis access is actually required.

---

## 22. CodeBuddy Networking Model

A practical CodeBuddy Compose architecture can be:

~~~text
                    Browser
                       |
                       | published port
                       v
                 React Frontend
                       |
                       | API request
                       v
                 Node/Express API
                    /        \
                   /          \
                  v            v
              MongoDB        Redis
~~~

Docker-internal connections:

~~~text
backend -> mongo:27017
backend -> redis:6379
~~~

Browser-facing access:

~~~text
browser -> localhost:<frontend-port>
~~~

The database and Redis services should normally remain private.

---

## 23. Common Mistakes

### Mistake 1 — Using localhost between containers

Wrong:

~~~text
mongodb://localhost:27017/codebuddy
~~~

Correct:

~~~text
mongodb://mongo:27017/codebuddy
~~~

### Mistake 2 — Using the published host port internally

If MongoDB uses:

~~~yaml
ports:
  - "27018:27017"
~~~

the backend should still use:

~~~text
mongo:27017
~~~

not mongo:27018.

### Mistake 3 — Publishing every service

Avoid unnecessary database port publishing.

### Mistake 4 — Hard-coding IP addresses

Avoid dynamic addresses such as 172.20.0.5.

Use the service name instead.

### Mistake 5 — Assuming the browser can resolve service names

The browser normally needs localhost, a public hostname, or a reverse-proxy URL.

### Mistake 6 — Connecting every service to every network

Connect services only to the networks they actually need.

---

## 24. Debugging Checklist

When service communication fails:

### Step 1 — Check services

~~~bash
docker compose ps
~~~

### Step 2 — Check networks

~~~bash
docker network ls
~~~

### Step 3 — Inspect the project network

~~~bash
docker network inspect <project>_default
~~~

### Step 4 — Enter the source container

~~~bash
docker compose exec backend sh
~~~

### Step 5 — Verify the hostname

Use the Compose service name.

### Step 6 — Verify the container port

Use the target service's internal listening port.

### Step 7 — Check logs

~~~bash
docker compose logs backend
docker compose logs database
~~~

### Step 8 — Check host port only when the request originates from the host/browser.

---

## 25. Interview Questions

### Q1. How does Docker Compose provide service discovery?

Compose creates a network and Docker DNS allows services to resolve other services by their Compose service names.

### Q2. Why should services use names instead of container IPs?

Container IP addresses can change when containers are recreated. Service names provide stable application-level discovery.

### Q3. Does a database need a published port for the backend to access it?

No. If both services share a Docker network, the backend can use the database service name and internal port.

### Q4. What is the difference between host port and container port?

The host port is where traffic enters through the host. The container port is where the application listens inside the container.

### Q5. Can one Compose service join multiple networks?

Yes. This is useful for connecting a service to different communication zones.

### Q6. Why use multiple networks?

To isolate services and restrict which components can communicate directly.

### Q7. What is an external network?

A Docker network created outside the current Compose project that Compose can attach services to.

### Q8. Can a browser use a Compose service name such as backend?

Normally no. Browser JavaScript runs outside the Docker network and generally needs a host-reachable or reverse-proxy URL.

---

## 26. Quick Revision

~~~text
Compose
   |
   v
Network
   |
   +---- frontend
   +---- backend
   +---- database
   +---- redis
~~~

### Core rules

~~~text
Container -> Container
service-name:container-port

Browser/Host -> Container
localhost:published-host-port

Do not use:
container IP
localhost between services
unnecessary published database ports
~~~

### Important commands

~~~bash
docker compose up -d
docker compose ps
docker compose exec backend sh
docker network ls
docker network inspect <network>
docker compose logs backend
~~~

---

## ⭐ Must Remember

1. **Compose creates a default network for the application.**
2. **Service names are the normal hostnames for service-to-service communication.**
3. **Do not hard-code container IP addresses.**
4. **Container-to-container communication uses the target container port.**
5. **Host/browser access uses the published host port.**
6. **A database usually does not need a published port for backend access.**
7. **Multiple networks can isolate public and private services.**
8. **A browser normally cannot resolve Docker service names.**
9. **Connect services only to the networks they actually need.**
10. **Docker networking provides isolation, but it does not replace application security.**

---

## 🎯 Interview Summary

> Docker Compose creates a network for services in the application. Services can discover each other using stable Compose service names through Docker DNS, so an API can connect to MongoDB using mongo:27017 without knowing the container IP. Published host ports are mainly for traffic coming from the host or browser. Multiple networks can isolate public-facing services from private databases and caches. A good architecture exposes only the ports that need external access and keeps internal services on private networks.
