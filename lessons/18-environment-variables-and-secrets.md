# Lesson 18 — Environment Variables & Secrets

## 1. Why Configuration Matters

Applications need configuration that changes between environments: database URLs, API URLs, ports, environment names, feature flags, and credentials.

Docker lets you keep configuration outside the image so the same image can be used in different environments.

## 2. Environment Variables

An environment variable is a key-value configuration value available to a process.

    NODE_ENV=production
    PORT=3000
    DATABASE_HOST=db

Node.js reads them with `process.env.NODE_ENV`, `process.env.PORT`, etc.

## 3. Setting Variables with docker run

    docker run -d --name api -e NODE_ENV=production -e PORT=3000 my-api

## 4. Using --env-file

Example `.env`:

    NODE_ENV=production
    PORT=3000
    DATABASE_HOST=db

Run:

    docker run --env-file .env my-api

Do not commit real secrets in `.env`. Add appropriate `.env` patterns to `.gitignore`.

## 5. ENV in a Dockerfile

    ENV NODE_ENV=production
    ENV PORT=3000

These are image/runtime defaults and can be overridden at container startup:

    docker run -e PORT=4000 my-api

## 6. ARG vs ENV

`ARG` is primarily a build-time variable:

    ARG APP_VERSION
    RUN echo "Building $APP_VERSION"

Build with:

    docker build --build-arg APP_VERSION=1.0.0 -t my-api .

`ENV` is available to processes in the running container.

**Mental model:** `ARG → build time`, `ENV → runtime configuration`.

Do not use `ARG` as a secure secret mechanism.

## 7. Environment Variable Precedence

Runtime configuration should be able to override image defaults.

Dockerfile:

    ENV PORT=3000

Runtime:

    docker run -e PORT=4000 my-api

The application receives `PORT=4000`.

## 8. Inspecting Environment Variables

    docker exec api env
    docker exec api printenv
    docker inspect api

Be careful when sharing output because it may contain credentials.

## 9. Environment Variables Are Not Secrets

Environment variables are a configuration mechanism, not automatically secure secrets storage.

If you pass a password with `-e`, the value exists as container configuration and may be visible through inspection or other runtime tooling.

**Do not assume that putting a password in an environment variable makes it secret.**

## 10. Docker Secrets

Docker provides a secrets mechanism for supported deployment environments. The basic idea is:

    Secret → mounted into container → application reads secret

In Docker Swarm, secrets are a first-class feature.

Example:

    echo "my-db-password" | docker secret create db_password -

Secrets are commonly used in production/orchestration workflows. For local development, an uncommitted `.env` file may be acceptable, but production secret management should be deliberate.

## 11. Build Secrets

Never bake private credentials into an image.

Modern Docker BuildKit supports secret mounts for build-time secrets:

    RUN --mount=type=secret,id=npm_token ...

This makes a secret available to a build step without intentionally storing it as a normal image environment variable.

## 12. Build-Time vs Runtime Configuration

Frontend applications make this distinction especially important.

**Build-time:** a value is consumed while creating the bundle. Examples include Vite `VITE_*` and Next.js `NEXT_PUBLIC_*` values used by client-side code.

Once included in a browser bundle, a value should be considered public.

**Runtime:** a backend reads values when the container starts or while handling requests. Examples include `DATABASE_URL`, `JWT_SECRET`, and payment secrets.

## 13. Public vs Private Configuration

Usually public:

    NEXT_PUBLIC_API_URL
    VITE_API_URL
    APP_ENV
    PORT

Usually private:

    DATABASE_PASSWORD
    JWT_SECRET
    API_SECRET
    PAYMENT_SECRET
    PRIVATE_KEY

A variable name alone does not guarantee security. If a value reaches browser JavaScript, treat it as public.

## 14. Docker Compose Environment Variables

Compose can pass variables directly:

    services:
      api:
        image: my-api
        environment:
          NODE_ENV: production
          PORT: 3000

Or from a file:

    services:
      api:
        image: my-api
        env_file:
          - .env

## 15. Compose Variable Substitution

Compose can also substitute variables into the Compose configuration:

    services:
      api:
        image: ${IMAGE_NAME}:${IMAGE_TAG}
        ports:
          - "${HOST_PORT}:3000"

With variables such as `IMAGE_NAME=my-api`, `IMAGE_TAG=1.0.0`, and `HOST_PORT=8080`.

This is different from simply passing environment variables into the application.

## 16. `.env` Has Multiple Roles

A `.env` file can be used:

1. By your application through environment configuration.
2. By Docker Compose for variable substitution.
3. As an `env_file` source for container environment variables.

These mechanisms are related but not identical. Always verify which mechanism your configuration actually uses.

## 17. Validate Configuration at Startup

A production application should validate required variables.

Example:

    DATABASE_URL → required
    JWT_SECRET   → required
    PORT         → optional/default

Fail fast if required configuration is missing instead of allowing the application to fail later during a request.

## 18. Common Mistakes

- Committing `.env` credentials.
- Putting secrets in the Dockerfile.
- Using `ARG` for secrets.
- Assuming environment variables are encrypted.
- Exposing private frontend variables.
- Hard-coding environment-specific values into images.
- Logging secrets.

## 19. Best-Practice Configuration Flow

Use the same application image with different configuration:

    Same image
       ├── Development config
       ├── Staging config
       └── Production config

For secrets:

    Secret manager / deployment secret
              ↓
        container runtime
              ↓
          application

This supports immutable application images and safer deployments.

## 20. Practical Node.js Example

    const port = Number(process.env.PORT || 3000);
    const databaseUrl = process.env.DATABASE_URL;

    if (!databaseUrl) {
      throw new Error("DATABASE_URL is required");
    }

    app.listen(port, "0.0.0.0");

Run with runtime configuration:

    docker run --rm \
      -e PORT=3000 \
      -e DATABASE_URL="postgresql://user:password@db:5432/app" \
      my-api

For real production deployments, do not put actual passwords into shell history or source-controlled configuration.

## 21. CodeBuddy Example

A Dockerized CodeBuddy backend may need:

    PORT
    MONGODB_URI
    JWT_SECRET
    FRONTEND_URL
    RAZORPAY_KEY_ID
    RAZORPAY_KEY_SECRET

Typical separation:

Public/non-secret configuration: `PORT`, `FRONTEND_URL`.

Secret configuration: `MONGODB_URI`, `JWT_SECRET`, `RAZORPAY_KEY_SECRET`.

The backend container receives these at runtime, while the image remains reusable across environments.

## 22. Interview Questions

### Q1. What is the difference between ARG and ENV?
`ARG` is primarily build-time configuration. `ENV` provides environment variables available to the image/container runtime.

### Q2. Are Docker environment variables secure?
No. They are configuration, not a complete secrets-management solution.

### Q3. Why should secrets not be baked into an image?
Images can be cached, copied, inspected, and distributed. A secret baked into an image can be difficult to remove completely.

### Q4. How should production secrets be provided?
Use the deployment platform's secret mechanism or an external secret manager rather than hard-coding credentials into images or source code.

### Q5. What does `--env-file` do?
It loads environment variables from a file and supplies them to the container.

### Q6. Can runtime `-e` values override Dockerfile `ENV` defaults?
Yes.

### Q7. What is the difference between Compose variable substitution and container environment variables?
Compose substitution changes the Compose configuration during processing; environment variables configure the process running inside the container.

### Q8. Why should the same image be used across environments?
It reduces environment-specific build differences and supports predictable, immutable deployments. Configuration changes at runtime.

## 23. Quick Revision

- Environment variables provide runtime configuration.
- `docker run -e KEY=value` sets a variable.
- `--env-file` loads variables from a file.
- Dockerfile `ENV` provides image/runtime defaults.
- `ARG` is primarily build-time.
- Do not use `ARG` as a secret store.
- Environment variables are not automatically secure secrets.
- Never commit real `.env` credentials.
- Do not bake secrets into images.
- BuildKit supports build-time secret mounts.
- Docker/Swarm provides a secrets mechanism.
- Frontend values shipped to browsers are public.
- Validate required configuration at application startup.
- Prefer the same image with different runtime configuration.
- Never log secrets.

## Must Remember

> **ARG = build time.**

> **ENV = runtime configuration/default.**

> **Environment variable ≠ secure secret store.**

> **Never bake secrets into Docker images.**

> **Build once, configure per environment.**

## Interview Summary

    One image
       +
    Environment-specific configuration
       +
    Secure secret management
       =
    Predictable deployment