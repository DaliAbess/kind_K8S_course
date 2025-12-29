# Kubernetes Hands-on Practice Tasks (Lab #1)

## Overview
This lab covers fundamental Kubernetes concepts including Pod management, Deployments, Services, scaling, rolling updates, and rollbacks.

## Lab Objectives

### 1. Basic Pod & Deployment Management

**Skills Covered:** Pods, Deployments, ReplicaSets, Rolling Updates

**Tasks:**
- ✅ Create a Pod running nginx and expose it via a ClusterIP Service
- ✅ Scale it manually to 3 replicas using a Deployment
- ✅ Update the image version and perform a rolling update
- ✅ Roll back to the previous version

---

## Step-by-Step Instructions

### 1) Create a Single Pod (nginx)

Create a file named `pod-nginx.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.23
    ports:
    - containerPort: 80
```

**Apply the configuration:**

```bash
kubectl apply -f pod-nginx.yaml
kubectl get pods -o wide
```

**Verify the Pod:**

```bash
kubectl describe pod/nginx-pod
kubectl logs nginx-pod
```

---

### 2) Expose the Pod with a ClusterIP Service

Create a file named `svc-clusterip.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-clusterip
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80        # service port
      targetPort: 80  # container port
  type: ClusterIP
```

**Apply the service:**

```bash
kubectl apply -f svc-clusterip.yaml
kubectl get svc nginx-clusterip
kubectl describe svc/nginx-clusterip
```

**Test the service (from inside cluster or using port-forward):**

```bash
# Port-forward to test locally
kubectl port-forward svc/nginx-clusterip 8080:80

# In another terminal
curl http://127.0.0.1:8080
```

**Expected Result:** nginx default page HTML

---

### 3) Create a Deployment and Scale to 3 Replicas

Create a file named `deployment-nginx.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.23
        ports:
        - containerPort: 80
```

**Apply the deployment:**

```bash
kubectl apply -f deployment-nginx.yaml
kubectl get deployments
kubectl get pods -l app=nginx -o wide
```

**Scale to 3 replicas:**

```bash
kubectl scale deployment nginx-deployment --replicas=3
kubectl get pods -l app=nginx
```

*Alternatively, edit the YAML to set `replicas: 3` and run `kubectl apply -f deployment-nginx.yaml` again.*

**Verify:**

```bash
kubectl describe deployment/nginx-deployment
kubectl get rs  # shows ReplicaSets
```

**Expected Result:** 3 Pods created, managed by ReplicaSet + Deployment

---

### 4) Expose the Deployment

You can reuse the earlier `nginx-clusterip` service (it selects `app: nginx`), or create a NodePort/LoadBalancer service.

**NodePort example** - create `svc-nodeport.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-nodeport
spec:
  selector:
    app: nginx
  type: NodePort
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080
```

**Apply:**

```bash
kubectl apply -f svc-nodeport.yaml
kubectl get svc nginx-nodeport
```

**Access via NodeIP:30080** (for minikube: `minikube ip`)

---

### 5) Rolling Update - Change Image Version

**Option A:** Update YAML image to `nginx:1.24`, then apply:

```bash
kubectl apply -f deployment-nginx.yaml
```

**Option B (imperative):**

```bash
kubectl set image deployment/nginx-deployment nginx=nginx:1.24 --record
kubectl rollout status deployment/nginx-deployment
kubectl get pods -l app=nginx -o wide
```

**Check rollout history:**

```bash
kubectl rollout history deployment/nginx-deployment
```

**Expected Result:** New pods with nginx:1.24 created and old ones terminated gradually (rolling update)

**To inspect container images:**

```bash
kubectl describe pods -l app=nginx | grep Image -A1
```

---

### 6) Rollback to Previous Version

If the update introduced problems, rollback:

```bash
# Rollback to previous revision
kubectl rollout undo deployment/nginx-deployment

# Or rollback to a specific revision
kubectl rollout history deployment/nginx-deployment
kubectl rollout undo deployment/nginx-deployment --to-revision=1
```

**Verify:**

```bash
kubectl rollout status deployment/nginx-deployment
kubectl get pods -l app=nginx -o wide
```

**Expected Result:** Pods return to previous image

---

### 7) Useful Inspection & Debug Commands

```bash
kubectl get all -l app=nginx
kubectl describe deployment nginx-deployment
kubectl describe rs   # examine ReplicaSets
kubectl logs <pod-name>
kubectl exec -it <pod-name> -- /bin/sh   # inspect container
kubectl port-forward deployment/nginx-deployment 8080:80
kubectl top pods    # requires metrics-server
kubectl delete -f deployment-nginx.yaml
kubectl delete pod nginx-pod  # deletes the standalone pod
```

---

### 8) Cleanup

When done with the lab:

```bash
kubectl delete svc nginx-clusterip nginx-nodeport
kubectl delete deployment nginx-deployment
kubectl delete pod nginx-pod
```

---

## Key Concepts & Tips

### Pod vs Deployment
- **Pod:** Ephemeral and not self-healing
- **Deployment:** Manages ReplicaSets to maintain desired state, allows rolling updates, rollbacks, and autoscaling
- **Best Practice:** Prefer Deployments for applications

### Service Types
- **ClusterIP:** Exposes app inside cluster only
- **NodePort:** Exposes app on each Node's IP at a static port
- **LoadBalancer:** Exposes app externally using cloud provider's load balancer
- Use port-forward/Ingress/NodePort/LoadBalancer for external access

### RollingUpdate Strategy
- Creates new pods then deletes old ones by default
- `maxSurge` and `maxUnavailable` tune speed vs availability
- Ensures zero-downtime deployments

### Rollout Management
- `kubectl set image` with `--record`: Records revision for easier rollback history
- `kubectl rollout undo`: Safe immediate rollback to last known good revision

### Debugging
- `kubectl describe` shows events — your primary debug tool for scheduling and crash loops
- Use `kubectl logs` for application-level debugging
- Use `kubectl exec` to get shell access into containers

---

## Prerequisites

- Kubernetes cluster (minikube, kind, or cloud provider)
- kubectl CLI installed and configured
- Basic understanding of YAML syntax

## Learning Outcomes

After completing this lab, you will understand:
- How to create and manage Pods
- How to expose applications using Services
- How to use Deployments for production workloads
- How to perform rolling updates safely
- How to rollback deployments when issues occur
- Essential kubectl commands for day-to-day operations

---

