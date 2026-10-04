# Lesson 30 — Docker Resource Management

## 1. Why Resource Management Matters

Containers share the host's CPU, memory, storage, and other resources.

Without limits, one container can consume too many resources and affect other applications.

~~~text
Host
├── Container A → CPU / Memory
├── Container B → CPU / Memory
└── Container C → CPU / Memory
~~~

Resource management helps with:
- Stability
- Predictable performance
- Multi-container workloads
- Preventing one service from exhausting host resources
- Capacity planning

## 2. Main Resources

The most important Docker resource controls are:
- CPU
- Memory
- Block I/O
- Process/PID count

For common web applications, CPU and memory are the highest-priority controls.

## 3. Memory Limits

Example:

~~~bash
docker run -d \
  --name api \
  --memory=512m \
  node:22-alpine
~~~

The container has a memory limit of 512 MB.

If the workload exceeds available memory, it can be terminated by the host's out-of-memory mechanism.

Check usage:

~~~bash
docker stats
~~~

## 4. Memory Reservation

A reservation represents a softer expected/reserved memory level.

~~~bash
docker run -d \
  --memory=1g \
  --memory-reservation=512m \
  my-app
~~~

~~~text
Reservation → planned/expected capacity
Limit       → maximum allowed usage
~~~

A reservation is not the same as a hard memory limit.

## 5. CPU Limits

Limit CPU usage:

~~~bash
docker run -d \
  --cpus=1.5 \
  my-app
~~~

Docker also provides lower-level controls such as CPU shares, quota, and period.

For most applications, the cpus option is easier to understand and use.

## 6. CPU Pinning

A container can be restricted to specific CPUs:

~~~bash
docker run -d \
  --cpuset-cpus="0,1" \
  my-app
~~~

CPU pinning is useful for specialized workloads but is not normally required for ordinary web applications.

## 7. PID Limits

Limit the number of processes a container can create:

~~~bash
docker run -d \
  --pids-limit=100 \
  my-app
~~~

This can reduce the impact of runaway process creation.

## 8. Block I/O

Docker provides controls such as:
- device-read-bps
- device-write-bps
- device-read-iops
- device-write-iops

These are specialized and depend on the host storage setup.

For normal web applications, CPU and memory are usually more important.

## 9. Docker Stats

The most useful runtime command:

~~~bash
docker stats
~~~

For one container:

~~~bash
docker stats api
~~~

It shows:
- CPU percentage
- Memory usage
- Memory limit
- Network I/O
- Block I/O
- Process count

## 10. Inspect Resource Configuration

Use:

~~~bash
docker inspect api
~~~

This is useful when debugging why a container behaves differently from another one.

## 11. Resource Limits in Compose

A practical Compose example:

~~~yaml
services:
  backend:
    image: my-backend:1.0
    mem_limit: 512m
    cpus: 1.0
~~~

For Docker Swarm, resource reservations and limits are commonly expressed under deploy.resources:

~~~yaml
services:
  backend:
    image: my-backend:1.0
    deploy:
      resources:
        limits:
          cpus: "1.0"
          memory: 512M
        reservations:
          cpus: "0.25"
          memory: 256M
~~~

The exact supported options depend on the Compose specification and deployment mode.

## 12. Limits vs Reservations

### Limit

The maximum resource amount the workload is allowed to consume.

### Reservation

The planned/expected resource capacity considered by the deployment platform.

~~~text
Reservation → planned capacity
Limit       → maximum usage
~~~

Do not treat them as interchangeable.

## 13. Why Limits Matter in Production

Suppose:

~~~text
Server memory = 4 GB

Backend → 3.5 GB
MongoDB → 1 GB
Nginx   → 200 MB
~~~

Possible result:

~~~text
Memory pressure
      ↓
OOM
      ↓
Processes/containers affected
      ↓
Application outage
~~~

Resource limits help make capacity requirements explicit.

## 14. Do Not Make Limits Too Small

A limit must match the real workload.

~~~text
Node.js application
Memory limit = 128 MB
~~~

If the application needs 300 MB, the limit can cause failures.

Correct process:

~~~text
Measure
  ↓
Understand workload
  ↓
Set reasonable limits
  ↓
Monitor
  ↓
Adjust
~~~

## 15. Node.js and Container Memory

Node.js has its own memory behavior.

The container memory limit and Node.js heap are related but not identical.

Example:

~~~bash
NODE_OPTIONS=--max-old-space-size=384
~~~

Do not set this value blindly.

~~~text
Container limit = 512 MB
Node heap ≠ entire container memory
~~~

The process also needs memory for native allocations, buffers, libraries, and runtime overhead.

## 16. OOM and OOMKilled

Inspect a container:

~~~bash
docker inspect api
~~~

Check whether it was killed because of memory exhaustion:

~~~bash
docker inspect -f '{{.State.OOMKilled}}' api
~~~

A result of true indicates an out-of-memory kill.

## 17. CPU Throttling

CPU limits can cause throttling.

Possible symptoms:
- Higher response latency
- Slow background jobs
- Lower throughput
- Increased request queueing

A CPU limit is a control, not a performance guarantee.

## 18. Database Resource Management

Databases need careful sizing.

~~~text
MongoDB
   ↓
Memory-heavy workload
   ↓
Needs sufficient RAM
~~~

Do not give a database an arbitrary tiny memory limit without understanding its workload.

Managed databases can simplify resource management, backups, and scaling.

## 19. CodeBuddy Example

Simplified deployment:

~~~text
Host
├── Nginx
├── CodeBuddy Frontend
├── CodeBuddy Backend
└── MongoDB
~~~

Example starting point:

~~~yaml
services:
  nginx:
    image: nginx:alpine

  backend:
    image: codebuddy-backend:1.0
    mem_limit: 512m
    cpus: 1.0

  frontend:
    image: codebuddy-frontend:1.0
    mem_limit: 256m
    cpus: 0.5
~~~

These values are examples, not universal production recommendations. Measure actual usage before choosing final limits.

## 20. Resource Management Workflow

~~~text
Deploy
  ↓
Measure with docker stats
  ↓
Observe CPU / memory
  ↓
Identify peaks
  ↓
Set limits
  ↓
Test under load
  ↓
Monitor
  ↓
Tune
~~~

This is better than guessing resource values.

## 21. Common Mistakes

1. No memory limits on a multi-service host.
2. Limits that are too small.
3. Confusing reservation and limit.
4. Ignoring monitoring.
5. Treating CPU limits as guaranteed CPU.
6. Giving databases arbitrary limits.

## 22. Best Practices

1. Measure before setting final limits.
2. Set reasonable memory limits for production workloads.
3. Set CPU limits when appropriate.
4. Monitor with docker stats and production monitoring tools.
5. Investigate OOMKilled containers.
6. Consider process limits for workloads that can create many processes.
7. Leave enough host resources for the OS and Docker itself.
8. Test under realistic load.
9. Tune limits based on real metrics.
10. Treat database workloads separately from stateless application containers.

# Interview Questions

### 1. Why are Docker resource limits important?

They prevent a container from consuming an uncontrolled amount of host resources and affecting other workloads.

### 2. How do you limit container memory?

Use the memory option.

### 3. How do you limit CPU?

Use the cpus option or lower-level CPU quota controls.

### 4. What is the difference between a limit and a reservation?

A limit defines a maximum; a reservation represents planned or expected capacity depending on the deployment platform.

### 5. How do you monitor Docker resource usage?

Use docker stats and production monitoring systems.

### 6. What does OOMKilled mean?

The container/process was terminated because of an out-of-memory condition.

### 7. Why can a CPU limit affect application latency?

The process may be throttled when it reaches its CPU allowance.

### 8. Should every container have the same resource limits?

No. Limits should reflect each workload's actual requirements.

### 9. Why should databases be sized differently from APIs?

Databases often have memory-intensive caches and storage workloads that require different resource characteristics.

### 10. What is the correct way to choose resource limits?

Measure real usage, identify peaks, set reasonable limits, load-test, monitor, and tune.

# Quick Revision

~~~text
Host resources
     ↓
Containers share resources
     ↓
Set reasonable limits
     ↓
Monitor usage
     ↓
Tune based on metrics
~~~

Important commands:

~~~bash
docker stats
docker stats <container>
docker inspect <container>
docker inspect -f '{{.State.OOMKilled}}' <container>
~~~

Important options:

~~~text
--memory
--memory-reservation
--cpus
--cpu-shares
--cpu-period
--cpu-quota
--cpuset-cpus
--pids-limit
~~~

# ⭐ Must Remember

1. **Containers share host resources.**
2. **Memory and CPU limits are the most important resource controls for common applications.**
3. **Memory limits control the maximum memory available to a container.**
4. **The cpus option provides an easy CPU limit.**
5. **docker stats is the first command to inspect runtime resource usage.**
6. **OOMKilled means the workload was terminated because of memory exhaustion.**
7. **A resource limit is not a performance guarantee.**
8. **Reservations and limits are different concepts.**
9. **Do not guess production limits—measure, test, monitor, and tune.**
10. **Leave enough resources for the host OS and Docker itself.**
11. **Database workloads need careful, workload-specific sizing.**
12. **Resource management becomes increasingly important when multiple services share one host.**
