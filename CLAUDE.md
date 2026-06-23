# ISAAC AI-Ready Record — Portal (developer notes)

This repo holds two things: the **data standard** (`schema/`, `examples/`, `wiki/` —
see `README.md`, the normative v1.0 spec) and the **portal application** under
`portal/` that serves and edits records. This file documents the portal app; the
README is the frozen public standard.

## Portal layout
- `portal/app.py` — Streamlit UI (records browser, form, Admin Review, API Keys, nano ISAAC).
- `portal/api.py` — Flask API (gunicorn sidecar) for programmatic access via Bearer tokens.
- `portal/database.py` — Postgres access (shared `isaac-psql` cluster).
- `portal/ontology.py` — vocabulary + wiki sync/push, plus the edge-identity gate.
- `portal/agent.py` — nano ISAAC (an LLM that emits read-only SQL over the records).

Deployed via Flux GitOps. Image tags are semver (never `:latest`); pushing to `main`
triggers a GitHub Actions build and image automation bumps the deployment.

## Security model
Defense in depth — each layer is independent.

### Identity is trusted only from the edge
The pod must not trust `X-authentik-username` on its own; anything reaching the pod
directly (in-cluster peer, port-forward, SSRF) could set it. `ontology.trusted_identity()`
trusts the Authentik headers only when the request also carries the `X-Isaac-Edge`
shared secret (`hmac.compare_digest` vs the `EDGE_AUTH_SECRET` env) injected and
overwritten by the ingress. Admin status is derived from the username against
`ISAAC_ADMINS` (the groups header is not consumed).
- FAIL-OPEN FALLBACK (local dev only): if `EDGE_AUTH_SECRET` is unset the gate fails
  open and logs a WARNING. In prod the env is required, so a missing value fails the
  pod rather than silently degrading.

### Free-form / AI SQL runs least-privilege (nano ISAAC, /records/query)
`database.execute_readonly_query()`: single statement only; SELECT/WITH only; mutation
keywords and system/file functions (`pg_*`, `lo_*`, `dblink`, `information_schema`, …)
rejected; wrapped in a READ ONLY transaction with a 5s timeout. It connects via
`get_readonly_db_connection()` as the `PGUSER_RO` role (non-superuser, SELECT only on
`records`/`templates`/`vocabulary_cache`). nano ISAAC additionally runs with
`agent_mode=True`, restricting reads to the `records` table by name. nano ISAAC is open
to all users; these layers are what make that safe.
- FALLBACK (loud, local dev only): if `PGUSER_RO` is unset, `get_readonly_db_connection()`
  logs a WARNING and falls back to the main connection. In prod the env is required.

### Other controls
- API keys expire (90 days) and are minted only for the edge-trusted caller (`app.py`).
- Privileged Admin Review actions re-check admin server-side at click time
  (`_require_admin_action()`), not just at page load.
- `GITHUB_TOKEN` is never inlined into the wiki remote URL; git auth goes via
  `http.extraHeader` (GIT_CONFIG_* env), and error strings are scrubbed (`_scrub_secrets`).
