# Lesson 27 — Docker Deployment

## 1. What Is Docker Deployment?

Docker deployment means running a tested Docker image in a target environment so the application becomes available to users.

~~~text
Developer
   ↓
GitHub
   ↓
CI/CD
   ↓
Build Docker image
   ↓
Registry
   ↓
Deployment server
   ↓
Container
   ↓
Users
~~~

**Key principle:** Build the image once, then deploy that same image.

## 2. Where Can Docker Containers Be Deployed?

Docker containers can run on Linux VMs, cloud servers, managed container platforms, Docker Compose environments, and Kubernetes.

Learning progression:

~~~text
Cloud VM → Docker → Container
Cloud VM → Docker Compose → Multiple containers
Kubernetes → Pods → Services / Ingress
~~~

## 3. Production Architecture

~~~text
Internet
   ↓
Server
   ↓
Docker
   ├── Frontend
   └── Backend
          ↓
       Database
~~~

| Component | Responsibility |
|---|---|
| GitHub | Source code |
| CI/CD | Build, test, package, deliver |
| Registry | Store images |
| Server | Run containers |
| Docker | Container runtime |
| Database | Persistent data |
| Reverse proxy | HTTPS and routing |
| DNS | Domain → server |

## 4. Pull and Run the Production Image

Suppose CI pushed:

~~~text
username/codebuddy-backend:a81ecfe
~~~

On the server:

~~~bash
docker login
docker pull username/codebuddy-backend:a81ecfe

docker run -d \
  --name codebuddy-backend \
  -p 3000:3000 \
  username/codebuddy-backend:a81ecfe
~~~

The server runs the exact image produced by CI/CD.

## 5. Runtime Configuration

Do not bake production secrets into the image.

~~~bash
docker run -d \
  --name codebuddy-backend \
  -p 3000:3000 \
  --env-file .env.production \
  username/codebuddy-backend:a81ecfe
~~~

The image remains unchanged while environment-specific configuration changes at runtime.

## 6. Docker Compose Deployment

For multiple containers:

~~~yaml
services:
  backend:
    image: username/codebuddy-backend:a81ecfe
    restart: unless-stopped
    env_file:
      - .env.production
    ports:
      - "3000:3000"

  frontend:
    image: username/codebuddy-frontend:a81ecfe
    restart: unless-stopped
    ports:
      - "80:80"
~~~

Deploy:

~~~bash
docker compose pull
docker compose up -d
docker compose ps
docker compose logs
~~~

Compose is useful for small multi-container deployments. Kubernetes provides more advanced orchestration.

## 7. Host Port vs Container Port

~~~yaml
ports:
  - "80:3000"
~~~

means:

~~~text
Server port 80
      ↓
Container port 3000
~~~

The application still listens on port 3000 inside the container.

## 8. Production Networking

Containers communicate through Docker networks using service/container names and internal ports.

Example:

~~~text
mongodb://mongo:27017/codebuddy
~~~

Do not use localhost for container-to-container communication.

**Inside a container, localhost means that same container.**

## 9. Reverse Proxy

A common production architecture:

~~~text
Internet
   ↓
HTTPS :443
   ↓
Reverse Proxy
   ├── /        → Frontend
   └── /api     → Backend
~~~

Common reverse proxies include Nginx, Caddy, and Traefik.

They can handle TLS, routing, headers, compression, static assets, and request forwarding.

Lesson 28 covers reverse proxies in detail.

## 10. Domain, DNS and HTTPS

Typical flow:

~~~text
codebuddy.example.com
        ↓
DNS
        ↓
Server IP
        ↓
Reverse Proxy
        ↓
Docker containers
~~~

Docker does not provide a public domain name. DNS and domain registration are separate services.

Production applications should use HTTPS. TLS can terminate at the reverse proxy while application traffic continues over the internal Docker network.

## 11. Restart Policies

Useful restart policies:

~~~text
no
always
on-failure
unless-stopped
~~~

Compose example:

~~~yaml
restart: unless-stopped
~~~

Restart policies help recover from container or host restarts, but they are not a replacement for monitoring or orchestration.

## 12. Updating a Deployment

Current:

~~~text
codebuddy-backend:a81ecfe
~~~

New:

~~~text
codebuddy-backend:b7c21d4
~~~

Pull:

~~~bash
docker pull username/codebuddy-backend:b7c21d4
~~~

With Compose:

~~~bash
docker compose pull
docker compose up -d
~~~

## 13. Rollback

If the new release fails:

~~~text
Current  → b7c21d4
Previous → a81ecfe
~~~

Deploy the previous known-good image.

With Compose, restore the previous image reference and run:

~~~bash
docker compose up -d
~~~

Immutable version/SHA tags make rollback predictable.

## 14. Zero-Downtime Deployment

A simple stop-old/start-new deployment can cause downtime.

More advanced strategies:

- Rolling deployment
- Blue-green deployment
- Canary deployment

Blue-green:

~~~text
Production → Blue
               ↓
           New version → Green
               ↓
          Test Green
               ↓
        Switch traffic
~~~

Canary:

~~~text
95% → old version
5%  → new version
~~~

These strategies are easier with load balancers and orchestration platforms.

## 15. Health Checks

A deployment should verify application readiness.

Example:

~~~yaml
healthcheck:
  test: ["CMD", "wget", "--spider", "-q", "http://localhost:3000/health"]
  interval: 30s
  timeout: 5s
  retries: 3
  start_period: 10s
~~~

A health check answers:

**Is the application actually ready to serve traffic?**

## 16. Logs and Debugging

Useful commands:

~~~bash
docker ps
docker logs codebuddy-backend
docker inspect codebuddy-backend
docker stats
~~~

Compose:

~~~bash
docker compose ps
docker compose logs backend
docker compose logs -f backend
~~~

Debugging flow:

~~~text
Container problem
      ↓
docker ps
      ↓
docker logs
      ↓
docker inspect
      ↓
Check network
      ↓
Check environment
      ↓
Check health
~~~

## 17. Persistent Data

Do not depend on a container writable layer for important data.

~~~text
Application container
       ↓
Temporary runtime state

Database
       ↓
Persistent storage
~~~

Use managed databases, Docker volumes where appropriate, backups, and restore testing.

For CodeBuddy, MongoDB data must survive container recreation.

## 18. Production Security

Important controls:

- Keep secrets out of images.
- Use least-privilege registry credentials.
- Run containers as non-root when possible.
- Expose only required ports.
- Use HTTPS.
- Keep Docker and host packages updated.
- Scan images.
- Restrict SSH access.
- Use firewall rules.
- Monitor logs and resources.
- Keep databases off the public internet whenever possible.

## 19. Deployment with Jenkins

Typical flow:

~~~text
GitHub
   ↓
Jenkins
   ↓
Test
   ↓
Build
   ↓
Scan
   ↓
Push image
   ↓
Deployment mechanism
   ↓
Server
   ↓
docker compose pull
   ↓
docker compose up -d
   ↓
Health check
~~~

Jenkins should deploy a known image version rather than rebuilding application source on production.

## 20. CodeBuddy Production Flow

~~~text
Frontend repo ──┐
                ├──→ Jenkins → Build → Scan → Registry
Backend repo ───┘
                                      ↓
                              Production server
                                      ↓
                              Docker Compose
                           ┌──────────┴──────────┐
                           ↓                     ↓
                      Frontend              Backend
                                                  ↓
                                               MongoDB
~~~

A reverse proxy can sit in front of the frontend and backend.

Runtime secrets remain outside the images.

## 21. Deployment Checklist

Before deployment:

- [ ] Tests pass
- [ ] Image builds successfully
- [ ] Image has version/SHA tag
- [ ] Image is scanned
- [ ] Image is pushed to registry
- [ ] Production configuration is ready
- [ ] Database/storage is ready
- [ ] Firewall rules are configured
- [ ] HTTPS/reverse proxy is configured
- [ ] Health check exists
- [ ] Previous image is known for rollback

After deployment:

- [ ] Containers are running
- [ ] Health checks pass
- [ ] Application is reachable
- [ ] Logs look healthy
- [ ] Database connectivity works
- [ ] Critical user flow works
- [ ] Resource usage is reasonable

## 22. Common Mistakes

1. Rebuilding the image on production instead of pulling the tested artifact.
2. Using latest as the only production identifier.
3. Storing secrets inside the image.
4. Exposing databases publicly.
5. Using localhost for container-to-container communication.
6. Deploying without health checks.
7. Having no rollback plan.
8. Storing important database data only in the container filesystem.

## 23. Best Practices

1. Build images in CI/CD.
2. Push images to a registry.
3. Deploy immutable version/SHA tags.
4. Keep production secrets outside images.
5. Use Compose for small multi-container deployments.
6. Use a reverse proxy for public applications.
7. Enable HTTPS.
8. Configure restart policies.
9. Add health checks.
10. Monitor logs and resources.
11. Keep databases persistent and backed up.
12. Maintain a rollback strategy.
13. Expose only necessary ports.
14. Use least-privilege credentials.
15. Move to Kubernetes when orchestration requirements justify it.

# Interview Questions

### 1. What is Docker deployment?

Running a tested Docker image in a target environment so users can access the application.

### 2. Why should production pull an image instead of building it?

The registry contains the tested artifact produced by CI/CD, improving reproducibility.

### 3. What is the role of Docker Compose in deployment?

It declaratively defines and manages multiple related containers.

### 4. Why use a reverse proxy?

For HTTPS termination, routing, headers, static assets, and forwarding requests to application containers.

### 5. Why are immutable image tags important?

They make deployments traceable and rollbacks predictable.

### 6. How do containers communicate with each other?

Through a Docker network using service/container names and internal ports.

### 7. Why is localhost usually wrong for container-to-container communication?

Because localhost refers to the current container, not another container.

### 8. How do you rollback a Docker deployment?

Deploy the previous known-good image tag or digest.

### 9. What is a restart policy?

A Docker rule controlling whether a container should be restarted after stopping or host restart.

### 10. Why should database data not be stored only inside a container?

Container writable storage is not an appropriate persistent data strategy; databases need durable storage and backups.

# Quick Revision

~~~text
Git
 ↓
Jenkins / CI/CD
 ↓
Test
 ↓
Build image
 ↓
Scan
 ↓
Push registry
 ↓
Production server
 ↓
Pull image
 ↓
Compose / Docker
 ↓
Health check
 ↓
Users
~~~

Production mental model:

~~~text
Domain
  ↓
DNS
  ↓
Server
  ↓
Reverse Proxy
  ↓
Docker Network
 ├── Frontend
 └── Backend
        ↓
     Database
        ↓
   Persistent Storage
~~~

# ⭐ Must Remember

1. **Production should run the image produced by CI/CD.**
2. **Do not rebuild the application image directly on production.**
3. **Use version/SHA tags instead of relying on latest.**
4. **Keep environment-specific configuration outside the image.**
5. **Use Docker Compose for small multi-container deployments.**
6. **Use service names for container-to-container communication.**
7. **Use a reverse proxy and HTTPS for public production applications.**
8. **Health checks tell you whether the application is actually ready.**
9. **Persistent databases need durable storage and backups.**
10. **Always keep a known-good image for rollback.**
11. **Jenkins can build → scan → push → deploy the exact image.**
12. **Kubernetes becomes useful when deployment needs grow beyond simple Docker/Compose management.**
