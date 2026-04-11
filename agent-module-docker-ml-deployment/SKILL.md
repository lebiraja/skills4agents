---
name: agent-module-docker-ml-deployment
description: "Module: Docker + ML Service Deployment Standard"
---
# Module: Docker + ML Service Deployment Standard

## 1. Module Metadata
- Module ID: `agent.module.docker-ml-deployment`
- Version: `1.0.0`
- Maturity: `production`
- Scope: End-to-end deployment lifecycle for containerized ML inference services — from model management and Docker composition to cloud hosting, CI/CD, observability, and rollback.
- Primary outcomes:
  - Reproducible, containerized ML service deployments across environments.
  - Safe model versioning and rollback procedures.
  - Production-grade health checks, monitoring, and scaling patterns.
  - Automated CI/CD with zero-downtime update capability.

---

## 2. Mission and Applicability
Use this module when deploying any ML-backed service using Docker Compose or a similar container orchestration layer, with a REST API serving model inference.

Apply when:
- A Python-based ML model (scikit-learn, PyTorch, TensorFlow, etc.) is served via a REST API (FastAPI, Flask, Django).
- The stack uses Docker Compose for multi-service orchestration (API + frontend + cache).
- Deployment target is a cloud VM (AWS EC2, GCP Compute Engine, Azure VM, DigitalOcean Droplet) or self-hosted server.
- CI/CD should automate deployment on code/model push.

Do not apply when:
- Deployment target is Kubernetes (use Helm charts and HPA patterns instead).
- Model serving uses a managed platform (AWS SageMaker, Vertex AI, Azure ML endpoints).
- The service is serverless (Lambda, Cloud Functions) — container startup cold-start patterns don't apply.

---

## 3. Architecture Pattern
- Pattern: `Multi-Service Docker Compose with ML Artifact Volume Mount`
- Core design rules:
  - Services are split by concern: `frontend` (static/nginx), `backend` (ML API), `cache` (Redis), `model-trainer` (one-off job).
  - ML model artifacts (`.pkl`, `.pt`, `.onnx`, `.safetensors`) are mounted as **read-only volumes** into the inference container — never baked into the image.
  - The model-trainer service runs under a Docker Compose `profile` (`--profile training`) so it is never started by accident on `docker compose up`.
  - Backend depends on cache with `condition: service_healthy` to prevent startup races.
  - All containers run as non-root users.
  - Log rotation is enforced at the Docker daemon level (`max-size: 10m, max-file: 5`).
  - Environment variables are supplied via `.env` files; no secrets are hardcoded or committed.

```
┌─────────────────────────────────────────────────┐
│               Cloud VM / Server                 │
│                                                 │
│  ┌──────────┐   ┌──────────────┐  ┌──────────┐ │
│  │ frontend │──▶│   backend    │─▶│  cache   │ │
│  │  (nginx) │   │  (ML API)    │  │  (Redis) │ │
│  │  :80/443 │   │  :8000       │  │  :6379   │ │
│  └──────────┘   └──────┬───────┘  └──────────┘ │
│                        │                        │
│                 ┌──────▼──────┐                 │
│                 │  ML Models  │                 │
│                 │ /app/models/│                 │
│                 │ (vol. mount)│                 │
│                 └─────────────┘                 │
└─────────────────────────────────────────────────┘
```

**Service roles:**
| Service | Image Base | Port | Purpose |
|---------|-----------|------|---------|
| `backend` | `python:3.11-slim` (two-stage) | 8000 | ML inference REST API |
| `frontend` | `node:20` → `nginx:alpine` (two-stage) | 80 / 443 | Static UI + API proxy |
| `cache` | `redis:7-alpine` | 6379 (internal) | Prediction result cache |
| `model-trainer` | Same as backend | — | One-off training job (profile-gated) |

---

## 4. Implementation Workflow

### Phase A: Repository and Environment Initialization
1. Confirm repository structure contains: `backend/`, `frontend/`, `docker-compose.yml`, `backend/.env.example`.
2. Create `backend/.env` from `.env.example`:
   ```bash
   cp backend/.env.example backend/.env
   ```
3. Generate a cryptographically secure `SECRET_KEY`:
   ```bash
   python3 -c "import secrets; print(secrets.token_hex(32))"
   ```
4. Set `ENVIRONMENT=production`, `LOG_LEVEL=INFO`, and `CORS_ORIGINS` with your actual domain in `backend/.env`.
5. Confirm `backend/.env` is listed in `.gitignore`:
   ```bash
   grep "\.env" .gitignore
   ```
   If missing: `echo "backend/.env" >> .gitignore`
6. Define the minimum `.env` contract for the project and commit `.env.example` with all keys present (values blank or placeholder).

**Minimum `.env` contract (adapt per project):**
```env
# Application
ENVIRONMENT=production
LOG_LEVEL=INFO
SECRET_KEY=<generate with secrets.token_hex(32)>

# ML model paths (inside container — match volume mount target)
MODEL_PATH=/app/models/<primary_model>.pkl
SCALER_PATH=/app/models/scaler.pkl
PREPROCESSOR_PATH=/app/models/preprocessor.pkl
MODEL_VERSION=1.0.0

# Cache
REDIS_URL=redis://cache:6379/0
CACHE_ENABLED=true
CACHE_TTL=300

# CORS — always include production domain
CORS_ORIGINS=["https://yourdomain.com"]

# Rate limiting
API_RATE_LIMIT=100/hour
```

Exit criteria:
- `backend/.env` exists and is not tracked by git.
- All required keys present in `.env.example`.
- `SECRET_KEY` is non-default and >= 32 bytes.

---

### Phase B: ML Model Artifacts Verification
1. Check that model artifact files exist under `backend/models/`:
   ```bash
   ls -lh backend/models/
   ```
2. If model files are absent, train them:
   ```bash
   # Option A — Docker (recommended for reproducibility)
   docker compose --profile training run --rm model-trainer

   # Option B — local Python
   cd backend && pip install -r requirements.txt && python train_model.py
   ```
3. Verify all required artifacts are present (minimum: primary model + any preprocessing artifacts — e.g., scaler, encoder):
   ```bash
   ls backend/models/*.pkl   # or *.pt, *.onnx, etc.
   ```
4. Record model metrics (R², RMSE, accuracy, etc.) from training output — these become the baseline for future comparisons.
5. Create initial versioned backups:
   ```bash
   VERSION="v1.0.0"
   DATE=$(date +%Y%m%d)
   for f in backend/models/<primary>.pkl backend/models/scaler.pkl backend/models/preprocessor.pkl; do
     cp "$f" "${f%.pkl}_${VERSION}_${DATE}.pkl"
   done
   ```

**Model file naming convention:**
```
<model_name>_v<MAJOR>.<MINOR>.<PATCH>_<YYYYMMDD>.pkl
# Example: gradient_boosting_model_v1.2.0_20260411.pkl
```

Exit criteria:
- All required model artifact files present in `backend/models/`.
- Training baseline metrics recorded.
- Initial versioned backup exists.

---

### Phase C: Docker Build and Local Validation
1. Build all service images:
   ```bash
   docker compose build --no-cache
   ```
2. Start all services:
   ```bash
   docker compose up -d
   ```
3. Confirm all containers are running and healthy:
   ```bash
   docker compose ps
   ```
4. Run health check against the ML API:
   ```bash
   curl http://localhost:8000/api/v1/health
   # Expected: {"status": "healthy", "model_loaded": true, "version": "1.0.0"}
   ```
5. Run a smoke prediction request:
   ```bash
   curl -X POST http://localhost:8000/api/v1/predict \
     -H "Content-Type: application/json" \
     -d '<minimal_valid_payload_for_your_model>'
   ```
6. Confirm frontend loads:
   ```bash
   curl -s -o /dev/null -w "%{http_code}" http://localhost:80
   # Expected: 200
   ```
7. Confirm cache is reachable:
   ```bash
   docker compose exec cache redis-cli ping
   # Expected: PONG
   ```

**Key Docker commands reference:**
```bash
# Rebuild after code changes
docker compose build --no-cache && docker compose up -d

# Follow logs
docker compose logs -f backend
docker compose logs -f frontend

# Enter a container for debugging
docker compose exec backend bash

# Stop and clean up
docker compose down
docker compose down -v  # also wipes cache volumes
```

Exit criteria:
- All services report healthy in `docker compose ps`.
- Health endpoint returns `model_loaded: true`.
- Smoke prediction request returns a valid response.
- Frontend serves HTML with HTTP 200.

---

### Phase D: Dockerfile Standards
Enforce two-stage builds for both backend and frontend.

**Backend two-stage pattern:**
```dockerfile
# Stage 1: install Python dependencies
FROM python:3.11-slim AS builder
WORKDIR /build
COPY requirements.txt .
RUN pip install --user --no-cache-dir -r requirements.txt

# Stage 2: production image
FROM python:3.11-slim AS production
# Install runtime system libs (e.g., OpenBLAS/LAPACK for numpy/scikit-learn)
RUN apt-get update && apt-get install -y --no-install-recommends \
    libopenblas-dev liblapack-dev && \
    rm -rf /var/lib/apt/lists/*
# Copy installed packages from builder
COPY --from=builder /root/.local /root/.local
# Copy application code (no model artifacts — they come via volume mount)
COPY app/ /app/app/
WORKDIR /app
# Run as non-root
RUN useradd -m appuser && chown -R appuser /app
USER appuser
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000", "--workers", "4"]
```

**Frontend two-stage pattern:**
```dockerfile
# Stage 1: build React/Vue/etc. static assets
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json .
RUN npm ci
COPY . .
RUN npm run build

# Stage 2: serve with nginx
FROM nginx:alpine AS production
COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
```

**nginx.conf API proxy pattern:**
```nginx
server {
    listen 80;
    root /usr/share/nginx/html;
    index index.html;

    # SPA fallback
    location / {
        try_files $uri $uri/ /index.html;
    }

    # Proxy API requests to backend container
    location /api/ {
        proxy_pass http://backend:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_read_timeout 60s;
    }
}
```

Exit criteria:
- Backend image uses two-stage build; no model files baked into image.
- Frontend image uses two-stage build; API proxied through nginx.
- Containers run as non-root users.

---

### Phase E: Cloud VM Provisioning (AWS EC2 / Equivalent)
1. **Instance sizing** (EC2 reference):
   | Use Case | Type | vCPU | RAM | Est. Cost |
   |----------|------|------|-----|-----------|
   | Dev / Test | t3.small | 2 | 2 GB | ~$15/mo |
   | Production | t3.medium | 2 | 4 GB | ~$30/mo |
   | High Traffic | t3.large | 2 | 8 GB | ~$60/mo |
   - Storage: 30 GB minimum (50 GB if training data stored on disk).
   - OS: Ubuntu 22.04 LTS or equivalent.

2. **Security group / firewall rules:**
   | Port | Protocol | Source | Purpose |
   |------|----------|--------|---------|
   | 22 | TCP | Your IP only | SSH |
   | 80 | TCP | 0.0.0.0/0 | HTTP |
   | 443 | TCP | 0.0.0.0/0 | HTTPS |
   | 8000 | TCP | 0.0.0.0/0 | Direct API (optional, dev only) |
   - **Never expose Redis port (6379/6380) publicly.**

3. **Docker installation (Ubuntu):**
   ```bash
   sudo apt-get update && sudo apt-get upgrade -y
   sudo apt-get install -y ca-certificates curl gnupg
   sudo install -m 0755 -d /etc/apt/keyrings
   curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
     sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
   sudo chmod a+r /etc/apt/keyrings/docker.gpg
   echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
     https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo $VERSION_CODENAME) stable" | \
     sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
   sudo apt-get update
   sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
   sudo usermod -aG docker $USER && newgrp docker
   ```

4. **Initial deployment:**
   ```bash
   git clone <repo-url> ~/app
   cd ~/app
   cp backend/.env.example backend/.env
   # Edit backend/.env: set SECRET_KEY, CORS_ORIGINS with real domain
   nano backend/.env

   # If model files not in repo, train them first
   docker compose --profile training run --rm model-trainer

   # Build and start
   docker compose build --no-cache
   docker compose up -d
   docker compose ps
   curl http://localhost:8000/api/v1/health
   ```

Exit criteria:
- Docker installed and running without sudo.
- All services healthy via `docker compose ps`.
- Health endpoint returns `model_loaded: true` from the cloud VM.

---

### Phase F: CI/CD Pipeline with GitHub Actions
1. Add GitHub repository secrets:
   | Secret | Value | Source |
   |--------|-------|--------|
   | `EC2_SSH_KEY` | Full `.pem` key content (including headers) | `cat your-key.pem` |
   | `EC2_HOST` | Public IP or domain of server | Cloud console |
   | `EC2_USERNAME` | OS user (e.g., `ubuntu`) | VM configuration |

2. Create `.github/workflows/deploy.yml`:
   ```yaml
   name: Deploy to Production

   on:
     push:
       branches: [main]

   jobs:
     deploy:
       runs-on: ubuntu-latest
       steps:
         - name: Checkout
           uses: actions/checkout@v4

         - name: Deploy via SSH
           uses: appleboy/ssh-action@v1
           with:
             host: ${{ secrets.EC2_HOST }}
             username: ${{ secrets.EC2_USERNAME }}
             key: ${{ secrets.EC2_SSH_KEY }}
             script: |
               cd ~/app
               git pull origin main
               docker compose down
               docker compose build --no-cache
               docker compose up -d
               docker compose ps
               curl -f http://localhost:8000/api/v1/health || exit 1
   ```

3. **If model files are stored in S3 (not git), add download step before build:**
   ```yaml
   - name: Download models from S3
     uses: appleboy/ssh-action@v1
     with:
       host: ${{ secrets.EC2_HOST }}
       username: ${{ secrets.EC2_USERNAME }}
       key: ${{ secrets.EC2_SSH_KEY }}
       script: |
         aws s3 cp s3://<your-bucket>/models/ ~/app/backend/models/ --recursive
   ```

4. Verify the health endpoint is called at the end of every deploy job; use `|| exit 1` to fail the pipeline if the API is unhealthy after deploy.

Exit criteria:
- Push to `main` triggers automatic deployment.
- CI/CD job fails if health check fails post-deploy.
- No secrets are hardcoded in workflow YAML.

---

### Phase G: SSL / HTTPS Setup
1. Ensure domain A record points to the cloud VM IP.
2. Install Certbot:
   ```bash
   sudo apt-get install -y certbot python3-certbot-nginx
   ```
3. Obtain certificate (stop nginx first to free port 80):
   ```bash
   docker compose stop frontend
   sudo certbot certonly --standalone -d yourdomain.com -d www.yourdomain.com
   docker compose start frontend
   ```
4. Update `frontend/nginx.conf`:
   ```nginx
   server {
       listen 80;
       server_name yourdomain.com;
       return 301 https://$server_name$request_uri;
   }
   server {
       listen 443 ssl;
       server_name yourdomain.com;
       ssl_certificate /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
       ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;
       # ... existing location blocks
   }
   ```
5. Mount certificates into frontend container in `docker-compose.yml`:
   ```yaml
   frontend:
     volumes:
       - /etc/letsencrypt:/etc/letsencrypt:ro
     ports:
       - "80:80"
       - "443:443"
   ```
6. Update `CORS_ORIGINS` in `backend/.env`:
   ```env
   CORS_ORIGINS=["https://yourdomain.com","https://www.yourdomain.com"]
   ```
7. Enable auto-renewal:
   ```bash
   sudo certbot renew --dry-run
   sudo certbot renew --post-hook "docker compose -f ~/app/docker-compose.yml restart frontend"
   sudo systemctl status certbot.timer
   ```

Exit criteria:
- `https://yourdomain.com` returns HTTP 200.
- HTTP redirects to HTTPS.
- Certificate renews automatically.

---

### Phase H: Model Update and Rollback Procedure
This is the most critical operational procedure. Follow exactly.

**Standard model update workflow:**
```
1. Train new model (Docker or local)
2. Record metrics — confirm improvement or acceptability
3. Backup current production artifacts with version suffix
4. Replace model files in backend/models/
5. Commit to git (or upload to S3)
6. Push to main → CI/CD deploys
7. Verify health endpoint after deploy
8. Monitor error logs for 15 minutes
```

**Step-by-step:**
```bash
# Step 1: Train
docker compose --profile training run --rm model-trainer
# Note: record printed metrics (RMSE, R², accuracy, etc.)

# Step 2: Backup current production model
VERSION="v1.0.0"
DATE=$(date +%Y%m%d)
cd backend/models
cp <primary_model>.pkl <primary_model>_${VERSION}_${DATE}.pkl
cp scaler.pkl scaler_${VERSION}_${DATE}.pkl
cp preprocessor.pkl preprocessor_${VERSION}_${DATE}.pkl

# Step 3: Commit new model
cd /path/to/repo
git add backend/models/<primary_model>.pkl backend/models/scaler.pkl backend/models/preprocessor.pkl
git commit -m "model: update to v1.1.0 — R2=0.951, RMSE=2.12"
git push origin main
# CI/CD deploys automatically

# Step 4: Verify post-deploy
curl https://yourdomain.com/api/v1/health
curl -X POST https://yourdomain.com/api/v1/predict \
  -H "Content-Type: application/json" \
  -d '<minimal_valid_payload>'
```

**Rollback procedure (if deploy fails):**
```bash
# On the server
cd ~/app/backend/models
cp <primary_model>_v1.0.0_<date>.pkl <primary_model>.pkl
cp scaler_v1.0.0_<date>.pkl scaler.pkl
cp preprocessor_v1.0.0_<date>.pkl preprocessor.pkl

# Restart backend to reload model
docker compose restart backend

# Verify
curl http://localhost:8000/api/v1/health
```

**Model storage strategy by size:**
| Model Size | Storage Strategy |
|-----------|-----------------|
| < 50 MB | Commit to git alongside code |
| 50 MB – 2 GB | Store in S3; download on deploy via CI/CD step |
| > 2 GB | Git LFS (`git lfs track "*.pkl"`) or S3 + streaming load |

Exit criteria:
- Rollback tested and confirmed < 2 minutes end-to-end.
- Model version is recorded in git commit message and in `/api/v1/health` response.
- At least one versioned backup exists before every production model update.

---

## 5. Decision Framework

| Decision Area | Preferred Option | Alternative | Selection Rule |
|---|---|---|---|
| Model artifact delivery | Volume mount (read-only) | Baked into Docker image | Volume mount: model updates don't require image rebuild. Bake in only if immutability is required. |
| Model storage (< 50 MB) | Commit to git | S3 | Git is simpler; use S3 only when files exceed GitHub LFS limits or build times become unacceptable. |
| Scaling strategy | Multiple Uvicorn workers (same container) | Multiple container replicas | Workers for I/O-bound routes; replicas only when CPU/memory is the bottleneck. |
| Cache strategy | Redis with TTL per prediction input hash | In-memory dict | Redis survives container restarts; use in-memory dict only for stateless/ephemeral deployments. |
| CI/CD trigger | Push to `main` branch | Manual SSH deploy | Automated on push reduces operator error; use manual only for regulated environments with change gates. |
| SSL termination | Certbot + nginx (in container) | Cloud load balancer TLS | LB TLS preferred for production HA; nginx TLS acceptable for single-VM deployments. |
| Backend API framework | FastAPI | Flask, Django REST | FastAPI for async ML workloads and OpenAPI auto-docs; Flask if team preference or existing codebase. |
| Zero-downtime model reload | Restart with volume swap | Hot-reload admin endpoint | Volume swap + restart is simpler and safe; hot-reload endpoint adds surface area but avoids brief downtime. |

---

## 6. Validation Strategy

### Functional Validation
- Health endpoint `GET /api/v1/health` returns `{"status": "healthy", "model_loaded": true}` after every deployment.
- Smoke prediction test with a known input returns expected output (can be regression-tested against a reference value from training).
- Frontend HTTP 200 and serves the `<title>` tag or a known HTML landmark.
- Cache returns `PONG` from `redis-cli ping`.
- All containers show `healthy` in `docker compose ps`.

### Failure Validation
- Remove a model file and confirm the backend fails to start with a clear log message (not a silent crash).
- Remove `backend/.env` and confirm compose fails with a readable error (not a null-key panic).
- Send a malformed prediction payload and confirm API returns HTTP 422 with a structured error (not HTTP 500).
- Simulate Redis failure (`docker compose stop cache`) and confirm API continues serving (degraded, no cache) without crashing.
- Kill the backend container mid-request and confirm Docker restarts it automatically (`restart: unless-stopped`).

### Security / Compliance Validation
- `grep -r "SECRET_KEY" .git` returns no results (secret never committed).
- `docker compose exec backend whoami` returns non-root user (e.g., `appuser`).
- `docker compose config | grep 6379` confirms Redis port is NOT mapped to `0.0.0.0` (only internal).
- CORS rejection: a request from an unlisted origin returns HTTP 403 or CORS error headers.
- `curl http://yourdomain.com/api/v1/health` redirects to HTTPS (301).

---

## 7. Benchmarks and SLO Targets
- API health check response time: `<= 200 ms` at p95
- Prediction endpoint latency (single request, warm model): `<= 500 ms` at p95 for tabular ML models; `<= 2 s` for deep learning models
- Service availability (uptime): `>= 99.5%` monthly for single-VM production deployments
- Model reload time after `docker compose restart backend`: `<= 30 s` for models < 500 MB
- Rollback execution time (backup → restart → health confirmed): `<= 5 minutes`
- CI/CD deploy pipeline duration (git pull → health confirmed): `<= 10 minutes`
- Cache hit ratio for repeated identical predictions: `>= 80%` under steady-state traffic
- Redis memory usage: `<= 256 MB` for prediction cache (enforce with `maxmemory` policy)
- Disk usage warning threshold: `>= 80%` triggers prune (`docker system prune -f`)

---

## 8. Risks and Controls

- Risk: Model artifact files missing or corrupted at startup.
  - Control: Startup lifespan hook (`@app.on_event("startup")`) loads and validates all model files; health endpoint immediately reports `model_loaded: false`; container restarts on health check failure.

- Risk: `.env` file committed to git, leaking secrets.
  - Control: `.env` in `.gitignore`; pre-commit hook or CI secret scan (e.g., `gitleaks`, `truffleHog`); audit `git log --all -- backend/.env` periodically.

- Risk: Deploying an untested model that degrades prediction quality.
  - Control: Training script prints metrics; gate model commit on metric comparison to baseline; add evaluation step to CI that fails if R²/accuracy drops below threshold.

- Risk: Disk full on VM halts containers.
  - Control: Monthly `docker system prune -f` via cron; `df -h /` alert at 80%; EBS volume sized with 50% headroom for model backups and logs.

- Risk: CI/CD deploys broken code and health check not caught.
  - Control: `curl -f http://localhost:8000/api/v1/health || exit 1` at end of deploy script; CI job fails and sends alert; rollback is manual but documented.

- Risk: Redis exposed publicly and accessed by external actors.
  - Control: Redis port mapped only on `127.0.0.1` (or not mapped at all); security group blocks 6379/6380 from internet; confirm with `docker compose config | grep "6379"`.

- Risk: Model version mismatch between scaler/preprocessor and primary model.
  - Control: All three artifacts are trained together in one run and committed atomically; `MODEL_VERSION` env var ties to git tag; rollback restores all three files together.

- Risk: SSL certificate expires and site goes down.
  - Control: `certbot.timer` systemd service enabled; `certbot renew --post-hook` restarts nginx; expiry alert via uptime monitor (UptimeRobot, BetterUptime, Pingdom).

---

## 9. Agent Execution Checklist
- [ ] `backend/.env` created from `.env.example` with real `SECRET_KEY` (not default).
- [ ] `backend/.env` confirmed absent from git history.
- [ ] `CORS_ORIGINS` set to include the production domain.
- [ ] All required model artifact files present in `backend/models/`.
- [ ] Model training baseline metrics recorded before first deploy.
- [ ] Versioned backup of all model artifacts created.
- [ ] Docker images build without errors (`docker compose build --no-cache`).
- [ ] All services healthy in `docker compose ps`.
- [ ] Health endpoint returns `model_loaded: true`.
- [ ] Smoke prediction request returns a valid response.
- [ ] Frontend serves HTTP 200.
- [ ] Redis `PONG` confirmed.
- [ ] Redis port NOT exposed to internet (confirm in security group and compose config).
- [ ] Backend container confirmed running as non-root user.
- [ ] CI/CD pipeline deployed and tested (push to `main` triggers deploy).
- [ ] CI/CD health check step present and configured to fail the job on unhealthy API.
- [ ] SSL certificate obtained and HTTPS working.
- [ ] HTTP → HTTPS redirect verified.
- [ ] SSL auto-renewal configured and dry-run passed.
- [ ] Rollback procedure tested end-to-end at least once.
- [ ] `docker system prune -f` scheduled monthly (cron or reminder).
- [ ] Log rotation configured (`max-size: "10m"`, `max-file: "5"` in compose logging block).

---

## 10. Operational Runbook

### Backend won't start
```bash
docker compose logs backend
# Check for: missing model file, missing .env, Redis not ready
ls backend/models/<primary>.pkl
ls backend/.env
docker compose logs cache
# Fix:
docker compose down && docker compose up -d --build
```

### `model_loaded: false` from health endpoint
```bash
docker compose exec backend ls -la /app/models/
docker compose config | grep -A5 volumes
docker compose logs backend | grep -i "error\|model\|pkl\|onnx"
```

### Prediction returns HTTP 500
```bash
docker compose logs backend | grep -A10 "500\|ERROR\|Exception"
# Common causes: input shape mismatch, scaler/model version mismatch, memory exhaustion
docker stats --no-stream
```

### Redis connection refused
```bash
docker compose exec cache redis-cli ping
grep REDIS_URL backend/.env
# REDIS_URL must use Docker service name: redis://cache:6379/0
docker compose restart cache
```

### Disk full
```bash
df -h /
docker system prune -f
docker compose exec backend find /app/logs -name "*.log.*" -delete
du -sh backend/models/* backend/data/* 2>/dev/null | sort -rh | head -20
```

### CI/CD fails
```bash
# Check GitHub Actions logs: Repository → Actions → (failed run)
# Common causes:
# 1. EC2_SSH_KEY missing header/footer lines
# 2. Security group blocks port 22 from GitHub Actions IPs
# 3. Disk full on VM: docker system prune -f on EC2, re-run
```

### SSL certificate expiry
```bash
sudo certbot renew --dry-run
sudo systemctl status certbot.timer
# If timer is inactive: sudo systemctl enable --now certbot.timer
```

---

## 11. Reuse Notes
This module is reusable across all Docker + ML service deployments. Adapt only:
- Model artifact file names and paths (`MODEL_PATH`, `SCALER_PATH`, `PREPROCESSOR_PATH` env keys).
- ML framework-specific runtime system libraries in backend Dockerfile (`libopenblas-dev` for scikit-learn; `libgomp1` for XGBoost; CUDA base image for GPU inference).
- Training script invocation (`train_model.py` → your actual script name).
- Prediction API endpoint path and payload schema.
- Health check response field names (keep `model_loaded` boolean consistent).
- CI/CD SSH deploy tool (replace `appleboy/ssh-action` with your preferred action or deployment service).
- Cloud provider VM sizing and storage recommendations.
- SLO targets by product tier (relax p95 latency for batch/async APIs, tighten for real-time user-facing inference).
