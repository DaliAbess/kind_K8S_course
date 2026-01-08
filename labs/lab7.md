# Kubernetes Lab #7 — Persistent Volumes & Claims (Storage Management)

## 📖 What are Persistent Volumes (PV) and Persistent Volume Claims (PVC)?

Kubernetes storage follows a separation of concerns model where administrators provision storage resources and users consume them without needing to know infrastructure details.

### The Problem Without Persistent Storage

```
┌─────────────────────────────────────────────────────────┐
│  Pod Lifecycle WITHOUT Persistent Storage               │
│                                                          │
│  Pod Created → Data Written → Pod Deleted → Data LOST   │
│                                                          │
│  Every restart = Fresh start = No data persistence      │
└─────────────────────────────────────────────────────────┘
```

**Example:** A database Pod crashes and restarts → all data is gone! 💥

### The Solution: Persistent Volumes

```
┌──────────────────────────────────────────────────────────┐
│  Pod Lifecycle WITH Persistent Storage                   │
│                                                           │
│  PV Created → PVC Binds → Pod Uses → Pod Deleted →       │
│  PV Still Exists → New Pod → Mounts Same PV → Data ✓    │
│                                                           │
│  Data survives Pod restarts and deletions                │
└──────────────────────────────────────────────────────────┘
```

### Key Concepts

**PersistentVolume (PV)**
- A piece of storage in the cluster provisioned by an administrator
- Exists independently of any Pod
- Has its own lifecycle separate from Pods
- Think of it as: "Real storage capacity available in the cluster"

**PersistentVolumeClaim (PVC)**
- A request for storage by a user/Pod
- Binds to a matching PV
- Pods use PVCs to access storage
- Think of it as: "A ticket to claim storage"

**The Relationship:**

```
Administrator              User/Developer
     │                          │
     │ 1. Provisions            │ 2. Requests
     ↓                          ↓
┌─────────┐                ┌─────────┐
│   PV    │ ←── Binds ───→ │   PVC   │
│ (Supply)│                │ (Demand)│
└─────────┘                └────┬────┘
                                │ 3. Uses
                                ↓
                           ┌─────────┐
                           │   Pod   │
                           └─────────┘
```

### Complete Storage Architecture

```
┌────────────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                           │
│                                                                 │
│  ┌──────────────────────────────────────────────────────┐     │
│  │              Control Plane                            │     │
│  │  ┌────────────────────────────────────────┐         │     │
│  │  │  PV Controller                          │         │     │
│  │  │  - Watches PV and PVC resources         │         │     │
│  │  │  - Binds PVCs to matching PVs           │         │     │
│  │  │  - Manages volume lifecycle             │         │     │
│  │  └────────────────────────────────────────┘         │     │
│  └──────────────────────────────────────────────────────┘     │
│                                                                 │
│  ┌──────────────────────────────────────────────────────┐     │
│  │              Worker Node                              │     │
│  │                                                       │     │
│  │  ┌────────────────────────────────┐                  │     │
│  │  │  Pod: nginx-pv-demo            │                  │     │
│  │  │  ┌──────────────────────────┐  │                  │     │
│  │  │  │  Container: nginx        │  │                  │     │
│  │  │  │  ┌────────────────────┐  │  │                  │     │
│  │  │  │  │ /usr/share/nginx/  │  │  │                  │     │
│  │  │  │  │       html/        │  │  │                  │     │
│  │  │  │  └─────────┬──────────┘  │  │                  │     │
│  │  │  │            │ mounted      │  │                  │     │
│  │  │  └────────────┼─────────────┘  │                  │     │
│  │  │               │                 │                  │     │
│  │  │               ↓                 │                  │     │
│  │  │  ┌────────────────────────┐    │                  │     │
│  │  │  │  PVC: pvc-demo         │    │                  │     │
│  │  │  │  Status: Bound         │    │                  │     │
│  │  │  └────────────┬───────────┘    │                  │     │
│  │  │               │ claims          │                  │     │
│  │  └───────────────┼─────────────────┘                  │     │
│  │                  │                                     │     │
│  │                  ↓                                     │     │
│  │  ┌────────────────────────────────────────┐           │     │
│  │  │  PV: pv-demo                           │           │     │
│  │  │  Capacity: 1Gi                         │           │     │
│  │  │  Access: ReadWriteOnce                 │           │     │
│  │  │  Reclaim: Retain                       │           │     │
│  │  └──────────────┬─────────────────────────┘           │     │
│  │                 │ backed by                            │     │
│  │                 ↓                                      │     │
│  │  ┌────────────────────────────────────────┐           │     │
│  │  │  Physical Storage: /mnt/data           │           │     │
│  │  │  (hostPath in this example)            │           │     │
│  │  └────────────────────────────────────────┘           │     │
│  └──────────────────────────────────────────────────────┘     │
└────────────────────────────────────────────────────────────────┘
```

### PV/PVC Binding Process

```
Step 1: Admin Creates PV
┌────────────────────┐
│   PV: pv-demo      │
│   Capacity: 1Gi    │
│   Status: Available│
└────────────────────┘

Step 2: User Creates PVC
┌────────────────────┐
│   PVC: pvc-demo    │
│   Requests: 1Gi    │
│   Status: Pending  │
└────────────────────┘

Step 3: Binding (Automatic)
┌────────────────────┐          ┌────────────────────┐
│   PV: pv-demo      │ ←─Bind─→ │   PVC: pvc-demo    │
│   Capacity: 1Gi    │          │   Requests: 1Gi    │
│   Status: Bound    │          │   Status: Bound    │
└────────────────────┘          └────────────────────┘

Step 4: Pod Uses PVC
                    ┌────────────────────┐
                    │   Pod              │
                    │   Mounts: pvc-demo │
                    └─────────┬──────────┘
                              │
                              ↓ (uses)
                    ┌────────────────────┐
                    │   PVC: pvc-demo    │
                    └────────────────────┘
```

### Access Modes Explained

```
ReadWriteOnce (RWO) - Most Common
┌─────────┐
│ Node A  │ ✓ Can mount (read/write)
└─────────┘
┌─────────┐
│ Node B  │ ✗ Cannot mount (already mounted on Node A)
└─────────┘

ReadOnlyMany (ROX)
┌─────────┐
│ Node A  │ ✓ Can mount (read-only)
└─────────┘
┌─────────┐
│ Node B  │ ✓ Can mount (read-only)
└─────────┘

ReadWriteMany (RWX) - Requires Network Storage
┌─────────┐
│ Node A  │ ✓ Can mount (read/write)
└─────────┘
┌─────────┐
│ Node B  │ ✓ Can mount (read/write)
└─────────┘
```

### Storage Lifecycle

```
┌─────────────┐
│ Provisioning│ ← Admin creates PV or uses StorageClass
└──────┬──────┘
       ↓
┌─────────────┐
│   Binding   │ ← User creates PVC, automatically binds to PV
└──────┬──────┘
       ↓
┌─────────────┐
│    Using    │ ← Pod mounts PVC and reads/writes data
└──────┬──────┘
       ↓
┌─────────────┐
│  Releasing  │ ← PVC is deleted, PV becomes "Released"
└──────┬──────┘
       ↓
┌─────────────┐
│  Reclaiming │ ← Based on reclaimPolicy:
└─────────────┘   - Retain: Manual cleanup
                  - Delete: Auto-delete PV and storage
                  - Recycle: Wipe and reuse (deprecated)
```

### Reclaim Policies Comparison

| Policy | What Happens | Use Case |
|--------|-------------|----------|
| **Retain** | PV kept, data preserved, manual cleanup needed | Production data, need manual review before deletion |
| **Delete** | PV and underlying storage auto-deleted | Dynamic provisioning, temporary data |
| **Recycle** | Data wiped, PV reused (deprecated) | Legacy systems only |

### Static vs Dynamic Provisioning

**Static Provisioning (This Lab):**
```
1. Admin manually creates PV
2. User creates PVC
3. PVC binds to existing PV
4. Pod uses PVC

Manual, pre-provisioned storage
```

**Dynamic Provisioning (Production):**
```
1. Admin creates StorageClass
2. User creates PVC with storageClassName
3. PV automatically created by StorageClass
4. Pod uses PVC

Automatic, on-demand storage
```

### Common Storage Backends

**Local (Development)**
- `hostPath`: Node's local filesystem
- `emptyDir`: Temporary, deleted with Pod

**Cloud (Production)**
- AWS: EBS (Elastic Block Store)
- GCP: Persistent Disk
- Azure: Disk Storage

**Network Storage**
- NFS (Network File System)
- Ceph / CephFS
- GlusterFS
- iSCSI

### Real-World Example: Database

```
Without PV/PVC:                  With PV/PVC:
┌──────────┐                    ┌──────────┐
│ Postgres │                    │ Postgres │
│   Pod    │                    │   Pod    │
└────┬─────┘                    └────┬─────┘
     │                                │
     ↓                                ↓
┌─────────┐                      ┌─────────┐
│ No Data │  ← Deleted           │   PVC   │
└─────────┘                      └────┬────┘
                                      │
Data Lost! 💥                         ↓
                                 ┌─────────┐
                                 │   PV    │
                                 └────┬────┘
                                      │
                                      ↓
                                 ┌─────────┐
                                 │  Disk   │
                                 └─────────┘
                                 
                                 Data Persists! ✓
```

### Why PV/PVC Pattern?

✅ **Separation of Concerns**: Admins manage infrastructure, users consume resources  
✅ **Portability**: Abstract storage details from application  
✅ **Flexibility**: Easy to change storage backend without changing Pod specs  
✅ **Reusability**: Multiple Pods can use the same PVC (depending on access mode)  
✅ **Data Protection**: Explicit lifecycle management prevents accidental data loss

---

## 🎯 Goal

- Understand how PersistentVolume (PV) and PersistentVolumeClaim (PVC) work
- Deploy an app (e.g., Nginx) that stores data persistently
- Verify data survives Pod restarts

## 🧩 Skills

- Persistent storage
- Pod data retention
- Volume mounting
- Storage troubleshooting

## 🧰 Scenario

You'll:

1. Create a PersistentVolume (PV) backed by host storage (in Minikube/local clusters)
2. Bind it to a PersistentVolumeClaim (PVC)
3. Mount it inside an Nginx Pod
4. Write data → delete Pod → verify data persists

## Step 1 — PersistentVolume (PV)

### File: `pv.yaml`

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-demo
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  hostPath:
    path: /mnt/data
```

### 🧠 Explanation

- **hostPath** → uses node's local storage (for learning/demo)
- **Retain** → PV will keep data even if PVC is deleted
- **manual** → no automatic provisioning

### Apply

```bash
kubectl apply -f pv.yaml
kubectl get pv
```

You'll see:

```
NAME      CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   AGE
pv-demo   1Gi        RWO            Retain           Available           manual         5s
```

## Step 2 — PersistentVolumeClaim (PVC)

### File: `pvc.yaml`

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-demo
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: manual
```

### Apply

```bash
kubectl apply -f pvc.yaml
kubectl get pvc
```

Now the PVC should bind to the PV automatically:

```
NAME       STATUS   VOLUME    CAPACITY   ACCESS MODES   STORAGECLASS   AGE
pvc-demo   Bound    pv-demo   1Gi        RWO            manual         5s
```

## Step 3 — Pod Using the PVC

### File: `pod-pv-demo.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pv-demo
spec:
  containers:
  - name: nginx
    image: nginx:alpine
    ports:
    - containerPort: 80
    volumeMounts:
    - mountPath: /usr/share/nginx/html
      name: web-content
  volumes:
  - name: web-content
    persistentVolumeClaim:
      claimName: pvc-demo
```

### Apply

```bash
kubectl apply -f pod-pv-demo.yaml
kubectl get pods
```

## Step 4 — Write Data to Persistent Volume

Let's write some custom content inside Nginx's web directory:

```bash
kubectl exec -it nginx-pv-demo -- /bin/sh
echo "Hello from Persistent Volume!" > /usr/share/nginx/html/index.html
exit
```

### Test the content

```bash
kubectl port-forward pod/nginx-pv-demo 8080:80
curl http://localhost:8080
```

Output:

```
Hello from Persistent Volume!
```

## Step 5 — Verify Data Persistence

Now simulate a Pod deletion:

```bash
kubectl delete pod nginx-pv-demo
```

Then re-create it:

```bash
kubectl apply -f pod-pv-demo.yaml
kubectl exec -it nginx-pv-demo -- cat /usr/share/nginx/html/index.html
```

✅ **The data should still be there!**

If you check on your Minikube host:

```bash
minikube ssh
cat /mnt/data/index.html
```

You'll see the same file.

## Step 6 — Cleanup

```bash
kubectl delete pod nginx-pv-demo
kubectl delete pvc pvc-demo
kubectl delete pv pv-demo
```

## 💡 Tips

- For production, use `StorageClass` (e.g., EBS, GCE Persistent Disk, NFS, Ceph)
- **ReadWriteOnce** → single node access
- **ReadWriteMany** → shared access (requires network storage like NFS)
- Consider using dynamic provisioning with StorageClasses in production

## ✅ What You've Mastered

- PV/PVC fundamentals
- Persistent data across Pod lifecycles
- Reclaim and reuse storage resources
- Volume mounting and data verification
- Understanding storage binding and lifecycle

## 📚 Key Concepts

**PersistentVolume (PV)**: Cluster-level storage resource provisioned by an administrator

**PersistentVolumeClaim (PVC)**: Request for storage by a user/pod

**Access Modes**:
- `ReadWriteOnce (RWO)`: Volume can be mounted as read-write by a single node
- `ReadOnlyMany (ROX)`: Volume can be mounted read-only by many nodes
- `ReadWriteMany (RWX)`: Volume can be mounted as read-write by many nodes

**Reclaim Policies**:
- `Retain`: Manual reclamation of the resource
- `Delete`: Associated storage asset is deleted
- `Recycle`: Basic scrub (deprecated)
