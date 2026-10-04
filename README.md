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

# 🐳 Lesson 5 — docker run Deep Dive

-p — Port Mapping ⭐

This is one of the most important concepts.

Suppose Nginx is listening inside the container on:

Port 80

Your computer is outside the container.

You want:

http://localhost:8080

to reach Nginx.

Use:
```bash
docker run -d \
  --name my-nginx \
  -p 8080:80 \
  nginx
```

---

### 🌐 Understanding `-p HOST_PORT:CONTAINER_PORT`

The exact syntax for port mapping is always:
```text
-p HOST_PORT:CONTAINER_PORT
```

Therefore, `-p 8080:80` means:
```text
      Host Machine                  Container
┌──────────────────────┐      ┌──────────────────┐
│  localhost           │      │                  │
│    :8080 ─────────────────> │  :80             │
│                      │      │    Nginx Server  │
└──────────────────────┘      └──────────────────┘
```

#### Why can't we simply use port 80 on the host?
You actually could:
```bash
docker run -d -p 80:80 nginx
```
Traffic flows directly: `localhost:80 ➔ container:80`.  
However, if another application (like a local web server or another container) is already using host port 80, Docker cannot bind to it, and the command will fail.

---

### 🏢 Host Port vs. Container Port

This distinction is extremely important for interviews and real-world deployments.

Suppose you run:
```bash
docker run -p 5000:3000 my-node-app
```

This maps the ports like this:
```text
     HOST                  CONTAINER
localhost:5000   ───────>    :3000
```
* The Node.js application inside the container is actively listening on **`container:3000`**.
* External users access your application via **`localhost:5000`**.

```text
User ➔ localhost:5000 ➔ Docker Routing ➔ container:3000 ➔ Node.js App
```

---

### ⚠️ A Common Beginner Mistake

Suppose your internal Node.js application specifies:
```javascript
app.listen(3000);
```

If you run this command, it works perfectly:
```bash
docker run -p 3000:3000 my-app
```

But if you try running this command, **it will fail**:
```bash
docker run -p 5000:5000 my-app
```

#### Why doesn't it work?
Because you explicitly told Docker to route traffic from **host port 5000** to **container port 5000**. However, your application inside the container isn't listening on port 5000; it is listening on **container port 3000**. The traffic goes to a dead end inside the container.

#### The Correct Fix:
To expose the app on port 5000 of your host computer, you must route it to the exact port the application expects inside:
```text
host 5000 ➔ container 3000
```
```bash
docker run -p 5000:3000 my-app
```
### 🔑 `-e` — Environment Variables

Applications frequently require environment variables to manage configurations, credentials, and settings safely, such as:
* `NODE_ENV`
* `PORT`
* `DATABASE_URL`
* `JWT_SECRET`
* `API_KEY`

Docker allows you to pass these variables directly into the container using the **`-e`** (or `--env`) flag.

#### Single Environment Variable:
```bash
docker run -e NODE_ENV=production nginx
```

#### Multiple Environment Variables:
You can use the flag multiple times in a single command to inject several variables:
```bash
docker run \
  -e NODE_ENV=production \
  -e PORT=3000 \
  -e API_KEY=xyz123 \
  my-app
```

---

### 🧹 `--rm` — Automatic Cleanup

By default, when a container stops running, it remains on your disk in a "Stopped" status (`Exited`) until you explicitly delete it with `docker rm`. 

If you are spinning up a temporary container for quick testing or one-off tasks, you can use the **`--rm`** flag:

```bash
docker run --rm nginx
```

#### The Difference:
* **Without `--rm`:** `Run ➔ Stop ➔ Container remains on disk (requires manual cleanup)`
* **With `--rm`:** `Run ➔ Stop ➔ Container is automatically deleted instantly`
### 🔄 `--restart` — Restart Policies

You can tell Docker how a container should behave when its internal processes exit or when the Docker daemon itself restarts (such as after a system reboot).

```bash
docker run -d \
  --restart unless-stopped \
  nginx
```

#### Common Restart Policies:
* **`no`** ➔ Do not automatically restart the container (Default).
* **`always`** ➔ Always restart the container if it stops. If the system reboots, it starts automatically.
* **`on-failure`** ➔ Restart only if the container exits due to an error (non-zero exit code).
* **`unless-stopped`** ➔ Always restart the container unless it was explicitly stopped by the user before the daemon restarted.

> 💡 **Tip:** Setting an explicit restart policy is a production-essential practice to ensure high availability for your applications.

---

### 🏁 Overriding the Container Command

The general syntax for starting a container is:
```text
docker run [OPTIONS] IMAGE [COMMAND] [ARG...]
```
You can optionally append a `[COMMAND]` at the very end to override the default process defined inside the image.

#### ⚠️ A Crucial Concept: Why do some containers exit immediately?
If you execute:
```bash
docker run ubuntu
```
You might expect a full operating system environment to stay running, but the container **exits immediately**. 

A container only stays alive as long as its **primary foreground process** is running.

```text
Nginx Container                  Ubuntu Container
  ├── Starts Nginx Process         ├── Starts default command (e.g., bash)
  ├── Process stays alive          ├── No interactive input/long-running task
  └── Container stays RUNNING      └── Process exits ➔ Container EXITS
```

#### 🕹️ Interactive Containers with `-it`
To keep an OS container like Ubuntu alive and interact with it, you must allocate a terminal and run an interactive shell:

```bash
docker run -it ubuntu bash
```

Let's break down exactly what this does:
* **`docker run`** ➔ Create and start the container.
* **`-i`** ➔ Interactive (keeps STDIN open).
* **`-t`** ➔ Allocates a pseudo-TTY (terminal).
* **`ubuntu`** ➔ The image template to use.
* **`bash`** ➔ The command overriding the default image entrypoint.

Your terminal prompt will change to:
```text
root@abc123:/#
```
You are now inside the running Ubuntu container and can safely execute commands like `ls` or `pwd`. Type `exit` to leave.

---

### 🧩 `docker run` — Putting It All Together

Now let's decode a fully featured production command:

```bash
docker run -d \
  --name my-api \
  -p 5000:3000 \
  -e NODE_ENV=production \
  --restart unless-stopped \
  my-node-app
```

#### The Complete Breakdown:
```text
 docker run ................ Create + start container
 ├── -d .................... Run in background (Detached mode)
 ├── --name my-api ......... Assign a custom name to the container
 ├── -p 5000:3000 .......... Map Host Port 5000 to Container Port 3000
 ├── -e NODE_ENV=prod ...... Inject an environment variable
 ├── --restart unless-st ... Apply the container restart policy
 └── my-node-app ........... The foundation Docker Image to execute
```

### 🌐 A Critical Docker Networking Issue: `0.0.0.0` vs `127.0.0.1`

Suppose your Node.js application runs inside Docker. In a standard local development environment, you might write:
```javascript
app.listen(3000);
```
When running inside a container, this is often not enough to allow external access. Instead, applications running inside Docker must be configured to explicitly listen on all network interfaces:
```javascript
app.listen(3000, "0.0.0.0");
```

#### Why is this necessary?
* **`127.0.0.1` (localhost)** inside a container refers strictly to the container's *own internal loopback interface*. If your app binds only to `127.0.0.1`, it will reject traffic routing in from the outside host via the Docker network.
* **`0.0.0.0`** tells the application to listen on *all available network interfaces* inside the container, allowing Docker to successfully forward inbound traffic from your host machine to your app.

> 💡 *We will cover this thoroughly in the dedicated Docker Networking lesson, so don't worry if this concept feels brand new!*

---

### 📋 `EXPOSE` vs `-p` (The Ultimate Interview Question)

This is one of the most frequent technical interview questions. You will eventually see a configuration file called a `Dockerfile` that contains the following instruction:
```dockerfile
EXPOSE 3000
```
It is a common mistake to assume that this command publishes the port to your computer. **It does not.**

#### The Key Differences:
* **`EXPOSE`**  
  ↳ **Documentation / Metadata.** It acts purely as a declaration or note stating that the application inside intends to use that port. It does not open or map anything on your host machine.
* **`-p` (or `--publish`)**  
  ↳ **Actual Network Action.** This is the runtime flag used during `docker run` that actually maps and opens up network access between your host machine and the container.

```text
EXPOSE 3000 ──────────> Documentation only (Metadata)
-p 5000:3000 ─────────> Active traffic routing (Host 5000 ➔ Container 3000)
```

> 💡 *We will study the `EXPOSE` instruction in deeper detail when we begin writing our own Dockerfiles!*

# Lesson 6 — Understanding Docker Images in Depth 🐳

Now we go one level deeper into Docker Images. This is an important topic because almost everything in Docker starts with an image.

### 1. What exactly is a Docker Image?

A Docker image is a **read-only package/template** containing everything needed to create a container.

For example, a Node.js image can contain:
* Node.js runtime
* Linux filesystem
* npm
* Application dependencies
* Your application code
* Configuration

Then Docker uses that image to create a container:

```text
             Docker Image
                  │
                  │ docker run
                  ▼
          ┌───────────────┐
          │   Container   │
          │               │
          │ Node.js App   │
          └───────────────┘
```

> 💡 **Think of it this way:**  
> **Image** = Blueprint  
> **Container** = Running instance of that blueprint  

---

### 2. What is actually inside an image?

Suppose you create this `Dockerfile`:

```dockerfile
FROM node:22

WORKDIR /app

COPY package*.json ./

RUN npm install

COPY . .

CMD ["node", "server.js"]
```

Docker doesn't treat this file as one single giant block. Instead, it creates **layers**.

Conceptually, the image structure looks like this:
```text
┌─────────────────────────────┐
│ CMD ["node", "server.js"]   │
├─────────────────────────────┤
│ COPY . .                    │
├─────────────────────────────┤
│ RUN npm install             │
├─────────────────────────────┤
│ COPY package*.json ./       │
├─────────────────────────────┤
│ WORKDIR /app                │
├─────────────────────────────┤
│ node:22 base image          │
└─────────────────────────────┘
```
These distinct slices are called **image layers**.

---

### 3. Why does Docker use layers?

Imagine you have a Node project structured like this:
```text
my-app/
├── package.json
├── package-lock.json
├── server.js
├── routes/
├── controllers/
└── models/
```

When you build your image for the first time, it executes sequentially:
```text
Downloading Node image ➔ Installing dependencies ➔ Copying application ➔ Building image
```

Now imagine you change only **one single file**, like `server.js`. If Docker had to redo every step from scratch, it would waste time downloading Node and re-running `npm install`.

Instead, Docker utilizes its **build cache**. It safely reuses the layers that haven't changed:
* `node:22` ➔ ✅ Cached
* `package.json` ➔ ✅ Cached
* `npm install` ➔ ✅ Cached

Docker will only rebuild the specific layer affected by your modified source code, making subsequent builds incredibly fast.

---

### 4. Image Layers are Read-Only

Every single layer inside an image is completely **immutable (read-only)**. When Docker creates a running container from that image, it dynamically injects a thin, temporary **writable layer** directly on top.

```text
             Container
┌──────────────────────────────┐
│  Writable Container Layer    │ ← All modifications happen here
├──────────────────────────────┤
│  Image Layer                 │
├──────────────────────────────┤
│  Image Layer                 │
├──────────────────────────────┤
│  Image Layer                 │
└──────────────────────────────┘
```

Suppose the underlying image contains a file at `/app/server.js`, and you decide to modify it inside the running container. Docker does not alter the original image layer. Instead, it copies the file up to the container's **writable layer** and modifies it there.

```text
Image ➔ Creates ➔ Container ➔ Writable Changes
```

---

### 5. Why shouldn't we store important data inside the container?

Because the container's writable layer is **temporary and tied directly to the lifecycle of that container instance**.

```text
Container
   │
   ├── Application
   ├── Logs
   └── Database Data
```

If you destroy the container using `docker rm my-container`, the writable layer is instantly wiped out, and **all stored data disappears permanently**. 
### 7. What is a Base Image?

A base image is the **starting point** for your custom image.

For example:
```dockerfile
FROM node:22
```

This instruction means: *"Start building my image on top of the existing `node:22` image."* The Node image itself is built on top of a foundational Linux-based operating system image.

Conceptually:
```text
Your application image
        ↓
     node:22
        ↓
   Linux base
```

---

### 🏷️ What is a Docker Image Tag?

When you pull or reference an image like this:
```bash
docker pull node:22
```
`22` is the **tag**. The general format is always:
```text
image-name:tag
```

Examples of standard tags:
* `node:22`
* `nginx:1.29`
* `ubuntu:24.04`
* `mongo:8`

You can also assign custom tags to your own images when building them:
```bash
docker build -t my-api:1.0 .
```
In this example:
* **`my-api`** = Image name
* **`1.0`** = Tag version

---

### ⚡ Docker Image Cache ⭐⭐⭐

Suppose you are using this `Dockerfile`:

```dockerfile
FROM node:22

WORKDIR /app

COPY package*.json ./

RUN npm install

COPY . .

CMD ["node", "server.js"]
```

#### First Build:
```bash
docker build -t my-api .
```
Docker executes every single instruction sequentially from top to bottom.

#### Subsequent Builds:
If you modify only `server.js` and run the build command again, Docker evaluates each layer. You will see output like this:
```text
=> [internal] load build definition from Dockerfile       ✅ CACHED
=> [internal] load .dockerignore                         ✅ CACHED
=> [1/5] FROM docker.io/library/node:22                  ✅ CACHED
=> [2/5] WORKDIR /app                                    ✅ CACHED
=> [3/5] COPY package*.json ./                           ✅ CACHED
=> [4/5] RUN npm install                                 ✅ CACHED
=> [5/5] COPY . .                                        🔄 EXECUTING...
```

Because the foundational layers, dependencies, and package configurations did not change, Docker reuses the **CACHED** layers instantly. It only re-runs the `COPY . .` instruction and any steps after it to capture your updated code change, saving significant development time.
