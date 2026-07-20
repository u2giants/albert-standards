# Designflow Cloud Run Environment

**Last updated:** 2026-07-20

This page records the non-code infrastructure, server, and operating-environment
standard for the Designflow PLM application. It is the exception to the default
Albert project pattern in the root prompts: Designflow does not deploy through
GitHub Actions -> Coolify on the Hetzner VPS.

## Runtime

Designflow runs in Google Cloud, not on the Hetzner/Coolify application host.

| Item | Current standard |
|---|---|
| GCP project | `lithe-breaker-323913` |
| Region | `us-east4` for current Albert sandbox Cloud Run services; older docs/legacy services may still mention `us-central1` |
| Runtime | Google Cloud Run, managed/serverless/stateless |
| Build/deploy | Google Cloud Build triggers on git push |
| Image registry | Google Artifact Registry |
| Database | Managed non-production: shared Supabase pooler; production: private-VPC Cloud SQL |
| Secrets | Google Secret Manager, injected with Cloud Run `--set-secrets` |
| Plain config | Cloud Run `--set-env-vars`; every value that must survive a deploy must be listed |
| File/media storage | DigitalOcean Spaces, with service-specific `DO_*` env/secrets |
| SSH/server editing | Not used for app deploys; deploy by image/revision only |

Cloud Build trigger substitutions are managed as Terraform in
`popcre/infrastructure`; GCP is the applied runtime copy. Every database deploy
must carry provider, expected port, network path, SSL mode, all five secret IDs,
and all five secret versions. If a repo-level `cloudbuild.yaml` adds or renames a
substitution, change the infrastructure contract and service docs together.

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

Database targets are intentionally different by environment:

| Environment | Provider / port | Secret set | Network / SSL |
|---|---|---|---|
| `develop` | Supabase pooler / `6543` | `DB_*_DEV` | public pooler / SSL on |
| `staging` | Supabase pooler / `6543` | `DB_*_STAGING` | public pooler / SSL on |
| `sandbox-albert`, `albert-2sandbox` | Supabase pooler / `6543` | `DB_*_SANDBOX` | public pooler / SSL on |
| `production` | Cloud SQL / `5432` | unsuffixed `DB_*` | private VPC / SSL off |

The host, port, user, password, and database name are one indivisible tuple.
Never copy or update only one member across environments.

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
- Tests must run in Cloud Build, not inside Cloud Run startup. Cloud Run
  containers should start the actual Node process directly (`node index.js` or
  `node src/server.js`) rather than running `npm test`, `yarn start:$NODE_ENV`,
  or `nodemon` in deployed revisions.
- The active Albert sandbox services used for interactive testing should keep
  one warm instance where first-load reliability matters. As of 2026-07-09:
  `popcre-albert-bff-sandbox`, `popcre-albert-core-sandbox`, and
  `popcre-albert-item-sandbox` use `autoscaling.knative.dev/minScale=1`.
- Shared Supabase pooler session limits are part of non-production app runtime
  design. Services using Sequelize/`pg` against the pooler should keep conservative pool defaults
  (`DB_POOL_MAX=5`, `pool.min=0`, short idle/evict, TCP keep-alive) unless the
  shared connection budget is deliberately recalculated.

## Secrets And Config Rules

- Secrets belong in Secret Manager and must be wired with `--set-secrets`.
- Secret values and recovery notes belong in the 1Password `vibe_coding` vault;
  secret values never belong in repositories, plans, logs, or command history.
- Never put credentials in `--set-env-vars`.
- Non-production may follow `latest` within its matching suffixed secret set.
  Production database secret references must use numeric versions; `latest` is
  forbidden so a new secret version cannot silently retarget a deployment.
- Production automatic build triggers stay disabled until the four guarded
  backend changes have passed sandbox review and Uma approves promotion.
- Any production DB secret mutation is a break-glass operation requiring explicit
  owner approval. The project alert `CRITICAL: production DB secret version changed`
  reports additions, enable/disable actions, and destruction of unsuffixed DB secrets.
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
Several repos physically duplicate model definitions, so app model/query changes
often require coordinated updates across services.

- `u2giants/shared-db` is the only schema authority. Every table, column, view,
  RPC, trigger, RLS, seed, backfill, or cross-app data-contract change starts as
  a new timestamped migration there, preview first. Never add app startup DDL,
  app-local migrations, dashboard edits, or direct production SQL.
- Runtime pool/concurrency changes are not schema changes, but they still affect
  the shared database. Document confirmed pooler behavior in `u2giants/shared-db`
  and keep the `popcre/infrastructure` connection contract and app deploy guards
  in sync.
- Each private Node service validates the environment/provider/port/host/user/SSL
  relationship before its first connection. Its Cloud Build preflight validates
  the secret IDs and versions before image build. Invalid combinations fail closed.
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
