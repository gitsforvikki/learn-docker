# Lesson 33 — Docker Security

## 1. Why Docker Security Matters

Containers provide isolation, but containers are not a complete security boundary by themselves.

Think about security across the lifecycle:

Image → Container → Process → Network → Secrets → Host → Supply chain

## 2. Run as a Non-Root User

Running the application as a non-root user reduces the impact of a container compromise.

Example Dockerfile:

    FROM node:22-alpine
    WORKDIR /app
    COPY package*.json ./
    RUN npm ci --omit=dev
    COPY --chown=node:node . .
    USER node
    CMD ["node", "server.js"]

USER controls the user used by subsequent Dockerfile instructions and the container's default process.

## 3. Principle of Least Privilege

Give a process only the permissions it actually needs.

- Run as non-root
- Use read-only filesystems where practical
- Drop unnecessary Linux capabilities
- Avoid privileged containers
- Expose only required ports
- Mount only required directories

## 4. Avoid --privileged

Do not use this simply to make an application work:

    docker run --privileged my-app

It grants a container significantly broader access to host resources and Linux capabilities. Find the specific permission actually required instead.

## 5. Linux Capabilities

Capabilities divide powerful root privileges into smaller permissions.

    docker run --cap-drop=ALL my-app

Only add a required capability deliberately:

    docker run --cap-drop=ALL --cap-add=<CAPABILITY> my-app

## 6. Read-Only Root Filesystem

If an application does not need to write to its root filesystem:

    docker run --read-only my-app

For temporary storage:

    docker run --read-only --tmpfs /tmp my-app

This can reduce the impact of attempts to modify files inside the container.

## 7. Avoid Unnecessary Host Mounts

Be especially careful with /var/run/docker.sock. Access to the Docker socket can provide extremely powerful control over the Docker daemon.

## 8. Image Security

Security starts before the container runs.

- Trusted base images
- Minimal base images
- Pinned versions where appropriate
- Updated dependencies
- Vulnerability scanning
- Multi-stage builds
- .dockerignore

A smaller image generally contains fewer packages and a smaller attack surface.

## 9. Scan Images

When Docker Scout is available:

    docker scout quickview my-app:1.0
    docker scout cves my-app:1.0

Scanning is not a guarantee of security. Review findings and keep dependencies updated.

## 10. Dependency Security

For Node.js:

    npm audit
    npm ci

Use the lockfile for reproducible installation and remove unnecessary packages.

## 11. Never Put Secrets in the Image

Do not put passwords or API keys in Dockerfiles or copy .env files into images.

Secrets embedded in image layers may remain recoverable.

Prefer runtime secret/configuration mechanisms.

## 12. Build-Time and Runtime Secrets

If a build requires a secret, use BuildKit secret mounts rather than ARG or ENV.

At runtime, use appropriate mechanisms such as Docker secrets, cloud secret managers, Kubernetes Secrets, or external secret-management platforms.

Important: secrets should not be baked into images or source code.

## 13. Environment Variables Are Not Automatically Secret

This does not bake a value into the image:

    docker run -e DB_PASSWORD=secret my-app

But the value is still sensitive runtime information and may be exposed through inspection or debugging. Use stronger secret mechanisms when required.

## 14. Network Security

Do not expose every service to the host.

Example Compose idea:

    frontend:
      ports:
        - "80:80"
    backend:
      expose:
        - "3000"
    mongo:
      expose:
        - "27017"

The backend and database can communicate through the Docker network without necessarily publishing their ports to the host.

Publish only what must be reachable externally.

## 15. Network Segmentation

Separate services where practical:

Public network → Frontend / reverse proxy

Private network → Backend → Database

This reduces unnecessary connectivity.

## 16. Avoid Hard-Coded IP Addresses

Use Docker DNS/service names such as mongo:27017 rather than container IP addresses, which can change when containers are recreated.

## 17. TLS and HTTPS

Production traffic should use HTTPS when sensitive data is transmitted.

Common architecture:

Internet → HTTPS → Reverse Proxy → Backend → Database

TLS termination is often handled by a reverse proxy or load balancer.

## 18. Container Resource Limits

Resource limits can reduce the impact of runaway processes.

    docker run --memory=512m --cpus=1 my-app

See Lesson 30 for detailed resource management.

## 19. Health Checks and Security

Health endpoints should not expose passwords, tokens, or internal credentials.

Prefer a minimal response such as:

    {
      "status": "ok"
    }

## 20. Protect the Docker Daemon

The Docker daemon is highly privileged.

- Protect the Docker socket
- Limit Docker daemon access
- Avoid exposing the daemon API publicly
- Use appropriate authentication/TLS when remote access is required
- Give CI/CD systems only the access they need

## 21. Secrets in Git

Never commit .env files, private keys, API tokens, database passwords, cloud credentials, or JWT secrets.

Example .gitignore:

    .env
    .env.*
    !.env.example

Keep .env.example with placeholder values only.

## 22. .dockerignore

Prevent unnecessary or sensitive files from entering the build context.

Example:

    node_modules
    .git
    .env
    .env.*
    npm-debug.log
    Dockerfile*
    docker-compose*.yml

Do not exclude files required during the build.

## 23. Supply Chain Security

Your application depends on more than your own source code:

Your code → Base image → OS packages → Runtime → npm dependencies → Build tools

Useful practices:
- Trusted registries
- Dependency lockfiles
- Vulnerability scanning
- Signed/verified images where supported
- Regular updates
- Minimal images
- Reproducible builds

## 24. Image Tags and Digests

Tags can move. For stronger reproducibility, images can be referenced by digest:

    image@sha256:<digest>

A digest identifies specific image content and provides stronger immutability than a mutable tag.

## 25. Dockerfile Security Checklist

- Trusted base image
- Minimal image
- Non-root USER
- No secrets
- .dockerignore
- Minimal packages
- Multi-stage build where useful
- Pinned dependencies
- Health check where appropriate

## 26. CodeBuddy Security Example

Internet → Reverse Proxy → Frontend → Backend → MongoDB

Security priorities:
- Backend should not run as root
- MongoDB should not be unnecessarily published
- JWT and Razorpay secrets must stay outside the image
- Use HTTPS in production
- Restrict networks
- Scan images
- Use resource limits
- Add health checks
- Keep dependencies updated

## 27. Security Is Layered

No single setting makes Docker secure.

Secure source code → Secure dependencies → Secure image → Secure container → Secure network → Secure secrets → Secure host → Monitoring

This is defense in depth.

## 28. Common Mistakes

1. Running applications as root without a reason.
2. Using --privileged.
3. Mounting the Docker socket unnecessarily.
4. Baking secrets into images.
5. Publishing database ports publicly.
6. Using outdated vulnerable images.
7. Installing unnecessary packages.
8. Committing .env files.
9. Assuming containers provide complete isolation.
10. Ignoring host and Docker daemon security.
11. Giving CI/CD systems excessive permissions.
12. Treating vulnerability scanning as the only security control.

## 29. Best Practices

1. Run as non-root.
2. Follow least privilege.
3. Use minimal trusted images.
4. Scan images and dependencies.
5. Keep images and packages updated.
6. Never bake secrets into images.
7. Publish only required ports.
8. Segment networks.
9. Avoid privileged containers.
10. Avoid unnecessary host mounts.
11. Use read-only filesystems where practical.
12. Apply resource limits.
13. Protect Docker daemon access.
14. Keep secrets out of Git.
15. Use immutable image references for important production deployments.
16. Treat security as a layered process.

# Interview Questions

### 1. Are Docker containers completely secure?

No. Containers provide isolation mechanisms, but security also depends on the image, application, runtime, host, network, secrets, and configuration.

### 2. Why should containers run as non-root?

It reduces the privileges available to a compromised application and follows least privilege.

### 3. What is --privileged?

It gives a container broad additional privileges and should be avoided unless there is a clearly understood requirement.

### 4. What is a Linux capability?

A granular permission that divides traditional root privileges into smaller units.

### 5. Why should secrets not be placed in Dockerfiles?

They can become part of image layers and may remain recoverable from image history or layers.

### 6. Why is exposing the Docker socket risky?

It can provide powerful control over the Docker daemon and potentially the host.

### 7. Why should database ports usually not be published?

If the database only needs to communicate with backend containers, exposing it unnecessarily increases attack surface.

### 8. What is least privilege?

Giving a process only the permissions it actually needs.

### 9. Why use image digests?

A digest identifies specific image content and provides stronger immutability than a mutable tag.

### 10. What is defense in depth?

Using multiple independent security layers so that failure of one control does not compromise the entire system.

# Quick Revision

Source → Dependencies → Image → Container → Network → Secrets → Host → Monitoring

Important commands:

    docker scout cves <image>
    docker inspect <container>
    docker run --read-only <image>
    docker run --cap-drop=ALL <image>
    docker run --memory=512m --cpus=1 <image>
    npm audit

# ⭐ Must Remember

1. Containers are not a complete security boundary.
2. Run application processes as non-root whenever practical.
3. Follow least privilege.
4. Avoid --privileged unless absolutely required and understood.
5. Never bake secrets into Docker images.
6. Protect the Docker socket and daemon.
7. Publish only the ports that must be externally reachable.
8. Keep databases on private Docker networks when possible.
9. Use trusted, minimal, updated images and scan them.
10. Use .dockerignore and keep secrets out of Git.
11. Use resource limits to reduce runaway-process impact.
12. For production, prefer immutable image references where appropriate.
13. Docker security is defense in depth, not one command or setting.