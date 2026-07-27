# VPS + Docker + GitHub Actions CI/CD — Setup Guide

A complete reference for setting up a new Node.js/Express + Docker project with automated CI/CD deployment to a VPS.

---

## PART 1: One-Time VPS Setup

### 1.1 Add your user to the docker group
This avoids `permission denied` errors when running Docker commands without `sudo`.

```bash
sudo usermod -aG docker $USER
newgrp docker
groups
```
`newgrp docker` applies the group change immediately in your current shell (otherwise you'd need to log out and back in). `groups` just confirms `docker` now appears in your user's group list.

### 1.2 Increase swap memory
Important for low-RAM VPS instances (e.g. 1GB RAM) — prevents the server from crashing during memory-heavy operations like builds.

```bash
sudo fallocate -l 2G /swapfile2
sudo chmod 600 /swapfile2
sudo mkswap /swapfile2
sudo swapon /swapfile2
free -h
```
This creates a 2GB swap file, locks its permissions to root-only (`600`), formats it as swap space, activates it, then `free -h` shows you the updated memory/swap totals to confirm it worked.

### 1.3 Clone your repository (first time only)
```bash
cd /var/www
git clone git@github.com:<your-username>/<your-repo>.git
cd <your-repo>
```
Replace `<your-username>/<your-repo>` with your actual GitHub path. This assumes your SSH key is already added to your GitHub account for cloning via SSH.

### 1.4 Create the production `.env` file
```bash
nano .env
```
Paste your production environment variables here (DB connection strings, API keys, secrets, etc). Save with `Ctrl+O`, exit with `Ctrl+X`. **Never commit this file to git.**

---

## PART 2: Generate SSH Key for GitHub Actions → VPS Deployment

### 2.1 Generate a new dedicated key pair
Don't reuse your personal GitHub SSH key — this one is specifically for GitHub Actions to log into your VPS.

```bash
ssh-keygen -t ed25519 -C "github-actions-deploy" -f ~/.ssh/github_actions_key -N ""
```
`-t ed25519` picks a modern key type, `-f` sets the output filename, `-N ""` means no passphrase (required since GitHub Actions runs non-interactively).

### 2.2 Authorize this key to log into the VPS
```bash
cat ~/.ssh/github_actions_key.pub >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```
This appends the public key to the list of keys allowed to SSH into this VPS user account.

### 2.3 Copy the private key for GitHub Secrets
```bash
cat ~/.ssh/github_actions_key
```
Copy the entire output, including the `-----BEGIN OPENSSH PRIVATE KEY-----` and `-----END-----` lines. You'll paste this into a GitHub Secret in Part 4.

---

## PART 3: GitHub Personal Access Token (for GHCR login)

### 3.1 Create the token (in browser)
Go to `https://github.com/settings/tokens` → **Generate new token (classic)** → name it something like `<project>-vps-ghcr` → check scopes `read:packages` (and `write:packages` if needed) → **Generate token** → copy it immediately (shown only once).

### 3.2 Log in on the VPS using that token
```bash
docker login ghcr.io -u <your-github-username>
```
When prompted for a password, paste the `ghp_xxxxxxxxxxxx` token — not your actual GitHub password.

---

## PART 4: GitHub Repository Secrets

Go to: **Repo → Settings → Secrets and variables → Actions → New repository secret**, and add:

| Secret Name | Value |
|---|---|
| `VPS_HOST` | Your VPS IP address |
| `VPS_USER` | Your VPS SSH username |
| `VPS_SSH_KEY` | The private key from step 2.3 (full content) |

> `GITHUB_TOKEN` is automatic — don't add it manually.

### 4.1 Enable write permission for GITHUB_TOKEN
Needed so the workflow can push images to GitHub Container Registry (GHCR):
**Repo → Settings → Actions → General → Workflow permissions → select "Read and write permissions" → Save**

---

## PART 5: GitHub Actions Workflow File

Save at: `.github/workflows/docker-build.yml`

```yaml
name: Build and Push Docker Image

on:
  push:
    branches:
      - main

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Set lowercase repo owner
        run: echo "OWNER_LC=$(echo '${{ github.repository_owner }}' | tr '[:upper:]' '[:lower:]')" >> $GITHUB_ENV

      - name: Build and push image
        uses: docker/build-push-action@v6
        with:
          context: .
          file: ./Dockerfile
          push: true
          tags: ghcr.io/${{ env.OWNER_LC }}/<your-repo>:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Deploy to VPS
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.VPS_HOST }}
          username: ${{ secrets.VPS_USER }}
          key: ${{ secrets.VPS_SSH_KEY }}
          script: |
            cd /var/www/<your-repo>
            docker login ghcr.io -u ${{ github.actor }} -p ${{ secrets.GITHUB_TOKEN }}
            docker compose pull app
            docker compose up -d
            docker image prune -f
```

**What this does step by step:**
- Triggers on every push to `main`
- Checks out the code, sets up Docker Buildx (needed for advanced build caching)
- Logs into GHCR using the automatic `GITHUB_TOKEN`
- Converts the repo owner name to lowercase (Docker image tags must be lowercase)
- Builds the Docker image and pushes it to GHCR, using GitHub Actions cache to speed up future builds
- SSHs into your VPS using the secrets from Part 4, pulls the new image, restarts containers with `docker compose up -d`, and cleans up old unused images

Replace `<your-repo>` in both the `tags` line and the `cd` path with your actual repository name.

---

## PART 6: docker-compose.yml (VPS side — uses prebuilt image, no local build)

```yaml
services:
  app:
    image: ghcr.io/<your-username>/<your-repo>:latest
    container_name: <project>-app
    restart: unless-stopped
    ports:
      - '${PORT:-5000}:${PORT:-5000}'
    env_file:
      - .env
    environment:
      NODE_ENV: production
      REDIS_HOST: redis
      REDIS_PORT: 6379
    deploy:
      resources:
        limits:
          cpus: '0.25'
          memory: 256M
    depends_on:
      redis:
        condition: service_healthy
    networks:
      - app-network

  redis:
    image: redis:7-alpine
    container_name: <project>-redis
    restart: unless-stopped
    command: redis-server --maxmemory 64mb --maxmemory-policy allkeys-lru
    deploy:
      resources:
        limits:
          cpus: '0.10'
          memory: 64M
    healthcheck:
      test: ['CMD', 'redis-cli', 'ping']
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - app-network

networks:
  app-network:
    driver: bridge
```

**Notes:**
- The `app` service pulls the image built in Part 5 — the VPS never builds anything itself, which avoids out-of-memory errors on small VPS instances.
- `env_file: .env` loads your production secrets without hardcoding them into the compose file.
- Resource `limits` prevent one container from eating all VPS RAM/CPU.
- If your project uses a background worker (BullMQ, etc), add a second service identical to `app` but pointing at a worker entrypoint/command, similar to the original README's `worker` service.
- Add a `mongodb` or `postgres` service here too if you're self-hosting the DB instead of using Atlas/managed DB.

---

## PART 7: Local Machine — Pushing Code

```bash
git add .
git commit -m "your commit message"
git push origin main
```
This is all you do locally. The push to `main` automatically triggers the GitHub Actions workflow: build → push to GHCR → deploy to VPS. No manual VPS commands are needed unless something fails.

---

## PART 8: VPS — Daily Operation Commands

**Pull latest code** (only needed if `docker-compose.yml` or `Dockerfile` changed):
```bash
git pull origin main
```

**Pull latest built image and restart containers:**
```bash
docker compose pull
docker compose up -d
```

**Check container status:**
```bash
docker compose ps
```

**Check server health** (adjust port/path to match your app):
```bash
curl http://localhost:5000/health
```

**View live logs:**
```bash
docker compose logs -f app
```

**Check resource usage (RAM/CPU per container):**
```bash
docker stats
```

**Check VPS memory/swap:**
```bash
free -h
```

**Stop everything:**
```bash
docker compose down
```

**Clean up unused images/cache** (frees disk space):
```bash
docker system prune -f
```

---

## PART 9: Troubleshooting Quick Reference

| Problem | Command / Fix |
|---|---|
| `permission denied` on docker.sock | `sudo usermod -aG docker $USER && newgrp docker` |
| `port already allocated` | Find and stop the conflicting process using that port |
| `pull access denied` | `docker login ghcr.io -u <your-username>` (use PAT as password) |
| `repository name must be lowercase` | Ensure image tag uses lowercase username in workflow + compose |
| `ssh: no key found` in Actions | Regenerate SSH key, update `VPS_SSH_KEY` secret with full key |
| `JavaScript heap out of memory` during build | Never build on VPS — always build via GitHub Actions |
| `denied` on docker pull | Re-login with a fresh token, check GHCR package visibility |

---

## Summary: Normal Day-to-Day Workflow

```bash
# On local machine, after making code changes:
git add .
git commit -m "update feature"
git push origin main

# GitHub Actions automatically builds, pushes, and deploys.
# No manual VPS commands needed unless something fails.

# If auto-deploy fails, manually run on VPS:
cd /var/www/<your-repo>
docker compose pull
docker compose up -d
docker compose ps
curl http://localhost:5000/health
```

---

### Still need to fill in
- `<your-username>` / `<your-repo>` — your GitHub username and repo name
- `<project>` — a short name for container naming
- `PORT` — your app's actual port
- Database service (Mongo/Postgres) if self-hosting instead of using a managed DB
- Nginx + SSL/Certbot config if you want a domain with HTTPS in front of this (not covered here — ask if you want this added)
