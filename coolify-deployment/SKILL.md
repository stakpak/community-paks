---
name: coolify-deployment
description: |
  Deploy Coolify self-hosted PaaS for application hosting. Covers VM provisioning, installation, GitHub integration, and application deployment with auto-SSL. Includes API-first automation workflows for agents.
license: MIT
tags:
  - coolify
  - paas
  - self-hosted
  - docker
  - deployment
  - heroku-alternative
  - ssl
  - traefik
  - api
metadata:
  author: Stakpak <team@stakpak.dev>
  version: "1.0.4"
---

# Coolify Self-Hosted PaaS Deployment

## Quick Start

### Install Coolify on VM

```bash
ssh -i <key-file> <user>@<instance-ip>
curl -fsSL https://cdn.coollabs.io/coolify/install.sh | sudo bash
# Access: http://<instance-ip>:8000
```

### Deploy Application

1. Access Coolify UI at `http://<instance-ip>:8000`
2. Create admin account (first visitor gets admin)
3. **Skip onboarding wizard** → Servers → localhost → Proxy → **Start Proxy** (see [Post-Install Setup](#post-install-setup))
4. Add GitHub source → Create project → Deploy from repo
5. Access app at `http://<container-id>.<instance-ip>.sslip.io`

## Server Requirements

| Resource | Minimum | Recommended |
|----------|---------|-------------|
| CPU | 2 vCPU | 4 vCPU |
| RAM | 2GB | 4GB |
| Storage | 30GB | 50GB |
| OS | Ubuntu 20.04/22.04/24.04 LTS | Ubuntu 24.04 LTS |

### Required Ports

| Port | Purpose | Notes |
|------|---------|-------|
| TCP 22 | SSH | Restrict to your IP |
| TCP 80 | HTTP + Let's Encrypt ACME | Required for auto-SSL |
| TCP 443 | HTTPS | |
| TCP 8000 | Coolify UI | |
| **TCP 6001-6002** | **Realtime websockets** | **Required — without these the UI shows stale state (e.g. "Proxy Exited" when running) and throws "Cannot connect to real-time service" warnings** |

## Cloud VM Deployment

### 1. Provision Infrastructure

```bash
# Security group must include ALL the ports above, including 6001-6002
# Launch instance: 2+ vCPU, 2GB+ RAM, 30GB+ root, Ubuntu LTS, public IP, SSH key
```

### 2. Install Coolify

```bash
chmod 400 /path/to/key.pem
ssh -o StrictHostKeyChecking=no -i /path/to/key.pem <user>@<instance-ip>
curl -fsSL https://cdn.coollabs.io/coolify/install.sh | sudo bash
```

Installation takes 3-5 minutes.

### 3. Initial Setup

1. Open browser: `http://<instance-ip>:8000`
2. **IMPORTANT**: Create admin account immediately (first visitor becomes admin)
3. Complete setup wizard

### 4. Verify Installation

```bash
ssh -i /path/to/key.pem <user>@<instance-ip> \
  "sudo docker ps --format 'table {{.Names}}\t{{.Status}}' | grep coolify"
```

## Post-Install Setup

**The Traefik proxy does NOT start automatically. Apps cannot be routed until you start it.**

After creating the admin account:

1. If the onboarding wizard opens: click **Skip Setup** (or choose "This Machine" if you want Coolify to deploy on the same VM it's running on — avoids SSH key setup)
2. Navigate to **Servers → localhost → Proxy**
3. Click **Start Proxy**
4. Verify with `sudo docker ps | grep coolify-proxy` — should show `(healthy)`

> **Note:** The UI may continue to show "Proxy Exited" even after start if ports 6001-6002 are blocked. Check Docker directly to confirm actual status.

### "This Machine" vs "Remote Server"

| Pattern | When to use |
|---------|-------------|
| **This Machine** | Coolify runs on the same VM as your apps. Cheap single-server setup. Skips SSH key generation. |
| **Remote Server** | Coolify is a control plane; apps deploy to other VMs via SSH. Production-grade separation. |

## Application Deployment

### Supported Build Methods

| Method | Use Case |
|--------|----------|
| Dockerfile | Custom container builds |
| Nixpacks | Auto-detected language builds |
| Docker Compose | Multi-container apps |
| Static | HTML/JS/CSS sites |

### Deploy from GitHub (UI)

1. **Add Source**: Sources → Add New → GitHub App
2. **Create Project**: Projects → Add New Project
3. **Add Resource**: Add New Resource → Public/Private Repository
4. **Configure**:
   - Repository URL
   - Branch
   - Build pack (Dockerfile/Nixpacks)
   - Port to expose
5. **Deploy**: Click Deploy button

### Domain Configuration

**Auto-generated (sslip.io):**
- Format: `<container-id>.<public-ip>.sslip.io`
- Works immediately, no DNS required
- Some browsers flag as suspicious — prefer custom domain for sharing

**Custom Domain:**
1. Add DNS A record → VM public IP
2. Update domain in Coolify resource settings
3. Enable SSL (Let's Encrypt auto-configured, requires port 80 open)

## API-First Automation

For agents, CI pipelines, and scripted workflows, the API is far more reliable than browser automation.

### Enable API (disabled by default)

1. Navigate to **Settings → Advanced**
2. Scroll to **API Settings**
3. Toggle **API Access** ON
4. Click **Save**
5. Optionally restrict **Allowed IPs for API Access** (default `0.0.0.0` = anywhere)

### Generate Token

1. **Keys & Tokens** (sidebar) → **API Tokens** tab
2. Enter description, choose expiry
3. Check permission (`root` for full access, `read-only` for safety)
4. Click **Create** — **copy the token immediately**, it's shown only once

Token format: `<id>|<secret>` (e.g. `2|5xSDTK796OnhlkD90GHjbYWJjKPNh0DHCEsUxUsk9746cb8a`)

### API Base

```
http://<instance-ip>:8000/api/v1
Authorization: Bearer <token>
Rate limit: 200 requests/minute (visible in X-RateLimit-* headers)
```

### End-to-End Deploy Recipe

```bash
export COOLIFY_URL="http://52.35.32.113:8000"
export TOKEN="2|your-token-here"

# 1. Create project
PROJECT=$(curl -s -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"name":"my-app","description":"My application"}' \
  $COOLIFY_URL/api/v1/projects | jq -r .uuid)

# 2. Get server UUID (use localhost if Coolify runs on the target VM)
SERVER=$(curl -s -H "Authorization: Bearer $TOKEN" $COOLIFY_URL/api/v1/servers | jq -r '.[0].uuid')

# 3. Create application from public repo
APP=$(curl -s -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d "{
    \"project_uuid\":\"$PROJECT\",
    \"server_uuid\":\"$SERVER\",
    \"environment_name\":\"production\",
    \"git_repository\":\"https://github.com/user/repo\",
    \"git_branch\":\"main\",
    \"build_pack\":\"nixpacks\",
    \"ports_exposes\":\"3000\",
    \"name\":\"my-app\",
    \"instant_deploy\":false
  }" \
  $COOLIFY_URL/api/v1/applications/public | jq -r .uuid)

# 4. Add env vars (note: is_buildtime AND is_runtime if build needs them)
curl -s -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"key":"DATABASE_URL","value":"postgres://...","is_buildtime":true,"is_runtime":true}' \
  $COOLIFY_URL/api/v1/applications/$APP/envs

# 5. Deploy (force=true is critical — see gotchas below)
DEPLOY_UUID=$(curl -s -X POST -H "Authorization: Bearer $TOKEN" \
  "$COOLIFY_URL/api/v1/deploy?uuid=$APP&force=true" | jq -r '.deployments[0].deployment_uuid')

# 6. Poll deployment status
while true; do
  STATUS=$(curl -s -H "Authorization: Bearer $TOKEN" \
    $COOLIFY_URL/api/v1/deployments/$DEPLOY_UUID | jq -r .status)
  echo "$(date +%H:%M:%S) $STATUS"
  [[ "$STATUS" == "finished" || "$STATUS" == "failed" ]] && break
  sleep 15
done

# 7. Get the app's public URL
curl -s -H "Authorization: Bearer $TOKEN" \
  $COOLIFY_URL/api/v1/applications/$APP | jq -r .fqdn
```

### Common API Endpoints

| Method | Path | Purpose |
|--------|------|---------|
| `GET` | `/health` | Unauthenticated health check |
| `GET` | `/api/v1/version` | Coolify version |
| `GET` | `/api/v1/servers` | List all servers |
| `POST` | `/api/v1/projects` | Create project |
| `GET` | `/api/v1/projects/{uuid}` | Get project + environments |
| `POST` | `/api/v1/applications/public` | Create app from public repo |
| `POST` | `/api/v1/applications/private-github-app` | Create app from private repo (via GitHub App) |
| `GET` | `/api/v1/applications/{uuid}` | Get app details (fqdn, status) |
| `PATCH` | `/api/v1/applications/{uuid}` | Update app config |
| `GET` | `/api/v1/applications/{uuid}/envs` | List env vars |
| `POST` | `/api/v1/applications/{uuid}/envs` | Create env var |
| `POST` | `/api/v1/deploy?uuid=<app>&force=true` | Trigger deploy |
| `GET` | `/api/v1/deployments/{uuid}` | Deployment status + logs |
| `GET` | `/api/v1/deployments?uuid=<app>` | List app deployments |

### API Gotchas (Hard-Won Knowledge)

1. **Field naming:** use `is_buildtime` / `is_runtime`, NOT `is_build_time` / `is_run_time`. Validation error is cryptic.
2. **Deploy dedup:** `POST /deploy` without `force=true` silently no-ops if a deploy is already queued for the current commit — you'll see `"Deployment already queued for this commit."` and nothing happens. Always use `force=true` when scripting.
3. **First deploy:** After creating an app, the first deploy must be triggered explicitly — `instant_deploy: true` in the create payload does not always work.
4. **Env var duplication:** When the app has a `preview` env, adding a var creates it in both `production` and `preview` scopes — expect duplicates in the list response.
5. **Nixpacks env vars:** `NIXPACKS_NODE_VERSION` and similar are auto-injected as buildtime-only. Don't try to override these via API.

## Environment Variables: Build-Time vs Runtime

**Critical distinction** — many deploy failures come from getting this wrong.

| Scope flag | When to use |
|------------|-------------|
| `is_buildtime: true` | Available as `ARG` during `docker build`. Needed if build step reads env (e.g. `drizzle-kit migrate`, `NEXT_PUBLIC_*` in Next.js, build-time codegen). |
| `is_runtime: true` | Available to the running container. Needed for anything the app reads at startup or request time. |
| Both | Most real apps need both (DB URLs, API keys read during build AND at runtime). |

If your build step touches the database or external services, **always set both**. Setting only `is_runtime` and then failing in build is the #1 deploy failure pattern.

## Nixpacks (Auto-Detected Builds)

Nixpacks detects your language and generates a Dockerfile. Works well for most Node/Python/Go/Ruby apps, but beware:

- Runs strict production build (e.g. `next build` with full type-checking) — **missing transitive deps fail**. If your `package.json` imports `@next/env` but doesn't list it, dev works via hoisting but prod build fails.
- Inspects `package.json` to pick the right package manager (npm/pnpm/yarn)
- `NODE_ENV=production` is set — devDependencies are skipped unless `NPM_CONFIG_PRODUCTION=false` is set

### Overriding Nixpacks Commands

Use `PATCH /api/v1/applications/{uuid}` to inject custom commands:

```json
{
  "install_command": "pnpm add @missing-dep && pnpm i --frozen-lockfile=false",
  "build_command": "pnpm run build:prod",
  "start_command": "node server.js"
}
```

### Inspecting the Generated Dockerfile

When a Nixpacks build fails, check the generated Dockerfile to understand what went wrong:

```bash
# Dockerfile output appears in deployment logs as a hidden entry
curl -s -H "Authorization: Bearer $TOKEN" \
  $COOLIFY_URL/api/v1/deployments/$DEPLOY_UUID | \
  jq -r '.logs | fromjson[] | select(.command and (.command | contains("Dockerfile"))) | .output'
```

## Verification

### Check Container Status

```bash
ssh -i /path/to/key.pem <user>@<instance-ip> \
  "sudo docker ps --format 'table {{.Names}}\t{{.Ports}}\t{{.Status}}'"
```

### Test Application

```bash
APP_URL="http://<container-id>.<public-ip>.sslip.io"
curl -s $APP_URL/
curl -s $APP_URL/health
```

### Check Logs

```bash
# Application container logs
sudo docker logs <container-name> --tail 50

# Coolify itself
sudo docker logs coolify --tail 50

# Deployment logs via API
curl -s -H "Authorization: Bearer $TOKEN" \
  $COOLIFY_URL/api/v1/deployments/$DEPLOY_UUID | \
  jq -r '.logs | fromjson[] | "[\(.type)] \(.output)"'
```

## Configuration Reference

### Coolify Containers

| Container | Purpose | Port |
|-----------|---------|------|
| coolify | Main application | 8000 |
| coolify-proxy | Traefik reverse proxy | 80, 443, 8080 |
| coolify-db | PostgreSQL database | 5432 |
| coolify-redis | Redis cache | 6379 |
| coolify-realtime | WebSocket server | 6001-6002 |
| coolify-sentinel | Metrics collector | (internal) |

### Important Paths

| Path | Purpose |
|------|---------|
| `/data/coolify/source/.env` | Coolify configuration (back this up!) |
| `/data/coolify/proxy/dynamic/` | Traefik dynamic config |
| `/data/coolify/applications/{uuid}/` | Per-application compose + artifacts |
| `/data/coolify/databases/` | Managed database volumes |

### Environment Variables (Installation)

| Variable | Purpose | Example |
|----------|---------|---------|
| `ROOT_USERNAME` | Admin username | `admin` |
| `ROOT_USER_EMAIL` | Admin email | `admin@example.com` |
| `ROOT_USER_PASSWORD` | Admin password | `SecurePass123` |
| `AUTOUPDATE` | Auto-update toggle | `true`/`false` |

## Production Checklist

- [ ] Create admin account immediately after install
- [ ] Start Traefik proxy (Servers → localhost → Proxy → Start)
- [ ] Open ports 6001-6002 in security group for websockets
- [ ] Backup `/data/coolify/source/.env` to secure location
- [ ] Configure firewall/security group rules
- [ ] Set up custom domain with SSL (sslip.io is fine for staging)
- [ ] Restrict SSH access to specific IPs
- [ ] Enable API + generate token (if using automation)
- [ ] Configure monitoring/alerts
- [ ] Test deployment rollback procedure
- [ ] Document access credentials securely

## Troubleshooting

### Debugging Playbook

When a deployment fails:

```bash
# 1. Find the failing deployment UUID
curl -s -H "Authorization: Bearer $TOKEN" \
  "$COOLIFY_URL/api/v1/deployments?uuid=$APP_UUID" | jq '.[0] | {deployment_uuid, status, finished_at}'

# 2. Pull full logs and find error lines
curl -s -H "Authorization: Bearer $TOKEN" \
  "$COOLIFY_URL/api/v1/deployments/$DEPLOY_UUID" | \
  jq -r '.logs | fromjson[] | select(.type=="stderr") | .output' | \
  grep -iE "error|fail|exit code" | head -20

# 3. Inspect the application directory on the VM
ssh -i /path/to/key.pem ubuntu@<ip> \
  "sudo ls -la /data/coolify/applications/$APP_UUID/"

# 4. Check the Coolify worker queue (Postgres)
ssh -i /path/to/key.pem ubuntu@<ip> \
  "sudo docker exec coolify-db psql -U coolify -d coolify -c \
   'SELECT deployment_uuid, status, finished_at FROM application_deployment_queues ORDER BY id DESC LIMIT 5;'"
```

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| 404 on public IP | Wrong URL format | Access via sslip.io domain, not raw IP |
| Cannot access Coolify UI | Port 8000 blocked | Open port 8000 in SG |
| UI shows "Cannot connect to real-time service" | Ports 6001-6002 blocked | Open them in SG |
| UI shows "Proxy Exited" but docker shows running | Stale UI from missing websockets | Open 6001-6002; verify via `docker ps` |
| Deploy succeeds but app returns 502 | Wrong `ports_exposes` value | Must match port your app actually listens on |
| Nixpacks build fails: `Cannot find module 'X'` | Transitive dep missing from `package.json` | Add as direct dep OR override `install_command` |
| Deploy says "already queued" but nothing happens | Dedup on same commit | Use `force=true` query param |
| Env var changes not picked up | Build cache | Redeploy with `force=true` |
| SSL certificate fails | Port 80 blocked or DNS not propagated | Check SG + `dig <domain>` |
| Container not starting | Runtime error | Check `docker logs <container>` |
| Out of disk space | Accumulated images/logs | `docker system prune -a` |
| Coolify unresponsive | Worker queue stuck | `docker restart coolify` |
| "Validation failed: is_build_time not allowed" | Wrong field name | Use `is_buildtime` (no underscore) |

## Update Coolify

```bash
ssh -i /path/to/key.pem <user>@<instance-ip> \
  "cd /data/coolify/source && sudo bash upgrade.sh"
```

## Cleanup

```bash
# Stop all containers
ssh -i /path/to/key.pem <user>@<instance-ip> \
  "cd /data/coolify/source && sudo docker compose down"

# Remove Coolify data (destructive)
ssh -i /path/to/key.pem <user>@<instance-ip> \
  "sudo rm -rf /data/coolify"

# Terminate cloud instance
```

## References

- [Coolify Documentation](https://coolify.io/docs)
- [Coolify API Reference](https://coolify.io/docs/api-reference/authorization)
- [Coolify Installation Guide](https://coolify.io/docs/get-started/installation)
- [Coolify GitHub](https://github.com/coollabsio/coolify)
- [Nixpacks Documentation](https://nixpacks.com/docs)
- [Traefik Documentation](https://doc.traefik.io/traefik/)
- [sslip.io](https://sslip.io/)
