# Lesson 28 — Reverse Proxy with Docker

## 1. What Is a Reverse Proxy?

A reverse proxy receives client requests and forwards them to one or more internal application services.

~~~text
Browser
   ↓
Internet
   ↓
Reverse Proxy
   ↓
Application
~~~

Common reverse proxies:
- Nginx
- Caddy
- Traefik

## 2. Why Use a Reverse Proxy?

A reverse proxy can provide:
- HTTPS/TLS termination
- Domain and path routing
- Load balancing
- Request forwarding
- Static file serving
- Compression
- Security headers
- Centralized access logging

Typical production architecture:

~~~text
Internet
    ↓
HTTPS :443
    ↓
Reverse Proxy
    ├── Frontend
    └── Backend API
~~~

## 3. Reverse Proxy vs Forward Proxy

### Forward proxy

Represents the client:

~~~text
Client → Forward Proxy → Internet
~~~

### Reverse proxy

Represents the servers:

~~~text
Client → Reverse Proxy → Application
~~~

**Interview shortcut:** Forward proxy represents/protects clients; reverse proxy represents/protects servers.

## 4. Why Docker Applications Commonly Use a Reverse Proxy

Suppose:

~~~text
Frontend → :3000
Backend  → :5000
~~~

Instead of exposing both publicly:

~~~text
Internet
   ↓
Nginx :80/:443
   ├── frontend:3000
   └── backend:5000
~~~

Only the reverse proxy needs public HTTP/HTTPS access. Application containers can remain on a private Docker network.

## 5. Docker Network Architecture

~~~text
                Internet
                    ↓
              Nginx :443
                    ↓
             Docker Network
              /           \
             ↓             ↓
        Frontend        Backend
                           ↓
                        MongoDB
~~~

The proxy reaches containers through Docker DNS/service names:

~~~text
http://frontend:3000
http://backend:5000
~~~

Do not use localhost for these connections. Inside the Nginx container, localhost means the Nginx container itself.

## 6. Basic Nginx Configuration

~~~nginx
server {
    listen 80;

    location / {
        proxy_pass http://frontend:3000;
    }

    location /api/ {
        proxy_pass http://backend:5000;
    }
}
~~~

Conceptually:

~~~text
GET /
   ↓
frontend:3000

GET /api/users
   ↓
backend:5000
~~~

## 7. Forwarded Headers

Useful proxy headers:

~~~nginx
location /api/ {
    proxy_pass http://backend:5000;

    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
~~~

Important headers:
- Host
- X-Real-IP
- X-Forwarded-For
- X-Forwarded-Proto

Applications should only trust forwarded headers when the proxy/network is trusted.

## 8. Docker Compose Example

Project structure:

~~~text
project/
├── compose.yaml
└── nginx/
    └── nginx.conf
~~~

Example:

~~~yaml
services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/conf.d/default.conf:ro
    depends_on:
      - frontend
      - backend
    networks:
      - app-network

  frontend:
    image: username/codebuddy-frontend:latest
    expose:
      - "3000"
    networks:
      - app-network

  backend:
    image: username/codebuddy-backend:latest
    expose:
      - "5000"
    networks:
      - app-network

networks:
  app-network:
~~~

The public entry point is Nginx. Frontend and backend are internal services.

## 9. ports vs expose

### ports

Publishes a container port through the host:

~~~yaml
ports:
  - "80:80"
~~~

### expose

Documents/declares a port intended for internal container communication without publishing it to the host:

~~~yaml
expose:
  - "3000"
~~~

Important: containers on the same Docker network can communicate without an expose declaration.

## 10. Path-Based Routing

One domain can serve different services:

~~~text
https://example.com/
        ↓
Frontend

https://example.com/api/
        ↓
Backend
~~~

Example:

~~~nginx
location /api/ {
    proxy_pass http://backend:5000;
}

location / {
    proxy_pass http://frontend:3000;
}
~~~

## 11. Subdomain-Based Routing

Routing can also use hostnames:

~~~text
app.example.com → Frontend
api.example.com → Backend
~~~

Example:

~~~nginx
server {
    server_name app.example.com;

    location / {
        proxy_pass http://frontend:3000;
    }
}

server {
    server_name api.example.com;

    location / {
        proxy_pass http://backend:5000;
    }
}
~~~

## 12. HTTPS and TLS Termination

Common architecture:

~~~text
Browser
   ↓
HTTPS :443
   ↓
Nginx
   ↓
Internal Docker network
   ↓
Backend
~~~

The reverse proxy can terminate TLS and manage the public certificate. Internal application traffic can then use the private Docker network.

## 13. HTTP to HTTPS Redirect

Example:

~~~nginx
server {
    listen 80;
    server_name example.com;

    return 301 https://$host$request_uri;
}
~~~

Conceptually:

~~~text
HTTP :80
   ↓
301 redirect
   ↓
HTTPS :443
~~~

## 14. Static Files and React SPA Routing

Nginx can serve static files directly.

React SPAs commonly need a fallback:

~~~nginx
server {
    listen 80;
    root /usr/share/nginx/html;

    location / {
        try_files $uri $uri/ /index.html;
    }
}
~~~

Without the fallback, directly opening a client-side route such as /dashboard can produce a 404 from Nginx.

## 15. Reverse Proxy and Next.js

Next.js can run as a Node server behind a reverse proxy:

~~~text
Internet
   ↓
Nginx
   ↓
Next.js container
   ↓
Node.js
~~~

Example:

~~~nginx
location / {
    proxy_pass http://nextjs:3000;
}
~~~

## 16. WebSockets and Socket.IO

CodeBuddy uses Socket.IO, so protocol upgrade support matters.

Typical Nginx configuration:

~~~nginx
location /socket.io/ {
    proxy_pass http://backend:5000;

    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";

    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
}
~~~

Without correct upgrade handling, WebSocket connections can fail or fall back unexpectedly.

## 17. Load Balancing

A reverse proxy can distribute requests across multiple backend instances:

~~~text
                 Nginx
                   ↓
          ┌────────┼────────┐
          ↓        ↓        ↓
       Backend1 Backend2 Backend3
~~~

Example:

~~~nginx
upstream backend_servers {
    server backend1:5000;
    server backend2:5000;
    server backend3:5000;
}

server {
    listen 80;

    location /api/ {
        proxy_pass http://backend_servers;
    }
}
~~~

This provides a basic load-balancing pattern.

## 18. Health and Availability

A reverse proxy can participate in routing to healthy upstreams when health-aware load balancing is configured.

However:

**A reverse proxy is not a complete orchestration system.**

Kubernetes provides broader capabilities such as scheduling, service discovery, rolling updates, and self-healing.

## 19. Nginx Configuration Validation

Validate before reload:

~~~bash
nginx -t
~~~

When Nginx runs in Docker:

~~~bash
docker exec nginx nginx -t
~~~

Reload after a valid configuration:

~~~bash
docker exec nginx nginx -s reload
~~~

Or recreate the Compose service when configuration is mounted into the container.

## 20. Debugging Reverse Proxy Problems

Check services:

~~~bash
docker compose ps
~~~

Check Nginx logs:

~~~bash
docker compose logs nginx
~~~

Check configuration:

~~~bash
docker exec nginx nginx -t
~~~

Test the backend from the proxy network:

~~~bash
docker exec nginx wget -qO- http://backend:5000/health
~~~

Inspect the network:

~~~bash
docker network inspect <network>
~~~

## 21. Common Errors

### 502 Bad Gateway

Usually means the proxy cannot successfully reach the upstream.

Check:
- backend is running
- correct service name
- correct internal port
- Docker network
- backend health
- application binding

### 404

Possible causes:
- incorrect Nginx location
- missing SPA fallback
- wrong path
- backend route does not exist

### WebSocket failure

Check:
- Upgrade header
- Connection header
- HTTP/1.1
- Socket.IO path
- backend/network availability

### Connection refused

Check:
- application is listening
- application binds to 0.0.0.0
- correct container port
- service is running

## 22. Nginx + Docker + CodeBuddy

Realistic architecture:

~~~text
                     Internet
                        ↓
                  HTTPS :443
                        ↓
                 Nginx Container
                    /        \
                   ↓          ↓
              Frontend      Backend
                                ↓
                             MongoDB
~~~

All containers can share a private Docker network.

Only Nginx needs to publish public HTTP/HTTPS ports.

The backend can remain internal while Nginx forwards /api and Socket.IO traffic.

## 23. Production Security

Recommended controls:

1. Publish only required ports.
2. Keep backend/database ports private when possible.
3. Use HTTPS.
4. Validate and protect forwarded headers.
5. Keep secrets outside images.
6. Use security headers where appropriate.
7. Limit unnecessary public endpoints.
8. Keep Nginx and application images updated.
9. Monitor access and error logs.
10. Apply rate limiting where appropriate.

## 24. Common Mistakes

### Mistake 1 — Using localhost as upstream

Wrong:

~~~nginx
proxy_pass http://localhost:5000;
~~~

Inside the Nginx container this points to Nginx itself.

Correct:

~~~nginx
proxy_pass http://backend:5000;
~~~

### Mistake 2 — Publishing every service

Avoid publishing backend or MongoDB ports publicly when those services do not need public access.

### Mistake 3 — Forgetting WebSocket headers

This can break Socket.IO/WebSocket traffic.

### Mistake 4 — Wrong internal port

The proxy must use the application's container/listening port, not necessarily the host port.

### Mistake 5 — No SPA fallback

React client-side routes can return 404 when opened directly.

## 25. Best Practices

1. Use a reverse proxy as the public entry point.
2. Keep application containers on private Docker networks.
3. Publish only required proxy ports.
4. Use service names for upstreams.
5. Terminate HTTPS at the proxy when appropriate.
6. Configure WebSocket upgrades for Socket.IO/WebSockets.
7. Add health checks.
8. Validate Nginx configuration before deployment.
9. Monitor access and error logs.
10. Use immutable application images.
11. Keep proxy configuration version-controlled.
12. Do not expose databases publicly without a strong reason.
13. Use load balancing when multiple application instances are required.

# Interview Questions

### 1. What is a reverse proxy?

A server that receives client requests and forwards them to internal application servers.

### 2. Reverse proxy vs forward proxy?

A forward proxy represents clients; a reverse proxy represents servers.

### 3. Why use Nginx with Docker?

It provides a stable public entry point for routing, HTTPS, static files, and forwarding traffic to internal containers.

### 4. Why should Nginx use backend:5000 instead of localhost:5000?

Because Nginx and the backend are separate containers; Docker DNS resolves the backend service name.

### 5. What does ports do in Compose?

It publishes a container port through the host.

### 6. What does expose do?

It documents/declares an intended container port for internal use without publishing it to the host.

### 7. What causes a 502 Bad Gateway?

Commonly the reverse proxy cannot successfully connect to the configured upstream.

### 8. Why are WebSocket headers important?

They allow the HTTP connection upgrade required for WebSocket communication.

### 9. How can one domain serve frontend and backend?

Use path-based routing such as / for frontend and /api/ for backend.

### 10. Can a reverse proxy load-balance containers?

Yes. It can distribute requests across multiple upstream instances.

# Quick Revision

~~~text
Internet
   ↓
DNS
   ↓
HTTPS
   ↓
Reverse Proxy
   ↓
Docker Network
   ├── Frontend
   ├── Backend
   └── Other services
           ↓
        Database
~~~

Core commands:

~~~bash
docker compose ps
docker compose logs nginx
docker exec nginx nginx -t
docker network inspect <network>
~~~

Core concept:

~~~text
Public request
     ↓
Nginx
     ↓
Docker DNS/service name
     ↓
Internal container
~~~

# ⭐ Must Remember

1. **Reverse proxy = public entry point in front of application services.**
2. **Nginx inside Docker should use service names, not localhost, for upstream containers.**
3. **Publish only the ports that actually need public access.**
4. **Use Docker networks for internal service communication.**
5. **HTTPS commonly terminates at the reverse proxy.**
6. **Path routing can send / to frontend and /api to backend.**
7. **React SPAs commonly need an index.html fallback.**
8. **Socket.IO/WebSockets require connection-upgrade handling.**
9. **502 usually means an upstream connectivity/configuration problem.**
10. **A reverse proxy can load-balance multiple application instances.**
11. **For CodeBuddy, Nginx can sit in front of React + Node/Express + Socket.IO while MongoDB remains internal.**
