# Notice

## Upstream software

This repository holds Railway configuration and one Caddyfile. It's a Plan itself is not stored or modified here: the template runs the Community edition images that the project publishes on each release. This is a community template, not affiliated with or endorsed by It's a Plan or its author.

| Software | License | Used as |
|---|---|---|
| It's a Plan 1.10.0, Community edition (api, web, worker, bot) | GNU Affero General Public License v3.0 only (AGPL-3.0-only), Copyright © 2026 Andrii Poluosmak. `packages/runner` is Apache-2.0. The author also offers a commercial licence. | Unmodified images, pinned by digest: `ghcr.io/croffasia/itsaplan-api:1.10.0@sha256:9f53d060db6594b2e51bc224183e8e8c60e544a1fd65dcd451c053568c642753`, `ghcr.io/croffasia/itsaplan-web:1.10.0@sha256:be213d6861f66ac68320db2a1481c050bc811a740d558f5ebde576b34f78b289`, `ghcr.io/croffasia/itsaplan-worker:1.10.0@sha256:840650f1dc8cec4d82cf8cdc6fa5cd43aa82df1fc9a9f9213358e3c33688aed0`, `ghcr.io/croffasia/itsaplan-bot:1.10.0@sha256:d0ab66c7454d003064ea455299db740746f6c6a53edcb375429881c88aad5780`. No licence key is used; the Pro images (`itsaplan-pro-*`) are not used. |
| Third-party components inside those images (Bun, Node.js, Next.js, Elysia, better-auth, Mastra, Drizzle, AWS SDK, Alpine Linux, PostgreSQL 18 client, ...) | Their own licenses | As shipped in the images; see `bun.lock` and the Dockerfiles at the source below. |
| Caddy 2.11.9 | Apache-2.0 | Official image `caddy:2.11.9-alpine@sha256:0bbb055bda6793c999bde78b1b5ce3e89f829fb031af2cf50056ea1246559fee` (Docker Official Image, source https://github.com/caddyserver/caddy-docker), used unmodified as the base of the Gateway service. |
| PostgreSQL 18 | PostgreSQL License | `ghcr.io/railwayapp-templates/postgres-ssl:18` (Railway's MIT-licensed image repository) |
| Lucide icons (`square-kanban`, `route`, `server`, `workflow`, `send`) | ISC | Referenced by URL from `cdn.jsdelivr.net/npm/lucide-static@1.59.0`; not copied |

## Corresponding source (AGPL-3.0)

The It's a Plan images used here are built by the project's own release workflow (`.github/workflows/publish-images.yml`: `actions/checkout` of the release tag, then `docker/build-push-action` per app and platform) from:

- Repository: https://github.com/croffasia/itsaplan
- Tag: `v1.10.0`, commit `6c7c2fe2cb9d810d29c68e054de25876b47434f8`: https://github.com/croffasia/itsaplan/tree/v1.10.0
- Dockerfiles: `apps/api/Dockerfile`, `apps/web/Dockerfile`, `apps/worker/Dockerfile`, `apps/bot/Dockerfile` at that tag
- License text: https://github.com/croffasia/itsaplan/blob/v1.10.0/LICENSE

Checked 2026-11-07: the source layer (`COPY . .`) of the api and worker images (3,383 files each) is byte-identical to the tag's tree, and the four images were created on 2026-11-05 between 00:51 and 00:53 UTC, right after the v1.10.0 release was published (00:50:33 UTC).

The template does not modify It's a Plan. If you modify it and let users interact with your modified version over a network, AGPL-3.0 section 13 requires you to offer those users the corresponding source of your version.

## Verified against

| What | Upstream file | Ref | Checked |
|---|---|---|---|
| Release, version, image tags and build | `CHANGELOG.md`, `package.json`, `.release-please-manifest.json`, `.github/workflows/publish-images.yml`; GitHub release `v1.10.0`; GHCR manifests and configs of the four images | `v1.10.0` (`6c7c2fe2`) | 2026-11-07 |
| Services, ports, commands, environment | `docker-compose.yml`, `docker-compose.coolify-images.yml`, `.env.example`, `apps/*/Dockerfile`, `charts/itsaplan/templates/*` | `v1.10.0` | 2026-11-07 |
| One origin with the api under a path; cookie domain and `__Host-` prefix | `docs/self-hosting.md` ("Where the api lives"), `packages/auth/src/index.ts`, `apps/web/src/lib/auth-client.ts`, `apps/web/src/proxy.ts`, `apps/api/src/app.ts` (OAuth discovery routes, CORS, CSRF) | `v1.10.0` | 2026-11-07 |
| Why the official Railway setup needs a custom domain | `docs/railway.md` | `v1.10.0` | 2026-11-07 |
| Migrations, pre-migration dump, waiting for migrations | `packages/db/src/migrate.ts`, `backup.ts`, `wait-for-migrations.ts`, `docs/dev/deploy-stack.md`, `charts/itsaplan/templates/_helpers.tpl`, `deployment-api.yaml` | `v1.10.0` | 2026-11-07 |
| S3 client and path style | `packages/storage/src/index.ts`, `.env.example` | `v1.10.0` | 2026-11-07 |
| Mail providers (SMTP, Resend) and From address | `packages/mailer/src/index.ts`, `docs/self-hosting.md` ("Mail") | `v1.10.0` | 2026-11-07 |
| First account becomes the instance owner; default registration mode | `packages/auth/src/index.ts`, `packages/auth/src/instance.ts` | `v1.10.0` | 2026-11-07 |
| Client IP for the rate limit | `docs/self-hosting.md` ("Client IP"), `docs/railway.md` | `v1.10.0` | 2026-11-07 |
| Telemetry | `TELEMETRY.md`, `apps/worker/src/telemetry.ts` | `v1.10.0` | 2026-11-07 |
| Next.js server behaviour behind the gateway | `apps/web/server.js` and Next.js 16.4.2 `dist/server` in the web image | image 1.10.0 | 2026-11-07 |
| Caddy directives and forwarded headers | caddyserver/caddy `modules/caddyhttp/reverseproxy/{reverseproxy,caddyfile,streaming}.go` | `v2.11.9` | 2026-11-07 |
| Railway behaviour | docs.railway.com: pre-deploy command (volumes not mounted), Storage Buckets (virtual-hosted URLs, variables, pricing), plans and prices | — | 2026-11-07 |

## Our files

| File | Origin |
|---|---|
| `services/gateway/Caddyfile`, `services/gateway/Dockerfile`, `template.json`, `TEMPLATE.md`, `README.md`, `NOTICE.md` | Written for this template. The routing follows the layout described in upstream's `docs/self-hosting.md`; no upstream file or text is copied. |

## Reference only (not copied)

- The author's own Railway template "It's a Plan" (Pro images, `latest` tag) was read for its wiring (bucket variables, `CLIENT_IP_HEADER`). None of its text is used here.
- Upstream README and docs were read for behaviour; no prose was copied.

## Trademarks

"It's a Plan" is the name of the upstream project and belongs to its author. "Railway", "Caddy", "PostgreSQL", "Linear", "Plane", "Trello", "Telegram", "Resend", "Claude" and "Cursor" belong to their respective owners. The template uses a neutral Lucide icon, not the project's logo. This is a community template and is not affiliated with or endorsed by It's a Plan, its author or Railway.
