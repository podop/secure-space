# hsm-management-service — CLAUDE.md

> **Umbrella entry-point:** [`../CLAUDE.md`](../CLAUDE.md) — read the Supreme Directive there first. Everything below scopes to this project only.

## Project Overview

**hsm-management-service** owns the **hardware security module and the PKCS#11
provider** for the secure-space platform. It is the only component that talks to
an HSM, and it presents a uniform interface so that consumers never depend on a
specific vendor SDK or on whether the backing token is real hardware or a
software substitute.

**Responsibilities:**

- **PKCS#11 provider ownership** — hold the session, manage slots and tokens.
- **HSM lifecycle** — initialisation, health, failover, firmware/driver compatibility.
- **Custody boundary** — perform sign/wrap/unwrap on behalf of callers; key material never crosses the boundary.
- **Dev/prod parity** — the same interface backed by SoftHSM in development and CI, by an HSM or cloud KMS in production.

**Components:** *[TODO: fill at `/dr-prd` — one component per container, per umbrella convention]*

**Terminology aliases:** *[TODO: fill in as decisions land]*

## Boundary with certificate-manager (supersession pending)

**Operator decision, 2026-07-31:** this service and `key-management-service`
**supersede `cert-signer`'s key-custody role**.

- [`../key-management-service`](../key-management-service/CLAUDE.md) owns key lifecycle.
- `hsm-management-service` owns the HSM / PKCS#11 provider.
- `cert-signer` becomes a **client** of both, rather than the sole holder of the PKCS#11 session.

**This contradicts a shipped, archived decision.**
[`../certificate-manager/documentation/explanation/architecture-decisions.md`](../certificate-manager/documentation/explanation/architecture-decisions.md)
**ADR-003** currently states that `cert-signer` accesses private keys *only*
through PKCS#11 and is the sole holder of that session. That ADR is **pending
supersession**, not silently overridden — until a superseding record lands,
ADR-003 remains the decision backed by primary research findings (F-15, F-16,
F-17, F-17a).

> **[TODO — carry forward, do not re-derive]** ADR-003 already settled three
> things this service inherits. Re-opening them needs a reason, not an oversight:
> 1. **PKCS#11 is the sole key interface** — no code path reads a key file directly, in any environment.
> 2. **SoftHSM in dev/CI, HSM/KMS in production** — custody differs by *configuration, not code*, so CI exercises the production signing path.
> 3. **The root key is never online** in any environment.
>
> ADR-003 also carries two primary-sourced implementation constraints worth
> honouring here: **one crypto token per HSM slot**, and **target P11 NG rather
> than SunPKCS11**, which EJBCA deprecated at 9.4 (finding F-17a).

## Tech Stack

- **Language / runtime:** Java 21 (LTS)
- **Framework:** Quarkus (latest stable)
- **Build:** Maven (with the wrapper `mvnw`)
- **Test:** JUnit 5 + Quarkus test extensions (`@QuarkusTest`); SoftHSM as the CI token
- **Containerisation:** Docker + Docker Compose (one container per component)
- **Crypto:** PKCS#11 via JDK 21; vendor provider selected at deploy time
- **Observability:** Quarkus SmallRye Health / Metrics (Micrometer + Prometheus format)

Stack chosen to match `certificate-manager` (operator decision, 2026-07-31).

> **[TODO — verify before committing to the stack]** Some HSM vendors ship only
> C / native PKCS#11 bindings, and a few provide no usable JVM path at all. Confirm
> the intended vendor exposes a PKCS#11 library the JDK can load before the Maven
> skeleton is built around that assumption. This was flagged at scaffold time as
> the one place the Java/Quarkus choice could fail.

## Build Commands

```bash
# Placeholders until the Maven skeleton lands via /dr-plan → /dr-do.
./mvnw verify                                  # compile + unit tests
./mvnw quarkus:dev                             # dev mode
docker compose up -d                           # bring up the stack incl. SoftHSM sidecar
```

## Conventions

- **One component = one container = one Maven module.**
- **No `latest` Docker tags** — pin every image, including the SoftHSM sidecar.
- **Secrets never in git.** HSM PINs, slot credentials, and token material live only in mounted volumes / secret stores.
- **No key material crosses the service boundary.** Callers send data to be signed or wrapped; they never receive a key.
- **Dev and production differ by configuration, not by code path.**
- **Structured logging** — JSON via Quarkus logging; correlation ID on every request. Never log a PIN, slot secret, or key handle that could be replayed.

## Gotchas

> Hard-won lessons. Each one line, imperative, specific.

1. *[none yet — add as they surface]*

## Datarim Workflow

Workflow state is **shared across the umbrella** — this project has no local `datarim/`.

- **Task prefix:** *[TODO: register a prefix — e.g. `HSM-` — in the umbrella `CLAUDE.md` § Task Prefix Registry. Without a registry entry, archives resolve to `general/`.]*
- **Start a task:** `/dr-init <description>`

## Documentation Map

Following [Diátaxis](https://diataxis.fr) — four orthogonal categories, one per directory.

| Document | Purpose |
|----------|---------|
| `documentation/tutorials/` | Learning-oriented — end-to-end walkthroughs |
| `documentation/how-to/` | Task recipes (HSM init, slot setup, failover drill) |
| `documentation/reference/` | Lookup — provider config, slot map, API |
| `documentation/explanation/` | Design rationale — ADRs live here, not in reference |
| `documentation/ephemeral/` | Transient plans / research / reviews |

## Key Files

*(seeded once scaffolding advances beyond docs)*

- `pom.xml` — parent POM
- `docker-compose.yml` — local topology incl. SoftHSM sidecar
- `ops/softhsm/` — non-production token configuration

## Additional Rules

- **The HSM is the trust anchor of the whole platform.** A compromise here is not recoverable by rotation alone.
- **Every option must be exercisable locally.** An HSM configuration with no dev/CI substitute means the signing path is first exercised in production — ADR-003 kills such options explicitly.
- **Failover is a tested path, not a documented one.** A failover drill that has never run is an assumption.
