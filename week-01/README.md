# Lab 01 — Build Docker Interactive Image & Linux Docker Images

## Objective
Learn how to build custom Docker images by making changes to a running container
and capturing those changes. Set up Linux Docker images and interact with Docker Hub
as a public image repository.

---

## Environment

| Item | Detail |
|------|--------|
| Host OS | Windows 11 Home |
| VM | VirtualBox 7 → Ubuntu 22.04 LTS |
| Docker in VM | Docker Engine via `apt`|
| Shell | bash |

---

## Part 1 — Build Docker Interactive Image

### Step 1: Start an Ubuntu Container

```bash
docker run -it ubuntu bash
```

- `-it` runs in interactive mode with a terminal
- `ubuntu` pulls the base Ubuntu image
- `bash` starts a shell inside the container

---

### Step 2: Update the Package Manager

Inside the container:

```bash
apt-get update
```

Updates the package repository so the latest packages are available.

**Output:**
```
Get:15 http://archive.ubuntu.com/ubuntu noble-updates/main amd64 Packages [1399 kB]
Fetched 30.2 MB in 14s (2176 kB/s)
Reading package lists... Done
```

---

### Step 3: Install wget

```bash
apt-get install -y wget
```

- Installs `wget` for downloading files from the web
- `-y` skips the confirmation prompt

**Output:**
```
The following NEW packages will be installed:
  ca-certificates libpsl5t64 openssl publicsuffix wget
0 upgraded, 5 newly installed, 0 to remove and 0 not upgraded.
Setting up wget (1.21.4-1ubuntu4.1) ...
```

> **Note:** Several `debconf` warnings appeared during install about missing
> dialog frontends. These are safe to ignore — they are cosmetic warnings
> common in minimal Docker base images that lack a full terminal environment.

---

### Step 4: Exit the Container

```bash
exit
```

Exits the bash shell and stops the container but preserves all changes made inside it.

---

### Step 5: Inspect What Changed

```bash
# List all containers including stopped ones
docker ps -a

# Check filesystem changes (replace with your actual container ID)
docker diff <containerID>
```

`docker diff` shows every file added (A), changed (C), or deleted (D) compared
to the base image. After installing `wget` and `ca-certificates`, the output
showed hundreds of new certificate files added under `/etc/ssl/certs/` and
new binaries under `/usr/sbin/`.

---

### Step 6: Commit the Container as a New Image

```bash
docker commit <containerID> ubuntu-wget:v1
```

Saves the container's current state as a reusable image named `ubuntu-wget:v1`.

---

### Step 7: Verify the New Image

```bash
docker images
```

`ubuntu-wget:v1` should now appear in the list alongside the base `ubuntu` image.

---

## Part 2 — Linux Docker Images & Docker Hub

### Step 1: Check Docker Version

```bash
docker --version
```

---

### Step 2: Write a Dockerfile for Apache

Created a `Dockerfile` that installs and runs Apache inside a container:

```dockerfile
FROM ubuntu:20.04

RUN apt-get update && \
    apt-get install -y apache2 && \
    rm -rf /var/lib/apt/lists/*

EXPOSE 80

CMD ["apache2ctl", "-D", "FOREGROUND"]
```

---

### Step 3: Build the Image

```bash
docker image build -t apache3:latest .
```

- `-t apache3:latest` tags the image as `apache3` with the `latest` tag
- `.` tells Docker to look for the Dockerfile in the current directory

---

### Step 4: Run the Container

```bash
docker run --name apachecontainer -d -p 80:80 apache3
```

- `--name apachecontainer` gives the container a readable name
- `-d` runs it in detached (background) mode
- `-p 80:80` maps port 80 on the host to port 80 in the container

> ⚠️ **Security Note:** Exposing port 80 is done here for learning purposes only.
> In production, never expose port 80 directly — use HTTPS (443) with a
> reverse proxy like nginx or a load balancer.

---

### Step 5: Tag and Push to Docker Hub

```bash
# Log in to Docker Hub
docker login

# Tag the image for Docker Hub
docker tag apache3:latest safiao/apache3:latest

# Push to Docker Hub
docker push safiao/apache3:latest
```

---

### Step 6: Pull Image Back to Verify

```bash
docker pull safiao/apache3:latest
```

Confirms the image was pushed successfully and is publicly accessible.

---

## Troubleshooting

| Issue | Cause | Fix |
|-------|-------|-----|
| `debconf` warnings during `apt-get install` | Minimal base image has no dialog frontend | Safe to ignore — cosmetic only |
| Container exits immediately after `docker run` | No foreground process keeping it alive | Use `-d` flag or add `CMD` to Dockerfile |
| Port 80 already in use | Another service running on host port 80 | Change host port: `-p 8080:80` |

---

## 💡 What I Learned

- `docker run -it` drops you into a live container just like SSHing into a Linux machine
- `apt-get update` must run before installing anything — skipping it causes package not found errors
- `docker diff` shows every file the container touched compared to the base image — useful for auditing what an install actually does
- `docker commit` saves container changes as a new image but Dockerfiles are the better approach — repeatable and version-controlled
- Minimal base images like `ubuntu` have no dialog frontend — the `debconf` warnings during install are harmless
- Always tag images before pushing to Docker Hub — `docker tag <image> safiao/<image>:latest`
- Port mapping with `-p 80:80` is what makes a containerized app reachable from outside
- Avoid exposing port 80 in production environments