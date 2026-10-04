# Lesson 31 — Docker Health Checks

## 1. What Is a Docker Health Check?

A health check tests whether the application inside a container is actually working.

A running container is not automatically a healthy container.

~~~text
Container running
      ↓
Is the application responding?
      ↓
Health check
      ↓
healthy / unhealthy
~~~

## 2. Running vs Healthy

A Node.js process can start successfully while the application cannot connect to MongoDB. Docker may still show the container as Up.

A health check can detect whether the application is actually ready to serve requests.

## 3. Why Health Checks Matter

Health checks help with:

- Detecting broken applications
- Startup/readiness decisions
- Debugging
- Load balancing
- Deployment verification
- Automated recovery/orchestration

## 4. Dockerfile HEALTHCHECK

Example:

~~~dockerfile
FROM node:22-alpine

WORKDIR /app

COPY package*.json ./
RUN npm ci --omit=dev

COPY . .

EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD wget --spider -q http://localhost:3000/health || exit 1

CMD ["node", "server.js"]
~~~

Docker runs the command periodically.

## 5. Health Check Exit Codes

The command indicates success or failure through its exit code.

~~~text
exit 0 → healthy
non-zero → failed check
~~~

Example:

~~~bash
wget --spider -q http://localhost:3000/health
~~~

## 6. Important HEALTHCHECK Options

Common options:

~~~text
--interval
--timeout
--start-period
--retries
~~~

- interval: how often Docker runs the check
- timeout: maximum time allowed for one check
- start-period: startup grace period
- retries: consecutive failures required before unhealthy

## 7. Health Status

A container can have states such as:

~~~text
starting
healthy
unhealthy
~~~

Commands:

~~~bash
docker ps
docker inspect <container>
docker inspect -f '{{.State.Health.Status}}' <container>
~~~

## 8. HTTP Health Endpoint

A good application often exposes a lightweight endpoint.

Express example:

~~~javascript
app.get("/health", (req, res) => {
  res.status(200).json({
    status: "ok"
  });
});
~~~

The endpoint should be fast and should not perform unnecessarily expensive work.

## 9. Liveness vs Readiness

### Liveness

Is the application process functioning?

### Readiness

Is the application ready to receive traffic?

~~~text
Liveness  → should this process continue running?
Readiness → should this instance receive traffic?
~~~

Docker HEALTHCHECK is useful for health detection. Kubernetes provides separate liveness and readiness probes.

## 10. Simple vs Dependency-Aware Checks

A simple check:

~~~text
GET /health
   ↓
200 OK
~~~

A dependency-aware check may verify important dependencies:

~~~text
GET /health
   ↓
Application
   ├── Database
   └── Required service
~~~

Do not make health checks unnecessarily expensive.

## 11. Startup and Readiness

Some applications need time to start because of:

- Database initialization
- Migrations
- Cache loading
- Connection establishment
- Large application startup

Use start-period appropriately.

Example:

~~~dockerfile
HEALTHCHECK --start-period=30s --interval=10s --timeout=3s --retries=5 \
  CMD wget --spider -q http://localhost:3000/health || exit 1
~~~

## 12. Compose Health Checks

Example:

~~~yaml
services:
  backend:
    image: codebuddy-backend:1.0
    healthcheck:
      test: ["CMD", "wget", "--spider", "-q", "http://localhost:3000/health"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 10s
~~~

Compose can display the service health state.

## 13. depends_on and Health

Compose can use health conditions when startup ordering depends on a service becoming healthy.

~~~yaml
services:
  backend:
    image: codebuddy-backend:1.0
    depends_on:
      mongo:
        condition: service_healthy

  mongo:
    image: mongo:8
    healthcheck:
      test: ["CMD", "mongosh", "--quiet", "--eval", "db.adminCommand('ping').ok"]
      interval: 10s
      timeout: 5s
      retries: 5
~~~

Important:

**depends_on controls startup ordering/conditions; it does not make your application automatically resilient to later dependency failures.**

Applications should still handle connection failures and retries properly.

## 14. Health Check Command Must Exist

The command used by a health check must exist inside the image.

For example, a minimal Alpine image may not contain curl.

If the health check uses curl but curl is not installed, the check fails.

Use a tool that exists in the image, such as wget, or install the required tool deliberately.

## 15. Health Checks and Restart Policies

A health check reports health.

A restart policy controls whether Docker restarts a stopped container.

~~~text
HEALTHCHECK
     ↓
healthy / unhealthy

restart policy
     ↓
restart behavior
~~~

Do not assume that unhealthy automatically means Docker will restart the container in every normal Docker setup.

## 16. Health Checks and Load Balancing

A load balancer or orchestrator can use health information to avoid sending traffic to unhealthy instances when configured to do so.

~~~text
             Load Balancer
                 ↓
       ┌─────────┼─────────┐
       ↓         ↓         ↓
    healthy   unhealthy   healthy
       ↓                   ↓
    traffic              traffic
~~~

Exact behavior depends on the platform.

## 17. Health Checks in CodeBuddy

Useful architecture:

~~~text
Nginx
  ↓
Backend containers
  ↓
/health
  ↓
Database/dependencies as appropriate
~~~

Example:

~~~text
GET /health
~~~

Expected:

~~~json
{
  "status": "ok"
}
~~~

## 18. Deployment Verification

After deployment:

~~~bash
docker compose ps
docker inspect <backend-container>
docker compose logs backend
~~~

Then test:

~~~bash
curl http://localhost:3000/health
~~~

Production verification:

~~~text
Deploy
  ↓
Container starts
  ↓
Health check passes
  ↓
Application smoke test
  ↓
Traffic enabled
~~~

## 19. Common Mistakes

1. Checking only whether the process exists.
2. Using a health-check command that is missing from the image.
3. Making health checks too expensive.
4. Using a start period that is too short.
5. Confusing health status with restart behavior.
6. Returning sensitive information from health endpoints.

## 20. Best Practices

1. Provide a lightweight health endpoint for long-running applications.
2. Keep checks fast and deterministic.
3. Use realistic intervals and timeouts.
4. Give slow-starting applications an appropriate start period.
5. Ensure the health-check command exists in the image.
6. Keep sensitive details out of health responses.
7. Distinguish liveness from readiness.
8. Make applications resilient to dependency failures.
9. Verify health after deployment.
10. Monitor unhealthy containers and investigate the root cause.

# Interview Questions

### 1. What is a Docker health check?

A command Docker periodically executes to determine whether the application inside a container is healthy.

### 2. What is the difference between running and healthy?

Running means the container's main process is active; healthy means the configured health check is succeeding.

### 3. What does HEALTHCHECK do?

It defines the command and timing Docker uses to evaluate container health.

### 4. What are interval, timeout, start-period, and retries?

They control check frequency, maximum check duration, startup grace period, and consecutive failures required to mark the container unhealthy.

### 5. What is liveness vs readiness?

Liveness asks whether the application should remain running; readiness asks whether it should receive traffic.

### 6. Does an unhealthy container automatically restart?

Not necessarily. Health status and restart policy are separate mechanisms.

### 7. Why can a health check fail even when the application works?

The check command may be missing, the endpoint may be wrong, the startup period may be too short, or the application may not be listening correctly.

### 8. Why should health checks be lightweight?

Because they run repeatedly and an expensive check can create unnecessary load.

### 9. How does Compose use health checks with depends_on?

Compose can wait for a dependency's configured health condition before starting a dependent service.

### 10. Why are health checks important in production?

They help detect broken instances and support reliable deployment, monitoring, routing, and orchestration.

# Quick Revision

~~~text
Container starts
     ↓
Health check runs
     ↓
starting
     ↓
healthy / unhealthy
     ↓
Monitoring / deployment / routing decisions
~~~

Important commands:

~~~bash
docker ps
docker inspect <container>
docker inspect -f '{{.State.Health.Status}}' <container>
docker compose ps
docker compose logs <service>
~~~

Important Dockerfile syntax:

~~~dockerfile
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD wget --spider -q http://localhost:3000/health || exit 1
~~~

# ⭐ Must Remember

1. **Running does not automatically mean healthy.**
2. **HEALTHCHECK tells Docker how to test the application.**
3. **Health checks use command exit codes to determine success or failure.**
4. **interval, timeout, start-period, and retries control health-check behavior.**
5. **The health-check command must exist inside the image.**
6. **Keep health checks lightweight.**
7. **Liveness and readiness are different concepts.**
8. **Health status and restart policies are separate mechanisms.**
9. **Use Compose health checks and dependency conditions carefully.**
10. **A production deployment should verify application health after starting.**
