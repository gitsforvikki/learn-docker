# Lesson 17 — Docker Networking

## 1. Why Docker Networking Matters

Containers are isolated by default, but real applications usually contain multiple services that must communicate.

Example:

    Browser
       │
       ▼
    Frontend container
       │
       ▼
    Backend container
       │
       ▼
    Database container

Docker networking provides connectivity between containers while controlling how they reach each other and the outside world.

## 2. Container Network Mental Model

> **Containers on the same user-defined network can communicate using container/service names.**

## 3. Default Networks

List networks:

    docker network ls

Important built-in networks include bridge, host, and none. User-defined bridge networks provide better service discovery and isolation than relying on the default bridge.

## 4. User-Defined Bridge Network

    docker network create app-network
    docker network inspect app-network

Run containers on it:

    docker run -d --name backend --network app-network node:22-alpine
    docker run -d --name frontend --network app-network nginx

## 5. Container-to-Container Communication

If the backend listens on port 3000, another container on the same network can use:

    http://backend:3000

Do not use localhost:3000. Inside a container, localhost means that same container.

## 6. Container Port vs Host Port

Example:

    docker run -d --name backend --network app-network -p 8080:3000 my-backend

Host port = 8080. Container port = 3000.

From the host use localhost:8080. From another container use backend:3000.

The published host port is normally not needed for container-to-container communication.

## 7. Docker DNS / Service Discovery

Docker provides DNS-based name resolution on user-defined networks.

Examples: backend, db, redis.

Applications can use:

    DATABASE_URL=postgresql://user:password@db:5432/app

Do not hard-code container IP addresses because they can change when containers are recreated.

## 8. Connecting an Existing Container

    docker network connect app-network existing-container
    docker network disconnect app-network existing-container
    docker network inspect app-network

## 9. Network Isolation

You can use separate networks to control communication.

    frontend ── frontend-network ── backend
                                      │
                               backend-network
                                      │
                                      db

A database does not necessarily need to be directly reachable by the frontend.

## 10. EXPOSE vs Networking

A Dockerfile instruction such as EXPOSE 3000 does not publish the port to the host.

Publishing requires:

    docker run -p 8080:3000 ...

For container-to-container communication on the same network, use the application's container port.

## 11. Application Must Listen on the Correct Interface

A server inside a container should normally listen on 0.0.0.0.

Example:

    app.listen(3000, "0.0.0.0");

Avoid binding only to 127.0.0.1 because that may make the service accessible only from inside its own container.

## 12. Host and None Network Modes

Host mode:

    docker run --network host nginx

uses the host's network namespace and changes the usual port isolation model. It is platform-dependent and not the normal choice for most application containers.

None mode:

    docker run --network none alpine

disables normal network connectivity.

For most multi-container applications, use user-defined bridge networks.

## 13. Network Drivers

| Driver | Typical use |
|---|---|
| bridge | Containers on the same Docker host |
| host | Share host network namespace |
| none | No normal networking |
| overlay | Multi-host Docker networking, commonly with Swarm |

For normal development on one machine, user-defined bridge networks are the most important.

## 14. Docker Compose Networking

Compose automatically creates a default application network.

Example:

    services:
      frontend:
        build: ./frontend
      backend:
        build: ./backend
      db:
        image: postgres

Services can communicate using service names:

    frontend → http://backend:3000
    backend  → db:5432

## 15. Compose Example

    services:
      backend:
        build: ./backend
        ports:
          - "3000:3000"

      db:
        image: postgres:17
        environment:
          POSTGRES_PASSWORD: secret

The backend can connect to PostgreSQL using:

    postgresql://postgres:secret@db:5432/postgres

Notice db:5432, not localhost:5432.

## 16. localhost in Different Contexts

This is a major source of Docker bugs.

Browser on the host: localhost:8080 means the host machine.

Backend container: localhost:3000 means the backend container itself.

Backend → database container: db:5432.

Frontend browser code → backend: browser JavaScript runs on the user's machine, not inside the frontend container. Therefore a browser request to http://backend:3000 usually will not work for a normal local browser.

The browser may need http://localhost:3000 while server-side code inside Docker may use http://backend:3000.

## 17. Common Networking Mistakes

### Mistake 1 — Using localhost between containers

Wrong: http://localhost:3000
Use: http://backend:3000

### Mistake 2 — Using host-published ports internally

If backend publishes 8080:3000, another container should normally use backend:3000, not backend:8080.

### Mistake 3 — Using container IPs

Container IPs can change. Use DNS names.

### Mistake 4 — Binding to 127.0.0.1

Use 0.0.0.0 for typical containerized servers.

### Mistake 5 — Assuming EXPOSE publishes a port

It does not.

### Mistake 6 — Forgetting network membership

Containers must share an appropriate network to communicate directly.

## 18. Debugging Docker Networking

Useful commands:

    docker network ls
    docker network inspect app-network
    docker inspect backend

Check listening ports:

    docker exec backend ss -lnt

Test DNS/connectivity from another container:

    docker run --rm --network app-network alpine ping backend

For HTTP testing, a container containing curl can be useful.

## 19. Practical Full-Stack Mental Model

Consider CodeBuddy:

    Browser
       │
       ▼
    Frontend container
       │
       ▼
    Backend container
      │           │
      ▼           ▼
    MongoDB      Redis

Typical internal configuration:

    Backend → mongodb:27017
    Backend → redis:6379

The exact architecture depends on whether frontend requests are browser-side, server-side, or proxied.

## 20. Interview Questions

### Q1. How do containers communicate with each other?

Containers attached to the same appropriate Docker network can communicate using container or service names and container ports.

### Q2. What does localhost mean inside a container?

It refers to the container itself.

### Q3. Why use Docker DNS instead of container IPs?

Container IPs can change. Docker DNS provides stable name-based service discovery.

### Q4. Does EXPOSE publish a port?

No. EXPOSE documents the intended container port. -p publishes a port to the host.

### Q5. What is the difference between 8080:3000 and 3000:3000?

The first number is the host port; the second is the container port.

### Q6. Why should Node.js applications often listen on 0.0.0.0 in Docker?

So the application accepts connections through the container's network interfaces rather than only its loopback interface.

### Q7. What network driver is commonly used for containers on one Docker host?

A user-defined bridge network.

### Q8. What networking does Compose provide by default?

Compose creates an application network that allows services to communicate using service names.

### Q9. Can a container be attached to multiple networks?

Yes. This can be used to control which groups of services can communicate.

## 21. Quick Revision

- Docker networking connects isolated containers.
- User-defined bridge networks are fundamental for multi-container applications.
- Containers on the same network can use names to communicate.
- localhost inside a container means that same container.
- Use backend:3000, not localhost:3000, for container-to-container communication.
- Container-to-container traffic normally uses the container port.
- -p HOST:CONTAINER publishes a port to the host.
- EXPOSE does not publish a port.
- Docker provides DNS-based service discovery.
- Avoid hard-coding container IP addresses.
- Typical servers should listen on 0.0.0.0.
- Compose automatically creates a network for services.
- Separate networks can provide better isolation.
- Browser-side requests and server-side container requests can require different hostnames.

## Must Remember

> **Inside a container, localhost means that container.**

> **Container-to-container communication uses Docker network names + container ports.**

> **-p HOST:CONTAINER is for host-to-container access.**

> **EXPOSE does not publish a port.**

> **Use service names, not container IPs.**

## Interview Summary

Host → published host port
Container → service/container name + container port

Example:

    Browser: http://localhost:8080
    Backend container: http://db:5432
    Backend container: http://redis:6379