forked from https://github.com/makeplane/plane

:)

# Plane Self-Host: Setup Guide & Troubleshooting

A walkthrough of deploying a fork of [Plane](https://github.com/makeplane/plane) for the `app-hack2026` hackathon on macOS, with all the bug fixes encountered along the way.

> **Stack:** macOS, Docker Desktop, Plane fork using Vite + Django + Postgres + Redis + RabbitMQ + MinIO + Celery
> **Repo:** https://github.com/mariia-osipova/hack

---

## TL;DR — full command sequence

```bash
# 1. Clone and enter the directory
cd ~/IdeaProjects/app-hack2026

# 2. Generate .env files
chmod +x setup.sh
./setup.sh

# 3. Apply bug fixes to .env (see "Bug Fixes" section below)
sed -i '' 's|USE_MINIO=0|USE_MINIO=1|' apps/api/.env
sed -i '' 's|WEB_URL="http://localhost:8000"|WEB_URL="http://localhost:3000"|' apps/api/.env
sed -i '' 's|AWS_S3_ENDPOINT_URL="http://localhost:9000"|AWS_S3_ENDPOINT_URL="http://plane-minio:9000"|' apps/api/.env

# 4. Add MinIO host alias to /etc/hosts (so the browser can upload files)
echo "127.0.0.1 plane-minio" | sudo tee -a /etc/hosts

# 5. Start
docker compose -f docker-compose-local.yml up -d

# 6. Open in browser
open http://localhost:3000
```

---

## What is Plane

**Plane** is an open-source alternative to Jira / Linear / Monday / ClickUp. Apache-2.0, 36K+ GitHub stars.

- Projects, cycles (sprints), modules (epics)
- Kanban / List / Gantt / Calendar views
- API + integrations with GitHub, GitLab, Slack
- Self-hostable via Docker

---

## Prerequisites

| Tool | Why |
|---|---|
| **Docker Desktop** | Containers |
| **Node.js + pnpm** | Only needed for dev mode (editing frontend code) |
| **PostgreSQL not on 5432** | Otherwise port conflicts with Plane's container |

Install Node + pnpm:
```bash
brew install node
corepack enable
corepack prepare pnpm@latest --activate
```

---

## Fork structure

In this fork:
- `docker-compose-local.yml` — **dev environment**, only spins up the backend (DB, API, workers). Frontend is built separately via pnpm.
- `docker-compose.yml` — production mode with prebuilt images + nginx
- `apps/api/` — Django backend
- `apps/web/`, `apps/admin/`, `apps/space/`, `apps/live/` — Vite frontends
- `setup.sh` — generates `.env` files

---

## Bug fixes encountered

The deployment surfaced a bunch of bugs specific to this fork. Full list below with fixes.

### 🐛 Bug 1: setup.sh requires pnpm for dev mode

**Symptom:**
```
./setup.sh: line 81: corepack: command not found
./setup.sh: line 83: pnpm: command not found
```

**Fix:** install Node.js + pnpm:
```bash
brew install node
corepack enable
corepack prepare pnpm@latest --activate
```

### 🐛 Bug 2: docker-credential-desktop not found

**Symptom:**
```
error getting credentials - err: exec: "docker-credential-desktop": executable file not found in $PATH
```

**Fix:** remove `credsStore` from Docker config:
```bash
nano ~/.docker/config.json
```

Delete the `"credsStore": "desktop",` line. Should be valid JSON, e.g.:
```json
{
  "auths": {}
}
```

### 🐛 Bug 3: docker-compose-local.yml has duplicated paths

**Symptom:**
```
resolve : lstat /Users/.../apps/api/apps: no such file or directory
```

**Cause:** in the compose file, `context: apps/api` and `dockerfile: apps/api/Dockerfile.dev` — Docker uses `apps/api` as context and then looks for `apps/api/...` inside it.

**Fix:**
```bash
sed -i '' 's|dockerfile: apps/api/Dockerfile.dev|dockerfile: Dockerfile.dev|g' docker-compose-local.yml
```

### 🐛 Bug 4: Port 5432 conflict with local PostgreSQL

**Symptom:**
```
Ports are not available: exposing port TCP 0.0.0.0:5432 -> ... bind: address already in use
```

**Cause:** PostgreSQL installed locally (e.g. via postgresql.org installer):
```bash
sudo lsof -iTCP:5432 -sTCP:LISTEN -n -P
# postgres 565 postgres ... TCP *:5432 (LISTEN)
ps -p 565 -o command=
# /Library/PostgreSQL/17/bin/postgres ...
```

**Fix:** don't touch the local Postgres, change Plane's external port instead:
```bash
sed -i '' 's|- 5432:5432|- 5433:5432|g' docker-compose-local.yml
```

### 🐛 Bug 5: WEB_URL points to API instead of frontend

**Symptom:** "🚧 Looks like Plane didn't start up correctly!" page.

**Cause:** `apps/api/.env` has `WEB_URL="http://localhost:8000"` (API port instead of frontend).

**Fix:**
```bash
sed -i '' 's|WEB_URL="http://localhost:8000"|WEB_URL="http://localhost:3000"|' apps/api/.env
docker compose -f docker-compose-local.yml restart api
```

### 🐛 Bug 6: API can't reach MinIO

**Symptom** in API logs:
```
Could not connect to the endpoint URL: "http://localhost:9000/uploads"
```

**Cause:** `AWS_S3_ENDPOINT_URL=http://localhost:9000` — inside Docker, `localhost` is the container itself, not MinIO.

**Fix:**
```bash
sed -i '' 's|AWS_S3_ENDPOINT_URL="http://localhost:9000"|AWS_S3_ENDPOINT_URL="http://plane-minio:9000"|' apps/api/.env
```

### 🐛 Bug 7: USE_MINIO=0 breaks presigned URLs

**Symptom:** "Failed to upload cover image" when creating a project.

**Cause:** Plane generates AWS-style URLs instead of MinIO-style.

**Fix:**
```bash
sed -i '' 's|USE_MINIO=0|USE_MINIO=1|' apps/api/.env
docker compose -f docker-compose-local.yml restart api worker beat-worker
```

### 🐛 Bug 8: DNS resolution fails after restart

**Symptom** in API logs:
```
django.db.utils.OperationalError: failed to resolve host 'plane-db': [Errno -2] Name does not resolve
```

**Cause:** Docker Desktop sometimes loses network aliases when restarting individual services.

**Fix:** full `down`/`up` cycle, optionally pruning networks:
```bash
docker compose -f docker-compose-local.yml down
docker network prune -f
docker compose -f docker-compose-local.yml up -d
```

⚠️ Do **NOT** use `down -v` — that wipes the DB volumes.

### 🐛 Bug 9: Browser can't upload files to MinIO

**Symptom:** "Failed to upload cover image" even after `USE_MINIO=1`.

**Cause:** the API returns presigned URLs like `http://plane-minio:9000/...`. The hostname `plane-minio` only exists inside the Docker network — the browser can't resolve it.

**Fix:** add a host alias in `/etc/hosts`:
```bash
echo "127.0.0.1 plane-minio" | sudo tee -a /etc/hosts
```

Now when the browser sees a URL like `http://plane-minio:9000/...`, it resolves it to `127.0.0.1:9000` — where MinIO is actually exposed from Docker.

**Revert:**
```bash
sudo sed -i '' '/plane-minio/d' /etc/hosts
```

---

## Port architecture

After all fixes:

| Service | Internal port | External URL |
|---|---|---|
| Web (main app) | 3000 | http://localhost:3000 |
| Admin (god-mode) | 3001 | http://localhost:3001/god-mode/ |
| Space (public boards) | 3002 | http://localhost:3002 |
| Live (real-time) | 3100 | http://localhost:3100 |
| API (Django) | 8000 | http://localhost:8000 |
| MinIO Console | 9000 | http://localhost:9000 |
| Postgres | 5432 (5433 if conflict) | — |
| Redis | 6379 | — |
| RabbitMQ | 5672, 15672 (UI) | http://localhost:15672 |

---

## Useful commands

### Container management
```bash
# Start in background
docker compose -f docker-compose-local.yml up -d

# Stop (volumes preserved)
docker compose -f docker-compose-local.yml down

# ⚠️ Full DB reset (deletes all data)
docker compose -f docker-compose-local.yml down -v

# Logs from all services
docker compose -f docker-compose-local.yml logs -f

# Logs from one service
docker logs -f app-hack2026-api-1

# Restart one service
docker compose -f docker-compose-local.yml restart api
```

### Diagnostics
```bash
# What containers are running
docker ps

# Who is using a port?
sudo lsof -iTCP:5432 -sTCP:LISTEN -n -P

# All API env vars without secrets
grep -v "^#" apps/api/.env | grep -v "^$" | \
  sed 's/PASSWORD=.*/PASSWORD=***/; s/SECRET=.*/SECRET=***/; s/KEY=.*/KEY=***/'
```

### Docker cleanup
```bash
# Remove stopped containers
docker container prune

# Remove all unused resources (containers, images, networks)
docker system prune -a

# Full cleanup including volumes (⚠️ wipes DB too)
docker system prune -a --volumes
```

---

## First-time launch

Once everything starts without errors:

1. Open **http://localhost:3000** — you'll see "Welcome to Plane"
2. Click **Get started** — redirects to god-mode (port 3001)
3. Open **http://localhost:3001/god-mode/** (important — with the trailing slash)
4. Fill in the instance setup form:
    - Email
    - First/Last name
    - Password
5. After registration, you land in the instance admin (god-mode)
6. Go back to **http://localhost:3000** and register as a regular user
7. Create a **workspace** → **project** → start working

---

## Git workflow

### Initialization (if repo isn't connected yet)
```bash
git init
git config user.name "Your Name"
git config user.email "you@example.com"
```

### GitHub authentication
```bash
brew install gh
gh auth login
# GitHub.com → HTTPS → Yes → Login with web browser
```

### Connect remote and push
```bash
git remote add origin https://github.com/USER/REPO.git
git add .
git status | grep "\.env"   # ⚠️ should be empty (.env in .gitignore)
git commit -m "Initial commit"
git branch -M main
git push -u origin main
```

### Protecting against secret leaks

`.gitignore` must include:
```
.env
apps/*/.env
```

If `.env` is already tracked — remove from index:
```bash
git rm --cached .env apps/*/.env
git commit -m "Remove env files from tracking"
```

---

## Deploying for the team

### Option 1: Local + tunnel (fastest, free)

**Tailscale Funnel** — best option if you don't have a credit card:
```bash
brew install --cask tailscale
# Open Tailscale.app → log in via GitHub
# In admin panel https://login.tailscale.com/admin/dns enable:
# - MagicDNS
# - HTTPS Certificates
# - Funnel

sudo tailscale funnel 3000
# You get a URL: https://your-mac.tail-XXXX.ts.net
```

**ngrok** — alternative:
```bash
brew install ngrok
# Sign up at ngrok.com
ngrok config add-authtoken YOUR_TOKEN
ngrok http 3000
```

⚠️ Requires the Mac to stay powered on.

### Option 2: GitHub Codespaces (60-120 free hours/month)

In your repo: **Code → Codespaces → Create codespace on main**

⚠️ Don't enable **prebuilds** — they're billed from the first hour.

In Codespaces you'll need to edit all `.env` files to use URLs like `https://<codespace>-PORT.app.github.dev`. Lots of fiddling.

### Option 3: VPS (most reliable)

- **Hetzner CX22** — €4.5/month, 4 GB RAM, AMD
- **Oracle Cloud Free Tier** — free forever (4 ARM cores + 24 GB RAM), but requires a credit card to register
- **DigitalOcean Droplet** — $6-12/month

On a VPS use `docker-compose.yml` (production), not `-local`.

---

## What does NOT work for Plane

- **Vercel / Netlify** — won't work (you need Postgres + Redis + RabbitMQ + MinIO + Celery)
- **Heroku free** — no longer has a free tier
- **Render free / Fly.io free** — too little RAM, Plane won't run reliably

---

## License

Plane: **Apache-2.0**. You can fork, modify, and use commercially. Keep `LICENSE.txt` and copyright notices intact.

---

## Useful links

- Original Plane: https://github.com/makeplane/plane
- Documentation: https://developers.plane.so/
- Self-host guide: https://developers.plane.so/self-hosting/overview
- Docker Compose docs: https://docs.docker.com/compose/

---

_Document compiled from real-world experience deploying the `app-hack2026` fork._