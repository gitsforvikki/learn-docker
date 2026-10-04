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
