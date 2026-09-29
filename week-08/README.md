# Lab 08 — Kubernetes Networking and Storage

## Objective
Learn Kubernetes networking concepts — ClusterIP, NodePort, and service
discovery — and compare temporary (emptyDir) versus persistent (PV/PVC)
storage using a single-node cluster on Docker Desktop for Windows.

---

## Environment

| Item | Detail |
|------|--------|
| Host OS | Windows 11 Home |
| Docker on Windows | Docker Desktop |
| Kubernetes | Built-in Docker Desktop single-node cluster |
| Node Name | `docker-desktop` |
| Shell | PowerShell |

---

## Part 1 — Enable Kubernetes and Verify

### Step 1: Enable Kubernetes in Docker Desktop

Opened Docker Desktop → Settings → Kubernetes → checked "Enable Kubernetes"
→ clicked Apply & Restart. Waited for Kubernetes to show as Running.

### Step 2: Verify kubectl and Cluster

```powershell
kubectl version --client
kubectl get nodes
```

```
NAME             STATUS   ROLES           AGE   VERSION
docker-desktop   Ready    control-plane   Xm    v1.x.x
```

### Step 3: Verify Network Pods

```powershell
kubectl get pods -n kube-system
```

All core networking pods were running. Docker Desktop's built-in networking
layer was confirmed functional — no need to install Weave Net.

---

## Part 2 — Deploy NGINX with ClusterIP Service

### Step 4: Create Working Directory

```powershell
mkdir "C:\Users\xyz\YAML files"
cd "C:\Users\xyz\YAML files"
```

### Step 5: Create Deployment YAML

```powershell
notepad deploy-nginx.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
```

### Step 6: Create ClusterIP Service YAML

```powershell
notepad service-nginx.yaml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: service-nginx
spec:
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
  type: ClusterIP
```

### Step 7: Apply Both Manifests

```powershell
kubectl apply -f deploy-nginx.yaml
kubectl apply -f service-nginx.yaml
```

Confirmed 2/2 replicas ready and both pods in Running status:

```powershell
kubectl get deployments
kubectl get pods
```

---

## Part 3 — Test ClusterIP (Internal Access)

### Step 8: Get Cluster IP

```powershell
kubectl get svc service-nginx
```

Noted the assigned Cluster IP (e.g. `10.96.168.133`).

### Step 9: Run a curl Test Pod

```powershell
kubectl run curlpod --image=radial/busyboxplus:curl -it --rm --restart=Never -- sh
```

Inside the pod:

```sh
curl http://10.96.168.133:80
```

NGINX welcome page returned successfully, confirming DNS-based service
discovery and internal load balancing across both pod replicas were working.

> **Note:** ClusterIP is only accessible from inside the cluster. The curl pod
> simulates internal service-to-service communication.

---

## Part 4 — Replace ClusterIP with NodePort

### Step 10: Delete the ClusterIP Service

```powershell
kubectl delete svc service-nginx
```

### Step 11: Create NodePort Service YAML

```powershell
notepad service-nginx-nodeport.yaml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: service-nginx
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080
```

### Step 12: Apply and Test

```powershell
kubectl apply -f service-nginx-nodeport.yaml
```

Attempted to access `http://localhost:30080` in browser. Direct access was
blocked because Docker Desktop Kubernetes runs behind a Hyper-V VM on Windows,
isolating it from the host network. Used port forwarding as a workaround:

```powershell
kubectl port-forward service/service-nginx 8080:80
```

Accessed at `http://localhost:8080` — NGINX welcome page confirmed.

---

## Part 5 — Temporary Storage with emptyDir

### Step 13: Create Redis Pod with emptyDir

```powershell
notepad redis-temp.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: redis
spec:
  containers:
  - name: redis
    image: redis
    volumeMounts:
    - mountPath: /data
      name: redis-storage
  volumes:
  - name: redis-storage
    emptyDir: {}
```

```powershell
kubectl apply -f redis-temp.yaml
```

### Step 14: Write Data and Delete Pod

```powershell
kubectl exec -it redis -- /bin/bash
```

```bash
echo "redis temporary data" > /data/temp.txt
exit
```

```powershell
kubectl delete pod redis
kubectl apply -f redis-temp.yaml
kubectl exec -it redis -- cat /data/temp.txt
```

File was not found — confirmed that `emptyDir` volumes are destroyed with the
pod. Appropriate only for temporary caches or working data that can be
regenerated.

---

## Part 6 — Persistent Storage with PV and PVC

### Step 15: Create Redis Pod with Persistent Volume

```powershell
notepad redis-perm.yaml
```

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: redis-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/tmp/redis-data"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: redis-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
---
apiVersion: v1
kind: Pod
metadata:
  name: redis
spec:
  containers:
  - name: redis
    image: redis
    volumeMounts:
    - mountPath: /data
      name: redis-storage
  volumes:
  - name: redis-storage
    persistentVolumeClaim:
      claimName: redis-pvc
```

```powershell
kubectl apply -f redis-perm.yaml
```

### Step 16: Write Data, Delete, and Verify Persistence

```powershell
kubectl exec -it redis -- /bin/bash
```

```bash
echo "redis persistent data" > /data/perm.txt
exit
```

```powershell
kubectl delete pod redis
kubectl apply -f redis-perm.yaml
kubectl exec -it redis -- cat /data/perm.txt
```

```
redis persistent data
```

File survived pod deletion and recreation — confirmed that PV/PVC storage
persists independently of pod lifecycle.

---

## Storage Comparison

| Feature | emptyDir | PersistentVolume |
|---------|----------|-----------------|
| Survives pod restart | No | Yes |
| Survives pod deletion | No | Yes |
| Managed by | Kubernetes (per pod) | Cluster admin |
| Use case | Temp cache, scratch space | Databases, stateful apps |

---

## Troubleshooting

| Issue | Cause | Fix |
|-------|-------|-----|
| `kubectl get nodes` refused connection on port 8080 | Kubernetes not enabled in Docker Desktop | Enabled Kubernetes under Docker Desktop Settings → Apply & Restart |
| TLS handshake timeout pulling `kindest/node` image | Leftover stopped Minikube container from Lab 07 consuming resources | Removed stale container with `docker rm`, restarted Docker Desktop |
| `localhost:30080` not accessible in browser | Docker Desktop Kubernetes runs behind Hyper-V VM on Windows | Used `kubectl port-forward` to access service at `localhost:8080` |

---

## What I Learned

- ClusterIP is the default service type and is only reachable from inside the
  cluster it is appropriate for internal service-to-service communication
- NodePort exposes a service on the host but on Windows Docker Desktop, Hyper-V
  isolation means direct host access requires port forwarding
- Kubernetes DNS resolves service names automatically within the cluster 
  pods reach other services by name, not by IP
- `emptyDir` is tied to pod lifetime and is wiped on deletion  suitable only
  for temporary or regenerable data
- PersistentVolumes are cluster-level resources decoupled from pods  a PVC
  claims storage from a PV, and that storage survives pod deletion and
  recreation, making it the correct choice for stateful workloads like databases
- A leftover container from a previous lab can interfere with a new cluster's
  network initialization  cleaning up between labs matters