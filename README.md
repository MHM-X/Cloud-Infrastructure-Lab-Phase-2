# Cloud Infrastructure Lab — Phase 2

This phase focuses on **containerizing the infrastructure** built in Phase 1 using Docker.

The existing multi-server architecture was migrated from traditional system services to Docker containers while keeping the same application flow and internal network design.

## Architecture

```text
                    Private Network
                  192.168.56.0/24

                         │
                         ▼
                ┌─────────────────┐
                │    OPNsense     │
                │ 192.168.56.9    │
                └─────────────────┘

                         │

                ┌─────────────────┐
                │  Load Balancer  │
                │ 192.168.56.8    │
                │  Nginx + Docker │
                └────────┬────────┘
                         │
                ┌────────┴────────┐
                │                 │
                ▼                 ▼
        ┌──────────────┐  ┌──────────────┐
        │ App Server 1 │  │ App Server 2 │
        │ 192.168.56.6 │  │ 192.168.56.7 │
        │    Docker    │  │    Docker    │
        │ Flask/Gunicorn│ │ Flask/Gunicorn│
        └──────┬───────┘  └──────┬───────┘
               │                 │
               └────────┬────────┘
                        │
                        ▼
                ┌─────────────────┐
                │   PostgreSQL    │
                │ 192.168.56.5    │
                │     Docker      │
                └─────────────────┘
```

## Infrastructure

| Server         | IP             | Role                | Container                 |
| -------------- | -------------- | ------------------- | ------------------------- |
| PostgreSQL     | `192.168.56.5` | Database            | `white-postgres`          |
| Flask Server 1 | `192.168.56.6` | Backend             | `white-backend-container` |
| Flask Server 2 | `192.168.56.7` | Backend             | `white-backend-container` |
| Load Balancer  | `192.168.56.8` | Nginx Load Balancer | `white-load-balancer`     |
| OPNsense       | `192.168.56.9` | Firewall            | —                         |

---

# 1. PostgreSQL Server

**IP:** `192.168.56.5`

The PostgreSQL server was migrated from a traditional system service to the official PostgreSQL Docker image.

## 1.1 Stop the existing PostgreSQL service

```bash
sudo systemctl stop postgresql
sudo systemctl disable postgresql
```

Verify:

```bash
sudo systemctl status postgresql
```

The service should be inactive.

## 1.2 Clone the repository

```bash
cd ~
git clone <POSTGRES-REPOSITORY-URL>
cd postgres-server
```

The project contains:

```text
postgres-server/
├── .env
└── init/
    └── schema.sql
```

## 1.3 Environment variables

The `.env` file contains the variables used by the official PostgreSQL image during initialization:

```env
POSTGRES_DB=white_db
POSTGRES_USER=white_app
POSTGRES_PASSWORD=pass
```

The `.env` file should not be committed to GitHub when it contains real credentials.

## 1.4 Create the Docker volume

```bash
sudo docker volume create postgres_data
```

The volume provides persistent storage for PostgreSQL data.

## 1.5 Pull the PostgreSQL image

```bash
sudo docker pull postgres:18
```

No custom Dockerfile is required because the official PostgreSQL image is used.

## 1.6 Run the PostgreSQL container

```bash
sudo docker run -d \
  --name white-postgres \
  --restart unless-stopped \
  --env-file .env \
  -v postgres_data:/var/lib/postgresql \
  -v ./init:/docker-entrypoint-initdb.d \
  -p 5432:5432 \
  postgres:18
```

### Volume mounts

```text
postgres_data
      │
      ▼
/var/lib/postgresql
```

provides persistent PostgreSQL storage.

And:

```text
./init/schema.sql
        │
        ▼
/docker-entrypoint-initdb.d/schema.sql
```

allows the official image to execute the schema during the initial database setup.

## 1.7 Verify the container

```bash
sudo docker ps
```

Check logs:

```bash
sudo docker logs white-postgres
```

The database should eventually report:

```text
database system is ready to accept connections
```

---

# 2. Flask Application Server 1

**IP:** `192.168.56.6`

The first backend server runs the Flask application inside a Docker container using Gunicorn.

Nginx was removed from the application server because load balancing is now handled by the dedicated Load Balancer server.

## 2.1 Stop the old services

```bash
sudo systemctl stop white-backend.service
sudo systemctl stop nginx.service

sudo systemctl disable white-backend.service
sudo systemctl disable nginx.service
```

## 2.2 Install Docker and Git

```bash
sudo apt update
sudo apt install -y docker.io git
```

Enable Docker:

```bash
sudo systemctl enable --now docker
```

## 2.3 Clone the Flask repository

```bash
cd ~
git clone <FLASK-REPOSITORY-URL>
cd flask-server
```

## 2.4 Configure environment variables

Create `.env`:

```env
DATABASE_URL=postgresql+psycopg://white_app:pass@192.168.56.5:5432/white_db
BACKEND_SERVER=VM1
```

The `.env` file is passed to the container at runtime and is not included in the Docker image.

## 2.5 Build the Docker image

```bash
sudo docker build -t white-backend-image .
```

The Dockerfile:

```dockerfile
FROM python:3.12-slim

WORKDIR /flask-app

RUN apt-get update \
    && apt-get install -y libpq5 \
    && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 5000

CMD ["gunicorn", "--bind", "0.0.0.0:5000", "run:app"]
```

`libpq5` is required by `psycopg` to communicate with PostgreSQL.

## 2.6 Run the container

```bash
sudo docker run -d \
  --name white-backend-container \
  --restart unless-stopped \
  --env-file .env \
  -p 80:5000 \
  white-backend-image
```

The port mapping is:

```text
VM :80
  ↓
Container :5000
  ↓
Gunicorn
  ↓
Flask
```

## 2.7 Verify the application

```bash
sudo docker ps
```

Check logs:

```bash
sudo docker logs white-backend-container
```

Test locally:

```bash
curl http://localhost/items
```

Test from another server:

```bash
curl http://192.168.56.6/items
```

---

# 3. Flask Application Server 2

**IP:** `192.168.56.7`

The second backend server uses the same Docker image and application code as Server 1.

The difference is its environment configuration.

## 3.1 Stop the old services

```bash
sudo systemctl stop white-backend.service
sudo systemctl stop nginx.service

sudo systemctl disable white-backend.service
sudo systemctl disable nginx.service
```

## 3.2 Install Docker and Git

```bash
sudo apt update
sudo apt install -y docker.io git

sudo systemctl enable --now docker
```

## 3.3 Clone the Flask repository

```bash
cd ~
git clone <FLASK-REPOSITORY-URL>
cd flask-server
```

## 3.4 Configure environment variables

```env
DATABASE_URL=postgresql+psycopg://white_app:pass@192.168.56.5:5432/white_db
BACKEND_SERVER=VM2
```

## 3.5 Build the image

```bash
sudo docker build -t white-backend-image .
```

## 3.6 Run the container

```bash
sudo docker run -d \
  --name white-backend-container \
  --restart unless-stopped \
  --env-file .env \
  -p 80:5000 \
  white-backend-image
```

## 3.7 Verify

```bash
sudo docker ps
```

```bash
sudo docker logs white-backend-container
```

Test:

```bash
curl http://192.168.56.7/items
```

---

# 4. Load Balancer

**IP:** `192.168.56.8`

The Load Balancer runs Nginx inside a Docker container.

Unlike Phase 1, Nginx is no longer installed directly on the VM. It runs entirely inside the container.

## 4.1 Install Docker and Git

```bash
sudo apt update
sudo apt install -y docker.io git

sudo systemctl enable --now docker
```

## 4.2 Clone the Load Balancer repository

```bash
cd ~
git clone <LOAD-BALANCER-REPOSITORY-URL>
cd load-balancer
```

The project contains:

```text
load-balancer/
├── Dockerfile
└── nginx.conf
```

## 4.3 Build the image

```bash
sudo docker build -t white-load-balancer .
```

Dockerfile:

```dockerfile
FROM nginx:latest

COPY nginx.conf /etc/nginx/nginx.conf

CMD ["nginx", "-g", "daemon off;"]
```

Nginx runs in the foreground so it remains the main process of the Docker container.

## 4.4 Nginx configuration

The Load Balancer forwards requests to both Flask servers:

```nginx
upstream white_backend {
    server 192.168.56.6:80;
    server 192.168.56.7:80;
}
```

Requests are forwarded using:

```nginx
location / {
    proxy_pass http://white_backend;

    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```

## 4.5 Run the Load Balancer

```bash
sudo docker run -d \
  --name white-load-balancer \
  --restart unless-stopped \
  -p 80:80 \
  white-load-balancer
```

Traffic flow:

```text
Client
  │
  ▼
192.168.56.8:80
  │
  ▼
Nginx Container
  │
  ├──────────────► 192.168.56.6:80
  │
  └──────────────► 192.168.56.7:80
```

## 4.6 Verify

```bash
sudo docker ps
```

Check logs:

```bash
sudo docker logs white-load-balancer
```

Test:

```bash
curl http://192.168.56.8/items
```

---

# 5. Load Balancing Test

The Flask application exposes the backend server identity through the `X-Backend-Server` response header.

Server 1:

```text
X-Backend-Server: VM1
```

Server 2:

```text
X-Backend-Server: VM2
```

Test the Load Balancer repeatedly:

```bash
curl -I http://192.168.56.8/items
curl -I http://192.168.56.8/items
curl -I http://192.168.56.8/items
curl -I http://192.168.56.8/items
```

The responses should show requests being handled by both backend servers.

---

# 6. Failure Test

Stop the backend container on Server 1:

```bash
sudo docker stop white-backend-container
```

Then test the Load Balancer:

```bash
curl http://192.168.56.8/items
```

The remaining backend server should continue serving requests.

Start Server 1 again:

```bash
sudo docker start white-backend-container
```

---

# 7. Container Persistence

All application and infrastructure containers use:

```text
--restart unless-stopped
```

This means Docker automatically restarts the container after the VM or Docker daemon starts again, unless the container was intentionally stopped by the administrator.

Current containers:

```text
192.168.56.5
└── white-postgres

192.168.56.6
└── white-backend-container

192.168.56.7
└── white-backend-container

192.168.56.8
└── white-load-balancer
```

---

# 8. Useful Docker Commands

## List running containers

```bash
sudo docker ps
```

## List all containers

```bash
sudo docker ps -a
```

## View logs

```bash
sudo docker logs <container-name>
```

## Follow logs

```bash
sudo docker logs -f <container-name>
```

## Stop a container

```bash
sudo docker stop <container-name>
```

## Start a container

```bash
sudo docker start <container-name>
```

## Restart a container

```bash
sudo docker restart <container-name>
```

## Remove a container

```bash
sudo docker rm <container-name>
```

## List images

```bash
sudo docker images
```

## List volumes

```bash
sudo docker volume ls
```

## Inspect a container

```bash
sudo docker inspect <container-name>
```

---

# Phase 2 Outcome

Phase 2 migrated the infrastructure from traditional server processes to containerized services.

The final architecture consists of:

* Dockerized Flask application servers
* Gunicorn application runtime
* Dockerized Nginx Load Balancer
* Dockerized PostgreSQL
* Persistent PostgreSQL storage using Docker volumes
* Environment-based configuration
* Automatic container restart policies
* Load balancing between two backend servers
* Backend failure testing

The next phase will introduce **automation using GitHub Actions**, followed by further infrastructure automation with Terraform and Ansible.
