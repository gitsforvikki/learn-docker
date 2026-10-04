# Lesson 32 — Docker Logging & Debugging

## 1. Why Logging and Debugging Matter

A container can be running while the application inside it is failing.

~~~text
Is the container running?
        ↓
Is the process running?
        ↓
Are logs showing an error?
        ↓
Can the service reach dependencies?
        ↓
Is networking correct?
        ↓
Are ports/configuration correct?
~~~

Good logs and a systematic debugging process are essential in development and production.

## 2. Docker Logs

The most important command:

~~~bash
docker logs <container>
~~~

Useful forms:

~~~bash
docker logs backend
docker logs -f backend
docker logs --tail 100 backend
docker logs -t backend
docker logs -f --tail 100 -t backend
~~~

## 3. Docker Compose Logs

~~~bash
docker compose logs
docker compose logs backend
docker compose logs -f backend
docker compose logs -f frontend backend
docker compose logs --tail 100 backend
~~~

## 4. stdout and stderr

Docker normally captures output written to standard output and standard error.

~~~text
Application
   ├── stdout ──┐
   └── stderr ──┤
                ↓
          Docker logging
                ↓
          docker logs
~~~

For containerized applications, logging to stdout/stderr is usually preferable to hiding logs only inside container files.

## 5. Why Not Store Logs Only Inside the Container?

Containers are replaceable. If a container is deleted, logs stored only inside it may disappear.

For production, logs should generally be collected by a centralized or durable logging system.

## 6. Logging Drivers

Docker supports logging drivers such as:

- json-file
- local
- syslog
- journald
- fluentd
- gelf
- awslogs
- splunk

Check the default driver:

~~~bash
docker info --format '{{.LoggingDriver}}'
~~~

Inspect a container:

~~~bash
docker inspect <container>
~~~

## 7. Log Rotation

Logs can consume disk space.

Example:

~~~bash
docker run -d \\
  --name backend \\
  --log-opt max-size=10m \\
  --log-opt max-file=3 \\
  my-backend:1.0
~~~

Plan retention based on operational requirements.

## 8. Application Logging

A Node.js application can log to stdout/stderr:

~~~javascript
console.log("Server started");
console.error("Database connection failed");
~~~

Production structured logging can provide:

- Log levels
- JSON logs
- Request IDs
- Timestamps
- Metadata
- Error context

Never log passwords, tokens, cookies, or other sensitive information.

## 9. Log Levels

Common levels:

~~~text
debug
info
warn
error
~~~

A production strategy can control the level through configuration:

~~~text
LOG_LEVEL=info
~~~

## 10. Inspecting Containers

Start with:

~~~bash
docker ps
docker ps -a
docker inspect <container>
~~~

Inspection can reveal:

- Environment
- Ports
- Mounts
- Networks
- Entrypoint
- Command
- Health status
- Resource settings

## 11. Entering a Running Container

Use:

~~~bash
docker exec -it <container> sh
~~~

If Bash exists:

~~~bash
docker exec -it <container> bash
~~~

Minimal images such as Alpine often contain sh but not Bash.

## 12. Run a Command Without Opening a Shell

~~~bash
docker exec backend env
docker exec backend ls -la /app
docker exec backend ps
~~~

Available commands depend on the image.

## 13. Inspecting Processes

~~~bash
docker top backend
~~~

This helps determine whether the expected application process is actually running.

## 14. Port Debugging

Check published ports:

~~~bash
docker ps
docker port backend
~~~

Remember:

~~~text
Host port ≠ container port
~~~

The application must listen on the expected container port.

## 15. The 0.0.0.0 Problem

A server inside a container should generally bind to:

~~~text
0.0.0.0
~~~

Example:

~~~javascript
app.listen(5000, "0.0.0.0");
~~~

Binding only to 127.0.0.1 can prevent requests arriving through the container network or published port.

## 16. Network Debugging

List networks:

~~~bash
docker network ls
~~~

Inspect a network:

~~~bash
docker network inspect <network>
~~~

Test another service using its Docker DNS name:

~~~bash
docker exec nginx wget -qO- http://backend:5000/health
~~~

Do not use localhost when you mean another container.

## 17. DNS Debugging

Docker user-defined networks provide service/container name resolution.

If backend connects to database, use:

~~~text
database:27017
~~~

rather than a hard-coded container IP.

If name resolution fails, inspect network membership and service configuration.

## 18. Environment Variable Debugging

Inspect environment variables:

~~~bash
docker exec backend env
~~~

Or:

~~~bash
docker inspect backend
~~~

Check for:

- Missing variables
- Wrong variable names
- Wrong values
- Incorrect environment files
- Build-time vs runtime confusion

Never expose secrets in logs or screenshots.

## 19. Volume and Mount Debugging

Inspect mounts:

~~~bash
docker inspect backend
~~~

Typical problems:

- Wrong host path
- Wrong container path
- Permission errors
- Bind mount hiding image content
- Missing named volume

## 20. Permission Problems

Check the current user:

~~~bash
docker exec backend id
~~~

Inspect files:

~~~bash
docker exec backend ls -la /app
~~~

Common cause:

~~~text
Host files
   ↓
bind mount
   ↓
container user
   ↓
permission denied
~~~

This is especially common when containers run as non-root users.

## 21. Container Won't Start

Start with:

~~~bash
docker ps -a
docker logs <container>
docker inspect <container>
~~~

Typical causes:

- Invalid command
- Missing environment variable
- Application crash
- Port/configuration error
- Missing file
- Permission issue

## 22. Container Starts Then Exits

The container stays alive only while its main process is running.

Check:

~~~bash
docker ps -a
docker logs <container>
docker inspect <container>
~~~

Common causes:

- Application finished normally
- Incorrect CMD/ENTRYPOINT
- Runtime exception
- Missing configuration
- Startup script failure

## 23. Image Debugging

~~~bash
docker images
docker image inspect <image>
docker history <image>
~~~

These help investigate image metadata, layers, missing files, entrypoints, and size.

## 24. Debugging Build Failures

~~~bash
docker build -t my-app .
docker build --no-cache -t my-app .
~~~

Check:

- Build context
- .dockerignore
- COPY paths
- package files
- dependency installation
- Dockerfile syntax
- network/package registry access

Do not use --no-cache automatically. First understand whether cache is actually the problem.

## 25. Compose Debugging Workflow

~~~bash
docker compose up -d
docker compose ps
docker compose logs -f
docker compose exec backend sh
docker network ls
docker network inspect <network>
docker compose build
docker compose up -d --force-recreate
~~~

## 26. Common HTTP Errors

### 502 Bad Gateway

Usually indicates a reverse proxy cannot successfully reach its upstream.

Check backend status, service name, internal port, network, and application binding.

### 404 Not Found

The server is reachable, but the requested route/resource does not exist.

Check URL, routing, backend route, and SPA fallback.

### 500 Internal Server Error

The application encountered a server-side error.

Check application logs first.

### Connection Refused

Usually means nothing is accepting connections at the target address/port.

Check process, port, 0.0.0.0 binding, network, and service name.

## 27. Systematic Debugging Order

Use this order instead of randomly changing configuration:

~~~text
1. Container status
       ↓
2. Container logs
       ↓
3. Health status
       ↓
4. Process
       ↓
5. Ports
       ↓
6. Environment
       ↓
7. Mounts
       ↓
8. Network/DNS
       ↓
9. Application dependency
       ↓
10. Host resources
~~~

This prevents unnecessary changes.

## 28. CodeBuddy Debugging Example

Suppose:

~~~text
Browser
  ↓
Nginx
  ↓
Backend
  ↓
MongoDB
~~~

API request returns 502.

Debug:

~~~text
1. docker compose ps
2. docker compose logs backend
3. docker compose logs nginx
4. docker inspect backend
5. test backend health
6. inspect Docker network
7. test backend → MongoDB
8. inspect environment variables
9. inspect resource usage
~~~

Follow the request path instead of guessing.

## 29. Production Logging Architecture

~~~text
Application containers
        ↓
stdout/stderr
        ↓
Docker/runtime logging
        ↓
Log collector
        ↓
Centralized log storage
        ↓
Search / dashboards / alerts
~~~

Examples include ELK/OpenSearch, Loki, cloud logging platforms, and managed observability services.

The important principle is that application logs should remain accessible even when containers are replaced.

## 30. Common Mistakes

1. Looking only at docker ps.
2. Ignoring container logs.
3. Using localhost for another container.
4. Debugging networking before checking whether the application is running.
5. Storing production logs only inside containers.
6. Logging secrets.
7. Running every container as root just to avoid permissions problems.
8. Using --no-cache without understanding the problem.
9. Changing many configuration values simultaneously.
10. Not reproducing the failure with a minimal test.

## 31. Best Practices

1. Log to stdout/stderr.
2. Use structured logs for production applications.
3. Configure log rotation/retention.
4. Never log secrets.
5. Use docker logs early in debugging.
6. Inspect containers instead of guessing.
7. Debug networking with service names.
8. Verify ports and binding.
9. Monitor health and resources.
10. Centralize logs for larger production systems.
11. Keep a repeatable debugging checklist.
12. Change one variable at a time when diagnosing failures.

# Interview Questions

### 1. How do you view container logs?

Use docker logs <container>.

### 2. How do you follow logs?

Use docker logs -f <container>.

### 3. How do you enter a running container?

Use docker exec -it <container> sh.

### 4. How do you inspect container configuration?

Use docker inspect <container>.

### 5. How do you inspect processes inside a container?

Use docker top <container>.

### 6. How do you debug container networking?

Inspect Docker networks and test connectivity using Docker service/container names.

### 7. Why should applications log to stdout/stderr?

Docker can capture those streams and forward them to the configured logging system.

### 8. Why can a container be running but the application unavailable?

The main process can remain alive while the application is unhealthy, misconfigured, or unable to reach dependencies.

### 9. What is the first thing you check when a container exits?

Check docker ps -a and then docker logs <container>.

### 10. How do you debug a 502 behind Nginx?

Verify Nginx logs, backend status, service name, internal port, network connectivity, and backend health.

# Quick Revision

~~~text
Status
  ↓
Logs
  ↓
Health
  ↓
Process
  ↓
Ports
  ↓
Environment
  ↓
Mounts
  ↓
Network
  ↓
Dependencies
  ↓
Resources
~~~

Important commands:

~~~bash
docker ps
docker ps -a
docker logs <container>
docker logs -f <container>
docker inspect <container>
docker exec -it <container> sh
docker top <container>
docker port <container>
docker stats
docker network ls
docker network inspect <network>
docker compose ps
docker compose logs -f
docker compose exec <service> sh
~~~

# ⭐ Must Remember

1. **docker ps tells you status; docker logs tells you what the application is doing.**
2. **Use docker ps -a when a container has stopped.**
3. **Use docker inspect for configuration details.**
4. **Use docker exec to investigate a running container.**
5. **Use docker top to inspect container processes.**
6. **Inside Docker networking, use service names instead of localhost for other containers.**
7. **Application servers should generally listen on 0.0.0.0 inside containers.**
8. **Log to stdout/stderr and avoid logging secrets.**
9. **Use log rotation/retention so logs do not exhaust disk space.**
10. **Debug systematically: status → logs → health → process → ports → config → mounts → network → dependencies → resources.**
11. **For CodeBuddy, follow the request path from Nginx → backend → MongoDB when debugging.**
12. **Production systems should centralize logs rather than depending only on individual containers.**
