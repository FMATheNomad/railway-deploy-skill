---
name: railway-deploy
description: Autonomously diagnose and fix Railway deployment failures for FastAPI + Next.js + PostgreSQL projects. Use this skill when the user encounters deployment crashes, failed builds, auto-deploy failures, health check errors, or any Railway deployment issue. Also use when the user says "deploy failed", "crash on railway", "build failed", "auto-deploy error", "deployment timeout", or "app not responding after deploy".
compatibility: opencode
metadata:
  author: custom
  stack: fastapi, nextjs, postgresql, docker, railway
  source: docs.railway.app
---

# Railway Autonomous Deployment Diagnosis

When the user reports a Railway deployment issue, follow this workflow automatically without asking for additional context.

## Step 1: Collect Evidence (run in parallel)

```bash
# Get deployment logs (build + runtime)
railway logs --lines 100
railway logs --build --lines 50

# Check environment variables
railway variable list --kv

# Check deployment status
railway status
```

Read these project files (if they exist):
- `package.json` — scripts (especially start/build), dependencies
- `requirements.txt` or `pyproject.toml` — Python dependencies
- `railway.json` or `railway.toml` — Railway config-as-code
- `Dockerfile` — custom Docker build
- `next.config.js` or `next.config.ts` — Next.js output config
- `start.sh` or `entrypoint.sh` — custom startup script
- `.gitignore` — check if critical files excluded

## Step 2: Classify the Error

### Pattern A: Build OOM (Memory)
**Error:** `Killed`, `exit code 137`, `signal 9`, `Out of memory`

**Common with Next.js.** Railway Hobby plan has memory limits.

**Fix:**
1. In `next.config.ts`:
```typescript
const nextConfig: NextConfig = {
  output: "standalone",
};
export default nextConfig;
```
2. Set env var: `railway variable set NODE_OPTIONS=--max-old-space-size=2048`
3. For Docker: use multi-stage build to reduce image size

### Pattern B: No Start Command
**Error:** `No start command could be found`

**Fix:** Set start command in railway.json or service settings:

**Next.js (standalone):**
```bash
node .next/standalone/server.js
```
**Note:** Requires `output: "standalone"` in next.config.ts AND start script in package.json: `"start": "node .next/standalone/server.js"`

**FastAPI:**
```bash
uvicorn main:app --host 0.0.0.0 --port $PORT
```

**With Dockerfile** — Start command in Railway settings for Dockerfile service needs shell wrapper:
```bash
/bin/sh -c "exec uvicorn main:app --host 0.0.0.0 --port $PORT"
```
**CMD in Dockerfile** can use shell form directly (automatically via `/bin/sh -c`):
```dockerfile
CMD uvicorn main:app --host 0.0.0.0 --port $PORT
```

### Pattern C: Port Binding / Application Failed to Respond
**Error:** `port already in use`, `application failed to respond`, `service unavailable`

**Always use Railway's $PORT env var:**

**Next.js** (in railway.json):
```json
{
  "deploy": { "startCommand": "node .next/standalone/server.js" }
}
```

**FastAPI:**
```python
import os
port = int(os.getenv("PORT", 8000))
```
Start command: `uvicorn main:app --host 0.0.0.0 --port $PORT`

### Pattern D: Database Connection
**Error:** `Connection refused`, `ECONNREFUSED`, `getaddrinfo ENOTFOUND`, `timeout`

**Fix:**
1. Check DATABASE_URL: `railway variable list --kv | grep DATABASE`
2. Railway auto-injects `DATABASE_URL` for PostgreSQL plugin
3. Use reference variables to link services
4. Add healthcheck endpoint BEFORE full DB init if DB is slow

### Pattern E: Healthcheck Failure
**Error:** `failed with service unavailable`, `failed with status 400`, healthcheck timeout

**From Railway docs:** Healthcheck timeout defaults to 300s (configurable via `RAILWAY_HEALTHCHECK_TIMEOUT_SEC` or in railway.json).

**Fix:**
1. Ensure app listens on `$PORT`
2. Add `/health` endpoint returning HTTP 200
3. If using custom hostname filtering, allow `healthcheck.railway.app`
4. Increase timeout if app startup is slow:
```json
{
  "deploy": {
    "healthcheckPath": "/health",
    "healthcheckTimeout": 300
  }
}
```

### Pattern F: Python Dependency Issues
**Error:** `ModuleNotFoundError`, `pip install` fails

**Fix:**
1. Pin all versions in `requirements.txt`
2. For psycopg2: use `psycopg2-binary`
3. For platform-specific packages: use Dockerfile
4. Install system deps via Railpack:
```
RAILPACK_DEPLOY_APT_PACKAGES=libpq-dev
```

### Pattern G: Auto-Deploy / Git Push Fails
**Error:** Deploy triggers on push but fails

**Fix:**
1. Check railway.json exists with correct config
2. Check watch patterns if using monorepo:
```json
{
  "build": { "watchPatterns": ["src/**"] }
}
```
3. Check .gitignore isn't excluding required files

### Pattern H: Monorepo Issues
**Fix — set root directory or commands:**

```json
{
  "build": {
    "builder": "RAILPACK",
    "buildCommand": "cd apps/api && pip install -r requirements.txt"
  },
  "deploy": {
    "startCommand": "cd apps/api && uvicorn main:app --host 0.0.0.0 --port $PORT"
  }
}
```
**Note:** railway.json path does NOT follow root directory — use absolute path.

## Step 3: Apply Fix

Based on pattern match, apply the fix:
- Edit config files (railway.json, next.config.ts, Dockerfile, etc.)
- Set env vars via `railway variable set KEY=VALUE`
- Or guide user to change in Railway dashboard

## Step 4: Verify

```bash
railway up
# or push to git for auto-deploy
# Then check:
railway logs --lines 30
railway logs --build --lines 20
```

## Step 5: If Still Failing

Check these in order:
1. `railway status` — any platform issues?
2. Check Railway status page: https://status.railway.com
3. Is it a Dockerfile issue? Try removing Dockerfile and using Railpack
4. Is it a monorepo path issue?
5. Large image? Use multi-stage Docker build + .dockerignore

## Railway CLI Reference (Official — from `railway --help`)

```bash
# Authentication & Project
railway login                          # Login to Railway account
railway init                           # Create new project
railway link [projectId]               # Link directory to existing project
railway unlink                         # Disassociate project from directory
railway status                         # Show current project info
railway status --json                  # JSON output
railway whoami                         # Current logged in user

# Deploying
railway up                             # Upload & deploy from current directory
railway up --detach                    # Deploy in background (don't attach logs)
railway up --ci                        # CI mode — build logs only, then exit
railway up --service api --env prod    # Deploy specific service/environment
railway up -m "fix: deploy"           # Attach deployment message
railway redeploy                       # Redeploy latest deployment
railway redeploy -y                    # Skip confirmation
railway redeploy --from-source         # Pull latest commit/image (not cached)
railway restart -y                     # Restart service (no rebuild)
railway down                          # Remove most recent deployment

# Logs
railway logs                           # Stream live deploy logs
railway logs --build                   # Build logs
railway logs --http                    # HTTP request logs
railway logs --lines 100               # Last 100 lines (no streaming)
railway logs --latest                  # Show latest deploy (even if failed)
railway logs --since 1h                # Logs from last hour
railway logs --filter "@level:error"   # Filter by error level
railway logs --filter "error"          # Text search in logs
railway logs --json                    # JSON format output
railway logs --service api --env prod  # Specific service/environment

# Environment Variables
railway variable list                  # List variables (JSON)
railway variable list --kv             # List in KEY=VALUE format
railway variable list --json           # JSON with raw values
railway variable set KEY=VALUE         # Set variable
railway variable set KEY=VALUE --skip-deploys  # Set without triggering deploy
echo "secret" | railway variable set KEY --stdin  # Set from stdin
railway variable delete KEY            # Delete variable

# Deployments & Services
railway deployment list                # List deployments with statuses
railway deployment list --json         # JSON output (IDs, statuses, metadata)
railway deployment list --limit 10     # Limit results
railway service list --json            # List all services
railway service status                 # Show deployment status for services
railway service logs                   # View logs from a service
railway service source connect --repo owner/repo --branch main  # Connect GitHub

# Databases
railway connect postgres               # Open psql shell
railway connect redis --env production # Connect to specific env

# Debugging
railway ssh                            # SSH into service
railway metrics                        # Resource & HTTP metrics
railway open                           # Open project dashboard in browser
```

## railway.json Schema (Official)

```json
{
  "$schema": "https://railway.com/railway.schema.json",
  "build": {
    "builder": "RAILPACK",
    "buildCommand": null,
    "watchPatterns": ["src/**"],
    "dockerfilePath": null,
    "railpackVersion": null
  },
  "deploy": {
    "startCommand": "uvicorn main:app --host 0.0.0.0 --port $PORT",
    "preDeployCommand": ["alembic upgrade head"],
    "healthcheckPath": "/health",
    "healthcheckTimeout": 300,
    "restartPolicyType": "ALWAYS",
    "restartPolicyMaxRetries": 3,
    "overlapSeconds": null,
    "drainingSeconds": null,
    "cronSchedule": null
  }
}
```

## FastAPI Healthcheck Template

```python
from fastapi import FastAPI
import os

app = FastAPI()

@app.get("/health")
async def health():
    return {"status": "ok", "port": os.getenv("PORT", "unknown")}
```
**Important:** Railway healthcheck uses hostname `healthcheck.railway.app`. If your app restricts hostnames, allow this.
