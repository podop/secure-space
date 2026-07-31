# Deployment — certificate-manager

> Last updated: 2026-07-31

## Environments

| Environment | URL | Purpose |
|-------------|-----|---------|
| Local | `http://localhost:8080` (cert-api) | Development, via `docker compose up` |
| Test | *[TBD]* | Integration verification before prod |
| Production | *[TBD]* | Live |

## Deploy Steps

*[Fill in once the pipeline is designed at `/dr-plan`.]*

1. Build all modules: `./mvnw -DskipTests package`
2. Build container images: `docker compose build`
3. Push images to the registry (tag with git SHA — never `latest`)
4. Roll out via the environment's deploy mechanism (compose / k8s / …)
5. Health-check every container (`/q/health`) before declaring success

## Rollback

- Redeploy the previous image tag (git SHA of the last known-good release).
- Certificates issued during the bad window remain valid until they expire.
  `cert-renewer` will not touch them until they cross the renewal threshold
  (1/3 of lifetime remaining).
- **If a bad issuance policy leaked, do not look for a revocation switch — there
  is none.** The architecture ships no leaf revocation mechanism: expiry is the
  mechanism, and a leaf lives at most 7 days. To shorten the exposure window,
  force an immediate roll of the affected consumers so `cert-renewer` reissues
  them under the corrected policy; the previous certificate stays valid through
  the overlap window, so the roll is non-disruptive and reversible.
  See [`../explanation/architecture-decisions.md`](../explanation/architecture-decisions.md)
  ADR-004 ⊗ ADR-005 for why, and ADR-006 for the roll mechanics.

## Monitoring

- `/q/health/live` — liveness
- `/q/health/ready` — readiness (includes DB reachability)
- `/q/metrics` — Prometheus scrape endpoint
- Alerts: expiring-cert budget, signing-error rate, DB connection saturation
