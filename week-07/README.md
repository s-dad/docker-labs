# Lab 07 — Kubernetes with Minikube

## Objective
Learn Kubernetes fundamentals using Minikube: deploying applications, exposing
services, scaling pods, and performing rolling updates on a single-node cluster
running on Docker Desktop for Windows.

---

## Environment

| Item | Detail |
|------|--------|
| Host OS | Windows 11 Home |
| Docker on Windows | Docker Desktop |
| Kubernetes Tool | Minikube v1.x + kubectl v1.28.0 |
| Shell | PowerShell |

---

## Key Concepts

| Term | Definition |
|------|-----------|
| Cluster | A group of machines working together |
| Node | A single machine in the cluster |
| Pod | Smallest deployable unit — runs one or more containers |
| Deployment | Manages and maintains a desired number of identical pods |
| Service | Exposes pods to the network |
| ReplicaSet | Ensures a specified number of pod replicas are running |

---

## Part 1 — Install Minikube and kubectl

### Step 1: Create Minikube Directory and Download

```powershell
New-Item -Path 'C:\' -Name 'minikube' -ItemType Directory -Force

Invoke-WebRequest -OutFile 'C:\minikube\minikube.exe' `
  -Uri 'https://github.com/kubernetes/minikube/releases/latest/download/minikube-windows-amd64.exe' `
  -UseBasicParsing
```

### Step 2: Download kubectl

```powershell
curl.exe -LO "https://dl.k8s.io/release/v1.28.0/bin/windows/amd64/kubectl.exe"
Move-Item .\kubectl.exe C:\minikube\kubectl.exe -Force
```

### Step 3: Add to PATH and Verify

```powershell
$env:Path += ";C:\minikube"

minikube version
kubectl version --client
```

---

## Part 2 — Start the Cluster

### Step 4: Start Minikube

```powershell
minikube start --driver=docker
minikube config set driver docker
```

### Step 5: Verify Cluster Status

```powershell
minikube status
```

```
minikube
type: Control Plane
host: Running
kubelet: Running
apiserver: Running
kubeconfig: Configured
```

### Step 6: Check Cluster Info

```powershell
kubectl cluster-info
```

---

## Part 3 — Deploy an Application

### Step 7: Create a Deployment

```powershell
kubectl create deployment my-k8s-nginx --image=nginx
```

### Step 8: Verify Deployment and Pods

```powershell
kubectl get deployments
kubectl get pods
```

```
NAME           READY   UP-TO-DATE   AVAILABLE   AGE
my-k8s-nginx   1/1     1            1           30s
```

```
NAME                             READY   STATUS    RESTARTS   AGE
my-k8s-nginx-7d8b94f94d-abc123   1/1     Running   0          1m
```

---

## Part 4 — Expose the Application

### Step 9: Create a NodePort Service

```powershell
kubectl expose deployment my-k8s-nginx --name=my-nginx-svc --type=NodePort --port=80
```

### Step 10: Check Service and Get Minikube IP

```powershell
kubectl get services
minikube ip
```

```
NAME           TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
kubernetes     ClusterIP   10.96.0.1       <none>        443/TCP        10m
my-nginx-svc   NodePort    10.96.123.456   <none>        80:32000/TCP   1m
```

Navigated to `http://<minikube-ip>:32000` — NGINX welcome page loaded.

### Step 11: Port Forward as Alternative Access

```powershell
kubectl port-forward service/my-nginx-svc 8080:80
```

Accessed at `http://localhost:8080` while the forwarding session was active.

---

## Part 5 — Inspect the Pod

### Step 12: Describe the Pod

```powershell
kubectl describe pod <pod-name>
```

### Step 13: View Pod Logs

```powershell
kubectl logs <pod-name>
```

---

## Part 6 — Scale the Application

### Step 14: Check Current ReplicaSet

```powershell
kubectl get rs
```

```
NAME                      DESIRED   CURRENT   READY   AGE
my-k8s-nginx-7d8b94f94d   1         1         1       10m
```

### Step 15: Scale Up to 3 Replicas

```powershell
kubectl scale deployments/my-k8s-nginx --replicas=3
kubectl get pods
```

```
NAME                             READY   STATUS    RESTARTS   AGE
my-k8s-nginx-7d8b94f94d-abc123   1/1     Running   0          15m
my-k8s-nginx-7d8b94f94d-def456   1/1     Running   0          10s
my-k8s-nginx-7d8b94f94d-ghi789   1/1     Running   0          10s
```

### Step 16: Verify Load Balancing Across Pods

```powershell
kubectl describe services/my-nginx-svc
```

The Endpoints section listed all three pod IP addresses, confirming the service
was distributing traffic across all replicas automatically.

### Step 17: Scale Down to 2 Replicas

```powershell
kubectl scale deployments/my-k8s-nginx --replicas=2
kubectl get pods
```

One pod was automatically terminated — only 2 remained.

---

## Part 7 — Declarative Deployment with YAML

### Step 18: Create Deployment YAML

```powershell
mkdir C:\kubernetes-lab
cd C:\kubernetes-lab
notepad app.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-k8s-learn
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```

### Step 19: Apply the YAML File

```powershell
kubectl apply -f app.yaml
kubectl get deployments
kubectl get pods
```

---

## Part 8 — Rolling Update

### Step 20: Create Update YAML

```powershell
notepad update.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-k8s-learn
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.25.3
        ports:
        - containerPort: 80
```

Only change from `app.yaml`: image version updated from `nginx:1.14.2` to
`nginx:1.25.3`.

### Step 21: Confirm Current Image Before Update

```powershell
kubectl describe pods | findstr "Image:"
```

```
Image: nginx:1.14.2
Image: nginx:1.14.2
```

### Step 22: Apply the Update and Watch

```powershell
kubectl apply -f update.yaml
kubectl get pods -w
```

Kubernetes created new pods with `nginx:1.25.3`, waited for them to become
ready, then terminated the old `nginx:1.14.2` pods — no downtime.

### Step 23: Confirm Update and Rollout Status

```powershell
kubectl describe pods | findstr "Image:"
kubectl rollout status deployment/nginx-k8s-learn
```

```
Image: nginx:1.25.3
Image: nginx:1.25.3
```

```
deployment "nginx-k8s-learn" successfully rolled out
```

---

## Troubleshooting

| Issue | Cause | Fix |
|-------|-------|-----|
| Minikube PATH only active for session | `$env:Path` is session-scoped | Added `C:\minikube` to Windows system PATH permanently via Environment Variables |
| NodePort not accessible in browser | Minikube runs inside Docker on Windows | Used `minikube ip` to get the correct cluster IP instead of `localhost` |
| Port forwarding needed as fallback | NodePort routing limited on Windows Docker driver | Used `kubectl port-forward` to access service at `localhost:8080` |

---

## What I Learned

- Kubernetes adds orchestration on top of containers  deployments manage pod
  lifecycle, scaling, and updates automatically without manual intervention
- Pods are ephemeral deployments are the durable abstraction scaling and
  updates operate at the deployment level, not the pod level
- Services decouple network access from pod identity  pods can be replaced and
  the service continues routing traffic without reconfiguration
- `kubectl apply -f` is the declarative approach and the recommended way to
  manage Kubernetes resources  the YAML file becomes the source of truth for
  the desired state
- Rolling updates replace pods incrementally so the application stays available
  throughout the update no downtime compared to stopping and restarting all
  pods at once
- Minikube on Windows uses the Docker driver, which means the cluster IP is
  the Minikube container's IP, not `localhost` , `minikube ip` gives the
  correct address for NodePort access