# Lesson 21 — Compose Services

## 1. What Is a Compose Service?

A **service** is a logical application component defined under the `services` section of a Compose file.

Example:

~~~yaml
services:
  frontend:
    image: nginx:alpine

  backend:
    build: ./backend

  database:
    image: postgres:18
~~~

Here there are three services:

- `frontend`
- `backend`
- `database`

A service describes **how a container should be created and run**.

> Important: A Compose service is a configuration definition. When Compose starts it, one or more containers are created from that service configuration.

---

## 2. Service vs Container

These terms are related but not identical.

### Service

A service is the configuration declared in `compose.yaml`.

### Container

A container is the actual running instance created from that service.

Mental model:

~~~text
compose.yaml
     |
     v
  service
     |
     v
 container
~~~

For example:

~~~yaml
services:
  backend:
    image: my-backend
~~~

The `backend` service can create a backend container.

If the service is scaled:

~~~bash
docker compose up --scale backend=3
~~~

the same service configuration can represent multiple container instances.

---

## 3. Anatomy of a Service

A service can define many properties:

~~~yaml
services:
  backend:
    build: ./backend
    ports:
      - "3000:3000"
    environment:
      NODE_ENV: production
    volumes:
      - ./backend:/app
    networks:
      - app-network
    depends_on:
      - database
    restart: unless-stopped
~~~

The most important service properties are:

- `image`
- `build`
- `ports`
- `expose`
- `environment`
- `env_file`
- `volumes`
- `networks`
- `depends_on`
- `restart`
- `command`
- `entrypoint`
- `healthcheck`
- `user`
- `working_dir`

We will focus on the properties that are most important for real projects and interviews.

---

## 4. image

Use an existing Docker image.

~~~yaml
services:
  redis:
    image: redis:8
~~~

Compose uses the specified image to create the container.

For production, prefer an intentional version/tag instead of blindly relying on `latest`.

---

## 5. build

Build the service image from a Dockerfile.

~~~yaml
services:
  backend:
    build: ./backend
~~~

This means:

~~~text
./backend
    |
    v
Dockerfile
    |
    v
Backend image
    |
    v
Backend container
~~~

### Advanced build configuration

~~~yaml
services:
  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
      args:
        NODE_ENV: production
~~~

### context

The build context is the directory Docker can access during the build.

### dockerfile

Specifies which Dockerfile to use.

### args

Provides Docker build arguments.

Remember:

`ARG` is primarily build-time configuration, while `environment` is runtime configuration.

---

## 6. image vs build

These are commonly confused.

### image

Use an already available image:

~~~yaml
image: postgres:18
~~~

### build

Build your own image:

~~~yaml
build: ./backend
~~~

You can also use both:

~~~yaml
services:
  backend:
    build: ./backend
    image: mycompany/backend:1.0
~~~

This tells Compose to build the image and assign the specified image name/tag.

---

## 7. ports

Publishes container ports to the host.

~~~yaml
ports:
  - "3000:3000"
~~~

Meaning:

~~~text
Host port 3000
       |
       v
Container port 3000
~~~

Another example:

~~~yaml
ports:
  - "8080:80"
~~~

Meaning:

~~~text
localhost:8080 -> container:80
~~~

### Important

Port publishing is mainly for traffic entering from outside the container network.

Service-to-service communication normally does not need published ports.

---

## 8. expose

`expose` documents or makes a container port available to linked/networked services without publishing it to the host.

Example:

~~~yaml
services:
  backend:
    expose:
      - "3000"
~~~

Other Compose services can communicate with the backend over the Docker network.

However, modern Compose networking already allows services to communicate using their internal ports, so `expose` is often unnecessary.

### Important comparison

~~~text
ports
  Host -> Container

expose
  Container-network visibility/documentation
~~~

Do not use `expose` as a replacement for `ports` when browser/host access is required.

---

## 9. environment

Defines runtime environment variables.

~~~yaml
services:
  backend:
    environment:
      NODE_ENV: production
      PORT: 3000
      DATABASE_URL: postgresql://database:5432/app
~~~

Inside Node.js:

~~~js
process.env.DATABASE_URL
~~~

### Do not hard-code secrets

Avoid committing passwords and API keys directly into Compose files.

Use appropriate environment or secret-management mechanisms.

---

## 10. env_file

Loads environment variables from a file.

Example:

~~~yaml
services:
  backend:
    env_file:
      - .env
~~~

The application can then access those values through its environment.

Be careful with sensitive files:

~~~text
.env
~~~

should normally be excluded from Git when it contains secrets.

---

## 11. environment vs env_file

Both provide environment variables, but their usage differs.

### environment

Good for explicitly declaring important service configuration:

~~~yaml
environment:
  NODE_ENV: production
  PORT: 3000
~~~

### env_file

Useful when many variables are stored externally:

~~~yaml
env_file:
  - .env
~~~

A project can use both when necessary.

---

## 12. volumes

Volumes mount persistent or host-managed data.

### Named volume

~~~yaml
services:
  database:
    image: postgres:18
    volumes:
      - postgres-data:/var/lib/postgresql/data

volumes:
  postgres-data:
~~~

The database data survives container recreation.

### Bind mount

~~~yaml
services:
  backend:
    volumes:
      - ./backend:/app
~~~

This maps a host directory into the container.

Bind mounts are particularly useful during development.

---

## 13. networks

Services can be attached to specific networks.

~~~yaml
services:
  backend:
    networks:
      - app-network

  database:
    networks:
      - app-network

networks:
  app-network:
~~~

Now backend and database share the `app-network`.

A service can also belong to multiple networks.

This is useful for separating public-facing and private communication.

---

## 14. Multi-Network Architecture

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
~~~

Architecture:

~~~text
          public
frontend ---------- backend
                     |
                     |
                   private
                     |
                     v
                  database
~~~

This creates a useful isolation boundary.

The database is not directly attached to the public network.

---

## 15. depends_on

Defines service dependencies.

~~~yaml
services:
  backend:
    depends_on:
      - database

  database:
    image: postgres:18
~~~

This expresses:

~~~text
database
   |
   v
backend
~~~

But remember:

> `depends_on` does not automatically guarantee that the dependency is ready to accept application connections.

For readiness-sensitive applications, use health checks and appropriate application retry logic.

---

## 16. depends_on with Health Checks

A more robust pattern is:

~~~yaml
services:
  backend:
    depends_on:
      database:
        condition: service_healthy

  database:
    image: postgres:18
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
~~~

Now Compose can wait for the database health condition before starting the dependent service.

The application should still handle connection failures gracefully.

---

## 17. restart

Controls when Compose restarts a container.

Common policies:

~~~yaml
restart: "no"
~~~

Do not automatically restart.

~~~yaml
restart: always
~~~

Always restart according to Docker's restart behavior.

~~~yaml
restart: on-failure
~~~

Restart when the container exits with a failure.

~~~yaml
restart: unless-stopped
~~~

Restart unless the container was explicitly stopped.

For long-running services, `unless-stopped` is often a useful choice in smaller deployments.

---

## 18. command

Overrides the image's default `CMD`.

Example:

~~~yaml
services:
  backend:
    image: node:22-alpine
    command: ["node", "server.js"]
~~~

The image still provides its normal configuration, but Compose supplies a different command.

---

## 19. entrypoint

Overrides the image's default `ENTRYPOINT`.

Example:

~~~yaml
services:
  backend:
    entrypoint: ["node"]
    command: ["server.js"]
~~~

Conceptually:

~~~text
ENTRYPOINT + CMD
       |
       v
node server.js
~~~

This follows the same CMD vs ENTRYPOINT concepts learned earlier.

---

## 20. healthcheck

A health check tells Docker how to determine whether a containerized service is healthy.

Example:

~~~yaml
services:
  backend:
    healthcheck:
      test: ["CMD", "wget", "--spider", "-q", "http://localhost:3000/api/health"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 10s
~~~

A health check is different from simply checking whether a process exists.

For an API, a health endpoint can provide a much better signal.

---

## 21. user and working_dir

You can define the runtime user:

~~~yaml
services:
  backend:
    user: "1000:1000"
~~~

This can help avoid running application processes as root.

You can also define the working directory:

~~~yaml
services:
  backend:
    working_dir: /app
~~~

This is equivalent in concept to Dockerfile `WORKDIR`.

---

## 22. Service Names Become DNS Names

Suppose:

~~~yaml
services:
  api:
    build: ./backend

  postgres:
    image: postgres:18
~~~

The API can connect to PostgreSQL using:

~~~text
postgres:5432
~~~

It does not need the container IP address.

This is one of the most important Compose features:

~~~text
service name
     |
     v
Docker DNS
     |
     v
container IP
~~~

Never hard-code dynamic container IP addresses.

---

## 23. Service Discovery

Imagine:

~~~yaml
services:
  api:
    ...
  redis:
    image: redis:8
  mongo:
    image: mongo:8
~~~

The API can use:

~~~text
redis:6379
mongo:27017
~~~

The names remain stable even when container IP addresses change.

This is much more reliable than storing container IP addresses in application configuration.

---

## 24. Scaling Services

Example:

~~~bash
docker compose up --scale backend=3
~~~

Conceptually:

~~~text
             backend service
             /      |      \
            v       v       v
       backend-1 backend-2 backend-3
~~~

Service discovery and load-balancing behavior should be considered carefully when scaling.

A fixed host port can prevent multiple replicas from binding the same port.

---

## 25. Resource Configuration

Compose can also define resource-related settings.

For example, depending on the deployment environment and Compose implementation:

~~~yaml
services:
  backend:
    mem_limit: 512m
~~~

Resource controls are important because one service should not consume unlimited host resources.

Detailed resource management is covered later in the advanced Docker section.

---

## 26. A Realistic Service Definition

Example:

~~~yaml
services:
  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
    image: myapp/backend:1.0
    ports:
      - "3000:3000"
    environment:
      NODE_ENV: production
      DATABASE_URL: postgresql://postgres:5432/app
    depends_on:
      database:
        condition: service_healthy
    networks:
      - app-network
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "wget", "--spider", "-q", "http://localhost:3000/api/health"]
      interval: 30s
      timeout: 5s
      retries: 3

  database:
    image: postgres:18
    environment:
      POSTGRES_DB: app
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: example
    volumes:
      - postgres-data:/var/lib/postgresql/data
    networks:
      - app-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

networks:
  app-network:

volumes:
  postgres-data:
~~~

This demonstrates how multiple service properties work together.

For real production use, do not put a real database password directly in a committed file.

---

## 27. Common Mistakes

### Mistake 1 — Confusing service and container

A service is the Compose definition; a container is the running instance.

### Mistake 2 — Using container IPs

Do not configure:

~~~text
172.x.x.x
~~~

Use the service name.

### Mistake 3 — Publishing every port

A database often does not need to be exposed to the host.

### Mistake 4 — Using localhost between services

Wrong:

~~~text
postgresql://localhost:5432/app
~~~

Correct:

~~~text
postgresql://database:5432/app
~~~

### Mistake 5 — Assuming depends_on means ready

Use health checks and retry logic where readiness matters.

### Mistake 6 — Storing secrets in Git

Keep secrets outside committed Compose configuration.

### Mistake 7 — Using bind mounts for production database storage without understanding the implications

Choose storage intentionally. Named volumes are generally simpler for container-managed database persistence.

---

## 28. Debugging Services

List services:

~~~bash
docker compose ps
~~~

View service logs:

~~~bash
docker compose logs backend
~~~

Follow logs:

~~~bash
docker compose logs -f backend
~~~

Inspect the generated configuration:

~~~bash
docker compose config
~~~

Execute a command inside a service:

~~~bash
docker compose exec backend sh
~~~

Inspect networks:

~~~bash
docker network ls
~~~

Inspect volumes:

~~~bash
docker volume ls
~~~

---

## 29. CodeBuddy Example

A simplified CodeBuddy Compose architecture could be:

~~~yaml
services:
  frontend:
    build: ./frontend
    ports:
      - "5173:80"

  backend:
    build: ./backend
    environment:
      MONGODB_URI: mongodb://mongo:27017/codebuddy
    depends_on:
      - mongo

  mongo:
    image: mongo:8
    volumes:
      - mongo-data:/data/db

volumes:
  mongo-data:
~~~

The important networking rule is:

~~~text
backend -> mongo:27017
~~~

The backend should not use `localhost` for MongoDB.

Later, CodeBuddy can add Redis, a reverse proxy, health checks, CI/CD, and Kubernetes.

---

## 30. Interview Questions

### Q1. What is a Compose service?

A service is a declarative configuration describing how one application component should be built and run.

### Q2. What is the difference between a service and a container?

A service is the Compose configuration; a container is the runtime instance created from that configuration.

### Q3. What is the difference between image and build?

`image` uses an existing image, while `build` tells Compose to build an image from a Dockerfile and build context.

### Q4. What is the difference between ports and expose?

`ports` publishes a container port to the host. `expose` does not publish the port to the host and is generally unnecessary for normal Compose-to-Compose communication.

### Q5. How do services discover each other?

Compose provides network DNS, so services can normally use each other's service names as hostnames.

### Q6. Why should you not use container IP addresses?

Container IP addresses can change when containers are recreated. Service names provide stable application-level discovery.

### Q7. Does depends_on guarantee readiness?

No. Use health checks and application retry logic when readiness matters.

### Q8. Why use multiple networks?

Multiple networks can isolate groups of services and limit which services can communicate directly.

### Q9. What is the purpose of a healthcheck?

It gives Docker a mechanism to determine whether the application inside a container is actually healthy.

---

## 31. Quick Revision

~~~text
Compose service
    |
    +-- image / build
    +-- ports
    +-- environment
    +-- volumes
    +-- networks
    +-- depends_on
    +-- restart
    +-- command / entrypoint
    +-- healthcheck
~~~

### Most important rules

~~~text
Service name -> DNS hostname

ports
Host -> Container

Internal communication
service-name:container-port

image
Use existing image

build
Build custom image

volume
Persist/share data

network
Control service communication

healthcheck
Check application health
~~~

---

## ⭐ Must Remember

1. **A Compose service is a configuration; a container is its runtime instance.**
2. **Use service names for service-to-service communication.**
3. **Never depend on dynamic container IP addresses.**
4. **Use `ports` when host/browser access is required.**
5. **Internal service communication normally does not require published ports.**
6. **`depends_on` controls dependency ordering, not complete application readiness.**
7. **Use health checks when readiness/health matters.**
8. **Use multiple networks when you need service isolation.**
9. **Keep secrets out of committed Compose files.**
10. **Dockerfile builds the image; Compose defines how the service runs.**

---

## 🎯 Interview Summary

> A Docker Compose service is a declarative definition of an application component. It can specify an existing image or a Dockerfile build, ports, environment variables, volumes, networks, dependencies, restart policies, commands, and health checks. Compose provides service discovery through Docker DNS, so services communicate using stable service names instead of container IP addresses. For more complex architectures, multiple networks can isolate public and private services. Health checks should be used when simple startup ordering is not enough.
