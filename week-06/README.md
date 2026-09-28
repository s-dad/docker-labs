# Lab 06 — Docker Swarm

## Objective
Create and configure a Docker Swarm cluster with one manager and two workers
using Docker-in-Docker (DinD) containers on Docker Desktop for Windows.

---

## Environment

| Item | Detail |
|------|--------|
| Host OS | Windows 11 Home |
| Docker on Windows | Docker Desktop |
| Shell | PowerShell |
| Swarm Setup | 1 Manager + 2 Workers via `docker:20.10-dind` |

---

## Part 1 — Create the Network and Containers

### Step 1: Create a Custom Bridge Network

```powershell
docker network create --driver bridge swarm_net
```

Uses an underscore in the name — Docker Compose reserved names with hyphens
can cause conflicts with Swarm networking.

### Step 2: Run the Manager Container

```powershell
docker run -d --privileged --name swarm-manager --hostname swarm-manager --network swarm_net docker:20.10-dind
```

### Step 3: Run Worker Containers

```powershell
docker run -d --privileged --name worker1 --hostname worker1 --network swarm_net docker:20.10-dind
docker run -d --privileged --name worker2 --hostname worker2 --network swarm_net docker:20.10-dind
```

`--privileged` is required for Docker-in-Docker — each container runs its own
Docker daemon.

---

## Part 2 — Initialize the Swarm

### Step 4: Get the Manager's IP Address

```powershell
docker inspect swarm-manager | ConvertFrom-Json | ForEach-Object {
    $_.NetworkSettings.Networks.swarm_net.IPAddress
}
```

Noted the returned IP address, e.g. `172.19.0.2`.

### Step 5: Exec into the Manager Container

```powershell
docker exec -it swarm-manager sh
```

### Step 6: Initialize the Swarm

```sh
docker swarm init --advertise-addr 172.19.0.2
```

### Step 7: Get the Worker Join Token

```sh
docker swarm join-token worker -q
```

Copied the token output for use in the next steps.

---

## Part 3 — Join Workers to the Swarm

### Step 8: Exec into Each Worker and Join

Opened separate PowerShell windows for each worker:

```powershell
docker exec -it worker1 sh
```

```sh
docker swarm join --token <WORKER_TOKEN> 172.19.0.2:2377
```

```powershell
docker exec -it worker2 sh
```

```sh
docker swarm join --token <WORKER_TOKEN> 172.19.0.2:2377
```

---

## Part 4 — Verify the Cluster

### Step 9: Check Node Status

```powershell
docker exec -it swarm-manager docker node ls
```

Expected output:

```
ID                            HOSTNAME        STATUS    AVAILABILITY   MANAGER STATUS
xxxxxxxxxxxx *   swarm-manager   Ready     Active         Leader
xxxxxxxxxxxx     worker1         Ready     Active
xxxxxxxxxxxx     worker2         Ready     Active
```

All three nodes showing `Ready` confirmed the swarm was operational.

---

## Troubleshooting

| Issue | Cause | Fix |
|-------|-------|-----|
| Docker Desktop wouldn't start | WSL2 VM failed to allocate memory in time | Freed disk space, relaunched Docker Desktop and waited for it to fully initialize |
| `docker-desktop` node joined swarm unexpectedly | Exited worker1 shell before running join command — ran it against host Docker engine instead | Re-exec'd into worker1 and ran the join command from inside the container |
| Stale `Down` node in `docker node ls` | Leftover from the accidental host join | Removed it with `docker node rm <node-id>` from the manager |

---

## What I Learned

- Docker Swarm turns multiple Docker hosts into a single cluster — a manager coordinates workloads and workers execute them
- Docker-in-Docker (`dind`) lets each container run its own Docker daemon, which makes it possible to simulate a multi-node swarm on a single machine
- `--advertise-addr` tells the swarm which IP other nodes should use to reach the manager — using the container's IP on `swarm_net` rather than the host IP is what makes intra-container communication work
- The join token is scoped — worker tokens and manager tokens are different; using the wrong one changes a node's role
- Staying aware of which shell context is active matters — running a swarm join command from the host PowerShell prompt instead of inside a worker container silently adds the wrong node to the cluster
- `docker node ls` must be run from the manager — workers have no visibility into cluster state