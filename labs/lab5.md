# Kubernetes Lab #5 — Resource Requests & Limits (Performance & Scheduling)

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

// Endpoint to simulate CPU load
app.get('/load', (req, res) => {
  const end = Date.now() + 10000; // 10s busy loop
  while (Date.now() < end) {
    Math.sqrt(Math.random());
  }
  res.send('CPU load simulated for 10 seconds!');
});

app.get('/', (req, res) => res.send('Hello from CPU test app!'));

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
#3.Wait until it’s running
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
for i in {1..5}; do curl http://<NodeIP>:<NodePort>/load; done
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
