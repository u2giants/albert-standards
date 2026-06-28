# Designflow Cloud Run Environment

**Last updated:** 2026-06-22

This page records the non-code infrastructure, server, and operating-environment
standard for the Designflow PLM application. It is the exception to the default
Albert project pattern in the root prompts: Designflow does not deploy through
GitHub Actions -> Coolify on the Hetzner VPS.

## Runtime

Designflow runs in Google Cloud, not on the Hetzner/Coolify application host.

| Item | Current standard |
|---|---|
| GCP project | `lithe-breaker-323913` |
| Region | `us-central1` |
| Runtime | Google Cloud Run, managed/serverless/stateless |
| Build/deploy | Google Cloud Build triggers on git push |
| Image registry | Google Artifact Registry |
| Database | PostgreSQL reachable by Cloud Run through configured networking/VPC connector |
| Secrets | Google Secret Manager, injected with Cloud Run `--set-secrets` |
| Plain config | Cloud Run `--set-env-vars`; every value that must survive a deploy must be listed |
| File/media storage | DigitalOcean Spaces, with service-specific `DO_*` env/secrets |
| SSH/server editing | Not used for app deploys; deploy by image/revision only |

Cloud Build trigger substitutions live in GCP, not in the repositories. Treat the
trigger settings as part of production configuration. If a repo-level
`cloudbuild.yaml` adds or renames a substitution, update the relevant trigger and
the service docs together.

## Service Topology

The browser runs an Angular SPA served from a public Cloud Run service. The SPA
does not call private domain services directly in deployed environments; it calls
the BFF at `api.*.designflow.app`.

| Service | Repo | Deployed role | Public? | Local default port |
|---|---|---|---|---|
| Frontend SPA | `designflow-frontend` | Angular build served by nginx/brotli | yes | `4200` |
| BFF/API gateway | `designflow-bff` | Browser-facing proxy and Cloud Run OIDC caller | yes | `5004` |
| Core PLM API | `designflow-backend` | RFQ, products, auth, users, AI/tariff/file helpers | no | `5000` |
| Item Master API | `designflow-item-master` | Item Library, Item Detail, Art Piece workflows | no | `5003` |
| Tracking API | `designflow-tracking` | Production, licensing, lead-time, sample tracking | no | `5002` |
| Data Sync API | `designflow-data-syncing` | Coldlion -> shared Postgres sync | no, system-to-system | `5001` |

The BFF is the only public API entry point. Private services are deployed with
`--no-allow-unauthenticated` and accept requests from the BFF service account via
Google Cloud Run service-to-service OIDC.

## Auth And Routing Standard

Browser requests carry the Designflow user JWT to the BFF. The BFF sends:

- `Authorization: Bearer <Cloud Run OIDC token>` so Cloud Run allows the request
- `X-User-Authorization: Bearer <Designflow user JWT>` so the private service can
  restore user identity before route handlers run

Each private service has first middleware that restores `x-user-authorization`
to `authorization`. Do not rename this header without coordinating the BFF and
all private services.

The BFF `*_BACKEND_URL` env vars are both proxy targets and OIDC audiences. They
must be the backend service `.run.app` URLs, never custom `api.*.designflow.app`
domains. Custom domains route HTTP correctly but do not work as Cloud Run OIDC
audiences and have caused repeated Microsoft SSO/login outages.

## Environments

The current branch/environment pattern is:

| Environment | Branch | Public app/API domains | Purpose |
|---|---|---|---|
| Albert sandbox | `sandbox-albert` | `alsand.designflow.app`, `api.alsand.designflow.app` | Albert's personal test environment |
| Shared sandbox/develop | `develop` | `sandbox.designflow.app`, `api.sandbox.designflow.app` | Shared staging/integration |
| Production | `main`/production trigger | `designflow.app`, `www.designflow.app`, `app.designflow.app`, `api.designflow.app` | Live production |

Frontend configuration is compiled into the Angular bundle from
`src/environments/*`; the frontend container has no runtime `.env` injection.
Backend services load `.env.<NODE_ENV>` locally and receive Cloud Run env/secrets
in deployed environments.

## Deployment Rules

- CI/CD is Google Cloud Build only; there is no GitHub Actions deploy pipeline
  for Designflow.
- Cloud Build builds Docker images, pushes to Artifact Registry, then deploys
  Cloud Run revisions.
- Rollback is by Cloud Run revision traffic shift or redeploying a previous
  Artifact Registry image tag.
- Do not hot-edit running containers or server files. Change the repo or Cloud
  Run/Cloud Build configuration, then redeploy.
- Tests are part of service startup/build in several repos. Do not bypass test
  gates to push a deploy.

## Secrets And Config Rules

- Secrets belong in Secret Manager and must be wired with `--set-secrets`.
- Never put credentials in `--set-env-vars`.
- `--set-env-vars` is a full replacement on deploy. Any plain env var not listed
  can be deleted by the next deploy.
- For DigitalOcean Spaces, keys go through Secret Manager; endpoint/bucket/public
  URL may be plain env vars when non-secret.
- Azure AD client and tenant IDs are not secrets, but redirect URIs must be kept
  in sync with deployed hostnames.

## Frontend Serving Rules

The frontend is an nginx-served SPA on Cloud Run.

- `index.html` must not be long-cached. It should revalidate so new deployments
  are picked up immediately.
- Content-hashed JS/CSS/assets may be long-cached.
- MSAL popup SSO requires `/assets/auth-redirect.html` and
  `/assets/msal-redirect-bridge.min.js` in every deployed frontend image.

## Shared Database And Encoding Rules

The Node backend services use Sequelize models over a shared PostgreSQL schema.
Several repos physically duplicate model definitions, so schema changes often
require coordinated updates across services.

- Additive schema changes use idempotent `ALTER TABLE ... ADD COLUMN IF NOT EXISTS`
  blocks in service startup migrations.
- Quote mixed-case identifiers in raw Postgres DDL.
- Keep `client_encoding` UTF-8; never load production or sandbox data through a
  Windows CP437 console. Use UTF-8-safe `pg_dump`/`pg_restore` or `psql` paths.

## When To Update This Page

Update this page whenever a Designflow infrastructure decision changes:

- Cloud Run service topology, public/private ingress, domains, or BFF routing
- Cloud Build trigger behavior, image path/tagging, deploy or rollback policy
- Runtime resource settings, VPC connector, service accounts, or project/region
- Secret/config handling, new required env vars, or storage providers
- Auth/OIDC/SSO flow, redirect URI requirements, or service-to-service headers
- Database hosting, schema ownership, migration style, or data-loading rules

Then update the relevant Designflow repo docs (`AGENTS.md`, `docs/deployment.md`,
`docs/configuration.md`, or `docs/system-architecture.md`) so future AI sessions
see both the local implementation detail and this cross-repo standard.
