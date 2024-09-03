
# CI/CD Pipeline for 3-Tier Docker Application

This guide provides step-by-step instructions to set up a CI/CD pipeline using GitHub Actions for a 3-tier application (React frontend, Django REST API backend, and PostgreSQL database) with Docker. The setup uses Traefik for reverse proxy and SSL certificate management on a VPS server.

## Prerequisites

1. **VPS Setup**:
   - Ensure you have a VPS running (e.g., Ubuntu 20.04+).
   - Docker and Docker Compose should be installed on your VPS.
   - Domains should point to your VPS IP: `decorationbd.com` for the frontend and `api.decorationbd.com` for the backend.

2. **SSL Certificates**:
   - Traefik will handle SSL with Let's Encrypt.

3. **GitHub Account**:
   - You should have a GitHub repository set up for your project.

4. **GitHub Secrets**:
   - Store VPS credentials in GitHub Secrets (`SSH_PRIVATE_KEY`, `SSH_USER`, `VPS_HOST`).

## Folder Structure

```bash
project-root/
│
├── frontend/             # React App (Node 18)
├── backend/              # Django REST API
├── docker-compose.yml    # Main docker-compose for services
├── traefik.yml           # Traefik configuration
└── .github/
    └── workflows/
        └── ci-cd.yml     # GitHub Actions workflow for CI/CD
```

## Docker Setup

### Dockerfiles

**Frontend (React App - Node 18)**

Create a `Dockerfile` in the `frontend/` directory:

```Dockerfile
# frontend/Dockerfile

FROM node:18-alpine AS builder

WORKDIR /app

COPY package*.json ./
RUN npm install

COPY . ./
RUN npm run build

FROM nginx:alpine
COPY --from=builder /app/build /usr/share/nginx/html
EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

**Backend (Django REST Framework - Python)**

Create a `Dockerfile` in the `backend/` directory:

```Dockerfile
# backend/Dockerfile

FROM python:3.10

WORKDIR /app

COPY requirements.txt ./
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 8000

CMD ["gunicorn", "--bind", "0.0.0.0:8000", "backend.wsgi:application"]
```

### `docker-compose.yml`

Here is the `docker-compose.yml` file to orchestrate the services:

```yaml
version: '3.8'

services:
  frontend:
    build: ./frontend
    container_name: frontend
    restart: always
    labels:
      - "traefik.http.routers.frontend.rule=Host(`decorationbd.com`)"
      - "traefik.http.routers.frontend.entrypoints=websecure"
      - "traefik.http.routers.frontend.tls=true"
    networks:
      - web
    expose:
      - "80"

  backend:
    build: ./backend
    container_name: backend
    restart: always
    labels:
      - "traefik.http.routers.backend.rule=Host(`api.decorationbd.com`)"
      - "traefik.http.routers.backend.entrypoints=websecure"
      - "traefik.http.routers.backend.tls=true"
    environment:
      - POSTGRES_DB=${POSTGRES_DB}
      - POSTGRES_USER=${POSTGRES_USER}
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
    depends_on:
      - db
    networks:
      - web
    expose:
      - "8000"

  db:
    image: postgres:14
    container_name: db
    restart: always
    environment:
      - POSTGRES_DB=${POSTGRES_DB}
      - POSTGRES_USER=${POSTGRES_USER}
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - web

  traefik:
    image: traefik:v2.9
    container_name: traefik
    command:
      - "--api.insecure=true"
      - "--providers.docker=true"
      - "--entrypoints.websecure.address=:443"
      - "--certificatesresolvers.myresolver.acme.tlschallenge=true"
      - "--certificatesresolvers.myresolver.acme.email=your-email@example.com"
      - "--certificatesresolvers.myresolver.acme.storage=/letsencrypt/acme.json"
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock"
      - "traefik_data:/letsencrypt"
    networks:
      - web

networks:
  web:
    external: false

volumes:
  postgres_data:
  traefik_data:
```

### Traefik Configuration

Create a `traefik.yml` file in the root of your project:

```yaml
api:
  dashboard: true

entryPoints:
  web:
    address: ":80"
  websecure:
    address: ":443"

certificatesResolvers:
  myresolver:
    acme:
      email: your-email@example.com
      storage: /letsencrypt/acme.json
      tlsChallenge: true
```

## GitHub Actions Setup

### Add Secrets to GitHub

Go to your repository's Settings > Secrets and Variables > Actions > New repository secret:

- `SSH_PRIVATE_KEY` – Your private key to access the VPS.
- `SSH_USER` – The SSH user for the VPS.
- `VPS_HOST` – The IP or domain of your VPS.

### GitHub Actions Workflow

Create a workflow file in `.github/workflows/ci-cd.yml`:

```yaml
name: CI/CD Pipeline

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Set up SSH
        uses: webfactory/ssh-agent@v0.8.1
        with:
          ssh-private-key: ${{ secrets.SSH_PRIVATE_KEY }}

      - name: Copy files to server
        run: |
          rsync -avz -e "ssh -o StrictHostKeyChecking=no"           ./ ${{ secrets.SSH_USER }}@${{ secrets.VPS_HOST }}:/path/to/your/project

      - name: Deploy with Docker Compose
        run: |
          ssh -o StrictHostKeyChecking=no ${{ secrets.SSH_USER }}@${{ secrets.VPS_HOST }} << 'EOF'
            cd /path/to/your/project
            docker-compose pull
            docker-compose up -d --build
          EOF
```

## VPS Setup

1. **Clone Your Repository**:  
   SSH into your VPS and clone your GitHub repository into a suitable directory (e.g., `/path/to/your/project`).

2. **Environment Variables**:  
   Create a `.env` file in your project root to store environment variables for the database:

```bash
POSTGRES_DB=your_db_name
POSTGRES_USER=your_db_user
POSTGRES_PASSWORD=your_db_password
```

3. **Run Docker Compose**:  
   Run `docker-compose up -d` to start your services. Traefik will automatically handle SSL with Let's Encrypt.

## Deployment Process

1. **Push Code**:  
   Once you push your code to the `main` branch, GitHub Actions will automatically run the CI/CD pipeline.

2. **GitHub Actions**:  
   The code will be pulled to your VPS, and Docker Compose will build and deploy your containers.

## Testing and Monitoring

- Access your frontend at `https://decorationbd.com`.
- Access your backend at `https://api.decorationbd.com`.
- Monitor your services using the Traefik dashboard (`http://your_vps_ip:8080/dashboard/`).
