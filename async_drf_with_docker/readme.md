# Betopian Server - A-Z VPS Deployment Guide (Docker)

**Server IP:** `103.149.105.4`
**Project path:** `/root/betopian-server`
**OS:** Ubuntu | **Web Server:** Nginx | **App Server:** Daphne (in Docker) | **DB:** PostgreSQL 16 (in Docker) | **Cache:** Redis (in Docker)

---

## Table of Contents
1. [Project Structure](#1-project-structure)
2. [Environment Setup (.env)](#2-environment-setup-env)
3. [Docker Setup (Dockerfile & docker-compose.yml)](#3-docker-setup-dockerfile--docker-composeyml)
4. [Database Setup](#4-database-setup)
5. [Build & Run](#5-build--run)
6. [Nginx Configuration](#6-nginx-configuration)
7. [Point Domain DNS (when ready)](#7-point-domain-dns-when-ready)
8. [SSL with Certbot (when ready)](#8-ssl-with-certbot-when-ready)
9. [Database Backups](#9-database-backups)
10. [Making Changes & Restarting Services](#10-making-changes--restarting-services)
11. [Final Checks](#11-final-checks)
12. [Quick Reference](#12-quick-reference)

---

## 1. Project Structure

Everything lives under `/root/betopian-server/`:

```
/root/betopian-server/
├── Dockerfile              # Image build instructions
├── docker-compose.yml      # Defines db, redis, web services
├── .env                     # Environment variables (secrets, DB config)
├── manage.py
├── requirements.txt
├── apps/                    # Django apps
├── config/                  # settings.py, urls.py, asgi.py, wsgi.py
├── templates/
├── static/                  # Source static files (in code)
├── staticfiles/             # Collected static files (bind-mounted, served by Nginx)
├── media/                   # User-uploaded media (bind-mounted, served by Nginx)
└── api_documentation.md
```

To find/edit any file, SSH in and `cd /root/betopian-server`, then navigate normally with `nano` or `cat`. Code changes are made directly to files under `apps/`, `config/`, `templates/`, etc.

---

## 2. Environment Setup (.env)

Located at `/root/betopian-server/.env`. Contains all secrets and environment-specific config — **never commit this file to git**.

```bash
nano /root/betopian-server/.env
```

Key variables:

```env
DEBUG=False
SECRET_KEY=your-production-secret-key
ALLOWED_HOSTS=103.149.105.4

# Database (Docker service name, NOT localhost)
DB_NAME=betopian
DB_USER=postgres
DB_PASSWORD=betopia@123
DB_HOST=db
DB_PORT=5432

# Redis (Docker service name)
REDIS_URL=redis://redis:6379/0

CSRF_TRUSTED_ORIGINS=http://103.149.105.4
```

> Once a domain + SSL is set up (Steps 7-8), update `ALLOWED_HOSTS` and `CSRF_TRUSTED_ORIGINS` to use `https://yourdomain.com` and restart the `web` container (Section 10).

---

## 3. Docker Setup (Dockerfile & docker-compose.yml)

### `Dockerfile`

```dockerfile
FROM python:3.12-slim

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

WORKDIR /app

RUN apt-get update && apt-get install -y \
    libpq-dev \
    gcc \
    && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

RUN python manage.py collectstatic --noinput

EXPOSE 8000

CMD ["daphne", "-b", "0.0.0.0", "-p", "8000", "config.asgi:application"]
```

### `docker-compose.yml`

```yaml
services:

  db:
    image: postgres:16-alpine
    restart: unless-stopped
    environment:
      POSTGRES_DB: betopian
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: betopia@123
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    restart: unless-stopped
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  web:
    build: .
    restart: unless-stopped
    ports:
      - "127.0.0.1:8000:8000"
    env_file:
      - .env
    volumes:
      - ./media:/app/media
      - ./staticfiles:/app/staticfiles
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    command: >
      sh -c "python manage.py migrate &&
             python manage.py collectstatic --noinput &&
             daphne -b 0.0.0.0 -p 8000 config.asgi:application"

volumes:
  postgres_data:
  redis_data:
```

**Key points:**
- `db` and `redis` have **no exposed ports** — only reachable from `web` via Docker's internal network (hostnames `db` and `redis`).
- `web` is bound to `127.0.0.1:8000` — only Nginx (on the same machine) can reach it, not the public internet.
- `./media` and `./staticfiles` are **bind mounts** (plain folders on the VPS), so Nginx can read them directly without permission issues.

---

## 4. Database Setup

PostgreSQL runs **inside Docker** — no separate installation needed. The `db` service in `docker-compose.yml` automatically:
- Creates the `betopian` database
- Creates the `postgres` user with the password from `docker-compose.yml`/`.env`
- Persists data in the `postgres_data` Docker volume (survives container restarts/rebuilds)

### Migrating data from a previous deployment (one-time, already done)

If you ever need to repeat this for another project:

```bash
# 1. Dump old DB (native postgres example)
sudo -u postgres pg_dump --no-owner --no-privileges old_db > ~/backups/old_db.sql

# 2. Stop web
docker compose stop web

# 3. Reset target DB
docker exec -it betopian-server-db-1 psql -U postgres -c "DROP DATABASE betopian;"
docker exec -it betopian-server-db-1 psql -U postgres -c "CREATE DATABASE betopian;"

# 4. Restore
cat ~/backups/old_db.sql | docker exec -i betopian-server-db-1 psql -U postgres -d betopian

# 5. Restart and migrate
docker compose up -d
docker compose exec web python manage.py migrate
```

### Accessing the database directly

```bash
docker exec -it betopian-server-db-1 psql -U postgres -d betopian
```

---

## 5. Build & Run

```bash
cd /root/betopian-server
docker compose up -d --build
```

This will:
1. Build the `web` image from `Dockerfile`
2. Start `db` and `redis`, wait until healthy
3. Start `web`, which runs `migrate` → `collectstatic` → starts Daphne on port 8000

Check status:

```bash
docker compose ps
```

All three should show `Up` / `healthy`. View logs:

```bash
docker compose logs -f web
```

Test locally:

```bash
curl http://127.0.0.1:8000/
```

---

## 6. Nginx Configuration

```bash
nano /etc/nginx/sites-available/betopian-server
```

Current config (using IP, no SSL yet):

```nginx
server {
    listen 80;
    server_name 103.149.105.4;

    # ACME challenge for Certbot
    location /.well-known/acme-challenge/ {
        root /var/www/html;
    }

    # Static files
    location /static/ {
        alias /root/betopian-server/staticfiles/;
        expires 30d;
        add_header Cache-Control "public, immutable";
    }

    # Media files
    location /media/ {
        alias /root/betopian-server/media/;
        expires 7d;
    }

    # Proxy to Daphne (ASGI) running in Docker
    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_read_timeout 120;
        proxy_connect_timeout 120;
        proxy_send_timeout 120;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # WebSocket support
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }

    access_log /var/log/nginx/betopian_server_access.log;
    error_log  /var/log/nginx/betopian_server_error.log;
    client_max_body_size 100M;
}
```

Enable + reload:

```bash
ln -s /etc/nginx/sites-available/betopian-server /etc/nginx/sites-enabled/   # one-time, already done
nginx -t && systemctl reload nginx
```

---

## 7. Point Domain DNS (when ready)

> SSL (Step 8) requires a domain — Certbot won't issue certs for a bare IP.

### Check VPS IP

```bash
curl ifconfig.me
```

### Update DNS A Records

At your DNS provider, add:

```
Type:  A
Name:  yourdomain (or subdomain)
Value: 103.149.105.4
TTL:   300
```

### Verify propagation

```bash
dig yourdomain.com +short
# Must return 103.149.105.4
```

### Update Nginx `server_name`

```bash
nano /etc/nginx/sites-available/betopian-server
```

Change:
```nginx
server_name 103.149.105.4;
```
to:
```nginx
server_name yourdomain.com www.yourdomain.com;
```

```bash
nginx -t && systemctl reload nginx
```

### Update `.env`

```env
ALLOWED_HOSTS=yourdomain.com,www.yourdomain.com
CSRF_TRUSTED_ORIGINS=https://yourdomain.com,https://www.yourdomain.com
```

```bash
docker compose restart web
```

---

## 8. SSL with Certbot (when ready)

```bash
certbot --nginx -d yourdomain.com -d www.yourdomain.com
```

When prompted:
- Enter your email
- Agree to terms - `A`
- Choose **redirect HTTP → HTTPS** (recommended)

Certbot automatically edits the Nginx config to add the `listen 443 ssl` block, certificate paths, and the redirect.

Test auto-renewal:

```bash
certbot renew --dry-run
```

---

## 9. Database Backups

Daily automatic backup via cron:

```bash
mkdir -p ~/backups
crontab -e
```

Add:

```
0 2 * * * docker compose -f /root/betopian-server/docker-compose.yml exec -T db pg_dump -U postgres betopian > /root/backups/db_$(date +\%F).sql
```

Test manually:

```bash
cd /root/betopian-server
docker compose exec -T db pg_dump -U postgres betopian > ~/backups/db_test.sql
head -5 ~/backups/db_test.sql
```

---

## 10. Making Changes & Restarting Services

This is the most common day-to-day section — refer here whenever you change something.

### A. Changed Python code (views, models, serializers, etc.) under `apps/` or `config/`

```bash
cd /root/betopian-server
docker compose up -d --build web
```

Rebuilds the image with new code, then restarts the container. Migrations run automatically if there are new ones.

### B. Added/changed a model → new migration files

```bash
docker compose exec web python manage.py makemigrations
docker compose up -d --build web
```

(`migrate` runs automatically on container start.)

### C. Changed `requirements.txt` (new package)

```bash
docker compose up -d --build web
```

Rebuild is required so the new package gets installed in the image.

### D. Changed static files (CSS/JS/images in `static/`) or templates

```bash
docker compose restart web
```

This re-runs `collectstatic`, which copies updated files into `staticfiles/` (the bind-mounted folder Nginx serves from). No Nginx restart needed — Nginx reads files from disk on every request.

Then hard-refresh the browser: `Ctrl+Shift+R`.

### E. Changed `.env` (secrets, ALLOWED_HOSTS, etc.)

```bash
docker compose restart web
```

`docker compose restart` re-reads `env_file` values.

### F. Changed Nginx config

```bash
nginx -t && systemctl reload nginx
```

`nginx -t` checks syntax first — fix any errors before reloading.

### G. Database/Redis acting up

```bash
docker compose restart db
docker compose restart redis
```

### H. Full restart of everything (safe — data persists in volumes/bind mounts)

```bash
docker compose down
docker compose up -d
```

> ⚠️ Never run `docker compose down -v` — the `-v` flag deletes the `postgres_data` and `redis_data` volumes (your database!).

---

## 11. Final Checks

```bash
# Containers running?
docker compose ps

# Nginx running?
systemctl status nginx

# Hit the site
curl -I http://103.149.105.4/
curl -I http://103.149.105.4/admin/
curl -I http://103.149.105.4/static/admin/css/base.css   # should be 200 OK
```

Visit `http://103.149.105.4/admin/` (or your domain after SSL) — login page should be fully styled.

---

## 12. Quick Reference

| Task | Command |
|---|---|
| Go to project folder | `cd /root/betopian-server` |
| View running containers | `docker compose ps` |
| View live logs (web) | `docker compose logs -f web` |
| View live logs (db) | `docker compose logs -f db` |
| Restart after code change | `docker compose up -d --build web` |
| Restart after static/template/.env change | `docker compose restart web` |
| Open shell inside web container | `docker compose exec web sh` |
| Run management command | `docker compose exec web python manage.py <cmd>` |
| Access database shell | `docker exec -it betopian-server-db-1 psql -U postgres -d betopian` |
| Reload Nginx after config edit | `nginx -t && systemctl reload nginx` |
| Manual DB backup | `docker compose exec -T db pg_dump -U postgres betopian > ~/backups/db_manual.sql` |
| Renew SSL | `certbot renew` |
| Check VPS IP | `curl ifconfig.me` |
| Check DNS | `dig yourdomain.com +short` |
| Full safe restart (keeps data) | `docker compose down && docker compose up -d` |
| ⚠️ NEVER run (deletes DB data) | `docker compose down -v` |