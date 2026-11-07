# Deploy and Host It's a Plan with Railway

[![Deploy on Railway](https://railway.com/button.svg)](https://railway.com/deploy/its-a-plan-v1-issue-tracker-with-ai-agen?referralCode=20H68q)

It's a Plan is an open-source (AGPL-3.0), self-hosted issue tracker in the spirit of Linear and Plane, where AI agents join projects as members and work on issues next to people. This community template runs the Community edition 1.10.0 from the project's published images, with Postgres, a Railway Bucket and one public URL that works without a custom domain.

## About Hosting It's a Plan

Upstream runs four processes (api, web app, worker, Telegram bot) beside Postgres and an S3 store. Self-hosting it means migrations with a database dump before each upgrade, a bucket for attachments, mail, and a session cookie the web app and the api must share. Railway's generated `*.up.railway.app` hostnames are on the Public Suffix List, so an api and a web app on two of them cannot share that cookie, which is why other setups ask for a domain of your own. This template puts a small Caddy gateway in front instead: web app and api share one origin, with the api under `/api`, a layout upstream supports. Sign-in works on the generated domain as soon as the deploy is green. Secrets are generated, no input is required, and only the gateway is public.

## Common Use Cases

- An issue tracker for a team that wants its projects in its own database, without per-seat fees: boards, cycles, docs, dashboards
- Assigning issues to AI agents: internal agents on your own model keys, or coding agents on your machine through `@itsaplan/runner`
- Working on the tracker from Claude, Cursor or other MCP clients, and from scripts over the REST API and webhooks
- Linking pull requests from GitHub, GitLab, Gitea, Forgejo or Bitbucket to issues
- Moving off Linear, Plane or Trello with the built-in importers

## Dependencies for It's a Plan Hosting

- It's a Plan 1.10.0 Community images: `ghcr.io/croffasia/itsaplan-api`, `-web`, `-worker`, `-bot` (pinned by digest)
- PostgreSQL 18 (Railway's `postgres-ssl` image) for all data
- A Railway Bucket (S3-compatible) for attachments
- Caddy 2.11 as the gateway
- Set in the app after sign-in, all optional: a mail provider, AI model provider keys, a Telegram bot token

### Deployment Dependencies

- Source at the pinned release: https://github.com/croffasia/itsaplan/tree/v1.10.0
- Release notes: https://github.com/croffasia/itsaplan/releases/tag/v1.10.0
- Self-hosting guide: https://github.com/croffasia/itsaplan/blob/v1.10.0/docs/self-hosting.md
- Coding agent runner: https://github.com/croffasia/itsaplan/blob/v1.10.0/docs/runner.md
- Telemetry: https://github.com/croffasia/itsaplan/blob/v1.10.0/TELEMETRY.md
- Railway Buckets: https://docs.railway.com/storage-buckets
- Caddy reverse proxy: https://caddyserver.com/docs/caddyfile/directives/reverse_proxy
- Resend (mail over HTTPS): https://resend.com/docs

### Implementation Details

| Service | Role | Public |
|---|---|---|
| Gateway | Caddy: `/api/*` to API (prefix removed), OAuth discovery to API, everything else to Web | **yes** |
| Web | Next.js web app | no |
| API | REST API, sign-in, MCP at `/mcp`, agent runs; migrates the database at start | no |
| Worker | Webhooks, email and Telegram notifications, agent schedules, imports | no |
| Telegram Bot | Long polling for the instance bot; idles until a bot token is set | no |
| Postgres | All application data | no |
| Bucket | Attachments, avatars, imported files | no |

**First login**

1. Wait until every service is green (a few minutes on the first deploy).
2. Open the Gateway's URL and register. **The first account becomes the instance owner** (god mode), so register right away.
3. In god mode under **Authentication**, set who may sign up to **invite only** (or closed). The default is open: anyone who finds the URL can create an account.
4. Mail for invites, password resets and notifications: god mode, **Integrations, Email provider**. Railway allows outbound SMTP only on the Pro plan, so on Hobby pick **Resend**, which sends over HTTPS, and enter a sender address verified at Resend in **From**.
5. AI model keys and the Telegram bot token are also set in the app, not in variables.

**Addresses**

- App: `https://{gateway-domain}`
- API: `https://{gateway-domain}/api`, OpenAPI reference at `/api/docs`
- MCP server: `https://{gateway-domain}/api/mcp` (OAuth sign-in or a personal API key)

**Custom domain (optional)**

Add it to the Gateway service (Settings, Networking, port 8080), set `APP_URL` on Gateway to `https://{your-domain}`, and deploy the staged changes; API and Web read it from there. Then use only that address. Sessions, passkeys and OAuth redirect URIs follow `APP_URL`; passkeys created on the old address stop working, and OIDC or Google redirect URIs must be updated at the provider.

**Storage and backups**

Attachments live in the Bucket (virtual-hosted URLs, `S3_FORCE_PATH_STYLE=false`). When an upgrade has migrations to apply, the API first writes a `pg_dump` to its `/backups` volume and stops if the dump fails. Dumps are kept 30 days (`BACKUP_RETENTION_DAYS`). To roll back, restore one from the API service's shell with `pg_restore --clean --if-exists --no-owner -d "$DATABASE_URL" /backups/{file}.dump`, then deploy the previous images. Enable Railway's volume backups on Postgres as well.

**Scaling**

Web and Gateway are stateless and can run replicas. API keeps its dumps on a volume, so it runs one replica and restarts on each deploy; the gateway holds requests for up to 10 seconds meanwhile. Give API more memory if many agents run at once. Telegram Bot must stay at one replica, because Telegram hands each update to a single poller.

**Upgrades**

API, Web, Worker and Telegram Bot run `1.10.0`, pinned by digest, with no automatic updates. To upgrade, read the release notes, then set the new `{tag}@sha256:{digest}` as the image of all four services and deploy them together. API dumps the database and migrates before it serves. Worker and Telegram Bot wait until the migrations are applied.

**Telemetry**

Upstream's worker sends one anonymous usage snapshot a day; the list of fields is in TELEMETRY.md. Set `TELEMETRY_DISABLED=1` on Worker to turn it off.

**Cost**

Estimated at Railway usage prices: about $10 a month idle and $20 to $30 for a small team with a few agents. Hobby plan recommended. Model calls are billed by your AI provider, not by Railway.

### Why Deploy It's a Plan on Railway?

Railway runs the web app, the api, the worker, the bot, Postgres and the bucket as one project on a private network, with HTTPS, volumes and logs handled for you. You get a working tracker on a generated URL in minutes, without a domain, a server or a reverse proxy of your own, and add a custom domain whenever you like.

This is a community template. It is not affiliated with or endorsed by It's a Plan or its author. It's a Plan is AGPL-3.0 software; this template runs the unmodified Community images, built from the `v1.10.0` tag of github.com/croffasia/itsaplan.
