# Kubernetes Lab #8 — Configuring StatefulSets (Persistent Apps)

## 📖 What are StatefulSets?

StatefulSets are Kubernetes workload resources designed for applications that require:
- **Stable, unique network identifiers** - Each pod gets a predictable name
- **Stable, persistent storage** - Each pod maintains its own dedicated storage
- **Ordered deployment and scaling** - Pods are created and terminated in a specific sequence
- **Ordered, automated rolling updates** - Updates happen one pod at a time in order

### StatefulSets vs Deployments

```
┌─────────────────────────────────────────────────────────────────┐
│                         DEPLOYMENT                               │
├─────────────────────────────────────────────────────────────────┤
│  Pod Names: Random (web-7d8f9-x4k2p, web-7d8f9-9mn3q)          │
│  Storage: Shared across pods or no storage                      │
│  Startup: All pods start simultaneously                         │
│  Use Case: Stateless applications (web servers, APIs)          │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                       STATEFULSET                                │
├─────────────────────────────────────────────────────────────────┤
│  Pod Names: Predictable (web-0, web-1, web-2)                  │
│  Storage: Dedicated PVC per pod (www-web-0, www-web-1)         │
│  Startup: Sequential (web-0 → web-1 → web-2)                   │
│  Use Case: Stateful applications (databases, message queues)    │
└─────────────────────────────────────────────────────────────────┘
```

### Pod Identity & Stable Networking

```
┌──────────────────────────────────────────────────────────────┐
│                   Headless Service: web                       │
│                   (clusterIP: None)                          │
└────────────┬──────────────┬──────────────┬──────────────────┘
             │              │              │
             ▼              ▼              ▼
    ┌─────────────┐ ┌─────────────┐ ┌─────────────┐
    │   web-0     │ │   web-1     │ │   web-2     │
    │             │ │             │ │             │
    │ DNS: web-0  │ │ DNS: web-1  │ │ DNS: web-2  │
    │ .web.default│ │ .web.default│ │ .web.default│
    │ .svc.cluster│ │ .svc.cluster│ │ .svc.cluster│
    │ .local      │ │ .local      │ │ .local      │
    └──────┬──────┘ └──────┬──────┘ └──────┬──────┘
           │                │                │
           ▼                ▼                ▼
    ┌─────────────┐ ┌─────────────┐ ┌─────────────┐
    │ PVC:        │ │ PVC:        │ │ PVC:        │
    │ www-web-0   │ │ www-web-1   │ │ www-web-2   │
    │ (1Gi)       │ │ (1Gi)       │ │ (1Gi)       │
    └─────────────┘ └─────────────┘ └─────────────┘
```

### Ordered Deployment & Scaling

**Scale Up (replicas: 0 → 3):**
```
Time 0:  [Creating web-0...]
         ↓
Time 1:  [web-0: Running] → [Creating web-1...]
         ↓
Time 2:  [web-0: Running] [web-1: Running] → [Creating web-2...]
         ↓
Time 3:  [web-0: Running] [web-1: Running] [web-2: Running] ✓
```

**Scale Down (replicas: 3 → 1):**
```
Time 0:  [web-0: Running] [web-1: Running] [web-2: Running]
         ↓
Time 1:  [web-0: Running] [web-1: Running] [Terminating web-2...]
         ↓
Time 2:  [web-0: Running] [Terminating web-1...]
         ↓
Time 3:  [web-0: Running] ✓

Note: PVCs www-web-1 and www-web-2 are retained!
```

### Persistent Storage Behavior

```
┌─────────────────────────────────────────────────────────────┐
│ Scenario: Pod web-1 crashes or is deleted                   │
└─────────────────────────────────────────────────────────────┘

Before:
  web-1 (Running) ──attached to──> www-web-1 PVC
                                   ├─ data.db
                                   └─ logs/

After deletion:
  [web-1 deleted]                  www-web-1 PVC (RETAINED!)
                                   ├─ data.db
                                   └─ logs/

After recreation:
  web-1 (Running) ──reattached──>  www-web-1 PVC
                                   ├─ data.db (PRESERVED!)
                                   └─ logs/ (PRESERVED!)
```

### StatefulSet Architecture

```
┌────────────────────────────────────────────────────────────────┐
│                      StatefulSet Controller                     │
│  - Manages pod lifecycle                                       │
│  - Ensures ordering guarantees                                 │
│  - Maintains stable identities                                 │
└────────────┬───────────────────────────────────────────────────┘
             │
             │ Creates & Manages
             ▼
┌────────────────────────────────────────────────────────────────┐
│  Pods with Stable Identity                                     │
│                                                                 │
│  web-0                web-1                web-2               │
│  ├─ hostname: web-0   ├─ hostname: web-1   ├─ hostname: web-2 │
│  ├─ DNS name          ├─ DNS name          ├─ DNS name        │
│  ├─ ordinal: 0        ├─ ordinal: 1        ├─ ordinal: 2      │
│  └─ PVC: www-web-0    └─ PVC: www-web-1    └─ PVC: www-web-2  │
└────────────────────────────────────────────────────────────────┘
```

### Common Use Cases

| Application Type | Why StatefulSet? |
|-----------------|------------------|
| **MySQL/PostgreSQL** | Requires stable network identity for replication, persistent data storage |
| **MongoDB** | Replica sets need stable hostnames, persistent storage for data |
| **Redis Cluster** | Nodes need stable identities for cluster formation |
| **Kafka** | Brokers need stable IDs and persistent message storage |
| **Elasticsearch** | Nodes need stable identities for cluster membership |
| **ZooKeeper** | Ensemble members need stable network identities |

---

## 🎯 Goal

- Understand StatefulSets and how they differ from Deployments
- Deploy a multi-pod stateful app (Nginx or MySQL)
- Observe stable Pod naming, ordered startup, and persistent storage behavior

## 🧩 Skills

- Stateful applications
- Persistent Volumes
- Ordered rollout
- Pod identity management

## 🧰 Scenario

We'll deploy an Nginx-based StatefulSet to simulate a simple stateful web app.

Each Pod will:

- Have a stable hostname (`web-0`, `web-1`, `web-2`)
- Get its own PersistentVolumeClaim (PVC)
- Serve content stored on its individual volume

## Step 1 — Headless Service

### File: `service-stateful.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  clusterIP: None
  selector:
    app: web
  ports:
  - port: 80
    name: web
```

### 🧠 Explanation

A headless service (`clusterIP: None`) lets StatefulSet pods get DNS entries like:
- `web-0.web.default.svc.cluster.local`
- `web-1.web.default.svc.cluster.local`

### Apply

```bash
kubectl apply -f service-stateful.yaml
```

## Step 2 — StatefulSet Definition

### File: `statefulset-nginx.yaml`

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "web"
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 1Gi
```

### 🧠 Explanation

- Each Pod (`web-0`, `web-1`, `web-2`) gets a unique PVC named: `www-web-0`, `www-web-1`, `www-web-2`
- StatefulSet guarantees ordered creation and stable identities

### Apply

```bash
kubectl apply -f statefulset-nginx.yaml
```

## Step 3 — Verify Pods and Volumes

### Check pods

```bash
kubectl get pods -l app=web
```

Output:

```
NAME    READY   STATUS    RESTARTS   AGE
web-0   1/1     Running   0          1m
web-1   1/1     Running   0          1m
web-2   1/1     Running   0          1m
```

### Check PVCs

```bash
kubectl get pvc
```

Output:

```
NAME         STATUS   VOLUME                                     CAPACITY   ACCESS MODES   AGE
www-web-0    Bound    pvc-abc123                                 1Gi        RWO            1m
www-web-1    Bound    pvc-def456                                 1Gi        RWO            1m
www-web-2    Bound    pvc-ghi789                                 1Gi        RWO            1m
```

## Step 4 — Write Unique Data to Each Pod

Let's simulate data difference across pods:

```bash
for i in 0 1 2; do
  kubectl exec web-$i -- sh -c "echo 'Hello from web-$i' > /usr/share/nginx/html/index.html"
done
```

### Verify each pod's content

```bash
for i in 0 1 2; do
  kubectl exec web-$i -- cat /usr/share/nginx/html/index.html
done
```

Output:

```
Hello from web-0
Hello from web-1
Hello from web-2
```

## Step 5 — Test Stability

Delete one pod and watch what happens:

```bash
kubectl delete pod web-1
kubectl get pods -l app=web
```

You'll see:

```
web-1   0/1   ContainerCreating   0   3s
```

After it restarts:

```bash
kubectl exec web-1 -- cat /usr/share/nginx/html/index.html
```

✅ It still says:

```
Hello from web-1
```

Because the PVC `www-web-1` was retained and reattached.

## Step 6 — Scaling

### Scale up

```bash
kubectl scale statefulset web --replicas=5
```

### Check

```bash
kubectl get pods -l app=web
```

New pods `web-3` and `web-4` will be created in order.

### Scale down

```bash
kubectl scale statefulset web --replicas=2
kubectl get pvc
```

Even after scaling down, PVCs for `web-3` and `web-4` remain —

🧠 **StatefulSets don't delete PVCs automatically (to prevent data loss).**

## Step 7 — Cleanup

```bash
kubectl delete statefulset web
kubectl delete svc web
kubectl delete pvc -l app=web
```

## 🧠 Concept Summary

| Feature       | Deployment     | StatefulSet                          |
|---------------|----------------|--------------------------------------|
| Pod Identity  | Random         | Stable (web-0, web-1, etc.)          |
| Startup Order | Parallel       | Ordered                              |
| Storage       | Shared         | Dedicated per Pod                    |
| Scaling       | Stateless      | Stateful                             |
| Use Case      | Web apps, APIs | Databases, Queues, Stateful services |

## 🔧 Real-world Uses

- MySQL, PostgreSQL, Redis, Kafka
- Elasticsearch, MongoDB
- Apps that need stable hostnames or persistent data

## ✅ What You've Mastered

- StatefulSet structure and behavior
- Pod identity & stable networking
- PVC creation and retention per replica
- Ordered scaling and restart recovery

## 📚 Key Concepts

**Stable Network Identity**: Each pod gets a persistent hostname that stays the same across restarts

**Ordered Deployment**: Pods are created sequentially (web-0 before web-1, etc.)

**Ordered Termination**: Pods are deleted in reverse order during scale-down

**Persistent Storage**: Each pod maintains its own dedicated PVC that survives pod deletion

**StatefulSet Use Cases**:
- Databases requiring stable network identities
- Applications needing ordered, graceful deployment and scaling
- Services requiring persistent storage tied to pod identity
