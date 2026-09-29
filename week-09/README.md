# Lab 09 — WordPress on Kubernetes

## Objective
Deploy a production-like multi-tier WordPress application on Kubernetes using
Minikube, combining secrets, persistent storage, ClusterIP and LoadBalancer
services, and multi-container coordination.

---

## Environment

| Item | Detail |
|------|--------|
| Host OS | Windows 11 Home |
| Docker on Windows | Docker Desktop |
| Kubernetes Tool | Minikube + kubectl |
| Shell | PowerShell |
| Working Directory | `C:\wordpress-k8s` |

---

## Architecture

```
Browser → LoadBalancer Service → WordPress Pod → MySQL Service → MySQL Pod
                                                                      ↓
                                                          Persistent Volume (5GB)
```

---

## Part 1 — Create Kubernetes Secret

### Step 1: Create MySQL Password Secret

```powershell
kubectl create secret generic mysql-dev-pwd --from-literal=password=MySecurePassword123
```

### Step 2: Verify Secret

```powershell
kubectl get secrets
kubectl describe secret mysql-dev-pwd
```

```
Name:     mysql-dev-pwd
Type:     Opaque
Data
====
password: 20 bytes
```

The password value is never displayed — only its byte size. Secrets are
encrypted at rest and only exposed to pods that reference them.

---

## Part 2 — Create StorageClass

### Step 3: Create Working Directory

```powershell
cd C:\
mkdir wordpress-k8s
cd wordpress-k8s
```

### Step 4: Create and Apply StorageClass

```powershell
notepad storageClass.yaml
```

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
```

```powershell
kubectl apply -f storageClass.yaml
kubectl get storageclass
```

```
NAME            PROVISIONER                    VOLUMEBINDINGMODE      AGE
local-storage   kubernetes.io/no-provisioner   WaitForFirstConsumer   10s
standard        k8s.io/minikube-hostpath       Immediate              5d
```

> **Note:** `WaitForFirstConsumer` means the PVC stays Pending until a pod
> actually requests it. This is expected behavior.

---

## Part 3 — Create Persistent Volume and Claim

### Step 5: Create Persistent Volume

```powershell
notepad pv.yaml
```

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv
spec:
  storageClassName: local-storage
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data"
```

```powershell
kubectl apply -f pv.yaml
```

### Step 6: Create Persistent Volume Claim

```powershell
notepad pvc.yaml
```

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
spec:
  storageClassName: local-storage
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
```

```powershell
kubectl apply -f pvc.yaml
kubectl get pvc
```

```
NAME        STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS    AGE
mysql-pvc   Pending   -        -          -              local-storage   10s
```

Status shows `Pending` — expected. Will change to `Bound` once the MySQL pod
requests it.

---

## Part 4 — Deploy MySQL

### Step 7: Create MySQL Deployment and Service YAML

```powershell
notepad mysql-deploy.yaml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql-service
spec:
  selector:
    app: mysql
  ports:
    - protocol: TCP
      port: 3306
      targetPort: 3306
  clusterIP: None
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql-deployment
spec:
  selector:
    matchLabels:
      app: mysql
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:5.7
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-dev-pwd
              key: password
        - name: MYSQL_DATABASE
          value: wordpress
        ports:
        - containerPort: 3306
          name: mysql
        volumeMounts:
        - name: mysql-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-storage
        persistentVolumeClaim:
          claimName: mysql-pvc
```

```powershell
kubectl apply -f mysql-deploy.yaml
```

### Step 8: Verify MySQL Pod and Storage

```powershell
kubectl get pods
kubectl logs <mysql-pod-name>
kubectl get pvc
```

```
NAME        STATUS   VOLUME     CAPACITY   ACCESS MODES   STORAGECLASS    AGE
mysql-pvc   Bound    mysql-pv   5Gi        RWO            local-storage   5m
```

PVC status changed from `Pending` to `Bound` once the MySQL pod started.
MySQL logs confirmed: `mysqld: ready for connections.`

---

## Part 5 — Deploy WordPress

### Step 9: Create WordPress Deployment and Service YAML

```powershell
notepad wordpress.yaml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: wordpress-service
spec:
  selector:
    app: wordpress
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: LoadBalancer
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress-deployment
spec:
  selector:
    matchLabels:
      app: wordpress
  replicas: 1
  template:
    metadata:
      labels:
        app: wordpress
    spec:
      containers:
      - name: wordpress
        image: wordpress:latest
        env:
        - name: WORDPRESS_DB_HOST
          value: mysql-service:3306
        - name: WORDPRESS_DB_USER
          value: root
        - name: WORDPRESS_DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-dev-pwd
              key: password
        - name: WORDPRESS_DB_NAME
          value: wordpress
        ports:
        - containerPort: 80
          name: wordpress
```

```powershell
kubectl apply -f wordpress.yaml
kubectl get pods
```

WordPress pod stayed in `ContainerCreating` for over 20 minutes while Minikube
pulled the 268 MB image. Both pods eventually reached `Running`.

---

## Part 6 — Access WordPress

### Step 10: Review All Resources

```powershell
kubectl get all
```

```
NAME                                      READY   STATUS    RESTARTS   AGE
pod/mysql-deployment-xxx                  1/1     Running   0          10m
pod/wordpress-deployment-xxx              1/1     Running   0          5m

NAME                       TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
service/mysql-service      ClusterIP      None            <none>        3306/TCP       10m
service/wordpress-service  LoadBalancer   10.96.123.456   <pending>     80:31234/TCP   5m

NAME                                  READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/mysql-deployment      1/1     1            1           10m
deployment.apps/wordpress-deployment  1/1     1            1           5m
```

### Step 11: Open WordPress via Minikube

```powershell
minikube service wordpress-service
```

Browser opened to `http://192.168.49.2:31234` showing the WordPress
installation page.

Completed setup:

- Site Title: My Kubernetes WordPress
- Created admin credentials
- Clicked Install WordPress
- Created a test post: "My First Kubernetes Post"

---

## Part 7 — Test Data Persistence

### Step 12: Delete WordPress Pod

```powershell
kubectl get pods
kubectl delete pod <wordpress-pod-name>
kubectl get pods -w
```

Kubernetes automatically created a replacement pod. Once it reached `Running`:

```powershell
minikube service wordpress-service
```

WordPress site and test post were still present — confirmed that persistent
storage retained all data through the pod deletion and recreation cycle.

---

## Resource Summary

| Resource | Name | Purpose |
|----------|------|---------|
| Secret | `mysql-dev-pwd` | Stores MySQL password securely |
| StorageClass | `local-storage` | Defines manual local storage provisioning |
| PersistentVolume | `mysql-pv` | 5GB storage at `/mnt/data` in Minikube VM |
| PersistentVolumeClaim | `mysql-pvc` | Pod's request for that storage |
| Deployment | `mysql-deployment` | Runs MySQL 5.7 |
| Service | `mysql-service` | Headless ClusterIP — internal DB access |
| Deployment | `wordpress-deployment` | Runs WordPress |
| Service | `wordpress-service` | LoadBalancer — external browser access |

---

## Troubleshooting

| Issue | Cause | Fix |
|-------|-------|-----|
| PVC stuck in `Pending` | `WaitForFirstConsumer` binding mode — expected | Resolved automatically once MySQL pod was created |
| WordPress pod in `ContainerCreating` for 20+ min | 268 MB image download on first pull | Waited for download to complete |
| `kubectl` stopped responding mid-lab | Cluster became unresponsive under resource load during image pull | Waited, then restarted Minikube |
| `minikube` command not found after restart | PATH not persisted across sessions | Re-downloaded Minikube, placed in correct location, re-added to PATH |

---

## What I Learned

- Kubernetes Secrets decouple sensitive values from YAML manifests  both MySQL
  and WordPress pods reference the same secret without the password ever
  appearing in plain text in any config file
- The PV/PVC pattern separates storage provisioning from storage consumption 
  admins create PVs, applications claim them via PVCs, and the binding happens
  automatically when a pod needs it
- `WaitForFirstConsumer` binding keeps PVCs in `Pending` until a pod actually
  requests storage this is intentional, not an error
- A headless service (`clusterIP: None`) for MySQL means DNS resolves directly
  to the pod IP rather than a virtual cluster IP appropriate for a single
  database instance that doesn't need load balancing
- `LoadBalancer` type on Minikube shows `<pending>` for EXTERNAL-IP because
  there is no cloud load balancer `minikube service` creates a tunnel to
  work around this
- Large image pulls can stall a resource-constrained Minikube cluster  the
  cluster may become temporarily unresponsive while pulling images over 200 MB
- PATH variables set with `$env:Path` in PowerShell are session-scoped and
  don't survive a restart tools like Minikube need to be added to the system
  PATH permanently