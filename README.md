# TP N°3 : Docker – Multi-container deployment with Docker Compose

## Objective

Deploy a three-tier application (static frontend + Flask API + MySQL database) using **Docker Compose**, then automate the build and publication of the images to **Docker Hub** with a **GitHub Actions** CI pipeline.

## Architecture

```
 Browser ──► Frontend (Nginx, :3000→80) ──fetch──► Backend (Flask, :5000) ──► MySQL 8 (:3307→3306)
              index.html                           /api/users                 db: flaskdb / table: users
```

| Service    | Technology         | Host port | Container port |
|------------|--------------------|-----------|----------------|
| `frontend` | Nginx (alpine)     | 3000      | 80             |
| `backend`  | Python 3.10 + Flask| 5000      | 5000           |
| `db`       | MySQL 8            | 3307      | 3306           |

The browser loads `index.html` from Nginx, then calls `http://localhost:5000/api/users`. Flask queries MySQL (host `db`, resolved by Docker Compose's internal network) and returns the rows as JSON. CORS is enabled with `flask-cors`.

## Project structure

```
dockcom/
├── .github/
│   └── workflows/
│       └── ci.yml            # Part 2 only
├── backend/
│   ├── app.py
│   ├── Dockerfile
│   └── requirements.txt
├── db/
│   └── init.sql
├── frontend/
│   ├── Dockerfile
│   └── index.html
└── docker-compose.yml
```

## Prerequisites

- Docker Desktop (or Docker Engine) with the Compose plugin: `docker compose version`
- A Git repository (GitHub) – required for Part 2
- A Docker Hub account and an access token – required for Part 2

---

## Part 1 – Build the images locally

### Step 1 – Prepare the repository

Create the repository and the folder structure above. **Do not create** `ci.yml` or `.github/workflows/` yet; images are built locally in this part.

**`frontend/index.html`** – static page that fetches `/api/users` and fills an HTML table (ID / Nom). On error it shows the error in red.

**`frontend/Dockerfile`**
```dockerfile
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/index.html
```

**`backend/app.py`** – Flask app exposing `GET /api/users`. It reads `DB_HOST`, `DB_USER`, `DB_PASSWORD`, `DB_NAME` from environment variables, runs `SELECT * FROM users;` and returns JSON. It listens on `0.0.0.0:5000`.

**`backend/Dockerfile`**
```dockerfile
FROM python:3.10-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
CMD ["python", "app.py"]
```

**`backend/requirements.txt`**
```
Flask
flask-cors
mysql-connector-python
```

**`db/init.sql`**
```sql
CREATE DATABASE IF NOT EXISTS flaskdb;
USE flaskdb;
CREATE TABLE IF NOT EXISTS users (
  id INT AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(100),
  email text
);
INSERT INTO users (name) VALUES ('Salim'), ('Nour'), ('Youssef');
```

**`docker-compose.yml`**
```yaml
services:
  backend:
    build: ./backend
    image: votre_username_docker/flask-api:latest
    ports:
      - "5000:5000"
    environment:
      DB_HOST: db
      DB_USER: root
      DB_PASSWORD: root
      DB_NAME: flaskdb
    depends_on:
      - db

  frontend:
    build: ./frontend
    image: votre_username_docker/frontend-app:latest
    ports:
      - "3000:80"
    depends_on:
      - backend

  db:
    image: mysql:8
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: flaskdb
    volumes:
      - ./db/init.sql:/docker-entrypoint-initdb.d/init.sql
    ports:
      - "3307:3306"
```

> Replace `votre_username_docker` with your **Docker Hub username** (needed for Part 2).

### Step 2 – Start the application

```bash
docker compose up --build -d
```

Check that the three containers (frontend, backend, db) are running:

```bash
docker compose ps
```

> MySQL needs a few seconds to initialise on first start. If the page shows an error right away, wait ~20 s and refresh.

### Step 3 – Test the application

1. Open **http://localhost:3000** in your browser.
2. **Frontend ↔ backend:** the table should be populated (no "Erreur" message). You can also test the API directly: http://localhost:5000/api/users
3. **Backend ↔ database:** the data returned comes from MySQL. You can inspect it with:
   ```bash
   docker compose exec db mysql -uroot -proot -e "SELECT * FROM flaskdb.users;"
   ```

**Expected result:** a page titled *Frontend Statique + Flask + MySQL* with a table listing:

| ID | Nom     |
|----|---------|
| 1  | Salim   |
| 2  | Nour    |
| 3  | Youssef |

### Useful commands

```bash
docker compose logs -f backend     # follow backend logs
docker compose down                # stop and remove containers
docker compose down -v             # also remove volumes
docker images                      # list local images
```

---

## Part 2 – Push the images to Docker Hub with GitHub Actions

### Step 1 – Create the GitHub secrets

In your GitHub repository: **Settings → Secrets and variables → Actions → New repository secret**

| Secret              | Value                                   |
|---------------------|-----------------------------------------|
| `DOCKER_USERNAME`   | Your Docker Hub username                |
| `DOCKERHUB_TOKEN`   | A Docker Hub access token (Account Settings → Security → New Access Token, with Read & Write permission) |

### Step 2 – Create `.github/workflows/ci.yml`

The pipeline triggers on every push to `main` and performs: checkout → Docker setup → Docker Hub login → build → push.

```yaml
name: CI - Build and Push Docker Images

on:
  push:
    branches: [ main ]

jobs:
  build-and-push:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build and push backend image
        uses: docker/build-push-action@v6
        with:
          context: ./backend
          push: true
          tags: ${{ secrets.DOCKER_USERNAME }}/flask-api:latest

      - name: Build and push frontend image
        uses: docker/build-push-action@v6
        with:
          context: ./frontend
          push: true
          tags: ${{ secrets.DOCKER_USERNAME }}/frontend-app:latest
```

The image names (`flask-api`, `frontend-app`) must match those in `docker-compose.yml`.

### Step 3 – Push and watch the pipeline

```bash
git add .
git commit -m "Add CI pipeline"
git push origin main
```

Go to the **Actions** tab of your repository and make sure the workflow finishes with a green check.

### Step 4 – Verify the images on Docker Hub

Open https://hub.docker.com → **Repositories**. You should see:

- `<your_username>/flask-api`
- `<your_username>/frontend-app`

### Step 5 – Pull the images and run locally

To make sure you really use the Docker Hub images, remove the local ones first:

```bash
docker compose down
docker rmi <your_username>/flask-api:latest <your_username>/frontend-app:latest
```

Pull and run (without `--build`):

```bash
docker compose pull
docker compose up -d
```

### Step 6 – Verify

- Frontend: http://localhost:3000
- Backend API: http://localhost:5000/api/users
- Database: the table shows Salim, Nour and Youssef.

---

## Troubleshooting

| Problem | Likely cause / fix |
|---------|--------------------|
| Page shows `Erreur : TypeError: Failed to fetch` | Backend not running or not reachable on port 5000. Check `docker compose ps` and `docker compose logs backend`. |
| API returns `{"error": "Can't connect to MySQL server..."}` | MySQL is still initialising. Wait a few seconds and retry. |
| API returns `Unknown database` | The `DB_NAME` must be `flaskdb` (matches `init.sql` and `MYSQL_DATABASE`). Note that `app.py` defaults to `dbflask`, but Compose overrides it. |
| `init.sql` changes are not applied | Init scripts only run on a fresh DB. Run `docker compose down -v` then start again. |
| Port already in use | Change the host port on the left side of the mapping (e.g. `"3308:3306"`). |
| CI fails at the login step | Check that `DOCKER_USERNAME` and `DOCKERHUB_TOKEN` exist and the token has write access. |
| CI push is denied | Image name must start with your Docker Hub username. |

## Notes

- `depends_on` only controls start order, not readiness. This works here because the backend opens a new DB connection on each request.
- Credentials (`root`/`root`) are for teaching only. Never use them in production; use secrets or an `.env` file instead.
- The `email` column is not filled by `init.sql`, so it is `NULL` and not displayed by the frontend.