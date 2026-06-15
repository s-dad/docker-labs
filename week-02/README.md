# Lab 02 — Build a Private Docker Registry & Image Tagging

## Objective
Build and run a private Docker registry locally, tag existing images to point
to that registry, push images into it, and verify the registry is operational
via the Docker Registry API.

---

## Environment

| Item | Detail |
|------|--------|
| Host OS | Windows 11 Home |
| Docker on Windows | Docker Desktop |
| VM | VirtualBox 7 → Ubuntu 22.04 LTS |
| Docker in VM | Docker Engine via `apt` |
| Shell | bash |
| Docker Hub | [hub.docker.com/u/safiao](https://hub.docker.com/u/safiao) |

---

## Part 1 — Launch the Private Registry

### Step 1: Start the Registry Container

```bash
docker run -d -p 5000:5000 --restart always --name registry registry:2
```

- `registry:2` is the official Docker registry image
- `-p 5000:5000` maps the registry to localhost port 5000
- `--restart always` ensures the registry restarts automatically after a reboot

### Step 2: Verify the Registry is Running

```bash
docker ps
```

Confirmed the registry container was up and listening on `localhost:5000`.

---

## Part 2 — Tag and Push Images to the Private Registry

Reused `nginx` and `busybox` images already available on the VM from the
previous lab — no need to pull them again.

### Step 3: Tag Images for the Private Registry

```bash
# Tag nginx
docker tag nginx localhost:5000/nginxprivate

# Tag busybox
docker tag busybox localhost:5000/busyboxprivate
```

Tagging with a registry prefix tells Docker where to push the image.

### Step 4: Push Images to the Private Registry

```bash
docker push localhost:5000/nginxprivate
docker push localhost:5000/busyboxprivate
```

Docker confirmed each push with layer digests:

```
The push refers to repository [localhost:5000/nginxprivate]
layer digest: sha256:...
latest: digest: sha256:... size: ...
```

---

## Part 3 — Configure Insecure Registry Access from the VM IP

### Step 5: Get the VM's IP Address

```bash
ip addr show
```

Located the VM's IP address from the output (e.g. `192.168.x.x`).

### Step 6: Allow Insecure Registry in Docker

Edited Docker's daemon config to allow the private registry over HTTP:

```bash
sudo nano /etc/docker/daemon.json
```

Added:

```json
{
  "insecure-registries": ["<VM-IP>:5000"]
}
```

### Step 7: Restart Docker

```bash
sudo systemctl restart docker
```

Required after any change to `daemon.json`.

---

## Part 4 — Pull and Push Alpine Image

### Step 8: Pull Alpine from Docker Hub

```bash
docker pull alpine
```

### Step 9: Tag and Push Alpine to Private Registry

```bash
docker tag alpine localhost:5000/alpineprivate
docker push localhost:5000/alpineprivate
```

> ⚠️ **Issue:** Laptop froze mid-push due to VirtualBox straining system
> resources on Windows 11 Home. After recovery, confirmed the registry
> container was still running with `docker ps`.

---

## Part 5 — Verify Registry via API

### Step 10: Install curl

```bash
sudo apt install -y curl
```

### Step 11: Query the Registry API

```bash
curl http://<VM-IP>:5000/v2/_catalog
```

**Output:**

```json
{"repositories":["alpineprivate","busyboxprivate","nginxprivate"]}
```

All three repositories returned — private registry confirmed operational.

---

## Troubleshooting

| Issue | Cause | Fix |
|-------|-------|-----|
| Laptop freezing repeatedly | VirtualBox consuming too many resources on Windows 11 Home | Reduced VM RAM allocation, closed background apps |
| Push rejected — HTTP not supported | Docker blocks insecure registries by default | Added VM IP to `insecure-registries` in `daemon.json` and restarted Docker |
| Registry not accessible after laptop freeze | Needed to verify container survived reboot | Ran `docker ps` to confirm `--restart always` kept it running |

---

## 💡 What I Learned

- A private registry lets you store images locally without pushing to Docker Hub — useful for sensitive or internal images
- `--restart always` is important for services like a registry — it keeps them running across reboots without manual intervention
- Docker blocks HTTP registries by default — `insecure-registries` in `daemon.json` is required for local non-SSL setups
- Image tagging with a registry prefix tells Docker where to push — `localhost:5000/myimage` vs `safiao/myimage`
- Layer digests in push output confirm each layer was stored successfully
- The Docker Registry API `/v2/_catalog` is a quick way to verify what images are stored in a registry
- VirtualBox on Windows 11 Home is resource-heavy — limiting VM RAM and closing background apps helps with stability
- Always restart the Docker daemon after editing `daemon.json` — changes do not apply until restart