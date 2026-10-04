# Lesson 26 — Docker CI/CD

## 1. What Is CI/CD?

CI/CD automates validating, building, packaging, and delivering software.

- **CI — Continuous Integration:** automatically test and validate code changes.
- **CD — Continuous Delivery/Deployment:** automatically prepare or deploy validated software.

Docker becomes the packaging and delivery unit.

~~~text
Developer
   ↓
Git push
   ↓
CI pipeline
   ├── Test
   ├── Build Docker image
   ├── Scan
   └── Push to registry
          ↓
       Deployment
          ↓
       Container
~~~

---

## 2. Why Docker Fits CI/CD

A strong Docker delivery flow is:

~~~text
Source
  ↓
Test
  ↓
Build Docker Image
  ↓
Registry
  ↓
Deploy same image
~~~

The same image can move through:

~~~text
Development → Staging → Production
~~~

This is the **build once, deploy the same artifact** principle.

---

## 3. Typical Docker CI/CD Pipeline

~~~text
Git push
   ↓
Checkout source
   ↓
Lint / Test
   ↓
Build Docker image
   ↓
Security scan
   ↓
Tag image
   ↓
Push image to registry
   ↓
Deploy image
   ↓
Health check
   ↓
Success / rollback
~~~

---

## 4. CI vs CD

| CI | CD |
|---|---|
| Validates code | Delivers/deploys artifact |
| Runs tests | Pulls image |
| Runs linting | Performs deployment |
| Builds image | Runs health checks |
| Finds problems early | Releases validated software |

**Continuous Delivery** can prepare a production release for manual approval.

**Continuous Deployment** automatically deploys after successful validation.

---

## 5. Git as the Pipeline Trigger

Typical flow:

~~~text
Developer
   ↓
git push
   ↓
GitHub
   ↓
CI trigger
   ↓
Pipeline
~~~

Common triggers:

- push to a branch
- pull request
- merge to main
- release/tag
- manual workflow
- scheduled workflow

A common production policy:

~~~text
Feature branch
      ↓
Pull Request
      ↓
Tests
      ↓
Review
      ↓
Merge main
      ↓
Production pipeline
~~~

---

## 6. Build the Docker Image in CI

Example:

~~~bash
docker build -t myapp:1.0.0 .
~~~

The CI runner performs the build.

The important principle is:

> CI creates the deployable image; production should not need to rebuild it.

---

## 7. Tagging Images in CI

A strong approach is to use the Git commit SHA:

~~~text
myapp:a81ecfe
~~~

You can also use a release version:

~~~text
myapp:1.4.0
~~~

A pipeline may create both:

~~~text
myapp:1.4.0
myapp:a81ecfe
~~~

Both references can point to the same image content.

---

## 8. Why Commit SHA Tags Matter

Suppose production is running:

~~~text
myapp:a81ecfe
~~~

You can identify the source commit that produced the image.

Rollback can then be:

~~~text
Current → a81ecfe
Previous → 91bd221
~~~

This gives strong traceability.

---

## 9. Push the Image to a Registry

After building and tagging:

~~~bash
docker login
docker push username/myapp:a81ecfe
~~~

The registry becomes the artifact store.

~~~text
CI runner
   ↓
Docker image
   ↓
Registry
   ↓
Deployment server
~~~

---

## 10. Never Put Credentials in the Repository

Never commit registry passwords, access tokens, SSH keys, or cloud credentials.

Use:

~~~text
CI/CD secret store
       ↓
Pipeline
       ↓
Registry authentication
~~~

Secrets should be injected only where required.

---

## 11. Build Secrets

A build may sometimes need temporary credentials, such as access to a private package registry.

Avoid putting secrets directly into Dockerfile arguments or image layers.

Prefer BuildKit secret mounts when a build-time secret is genuinely required.

Conceptually:

~~~text
CI secret
   ↓
BuildKit secret mount
   ↓
Build step
   ↓
Secret not baked into final image
~~~

Runtime secrets are a separate concern and should be supplied at deployment/runtime.

---

## 12. Example GitHub Actions Flow

A simplified workflow:

~~~yaml
name: Docker CI

on:
  push:
    branches:
      - main

jobs:
  docker:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Test
        run: npm ci && npm test

      - name: Build image
        run: docker build -t myapp:${{ github.sha }} .

      - name: Login
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Push image
        run: |
          docker tag myapp:${{ github.sha }} username/myapp:${{ github.sha }}
          docker push username/myapp:${{ github.sha }}
~~~

Important concepts:

1. checkout
2. test
3. build
4. authenticate
5. tag
6. push

---

## 13. Jenkins Docker Pipeline

A typical Jenkins flow:

~~~text
GitHub
   ↓
Jenkins
   ↓
Checkout
   ↓
Test
   ↓
Docker Build
   ↓
Docker Push
   ↓
Deployment Server
~~~

Conceptual Jenkinsfile:

~~~groovy
pipeline {
    agent any

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Test') {
            steps {
                sh 'npm ci'
                sh 'npm test'
            }
        }

        stage('Build') {
            steps {
                sh 'docker build -t username/myapp:$GIT_COMMIT .'
            }
        }

        stage('Push') {
            steps {
                sh 'docker push username/myapp:$GIT_COMMIT'
            }
        }
    }
}
~~~

This is a learning example. Production Jenkinsfiles should handle credentials, failures, scanning, approvals, deployment, and rollback carefully.

---

## 14. Jenkins Credentials

Jenkins should store credentials in its credential store rather than inside the Jenkinsfile.

~~~text
Jenkins Credentials
       ↓
Pipeline
       ↓
Docker Registry
~~~

The Jenkinsfile should reference a credential ID instead of containing the secret.

Interview rule:

> **Configuration can be stored in source control; secrets should not be.**

---

## 15. CI Pipeline vs Deployment Pipeline

~~~text
CI
 ├── checkout
 ├── lint
 ├── test
 ├── build image
 └── scan image
          ↓
      Registry
          ↓
CD
 ├── pull exact image
 ├── deploy
 ├── health check
 └── rollback if required
~~~

---

## 16. Build Once, Deploy Many

Bad:

~~~text
CI builds image
Staging rebuilds image
Production rebuilds image
~~~

The resulting images may differ.

Better:

~~~text
CI
 ↓
Build image A
 ↓
Registry
 ↓
Staging uses image A
 ↓
Production uses image A
~~~

The artifact remains unchanged as it moves between environments.

---

## 17. Environment Configuration

The image should generally contain the application and runtime dependencies, not environment-specific secrets.

Example:

~~~text
Same image:
codebuddy-backend:a81ecfe

Development:
MONGO_URI=dev-db

Staging:
MONGO_URI=staging-db

Production:
MONGO_URI=prod-db
~~~

The image stays the same while configuration changes at runtime.

---

## 18. Database Deployment

Do not treat a production database like a disposable application container.

Application:

~~~text
Backend image
   ↓
Container
~~~

Database:

~~~text
Managed DB / persistent database
   ↓
Persistent storage
~~~

For CodeBuddy, MongoDB requires persistent data management and backup/recovery planning.

---

## 19. Health Checks After Deployment

A deployment pipeline should verify that the application is actually working.

~~~text
Deploy
  ↓
Wait
  ↓
Health endpoint
  ↓
HTTP 200?
  ├── Yes → success
  └── No  → failure / rollback
~~~

A backend can expose:

~~~text
GET /health
~~~

Health checks should verify meaningful application readiness, not merely that a process exists.

---

## 20. Rollback

Suppose:

~~~text
Current production:
myapp:a81ecfe

Previous:
myapp:91bd221
~~~

If the new release fails:

~~~text
a81ecfe
   ↓
failure
   ↓
rollback
   ↓
91bd221
~~~

Immutable, versioned images make rollback much easier.

---

## 21. Image Scanning

Before deployment, scan images for known vulnerabilities.

~~~text
Build image
   ↓
Security scan
   ↓
No critical issues
   ↓
Push
~~~

Possible tools include:

- Docker Scout
- Trivy
- cloud-provider image scanners
- registry-integrated scanners

Scanning complements secure coding and dependency management.

---

## 22. Docker CI/CD with CodeBuddy

A realistic CodeBuddy pipeline:

~~~text
Frontend repo
      │
      ▼
   CI/CD
      │
 ┌────┴────┐
 │         │
Test      Build
 │         │
 └────┬────┘
      ▼
Frontend image
      │
      ▼
   Registry
      │
      ▼
  Deployment

Backend repo
      │
      ▼
   CI/CD
      │
 ┌────┴────┐
 │         │
Test      Build
 │         │
 └────┬────┘
      ▼
Backend image
      │
      ▼
   Registry
      │
      ▼
  Deployment
~~~

Backend runtime configuration supplies database connection details, JWT configuration, payment configuration, frontend origin, and other secrets.

These should not be baked into the image.

---

## 23. Recommended Production Flow for Your Learning Path

For the Docker → Jenkins → Kubernetes learning path:

~~~text
Developer
   ↓
GitHub
   ↓
Jenkins
   ↓
Run tests
   ↓
Build Docker image
   ↓
Tag with Git SHA
   ↓
Security scan
   ↓
Push to registry
   ↓
Deploy
   ↓
Health check
   ↓
Rollback if required
~~~

Later Kubernetes can become the deployment layer:

~~~text
Jenkins
   ↓
Container Registry
   ↓
Kubernetes
   ↓
Pods
   ↓
Services / Ingress
~~~

---

## 24. Common Mistakes

### Mistake 1 — Building on production

Prefer production to run a tested image produced by CI/CD.

### Mistake 2 — Using latest

Use version or commit-based tags for traceability.

### Mistake 3 — Hard-coding secrets

Use CI/CD credentials and runtime secret management.

### Mistake 4 — Rebuilding per environment

Build once and promote the same image.

### Mistake 5 — Deploying without health checks

A successful Docker command does not necessarily mean the application is healthy.

### Mistake 6 — Ignoring rollback

Production deployment should have a known previous version.

### Mistake 7 — Treating database storage like image storage

Databases require persistent storage and backup/recovery planning.

### Mistake 8 — Ignoring image vulnerabilities

Container images and their OS/package dependencies need security scanning.

---

## 25. Best Practices

1. Keep Dockerfiles in source control.
2. Run tests before building/pushing production images.
3. Use immutable version or commit-SHA tags.
4. Build once and deploy the same image.
5. Keep secrets out of Git and Docker images.
6. Store CI/CD credentials securely.
7. Scan images for vulnerabilities.
8. Use health checks.
9. Keep a rollbackable previous image.
10. Separate application configuration from the image.
11. Restrict production deployment permissions.
12. Prefer least-privilege registry and deployment credentials.
13. Record deployment versions and image digests.
14. Do not rebuild application images directly on production servers.

---

# Interview Questions

### 1. What is CI/CD?

CI automatically validates code changes; CD automates delivery and/or deployment of validated artifacts.

### 2. Why use Docker in CI/CD?

Docker creates a portable, reproducible deployment artifact that can move across environments.

### 3. What does build once, deploy many mean?

Build one immutable image and promote that same image through environments instead of rebuilding it.

### 4. Why tag Docker images with a Git SHA?

It provides direct traceability from the deployed image to the source commit.

### 5. Where should Docker registry credentials be stored?

In the CI/CD platform's secure credential or secret store.

### 6. Should production build the Docker image?

Preferably no. CI/CD should build and test the image, then production should run that exact image.

### 7. How does rollback work with Docker images?

Deploy the previously known-good image tag or digest.

### 8. Why should environment variables not be baked into the image?

The same image should be reusable across environments while environment-specific configuration changes at runtime.

### 9. What is the purpose of a health check after deployment?

To verify that the deployed application is actually ready and functioning.

### 10. What is the role of a registry in CI/CD?

It stores the built image artifact so deployment systems can pull the exact version.

---

# Quick Revision

~~~text
Git
 ↓
CI
 ├── Test
 ├── Build
 └── Scan
 ↓
Docker Image
 ↓
Tag
 ↓
Registry
 ↓
CD
 ├── Pull
 ├── Deploy
 ├── Health Check
 └── Rollback
~~~

### Core concepts

~~~text
CI  = validate + build
CD  = deliver + deploy
Registry = store images
Git SHA = traceability
Health check = verify deployment
Rollback = restore known-good image
Build once = same artifact everywhere
~~~

---

# ⭐ Must Remember

1. **CI validates code and creates the deployable artifact.**
2. **CD delivers and/or deploys the validated artifact.**
3. **Build once, deploy the same image.**
4. **Use Git SHA or version tags instead of relying on latest.**
5. **Never bake production secrets into Docker images.**
6. **Store registry/Jenkins credentials in secure credential stores.**
7. **Push images to a registry before remote deployment.**
8. **Health checks verify that deployment actually works.**
9. **Immutable images make rollback easier.**
10. **Future Jenkins flow: GitHub → Jenkins → Test → Build → Scan → Registry → Deploy → Health Check → Rollback.**
