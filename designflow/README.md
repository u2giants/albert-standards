# DesignFlow PLM — cross-repo system docs (canonical)

Canonical home for DesignFlow documents that describe the **whole** system,
not one service.

| File | Purpose |
|---|---|
| [system-architecture.md](system-architecture.md) | Whole-system map (services, request path, runtime) |
| [system-interconnectivity.md](system-interconnectivity.md) | Service wiring and integration edges |
| [feature-dependency-matrix.md](feature-dependency-matrix.md) | Feature → service dependency map |

## Edit rule

- These three files are the source of truth. Update them here in the same
  change-set that changes the system map (second PR if the change started in a
  service repo).
- Each `designflow-*` service repo keeps a **stub** at `docs/<same-name>.md`
  with this absolute URL and “do not edit here.”
- Do **not** use relative `../designflow-...` links; they break in single-repo
  clones. Use this GitHub URL:
  `https://github.com/u2giants/albert-standards/blob/main/designflow/`

## Boundary

| Content | Home |
|---|---|
| Multi-app / data contracts (schema, RLS, API contracts) | `popcre/shared-db` (or `u2giants/shared-db` as used in docs) |
| PLM service topology (this folder) | `u2giants/albert-standards` `designflow/` |
| Single-service detail | that service repo’s `docs/` |

## Provenance

Seeded 2026-10-13 from `designflow-frontend/docs/` copies (PR popcre/designflow-frontend#309 series). Other service forks still hold drifted copies until they are converted to stubs — keep history in those repos; do not delete a fork body until the stub lands.
