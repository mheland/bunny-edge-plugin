# Bunny Edge Scripting — Deployment Guide

## GitHub Integration

### Key Constraint

You **cannot** link an existing GitHub repository to a Bunny edge script after creation. The repository connection must be established when creating the script from Bunny's side. Options:

1. **New repo from template** — Bunny creates a preconfigured GitHub repo with deployment workflow included.
2. **Connect existing repo** — Select an existing repo during script creation and configure entry file, build/install commands. Bunny sets up the Actions workflow.

Either way, the connection is made at script creation time in the Bunny dashboard.

### Setup Steps

1. Navigate to **Edge Platform > Scripting > Add Script**
2. Choose **"Deploy and edit with GitHub"**
3. Name the script, select type (Standalone/Middleware)
4. Click **Connect GitHub Account** (first time) and grant Bunny GitHub app permissions
5. Choose to create new repo from template or connect existing repo
6. If connecting existing repo, configure:
   - Project framework/preset
   - Install command (if any)
   - Build command (if any)
   - Entry file location (e.g., `script.ts`)
7. Bunny creates the GitHub Actions deployment workflow

### After Setup

- Get **Script ID** and **Deploy Key** from: Script > Deployments > Settings
- Add as GitHub repository secrets: `SCRIPT_ID` and `DEPLOY_KEY`
- Push to `main` to trigger deployment

---

## GitHub Actions Workflow

Place in `.github/workflows/deploy.yml`:

```yml
name: Deploy to Bunny Edge

on:
  push:
    branches:
      - "main"

jobs:
  publish:
    runs-on: ubuntu-latest
    name: "Deploy Edge Script"
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Deploy Script to Bunny Edge Scripting
        uses: BunnyWay/actions/deploy-script@main
        with:
          script_id: ${{ secrets.SCRIPT_ID }}
          deploy_key: ${{ secrets.DEPLOY_KEY }}
          file: "script.ts"
```

Required secrets (from Script > Deployments > Settings):
- `SCRIPT_ID` — the numeric script identifier
- `DEPLOY_KEY` — deployment authorization key

---

## Bunny API Deployment

Base URL: `https://api.bunny.net`
Auth: `AccessKey` header with your account API key.

### Script Management

| Method | Path | Purpose |
|--------|------|---------|
| `GET` | `/compute/script` | List all scripts (paginated, searchable) |
| `POST` | `/compute/script` | Create new script |
| `GET` | `/compute/script/{id}` | Get script details |
| `POST` | `/compute/script/{id}` | Update script config |
| `DELETE` | `/compute/script/{id}` | Delete script |

### Code Management

| Method | Path | Purpose |
|--------|------|---------|
| `GET` | `/compute/script/{id}/code` | Get script source code |
| `POST` | `/compute/script/{id}/code` | Upload/update code |

### Releases

| Method | Path | Purpose |
|--------|------|---------|
| `POST` | `/compute/script/{id}/publish` | Publish current code as new release |
| `POST` | `/compute/script/{id}/publish/{uuid}` | Publish specific release (rollback) |
| `GET` | `/compute/script/{id}/releases` | List releases (paginated) |
| `GET` | `/compute/script/{id}/releases/active` | Get active release |

### Secrets

| Method | Path | Purpose |
|--------|------|---------|
| `GET` | `/compute/script/{id}/secrets` | List secrets |
| `POST` | `/compute/script/{id}/secrets` | Add secret |
| `PUT` | `/compute/script/{id}/secrets` | Upsert secret |
| `POST` | `/compute/script/{id}/secrets/{secretId}` | Update secret |
| `DELETE` | `/compute/script/{id}/secrets/{secretId}` | Delete secret |

### Variables

| Method | Path | Purpose |
|--------|------|---------|
| `GET` | `/compute/script/{id}/variables/{varId}` | Get variable |
| `POST` | `/compute/script/{id}/variables/{varId}` | Update variable |
| `DELETE` | `/compute/script/{id}/variables/{varId}` | Delete variable |
| `POST` | `/compute/script/{id}/variables/add` | Add variable |
| `PUT` | `/compute/script/{id}/variables` | Upsert variable |

### Other

| Method | Path | Purpose |
|--------|------|---------|
| `POST` | `/compute/script/{id}/deploymentKey/rotate` | Rotate deploy key |
| `GET` | `/compute/script/{id}/statistics` | Get usage statistics |

### Example: Deploy via API

```bash
# Upload code
curl -X POST "https://api.bunny.net/compute/script/12345/code" \
  -H "AccessKey: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"Code": "import * as BunnySDK from \"@bunny.net/edgescript-sdk\"; BunnySDK.net.http.serve(async (req) => new Response(\"Hello\"));"}'

# Publish
curl -X POST "https://api.bunny.net/compute/script/12345/publish" \
  -H "AccessKey: YOUR_API_KEY"
```

---

## Deployments & Rollback

Each publish creates a versioned release. To rollback:

1. Open Script > Deployments > Latest
2. Find the previous version
3. Click **Publish** next to it

Or via API: `POST /compute/script/{id}/publish/{releaseUuid}`

---

## Limits

| Resource | Limit |
|----------|-------|
| CPU time per request | 30 seconds |
| Active memory | 128 MB |
| Subrequests per request | 50 |
| Script size | 10 MB |
| Startup time | 500 ms |
| Env variables per script | 128 |
| Env variable size | 2 KB |
| Virtual FS storage | 64 MB |

CPU time excludes I/O wait. Platform is forgiving for infrequent spikes.

---

## Pricing

| Component | Cost |
|-----------|------|
| CPU time | $0.02 / 1,000 seconds |
| Requests | $0.20 / 1M requests |

CDN bandwidth billed separately. Billing is aggregated across all scripts, not per-script.

Rough estimates:
- 10M req/mo, 10ms CPU each: ~$4/mo
- 100M req/mo, 7ms CPU each: ~$34/mo
