# Learn Docker 🐳

A lesson-by-lesson Docker learning path from fundamentals to production, followed by a complete CodeBuddy production workflow.

## Course Roadmap

| # | Section | Lessons |
|---|---|---|
| 1 | [Docker Fundamentals](#1-docker-fundamentals) | 01–05 |
| 2 | [Docker Images](#2-docker-images) | 06–10 |
| 3 | [Dockerizing Real Applications](#3-dockerizing-real-applications) | 11–14 |
| 4 | [Storage, Networking & Configuration](#4-storage-networking--configuration) | 15–18 |
| 5 | [Docker Compose](#5-docker-compose) | 19–24 |
| 6 | [Docker in Production](#6-docker-in-production) | 25–28 |
| 7 | [Advanced Docker](#7-advanced-docker) | 30–35 |
| 8 | [CodeBuddy Production Project](#8-codebuddy-production-project) | 36 |

> **Note:** Lesson 29 — Container Lifecycle was intentionally skipped during the original learning path. Lesson 36 was added later as the project-level production flow.

---

## 1. Docker Fundamentals

- [Lesson 01 — What is Docker?](lessons/01-what-is-docker.md)
- [Lesson 02 — Docker Architecture](lessons/02-docker-architecture.md)
- [Lesson 03 — Containers vs Images](lessons/03-containers-vs-images.md)
- [Lesson 04 — Docker CLI & Basic Commands](lessons/04-docker-cli-basic-commands.md)
- [Lesson 05 — docker run](lessons/05-docker-run-first-container.md)

## 2. Docker Images

- [Lesson 06 — Docker Images in Depth](lessons/06-docker-images-in-depth.md)
- [Lesson 07 — Dockerfile Fundamentals](lessons/07-dockerfile-fundamentals.md)
- [Lesson 08 — Building Your Own Image](lessons/08-building-your-own-image.md)
- [Lesson 09 — CMD vs ENTRYPOINT](lessons/09-cmd-vs-entrypoint.md)
- [Lesson 10 — Docker Image Optimization](lessons/10-docker-image-optimization.md)

## 3. Dockerizing Real Applications

- [Lesson 11 — Dockerizing Node.js + Express](lessons/11-dockerizing-node-express.md)
- [Lesson 12 — Dockerizing React / Next.js](lessons/12-dockerizing-react-nextjs.md)
- [Lesson 13 — Multi-Stage Builds](lessons/13-multi-stage-builds.md)
- [Lesson 14 — Development vs Production Dockerfiles](lessons/14-development-vs-production-dockerfiles.md)

## 4. Storage, Networking & Configuration

- [Lesson 15 — Docker Volumes](lessons/15-docker-volumes.md)
- [Lesson 16 — Bind Mounts](lessons/16-bind-mounts.md)
- [Lesson 17 — Docker Networking](lessons/17-docker-networking.md)
- [Lesson 18 — Environment Variables & Secrets](lessons/18-environment-variables-secrets.md)

## 5. Docker Compose

- [Lesson 19 — What is Docker Compose?](lessons/19-docker-compose.md)
- [Lesson 20 — Build a Full-Stack Application with Compose](lessons/20-full-stack-application-with-compose.md)
- [Lesson 21 — Compose Services](lessons/21-compose-services.md)
- [Lesson 22 — Compose Networking](lessons/22-compose-networking.md)
- [Lesson 23 — Compose Volumes & Persistent Databases](lessons/23-compose-volumes-persistent-databases.md)
- [Lesson 24 — Compose Development Workflow](lessons/24-compose-development-workflow.md)

## 6. Docker in Production

- [Lesson 25 — Docker Image Registries](lessons/25-docker-image-registries.md)
- [Lesson 26 — Docker + CI/CD](lessons/26-docker-ci-cd.md)
- [Lesson 27 — Docker Deployment](lessons/27-docker-deployment.md)
- [Lesson 28 — Reverse Proxy with Docker](lessons/28-reverse-proxy-with-docker.md)

## 7. Advanced Docker

- ~~Lesson 29 — Container Lifecycle~~ *(Skipped)*
- [Lesson 30 — Docker Resource Management](lessons/30-docker-resource-management.md)
- [Lesson 31 — Docker Health Checks](lessons/31-docker-health-checks.md)
- [Lesson 32 — Docker Logging & Debugging](lessons/32-docker-logging-debugging.md)
- [Lesson 33 — Docker Security](lessons/33-docker-security.md)
- [Lesson 34 — Docker BuildKit & Advanced Build Techniques](lessons/34-docker-buildkit-advanced-builds.md)
- [Lesson 35 — Docker Swarm & Kubernetes — Where Docker Fits](lessons/35-docker-swarm-kubernetes.md)

## 8. CodeBuddy Production Project

- [Lesson 36 — CodeBuddy: Complete Production Docker → Compose → CI/CD → Kubernetes Flow](lessons/36-codebuddy-production-flow.md)

---

## Lesson Format

Each lesson will be documented as a standalone Markdown file containing:

1. Concept / definition
2. Why it is needed
3. How it works
4. Important commands and syntax
5. Practical examples
6. Common mistakes
7. Best practices
8. Interview questions
9. Quick revision
10. ⭐ Must Remember
11. 🎯 Interview Summary

## Learning Progress

**Completed:** Docker lessons 1–28, 30–36  
**Skipped:** Lesson 29 — Container Lifecycle

---

## Repository Structure

```text
learn-docker/
├── README.md
├── lessons/
│   ├── 01-what-is-docker.md
│   ├── 02-docker-architecture.md
│   ├── 03-containers-vs-images.md
│   ├── ...
│   └── 36-codebuddy-production-flow.md
└── projects/
    └── codebuddy/
```

The README is the course index; detailed learning material belongs in the individual lesson files.
