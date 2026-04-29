# MEAN Stack CRUD App — Tutorials Manager
\- MEAN Stack CRUD App — Tutorials Manager (AWS Deployed)
A full-stack CRUD application built with the MEAN stack (MongoDB, Express, Angular 15, Node.js), containerised with Docker, and served via Nginx. The application is hosted on an AWS EC2 instance.

> \*\*Live URL (local):\*\* http://localhost  
> \*\*API Base:\*\* http://localhost/api/tutorials

\---

## 📁 Project Structure

```
crud-dd-task-mean-app/
├── backend/                  # Node.js + Express API
│   ├── app/                  # Controllers, Models, Routes
│   ├── Dockerfile            # Backend Docker instructions
│   ├── package.json          # Dependencies
│   └── server.js             # Entry point
├── frontend/                 # Angular 15 Application
│   ├── src/                  # Components & Services
│   ├── Dockerfile            # Multi-stage Docker build
│   ├── nginx.conf            # Internal Nginx config (Port 8081)
│   └── package.json          # Frontend dependencies
├── nginx/                    # Reverse Proxy Setup
│   └── default.conf          # Main Nginx config (Port 80)
├── docker-compose.yml        # Orchestration (Images: Aditya)
└── README.md                 # Project Documentation
```

\---

## 🛠️ Prerequisites

|Tool|Version|Install|
|-|-|-|
|Docker|24+|https://docs.docker.com/get-docker/|
|Docker Compose|v2+|Bundled with Docker Desktop|
|Node.js|25-alpine|https://nodejs.org (for local dev only)|
|Git|Any|https://git-scm.com|

\---

## 🚀 Quick Start with Docker Compose

### 1\. Clone the Repository

```bash
git clone https://github.com/<your-username>/crud-dd-task-mean-app.git
cd crud-dd-task-mean-app
```

### 2\. Start All Services

```bash
docker compose up -d --build
```

This single command:

* Starts **MongoDB** (with health check)
* Builds \& runs the **Node.js backend** (waits for MongoDB to be healthy)
* Builds \& runs the **Angular frontend** (nginx, port 8081 internal)
* Starts **Nginx reverse proxy** on **port 80**

### 3\. Open the App

Open your browser and go to: **http://localhost**

### 4\. Verify Services

```bash
docker compose ps
```

Expected output:

```
NAME            IMAGE                    STATUS          PORTS
crud-mongodb    mongo:latest             healthy         27017/tcp
crud-backend    ...-backend:latest       running         8080/tcp
crud-frontend   ...-frontend:latest      running         8081/tcp
crud-nginx      nginx:stable-alpine      running         0.0.0.0:80->80/tcp
```

\---

## 🧱 Architecture

```
Browser
   │
   ▼
┌─────────────────────────────────┐
│  Nginx Reverse Proxy  :80       │
│  (crud-nginx container)         │
└──────────┬──────────────────────┘
           │
    ┌──────┴──────────────┐
    │                     │
    ▼                     ▼
/api/\*              everything else
    │                     │
    ▼                     ▼
┌──────────┐        ┌───────────┐
│ Backend  │        │ Frontend  │
│ Node.js  │        │ Angular   │
│  :8080   │        │  :8081    │
└────┬─────┘        └───────────┘
     │
     ▼
┌──────────┐
│ MongoDB  │
│  :27017  │
└──────────┘


All containers → custom-network (Docker bridge)
MongoDB data → persisted in `mongo\_data` named volume
```

\---

## ⚙️ Dockerfiles

### Backend — `backend/Dockerfile`

```dockerfile
FROM node:25-alpine
WORKDIR /app
COPY package\*.json ./
RUN npm install
COPY . .
EXPOSE 8080
CMD \["node", "server.js"]
```

* Uses **Node.js 25 Alpine** (minimal image)
* Exposes **port 8080**
* Runs directly with `node server.js`

### Frontend — `frontend/Dockerfile`

```dockerfile
# Stage 1: Build
FROM node:25-alpine AS build
WORKDIR /app
COPY package\*.json ./
RUN npm install
COPY . .
RUN npm run build -- --configuration production

# Stage 2: Serve
FROM nginx:stable-alpine
RUN rm -rf /usr/share/nginx/html/\*
COPY --from=build /app/dist/angular-15-crud /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 8081
CMD \["nginx", "-g", "daemon off;"]
```

* **Multi-stage build**: only the compiled static files are in the final image
* Build stage uses **Node 25 Alpine**, serve stage uses **Nginx stable-alpine**
* Final image is very small (no Node.js at all)

\---

## 🐳 Docker Compose — `docker-compose.yml`

```yaml
services:
  mongodb:   # mongo:latest + healthcheck + named volume
  backend:   # depends\_on mongodb (condition: service\_healthy)
  frontend:  # depends\_on backend
  nginx:     # port 80 exposed, mounts ./nginx/default.conf

volumes:
  mongo\_data:          # persistent MongoDB storage
networks:
  custom-network:      # bridge network for all containers
```

Key features:

* **Healthcheck** on MongoDB — backend waits until Mongo is truly ready
* **Named volume** `mongo\_data` — data persists across container restarts
* Only **nginx** exposes a host port (port 80); all other services are internal

\---

## 🔀 Nginx Configuration

### Reverse Proxy — `nginx/default.conf`

Routes all traffic entering on port 80:

|Path|Destination|
|-|-|
|`/api/\*`|`backend:8080` (Node.js API)|
|`/` (everything else)|`frontend:8081` (Angular SPA)|

```nginx
location /api/ {
    proxy\_pass http://backend;
}
location / {
    proxy\_pass http://frontend;
}
```

### Frontend Nginx — `frontend/nginx.conf`

Serves the Angular build inside the frontend container:

* Listens on **port 8081**
* `try\_files $uri $uri/ /index.html` — enables Angular client-side routing
* Cache headers for static assets (JS, CSS, images)

\---

## 🖥️ Manual Docker Image Build \& Push

If you want to build and push images manually (without CI/CD):

```bash
# Login to Docker Hub
docker login

# Build backend image
docker build -t <your-dockerhub-username>/crud-backend:latest ./backend

# Build frontend image
docker build -t <your-dockerhub-username>/crud-frontend:latest ./frontend

# Push images
docker push <your-dockerhub-username>/crud-backend:latest
docker push <your-dockerhub-username>/crud-frontend:latest
```

\---

## 💻 Local Development (Without Docker)

### Backend

```bash
cd backend
npm install
# Edit app/config/db.config.js → set URL to mongodb://localhost:27017/dd\_db
node server.js
# API runs on: http://localhost:8080
```

### Frontend

```bash
cd frontend
npm install
ng serve --port 8081
# App runs on: http://localhost:8081
```

> For local dev, the frontend calls `http://localhost:8080/api/tutorials` (update `tutorial.service.ts`).  
> For Docker, it uses a relative `/api/tutorials` URL routed by nginx.

\---

## 🔌 API Endpoints

Base URL: `http://localhost/api/tutorials`

|Method|Endpoint|Description|
|-|-|-|
|`GET`|`/api/tutorials`|Get all tutorials|
|`GET`|`/api/tutorials/:id`|Get tutorial by ID|
|`GET`|`/api/tutorials/published`|Get published tutorials|
|`GET`|`/api/tutorials?title=<title>`|Search by title|
|`POST`|`/api/tutorials`|Create new tutorial|
|`PUT`|`/api/tutorials/:id`|Update tutorial|
|`DELETE`|`/api/tutorials/:id`|Delete tutorial|
|`DELETE`|`/api/tutorials`|Delete all tutorials|

### Example: Create a Tutorial

```bash
curl -X POST http://localhost/api/tutorials \\
  -H "Content-Type: application/json" \\
  -d '{"title": "Docker Basics", "description": "Learn Docker"}'
```

\---

## 🧹 Useful Commands

```bash
# View all container status
docker compose ps

# View live logs
docker compose logs -f backend
docker compose logs -f nginx

# Restart a single service
docker compose restart backend

# Stop all containers (data preserved)
docker compose down

# Stop all containers AND delete data
docker compose down -v

# Force full rebuild (no cache)
docker compose build --no-cache
docker compose up -d
```

\---

## ☁️ AWS EC2 Deployment (Manual Docker Setup)

You have an EC2 instance with Docker installed manually. This section covers how to deploy the app on it.

### Step 1: EC2 Instance Setup

#### 1.1 — Security Group Inbound Rules

In **AWS Console → EC2 → Security Groups**, add these inbound rules:

|Type|Protocol|Port|Source|Purpose|
|-|-|-|-|-|
|SSH|TCP|22|Your IP|SSH access|
|HTTP|TCP|80|0.0.0.0/0|App via Nginx|
|Custom TCP|TCP|8080|0.0.0.0/0|Backend (optional debug)|

> \*\*Tip:\*\* For production, only keep port 22 (your IP only) and port 80 (everywhere). Port 8080 is optional.

#### 1.2 — Connect to Your EC2 Instance

```bash
# Download your .pem key from AWS when creating the instance
chmod 400 your-key.pem
ssh -i your-key.pem ubuntu@<EC2\_PUBLIC\_IP>
```

\---

### Step 2: Install Docker on EC2 (Already Done)

Since you've already installed Docker manually, here's what was done for reference:

```bash
# Update packages
sudo apt update \&\& sudo apt upgrade -y

# Add Docker's official GPG key:
sudo apt update
sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release \&\& echo "${UBUNTU\_CODENAME:-$VERSION\_CODENAME}")
Components: stable
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update

# Install Docker Compose plugin
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Start and enable Docker
sudo systemctl start docker
sudo systemctl enable docker

# Allow ubuntu user to run Docker without sudo
sudo usermod -aG docker ubuntu
newgrp docker

# Verify
docker --version
docker compose version
```

\---

### Step 3: Clone the Repository on EC2

```bash
# On your EC2 instance
git clone https://github.com/<your-username>/crud-dd-task-mean-app.git
cd crud-dd-task-mean-app
```

\---

### Step 4: Deploy the App on EC2

```bash
# Start all containers
docker compose up -d --build

# Verify containers are running
docker compose ps

# Check logs
docker compose logs -f
```

### Running Container Screenshot

!\[Running Container](./screenshots/server-running-containers.png)



App will be available at: **http://<EC2\_PUBLIC\_IP>**

\---

### Step 5: CI/CD Auto-Deploy to EC2 via GitHub Actions

The `main.yml` pipeline deploys automatically to your EC2 on every push to `main`.

#### 5.1 — Set GitHub Secrets for EC2

|Secret|Value|
|-|-|
|`EC2\_HOST`|EC2 Public IP (e.g. `13.233.xx.xx`)|
|`EC2\_USER`|`ubuntu` (default for Ubuntu AMI)|
|`EC2\_SSH\_KEY`|Contents of your `.pem` private key file|

```bash
# Copy your .pem file contents
cat your-key.pem
# Paste the ENTIRE content (including -----BEGIN ... END-----) as EC2\_SSH\_KEY
```

#### 5.2 — Make EC2 Ready for Auto-Deploy

```bash
# On EC2: clone once, then CI/CD will git pull + docker compose up on every push
git clone https://github.com/<your-username>/crud-dd-task-mean-app.git \~/crud-dd-task-mean-app
```

After this, every push to `main` will:

1. Build backend + frontend Docker images → push to Docker Hub
2. SSH into EC2 → `git pull` → `docker compose up -d --build`

\---

### EC2 Deployment Architecture

```
Developer
    │
    │  git push origin main
    ▼
GitHub Actions (main.yml)
    │
    ├──→ Build \& push images to Docker Hub
    │
    └──→ SSH into EC2 (appleboy/ssh-action)
              │
              ▼
         EC2 Instance
         ┌──────────────────────────┐
         │  docker compose up -d    │
         │  ┌────────┐ ┌─────────┐  │
         │  │ nginx  │ │ backend │  │
         │  │  :80   │ │  :8080  │  │
         │  └────────┘ └─────────┘  │
         │  ┌──────────┐            │
         │  │ frontend │            │
         │  │  :8081   │            │
         │  └──────────┘            │
         │  ┌──────────┐            │
         │  │ mongodb  │            │
         │  │  :27017  │            │
         │  └──────────┘            │
         └──────────────────────────┘
              Public IP:80 → Internet
```

\---

## 🐛 Troubleshooting

|Problem|Cause|Fix|
|-|-|-|
|`Cannot connect to database`|DB URL wrong|Check `db.config.js` — should be `mongodb://mongodb:27017/dd\_db`|
|`buffering timed out`|Backend started before Mongo ready|Healthcheck should handle it; run `docker compose restart backend`|
|Frontend shows blank page|Build issue|Run `docker compose logs frontend`|
|Port 80 already in use|Another service on port 80|Run `sudo lsof -i :80` to find and stop it|

\---

## 📄 Tech Stack

|Layer|Technology|
|-|-|
|Frontend|Angular 15|
|Backend|Node.js + Express.js|
|Database|MongoDB 6+ (Mongoose ODM)|
|Container Runtime|Docker|
|Orchestration|Docker Compose|
|Reverse Proxy|Nginx|
|CI/CD|GitHub Actions|
|Image Registry|Docker Hub|
