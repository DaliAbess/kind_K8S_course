# Kubernetes Hands-on Practice Tasks (Lab #2)

## 🧱 Application Deployment Scenario

### 🎯 Goal

1. Deploy a simple Node.js API (`/api/hello`) using `myapp:v1`
2. Expose it via:
   - **ClusterIP** (internal access)
   - **NodePort** (external)
   - **LoadBalancer** (cloud)
3. Update the app image to `myapp:v2` (simulate version upgrade)
4. Test availability with curl and browser

### 🧩 Skills Covered
Deployment strategies, Services, Rolling Updates, Versioning

---

## 🧰 App Setup (Docker Image)

If you want to build locally, here's a minimal Node.js app image. Otherwise, replace `myapp:v1` with your own Docker Hub image.

### File: `app.js`

```javascript
const express = require('express');
const app = express();
const port = 8080;

app.get('/api/hello', (req, res) => {
  res.json({ message: "Hello from MyApp v1!" });
});

app.listen(port, () => {
  console.log(`MyApp v1 running on port ${port}`);
});
```

### File: `Dockerfile`

```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY app.js .
RUN npm install express
EXPOSE 8080
CMD ["node", "app.js"]
```

### Build & Push to Docker Hub

```bash
docker build -t <your-dockerhub-user>/myapp:v1 .
docker push <your-dockerhub-user>/myapp:v1
```

### Create v2 Image

Update the message in `app.js`:

```javascript
res.json({ message: "Hello from MyApp v2!" });
```

Then build and push v2:

```bash
docker build -t <your-dockerhub-user>/myapp:v2 .
docker push <your-dockerhub-user>/myapp:v2
```

---

## 🧩 Kubernetes Deployment YAMLs

### 📁 File: `deployment-myapp.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-deployment
  labels:
    app: myapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: <your-dockerhub-user>/myapp:v1
        ports:
        - containerPort: 8080
```

### 📁 File: `svc-clusterip.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-clusterip
spec:
  selector:
    app: myapp
  ports:
    - port: 80
      targetPort: 8080
  type: ClusterIP
```

### 📁 File: `svc-nodeport.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-nodeport
spec:
  selector:
    app: myapp
  type: NodePort
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 30081
```

### 📁 File: `svc-loadbalancer.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-lb
spec:
  selector:
    app: myapp
  type: LoadBalancer
  ports:
    - port: 80
      targetPort: 8080
```

---

## ⚙️ Step-by-Step Commands

### 1️⃣ Deploy the App

```bash
kubectl apply -f deployment-myapp.yaml
kubectl get deployments
kubectl get pods -l app=myapp -o wide
```

### 2️⃣ Expose Internally (ClusterIP)

```bash
kubectl apply -f svc-clusterip.yaml
kubectl get svc myapp-clusterip
kubectl port-forward svc/myapp-clusterip 8080:80
curl http://127.0.0.1:8080/api/hello
```

**✅ Expected Output:**

```json
{"message": "Hello from MyApp v1!"}
```

### 3️⃣ Expose Externally (NodePort)

```bash
kubectl apply -f svc-nodeport.yaml
kubectl get svc myapp-nodeport
```

**Access the service:**

```bash
# For minikube
minikube service myapp-nodeport --url

# Or use Node IP directly
curl http://<NodeIP>:30081/api/hello
```

### 4️⃣ (Optional) Expose via LoadBalancer (Cloud clusters)

```bash
kubectl apply -f svc-loadbalancer.yaml
kubectl get svc myapp-lb
```

Wait for `EXTERNAL-IP` and test:

```bash
curl http://<external-ip>/api/hello
```

### 5️⃣ Perform Rolling Update to v2

Assuming you built/pushed `myapp:v2`:

```bash
kubectl set image deployment/myapp-deployment myapp=<your-dockerhub-user>/myapp:v2 --record
kubectl rollout status deployment/myapp-deployment
```

**Check rollout history:**

```bash
kubectl rollout history deployment/myapp-deployment
```

**Validate the update:**

```bash
curl http://<NodeIP>:30081/api/hello
```

**✅ Expected Output:**

```json
{"message": "Hello from MyApp v2!"}
```

### 6️⃣ Rollback if Needed

```bash
kubectl rollout undo deployment/myapp-deployment
kubectl rollout status deployment/myapp-deployment
```

### 7️⃣ Clean Up

```bash
kubectl delete svc myapp-clusterip myapp-nodeport myapp-lb
kubectl delete deployment myapp-deployment
```

---

## 🧠 Concept Notes

| Concept | Key Idea |
|---------|----------|
| **Deployment** | Manages versions & replicas of your app |
| **Service Types** | - **ClusterIP**: Internal only<br>- **NodePort**: External via node IP<br>- **LoadBalancer**: Cloud-based external access |
| **Rolling Update** | Smooth version upgrade without downtime |
| **Rollback** | Revert to previous image automatically |
| **Versioning** | Always tag your images (v1, v2, etc.), never use `latest` |

---

## 📋 Service Type Comparison

### ClusterIP (Default)
- **Use Case:** Internal communication between services
- **Access:** Only from within the cluster
- **Example:** Microservices talking to each other

### NodePort
- **Use Case:** Development/testing, exposing services externally
- **Access:** Via `<NodeIP>:<NodePort>`
- **Port Range:** 30000-32767
- **Example:** Quick external access without cloud LB

### LoadBalancer
- **Use Case:** Production external access
- **Access:** Via cloud provider's load balancer
- **Requirements:** Cloud provider support (AWS, GCP, Azure)
- **Example:** Production web applications

---

## 🔍 Troubleshooting Tips

### Check Pod Status
```bash
kubectl get pods -l app=myapp
kubectl describe pod <pod-name>
kubectl logs <pod-name>
```

### Check Service Endpoints
```bash
kubectl get endpoints myapp-clusterip
kubectl describe svc myapp-nodeport
```

### Test Service Connectivity
```bash
# From within cluster
kubectl run curl-test --image=curlimages/curl -i --tty --rm -- sh
curl http://myapp-clusterip/api/hello

# Port forward for local testing
kubectl port-forward deployment/myapp-deployment 8080:8080
curl http://localhost:8080/api/hello
```

### Check Rollout Status
```bash
kubectl rollout status deployment/myapp-deployment
kubectl rollout history deployment/myapp-deployment
kubectl describe deployment myapp-deployment
```

---

## 🎓 Learning Outcomes

After completing this lab, you will understand:
- How to containerize a Node.js application
- How to deploy applications using Kubernetes Deployments
- The differences between ClusterIP, NodePort, and LoadBalancer services
- How to perform zero-downtime rolling updates
- How to rollback deployments when issues occur
- Best practices for image versioning and tagging

---

## 📚 Additional Resources

- [Kubernetes Services Documentation](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Kubernetes Deployments Documentation](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Rolling Updates Best Practices](https://kubernetes.io/docs/tutorials/kubernetes-basics/update/update-intro/)

---

