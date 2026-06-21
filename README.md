# taskapp_deploy — GitOps stack for Portainer

This repo is the **single source of truth** for what runs on the server. Portainer
watches it and redeploys the TaskApp stack whenever it changes.

```
GitHub push (app repo)
  -> CI builds image -> pushes to GHCR (tagged with commit SHA)
  -> CI bumps BACKEND_TAG / FRONTEND_TAG in this repo's .env, commits & pushes
  -> Portainer (git polling) sees .env change -> re-pulls + redeploys the stack
```

No SSH keys in CI, no manual server logins — the running version is whatever git says.

## Files

| File | Purpose |
|---|---|
| `docker-compose.prod.yml` | The stack: `frontend` + `backend` + `postgres` (GHCR images) |
| `.env` | **Committed**, non-secret deploy state (registry, owner, image tags). CI updates the tags |
| `.env.example` | Documents committed vars **and** the secret vars set in Portainer |

## One-time setup in Portainer

1. **Add the GHCR registry** (Registries -> Add registry -> Custom):
   `ghcr.io`, username = your GitHub user, password = a PAT with `read:packages`.
2. **Create the stack from this git repo** (Stacks -> Add stack -> Repository):
   - Repository URL + reference (`refs/heads/main`)
   - Compose path: `docker-compose.prod.yml`
   - **Enable automatic updates** (polling, e.g. 5m — or use a webhook).
   - **Enable "Re-pull image"** so a changed digest actually redeploys.
3. **Set the secret env vars** (Stack -> Environment variables), NOT in git:
   - `POSTGRES_PASSWORD` = a strong password
   - `SECRET_KEY` = `python -c "import secrets; print(secrets.token_hex(32))"`
4. `IMAGE_OWNER` in `.env` is already set to `ts-a-devops` (the org that owns the GHCR
   packages). Commit and push the repo so Portainer can read it.

## Why tags are pinned to the commit SHA (not `latest`)

Pinning makes the deploy **declarative and reversible**: the exact image running is
recorded in git, and rolling back is just reverting the `.env` commit. `latest`
hides which build is live and only redeploys if "Re-pull image" is on. We ship
SHA-pinning as the primary path; floating `latest` + re-pull is the simpler fallback.
