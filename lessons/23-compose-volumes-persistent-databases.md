# Lesson 23 — Compose Volumes & Persistent Databases

## 1. Why Database Persistence Matters

Containers are replaceable runtime environments. If a database stores data only in the container writable layer, removing that container can remove the data.

Therefore:

**Database containers should use persistent storage.**

~~~text
Database container
       |
       v
Persistent volume
       |
       v
Data survives container recreation
~~~

Persistence and backup are different concepts. A volume protects data from normal container replacement; it is not a complete disaster-recovery strategy.

## 2. Named Volumes in Compose

Example:

~~~yaml
services:
  database:
    image: postgres:18
    volumes:
      - postgres-data:/var/lib/postgresql/data

volumes:
  postgres-data:
~~~

The service mount connects the named volume to PostgreSQL's data directory. The top-level volume declaration tells Compose about the named volume.

## 3. What Happens During Compose Startup?

Run:

~~~bash
docker compose up -d
~~~

If the volume does not exist, Compose creates it.

Conceptually:

~~~text
compose.yaml
     |
     v
database service
     |
     v
postgres-data
     |
     v
/var/lib/postgresql/data
~~~

If the database container is recreated, the same named volume can be attached again.

## 4. Compose Down and Volumes

Normally:

~~~bash
docker compose down
~~~

removes the containers and Compose network but keeps named volumes.

Therefore:

~~~text
down
  |
  +--> containers removed
  |
  +--> network removed
  |
  +--> named volume remains
  |
  +--> database data remains
~~~

To explicitly remove associated volumes:

~~~bash
docker compose down -v
~~~

**Warning:** For databases, this can delete local persistent data.

## 5. Important Lifecycle Table

| Command | Containers | Network | Named volumes |
|---|---|---|---|
| docker compose up -d | Create/start | Create/use | Create/use |
| docker compose down | Remove | Remove | Keep |
| docker compose down -v | Remove | Remove | Remove |
| docker compose restart | Restart | Keep | Keep |

## 6. PostgreSQL Example

~~~yaml
services:
  database:
    image: postgres:18
    environment:
      POSTGRES_DB: app
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: change-me
    volumes:
      - postgres-data:/var/lib/postgresql/data

volumes:
  postgres-data:
~~~

The PostgreSQL data directory in the official image is:

~~~text
/var/lib/postgresql/data
~~~

For real projects, credentials should not be committed directly into Git.

## 7. Backend + PostgreSQL

~~~yaml
services:
  backend:
    build: ./backend
    environment:
      DATABASE_URL: postgresql://postgres:password@database:5432/app
    depends_on:
      - database

  database:
    image: postgres:18
    environment:
      POSTGRES_DB: app
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
    volumes:
      - postgres-data:/var/lib/postgresql/data

volumes:
  postgres-data:
~~~

The backend connects to:

~~~text
database:5432
~~~

It should not use localhost:5432 because localhost inside the backend container refers to the backend container itself.

## 8. MongoDB with Compose

MongoDB can use a named volume:

~~~yaml
services:
  mongo:
    image: mongo:8
    volumes:
      - mongo-data:/data/db

volumes:
  mongo-data:
~~~

The standard MongoDB data directory in the official image is:

~~~text
/data/db
~~~

For CodeBuddy, this is the normal pattern for keeping MongoDB data across container recreation.

## 9. Redis Persistence

Redis can also use persistent storage when the application requires it:

~~~yaml
services:
  redis:
    image: redis:8
    command: redis-server --appendonly yes
    volumes:
      - redis-data:/data

volumes:
  redis-data:
~~~

Whether Redis should persist depends on its role.

If Redis is only a disposable cache, persistence may not be necessary.

## 10. Named Volume vs Bind Mount

### Named volume

~~~yaml
volumes:
  - postgres-data:/var/lib/postgresql/data
~~~

Advantages:

- Docker manages the storage.
- Easy to reuse after container replacement.
- Less dependent on a particular host path.
- Usually the simpler choice for containerized databases.

### Bind mount

~~~yaml
volumes:
  - ./postgres-data:/var/lib/postgresql/data
~~~

Advantages:

- Host location is explicit.
- Useful when direct host filesystem access is intentionally required.

Disadvantages include host path, permissions, and portability concerns.

**Practical rule:** For most local Compose database setups, prefer a named volume unless you have a specific reason to manage the host directory yourself.

## 11. Volume Lifecycle

Think of a volume as an independent storage object:

~~~text
Volume
  |
  +---- Container A
  |
  +---- Container A removed
  |
  +---- Container B
  |
  v
Same data
~~~

The container can be replaced without automatically destroying the named volume.

## 12. Inspecting Volumes

List volumes:

~~~bash
docker volume ls
~~~

Inspect a volume:

~~~bash
docker volume inspect <volume-name>
~~~

Compose may create a project-scoped name such as:

~~~text
myproject_postgres-data
~~~

This helps separate volumes belonging to different Compose projects.

## 13. Reusing an Existing Volume

Create a volume:

~~~bash
docker volume create shared-postgres-data
~~~

Then configure:

~~~yaml
volumes:
  postgres-data:
    external: true
    name: shared-postgres-data
~~~

With an external volume, Compose expects the volume to already exist.

Use this intentionally because it changes the normal Compose-managed lifecycle.

## 14. Read-Only Volume Mounts

A volume can be mounted read-only:

~~~yaml
services:
  backend:
    volumes:
      - app-config:/app/config:ro

volumes:
  app-config:
~~~

The container can read the mounted data but cannot write to it.

Use read-only mounts when a service only needs to consume configuration or other data.

## 15. Database Initialization

Database images commonly initialize a new database when their data directory is empty.

For example, PostgreSQL can initialize the configured database and user.

Important:

**Initialization generally applies to a new or empty data directory.**

If the volume already contains an initialized database, changing initialization environment variables does not normally recreate that database.

## 16. Why Changing POSTGRES_PASSWORD May Not Reset the Database

Suppose the database was first initialized with:

~~~yaml
POSTGRES_PASSWORD: old-password
~~~

Later you change it to:

~~~yaml
POSTGRES_PASSWORD: new-password
~~~

If the same database volume is reused, PostgreSQL continues using the existing database.

For local development, if intentionally resetting the database is acceptable:

~~~bash
docker compose down -v
docker compose up -d
~~~

This deletes the associated volume, so use it only when data can be discarded.

## 17. Persistence Is Not Backup

A named volume protects against normal container replacement.

It does not protect against every failure:

~~~text
Database
   |
   v
Named volume
   |
   X
Host failure
   |
   X
Volume may be lost
~~~

Production databases need a backup and recovery strategy.

Remember:

**Persistence != Backup**

## 18. Database Health Checks

A database container can be running while the database is still starting.

Example:

~~~yaml
services:
  database:
    image: postgres:18
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
~~~

A backend can wait for the health condition:

~~~yaml
services:
  backend:
    depends_on:
      database:
        condition: service_healthy
~~~

Health checks improve startup coordination, while application retry logic is still valuable.

## 19. Complete PostgreSQL Example

~~~yaml
services:
  backend:
    build: ./backend
    environment:
      DATABASE_URL: postgresql://postgres:password@database:5432/app
    depends_on:
      database:
        condition: service_healthy
    networks:
      - private

  database:
    image: postgres:18
    environment:
      POSTGRES_DB: app
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
    volumes:
      - postgres-data:/var/lib/postgresql/data
    networks:
      - private
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

networks:
  private:

volumes:
  postgres-data:
~~~

Flow:

~~~text
backend
   |
   | database:5432
   v
PostgreSQL
   |
   v
postgres-data
~~~

## 20. CodeBuddy Database Architecture

For CodeBuddy:

~~~yaml
services:
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

Architecture:

~~~text
Backend container
       |
       v
MongoDB container
       |
       v
mongo-data volume
~~~

If the MongoDB container is recreated, the database data remains in the named volume.

## 21. Development Workflow

Start:

~~~bash
docker compose up -d
~~~

Check:

~~~bash
docker compose ps
~~~

View database logs:

~~~bash
docker compose logs database
~~~

Remove containers while keeping named volumes:

~~~bash
docker compose down
~~~

Start again:

~~~bash
docker compose up -d
~~~

The database can reuse its existing data.

Reset the local database intentionally:

~~~bash
docker compose down -v
docker compose up -d
~~~

## 22. Common Mistakes

### Mistake 1 — Storing database data only in the container

Data can disappear when the container is removed.

### Mistake 2 — Running docker compose down -v casually

It can remove local database volumes.

### Mistake 3 — Treating a volume as a backup

A volume is persistence, not a backup strategy.

### Mistake 4 — Expecting environment changes to reinitialize an existing database

Existing data directories are normally reused.

### Mistake 5 — Using bind mounts without understanding permissions

Host filesystem permissions can cause database startup failures.

### Mistake 6 — Exposing database ports unnecessarily

The backend can normally connect through the Compose network.

### Mistake 7 — Forgetting readiness

A running database container is not necessarily a ready database service.

## 23. Debugging Checklist

If database data appears missing:

1. Check services:
~~~bash
docker compose ps
~~~

2. Check volumes:
~~~bash
docker volume ls
~~~

3. Inspect the volume:
~~~bash
docker volume inspect <volume-name>
~~~

4. Check database logs:
~~~bash
docker compose logs database
~~~

5. Verify the expected volume is attached.

6. Check whether docker compose down -v was previously executed.

7. Verify the application is connecting to the correct service and database.

## 24. Interview Questions

### Q1. Why should databases use Docker volumes?

Volumes allow database data to survive container replacement.

### Q2. Does docker compose down delete named volumes?

Normally no. docker compose down -v explicitly removes the associated named volumes.

### Q3. Is a Docker volume a backup?

No. It provides persistence but is not a complete backup and disaster-recovery strategy.

### Q4. Why prefer named volumes for many Compose databases?

Docker manages the storage, and the setup is less dependent on host filesystem paths.

### Q5. What happens when a database volume already contains data?

The database normally starts using the existing data instead of performing first-time initialization again.

### Q6. Why might changing POSTGRES_PASSWORD not change an existing database?

Initialization settings are primarily applied when the database data directory is initialized. An existing volume contains the already initialized database.

### Q7. Does a backend need a published PostgreSQL port?

No. If both services share a Docker network, the backend can connect using database:5432.

### Q8. What is the difference between persistence and backup?

Persistence keeps data across container replacement. Backup creates recoverable copies for failure or disaster scenarios.

## 25. Quick Revision

~~~text
Container
    |
    v
Database
    |
    v
Named Volume
    |
    v
Persistent Data
~~~

Important commands:

~~~bash
docker compose up -d
docker compose down
docker compose down -v
docker compose ps
docker compose logs database
docker volume ls
docker volume inspect <volume>
~~~

Core rules:

~~~text
Database data -> persistent volume

down
  -> removes containers
  -> keeps named volumes

down -v
  -> removes containers
  -> removes associated named volumes

Persistence != Backup

Backend -> database:internal-port
~~~

## ⭐ Must Remember

1. **Database containers need persistent storage.**
2. **Named volumes survive normal container recreation.**
3. **docker compose down normally keeps named volumes.**
4. **docker compose down -v removes associated volumes and can delete database data.**
5. **A volume is not a backup.**
6. **Changing database initialization variables does not normally recreate an existing database.**
7. **Use service names and internal ports for backend-to-database communication.**
8. **Avoid publishing database ports unless host access is actually required.**
9. **Health checks help coordinate database readiness.**
10. **For CodeBuddy, MongoDB data should live in a named volume such as mongo-data.**

## 🎯 Interview Summary

> Docker Compose named volumes provide persistent storage for stateful services such as PostgreSQL and MongoDB. A named volume survives normal container removal and can be reattached to a replacement container. docker compose down normally keeps named volumes, while docker compose down -v removes them and can delete local database data. Persistence is not the same as backup, so production databases still need backups and recovery procedures. Services should communicate over the Compose network using service names and internal ports rather than published host ports.
