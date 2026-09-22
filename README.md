# 🐳 docker-labs

Lab journal documenting Docker work commands, troubleshooting, and lessons learned.

---

## 🖥 Environment

| Item | Detail |
|------|--------|
| Host OS | Windows 11 Home |
| VM | VirtualBox 7 → Ubuntu 22.04 LTS |
| Docker in VM | Docker Engine via apt |
| Shell | bash |
| Docker Hub | [hub.docker.com/u/safiao](https://hub.docker.com/u/safiao) |


---

## 📁 Labs

| Lab | Topic |
|------|-------|
| [Lab 01](./week-01/) | Build Docker Interactive Image & Linux Docker Images  |
| [Lab 02](./week-02/) | Build a Private Docker Registry (Windows – No SSL) & Image Tagging  |
| [Lab 03](./week-03/) | Docker Private Registry with Basic Authentication & Docker Storage |


## 📌 Quick Reference

```bash
# Check Docker is running
sudo systemctl status docker

# Start Docker daemon if stopped
sudo systemctl start docker

# Run a container and remove after exit
docker run --rm -it ubuntu bash

# Fix DNS issues inside a container
docker run --dns 8.8.8.8 <image>

# List all containers including stopped
docker ps -a

# Remove all stopped containers
docker container prune
```

