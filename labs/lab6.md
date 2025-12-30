# Kubernetes Lab #6 — Horizontal Pod Autoscaling (HPA)

## 🎯 Goal

- Configure a Horizontal Pod Autoscaler (HPA) to automatically scale pods based on CPU usage
- Simulate CPU load and observe scaling behavior

## 🧩 Skills

- Scaling automation
- Metrics utilization
- Performance tuning
- Observability

## 🧰 Prerequisites

Before running this lab, ensure the metrics server is installed:

```bash
kubectl get pods -n kube-system | grep metrics
```

If no metrics server is installed:

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

Verify it's working:

```bash
kubectl top nodes
kubectl top pods
```

You should see CPU/memory usage values.

## Step 1 — App That Generates CPU Load

### File: `hpa-app.js`

```javascript
const express = require('express');
const app = express();
const port = 8080;

app.get('/', (req, res) => res.send('HPA demo is running 🚀'));

app.get('/load', (req, res) => {
  const end = Date.now() + 15000; // busy loop for 15 seconds
  while (Date.now() < end) {
    Math.sqrt(Math.random());
  }
  res.send('Generated CPU load for 15s 🔥');
});

app.listen(port, () => console.log(`HPA app listening on port ${port}`));
```

### File: `Dockerfile`

```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY hpa-app.js .
RUN npm install express
EXPOSE 8080
CMD ["node", "hpa-app.js"]
```

### Build and Push

```bash
docker build -t <your-dockerhub-user>/hpa-demo:v1 .
docker push <your-dockerhub-user>/hpa-demo:v1
```

## Step 2 — Deployment & Service

### File: `deployment-hpa.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hpa-demo
  labels:
    app: hpa-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hpa-demo
  template:
    metadata:
      labels:
        app: hpa-demo
    spec:
      containers:
      - name: hpa-demo
        image: <your-dockerhub-user>/hpa-demo:v1
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: 100m
          limits:
            cpu: 200m
```

### File: `svc-hpa.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: hpa-demo-svc
spec:
  selector:
    app: hpa-demo
  ports:
    - port: 80
      targetPort: 8080
  type: ClusterIP
```

### Apply

```bash
kubectl apply -f deployment-hpa.yaml
kubectl apply -f svc-hpa.yaml
```

## Step 3 — Create the Horizontal Pod Autoscaler

### File: `hpa.yaml`

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: hpa-demo
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: hpa-demo
  minReplicas: 1
  maxReplicas: 5
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

### Apply

```bash
kubectl apply -f hpa.yaml
kubectl get hpa
```

You should see:

```
NAME       REFERENCE            TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
hpa-demo   Deployment/hpa-demo  10%/50%   1         5         1          1m
```

## Step 4 — Generate Load

Let's simulate CPU load so the HPA scales up.

### Run a test Pod

```bash
kubectl run loader --image=busybox -- /bin/sh -c "while true; do wget -q -O- http://hpa-demo-svc/load; done"
```

### Watch scaling

```bash
kubectl get hpa -w
```

You'll see CPU utilization rising:

```
TARGETS   80%/50%   →   120%/50%
```

And soon:

```
REPLICAS: 3 → 4 → 5
```

You can also check:

```bash
kubectl get pods -l app=hpa-demo
```

You'll see more replicas created dynamically.

## Step 5 — Observe Cooldown

After you stop the load generator:

```bash
kubectl delete pod loader
```

Wait a few minutes and watch:

```bash
kubectl get hpa -w
```

Pods will gradually scale back down to 1.

## Step 6 — Cleanup

```bash
kubectl delete -f hpa.yaml
kubectl delete -f deployment-hpa.yaml
kubectl delete -f svc-hpa.yaml
```

## 💡 Tuning Tips

- For fast-reacting apps, use `--horizontal-pod-autoscaler-sync-period=10s` (controller manager flag)
- Combine with Cluster Autoscaler for node-level scaling
- Monitor with `kubectl top pods` and `kubectl describe hpa`

## ✅ What You've Mastered

- CPU metrics collection
- Autoscaling based on utilization
- Scaling behavior under load and cooldown
- HPA configuration and monitoring
- Load testing and performance observation

## 📚 Key Concepts

**minReplicas**: Minimum number of pods to maintain  
**maxReplicas**: Maximum number of pods to scale to  
**averageUtilization**: Target CPU percentage that triggers scaling  
**scaleTargetRef**: The deployment/resource to scale

