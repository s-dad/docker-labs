# Lab 04 — Docker Private Registry with SSL Authentication & Docker Networking

## Objective
Secure a private Docker registry with SSL/TLS encryption and basic authentication on both Windows and Linux, then explore Docker networking concepts including bridge networks, host networking, port mapping, and container communication.

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

## Part 1 — SSL Registry: Directory Structure and Certificates (Windows)

### Step 1: Create Directory Structure

```powershell
mkdir C:\registry
cd C:\registry
mkdir certs
mkdir auth
```

### Step 2: Install OpenSSL and Add to PATH

Downloaded Win64 OpenSSL v3.x Light from https://slproweb.com/products/Win32OpenSSL.html and added it to PATH:

```powershell
$env:Path += ";C:\Program Files\OpenSSL-Win64\bin"
openssl version
```

### Step 3: Generate Private Key and Self-Signed Certificate

```powershell
cd C:\registry\certs
openssl genrsa -out domain.key 2048
openssl req -new -x509 -nodes -sha256 -days 365 -key domain.key -out domain.crt
```

When prompted, entered the following:

| Field | Value |
|-------|-------|
| Country Name | US |
| State | Washington |
| Locality | — |
| Organization | MyCompany |
| Organizational Unit | IT |
| Common Name | localhost |
| Email | — |

Verified both files exist:

```powershell
dir
# domain.crt
# domain.key
```

### Step 4: Generate Password File

```powershell
cd C:\registry\auth
docker run --rm --entrypoint htpasswd httpd:2 -Bbn myuser mypassword | Out-File -Encoding ASCII -NoNewline registry.password
type registry.password
```

---

## Part 2 — SSL Registry: Launch and Test (Windows)

### Step 5: Run Registry with SSL and Authentication

```powershell
cd C:\registry
docker run -d --restart=always --name privregistry2 `
  -v C:/registry/auth:/auth `
  -v C:/registry/certs:/certs `
  -e REGISTRY_AUTH=htpasswd `
  -e REGISTRY_AUTH_HTPASSWD_REALM="Registry Realm" `
  -e REGISTRY_AUTH_HTPASSWD_PATH=/auth/registry.password `
  -e REGISTRY_HTTP_ADDR=0.0.0.0:443 `
  -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt `
  -e REGISTRY_HTTP_TLS_KEY=/certs/domain.key `
  -p 443:443 registry:2
```

### Step 6: Verify Registry is Running

```powershell
docker ps
docker logs privregistry2
# Expected: level=info msg="listening on [::]:443"
```

### Step 7: Configure Docker to Trust the Self-Signed Certificate

In Docker Desktop → Settings → Docker Engine, added:

```json
{
  "insecure-registries": ["localhost:443"]
}
```

Clicked Apply & Restart.

### Step 8: Tag, Login, and Push

```powershell
docker pull nginx
docker tag nginx localhost:443/mynginx
docker login localhost:443
docker push localhost:443/mynginx
```

### Step 9: Verify by Pulling Back

```powershell
docker rmi localhost:443/mynginx
docker pull localhost:443/mynginx
```

---

## Part 3 — SSL Registry: Directory Structure and Certificates (Linux)

### Step 1: Create Directory Structure

```bash
mkdir registry && cd registry
mkdir certs auth
```

### Step 2: Generate Private Key and Self-Signed Certificate

```bash
cd certs
openssl genrsa 1024 > domain.key
chmod 400 domain.key
openssl req -new -x509 -nodes -sha1 -days 365 -key domain.key -out domain.crt
```

Verified both files exist:

```bash
ls
# domain.crt  domain.key
```

### Step 3: Generate Password File

```bash
cd ../auth
htpasswd -Bc registry.password learningdocker
```

---

## Part 4 — SSL Registry: Launch and Test (Linux)

### Step 4: Run Registry with SSL and Authentication

```bash
cd ..
docker run -d \
  --restart=always \
  --name privregistry2 \
  -v $(pwd)/auth:/auth \
  -v $(pwd)/certs:/certs \
  -e REGISTRY_AUTH=htpasswd \
  -e REGISTRY_AUTH_HTPASSWD_REALM="Registry Realm" \
  -e REGISTRY_AUTH_HTPASSWD_PATH=/auth/registry.password \
  -e REGISTRY_HTTP_ADDR=0.0.0.0:443 \
  -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt \
  -e REGISTRY_HTTP_TLS_KEY=/certs/domain.key \
  -p 443:443 \
  registry:2
```

> **Note:** The `docker run` command must be executed from inside the `registry/` directory for `$(pwd)` volume mounts to resolve correctly.

### Step 5: Configure Insecure Registry

```bash
sudo nano /etc/docker/daemon.json
```

Added:

```json
{
  "insecure-registries": ["<VM-IP>:443"]
}
```

Restarted Docker:

```bash
sudo systemctl restart docker.service
```

### Step 6: Login and Push

```bash
docker pull alpine
docker tag alpine <VM-IP>:443/alpineprivate
docker login <VM-IP>:443
docker push <VM-IP>:443/alpineprivate
```

---

## Part 5 — Docker Networking (Lab 6B)

### Network Types Overview

| Type | Behavior |
|------|----------|
| Bridge (default) | Containers communicate by IP only, no DNS |
| Bridge (custom) | Containers communicate by name via built-in DNS |
| Host | Container shares host network directly (Linux only) |
| None | Complete network isolation |

### Default Bridge — IP Only

```powershell
docker run -dit --name box1 busybox /bin/sh
docker run -dit --name box2 busybox /bin/sh
docker network inspect bridge
```

Pinged box1 from box2 by IP — succeeded. Pinged by container name — failed with `ping: bad address 'box1'`. The default bridge has no DNS.

```powershell
docker exec box2 ping -c 3 172.17.0.2   # succeeds
docker exec box2 ping -c 3 box1          # fails
docker stop box1 box2 && docker rm box1 box2
```

### Custom Bridge — DNS Enabled

```powershell
docker network create my-bridge-network
docker run -dit --name container1 --network my-bridge-network busybox /bin/sh
docker run -dit --name container2 --network my-bridge-network busybox /bin/sh
docker exec container2 ping -c 3 container1   # succeeds by name
docker stop container1 container2
docker rm container1 container2
docker network rm my-bridge-network
```

### Host Network (Limited on Windows)

```powershell
docker run --rm -d --network host --name nginx_host nginx
docker logs nginx_host
docker stop nginx_host
```

Host networking binds to WSL2's network on Windows, not directly to Windows. The container started successfully but was not reachable from a Windows browser. Port mapping is the correct approach on Windows.

### None Network — Full Isolation

```powershell
docker run -dit --name isolated --network none busybox /bin/sh
docker exec isolated ping -c 3 8.8.8.8   # fails: Network is unreachable
docker exec isolated ifconfig              # only loopback (lo) present
docker stop isolated && docker rm isolated
```

### Port Mapping

```powershell
# Without mapping — not accessible from browser
docker run -d --name web-hidden nginx
docker stop web-hidden && docker rm web-hidden

# With mapping — accessible at localhost:8080
docker run -d --name web-exposed -p 8080:80 nginx
# http://localhost:8080 shows NGINX welcome page

docker stop web-exposed && docker rm web-exposed
```

---

## Troubleshooting

| Issue | Cause | Fix |
|-------|-------|-----|
| Registry failed to start (Linux) | `$(pwd)` resolved to wrong path | Moved to `registry/` directory before running `docker run` |
| Docker rejected self-signed cert | `insecure-registries` not configured | Added VM IP to `/etc/docker/daemon.json` and restarted Docker |
| Auth errors on push (Windows) | Old `daemon.json` still pointing to `172.17.0.2:5000` from Lab 03 | Updated `insecure-registries` to `localhost:443` |
| High CPU during login attempts | bcrypt hashing is CPU-intensive; multiple retries compounded this | Resolved credentials issue to reduce retry loops |
| Forgot htpasswd password (Linux) | No recovery option for bcrypt hashes | Regenerated password file with a new password |
| Container name not resolving on bridge | Default bridge has no DNS | Switched to a custom bridge network |
| NGINX not accessible on host network (Windows) | Host network binds to WSL2, not Windows | Used port mapping (`-p 8080:80`) instead |

---

## What I Learned

- SSL/TLS adds a transport encryption layer on top of basic auth the registry now encrypts traffic in addition to requiring credentials
- Self-signed certificates require adding the registry host to `insecure-registries` in `daemon.json`Docker blocks untrusted certificates by default
- A `400` error from the registry indicates a daemon trust configuration issue; a `401` indicates a credentials issue  useful for narrowing down what to fix
- `$(pwd)` volume mounts on Linux are directory-sensitive  the command must run from the correct working directory
- The default bridge network supports IP-based communication only; custom bridge networks add DNS so containers can resolve each other by name
- Host networking works fully on Linux but is limited on Windows because containers run inside WSL2, not directly on the Windows network stack
- Port mapping (`-p host:container`) is the correct way to expose container services on Windows
- Running multiple registry containers simultaneously on different ports increases CPU load noticeably, especially with bcrypt-based auth