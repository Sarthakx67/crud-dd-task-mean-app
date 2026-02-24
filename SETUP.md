# crud-dd-task-mean-app – Setup & Start Guide

This file summarizes everything you need to build, run, and deploy the app.

## 1. Local Development (without Docker)

### Backend (Node + MongoDB)

1. Prerequisites:
   - Node.js 18+
   - Local MongoDB running on `mongodb://localhost:27017`

2. Install and run backend:

   ```bash
   cd backend
   npm install
   node server.js
   ```

   Backend runs on `http://localhost:8080` and connects to DB `dd_db`.

### Frontend (Angular)

1. Prerequisites:
   - Node.js 18+
   - Angular CLI (optional for dev): `npm install -g @angular/cli`

2. Install and run frontend (dev mode):

   ```bash
   cd frontend
   npm install
   npm run start
   ```

   Angular dev server runs on `http://localhost:4200` and calls the backend API according to `src/app/services/tutorial.service.ts`.

---

## 2. Docker & docker-compose (single host)

Docker setup is defined in:
- `backend/Dockerfile`
- `frontend/Dockerfile`
- `docker-compose.yml`

### Build & Run with docker-compose

From project root:

```bash
cd crud-dd-task-mean-app   # if not already

# Build images and start containers
docker compose up --build -d

# or (older Compose v1)
# docker-compose up --build -d
```

Services:
- `mongo`  : MongoDB 5, exposed on `27017` (volume `mongo_data`)
- `backend`: Node/Express API, exposed on `8080`
- `frontend`: Nginx + Angular, exposed on `80`

Application entrypoint: `http://<host-ip>` (port 80).

---

## 3. Nginx Reverse Proxy (inside frontend container)

Frontend nginx is configured in:
- `frontend/nginx.conf`
- `frontend/Dockerfile`

Behavior:
- Serves Angular SPA from `/usr/share/nginx/html`.
- SPA fallback: all unknown routes → `index.html`.
- Proxies API calls:
  - `location /api/` → `http://backend:8080/` (Docker service name `backend`).

Angular calls a relative API URL:
- `frontend/src/app/services/tutorial.service.ts`:
  - `const baseUrl = '/api/tutorials';`

So browser only talks to port 80; nginx forwards `/api/...` to backend container on 8080.

---

## 4. Environment & DB Configuration

Backend DB config:
- `backend/app/config/db.config.js`:

  ```js
  module.exports = {
    url: process.env.DB_URL || "mongodb://localhost:27017/dd_db"
  };
  ```

- In Docker (`docker-compose.yml`):
  - Backend service has `DB_URL=mongodb://mongo:27017/dd_db`.

Mongo container:
- Service `mongo` uses image `mongo:5`.
- Initializes database `dd_db`.
- Persists data in named volume `mongo_data`.

---

## 5. CI/CD with Jenkins + Docker Hub

CI/CD pipeline is defined in:
- `Jenkinsfile`

### What the Jenkins pipeline does

1. **Checkout**: pulls latest code from GitHub.
2. **Build images**:
   - `docker build -t <DOCKERHUB_USER>/crud-dd-task-backend:latest backend`
   - `docker build -t <DOCKERHUB_USER>/crud-dd-task-frontend:latest frontend`
3. **Push to Docker Hub**:
   - Logs in with Jenkins credentials `dockerhub-creds`.
   - Pushes both images.
4. **Deploy to VM** (via SSH):
   - SSH to EC2 (or any VM) and run:
     - `git pull`
     - `docker compose pull`
     - `docker compose up -d`

### Variables to customize (Jenkinsfile)

In `Jenkinsfile` edit:

- `DOCKERHUB_USER = 'your-dockerhub-username'`
- `DEPLOY_HOST = 'ec2-user@your-ec2-public-ip'`
- `DEPLOY_PATH = '/home/ec2-user/crud-dd-task-mean-app'`

Also in Jenkins UI:
- Create Docker Hub credentials with ID `dockerhub-creds` (Username + Password).
- Make sure Jenkins agent has Docker installed and SSH access to the VM.

---

## 6. VM / EC2 Deployment Steps (One-Time Setup)

On the target VM (e.g., Amazon EC2):

1. Install Docker & docker-compose / Docker Compose v2.
2. Clone the repo to match `DEPLOY_PATH` in Jenkinsfile:

   ```bash
   cd /home/ec2-user
   git clone https://github.com/<your-username>/crud-dd-task-mean-app.git
   cd crud-dd-task-mean-app
   ```

3. Test manual deployment once:

   ```bash
   docker compose up --build -d
   # or: docker-compose up --build -d
   ```

4. Open in browser: `http://<vm-public-ip>`.

If this works, Jenkins can automate future builds and deployments.

---

## 7. Common Checks / Troubleshooting

- **Backend health**:
  - `docker logs crud-dd-task-backend` should show `Server is running on port 8080` and `Connected to the database!`.
- **Mongo health**:
  - Ensure `crud-dd-task-mongo` container is running.
- **Frontend / nginx**:
  - `docker logs crud-dd-task-frontend` should show successful startup.
  - 404s at `/api/...` usually mean nginx is not proxying or backend is down.
- **Ports & Security Groups**:
  - Ensure VM firewall / security group allows TCP 80 (and 8080 if debugging backend directly).

This file should be enough to set up, run, and redeploy the project locally and on a VM with Jenkins. Update hostnames, IPs, and usernames as needed for your environment.
