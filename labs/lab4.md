# Kubernetes Lab #4 — Liveness & Readiness Probes (Health Checks)

## 📖 What are Health Probes?

Health probes are diagnostic checks that Kubernetes performs on containers to determine their health status. They enable Kubernetes to automatically detect and recover from application failures, ensuring high availability and reliability.

### The Three Types of Probes

```
┌────────────────────────────────────────────────────────────────┐
│                    READINESS PROBE                              │
├────────────────────────────────────────────────────────────────┤
│  Question: "Is the container ready to serve traffic?"          │
│  Purpose: Controls when Pod receives traffic from Service      │
│  Action on Failure: Pod removed from Service endpoints         │
│  Container Status: Keeps running (not restarted)               │
│  Use Case: Startup delays, temporary unavailability            │
└────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────┐
│                     LIVENESS PROBE                              │
├────────────────────────────────────────────────────────────────┤
│  Question: "Is the container still alive and functioning?"     │
│  Purpose: Detects deadlocks, crashes, or unrecoverable states  │
│  Action on Failure: Container is restarted                     │
│  Container Status: Terminated and recreated                    │
│  Use Case: Deadlocks, memory leaks, application hangs          │
└────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────┐
│                     STARTUP PROBE                               │
├────────────────────────────────────────────────────────────────┤
│  Question: "Has the container finished starting up?"           │
│  Purpose: Protects slow-starting containers from early kills   │
│  Action on Failure: Container is restarted                     │
│  Container Status: Disables liveness/readiness until success   │
│  Use Case: Legacy apps, applications with long initialization  │
└────────────────────────────────────────────────────────────────┘
```

### Probe Configuration Parameters

```
┌──────────────────────────────────────────────────────────────┐
│              PROBE TIMING PARAMETERS                          │
└──────────────────────────────────────────────────────────────┘

initialDelaySeconds: 10
│
├─► Time to wait before performing the first probe
├─► Gives container time to start up
└─► Example: 10 = wait 10 seconds after container starts

periodSeconds: 5
│
├─► How often to perform the probe
├─► Affects how quickly failures are detected
└─► Example: 5 = check every 5 seconds

timeoutSeconds: 1
│
├─► How long to wait for probe response
├─► Probe fails if no response within timeout
└─► Example: 1 = wait 1 second for response

successThreshold: 1
│
├─► Consecutive successes needed to mark as healthy
├─► Usually 1 (except for readiness after failure)
└─► Example: 1 = one success is enough

failureThreshold: 3
│
├─► Consecutive failures before taking action
├─► Higher = more tolerant of transient failures
└─► Example: 3 = must fail 3 times in a row
```

### Probe Mechanisms

```
┌──────────────────────────────────────────────────────────────┐
│                  HTTP GET PROBE                               │
├──────────────────────────────────────────────────────────────┤
│  httpGet:                                                     │
│    path: /healthz                                             │
│    port: 8080                                                 │
│                                                               │
│  How it works:                                                │
│  ├─ Kubelet sends HTTP GET request                           │
│  ├─ Success: Status code 200-399                             │
│  └─ Failure: Status code ≥400 or no response                 │
└──────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│                    TCP PROBE                                  │
├──────────────────────────────────────────────────────────────┤
│  tcpSocket:                                                   │
│    port: 8080                                                 │
│                                                               │
│  How it works:                                                │
│  ├─ Kubelet attempts TCP connection                          │
│  ├─ Success: Connection established                          │
│  └─ Failure: Cannot connect                                  │
└──────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│                    EXEC PROBE                                 │
├──────────────────────────────────────────────────────────────┤
│  exec:                                                        │
│    command:                                                   │
│      - cat                                                    │
│      - /tmp/healthy                                           │
│                                                               │
│  How it works:                                                │
│  ├─ Kubelet executes command in container                    │
│  ├─ Success: Exit code 0                                     │
│  └─ Failure: Non-zero exit code                              │
└──────────────────────────────────────────────────────────────┘
```

### Pod Lifecycle with Probes

```
┌──────────────────────────────────────────────────────────────────┐
│                     POD LIFECYCLE TIMELINE                        │
└──────────────────────────────────────────────────────────────────┘

Time 0s:  Container starts
          │
          ├─► Status: Pending
          │
Time 5s:  initialDelaySeconds elapsed
          │
          ├─► Readiness Probe: Checking... ❌ (not ready)
          ├─► Liveness Probe: Checking... ✅ (alive)
          │   Status: Running but NOT READY
          │   Traffic: ❌ No traffic sent to pod
          │
Time 10s: Readiness check again
          │
          ├─► Readiness Probe: Checking... ✅ (ready!)
          │   Status: Running and READY
          │   Traffic: ✅ Pod added to Service endpoints
          │
Time 15s-60s: Normal operation
          │
          ├─► Readiness Probe: ✅ ✅ ✅ (every 5s)
          ├─► Liveness Probe: ✅ ✅ ✅ (every 5s)
          │   Traffic: ✅ Receiving requests
          │
Time 60s: Application crashes (simulated)
          │
          ├─► Liveness Probe: ❌ (unhealthy)
          │
Time 65s: Liveness check again
          │
          ├─► Liveness Probe: ❌ (still unhealthy)
          │
Time 70s: Liveness failureThreshold (2) reached
          │
          ├─► Action: CONTAINER RESTART
          │   Status: Container terminated, new one starting
          │   RESTARTS count: +1
          │
Time 75s: New container initializing...
          │
          └─► Cycle repeats from beginning
```

### Readiness Probe Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                   READINESS PROBE BEHAVIOR                       │
└─────────────────────────────────────────────────────────────────┘

                    ┌──────────────┐
                    │  Pod Starts  │
                    └──────┬───────┘
                           │
                           ▼
              ┌────────────────────────┐
              │ Readiness Probe Fails  │
              │   (not ready yet)      │
              └────────────┬───────────┘
                           │
                           ▼
         ┌──────────────────────────────────┐
         │   Pod Status: Running             │
         │   Ready: False                    │
         │   Service: Pod NOT in endpoints   │
         └──────────────┬───────────────────┘
                        │
                        │ Probe checks every periodSeconds
                        │
                        ▼
              ┌────────────────────────┐
              │ Readiness Probe Passes │
              │    (app ready!)        │
              └────────────┬───────────┘
                           │
                           ▼
         ┌──────────────────────────────────┐
         │   Pod Status: Running             │
         │   Ready: True                     │
         │   Service: Pod ADDED to endpoints │
         │   Traffic: ✅ Receiving requests  │
         └──────────────┬───────────────────┘
                        │
                        │ If probe fails again...
                        │
                        ▼
              ┌────────────────────────┐
              │ Readiness Probe Fails  │
              │  (temporary issue)     │
              └────────────┬───────────┘
                           │
                           ▼
         ┌──────────────────────────────────┐
         │   Pod Status: Running             │
         │   Ready: False                    │
         │   Service: Pod REMOVED            │
         │   Traffic: ❌ No new requests     │
         │   Container: STILL RUNNING        │
         └───────────────────────────────────┘
```

### Liveness Probe Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                    LIVENESS PROBE BEHAVIOR                       │
└─────────────────────────────────────────────────────────────────┘

                    ┌──────────────┐
                    │  Pod Running │
                    └──────┬───────┘
                           │
                           ▼
              ┌────────────────────────┐
              │ Liveness Probe Passes  │
              │  (app is healthy)      │
              └────────────┬───────────┘
                           │
                           │ Checking every periodSeconds
                           │
                           ▼
         ┌──────────────────────────────────┐
         │   Everything normal               │
         │   Probe: ✅ ✅ ✅                 │
         └──────────────┬───────────────────┘
                        │
                        │ Application deadlocks/crashes
                        │
                        ▼
              ┌────────────────────────┐
              │ Liveness Probe Fails   │
              │   Failure count: 1     │
              └────────────┬───────────┘
                           │
                           ▼
              ┌────────────────────────┐
              │ Liveness Probe Fails   │
              │   Failure count: 2     │
              └────────────┬───────────┘
                           │
                           │ failureThreshold reached!
                           │
                           ▼
         ┌──────────────────────────────────┐
         │   ACTION: RESTART CONTAINER       │
         │   - Container terminated          │
         │   - New container created         │
         │   - RESTARTS counter incremented  │
         │   - Pod state reset to init       │
         └───────────────────────────────────┘
```

### Service Endpoint Management

```
┌─────────────────────────────────────────────────────────────────┐
│              HOW READINESS AFFECTS TRAFFIC                       │
└─────────────────────────────────────────────────────────────────┘

┌──────────────────┐
│     Service      │
│  (LoadBalancer)  │
└────────┬─────────┘
         │
         │ Queries: Which pods are Ready?
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Endpoint Controller                           │
│  Monitors pod readiness and updates Service endpoints           │
└───────┬────────────────────────────┬───────────────────────────┘
        │                            │
        ▼                            ▼
┌──────────────┐              ┌──────────────┐
│   Pod A      │              │   Pod B      │
│ Ready: True  │              │ Ready: False │
│ ✅ Healthy   │              │ ❌ Starting  │
└──────────────┘              └──────────────┘
       │                              │
       │                              │
       ▼                              ▼
  IN endpoints                   NOT in endpoints
  Receives traffic               No traffic sent


Example Service endpoints:

Endpoints: 10.0.1.5:8080  ← Pod A (Ready)
          (Pod B excluded because not ready)


When Pod B becomes ready:

Endpoints: 10.0.1.5:8080  ← Pod A
           10.0.1.6:8080  ← Pod B (now added!)
```

### Common Probe Patterns

```
┌─────────────────────────────────────────────────────────────────┐
│                    PATTERN 1: BASIC SETUP                        │
├─────────────────────────────────────────────────────────────────┤
│  Use Case: Simple stateless application                         │
│                                                                  │
│  readinessProbe:                                                │
│    httpGet:                                                     │
│      path: /healthz                                             │
│      port: 8080                                                 │
│    initialDelaySeconds: 5                                       │
│    periodSeconds: 10                                            │
│                                                                  │
│  livenessProbe:                                                 │
│    httpGet:                                                     │
│      path: /healthz                                             │
│      port: 8080                                                 │
│    initialDelaySeconds: 15                                      │
│    periodSeconds: 20                                            │
│                                                                  │
│  Notes:                                                         │
│  ├─ Same endpoint for both (simple)                            │
│  ├─ Liveness delayed more (avoid restart during startup)       │
│  └─ Liveness checked less frequently (reduce overhead)         │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│              PATTERN 2: SLOW STARTING APPLICATION                │
├─────────────────────────────────────────────────────────────────┤
│  Use Case: Database, Java app with long initialization          │
│                                                                  │
│  startupProbe:                                                  │
│    httpGet:                                                     │
│      path: /startup                                             │
│      port: 8080                                                 │
│    failureThreshold: 30                                         │
│    periodSeconds: 10                                            │
│    # Total startup time: 30 * 10 = 300s (5 minutes)            │
│                                                                  │
│  readinessProbe:                                                │
│    httpGet:                                                     │
│      path: /ready                                               │
│      port: 8080                                                 │
│    periodSeconds: 5                                             │
│                                                                  │
│  livenessProbe:                                                 │
│    httpGet:                                                     │
│      path: /live                                                │
│      port: 8080                                                 │
│    periodSeconds: 10                                            │
│                                                                  │
│  Notes:                                                         │
│  ├─ Startup probe protects during initialization               │
│  ├─ Liveness/readiness disabled until startup succeeds         │
│  └─ Different endpoints for different health aspects           │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│              PATTERN 3: DEPENDENCY-AWARE READINESS               │
├─────────────────────────────────────────────────────────────────┤
│  Use Case: App depends on database, cache, external APIs        │
│                                                                  │
│  readinessProbe:                                                │
│    httpGet:                                                     │
│      path: /ready  # Checks DB, cache, dependencies            │
│      port: 8080                                                 │
│    periodSeconds: 5                                             │
│    failureThreshold: 3                                          │
│                                                                  │
│  livenessProbe:                                                 │
│    httpGet:                                                     │
│      path: /live   # Only checks app itself                    │
│      port: 8080                                                 │
│    periodSeconds: 10                                            │
│    failureThreshold: 3                                          │
│                                                                  │
│  Notes:                                                         │
│  ├─ Readiness checks dependencies (OK to fail temporarily)     │
│  ├─ Liveness only checks app health (avoid cascading restarts) │
│  └─ Pod removed from service if dependencies unavailable       │
└─────────────────────────────────────────────────────────────────┘
```

### Real-World Scenarios

```
┌──────────────────────────────────────────────────────────────┐
│   SCENARIO 1: Database Connection Lost                       │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  Problem: App loses DB connection temporarily                │
│                                                               │
│  Without Readiness Probe:                                    │
│  ├─ Service keeps sending traffic                            │
│  ├─ Requests fail with 500 errors                            │
│  └─ Poor user experience                                     │
│                                                               │
│  With Readiness Probe:                                       │
│  ├─ /ready endpoint checks DB connection                     │
│  ├─ Probe fails → Pod marked not ready                       │
│  ├─ Service stops sending traffic to this pod                │
│  ├─ Traffic routed to healthy pods                           │
│  └─ Pod self-heals when DB reconnects                        │
└──────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│   SCENARIO 2: Application Deadlock                           │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  Problem: App enters deadlock, stops responding              │
│                                                               │
│  Without Liveness Probe:                                     │
│  ├─ Container keeps running (but useless)                    │
│  ├─ Manual intervention required                             │
│  └─ Extended downtime                                        │
│                                                               │
│  With Liveness Probe:                                        │
│  ├─ /livez endpoint stops responding                         │
│  ├─ Probe fails repeatedly                                   │
│  ├─ Kubelet restarts container                               │
│  └─ Fresh container starts, problem resolved                 │
└──────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│   SCENARIO 3: Rolling Update                                 │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  Problem: Deploy new version without downtime                │
│                                                               │
│  How Readiness Helps:                                        │
│  ├─ New pod starts                                           │
│  ├─ Readiness probe fails during initialization              │
│  ├─ No traffic sent to new pod yet                           │
│  ├─ Old pod continues serving traffic                        │
│  ├─ New pod becomes ready                                    │
│  ├─ Traffic shifts to new pod                                │
│  ├─ Old pod terminated only after new pod is ready           │
│  └─ Zero-downtime deployment achieved                        │
└──────────────────────────────────────────────────────────────┘
```

### Troubleshooting Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| **CrashLoopBackOff** | Liveness probe failing immediately | Increase `initialDelaySeconds` and `failureThreshold` |
| **Pod never ready** | Readiness probe always failing | Check probe endpoint, verify app actually becomes ready |
| **Frequent restarts** | Liveness probe too aggressive | Increase `periodSeconds` and `failureThreshold` |
| **Slow rollouts** | Readiness takes too long | Optimize app startup, adjust probe timing |
| **Traffic sent to unready pods** | No readiness probe configured | Add readiness probe to deployment |
| **Cascade failures** | Liveness checks dependencies | Liveness should only check app, not dependencies |

---

## 🎯 Goal

- Add readiness and liveness probes to an application
- Simulate startup delay or crash to observe behavior
- Learn how K8s restarts unhealthy Pods and delays traffic until ready

## 🧩 Skills

- Pod health management
- Troubleshooting restart loops
- Tuning probe thresholds

## 🧰 Scenario

We'll deploy a small Node.js API that:

- Starts slowly (simulating boot-time delay)
- Exposes:
  - `/healthz` → readiness probe endpoint
  - `/livez` → liveness probe endpoint

## Step 1 — Node.js App

### File: `health-app.js`

```javascript
const express = require("express");
const app = express();
const port = 8080;

let healthy = true;
let ready = false;

// Simulate startup delay (10s)
setTimeout(() => {
  ready = true;
  console.log("✅ App is ready to receive traffic");
}, 10000);

// Liveness endpoint
app.get("/livez", (req, res) => {
  if (healthy) res.status(200).send("I'm alive!");
  else res.status(500).send("I'm unhealthy!");
});

// Readiness endpoint
app.get("/healthz", (req, res) => {
  if (ready) res.status(200).send("Ready!");
  else res.status(503).send("Not ready yet!");
});

// Simulate crash after 60s
setTimeout(() => {
  healthy = false;
  console.log("💥 Simulating crash (liveness will fail)");
}, 60000);

app.listen(port, () => console.log(`App running on port ${port}`));
```

### File: `Dockerfile`

```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY health-app.js .
RUN npm install express
EXPOSE 8080
CMD ["node", "health-app.js"]
```

### Build & Push

```bash
docker build -t <your-dockerhub-user>/health-demo:v1 .
docker push <your-dockerhub-user>/health-demo:v1
```

## Step 2 — Deployment with Probes

### File: `deployment-health.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: health-demo
  labels:
    app: health-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: health-demo
  template:
    metadata:
      labels:
        app: health-demo
    spec:
      containers:
      - name: health-demo
        image: <your-dockerhub-user>/health-demo:v1
        ports:
        - containerPort: 8080

        # --- Probes ---
        readinessProbe:
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
          failureThreshold: 3

        livenessProbe:
          httpGet:
            path: /livez
            port: 8080
          initialDelaySeconds: 15
          periodSeconds: 5
          failureThreshold: 2
```

### Apply

```bash
kubectl apply -f deployment-health.yaml
kubectl get pods -l app=health-demo
```

## Step 3 — Observe Pod Lifecycle

### Check probe statuses

```bash
kubectl describe pod -l app=health-demo | grep -A5 "Conditions"
kubectl get events --sort-by=.metadata.creationTimestamp
```

### Expected Behavior

- Pod stays `NotReady` for ~10s (until readiness passes)
- Pod becomes `Ready` after `/healthz` responds 200
- Around 60s later, `/livez` fails → container restarts

### Watch live

```bash
kubectl get pods -w
```

You'll see the `RESTARTS` count increase after the simulated crash.

## Step 4 — Expose and Test

### File: `svc-health.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: health-demo-svc
spec:
  selector:
    app: health-demo
  ports:
    - port: 80
      targetPort: 8080
  type: NodePort
```

### Apply

```bash
kubectl apply -f svc-health.yaml
kubectl get svc health-demo-svc
```

### Test endpoints

Get Node IP (or `minikube ip`) and test:

```bash
curl http://<NodeIP>:<NodePort>/healthz
curl http://<NodeIP>:<NodePort>/livez
```

## Step 5 — Tuning and Experimenting

Try making readiness stricter:

```bash
kubectl edit deployment health-demo
# change failureThreshold or periodSeconds
```

Observe rollout:

```bash
kubectl rollout restart deployment health-demo
```

## Step 6 — Cleanup

```bash
kubectl delete deployment health-demo
kubectl delete svc health-demo-svc
```

## 💡 Tuning Tips

- **initialDelaySeconds** – delay before first probe
- **periodSeconds** – probe interval
- **failureThreshold** – number of consecutive failures before action
- **Pro tip**: Combine `startupProbe` + `livenessProbe` for apps with long warm-up

## 📚 What You Learned

- How readiness probes control traffic flow to Pods
- How liveness probes trigger container restarts
- How to tune probe parameters for different application behaviors
- How Kubernetes maintains application health automatically
