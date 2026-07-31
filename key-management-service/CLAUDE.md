# key-management-service — CLAUDE.md

> **Umbrella entry-point:** [`../CLAUDE.md`](../CLAUDE.md) — read the Supreme Directive there first. Everything below scopes to this project only.

## Project Overview

**key-management-service** owns the **lifecycle of cryptographic keys** across the
secure-space platform: generation, storage references, rotation, retirement, and
the policy that governs each. It is the authority on *what keys exist, under what
policy, and in what state* — not on the hardware that holds them (that is
[`../hsm-management-service`](../hsm-management-service/CLAUDE.md)).

**Responsibilities:**

- **Key lifecycle** — generate, activate, rotate, retire, destroy; enforce cryptoperiods.
- **Key inventory** — the authoritative record of every managed key, its purpose, its state, and its custody location.
- **Key policy** — algorithm, size, cryptoperiod, and permitted usage per key class.
- **Delegated signing/wrapping** — expose operations to consumers without ever exposing key material.

**Components:** *[TODO: fill at `/dr-prd` — one component per container, per umbrella convention]*

**Terminology aliases:** *[TODO: fill in as decisions land]*

## Boundary with certificate-manager (supersession pending)

**Operator decision, 2026-07-31:** this service and `hsm-management-service`
**supersede `cert-signer`'s key-custody role**.

- `key-management-service` owns key lifecycle.
- [`../hsm-management-service`](../hsm-management-service/CLAUDE.md) owns the HSM / PKCS#11 provider.
- `cert-signer` becomes a **client** of both, rather than the sole holder of the PKCS#11 session.

**Supersession recorded 2026-07-31.** The prior ADR-003 made `cert-signer` the
*sole* holder of the PKCS#11 session. It is superseded by
[`../certificate-manager/documentation/explanation/adr-003-key-custody-delegated-to-platform-services.md`](../certificate-manager/documentation/explanation/adr-003-key-custody-delegated-to-platform-services.md),
written as a new file with `supersedes:` frontmatter — the prior decision was not
edited in place, per the supersession rule in that project.

Two things that record makes explicit, and this service inherits:

- The change is driven by **platform scope, not by an evidence correction**. The
  prior ADR's mechanism — PKCS#11-only access, dev/prod parity by configuration,
  root key never online — carries forward **unchanged**.
- The prior ADR still describes the **currently implemented** system.
  `cert-signer` holds the session today; this delegation is the target state.

> **[TODO — resolve before implementation]** Two questions this boundary raises:
> 1. Does `cert-signer` call this service for signing, or does it hold a
>    short-lived delegated credential? ADR-003's trust boundary ("`cert-api` has
>    no path to key material") must survive whichever answer wins.
> 2. Does the CA root key move here, or stay offline under certificate-manager's
>    ceremony? ADR-001 keeps it offline; that constraint should outlive this change.

## Tech Stack

- **Language / runtime:** Java 21 (LTS)
- **Framework:** Quarkus (latest stable)
- **Build:** Maven (with the wrapper `mvnw`)
- **Test:** JUnit 5 + Quarkus test extensions (`@QuarkusTest`)
- **Persistence:** *[TODO: confirm at `/dr-prd` — Oracle for umbrella consistency, or a dedicated store]*
- **Containerisation:** Docker + Docker Compose (one container per component)
- **Crypto:** JDK 21 built-in (`java.security`), PKCS#11 via the provider owned by `hsm-management-service`
- **Observability:** Quarkus SmallRye Health / Metrics (Micrometer + Prometheus format)

Stack chosen to match `certificate-manager` (operator decision, 2026-07-31) so the
three services share crypto tooling, build commands, and patterns.

## Build Commands

```bash
# Placeholders until the Maven skeleton lands via /dr-plan → /dr-do.
./mvnw verify                                  # compile + unit tests
./mvnw quarkus:dev                             # dev mode
docker compose up -d                           # bring up the stack
```

## Conventions

- **One component = one container = one Maven module.**
- **No `latest` Docker tags** — pin every image.
- **Secrets never in git.** Key material, DB passwords, and HSM PINs live only in mounted volumes / secret stores.
- **Key material never leaves its custody boundary.** This service manages *references and policy*; the bytes stay behind PKCS#11.
- **Migration-first schema changes** — Flyway or Liquibase; no manual DDL.
- **Structured logging** — JSON via Quarkus logging; correlation ID on every request.

## Gotchas

> Hard-won lessons. Each one line, imperative, specific.

1. *[none yet — add as they surface]*

## Datarim Workflow

Workflow state is **shared across the umbrella** — this project has no local `datarim/`.

- **Task prefix:** *[TODO: register a prefix — e.g. `KMS-` — in the umbrella `CLAUDE.md` § Task Prefix Registry. Without a registry entry, archives resolve to `general/`.]*
- **Start a task:** `/dr-init <description>`

## Documentation Map

Following [Diátaxis](https://diataxis.fr) — four orthogonal categories, one per directory.

| Document | Purpose |
|----------|---------|
| `documentation/tutorials/` | Learning-oriented — end-to-end walkthroughs |
| `documentation/how-to/` | Task recipes |
| `documentation/reference/` | Lookup — key classes, API, storage schema |
| `documentation/explanation/` | Design rationale — ADRs live here, not in reference |
| `documentation/ephemeral/` | Transient plans / research / reviews |

## Key Files

*(seeded once scaffolding advances beyond docs)*

- `pom.xml` — parent POM
- `docker-compose.yml` — local topology
- `db/migration/` — migrations

## Additional Rules

- **Key material is a first-class secret.** Never log a key, never write one to a log-shipping surface, never commit one outside `**/test-fixtures/**`.
- **Cryptoperiods are enforced, not advisory.** NIST SP 800-57 bounds a private signature key at 1–3 years (see certificate-manager's research finding F-11); this service is where that gets enforced.
- **Rotation must be idempotent and reversible** — the same constraint certificate-manager's ADR-006 places on certificate rolling.
