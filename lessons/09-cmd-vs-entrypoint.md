# Lesson 09 — CMD vs ENTRYPOINT

## 1. Why CMD and ENTRYPOINT Matter

`CMD` and `ENTRYPOINT` define what happens when a container starts.

They are both **runtime instructions**. They do not execute while the image is being built.

The most important distinction is:

- **CMD** → provides a default command and/or default arguments.
- **ENTRYPOINT** → defines the main executable of the container.

Understanding how they work together is important for both Docker usage and interviews.

---

## 2. CMD

Example:

```dockerfile
CMD ["node", "server.js"]
```

When the container starts, Docker runs this command by default.

A user can override a Dockerfile's CMD when running the container.

Example:

```bash
docker run my-app:1.0 node test.js
```

The command supplied after the image name replaces the default CMD.

Think:

```text
CMD → default behavior
```

---

## 3. ENTRYPOINT

Example:

```dockerfile
ENTRYPOINT ["node"]
```

This defines the main executable.

If the container is run with:

```bash
docker run my-app:1.0 server.js
```

Docker effectively runs:

```text
node server.js
```

Arguments supplied after the image name are passed to the ENTRYPOINT.

Think:

```text
ENTRYPOINT → fixed main executable
```

---

## 4. CMD and ENTRYPOINT Together

This is one of the most important patterns.

Dockerfile:

```dockerfile
ENTRYPOINT ["node"]
CMD ["server.js"]
```

Default:

```bash
docker run my-app:1.0
```

Runs approximately:

```text
node server.js
```

If you run:

```bash
docker run my-app:1.0 worker.js
```

the CMD value is replaced, so Docker runs:

```text
node worker.js
```

Therefore:

```text
ENTRYPOINT → executable
CMD        → default arguments
```

---

## 5. CMD vs ENTRYPOINT

| CMD | ENTRYPOINT |
|---|---|
| Defines default command/arguments | Defines main executable |
| Easily overridden by docker run arguments | Usually remains the executable |
| Useful for default behavior | Useful when the container should behave like a specific executable |
| Can work alone | Can work alone |
| Often paired with ENTRYPOINT | Often paired with CMD |

---

## 6. Exec Form

The preferred form for both CMD and ENTRYPOINT is the **exec form**.

Example:

```dockerfile
CMD ["node", "server.js"]
ENTRYPOINT ["node"]
```

This uses JSON array syntax.

The command is executed directly rather than through a shell.

### Why exec form is preferred

It provides better process behavior and allows the application to receive signals directly.

This is especially important because the main application process normally becomes PID 1 inside the container.

---

## 7. Shell Form

Shell form looks like:

```dockerfile
CMD node server.js
```

or:

```dockerfile
ENTRYPOINT node server.js
```

Docker runs shell-form commands through a shell.

On Linux, this is generally:

```text
/bin/sh -c ...
```

Shell form can be useful when shell features are intentionally required, but exec form is generally preferred for application startup commands.

---

## 8. Signal Handling and PID 1

The main process in a container normally runs as **PID 1**.

For example:

```dockerfile
CMD ["node", "server.js"]
```

The Node.js process can become the container's main process.

This matters because PID 1 is responsible for receiving and handling termination signals.

With shell form:

```dockerfile
CMD node server.js
```

a shell may become the direct process instead of Node.js.

This can cause signal-handling and process-management issues if the application does not receive signals as expected.

Therefore, for normal application startup:

```dockerfile
CMD ["node", "server.js"]
```

is generally preferable.

---

## 9. CMD Alone

You can use CMD without ENTRYPOINT.

Example:

```dockerfile
FROM node:22-alpine

WORKDIR /app

COPY . .

CMD ["node", "server.js"]
```

Run normally:

```bash
docker run my-app
```

Runs:

```text
node server.js
```

Override it:

```bash
docker run my-app node worker.js
```

Runs:

```text
node worker.js
```

---

## 10. ENTRYPOINT Alone

You can also use ENTRYPOINT without CMD.

Example:

```dockerfile
FROM alpine

ENTRYPOINT ["echo"]
```

Run:

```bash
docker run my-app "Hello Docker"
```

The arguments are passed to the ENTRYPOINT:

```text
echo Hello Docker
```

This pattern is useful when the image represents a specific command-line tool.

---

## 11. ENTRYPOINT + CMD Pattern

A common pattern is:

```dockerfile
ENTRYPOINT ["node"]
CMD ["server.js"]
```

This gives:

- A fixed executable: `node`
- A default argument: `server.js`
- Ability to replace the default argument at runtime

Example:

```bash
docker run my-app
```

→ `node server.js`

```bash
docker run my-app worker.js
```

→ `node worker.js`

This pattern is useful when the image is designed around a particular executable.

---

## 12. Overriding ENTRYPOINT

ENTRYPOINT can be overridden at runtime using:

```bash
docker run --entrypoint /bin/sh my-app
```

This replaces the image's configured ENTRYPOINT.

This is useful for debugging or running an alternative executable.

---

## 13. Multiple CMD or ENTRYPOINT Instructions

A Dockerfile should normally have one effective CMD and one effective ENTRYPOINT.

If multiple CMD instructions are present, only the **last CMD** takes effect.

If multiple ENTRYPOINT instructions are present, only the **last ENTRYPOINT** takes effect.

Earlier instructions are effectively replaced.

---

## 14. CMD Is Not RUN

This distinction is extremely important.

### RUN

```dockerfile
RUN npm ci
```

Runs during **image build**.

### CMD

```dockerfile
CMD ["npm", "start"]
```

Runs when the **container starts**.

Flow:

```text
Dockerfile
   ↓
docker build
   ↓
RUN instructions execute
   ↓
Image created
   ↓
docker run
   ↓
CMD / ENTRYPOINT starts
   ↓
Application process
```

---

## 15. CMD vs ENTRYPOINT vs RUN

| Instruction | When | Purpose |
|---|---|---|
| RUN | Build time | Execute build/setup commands |
| CMD | Container startup | Default command/arguments |
| ENTRYPOINT | Container startup | Main executable |

Example:

```dockerfile
RUN npm ci
ENTRYPOINT ["node"]
CMD ["server.js"]
```

Means:

```text
RUN        → install dependencies while building
ENTRYPOINT → node is the executable
CMD        → server.js is the default argument
```

---

## 16. Practical Example — Node.js

Dockerfile:

```dockerfile
FROM node:22-alpine

WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .

ENTRYPOINT ["node"]
CMD ["server.js"]
```

Build:

```bash
docker build -t my-node-app:1.0 .
```

Default run:

```bash
docker run my-node-app:1.0
```

Equivalent command:

```text
node server.js
```

Run a different script:

```bash
docker run my-node-app:1.0 worker.js
```

Equivalent command:

```text
node worker.js
```

---

## 17. Choosing Between CMD and ENTRYPOINT

Use **CMD** when:

- You want a default command.
- Users should easily be able to replace the command.
- The image does not need a fixed executable.

Use **ENTRYPOINT** when:

- The image represents a specific executable/tool.
- You want a fixed main executable.
- Runtime arguments should be passed to that executable.

Use **ENTRYPOINT + CMD** when:

- You want a fixed executable with configurable default arguments.

---

## 18. Common Mistakes

### Mistake 1: Thinking CMD is mandatory

A Dockerfile can use ENTRYPOINT without CMD, and some images can rely on other runtime configuration.

### Mistake 2: Confusing CMD with RUN

RUN is build-time; CMD is runtime.

### Mistake 3: Using shell form without understanding PID 1

Shell form can introduce an intermediate shell and affect signal handling.

### Mistake 4: Using ENTRYPOINT when users need to replace the entire command

ENTRYPOINT is less convenient to override with ordinary arguments. Use CMD when command replacement is expected, or explicitly override ENTRYPOINT when needed.

### Mistake 5: Using shell syntax in exec form

This does not perform shell expansion:

```dockerfile
CMD ["echo", "$HOME"]
```

It passes `$HOME` literally unless a shell is explicitly invoked.

---

## 19. Interview Questions

### Q1. What is the difference between CMD and ENTRYPOINT?

CMD provides default command/arguments that can be replaced by the command supplied to `docker run`. ENTRYPOINT defines the main executable and is normally combined with CMD for default arguments.

### Q2. What happens when CMD and ENTRYPOINT are used together?

ENTRYPOINT provides the executable and CMD provides its default arguments.

### Q3. What is the difference between shell form and exec form?

Exec form uses JSON array syntax and runs the executable directly. Shell form runs through a shell.

### Q4. Why is exec form preferred?

It gives the application better process and signal behavior and avoids unnecessary shell involvement.

### Q5. Can CMD be overridden?

Yes. Arguments/commands supplied after the image name replace the Dockerfile's CMD.

### Q6. Can ENTRYPOINT be overridden?

Yes, using the `--entrypoint` option of `docker run`.

### Q7. Why is PID 1 important in Docker?

The main container process normally runs as PID 1 and has special responsibilities for signal handling and child-process management.

### Q8. What is the difference between RUN and CMD?

RUN executes while building the image; CMD defines what should run when the container starts.

---

## Quick Revision

```text
RUN
 ↓
Build time

ENTRYPOINT
 ↓
Main executable

CMD
 ↓
Default command / arguments

Exec form
 ↓
Direct process execution

Shell form
 ↓
Runs through a shell
```

### Must Remember

1. **RUN = build time.**
2. **CMD = default runtime command/arguments.**
3. **ENTRYPOINT = main runtime executable.**
4. **ENTRYPOINT + CMD = fixed executable + default arguments.**
5. **Exec form is generally preferred for application startup.**
6. **The main application process normally runs as PID 1.**
7. **CMD can be replaced by arguments supplied to `docker run`.**
8. **ENTRYPOINT can be overridden using `--entrypoint`.**
