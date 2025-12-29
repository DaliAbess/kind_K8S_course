# Kubernetes Hands-on Practice Tasks (Lab #3)

## 🧱 ConfigMaps & Secrets

### 🎯 Goal

1. Use **ConfigMap** to inject non-sensitive configuration (like DB host, environment)
2. Use **Secret** to inject sensitive data (like DB password)
3. Access them as environment variables inside the Pod
4. Verify inside the container
5. Observe how changes propagate

### 🧩 Skills Covered
ConfigMaps, Secrets, Environment Injection, Security Best Practices

---

## 🧰 Scenario

We'll deploy a simple Node.js app (`config-demo`) that reads configuration values from environment variables and prints them via `/config` API.

---

## 🧩 Step 1 — Create ConfigMap

### 📁 File: `configmap.yaml`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_ENV: "production"
  DB_HOST: "db-service"
  DB_PORT: "5432"
  APP_VERSION: "v1"
```

**Apply the ConfigMap:**

```bash
kubectl apply -f configmap.yaml
kubectl get configmaps
kubectl describe configmap app-config
```

---

## 🧩 Step 2 — Create Secret

### 📁 File: `secret.yaml`

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
type: Opaque
data:
  DB_USER: YWRtaW4=          # base64 of 'admin'
  DB_PASSWORD: cGFzc3dvcmQxMjM=  # base64 of 'password123'
```

### 💡 Encoding Values Manually

You can encode values using base64:

```bash
echo -n 'admin' | base64
echo -n 'password123' | base64
```

**Apply the Secret:**

```bash
kubectl apply -f secret.yaml
kubectl get secrets
kubectl describe secret app-secret
```

---

## 🧩 Step 3 — Deployment using ConfigMap & Secret

### 📁 File: `deployment-config-demo.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: config-demo
  labels:
    app: config-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: config-demo
  template:
    metadata:
      labels:
        app: config-demo
    spec:
      containers:
      - name: config-demo
        image: <your-dockerhub-user>/config-demo:v1
        ports:
        - containerPort: 8080
        env:
        # from ConfigMap
        - name: APP_ENV
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: APP_ENV
        - name: DB_HOST
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: DB_HOST
        - name: DB_PORT
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: DB_PORT
        - name: APP_VERSION
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: APP_VERSION

        # from Secret
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: app-secret
              key: DB_USER
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secret
              key: DB_PASSWORD
```

**Apply the Deployment:**

```bash
kubectl apply -f deployment-config-demo.yaml
kubectl get pods -l app=config-demo
```

---

## 🧩 Step 4 — Expose via Service

### 📁 File: `svc-config-demo.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: config-demo-svc
spec:
  selector:
    app: config-demo
  ports:
    - port: 80
      targetPort: 8080
  type: ClusterIP
```

**Apply the Service:**

```bash
kubectl apply -f svc-config-demo.yaml
kubectl get svc config-demo-svc
```

---

## 🧩 Step 5 — Verify Environment Variables

### Exec into Pod

```bash
POD=$(kubectl get pod -l app=config-demo -o jsonpath='{.items[0].metadata.name}')
kubectl exec -it $POD -- /bin/sh
```

### Inside the Container

```bash
env | grep APP
env | grep DB
```

**✅ Expected Output:**

```
APP_ENV=production
APP_VERSION=v1
DB_HOST=db-service
DB_PORT=5432
DB_USER=admin
DB_PASSWORD=password123
```

---

## 🧩 Step 6 — Update ConfigMap or Secret and Observe

### Edit ConfigMap

```bash
kubectl edit configmap app-config
# Change APP_VERSION: "v2"
```

### Check Rollout Behavior

```bash
kubectl get pods -l app=config-demo
```

### ⚠️ Important Note

By default, pods **won't automatically restart** when ConfigMaps/Secrets change.

**To reload automatically, you can:**

1. **Restart manually:**
   ```bash
   kubectl rollout restart deployment/config-demo
   ```

2. **Use a reloader sidecar** (like [Stakater Reloader](https://github.com/stakater/Reloader))

---

## 🧩 Step 7 — Clean Up

```bash
kubectl delete deployment config-demo
kubectl delete svc config-demo-svc
kubectl delete configmap app-config
kubectl delete secret app-secret
```

---

## 🧠 Concept Notes

| Concept | Explanation |
|---------|-------------|
| **ConfigMap** | Non-sensitive app configuration (env, file, command args) |
| **Secret** | Encoded sensitive data (passwords, API keys) |
| **Mounting Methods** | As environment variables or as files under `/etc/...` |
| **Security Best Practice** | Use Opaque secrets, enable encryption at rest, integrate with external secret stores (e.g., AWS Secrets Manager) |
| **Auto Reload** | Pods must restart to pick up ConfigMap/Secret changes |

---

## 📚 Alternative Mounting Methods

### Method 1: Environment Variables (Current Method)

```yaml
env:
- name: APP_ENV
  valueFrom:
    configMapKeyRef:
      name: app-config
      key: APP_ENV
```

### Method 2: Volume Mounts (File-based)

```yaml
volumes:
- name: config-volume
  configMap:
    name: app-config
containers:
- name: config-demo
  volumeMounts:
  - name: config-volume
    mountPath: /etc/config
```

Access files at `/etc/config/APP_ENV`, `/etc/config/DB_HOST`, etc.

### Method 3: envFrom (All Keys at Once)

```yaml
envFrom:
- configMapRef:
    name: app-config
- secretRef:
    name: app-secret
```

All keys from ConfigMap and Secret become environment variables automatically.

---

## 🔐 Security Best Practices

### 1. Secret Encryption at Rest
Enable encryption at rest in your cluster:

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: <base64-encoded-secret>
```

### 2. External Secret Management

Consider using external secret stores:
- **AWS Secrets Manager**
- **HashiCorp Vault**
- **Azure Key Vault**
- **GCP Secret Manager**

Tools like [External Secrets Operator](https://external-secrets.io/) can sync external secrets to Kubernetes.

### 3. RBAC for Secrets

Restrict access to secrets using RBAC:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: secret-reader
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list"]
```

### 4. Avoid Logging Secrets

Never log secret values in application logs or debug output.

---

## 🔍 Troubleshooting

### View ConfigMap Content

```bash
kubectl get configmap app-config -o yaml
kubectl describe configmap app-config
```

### View Secret Content (decoded)

```bash
kubectl get secret app-secret -o jsonpath='{.data.DB_USER}' | base64 --decode
kubectl get secret app-secret -o jsonpath='{.data.DB_PASSWORD}' | base64 --decode
```

### Check Environment Variables in Pod

```bash
kubectl exec -it <pod-name> -- env
kubectl exec -it <pod-name> -- printenv | grep DB
```

### Debug ConfigMap/Secret Not Loading

```bash
kubectl describe pod <pod-name>
kubectl logs <pod-name>
```

Look for events related to missing ConfigMaps or Secrets.

---

## 📝 Sample Node.js Application

If you want to build the `config-demo` app yourself:

### File: `app.js`

```javascript
const express = require('express');
const app = express();
const port = 8080;

app.get('/config', (req, res) => {
  res.json({
    app_env: process.env.APP_ENV || 'not set',
    app_version: process.env.APP_VERSION || 'not set',
    db_host: process.env.DB_HOST || 'not set',
    db_port: process.env.DB_PORT || 'not set',
    db_user: process.env.DB_USER || 'not set',
    // Don't expose password in real apps!
    db_password: process.env.DB_PASSWORD ? '***hidden***' : 'not set'
  });
});

app.get('/health', (req, res) => {
  res.json({ status: 'healthy' });
});

app.listen(port, () => {
  console.log(`Config-demo app running on port ${port}`);
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

### Build and Push

```bash
docker build -t <your-dockerhub-user>/config-demo:v1 .
docker push <your-dockerhub-user>/config-demo:v1
```

---

## 🎓 Learning Outcomes

After completing this lab, you will understand:
- How to create and manage ConfigMaps for non-sensitive configuration
- How to create and manage Secrets for sensitive data
- Different methods to inject configuration into containers
- Security best practices for handling secrets
- How configuration changes propagate to running pods
- When and how to trigger pod restarts for configuration updates

---

## 📚 Additional Resources

- [Kubernetes ConfigMaps Documentation](https://kubernetes.io/docs/concepts/configuration/configmap/)
- [Kubernetes Secrets Documentation](https://kubernetes.io/docs/concepts/configuration/secret/)
- [Stakater Reloader](https://github.com/stakater/Reloader) - Auto-reload on ConfigMap/Secret changes
- [External Secrets Operator](https://external-secrets.io/)

---

