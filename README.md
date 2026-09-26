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

# Part 2 — Docker Compose & Internal Reverse Proxy

---

## Architecture

![Architecture](architecture2.png)

---

This part extends the previous Dockerization phase by improving the container architecture.

Instead of running containers manually with `docker run`, Docker Compose is now used to manage the services on each server.

The Flask application is also no longer exposed directly to the VM. Nginx is placed in front of the Flask backend as a reverse proxy.

The backend and Nginx containers communicate through a dedicated Docker bridge network.

The architecture is now:

```text
                         Load Balancer
                        192.168.56.8:80
                              |
                +-------------+-------------+
                |                           |
                v                           v
        Flask Server 1               Flask Server 2
        192.168.56.6                 192.168.56.7
                |                           |
        +-------+-------+           +-------+-------+
        | Docker Network|           | Docker Network|
        | white-network |           | white-network |
        |               |           |               |
        | Nginx :80     |           | Nginx :80     |
        |      |        |           |      |        |
        |      v        |           |      v        |
        | Backend :5000 |           | Backend :5000 |
        +---------------+           +---------------+
                |                           |
                +-------------+-------------+
                              |
                              v
                    PostgreSQL Server
                       192.168.56.5
```

The main changes are:

* Docker Compose is used instead of manually running containers.
* Nginx is added to each Flask server as a reverse proxy.
* Gunicorn continues to serve the Flask application on port `5000`.
* Nginx listens on port `80`.
* The backend container is no longer exposed directly on the VM.
* Nginx and the backend communicate through a Docker bridge network.
* The external Load Balancer continues to communicate with Nginx on port `80`.

---

# 10. Flask Server Architecture

Each Flask server now contains two containers:

```text
Flask Server
192.168.56.6 / 192.168.56.7
        |
        +----------------------+
        |   Docker Network     |
        |   white-network      |
        |                      |
        |  Nginx       Backend |
        |  :80   --->  :5000   |
        +----------------------+
```

The backend container runs:

```text
Gunicorn → Flask
```

while the Nginx container acts as a reverse proxy:

```text
Client
   |
   v
Nginx :80
   |
   v
Backend :5000
   |
   v
Gunicorn → Flask
```

The backend port `5000` is intentionally not published to the host.

Only Nginx exposes port `80`.

---

# 11. Create the Docker Network

A dedicated Docker bridge network is used so that the Nginx and backend containers can communicate using Docker's internal networking.

The network is defined in `compose.yaml`:

```yaml
networks:
  white-network:
    driver: bridge
```

Docker creates the network automatically when the Compose project is started.

To verify the network:

```bash
sudo docker network ls
```

The expected network is:

```text
white-network
```

Inspect it with:

```bash
sudo docker network inspect white-network
```

The backend and Nginx containers should both appear as members of the network.

---

# 12. Backend Docker Image

The backend continues to use Gunicorn and listens on port `5000`.

The Dockerfile contains:

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

The important part is:

```dockerfile
CMD ["gunicorn", "--bind", "0.0.0.0:5000", "run:app"]
```

Gunicorn listens on:

```text
0.0.0.0:5000
```

inside the container.

The port is not published directly to the VM.

---

# 13. Nginx Reverse Proxy

A separate Nginx container is placed in front of the Flask backend.

The Nginx configuration is:

```nginx
server {
    listen 80;

    location / {
        proxy_pass http://white-backend-container:5000;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

The important part is:

```nginx
proxy_pass http://white-backend-container:5000;
```

Docker's internal DNS resolves:

```text
white-backend-container
```

to the backend container's IP address because both containers belong to the same Docker network.

Therefore, Nginx does not need to know the backend container's IP address.

The communication becomes:

```text
Nginx
   |
   | Docker DNS
   v
white-backend-container:5000
```

---

# 14. Nginx Dockerfile

The Nginx image is built from the official Nginx Alpine image:

```dockerfile
FROM nginx:alpine

COPY nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

The command:

```dockerfile
CMD ["nginx", "-g", "daemon off;"]
```

starts Nginx in the foreground.

This is important inside Docker because the main process must remain running for the container to stay alive.

---

# 15. Docker Compose — Flask Server

The Flask server now uses Docker Compose to manage both containers.

Create:

```text
compose.yaml
```

with:

```yaml
services:

  backend:
    image: mahmoudeng/white-backend:${IMAGE_TAG}
    container_name: white-backend-container
    restart: unless-stopped
    env_file:
      - .env
    networks:
      - white-network

  nginx:
    image: mahmoudeng/white-nginx:v1
    container_name: white-nginx-container
    restart: unless-stopped
    ports:
      - "80:80"
    depends_on:
      - backend
    networks:
      - white-network

networks:
  white-network:
    driver: bridge
```

The backend does not contain:

```yaml
ports:
  - "5000:5000"
```

because the backend does not need to be accessible directly from the VM.

Instead, Nginx exposes:

```yaml
ports:
  - "80:80"
```

and communicates with the backend through:

```text
white-network
```

---

# 16. Environment Configuration

Create the `.env` file:

```bash
nano .env
```

For Flask Server 1:

```env
DATABASE_URL=postgresql+psycopg://white_app:pass@192.168.56.5:5432/white_db
BACKEND_SERVER=VM1
```

For Flask Server 2:

```env
DATABASE_URL=postgresql+psycopg://white_app:pass@192.168.56.5:5432/white_db
BACKEND_SERVER=VM2
```

The `.env` file contains environment-specific configuration and should not be committed to Git.

Verify:

```bash
cat .env
```

---

# 17. Start the Flask Server

The backend image is now pulled from Docker Hub.

The `IMAGE_TAG` variable determines which backend image version Docker Compose should use.

For example:

```bash
export IMAGE_TAG=<IMAGE_TAG>
```

Then start the Compose project:

```bash
sudo -E docker compose up -d
```

The `-E` option preserves the environment variable so Docker Compose can resolve:

```yaml
image: mahmoudeng/white-backend:${IMAGE_TAG}
```

For example, if:

```text
IMAGE_TAG=a81f32c9e7...
```

Compose will use:

```text
mahmoudeng/white-backend:a81f32c9e7...
```

---

# 18. Verify the Containers

Check running containers:

```bash
sudo docker compose ps
```

Expected services:

```text
white-backend-container
white-nginx-container
```

You can also use:

```bash
sudo docker ps
```

The backend should show its internal port:

```text
5000/tcp
```

while Nginx should show:

```text
0.0.0.0:80->80/tcp
```

This demonstrates that the backend is not directly published to the host.

---

# 19. Check the Backend Logs

View backend logs:

```bash
sudo docker compose logs backend
```

Follow the logs:

```bash
sudo docker compose logs -f backend
```

Gunicorn should start successfully and listen on:

```text
0.0.0.0:5000
```

---

# 20. Check Nginx Logs

View Nginx logs:

```bash
sudo docker compose logs nginx
```

Follow the logs:

```bash
sudo docker compose logs -f nginx
```

---

# 21. Test the Nginx Reverse Proxy

Because Nginx exposes port `80`, requests can now be sent to the server normally:

```bash
curl http://localhost/items
```

The request path is:

```text
curl
  |
  v
localhost:80
  |
  v
Nginx
  |
  | white-backend-container:5000
  v
Gunicorn
  |
  v
Flask
  |
  v
PostgreSQL
```

The expected response is the items stored in PostgreSQL.

---

# 22. Verify Docker Network Communication

Check the Docker network:

```bash
sudo docker network inspect white-network
```

Both containers should appear under the network's containers.

The backend can also be reached from the Nginx container using its Docker DNS name.

For example:

```bash
sudo docker exec white-nginx-container getent hosts white-backend-container
```

This should return the internal IP address assigned to the backend container.

This confirms that Docker's internal DNS is resolving the backend container name.

---

# 23. Test the Load Balancer

The external Load Balancer continues to communicate with the Flask servers through port `80`.

Its upstream configuration is:

```nginx
upstream white_backend {
    server 192.168.56.6:80;
    server 192.168.56.7:80;
}
```

The complete request path is now:

```text
Client
   |
   v
192.168.56.8:80
   |
   | Nginx Load Balancer
   |
   +--------------------+
   |                    |
   v                    v
192.168.56.6:80     192.168.56.7:80
   |                    |
   v                    v
Nginx :80           Nginx :80
   |                    |
   v                    v
Backend :5000       Backend :5000
   |                    |
   +---------+----------+
             |
             v
      PostgreSQL
      192.168.56.5
```

Test:

```bash
curl http://192.168.56.8/items
```

The request should be forwarded to one of the Flask servers.

---

# 24. Verify Load Balancing

The backend identifies which server handled the request using:

```text
X-Backend-Server
```

Check the response headers:

```bash
curl -I http://192.168.56.8/items
```

The response should contain either:

```text
X-Backend-Server: VM1
```

or:

```text
X-Backend-Server: VM2
```

Run multiple requests:

```bash
curl -I http://192.168.56.8/items
curl -I http://192.168.56.8/items
curl -I http://192.168.56.8/items
curl -I http://192.168.56.8/items
```

Nginx uses round-robin load balancing by default, so requests are distributed between the available backend servers.

---

# 25. Update the Backend Container

When a new backend image is available on Docker Hub, update the `IMAGE_TAG` variable:

```bash
export IMAGE_TAG=<NEW_IMAGE_TAG>
```

Then pull the new image:

```bash
sudo docker compose pull backend
```

Recreate the backend container:

```bash
sudo -E docker compose up -d backend
```

Verify:

```bash
sudo docker compose ps
```

The Nginx container does not need to be rebuilt because its configuration has not changed.

---

# 26. Restart the Complete Stack

To recreate the complete stack:

```bash
sudo docker compose up -d
```

To stop the stack:

```bash
sudo docker compose down
```

`docker compose down` removes the containers and network created by Compose but does not remove Docker images.

---

# 27. Inspect the Compose Configuration

Docker Compose can render the final configuration after environment variables have been substituted:

```bash
sudo -E docker compose config
```

This is useful for verifying that:

```yaml
image: mahmoudeng/white-backend:${IMAGE_TAG}
```

has been resolved to the expected image tag.

For example:

```text
mahmoudeng/white-backend:a81f32c9e7...
```

---
# 28. PostgreSQL Docker Compose

The PostgreSQL server also uses Docker Compose.

PostgreSQL runs on:

192.168.56.5

Its Compose project contains the PostgreSQL service only.

PostgreSQL compose.yaml
services:

  postgres:
    image: postgres:18
    container_name: white-postgres
    restart: unless-stopped
    env_file:
      - .env
    volumes:
      - postgres_data:/var/lib/postgresql
      - ./init:/docker-entrypoint-initdb.d
    ports:
      - "5432:5432"

volumes:
  postgres_data:
Environment file

The .env file contains:

POSTGRES_USER=white_app
POSTGRES_PASSWORD=YOUR_PASSWORD
POSTGRES_DB=white_db

The database is published on the VM's port 5432:

192.168.56.5:5432

The Flask servers connect to it using:

192.168.56.5:5432
PostgreSQL persistence

The named volume:

volumes:
  - postgres_data:/var/lib/postgresql

allows PostgreSQL data to persist independently from the lifecycle of the container.

The init directory is mounted to:

/docker-entrypoint-initdb.d

PostgreSQL's official image can execute initialization scripts from this directory when initializing a new database data directory.

For example:

init/
├── schema.sql
└── ...

These initialization scripts are intended for the initial database setup. They are not automatically re-executed every time the container starts if the database has already been initialized.

Start PostgreSQL

From the PostgreSQL server:

sudo docker compose up -d

Check the service:

sudo docker compose ps

Check logs:

sudo docker compose logs postgres

The database should eventually report that it is ready to accept connections.

# 29. Load Balancer Docker Compose

The Load Balancer VM is:

192.168.56.8

It runs its own Docker Compose project containing the Nginx load balancer.

The important point is that the Docker network on the Load Balancer VM is separate from the Docker networks on the Flask VMs.

Therefore, the Load Balancer communicates with the Flask servers through their VM IP addresses:

192.168.56.6:80
192.168.56.7:80
Load Balancer compose.yaml
services:

  load-balancer:
    image: mahmoudeng/white-load-balancer:v1
    container_name: white-load-balancer
    restart: unless-stopped
    ports:
      - "80:80"

The Load Balancer exposes:

192.168.56.8:80
Load Balancer Nginx configuration
upstream white_backend {
    server 192.168.56.6:80;
    server 192.168.56.7:80;
}

server {
    listen 80;

    location / {
        proxy_pass http://white_backend;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

Nginx uses round-robin load balancing by default.

Requests are distributed between:

192.168.56.6:80
192.168.56.7:80
Start the Load Balancer

From the Load Balancer VM:

sudo docker compose up -d

Check the service:

sudo docker compose ps

Check the logs:

sudo docker compose logs load-balancer

---
# 30. Phase 2 Updated Outcome

The infrastructure has now evolved from manually running individual Docker containers to a multi-container architecture managed by Docker Compose.

The Flask application servers now contain:

```text
Flask Server
192.168.56.6 / 192.168.56.7
        |
        +-------------------------------+
        |       Docker Bridge Network   |
        |                               |
        |   Nginx :80                  |
        |       |                       |
        |       v                       |
        |   Backend :5000              |
        |       |                       |
        +-------+-----------------------+
                |
                v
        PostgreSQL
        192.168.56.5
```

The complete architecture is:

```text
                         OPNsense
                       192.168.56.9
                             |
                             |
                    Private Network
                   192.168.56.0/24
                             |
                             v
                   Load Balancer
                   192.168.56.8
                         Nginx
                           |
              +------------+------------+
              |                         |
              v                         v
       Flask Server 1            Flask Server 2
       192.168.56.6              192.168.56.7
              |                         |
       +------+-------+          +------+-------+
       | Docker       |          | Docker       |
       | white-network|          | white-network|
       |              |          |              |
       | Nginx :80    |          | Nginx :80    |
       |      |       |          |      |       |
       |      v       |          |      v       |
       | Backend :5000|         | Backend :5000|
       +------+-------+          +------+-------+
              |                         |
              +------------+------------+
                           |
                           v
                    PostgreSQL
                    192.168.56.5
```

>This completes the Dockerization phase and prepares the infrastructure for the next phase: automation using GitHub Actions.
