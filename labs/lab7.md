# Kubernetes Lab #7 — Persistent Volumes & Claims (Storage Management)

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
