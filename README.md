# Cloud Infrastructure Lab — Phase 2

This phase focuses on containerizing the infrastructure built in Phase 1 using Docker.

The goal is to transform the manually configured services into containerized services while keeping the same architecture and functionality.

---

## Architecture

![Architecture](architecture.png)

---

## Infrastructure

| Server            | IP Address     | Role                |
| ----------------- | -------------- | ------------------- |
| PostgreSQL Server | `192.168.56.5` | Database            |
| Flask Server 1    | `192.168.56.6` | Backend Application |
| Flask Server 2    | `192.168.56.7` | Backend Application |
| Load Balancer     | `192.168.56.8` | Nginx Load Balancer |
| OPNsense          | `192.168.56.9` | Firewall            |

---

# 1. PostgreSQL Server

IP:

```text
192.168.56.5
```

The PostgreSQL server is containerized using the official PostgreSQL Docker image.

### Repository

[MHM-X/postgres-server](https://github.com/MHM-X/postgres-server?utm_source=chatgpt.com)

Clone the repository:

```bash
git clone https://github.com/MHM-X/postgres-server.git
cd postgres-server
```

### Stop the existing PostgreSQL service

```bash
sudo systemctl stop postgresql
sudo systemctl disable postgresql
```

### Create the Docker volume

```bash
sudo docker volume create postgres_data
```

### Pull the PostgreSQL image

```bash
sudo docker pull postgres:18
```

### Run PostgreSQL

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

### Verify the container

```bash
sudo docker ps
```

Check the logs:

```bash
sudo docker logs white-postgres
```

The database should eventually show:

```text
database system is ready to accept connections
```

The `schema.sql` file inside the `init` directory is automatically executed when PostgreSQL initializes an empty data volume.

---

# 2. Flask Server 1

IP:

```text
192.168.56.6
```

This server runs the Flask backend inside a Docker container using Gunicorn.

### Repository

[MHM-X/flask-server](https://github.com/MHM-X/flask-server?utm_source=chatgpt.com)

Clone the repository:

```bash
git clone https://github.com/MHM-X/flask-server.git
cd flask-server
```

### Create the environment file

Create:

```text
.env
```

Add:

```env
DATABASE_URL=postgresql+psycopg://white_app:pass@192.168.56.5:5432/white_db
BACKEND_SERVER=VM1
```

### Build the Docker image

```bash
sudo docker build -t white-backend-image .
```

### Run the container

```bash
sudo docker run -d \
  --name white-backend-container \
  --restart unless-stopped \
  --env-file .env \
  -p 80:5000 \
  white-backend-image
```

### Verify

```bash
sudo docker ps
```

Test the backend:

```bash
curl http://localhost/items
```

The application should return the items stored in PostgreSQL.

---

# 3. Flask Server 2

IP:

```text
192.168.56.7
```

This server runs the same Flask backend as Server 1.

### Repository

[MHM-X/flask-server](https://github.com/MHM-X/flask-server?utm_source=chatgpt.com)

Clone the repository:

```bash
git clone https://github.com/MHM-X/flask-server.git
cd flask-server
```

### Create the environment file

Create:

```text
.env
```

Add:

```env
DATABASE_URL=postgresql+psycopg://white_app:pass@192.168.56.5:5432/white_db
BACKEND_SERVER=VM2
```

### Build the Docker image

```bash
sudo docker build -t white-backend-image .
```

### Run the container

```bash
sudo docker run -d \
  --name white-backend-container \
  --restart unless-stopped \
  --env-file .env \
  -p 80:5000 \
  white-backend-image
```

### Verify

```bash
sudo docker ps
```

Test the backend:

```bash
curl http://localhost/items
```

---

# 4. Load Balancer

IP:

```text
192.168.56.8
```

The Load Balancer uses Nginx inside a Docker container.

It distributes incoming requests between:

```text
192.168.56.6:80
192.168.56.7:80
```

### Repository

[MHM-X/lb-server](https://github.com/MHM-X/lb-server?utm_source=chatgpt.com)

Clone the repository:

```bash
git clone https://github.com/MHM-X/lb-server.git
cd lb-server
```

### Build the Docker image

```bash
sudo docker build -t white-load-balancer .
```

### Run the container

```bash
sudo docker run -d \
  --name white-load-balancer \
  --restart unless-stopped \
  -p 80:80 \
  white-load-balancer
```

### Verify

```bash
sudo docker ps
```

Check the Nginx logs:

```bash
sudo docker logs white-load-balancer
```

---

# 5. Load Balancer Configuration

The Nginx configuration uses an upstream group containing both Flask servers:

```nginx
upstream white_backend {
    server 192.168.56.6:80;
    server 192.168.56.7:80;
}
```

Requests received by the Load Balancer are forwarded to the backend servers:

```nginx
location / {
    proxy_pass http://white_backend;

    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```

Nginx uses round-robin load balancing by default.

---

# 6. Test the Load Balancer

From any machine that can reach the private network:

```bash
curl http://192.168.56.8/items
```

You should receive the response from the Flask backend.

To verify that requests are reaching both backend servers, use:

```bash
curl -I http://192.168.56.8/items
```

The response contains:

```text
X-Backend-Server: VM1
```

or:

```text
X-Backend-Server: VM2
```

Run the command multiple times:

```bash
curl -I http://192.168.56.8/items
curl -I http://192.168.56.8/items
curl -I http://192.168.56.8/items
curl -I http://192.168.56.8/items
```

The backend server should alternate between VM1 and VM2.

---

# 7. Failure Test

Stop one of the backend containers.

For example, on `192.168.56.6`:

```bash
sudo docker stop white-backend-container
```

Then test the Load Balancer:

```bash
curl http://192.168.56.8/items
```

The request should still be served by the remaining backend server.

Start the container again:

```bash
sudo docker start white-backend-container
```

---

# 8. Container Persistence

The containers use:

```text
--restart unless-stopped
```

This allows Docker to automatically restart the containers after a VM reboot or Docker service restart.

Check the restart policy:

```bash
sudo docker inspect -f '{{.HostConfig.RestartPolicy.Name}}' white-backend-container
```

Expected:

```text
unless-stopped
```

The same applies to:

```text
white-postgres
white-load-balancer
```

---

# 9. Useful Docker Commands

### List running containers

```bash
sudo docker ps
```

### List all containers

```bash
sudo docker ps -a
```

### View container logs

```bash
sudo docker logs <container-name>
```

### Follow container logs

```bash
sudo docker logs -f <container-name>
```

### Stop a container

```bash
sudo docker stop <container-name>
```

### Start a container

```bash
sudo docker start <container-name>
```

### Restart a container

```bash
sudo docker restart <container-name>
```

### Remove a container

```bash
sudo docker rm <container-name>
```

### List Docker images

```bash
sudo docker images
```

### List Docker volumes

```bash
sudo docker volume ls
```

---

# Phase 2 Outcome

At the end of Phase 2, the infrastructure has been transformed from manually running services into a Docker-based environment.

The final setup contains:

```text
PostgreSQL
    ↓
Docker Container
    ↓
192.168.56.5


Flask Server 1
    ↓
Docker Container
    ↓
Gunicorn
    ↓
192.168.56.6


Flask Server 2
    ↓
Docker Container
    ↓
Gunicorn
    ↓
192.168.56.7


Load Balancer
    ↓
Docker Container
    ↓
Nginx
    ↓
192.168.56.8
```

The Load Balancer distributes traffic between the two containerized Flask servers, while both application servers communicate with the containerized PostgreSQL database.

This completes the Dockerization phase and prepares the infrastructure for the next phase: automation using GitHub Actions.
