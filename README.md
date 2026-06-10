# template-cicd-github-go

Drop-in GitHub Actions CI/CD for Go backends. Copy `.github/` into your project, set secrets, done.

---

## What this template includes

- **Test matrix** — Go 1.22 + 1.23, race detector, configurable proxy via `GOPROXY` variable
- **Wire codegen** — optional `google/wire` DI codegen step (remove if unused)
- **gosec** — Go security linter (advisory only, non-blocking)
- **PR comments** — test + gosec results posted to PR (update-or-create, no spam)
- **Docker build** — Buildx with GitHub Actions layer cache
- **Dual registry push** — custom registry + GHCR (`ghcr.io`)
- **Trivy scan** — CVE scan on built image (fails on CRITICAL unfixed CVEs)
- **Deploy staging** — PR → `main` deploys to staging server
- **Deploy prod** — push to `ops/**` or manual dispatch deploys to production server
- **DB migrations** — `docker compose run --rm migrate` before container swap
- **Health check + rollback** — curl `/health` after deploy, auto-rollback to previous SHA on failure
- **Notifications** — Telegram + Bale on deploy success/failure
- **Manual trigger** — `workflow_dispatch` with staging/prod choice
- **Concurrency groups** — cancels redundant runs on same branch
- **CodeQL SAST** — GitHub's static security analysis on PRs

---

## Prerequisites

- Docker + `docker compose` on deploy server
- Go project with a `Dockerfile` and `docker-compose.yml`
- `docker-compose.yml` must define a `migrate` service (or remove migration step — see Setup Checklist)
- Project exposes a `GET /health` endpoint returning 2xx

---

## Quickstart

```bash
# Copy .github/ into your Go project
cp -r .github/ /path/to/your/project/

# Set all secrets in your repo (see Secrets Reference below)
# Settings → Secrets and variables → Actions → New repository secret

# First deploy — push to ops branch
git push origin main:ops/initial-deploy
```

---

## Setup Checklist

- [ ] Copy `.github/` into your project root
- [ ] Set all **Repository Secrets** listed below (Settings → Secrets and variables → Actions)
- [ ] Set `HEALTH_PORT` as a **Repository Variable** (not secret — Settings → Variables → Actions)
- [ ] Optionally set `GOPROXY` as a **Repository Variable** for a custom Go proxy (e.g. Iran mirrors)
- [ ] Verify `docker-compose.yml` has a `migrate` service — or delete the migration step from both deploy jobs
- [ ] Remove `Install Wire` and `Generate Wire` steps if project does not use `google/wire`
- [ ] Remove JWT key steps from deploy jobs if project does not use RS256 JWT (`keys/private.pem` / `keys/public.pem`)
- [ ] Add public key of `SSH_PRIVATE_KEY` to `~/.ssh/authorized_keys` on prod server
- [ ] Add public key of `STAGING_SSH_PRIVATE_KEY` to staging server
- [ ] Push to `ops/first-deploy` to trigger first production deploy

---

## Secrets Reference

Set at: **GitHub repo → Settings → Secrets and variables → Actions → Repository secrets**

### Registry

| Secret | Description | Example |
|---|---|---|
| `REGISTRY` | Custom registry host | `registry.hamdocker.ir` |
| `REGISTRY_USERNAME` | Registry login username | `myuser` |
| `REGISTRY_PASSWORD` | Registry login password | `s3cr3t` |
| `BACKEND_REGISTRY_IMAGE` | Full image path | `registry.hamdocker.ir/org/myapp` |

> **GHCR** uses the built-in `GITHUB_TOKEN` automatically — no secret needed.

### SSH — Production

| Secret | Description | Example |
|---|---|---|
| `SSH_PRIVATE_KEY` | Ed25519 private key PEM for prod server | `-----BEGIN OPENSSH PRIVATE KEY-----...` |
| `SERVER_HOST` | Prod server IP or hostname | `1.2.3.4` |
| `SERVER_USER` | SSH user | `root` |
| `SERVER_DIR` | Deploy directory on server | `/root/backend` |

### SSH — Staging

| Secret | Description | Example |
|---|---|---|
| `STAGING_SSH_PRIVATE_KEY` | Ed25519 private key PEM for staging server | `-----BEGIN OPENSSH PRIVATE KEY-----...` |
| `STAGING_SERVER_HOST` | Staging server IP or hostname | `5.6.7.8` |
| `STAGING_SERVER_USER` | SSH user | `root` |
| `STAGING_SERVER_DIR` | Deploy directory on staging | `/root/backend-staging` |

### App Config

| Secret | Description |
|---|---|
| `BACKEND_ENV` | Full `.env` file contents for production (multiline) |
| `STAGING_BACKEND_ENV` | Full `.env` file contents for staging (multiline) |
| `JWT_PRIVATE_KEY` | RS256 private key PEM — remove deploy steps if unused |
| `JWT_PUBLIC_KEY` | RS256 public key PEM — remove deploy steps if unused |

### Notifications

| Secret | Description |
|---|---|
| `TELEGRAM_BOT_TOKEN` | Bot token from @BotFather |
| `TELEGRAM_CHAT_ID` | Target chat or group ID |
| `BALE_BOT_TOKEN` | Bale bot token |
| `BALE_CHAT_ID` | Bale target chat ID |

### Variables (non-secret)

Set at: **Settings → Secrets and variables → Actions → Variables tab**

| Variable | Description | Example |
|---|---|---|
| `HEALTH_PORT` | Port your app listens on | `8080` |
| `GOPROXY` | Custom Go module proxy (optional) | `https://go.iranserver.com/repository/go/,https://package-mirror.liara.ir/repository/go/,direct` |

---

## How to Generate SSH Keys

```bash
# Run on your local machine
ssh-keygen -t ed25519 -C "github-actions-deploy" -f ~/.ssh/deploy_key

# Copy public key to server (run on server or use ssh-copy-id)
cat ~/.ssh/deploy_key.pub >> ~/.ssh/authorized_keys

# Paste private key into GitHub secret
cat ~/.ssh/deploy_key
# Copy the entire output (including -----BEGIN / END lines) into SSH_PRIVATE_KEY secret
```

Repeat with a different key name for staging (`~/.ssh/deploy_key_staging`).

---

## Workflow Triggers

| Trigger | Jobs run |
|---|---|
| PR → `develop` | `test` (matrix 1.22 + 1.23) + CodeQL SAST |
| PR → `main` | `test` + `build-push` + `deploy-staging` + CodeQL SAST |
| Push to `ops/**` | `build-push` + `deploy-prod` |
| `workflow_dispatch` (staging) | `build-push` + `deploy-staging` |
| `workflow_dispatch` (prod) | `build-push` + `deploy-prod` |
| PR → `main` or `develop` | CodeQL SAST (separate workflow) |

> **Note:** Pushing to `ops/**` deploys directly to production without a PR review gate. This is intentional — `ops/` branches are used for infrastructure and deployment commits. Use branch protection or GitHub Environments with required reviewers if you need an approval gate.

---

## Project Structure (after copying into your project)

```
your-go-project/
├── .github/
│   └── workflows/
│       ├── ci-cd.yml       ← main CI/CD
│       └── codeql.yml      ← SAST
├── docker-compose.yml      ← must define app + migrate services
├── Dockerfile
└── ...
```

---

## Rollback Behavior

On each deploy, the current image SHA is written to `.current_tag` on the server. If the health check fails after deploy, the workflow automatically re-deploys the previous SHA and updates `.current_tag`. If no previous tag exists (first deploy), rollback is skipped and the job fails loudly.

---

## Security Notes

- Secrets passed through SSH heredocs are expanded by GitHub Actions before transmission. **Do not enable `ACTIONS_STEP_DEBUG=true`** in repositories that use this workflow — it will log expanded secret values.
- `docker compose run --rm migrate || true` swallows migration failures. If you need migration failures to block deployment, remove `|| true`.
