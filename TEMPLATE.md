# It's a Plan v1 Issue Tracker with AI Agents (Community) — Railway template spec

## Listing

| Field | Value |
|---|---|
| Name | **It's a Plan v1 Issue Tracker with AI Agents (Community)** (55 chars; `v1` = major of the pinned 1.10.0, per our naming rule; "(Community)" marks the Community edition and keeps it apart from the author's own listing "It's a Plan") |
| Description | Self-hosted Linear alternative where AI agents work issues as teammates. (72 chars) |
| Category | Other (the author's listing uses the same; it is a tracker, not an AI/ML tool) |
| Tags | itsaplan, issue-tracker, project-management, kanban, linear-alternative, ai-agents, mcp, self-hosted |
| Icon | `https://cdn.jsdelivr.net/npm/lucide-static@1.59.0/icons/square-kanban.svg` (neutral Lucide icon, ISC; the project's logo is not used, so the card does not look like the author's listing) |
| Repo | `baranberkay96/itsaplan-railway`, one repo service: `/services/gateway` |
| User input at deploy | **None.** All secrets are generated; the public URL comes from Railway's generated domain. Optional later: `PRIVACY_URL`/`TERMS_URL` (Web), `EMAIL_FROM` (API), `TELEMETRY_DISABLED` (Worker). Mail, AI keys and the Telegram token are set in the app. |

The description is delivered as written: "self-hosted" = the whole stack in the user's Railway project; "Linear alternative" = upstream's own positioning (issue tracker with projects, boards, cycles); "AI agents work issues as teammates" = agents as project members that take assigned or @mentioned issues (internal agents run in the API service; external ones through `@itsaplan/runner`).

## Base market templates

| Template | Active / total projects | Notes |
|---|---|---|
| "It's a Plan" (`its-a-plan`, verified, by the author; `_market/its-a-plan.json`) | 56 / 111 | Pro images `ghcr.io/croffasia/itsaplan-pro-*:latest` with auto-updates; api and web each get a public domain; **requires two custom hostnames** under one registrable domain (two inputs `API_URL`, `APP_URL` with empty defaults). |

Kept from it (wiring only, no text): the Railway Bucket and its variable mapping (`S3_FORCE_PATH_STYLE=false`), `CLIENT_IP_HEADER=x-real-ip`, Railway's Postgres 18 base, `/` and `/login` as healthchecks. Dropped: `WORKER_INTERNAL_TOKEN` and `SERVICE_URL_API` on worker/bot (no longer read by 1.10.0: no match in the tag's source).

## Pinned versions

| Service | Source | Version | Built / released | Checked |
|---|---|---|---|---|
| API | `ghcr.io/croffasia/itsaplan-api:1.10.0@sha256:9f53d060…8c642753` (index; amd64 `sha256:9d5d1dc1…5bda8881`) | 1.10.0 Community (Bun 1.4.4, PostgreSQL 18 client) | image 2026-11-05 00:51 UTC; release published 00:50 UTC | 2026-11-07 |
| Web | `ghcr.io/croffasia/itsaplan-web:1.10.0@sha256:be213d68…4f78b289` (amd64 `sha256:ad767dc2…9a132aca`) | 1.10.0 (Next.js 16.4.2, Node 24.22.2) | 2026-11-05 00:52 UTC | 2026-11-07 |
| Worker | `ghcr.io/croffasia/itsaplan-worker:1.10.0@sha256:840650f1…3688aed0` (amd64 `sha256:703a1279…8ec60acd`) | 1.10.0 (`BUILD_NAME=community`, Bun 1.4.4) | 2026-11-05 00:51 UTC | 2026-11-07 |
| Telegram Bot | `ghcr.io/croffasia/itsaplan-bot:1.10.0@sha256:d0ab66c7…8aad5780` (amd64 `sha256:13de32d6…28f6f386`) | 1.10.0 (grammY, Bun 1.4.4) | 2026-11-05 00:51 UTC | 2026-11-07 |
| Gateway | repo build `FROM caddy:2.11.9-alpine@sha256:0bbb055b…46559fee` (amd64 `sha256:fc9cbc10…4efe6424`) + `services/gateway/Caddyfile` | Caddy v2.11.9 | release 2026-11-05, image 2026-11-05 | 2026-11-07 |
| Postgres | `ghcr.io/railwayapp-templates/postgres-ssl:18` (Railway base; index `sha256:4fc833b4…7217c863` on 2026-11-07) | PostgreSQL 18 | — | 2026-11-07 |

- Upstream: `croffasia/itsaplan`, AGPL-3.0, release `v1.10.0` = commit `6c7c2fe2cb9d810d29c68e054de25876b47434f8` (2026-11-05), the newest release (`latest` points to the same digests today). The images are built by `.github/workflows/publish-images.yml` from the release tag. The `COPY . .` layer of the api and worker images (3,383 files) is byte-identical to the tag's tree; `_upstream/croffasia__itsaplan` is identical to the tag (3,400 of 3,401 blobs; one not present locally).
- All four It's a Plan images are multi-arch (amd64 + arm64) and pull anonymously from GHCR. Digest-pinned GHCR image sources are used by other packages in this repo (`11-firecrawl`, `37-cap`).
- **One version per deployment:** API, Web, Worker and Telegram Bot must run the same release. API applies the migrations; Worker and Bot wait until the database holds every migration their own image ships (`wait-for-migrations.ts`, 175 migrations at 1.10.0, last `0174_agent_schedules_permission`).

## Architecture

```
Browser / MCP client / runner / Git webhooks
        │ HTTPS (Railway edge, generated *.up.railway.app or custom domain)
        ▼
Gateway (Caddy :8080) ── /api/* (strip /api) ──────────────► API (Bun :3000) ── volume /backups
        │              ── /.well-known/oauth-*, openid-* ──►   │  migrate.ts + pg_dump at start
        └──────────────── everything else ────────────────► Web (Next.js :3001)
                                                              │  /media → http://api.railway.internal:3000
API, Worker, Telegram Bot ── postgres:// ──► Postgres (volume /var/lib/postgresql/data)
API, Worker ── S3 (virtual-hosted, https://{bucket}.t3.storageapi.dev) ──► Railway Bucket
Worker ── webhooks, Resend/SMTP, Telegram, telemetry (outbound)
Telegram Bot ── long polling (outbound to api.telegram.org)
```

## Design decisions (verified in the 1.10.0 sources and images)

### Why the author's template needs a custom domain, and why this one does not

- `docs/railway.md`: `*.up.railway.app` is on the Public Suffix List. With api and web on two such hostnames, they are two sites; the api's session cookie cannot be scoped to a shared parent (`.up.railway.app` is a public suffix, browsers refuse it), so sign-in "returns 200 and then bounces straight back to the login page". The author's fix is two hostnames under the user's own domain, with the cookie on their parent.
- Upstream also supports the api **under a path of the web's host** since 1.5.0 (#519, `docs/self-hosting.md`, "Where the api lives"): `API_URL=https://app.example.com/api`, the proxy strips `/api` and also sends `/.well-known/oauth-authorization-server*`, `/.well-known/oauth-protected-resource*` and `/.well-known/openid-configuration*` to the api. In `packages/auth/src/index.ts`, `sameHost` (APP_URL and API_URL hostnames equal) makes the cookie host-only, and over HTTPS its names carry the `__Host-` prefix; `baseURL` is reduced to the origin and `withPublicOrigin`/`withPublicDiscoveryUrls` put the path back into email links, OAuth redirect URIs and discovery documents. The web client spells out `{API_URL}/api/auth` (`apps/web/src/lib/auth-client.ts`), and the web middleware finds `__Host-better-auth.*` cookies (`apps/web/src/proxy.ts`).
- A host-only cookie on one `*.up.railway.app` hostname is an ordinary first-party cookie, so this layout works on Railway's generated domain. The template therefore adds the **Gateway** (Caddy) as the only public service: `APP_URL=https://${{RAILWAY_PUBLIC_DOMAIN}}` and `API_URL=${{APP_URL}}/api`, both defined once on Gateway and referenced by API and Web.
- No conflicts with app routes: `api` is a reserved team slug (`apps/api/src/modules/teams/ref.ts`), the web app has no `/api` or `/.well-known` routes, and team slugs must start with a letter, so `/_gateway/health` cannot shadow a page.
- Caddy details: `header_up X-Forwarded-Proto https` (Railway's edge talks plain HTTP to the container; Caddy would otherwise send `http`); X-Forwarded-Host stays the public host, so Next.js Server Action origin checks match; `X-Real-IP` from Railway's edge passes through unchanged for the api's rate limit; responses with `text/event-stream` or unknown length are flushed immediately by Caddy itself (MCP and agent chat streams); upstream keep-alive is shorter than the apps' idle timeouts (Bun/Elysia 30 s, Node 5 s) to avoid reusing closed connections; `lb_try_duration 10s` covers an API restart. `caddy validate` runs at build time.
- Custom domain: optional. Add it to Gateway and set `APP_URL` there; API and Web follow by reference. Passkeys (RP ID = APP_URL host) and OAuth redirect URIs follow the address, as documented in the README.

### Migrations at API start, with a durable pre-migration dump

- The API image's default CMD is `migrate.ts && index.ts`. `migrate.ts` takes an advisory lock and, when an existing database has pending migrations, `pg_dump`s it to `BACKUP_DIR` (`/backups`) first and refuses to migrate if the dump fails; dumps older than `BACKUP_RETENTION_DAYS` (30) are pruned.
- A Railway pre-deploy command runs in a separate container where **volumes are not mounted** (Railway docs), so a dump taken there is lost, as it is in the author's template (no volume) and as upstream's own `docs/dev/deploy-stack.md` notes for Railway. This template runs the migration in the API's start command and mounts a volume at `/backups`, like upstream's Compose (`db-backups`) and Helm (`api.backups.persistence`). Upstream's "what changed" screen then points at a dump that exists.
- `migrate.ts` waits only 15 × 2 s for Postgres; the start command wraps it in a retry loop (20 attempts, 10 s apart, ≈13 min worst case) before `exec`ing the server, so the api survives a slow first Postgres boot (Rule 8.5). The healthcheck timeout is 1200 s (Rule 8.5/8.6). A migration that keeps failing exits non-zero after 20 attempts.
- Cost of this choice: API has a volume, so it runs one replica and Railway stops it before starting the new deployment (seconds of downtime per deploy; Gateway retries for 10 s). Acceptable for a team tracker; documented.
- Worker and Telegram Bot start through upstream's `wait-for-migrations.ts` (the Helm chart's init container), present in both images with its `postgres` dependency (checked in the image layers). If the script itself cannot run, they start anyway (the worker tolerates missing tables, per its Dockerfile).

### Storage: Railway Bucket

- `packages/storage` uses AWS SDK v3 `S3Client` with `endpoint`, `region` (`S3_REGION`), `forcePathStyle: S3_FORCE_PATH_STYLE !== 'false'`. Railway Buckets use virtual-hosted URLs by default, so `S3_FORCE_PATH_STYLE=false` (also upstream's `.env.example` advice for Railway buckets, and the author's template). The SDK puts the bucket into the hostname of `https://t3.storageapi.dev`. Region is `${{Bucket.REGION}}` (`auto`).
- The app only puts, gets and deletes objects; no bucket creation step is needed (upstream's `minio-init` exists only for RustFS). RustFS, the Compose store, is not used.
- Downloads stream through the api (`/media` on Web proxies to the api privately), so the bucket never needs to be public.

### Mail

- Upstream sends mail through the provider configured in god mode (**Integrations → Email provider**): SMTP (incl. Microsoft 365 OAuth) or **Resend** (HTTPS API). Railway allows outbound SMTP only on Pro (Rule 8.11), so the README directs Hobby users to Resend. The From address can be set in god mode; `EMAIL_FROM` (API, referenced by Worker) is an optional fallback; Resend refuses to send without one.
- No mail service is part of the template; without a provider the instance works, but invites, password resets and email notifications are not sent.

### Accounts and security

- No default credentials. The first account registered becomes the instance owner (`role: 'god'` when the user table is empty, `packages/auth/src/index.ts`); registration defaults to `open` (`packages/auth/src/instance.ts`). The README tells the deployer to register first and then pick open / invite / closed in god mode. The generated `*.up.railway.app` name is not published anywhere by the template.
- Secrets: `BETTER_AUTH_SECRET`, `APP_ENCRYPTION_KEY` (any string; SHA-256 → AES-256-GCM key, `packages/crypto`) = `secret(48)`; Postgres password `secret(32)` letters only.
- Only Gateway has a domain. API, Web, Worker, Bot and Postgres are private; no TCP proxies. Everything the api serves is meant to be public upstream (it is its own public origin in every upstream layout); there are no internal-only api routes in 1.10.0.
- `CLIENT_IP_HEADER=x-real-ip` (upstream's Railway recommendation). Caddy passes `X-Real-IP` on as Railway's edge set it; the api is not reachable from the internet except through the Gateway.

### Networking (Rules 8.1, 8.2, 8.13, 8.14)

- Each healthchecked service sets `PORT` to its listener: Gateway 8080 (`/_gateway/health`, answered by Caddy alone, so it never waits on the apps), Web 3001 (`/login`, renders without a session and without the api), API 3000 (`/`, the api's liveness route, only reachable after the migrations).
- Caddy (Go) listens dual-stack. Web binds `HOSTNAME=0.0.0.0` (upstream default; Next.js builds its internal URL from it) and the api binds Bun's documented default `0.0.0.0` (Elysia passes no hostname; upstream has no setting for it). Both are IPv4-only, so Rule 8.14's `::` is not available without patching upstream. Their only callers on the private network are Caddy (Go dialer, RFC 6555 fallback) and the Web server (Node 24, `autoSelectFamily` on by default), which fall back to IPv4, available in all new Railway environments.
- No service resolves its own private name at boot; Caddy resolves upstreams per connection, so it starts before API and Web exist.

### Telemetry

Upstream's worker sends one anonymous snapshot a day (TELEMETRY.md lists every field); on by default in Compose, off by default in upstream's Helm values. The template keeps the Compose/docs default (the author's software, documented behaviour) and exposes `TELEMETRY_DISABLED` on Worker, mentioned in the README.

### Left out on purpose

- RustFS and `minio-init` (Bucket instead), the separate `migrate` one-shot service (Rule 8.6; runs in API's start), Pro images and licence key, image auto-updates on `latest` (pinned digests instead), `COOKIE_DOMAIN` (must stay unset for the single-origin layout), `GIT_WEBHOOK_BASE_URL` (only for private instances), `TRUSTED_PROXIES` (not needed with `CLIENT_IP_HEADER`).

## Services

Values are entered exactly as shown (Railway resolves `${{...}}`).

### Gateway

The only public service. Caddy routes `/api/*` (prefix removed) and the OAuth discovery documents to API, everything else to Web.

- **Source:** repo `baranberkay96/itsaplan-railway`, root directory `/services/gateway` (Dockerfile build)
- **Start command:** image default
- **Healthcheck:** `/_gateway/health` on `PORT` 8080 (timeout 120 s)
- **Public domain:** yes, target port `8080` · **TCP proxy:** none
- **Volume:** none
- **Restart:** ALWAYS · **Replicas:** 1 · **Cron:** none · **Pre-deploy:** none

| Variable | Value | Description |
|---|---|---|
| `GATEWAY_CADDY_IMAGE` | `caddy:2.11.9-alpine@sha256:0bbb055bda6793c999bde78b1b5ce3e89f829fb031af2cf50056ea1246559fee` | Caddy image this service's Docker build starts FROM (passed as a build argument). Pinned by tag and digest. |
| `PORT` | `8080` | Port Caddy listens on (public domain target; Railway health-checks /_gateway/health here). |
| `APP_URL` | `https://${{RAILWAY_PUBLIC_DOMAIN}}` | Public origin of It's a Plan; API and Web read it from here. After adding a custom domain to this service, set https://{your-domain} and deploy. |
| `API_URL` | `${{APP_URL}}/api` | Public URL of the api: a path on the same host, routed by this gateway. Keep it as APP_URL plus /api. |
| `ITSAPLAN_API_UPSTREAM` | `${{API.RAILWAY_PRIVATE_DOMAIN}}:${{API.PORT}}` | API service on the private network (host:port) for /api/* and the OAuth discovery documents. |
| `ITSAPLAN_WEB_UPSTREAM` | `${{Web.RAILWAY_PRIVATE_DOMAIN}}:${{Web.PORT}}` | Web service on the private network (host:port) for every other path. |
| `RAILWAY_DEPLOYMENT_DRAINING_SECONDS` | `30` | Seconds open requests get to finish when the gateway is redeployed. |

### Web

Next.js 16 web app (standalone server, Node 24). Reads `API_URL` at startup and hands it to the browser.

- **Source:** image `ghcr.io/croffasia/itsaplan-web:1.10.0@sha256:be213d6861f66ac68320db2a1481c050bc811a740d558f5ebde576b34f78b289`
- **Start command:** image default
- **Healthcheck:** `/login` on `PORT` 3001 (timeout 300 s)
- **Public domain:** none · **TCP proxy:** none
- **Volume:** none
- **Restart:** ALWAYS · **Replicas:** 1 · **Cron:** none · **Pre-deploy:** none

| Variable | Value | Description |
|---|---|---|
| `PORT` | `3001` | Port the Next.js server listens on (Railway health-checks /login here; the Gateway proxies to it). |
| `HOSTNAME` | `0.0.0.0` | Bind address of the Next.js server (upstream default). |
| `NODE_ENV` | `production` | Runtime mode. Keep production. |
| `API_URL` | `${{Gateway.API_URL}}` | Public api URL the browser calls, read at startup (set on Gateway). |
| `SERVICE_URL_API` | `http://${{API.RAILWAY_PRIVATE_DOMAIN}}:${{API.PORT}}` | API on the private network; the web server streams avatars and attachments (/media) from it. |
| `PRIVACY_URL` | `(empty)` | Optional. Privacy policy URL shown on the sign-in screens (Google sign-in needs one). Empty hides the notice. |
| `TERMS_URL` | `(empty)` | Optional. Terms of service URL shown on the sign-in screens. Empty hides the notice. |
| `RAILWAY_DEPLOYMENT_DRAINING_SECONDS` | `15` | Seconds open requests get to finish when Web is redeployed. |

### API

Elysia on Bun: REST API, better-auth, MCP, SCIM, Git webhooks, agent runs and background sweeps. Migrates the database (with a dump) before it starts serving.

- **Source:** image `ghcr.io/croffasia/itsaplan-api:1.10.0@sha256:9f53d060db6594b2e51bc224183e8e8c60e544a1fd65dcd451c053568c642753`
- **Start command:** `/bin/sh -c 'i=1; until bun run packages/db/src/migrate.ts; do if [ "$i" -ge 20 ]; then echo "[railway] migrations failed 20 times, giving up (see the errors above)" >&2; exit 1; fi; echo "[railway] migration attempt $i failed (Postgres still starting?), retrying in 10 s" >&2; i=$((i+1)); sleep 10; done; exec bun run apps/api/src/index.ts'`
- **Healthcheck:** `/` on `PORT` 3000 (timeout 1200 s)
- **Public domain:** none · **TCP proxy:** none
- **Volume:** `/backups`
- **Restart:** ALWAYS · **Replicas:** 1 · **Cron:** none · **Pre-deploy:** none

| Variable | Value | Description |
|---|---|---|
| `PORT` | `3000` | Port Railway health-checks (GET / answers status ok). Must equal API_PORT. |
| `API_PORT` | `${{PORT}}` | Port the api listens on. |
| `NODE_ENV` | `production` | Runtime mode. Keep production (secure cookies). |
| `DATABASE_URL` | `${{Postgres.DATABASE_URL}}` | Postgres connection string (private network). |
| `BETTER_AUTH_SECRET` | `${{secret(48, "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789")}}` | Signs sessions and tokens; generated at deploy. Changing it signs everyone out. |
| `APP_ENCRYPTION_KEY` | `${{secret(48, "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789")}}` | Encrypts stored credentials (AI, mail, Telegram, Git) at rest; generated at deploy. Changing it makes them unreadable. |
| `APP_URL` | `${{Gateway.APP_URL}}` | Public web origin (CORS, cookies, passkeys, email links). Set on Gateway. |
| `API_URL` | `${{Gateway.API_URL}}` | Public api URL (auth handler, OAuth issuer, MCP resource, email links). Set on Gateway. |
| `CLIENT_IP_HEADER` | `x-real-ip` | Header with the client address for the sign-in rate limit; Railway's edge sets X-Real-IP and the Gateway passes it on. |
| `S3_ENDPOINT` | `${{Bucket.ENDPOINT}}` | Railway Bucket S3 endpoint for attachments. |
| `S3_BUCKET` | `${{Bucket.BUCKET}}` | Bucket name used by the S3 API. |
| `S3_ACCESS_KEY_ID` | `${{Bucket.ACCESS_KEY_ID}}` | Bucket access key. |
| `S3_SECRET_ACCESS_KEY` | `${{Bucket.SECRET_ACCESS_KEY}}` | Bucket secret key. |
| `S3_REGION` | `${{Bucket.REGION}}` | Bucket region for request signing. |
| `S3_FORCE_PATH_STYLE` | `false` | false = virtual-hosted URLs, as Railway Buckets use. Set true only for a bucket whose Credentials tab says path-style. |
| `BACKUP_DIR` | `/backups` | Where the database dump taken before each upgrade's migrations is written (this service's volume). |
| `BACKUP_RETENTION_DAYS` | `30` | Days a pre-migration dump is kept; older ones are deleted at the next start. |
| `SKIP_PRE_MIGRATION_BACKUP` | `(empty)` | Optional. 1 = apply migrations without the dump (only if you back up Postgres another way). |
| `EMAIL_FROM` | `(empty)` | Optional. Default From address for instance mail; the From field in god mode wins. Resend needs a verified sender. |
| `SSRF_ALLOWED_HOSTS` | `(empty)` | Optional. Comma-separated private hostnames the api and worker may reach (e.g. a self-hosted Git server). |
| `RAILWAY_DEPLOYMENT_DRAINING_SECONDS` | `30` | Seconds running requests get to finish when the API is redeployed. |

### Worker

Webhook and notification delivery (email, Telegram), agent schedules, imports, telemetry. No HTTP port.

- **Source:** image `ghcr.io/croffasia/itsaplan-worker:1.10.0@sha256:840650f1dc8cec4d82cf8cdc6fa5cd43aa82df1fc9a9f9213358e3c33688aed0`
- **Start command:** `/bin/sh -c 'bun run packages/db/src/wait-for-migrations.ts || echo "[railway] could not check the database migrations, starting anyway" >&2; exec bun run apps/worker/src/index.ts'`
- **Healthcheck:** none (no HTTP port)
- **Public domain:** none · **TCP proxy:** none
- **Volume:** none
- **Restart:** ALWAYS · **Replicas:** 1 · **Cron:** none · **Pre-deploy:** none

| Variable | Value | Description |
|---|---|---|
| `NODE_ENV` | `production` | Runtime mode. Keep production. |
| `DATABASE_URL` | `${{Postgres.DATABASE_URL}}` | Postgres connection string (private network). |
| `APP_ENCRYPTION_KEY` | `${{API.APP_ENCRYPTION_KEY}}` | Same key as the API (decrypts the stored mail and Telegram credentials). |
| `S3_ENDPOINT` | `${{API.S3_ENDPOINT}}` | Same bucket as the API (imported attachments). |
| `S3_BUCKET` | `${{API.S3_BUCKET}}` | Same bucket as the API. |
| `S3_ACCESS_KEY_ID` | `${{API.S3_ACCESS_KEY_ID}}` | Same bucket as the API. |
| `S3_SECRET_ACCESS_KEY` | `${{API.S3_SECRET_ACCESS_KEY}}` | Same bucket as the API. |
| `S3_REGION` | `${{API.S3_REGION}}` | Same bucket as the API. |
| `S3_FORCE_PATH_STYLE` | `${{API.S3_FORCE_PATH_STYLE}}` | Same setting as the API. |
| `EMAIL_FROM` | `${{API.EMAIL_FROM}}` | Same default From address as the API (notification email). |
| `SSRF_ALLOWED_HOSTS` | `${{API.SSRF_ALLOWED_HOSTS}}` | Same list as the API (webhook delivery). |
| `TELEMETRY_DISABLED` | `(empty)` | Optional. Upstream sends one anonymous usage snapshot a day (TELEMETRY.md); 1 turns it off. |
| `RAILWAY_DEPLOYMENT_DRAINING_SECONDS` | `30` | Seconds in-flight deliveries get to finish on redeploy. |

### Telegram Bot

Long polling for the instance's Telegram bot (account linking). Idles until a bot token is saved in god mode. Must stay at one replica.

- **Source:** image `ghcr.io/croffasia/itsaplan-bot:1.10.0@sha256:d0ab66c7454d003064ea455299db740746f6c6a53edcb375429881c88aad5780`
- **Start command:** `/bin/sh -c 'bun run packages/db/src/wait-for-migrations.ts || echo "[railway] could not check the database migrations, starting anyway" >&2; exec bun run apps/bot/src/index.ts'`
- **Healthcheck:** none (no HTTP port)
- **Public domain:** none · **TCP proxy:** none
- **Volume:** none
- **Restart:** ALWAYS · **Replicas:** 1 · **Cron:** none · **Pre-deploy:** none

| Variable | Value | Description |
|---|---|---|
| `NODE_ENV` | `production` | Runtime mode. Keep production. |
| `DATABASE_URL` | `${{Postgres.DATABASE_URL}}` | Postgres connection string (private network). |
| `APP_ENCRYPTION_KEY` | `${{API.APP_ENCRYPTION_KEY}}` | Same key as the API (decrypts the bot token set in god mode). |
| `RAILWAY_DEPLOYMENT_DRAINING_SECONDS` | `10` | Seconds the bot gets to end its Telegram long poll on redeploy. |

### Postgres

Railway base Postgres 18 (`_railway-base/postgres.json`), all application data.

- **Source:** image `ghcr.io/railwayapp-templates/postgres-ssl:18`
- **Start command:** image default
- **Healthcheck:** none (TCP)
- **Public domain:** none · **TCP proxy:** none
- **Volume:** `/var/lib/postgresql/data`
- **Restart:** ON_FAILURE (10) · **Replicas:** 1 · **Cron:** none · **Pre-deploy:** none

| Variable | Value | Description |
|---|---|---|
| `PGDATA` | `/var/lib/postgresql/data/pgdata` | Data directory inside the volume (Railway base layout). |
| `PGHOST` | `${{RAILWAY_PRIVATE_DOMAIN}}` | Private hostname of this database, used by the Data panel. |
| `PGPORT` | `5432` | Postgres port. |
| `PGUSER` | `${{POSTGRES_USER}}` | User for the Data panel and the other services. |
| `PGPASSWORD` | `${{POSTGRES_PASSWORD}}` | Password for the Data panel and the other services. |
| `PGDATABASE` | `${{POSTGRES_DB}}` | Database for the Data panel and the other services. |
| `POSTGRES_USER` | `postgres` | Superuser created on first boot. |
| `POSTGRES_PASSWORD` | `${{ secret(32, "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ") }}` | Superuser password, generated at deploy (letters only, safe in URLs). Changing it later needs ALTER USER too. |
| `POSTGRES_DB` | `railway` | Database created on first boot; It's a Plan keeps all its tables here. |
| `DATABASE_URL` | `postgresql://${{PGUSER}}:${{POSTGRES_PASSWORD}}@${{RAILWAY_PRIVATE_DOMAIN}}:5432/${{PGDATABASE}}` | Private connection string used by API, Worker and Telegram Bot. |
| `SSL_CERT_DAYS` | `820` | Validity of the self-signed TLS certificate. |
| `RAILWAY_DEPLOYMENT_DRAINING_SECONDS` | `60` | Time Postgres gets to shut down cleanly on redeploy. |

### Bucket

- A Railway Bucket named `Bucket` (`buckets` in template.json). API and Worker read `BUCKET`, `ENDPOINT`, `REGION`, `ACCESS_KEY_ID`, `SECRET_ACCESS_KEY` by reference. No setup step: the app only puts, gets and deletes objects (attachments, avatars, agent skill files, imports); it never creates or lists buckets.

## Improvements over market templates

Compared with the author's "It's a Plan" template:

- **No custom domain needed.** One public origin through the Gateway; sign-in works on the generated `*.up.railway.app` domain right after deploy. The author's template needs two hostnames on a domain the user owns, two DNS records and two manual inputs before anyone can sign in. A custom domain stays optional (one variable).
- **Zero required inputs** (the author's has two empty required-looking fields, `API_URL` and `APP_URL`).
- **Pinned release:** 1.10.0 by tag and digest, no `latest` auto-updates that can roll a breaking pre-1.0-style change (upstream: "Expect breaking changes before the first stable release") into a running instance at night.
- **Durable pre-migration dumps** on a volume (`/backups`) instead of the container filesystem, so upgrades keep upstream's rollback safety net.
- **Startup ordering:** API retries the migration until Postgres is up; Worker and Bot wait for the migrations (upstream's Helm pattern) instead of starting against an old schema.
- **Smaller attack surface:** only the Gateway is public; the API and Web services have no domains of their own.
- **Community edition** (AGPL images), no Pro images that run as Community without a licence.
- Drops variables 1.10.0 no longer reads (`WORKER_INTERNAL_TOKEN`, `SERVICE_URL_API` on worker/bot); every variable has a description; the README covers mail on Hobby (Resend), first-owner registration, telemetry, backups, upgrades and costs.

## Scaling

- **Gateway, Web:** stateless; replicas are fine.
- **API:** one replica (it owns the `/backups` volume; Railway does not attach a volume to replicas). Scale vertically; internal agent runs execute here (LLM calls, Mastra memory in Postgres). Upstream supports several api replicas (advisory lock, `FOR UPDATE SKIP LOCKED` queues): drop the volume and set `SKIP_PRE_MIGRATION_BACKUP=1` only if you back up Postgres another way.
- **Worker:** one replica is enough for a team; it polls every 2–3 s. Tuning variables (`WEBHOOK_*`, `NOTIFICATION_*`, `IMPORT_POLL_INTERVAL_MS`) are documented in upstream's `.env.example`.
- **Telegram Bot:** exactly one replica (Telegram gives each `getUpdates` to one caller). A redeploy briefly overlaps old and new process; the old one stops its poll on SIGTERM.
- **Postgres:** vertical; Railway volume backups recommended.

## Resource estimate

Prices: RAM $10/GB-month, vCPU $20/vCPU-month, volume $0.15/GB-month, egress $0.05/GB, Bucket storage $0.015/GB-month (bucket operations and bucket egress free). **Not measured** (no container run this round); estimated from the runtimes (Bun, Node.js/Next.js, Caddy, Postgres) and the polling loops each process runs.

| Service | Idle RAM / vCPU | Typical RAM / vCPU | Notes |
|---|---|---|---|
| Gateway | 0.03 GB / 0.002 | 0.05 GB / 0.01 | Caddy |
| Web | 0.18 GB / 0.005 | 0.35 GB / 0.05 | Next.js SSR |
| API | 0.30 GB / 0.02 | 0.60 GB / 0.15 | Bun; background loops every 1–2 s; agent runs; volume < 0.1 GB idle |
| Worker | 0.12 GB / 0.01 | 0.18 GB / 0.03 | polling every 2–3 s |
| Telegram Bot | 0.08 GB / 0.002 | 0.10 GB / 0.005 | idle until a token is set |
| Postgres | 0.12 GB / 0.005 | 0.40 GB / 0.05 | volume ~0.3 GB idle, ~2 GB typical |

- **Idle ≈ $10/month** (0.83 GB × $10 + 0.045 vCPU × $20 + 0.4 GB volumes × $0.15).
- **Typical ≈ $22–30/month** (1.7 GB × $10 + 0.3 vCPU × $20 + ~2.5 GB volumes + ~5 GB bucket + a few GB egress) for a team of 5–20 with a few agents.
- AI model calls are billed by the user's provider. The Hobby plan's $5 included usage covers part of it.
- **Minimum plan:** Hobby. Trial (1 GB RAM and 0.5 GB volume per service) can run an evaluation; Free's 0.5 GB per service is tight for API during agent runs. Bucket availability per plan is not stated in Railway's docs.

## Verification checklist

1. **Pulls and build:** API, Web, Worker, Telegram Bot pull from GHCR by digest. Gateway build log ends with Caddy's `Valid configuration`.
2. **API start:** logs show `⏳ Running migrations...` and `✅ Migrations applied` (no backup on a fresh database), then `🦊 API running at ...:3000`; healthcheck `/` 200 (`{"name":"It's a Plan api","status":"ok"}`).
3. **Worker / Bot:** `✅ Database is migrated`, then `[worker] worker starting` / `[bot] bot service starting`.
4. **Gateway routing:** `https://{gateway-domain}/_gateway/health` → `ok`; `https://{gateway-domain}/api` → the api JSON above; `https://{gateway-domain}/login` → sign-in page; `https://{gateway-domain}/api/docs` → OpenAPI reference.
5. **Cookie on the generated domain (the main risk):** register the first account at `https://{gateway-domain}/register`; it lands in the app (not back on /login); the browser holds `__Host-better-auth.session_token` for that host only; reload keeps the session; god mode is visible.
6. **Attachments → Bucket:** upload a file to an issue, open it again (served via `/media/...`), and see the object in the Bucket's file browser. A failure like `PermanentRedirect`/`NoSuchBucket` points at the path style.
7. **Worker job:** create a webhook to a request bin on a project and change an issue; the delivery log shows a delivery. Optionally set Resend in god mode and send a test email.
8. **MCP discovery:** `curl -i https://{gateway-domain}/api/mcp` → 401 with `WWW-Authenticate: Bearer resource_metadata="https://{gateway-domain}/api/.well-known/oauth-protected-resource/mcp"`; that URL and `https://{gateway-domain}/.well-known/oauth-authorization-server/api` return JSON whose endpoints start with `https://{gateway-domain}/api/api/auth/`.
9. **Rate limit IP:** API logs do not show `Rate limiting could not determine a client IP` after a sign-in.
10. **Redeploy everything:** session, issues and attachments persist; API logs no new migrations and no backup.
11. **Upgrade path (dry):** with data present, a redeploy of the same image writes no dump (nothing pending); `ls /backups` in the API shell works (volume mounted).
12. **Private services:** API, Web, Worker, Bot, Postgres have no public domain or TCP proxy.

## Open questions

- **Not deployed yet; no Docker build or container run** (instruction). Verified statically: GHCR manifests, configs and source/dependency layers of all four images, the v1.10.0 source, Caddy v2.11.9 source for every directive used, `sh -n` and stub runs of the three start commands, JSON, reference lint, `to_serialized.py`.
- **Cookie flow end to end** on `*.up.railway.app` behind Railway's edge plus Caddy is the key unverified behaviour (upstream documents and implements the layout; nobody here has run it on Railway). Fallback if it fails: the author's two-hostname layout with a custom domain.
- **Railway's edge headers:** `X-Real-IP` set (and overwritten) by the edge is taken from upstream's Railway doc, not checked here. `X-Forwarded-Proto` is forced to `https` by the Gateway regardless.
- **IPv4-only listeners:** API (Bun's documented default `0.0.0.0`; Elysia passes no hostname) and Web (`HOSTNAME=0.0.0.0`). Caddy and Node fall back to IPv4, which new Railway environments provide; a legacy IPv6-only environment would fail (not a template deploy target). Watch the first deploy for `connection refused` on `[fd12:...]` in Gateway logs.
- **API downtime per deploy** (volume → stop before start) is a deliberate trade for durable dumps; long `pg_dump`s on large databases could approach the 1200 s healthcheck window during an upgrade.
- **Telegram Bot overlap** on redeploy may log one `409 Conflict` from Telegram; grammY retries.
- **Bucket on Trial/Free plans** is not documented by Railway.
- **Upstream pace:** 1.5 → 1.10 in about a month, still labelled pre-stable. Re-check `docs/self-hosting.md` ("Where the api lives") and the env list on every bump.
