# 🚀 Lesson 1 — What is Docker?

Let's start from the absolute beginning. Imagine you build a **Node.js** application. Your application might require:
* Node.js 22
* npm
* Express
* MongoDB
* Environment variables
* Specific OS dependencies

It works perfectly on your machine. Then you give the project to another developer. They run:

```bash
npm install
npm run dev
```

And get:
* ❌ Node version mismatch
* ❌ Package version mismatch
* ❌ Environment variable missing
* ❌ OS dependency missing
* ❌ MongoDB not running

This is the classic **"It works on my machine"** problem. 

Docker helps solve this by packaging the application together with its runtime environment and dependencies into a standardized unit called a **container**.

### With Docker:
```text
┌──────────────────────────┐
│        Container         │
│                          │
│   Node.js                │
│   Dependencies           │
│   Application            │
│   Configuration          │
│                          │
└──────────────────────────┘
             ↓
     Runs consistently
```

You can then run that container on another machine that has Docker installed.

> 💡 **One important clarification:** Docker doesn't magically make every application environment identical. The image packages the application and its user-space dependencies, while the underlying host still matters for things such as CPU architecture, kernel capabilities, and hardware.

---

### 🧪 Your first mental exercise
Before we move to Lesson 2, make sure these three statements make sense:

* **Docker**  
  ↳ Tool/platform for packaging and running applications.
* **Container**  
  ↳ Isolated running environment for an application.
* **Image**  
  ↳ Template used to create containers.


  # What exactly is Docker?

Docker is a platform for **building, packaging, distributing, and running applications in containers**.

Think of a container as a **small isolated environment** in which your application runs.

For example:
```text
┌─────────────────────────────┐
│         Container           │
│                             │
│   Node.js                   │
│   npm dependencies          │
│   Your application          │
│   Required configuration    │
│                             │
└─────────────────────────────┘
```
You can take the same application image and run it on your local device.


---

### What does Docker actually give us?

Docker gives us several important benefits:

#### 📦 Packaging
We can package an application and all of its dependencies together.
```text
Application + Dependencies + Runtime ➔ Docker Image
```

#### 🔒 Isolation
Different applications can run in completely separate containers on the same system.
```text
┌──────────────────┐      ┌──────────────────┐
│   Container A    │      │   Container B    │
│   Node.js 20     │      │   Node.js 22     │
└──────────────────┘      └──────────────────┘
```
They can use different runtime versions without conflicts and without installing multiple versions directly into the host OS.

---

### The three words you must understand

You will hear these terms constantly: **Docker**, **Image**, and **Container**. At this stage, remember:

* **Docker**  
  ↳ The platform/tooling used to build, distribute, and run containers.
* **Image**  
  ↳ A read-only template/package used to create containers.  
  `Image ➔ docker run ➔ Container`
* **Container**  
  ↳ A running, isolated instance of an image.

For example, one single image can be used to spin up multiple distinct containers:
```text
nginx Image
    │
    ├──→ Container 1
    │
    ├──→ Container 2
    │
    └──→ Container 3
```
### Your first Docker mental model

For now, remember this core relationship:

```text
             DOCKER
                │
        ┌───────┴───────┐
        ↓               ↓
      IMAGE         CONTAINER
        │               │
     Template       Running
        │           application
        └──────┬────────┘
               │
          Application
```

And the overall workflow and purpose:

```text
 Application + Dependencies + Environment
                   ↓
              Docker Image
                   ↓
               Container
                   ↓
           Runs consistently
```
# Docker Architecture & How `docker run` Works

### Docker Client
When you type:
```bash
docker run nginx
```
The `docker` command is the **Docker CLI** (Command Line Interface). It is the interface you use to communicate with Docker. 

The CLI itself doesn't directly create the container. It sends requests to the Docker daemon.

```text
You
 ↓
Docker CLI
 ↓
"Hey Docker, run nginx"
```

---

### Docker Daemon
The Docker daemon is usually called **`dockerd`**. It is the process responsible for managing Docker objects such as:
* Containers
* Images
* Networks
* Volumes

Think of it as the worker/manager behind Docker:

```text
Docker CLI
     │
     │ request
     ↓
  dockerd
     │
     ├── Images
     ├── Containers
     ├── Networks
     └── Volumes
```
When you execute `docker run nginx`, the CLI communicates with `dockerd`, and `dockerd` performs the work.

---

### Docker Engine & API
You will hear the term **Docker Engine** frequently. At a high level, Docker Engine is the core Docker technology that provides the functionality to build and run containers.

A simplified mental model of Docker Engine:
```text
Docker Engine
      │
      ├── Docker daemon
      ├── Docker API
      └── Container runtime components
```
> 💡 **For interviews, understand:** Docker Engine is the core technology that enables Docker containers to be built and run.

#### Docker API
The Docker CLI communicates with the Docker daemon through an API. Conceptually:
```text
Terminal
   │
   │ docker run nginx
   ↓
Docker CLI
   │
   │ Docker API request
   ↓
Docker Daemon
   │
   ↓
Create / start container
```
This design means Docker isn't limited to the CLI; other applications can communicate with Docker using the Docker API as well.

---

### Docker Registry & Docker Hub
If you want to run `docker run nginx`, but you don't have the `nginx` image on your computer, where does Docker get it? From a **Docker registry**. A registry is a place where Docker images are stored.

* **Docker Hub** is a popular public Docker registry. Think of it somewhat like GitHub, but for container images.

---

### What happens when you run `docker run nginx`?
This is the most critical workflow to understand. Suppose you type `docker run nginx`:

#### Step 1 — Docker CLI receives the command
```text
You ➔ docker run nginx
```
The Docker CLI understands: *"The user wants to create and start a container from the nginx image."*

#### Step 2 — CLI communicates with Docker daemon
```text
Docker CLI ➔ Docker Daemon
```
The daemon checks whether the required image exists locally.

#### Step 3 — Docker checks local images
Suppose you don't have `nginx` locally. Docker needs to obtain it.

#### Step 4 — Docker contacts the registry
```text
Docker Daemon ➔ "Give me nginx image" ➔ Docker Hub
```
Docker downloads the image. This operation is essentially what `docker pull nginx` does explicitly.

#### Step 5 — Image is stored locally
```text
Local Docker ➔ nginx image
```
### Complete `docker run` Flow

```text
              docker run nginx
                     │
                     ↓
                Docker CLI
                     │
                     ↓
               Docker Daemon
                     │
              ┌──────┴──────┐
              │             │
     Image exists?       No image
              │             │
              │             ↓
              │        Docker Registry
              │             │
              │             ↓
              │        Download image
              │             │
              └──────┬──────┘
                     ↓
                nginx Image
                     │
                     ↓
              Create Container
                     │
                     ↓
               Start Container
                     │
                     ↓
               Running Nginx
```

#### Step 6 — Docker creates a container
The image is used as a template:
```text
nginx Image ➔ Create ➔ nginx Container
```
## `docker pull` vs `docker run`

This is another common beginner question. Let's break down the exact difference:

### `docker pull nginx`
**Download the nginx image.**
```text
Docker Hub ➔ nginx Image ➔ Your Computer
```
This command only fetches the image template. It **does not** start a container.

### `docker run nginx`
**Use the nginx image to create and start a container.**
* If the image isn't available locally, Docker will generally **pull it first**, then run it.

```text
docker pull nginx  ➔  Download image
docker run nginx   ➔  Download image (if missing) + Create Container + Start Container
```

---

### One Image ➔ Many Containers
This is an extremely important architectural concept. Suppose you have one single **nginx Image** stored on your machine. You can create multiple isolated running environments from it:

```text
nginx Image
     │
     ├── Container A (Running on Port 8080)
     ├── Container B (Running on Port 8081)
     └── Container C (Running on Port 8082)
```

All three containers are completely distinct running instances, but they all share the exact same foundation template.




# 🐳 Lesson 3 — Docker Images vs Containers ⭐

### 1. First, the simplest definition

Remember this:
* **Image** = blueprint/template
* **Container** = running instance created from that image

For example:
```text
        nginx IMAGE
        (template)
             │
       ┌─────┼─────┐
       ↓     ↓     ↓
 Container  Container  Container
    A          B          C
```
One image can create many containers.

---

### 2. What is a Docker Image?
A Docker image is a **read-only package/template** containing the files and metadata needed to create a container.

For example, the `node` image contains things needed to run Node.js applications.

---

### 3. What is a Container?
A container is a **created/running instance** of an image with its own writable container layer and isolated runtime environment.

For example:
```text
nginx Image
     │
     ↓
nginx Container
     │
     ↓
Nginx process running
```

So the sequence is:
```text
Image ➔ Create container ➔ Start container ➔ Application runs
```

---

### 4. Let's see it practically
If Docker is installed, run:
```bash
docker images
```
You might initially see something like:
```text
REPOSITORY   TAG       IMAGE ID       CREATED       SIZE
nginx        latest    abc123...      ...           ...
```
This shows **images**, not containers.

Now run:
```bash
docker ps
```
You might see:
```text
CONTAINER ID   IMAGE   COMMAND   STATUS   PORTS   NAMES
```
This shows **currently running containers**.

---

### 5. Let's create our first container
Run:
```bash
docker run nginx
```

Docker will follow this process:
```text
nginx image ➔ create container ➔ start container ➔ run nginx
```

Now open another terminal window and run:
```bash
docker ps
```
You should see something similar to:
```text
CONTAINER ID   IMAGE   STATUS          PORTS   NAMES
abc123         nginx   Up 10 seconds           ...
```
Notice something important: The container is using the `nginx` image.

---

### 6. Container name
By default, Docker generates a random name for your container. Instead, you can give it your own custom name using the `--name` flag:
```bash
docker run -d --name my-nginx nginx
```

---

### 7. One image ➔ multiple containers
This is an extremely important concept. You can spin up multiple isolated instances from a single base template. 

For example, you can run:
```bash
docker run -d --name nginx1 nginx
docker run -d --name nginx2 nginx
```

---

### 8. Image is read-only
This is a vital architectural concept. A Docker image is treated as **immutable (read-only)**. When Docker creates a container, it adds a temporary **writable layer** directly on top of the image structure.

Conceptually:
```text
┌─────────────────────────────┐
│  Container writable layer   │ ← changes occur here
├─────────────────────────────┤
│  nginx image layer          │ ← read-only
├─────────────────────────────┤
│  nginx image layer          │ ← read-only
├─────────────────────────────┤
│  Base image layer           │ ← read-only
└─────────────────────────────┘
```
This architecture is known as a **layered filesystem**. We will study image layers in much more detail in the upcoming image-specific lessons.

### The Complete Lifecycle

Now put everything together:

```text
             Docker Image
                  │
                  │ docker run
                  ↓
          Container Created
                  │
                  ↓
          Container Running
                  │
             docker stop
                  ↓
          Container Stopped
                  │
             docker start
                  ↓
          Container Running
                  │
              docker rm
                  ↓
          Container Deleted
```
# Important Docker Commands

```bash
# List images
docker images

# List running containers
docker ps

# List all containers
docker ps -a

# Create + start container
docker run nginx

# Run in background
docker run -d nginx

# Give container a name
docker run -d --name my-nginx nginx

# Stop
docker stop my-nginx

# Start stopped container
docker start my-nginx

# Remove container
docker rm my-nginx

# Force remove running container
docker rm -f my-nginx
```


# 🐳 Lesson 4 — Docker Installation & Essential CLI Commands

### 1. Check whether Docker is installed

Run the following command in your terminal:
```bash
docker --version
```

You should get a response similar to:
```text
Docker version 29.x.x, build ...
```
This confirms that the Docker CLI is installed and ready to use.

---

### 2. View logs with `docker logs`

This is one of the most important commands for troubleshooting and debugging your applications.

First, start an Nginx container in the background:
```bash
docker run -d --name my-nginx nginx
```

Then, view the output logs generated by that container:
```bash
docker logs my-nginx
```
Docker will display the standard output and error streams (logs) directly from the running container.

---

### 3. Run commands inside a container with `docker exec`

This is another extremely important command. It allows you to execute commands inside an already running container instance.

Suppose your container is running (verify with `docker ps`). You can list the files inside it by running:
```bash
docker exec my-nginx ls
```
Docker executes the `ls` command directly inside the container environment and prints the results to your terminal.

#### 🐚 Open an interactive shell inside a container
If you need to explore or debug inside the container dynamically, you can attach an interactive terminal:
```bash
docker exec -it my-nginx bash
```

Let's break down the components of this command:
* **`docker exec`** ➔ Execute a command inside a running container.
* **`-it`** ➔ Interactive terminal (`-i` keeps STDIN open, `-t` allocates a pseudo-TTY).
* **`bash`** ➔ The shell program you want to start inside the container.

Once executed, your prompt will change to look similar to this:
```text
root@abc123:/#
```
You are now inside the container filesystem! You can safely test standard terminal commands:
```bash
ls
pwd
exit
```
*(Typing `exit` safely disconnects you from the container shell without stopping the application).*

> ⚠️ **What if Bash doesn't exist?**  
> Some minimal images (like those optimized for performance or security) do not include `bash`. For example, **Alpine-based images** often use standard **`sh`**.  
> If `bash` fails, simply switch to `sh`:
> ```bash
> docker exec -it <container-name> sh
> ```
> This is a very common practical issue and a frequent interview talking point!


### 🔍 Deep Dive: `docker inspect`

This command gives detailed information about a Docker object.

```bash
docker inspect my-nginx
```

You will get a large JSON structure containing comprehensive information, such as:
* Container ID
* Image reference
* Network configuration
* Mounts (Volumes)
* Environment variables
* IP address
* Port bindings
* State (Running, Stopped, OOMKilled, etc.)

> 💡 **Core Concept:** `docker inspect` = *"Give me detailed metadata about this specific Docker object."* It is incredibly useful for deep troubleshooting and debugging.

#### Inspecting an Image
You can also inspect images to see their layers, environment defaults, and configuration:

```bash
docker image inspect nginx
```

Summary:
* **`docker inspect my-nginx`** ➔ Fetches detailed **container** metadata.
* **`docker image inspect nginx`** ➔ Fetches detailed **image** metadata.

---

### 🌐 Port Mapping

By default, applications running inside a container are isolated and cannot be accessed from outside the container. Port mapping bridges this gap.

Run this command:
```bash
docker run -d --name my-nginx -p 8080:80 nginx
```

Here is how the traffic flows:
```text
  Host Machine                     Container
┌────────────────┐               ┌───────────┐
│ localhost:8080 │ ────────────> │  Port 80  │ ──> Nginx Server
└────────────────┘   -p 8080:80  └───────────┘
```

Now open your web browser and navigate to:
```text
http://localhost:8080
```
You should see the official Nginx welcome page!

---

### 🧠 Core Commands to Remember

Do not try to memorize dozens of niche flags right away. Focus heavily on mastering this essential core set:

```bash
# Get detailed JSON metadata about a container
docker inspect <container_name>

# Get detailed JSON metadata about an image
docker image inspect <image_name>

# View running container logs
docker logs <container_name>

# Execute a one-off command inside a running container
docker exec <container_name> <command>

# Open an interactive shell inside a running container
docker exec -it <container_name> bash

# Run a container in the background with port mapping and a custom name
docker run -d --name <custom_name> -p <host_port>:<container_port> <image_name>
```
