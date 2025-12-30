# Kubernetes Lab #4 — Liveness & Readiness Probes (Health Checks)

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
