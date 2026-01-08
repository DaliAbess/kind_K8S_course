# Kubernetes Lab #5 — Resource Requests & Limits (Performance & Scheduling)

## 📖 What are Resource Requests & Limits?

Resource requests and limits are mechanisms in Kubernetes that control how much CPU and memory containers can consume. They ensure fair resource distribution, prevent resource starvation, and maintain cluster stability.

### The Two Key Concepts

```
┌────────────────────────────────────────────────────────────────┐
│                    RESOURCE REQUESTS                            │
├────────────────────────────────────────────────────────────────┤
│  What: Minimum guaranteed resources for a container            │
│  Purpose: Used by the scheduler to decide pod placement        │
│  Guarantee: Pod will always have AT LEAST this much            │
│  Example: requests.cpu: "100m", requests.memory: "128Mi"       │
└────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────┐
│                     RESOURCE LIMITS                             │
├────────────────────────────────────────────────────────────────┤
│  What: Maximum resources a container can use                   │
│  Purpose: Prevents containers from consuming excessive resources│
│  Enforcement: Container is throttled (CPU) or killed (Memory)  │
│  Example: limits.cpu: "200m", limits.memory: "256Mi"           │
└────────────────────────────────────────────────────────────────┘
```

### Resource Units

**CPU (measured in cores):**
```
1 CPU = 1000m (millicores/millicpu)

Examples:
├─ "100m"  = 0.1 CPU core (10% of one core)
├─ "500m"  = 0.5 CPU core (half a core)
├─ "1"     = 1 full CPU core
└─ "2000m" = 2 CPU cores
```

**Memory (measured in bytes):**
```
Units:
├─ Mi = Mebibytes (1 Mi = 1024 Ki = 1,048,576 bytes)
├─ Gi = Gibibytes (1 Gi = 1024 Mi)
├─ M  = Megabytes (1 M = 1000 K = 1,000,000 bytes)
└─ G  = Gigabytes (1 G = 1000 M)

Examples:
├─ "128Mi" = 134,217,728 bytes
├─ "256Mi" = 268,435,456 bytes
└─ "1Gi"   = 1,073,741,824 bytes
```

### How Scheduling Works

```
┌──────────────────────────────────────────────────────────────────┐
│                    CLUSTER NODES                                  │
│                                                                   │
│  Node A                 Node B                 Node C             │
│  ├─ Total: 4 CPU       ├─ Total: 4 CPU       ├─ Total: 2 CPU    │
│  ├─ Total: 8Gi RAM     ├─ Total: 8Gi RAM     ├─ Total: 4Gi RAM  │
│  ├─ Used:  2 CPU       ├─ Used:  3.5 CPU     ├─ Used:  1.8 CPU  │
│  └─ Used:  4Gi RAM     └─ Used:  6Gi RAM     └─ Used:  3Gi RAM  │
└──────────────────────────────────────────────────────────────────┘
                                ▲
                                │
                    ┌───────────┴───────────┐
                    │   Kubernetes Scheduler │
                    └───────────┬───────────┘
                                │
                        Evaluates Request
                                │
                                ▼
┌──────────────────────────────────────────────────────────────────┐
│                    NEW POD REQUEST                                │
│  Requests: CPU: 1, Memory: 2Gi                                   │
└──────────────────────────────────────────────────────────────────┘

Scheduler Decision:
├─ Node A: ✅ Available (2 CPU, 4Gi remaining)
├─ Node B: ❌ Not enough CPU (only 0.5 CPU remaining)
└─ Node C: ❌ Not enough resources (0.2 CPU, 1Gi remaining)

Result: Pod scheduled to Node A
```

### Resource Behavior: Requests vs Limits

```
┌────────────────────────────────────────────────────────────────┐
│              Container Resource Lifecycle                       │
└────────────────────────────────────────────────────────────────┘

Configuration:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "200m"
    memory: "256Mi"

┌─────────────────────────────────────────────────────────────────┐
│                        CPU USAGE                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  200m │────────────────────────LIMIT─────────────────────────   │
│       │                          ▲                               │
│  150m │                      ┌───┴───┐                          │
│       │                      │THROTTLE│                         │
│  100m │──REQUEST──┬─────────┴────────┴─────────────            │
│       │  ▲        │ Normal Usage                                │
│   50m │  │  ┌─────┴──┐                                          │
│       │  │  │        │                                          │
│    0m │──┴──┴────────┴──────────────────────────────────────   │
│       └──────────────────────────────────────────────────────   │
│         Guaranteed   Can use up to limit   Gets throttled       │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                      MEMORY USAGE                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  256Mi│────────────────────────LIMIT─────────────────────────   │
│       │                          ▲                               │
│  200Mi│                      ┌───┴────┐                         │
│       │                      │OOMKilled│                        │
│  128Mi│──REQUEST──┬─────────┴─────────┴────────────            │
│       │  ▲        │ Normal Usage                                │
│   64Mi│  │  ┌─────┴──┐                                          │
│       │  │  │        │                                          │
│    0Mi│──┴──┴────────┴──────────────────────────────────────   │
│       └──────────────────────────────────────────────────────   │
│         Guaranteed   Can grow to limit    Pod terminated!       │
└─────────────────────────────────────────────────────────────────┘
```

### Quality of Service (QoS) Classes

Kubernetes assigns QoS classes based on requests and limits:

```
┌─────────────────────────────────────────────────────────────────┐
│                      GUARANTEED                                  │
├─────────────────────────────────────────────────────────────────┤
│  Condition: requests = limits for ALL resources                 │
│  Priority: HIGHEST                                              │
│  Eviction: Last to be evicted                                   │
│                                                                  │
│  resources:                                                     │
│    requests:                                                    │
│      cpu: "500m"                                                │
│      memory: "256Mi"                                            │
│    limits:                                                      │
│      cpu: "500m"          ◄── Same as request                  │
│      memory: "256Mi"      ◄── Same as request                  │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                       BURSTABLE                                  │
├─────────────────────────────────────────────────────────────────┤
│  Condition: requests < limits OR only requests set              │
│  Priority: MEDIUM                                               │
│  Eviction: Evicted before Guaranteed                            │
│                                                                  │
│  resources:                                                     │
│    requests:                                                    │
│      cpu: "100m"                                                │
│      memory: "128Mi"                                            │
│    limits:                                                      │
│      cpu: "200m"          ◄── Higher than request              │
│      memory: "256Mi"      ◄── Higher than request              │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                      BESTEFFORT                                  │
├─────────────────────────────────────────────────────────────────┤
│  Condition: NO requests or limits set                           │
│  Priority: LOWEST                                               │
│  Eviction: First to be evicted under pressure                   │
│                                                                  │
│  resources: {}          ◄── Nothing specified                   │
└─────────────────────────────────────────────────────────────────┘
```

### What Happens When Limits Are Exceeded?

```
┌─────────────────────────────────────────────────────────────────┐
│                    CPU LIMIT EXCEEDED                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Container tries to use more CPU than limit                     │
│                     │                                            │
│                     ▼                                            │
│          ┌──────────────────────┐                               │
│          │   CPU THROTTLING     │                               │
│          └──────────────────────┘                               │
│                     │                                            │
│                     ▼                                            │
│  • Container slowed down (not killed)                           │
│  • Process continues running                                    │
│  • Performance degraded                                         │
│  • Visible in metrics as throttling                             │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                  MEMORY LIMIT EXCEEDED                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Container tries to use more memory than limit                  │
│                     │                                            │
│                     ▼                                            │
│          ┌──────────────────────┐                               │
│          │     OOMKilled        │                               │
│          │ (Out Of Memory)      │                               │
│          └──────────────────────┘                               │
│                     │                                            │
│                     ▼                                            │
│  • Container immediately terminated                             │
│  • Pod status: CrashLoopBackOff                                 │
│  • Event: "OOMKilled"                                           │
│  • Pod restarts automatically                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Pod Eviction Under Resource Pressure

```
Scenario: Node running low on memory

┌──────────────────────────────────────────────────────────────┐
│                     NODE UNDER PRESSURE                       │
│                                                               │
│  Available Memory: 512Mi (critically low!)                   │
└──────────────────────────────────────────────────────────────┘
                            │
                            ▼
              ┌─────────────────────────┐
              │   Kubelet Evaluates     │
              │   Eviction Candidates   │
              └─────────────────────────┘
                            │
              ┌─────────────┴─────────────┐
              │                           │
              ▼                           ▼
    ┌──────────────────┐      ┌──────────────────┐
    │  BestEffort Pods │      │  Burstable Pods  │
    │   (No limits)    │      │ (Using > request)│
    └────────┬─────────┘      └────────┬─────────┘
             │                         │
             ▼ EVICTED FIRST           ▼ EVICTED SECOND
                                       
                      ┌──────────────────┐
                      │ Guaranteed Pods  │
                      │ (Safe - Last)    │
                      └──────────────────┘
                              ▲
                              │
                     Protected from eviction
```

### Best Practices

```
┌─────────────────────────────────────────────────────────────────┐
│                   SETTING REQUESTS & LIMITS                      │
└─────────────────────────────────────────────────────────────────┘

1. START WITH MONITORING
   │
   ├─► Run app without limits
   ├─► Monitor actual usage: kubectl top pod
   ├─► Use metrics over days/weeks
   └─► Understand baseline and peaks

2. SET REQUESTS BASED ON BASELINE
   │
   ├─► requests = average usage
   ├─► Ensures predictable scheduling
   └─► Example: If avg is 80m CPU → set request: "100m"

3. SET LIMITS WITH HEADROOM
   │
   ├─► limits = peak usage + safety margin (20-50%)
   ├─► CPU: 2-4x requests (throttling is OK)
   ├─► Memory: 1.5-2x requests (OOMKill is NOT OK)
   └─► Example: peak 180m CPU → set limit: "250m"

4. NEVER SKIP MEMORY LIMITS
   │
   ├─► Memory leaks can crash nodes
   ├─► Always set limits for production
   └─► Monitor OOMKilled events

5. USE RESOURCE QUOTAS
   │
   ├─► Set namespace-level quotas
   ├─► Prevent resource exhaustion
   └─► Force teams to set requests/limits
```

### Common Scenarios

| Scenario | Configuration | Result |
|----------|--------------|--------|
| **Development Pod** | No requests/limits | BestEffort QoS, can use any free resources, first to be evicted |
| **Microservice (Prod)** | requests: 100m CPU, 128Mi<br>limits: 500m CPU, 512Mi | Burstable QoS, guaranteed minimum, can burst to handle spikes |
| **Database** | requests = limits<br>cpu: 2, memory: 4Gi | Guaranteed QoS, predictable performance, last to be evicted |
| **Batch Job** | requests: 500m CPU, 1Gi<br>limits: 2 CPU, 2Gi | Burstable, can use extra resources when available |
| **Memory Leak** | limits: memory: 256Mi | Pod killed with OOMKilled when exceeding 256Mi |

---

## 🎯 Goal

- Understand how resource requests and limits affect Pod scheduling
- Simulate high CPU/memory usage and observe throttling or eviction
- Tune limits to balance performance and stability

## 🧩 Skills

- Scheduling
- Resource quotas
- Performance tuning
- Troubleshooting OOMKilled & Throttling

## 🧰 Scenario

We'll deploy a simple CPU-intensive Node.js app and apply various requests/limits to see how Kubernetes behaves when the container exceeds its limits.

## Step 1 — App That Uses CPU

### File: `cpu-app.js`

```javascript

const express = require('express');
const app = express();
const port = 8080;

// Global array to simulate memory usage
const memoryLoad = [];

// Endpoint to simulate CPU load
app.get('/load', (req, res) => {
  const end = Date.now() + 10000; // 10s busy loop
  while (Date.now() < end) {
    Math.sqrt(Math.random());
  }
  res.send('CPU load simulated for 10 seconds!');
});

// Endpoint to simulate memory load
app.get('/memload', (req, res) => {
  // Allocate ~10MB per request
  const size = 10 * 1024 * 1024 / 8; // Number of floats
  for (let i = 0; i < size; i++) {
    memoryLoad.push(Math.random());
  }
  res.send(`Allocated more memory! Current array length: ${memoryLoad.length}`);
});

// Root endpoint
app.get('/', (req, res) => res.send('Hello from CPU + Memory test app!'));

// Start server
app.listen(port, () => console.log(`App running on port ${port}`));

```

### File: `Dockerfile`

```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY cpu-app.js .
RUN npm install express
EXPOSE 8080
CMD ["node", "cpu-app.js"]
```

### Build and Push

```bash
docker build -t <your-dockerhub-user>/cpu-demo:v1 .
docker push <your-dockerhub-user>/cpu-demo:v1
```

## Step 2 — Deployment with Resource Requests/Limits

### File: `deployment-resources.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cpu-demo
  labels:
    app: cpu-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cpu-demo
  template:
    metadata:
      labels:
        app: cpu-demo
    spec:
      containers:
      - name: cpu-demo
        image: <your-dockerhub-user>/cpu-demo:v1
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "200m"
            memory: "256Mi"
```

### Apply

```bash
kubectl apply -f deployment-resources.yaml
kubectl get pods -l app=cpu-demo
```

## Step 3 — Expose the App

### File: `svc-cpu.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: cpu-demo-svc
spec:
  selector:
    app: cpu-demo
  ports:
    - port: 80
      targetPort: 8080
  type: NodePort
```

### Apply

```bash
kubectl apply -f svc-cpu.yaml
kubectl get svc cpu-demo-svc
```

## Step 4 — Generate Load

### Get Node IP and test

```bash
curl http://<NodeIP>:<NodePort>/
curl http://<NodeIP>:<NodePort>/load
```

### Monitor CPU usage

While load runs, open another terminal:

```bash

#Install Metrics Server
#1.Install Metrics Server
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
#2.Patch Metrics Server (REQUIRED on Killercoda)
kubectl patch deployment metrics-server -n kube-system \
  --type=json \
  -p='[
    {"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}
  ]'
#3.Wait until it's running
kubectl get pods -n kube-system | grep metrics

#################
kubectl top pod -l app=cpu-demo
```

**Observe:** CPU usage. Even if the process tries to exceed 200m CPU, it'll be throttled by the kubelet.

## Step 5 — Test Memory Limit

### Edit deployment to make memory limit too low

```bash
kubectl edit deployment cpu-demo
```

Change to:

```yaml
limits:
  cpu: "200m"
  memory: "64Mi"
```

### Apply changes

```bash
kubectl rollout restart deployment cpu-demo
```

### Simulate load repeatedly

```bash
for i in {1..5}; do curl http://<NodeIP>:<NodePort>/memload; done
```

### Inspect the results

```bash
kubectl describe pod -l app=cpu-demo | grep -A3 "State"
kubectl get events --sort-by=.metadata.creationTimestamp
```

You'll likely see:

```
Reason: OOMKilled
```

💡 **This means the container exceeded its memory limit and was terminated.**

## Step 6 — Verify Scheduling Behavior

### Try to schedule a pod that requests too many resources

### File: `heavy-pod.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: heavy-pod
spec:
  containers:
  - name: heavy
    image: busybox
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests:
        cpu: "10"
        memory: "10Gi"
```

### Apply

```bash
kubectl apply -f heavy-pod.yaml
kubectl get pod heavy-pod -o wide
```

You'll see:

```
Status: Pending
Reason: Unschedulable
```

**This means the scheduler can't find a node with enough resources.**

## Step 7 — Cleanup

```bash
kubectl delete deployment cpu-demo
kubectl delete svc cpu-demo-svc
kubectl delete pod heavy-pod
```

## 💡 Tips

- **Use requests** for predictable scheduling
- **Use limits** to avoid noisy-neighbor issues
- **Monitor real usage** with `kubectl top pod` or metrics-server
- **In production**, define ResourceQuotas per namespace

## 📚 What You Learned

- How resource requests affect Pod scheduling decisions
- How resource limits prevent containers from consuming excessive resources
- What happens when containers exceed memory limits (OOMKilled)
- How CPU throttling works when limits are exceeded
- How to troubleshoot scheduling issues related to insufficient resources
- Best practices for setting requests and limits in production environments
