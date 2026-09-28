# Lab 03 — Docker Private Registry with Authentication & Storage

## Objective
Configure a private Docker registry with basic HTTP authentication on both Windows and Linux, and explore Docker's two persistent storage mechanisms: volumes and bind mounts.

---

## Environment

| Item | Detail |
|------|--------|
| Host OS | Windows 11 Home |
| Docker on Windows | Docker Desktop |
| VM | VirtualBox 7 → Ubuntu 20.04 LTS |
| Docker in VM | Docker Engine via `apt` |
| Shell | PowerShell (Windows) / bash (Linux) |

---

## Part 1 — Private Registry with Basic Authentication (Windows)

### Step 1: Start a Basic Registry

```powershell
docker run -d -p 5000:5000 --restart always --name privregistry registry:2
```

### Step 2: Pull, Tag, and Push NGINX

```powershell
docker pull nginx
docker tag nginx localhost:5000/privnginx
docker push localhost:5000/privnginx
```

### Step 3: Create the Auth Directory

```powershell
mkdir C:\registry
mkdir C:\registry\auth
```

### Step 4: Generate the Password File

Windows has no native `htpasswd` utility, so the `httpd:2` Docker image was used instead:

```powershell
docker run --rm --entrypoint htpasswd httpd:2 -Bbn learningdocker yourpassword | Out-File -Encoding ASCII -NoNewline C:\registry\auth\registry.password
```

Verified the file was created correctly:

```powershell
type C:\registry\auth\registry.password
```

Expected output: learningdocker:$2y$05$xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx


### Step 5: Relaunch Registry with Authentication

```powershell
docker stop privregistry
docker rm privregistry

docker run -d -p 5000:5000 --restart always `
  --env REGISTRY_AUTH=htpasswd `
  --env REGISTRY_AUTH_HTPASSWD_REALM="Registry Realm" `
  --env REGISTRY_AUTH_HTPASSWD_PATH=/auth/registry.password `
  -v C:/registry/auth:/auth `
  --name privregistry registry:2
```

### Step 6: Confirm Authentication is Working

Attempted a push without logging in — received an authorization error, confirming auth was active. Logged in and pushed successfully:

```powershell
docker login localhost:5000
docker push localhost:5000/privnginx
```

---

## Part 2 — Private Registry with Basic Authentication (Linux)

### Step 1: Install htpasswd and Create Auth Directory

```bash
sudo apt install apache2-utils
mkdir -p registry/auth && cd registry/auth
```

### Step 2: Generate the Password File

```bash
htpasswd -Bc registry.password learningdocker
```

### Step 3: Relaunch Registry with Authentication

```bash
docker stop privregistry && docker rm privregistry

docker run -d -p 5000:5000 --restart always \
  --env REGISTRY_AUTH=htpasswd \
  --env REGISTRY_AUTH_HTPASSWD_REALM="Registry Realm" \
  --env REGISTRY_AUTH_HTPASSWD_PATH=/auth/registry.password \
  -v ./auth:/auth \
  --name privregistry registry:2
```

### Step 4: Configure Insecure Registry Access

```bash
sudo nano /etc/docker/daemon.json
```

Added:

```json
{
  "insecure-registries": ["172.17.0.2:5000"]
}
```

Restarted Docker:

```bash
sudo systemctl restart docker
```

### Step 5: Confirm Authentication is Working

Unauthenticated push returned an authorization error. Logged in and confirmed the push succeeded:

```bash
docker login 172.17.0.2:5000
docker push 172.17.0.2:5000/privnginx
```

---

## Part 3 — Docker Volumes (Lab 5B)

Docker volumes are managed by Docker and persist independently of container lifecycles.

### Step 1: Create and Inspect a Volume

```bash
docker volume create dockervol
docker volume ls
```

### Step 2: Mount Volume in a Container and Write Data

```bash
docker run -it -v dockervol:/data ubuntu:22.04
```

Inside the container:

```bash
echo "Docker is great" > /data/index.html
cat /data/index.html
exit
```

### Step 3: Remove the Container

```bash
docker ps -a
docker stop <container_id>
docker rm <container_id>
```

### Step 4: Verify Data Persists in a New Container

Launched a new Alpine container mounting the same volume:

```bash
docker run -it -v dockervol:/data alpine:latest
```

Inside the container:

```bash
cat /data/index.html   # Output: Docker is great
exit
```

Data survived container deletion, confirming volumes are independent of individual containers.

### Step 5: Inspect and Clean Up

```bash
docker volume inspect dockervol
docker volume prune
```

---

## Part 4 — Docker Bind Mounts (Lab 5B)

Bind mounts link a host directory directly into a container with two-way sync.

### Step 1: Create a Host Directory and File

**Windows:**

```powershell
mkdir C:\temp\dockerfun
echo "Docker Rocks" > C:\temp\dockerfun\docker.html
```

**Linux:**

```bash
mkdir ~/dockerfun
echo "Docker Rocks" > ~/dockerfun/docker.html
```

### Step 2: Run a Container with a Bind Mount

```powershell
docker run -it -d `
  --mount type=bind,source=C:\temp\dockerfun,target=/dockerissurefun `
  --name=Alpineimg alpine
```

### Step 3: Verify Two-Way Sync

Accessed the container and created a new file:

```bash
docker exec -it Alpineimg sh
echo "Bind mounts are awesome" > /dockerissurefun/newfile.txt
exit
```

Verified the file appeared immediately on the Windows host:

```powershell
dir C:\temp\dockerfun
type C:\temp\dockerfun\newfile.txt
```

Modified the file on the host and confirmed changes were instantly visible inside the container:

```powershell
echo "Modified from Windows" > C:\temp\dockerfun\docker.html
docker exec -it Alpineimg cat /dockerissurefun/docker.html
# Output: Modified from Windows
```

### Step 4: Clean Up

```powershell
docker stop Alpineimg
docker rm Alpineimg
Remove-Item -Recurse -Force C:\temp\dockerfun
```

---

## Troubleshooting

| Issue | Cause | Fix |
|-------|-------|-----|
| Windows has no `htpasswd` utility | Not natively available on Windows | Used `httpd:2` Docker image to generate the password file |
| Push rejected without login | Authentication confirmed working as expected | Logged in with `docker login` before pushing |
| VirtualBox freezes during lab | Resource strain on Windows 11 Home | Reduced VM RAM, closed background apps |

---

## What I Learned

- Basic authentication adds a credential layer to a private registry but is not production-ready without SSL, covered in Lab 6A
- On Linux, `htpasswd` is available via `apache2-utils`; on Windows, the `httpd:2` Docker image is required as a workaround
- Docker volumes are managed by Docker and persist across container lifecycles — the preferred method for production storage
- Bind mounts create a direct, real-time link between a host directory and a container — ideal for development workflows
- Volumes and bind mounts serve different purposes: volumes for durable persistence, bind mounts for live host-container file sharing