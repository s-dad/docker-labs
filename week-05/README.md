# Lab 05 — Docker Compose: WordPress and MySQL

## Objective
Use Docker Compose to define and run a multi-container WordPress application
with a MySQL database, persistent storage, and automatic container networking.

---

## Environment

| Item | Detail |
|------|--------|
| Host OS | Windows 11 Home |
| Docker on Windows | Docker Desktop |
| Shell | PowerShell |

---

## Part 1 — Project Setup

### Step 1: Verify Docker Compose is Installed

```powershell
docker-compose --version
```

### Step 2: Create Project Directory

```powershell
mkdir C:\dockerfiles
cd C:\dockerfiles
```

### Step 3: Create docker-compose.yml

```powershell
notepad docker-compose.yml
```

Pasted the following configuration:

```yaml
version: '3'

services:
  db:
    image: mysql:5.7
    volumes:
      - db_data:/var/lib/mysql
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: mypassword123
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wpuser
      MYSQL_PASSWORD: wppassword123

  wordpress:
    depends_on:
      - db
    image: wordpress:latest
    ports:
      - "8080:80"
    restart: always
    environment:
      WORDPRESS_DB_HOST: db:3306
      WORDPRESS_DB_USER: wpuser
      WORDPRESS_DB_PASSWORD: wppassword123
      WORDPRESS_DB_NAME: wordpress

volumes:
  db_data: {}
```

---

## Part 2 — Configuration Breakdown

### Database Service (`db`)

| Key | Purpose |
|-----|---------|
| `image: mysql:5.7` | Pins MySQL to version 5.7 for compatibility |
| `volumes` | Mounts `db_data` volume to persist database files |
| `restart: always` | Auto-restarts the container if it crashes |
| `MYSQL_ROOT_PASSWORD` | Root admin password for MySQL |
| `MYSQL_DATABASE` | Creates a database named `wordpress` on startup |
| `MYSQL_USER` / `MYSQL_PASSWORD` | Creates a non-root user for WordPress to connect with |

### WordPress Service

| Key | Purpose |
|-----|---------|
| `depends_on: db` | Ensures MySQL starts before WordPress |
| `image: wordpress:latest` | Always pulls the latest WordPress release |
| `ports: "8080:80"` | Exposes WordPress on `localhost:8080` |
| `WORDPRESS_DB_HOST: db:3306` | Connects to MySQL using the service name `db` as the hostname |

> **Note:** WordPress connects to MySQL using the service name `db` rather than an IP address. Docker's built-in DNS on the Compose network resolves `db` to the MySQL container automatically. Hardcoding an IP would break if the container restarted and received a new address.

### Volume

```yaml
volumes:
  db_data: {}
```

Creates a named volume managed by Docker. Database files persist even if containers are stopped or removed.

---

## Part 3 — Launch and Verify

### Step 4: Validate Configuration

```powershell
docker-compose config
```

### Step 5: Start the Application

```powershell
docker-compose up -d
```

Docker pulled the MySQL and WordPress images, created the `dockerfiles_default`
network, created the `dockerfiles_db_data` volume, and started both containers.

Expected output: 

dockerfiles_db_1 Up 3306/tcp
dockerfiles_wordpress_1 Up 0.0.0.0:8080->80/tcp


### Step 7: Open WordPress in Browser

Navigated to `http://localhost:8080` and completed the WordPress installation:

- Site Title: My Docker WordPress Site
- Created admin credentials
- Clicked Install WordPress

---

## Part 4 — Inspect the Stack

### View the Auto-Created Network

```powershell
docker network ls
docker network inspect dockerfiles_default
```

Docker Compose automatically created `dockerfiles_default` so both containers
could communicate. Both `db` and `wordpress` appeared in the Containers section.

### View the Volume

```powershell
docker volume ls
```

Confirmed `dockerfiles_db_data` was created for MySQL persistence.

### Test Data Persistence

```powershell
docker-compose down
docker-compose up -d
```

Navigated back to `http://localhost:8080` — the WordPress site and post were
still there, confirming the volume preserved data across container recreation.

---

## Troubleshooting

| Issue | Cause | Fix |
|-------|-------|-----|
| WordPress can't connect to database | MySQL not fully initialized when WordPress starts | `depends_on` handles start order; WordPress retries until MySQL is ready |
| Data lost after `docker-compose down` | Volume not defined | Named volume in `volumes:` block persists data across restarts |
| YAML syntax error on `docker-compose config` | Indentation issue in `.yml` file | Fixed spacing — YAML is whitespace-sensitive |

---

## What I Learned

- Docker Compose treats a multi-container app as a single system defined in one file rather than a set of individually managed containers
- `depends_on` controls start order but Compose also handles retry logic so WordPress waits for MySQL to be ready without manual intervention
- Service names act as stable DNS hostnames within a Compose network — using `db` instead of an IP address means the connection survives container restarts
- Named volumes persist database data independently of container lifecycles — `docker-compose down` removes containers but not volumes
- The `docker-compose.yml` file is a reproducible contract for the entire application stack — the same file rebuilds the identical environment anywhere Docker is installed