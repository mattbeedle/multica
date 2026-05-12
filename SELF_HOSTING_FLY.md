# Self-Hosting Multica on Fly.io

This guide deploys Multica as **two Fly apps** (`multica-backend` + `multica-web`) sharing a single public hostname, with **PlanetScale Postgres** for the database and **Tigris** for file uploads. The operator does not need to modify any source code.

For the Docker Compose self-host, see `SELF_HOSTING.md`. Multi-region, observability, and custom CDN are not yet covered for Fly — see "What is not covered here" at the end.

## Architecture

```
                      ┌─────────────────────────┐
   Browser ──HTTPS───▶│ multica.<your-domain>   │
   CLI/daemon ──────▶│ (Fly app, region iad)   │
                      └────────────┬────────────┘
                                   │
              ┌────────────────────┴────────────────────┐
              │                                         │
        ┌─────▼─────┐                            ┌──────▼──────┐
        │   web     │ ──flycast (private)─────▶ │   backend   │
        │ (Next.js) │   /api  /ws  /auth        │    (Go)     │
        │ port 3000 │   /uploads                │ port 8080   │
        └───────────┘                            └──────┬──────┘
                                                        │
                            ┌───────────────────────────┼─────────────┐
                            │                           │             │
                      ┌─────▼─────┐             ┌───────▼──────┐ ┌────▼────┐
                      │PlanetScale│             │   Tigris     │ │ (Resend │
                      │ Postgres  │             │ (S3-compat   │ │  later) │
                      │ +pgvector │             │  uploads)    │ └─────────┘
                      └───────────┘             └──────────────┘
```

The Next.js rewrites in `apps/web/next.config.ts` proxy `/api/*`, `/ws`, `/auth/*`, and `/uploads/*` from the public host to `http://multica-backend.flycast:8080`. The backend has no public hostname.

## Prerequisites

- A PlanetScale Postgres database in your preferred region. **pgvector must be available on your plan** — verify with `psql "$DATABASE_URL" -c 'CREATE EXTENSION IF NOT EXISTS vector;'` before continuing.
- A custom domain you control (DNS editable).
- `flyctl` installed (`brew install flyctl`) and `fly auth login` complete.

## First-time setup

1. **Clone and verify**
   ```bash
   git clone https://github.com/multica-ai/multica.git
   cd multica
   ```

2. **Pre-deploy DB checks**
   ```bash
   export DATABASE_URL='postgres://USER:PASS@HOST/DB?sslmode=require'
   psql "$DATABASE_URL" -c 'CREATE EXTENSION IF NOT EXISTS vector;'
   ```
   If this fails, do not continue — pgvector is not on your plan.

3. **Create the apps**
   ```bash
   # Drop the --org flag if you only have one Fly org (the default).
   # Otherwise pass --org <your-org-slug>; list orgs with `fly orgs list`.
   fly apps create multica-backend
   fly apps create multica-web
   ```

4. **Provision Tigris**
   ```bash
   fly storage create -a multica-backend
   ```
   Save the printed bucket name and endpoint URL. `fly storage create` also automatically sets `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` as Fly secrets — those are what Multica's S3 client reads, so no further action is needed for credentials. Multica reads `S3_BUCKET`, `S3_REGION`, and `AWS_ENDPOINT_URL` (not the Tigris defaults `BUCKET_NAME` / `AWS_ENDPOINT_URL_S3`), so add the aliases:
   ```bash
   fly secrets set -a multica-backend \
     S3_BUCKET=<bucket-name-from-tigris> \
     S3_REGION=auto \
     AWS_ENDPOINT_URL=https://fly.storage.tigris.dev
   ```

5. **Set backend secrets** (replace `<your-domain>` and the PlanetScale URL)
   ```bash
   fly secrets set -a multica-backend \
     DATABASE_URL='postgres://USER:PASS@HOST/DB?sslmode=require&pool_max_conns=10' \
     JWT_SECRET="$(openssl rand -hex 32)" \
     FRONTEND_ORIGIN=https://multica.<your-domain> \
     CORS_ALLOWED_ORIGINS=https://multica.<your-domain> \
     COOKIE_DOMAIN=multica.<your-domain> \
     APP_ENV=production \
     ALLOW_SIGNUP=false \
     ALLOWED_EMAILS=you@<your-domain>
   ```

   The `&pool_max_conns=10` query param caps the backend's pgxpool below typical PlanetScale connection limits. If your plan allows more, raise it but always leave headroom for `psql` sessions and migrations.

6. **Deploy** (backend first so its `/readyz` health check is passing before the web app starts proxying traffic to it)
   ```bash
   fly deploy -c fly.backend.toml
   fly deploy -c fly.web.toml
   ```

7. **Custom domain + TLS**
   ```bash
   fly certs create -a multica-web multica.<your-domain>
   ```
   Add the DNS records Fly prints (CNAME to `multica-web.fly.dev` plus the ACME challenge record). Then:
   ```bash
   fly certs show multica.<your-domain> -a multica-web
   ```
   Wait for `Status: Ready`.

8. **Log in for the first time**
   Open `https://multica.<your-domain>` in a browser. Enter your email (the one in `ALLOWED_EMAILS`) and request a code. Until Resend is configured, the code prints to the backend logs:
   ```bash
   fly logs -a multica-backend | grep -i "verification code"
   ```
   Copy the 6-digit code into the browser and submit. Create a workspace.

9. **Point your CLI**
   ```bash
   multica config set server_url https://multica.<your-domain>
   multica config set app_url    https://multica.<your-domain>
   multica login
   multica daemon start
   ```
   The CLI uses the same login flow.

## Day-2 operations

| Action | Command |
|---|---|
| Deploy backend | `fly deploy -c fly.backend.toml` |
| Deploy web | `fly deploy -c fly.web.toml` |
| Tail backend logs | `fly logs -a multica-backend` |
| Tail web logs | `fly logs -a multica-web` |
| SSH | `fly ssh console -a multica-backend` |
| List secrets | `fly secrets list -a multica-backend` |
| Add a teammate | `fly secrets set -a multica-backend ALLOWED_EMAILS=a@x,b@y` (comma-separated) |
| List releases | `fly releases list -a multica-backend` |

## Adding email auth (Resend)

Until Resend is configured, login codes only print to backend logs — fine for solo use, blocks teammates. To enable real emails:

1. Create a Resend account, verify a sender domain (e.g. `mail.<your-domain>`), generate an API key.
2. Set the secrets:
   ```bash
   fly secrets set -a multica-backend \
     RESEND_API_KEY=re_xxx \
     RESEND_FROM_EMAIL=noreply@mail.<your-domain>
   ```
3. The backend restarts automatically. Test with a fresh login.

## Rollback

| Failure | Response |
|---|---|
| Bad backend deploy crashes on startup | `fly releases list -a multica-backend` to find the previous deployment ID, then `fly deploy -c fly.backend.toml --image registry.fly.io/multica-backend:deployment-<old-id>` |
| Bad web deploy 500s on every request | `fly releases list -a multica-web` to find the previous deployment ID, then `fly deploy -c fly.web.toml --image registry.fly.io/multica-web:deployment-<old-id>` |
| Migration fails partway through | **`./migrate down` rolls back ALL migrations** (the binary has no `--target` / `--version` flag — it walks every `.down.sql` file in reverse). Prefer redeploying the previous backend image (see the row above) and letting `./migrate up` settle at the prior version. Only run `./migrate down` if you have a verified backup and are prepared to reprovision. |

## Cost estimate

- Fly: 2× `shared-cpu-1x` @ 512 MB ≈ ~$10-12 / month.
- Tigris: free at low volume.
- PlanetScale: per your plan.
- Resend (when added): free tier covers low volume.

## What is not covered here

- Multi-region deployment (single `iad` region).
- High availability (single machine per app).
- Backend public hostname for direct CLI access (CLI traffic goes through the web proxy).
- Postgres backups (PlanetScale's responsibility).
- Observability beyond `fly logs`.
