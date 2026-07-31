# certificate-manager — CLAUDE.md

> **Umbrella entry-point:** [`../CLAUDE.md`](../CLAUDE.md) — read the Supreme Directive there first. Everything below scopes to this project only.

## Project Overview

**certificate-manager** implements the full X.509 certificate lifecycle for internal services:

- **Issuance** — create private keys and self-issue leaf certificates from an internal CA
- **CSR (Certificate Signing Request) handling** — accept, validate, and sign external CSRs
- **Expiration monitoring** — continuously watch every managed certificate's `notAfter`
- **Automatic rolling / renewal** — reissue certificates before expiry and swap them into consumers

Each of these responsibilities is intended to live in its **own container** — the deployment topology is a set of Quarkus microservices around a shared Oracle DB, wired by Docker Compose.

**Components (settled at CERT-0001 — see [`documentation/reference/architecture.md`](documentation/reference/architecture.md)):**

| Component | Container role | Notes |
|-----------|---------------|-------|
| `cert-api` | Public HTTP API — EST enrolment / status | Quarkus (RESTEasy Reactive). Holds no key material. |
| `cert-signer` | CA / signing worker | Sole holder of the PKCS#11 session; never externally routable |
| `cert-renewer` | Expiry scanning **and** auto-renewal | Merged `cert-monitor` + `cert-roller` — same privileges and data access, so a container boundary between them isolated nothing |
| `cert-db` | Oracle DB | Persistence for certificates, enrolment requests, roll state, audit log |
| `softhsm` | PKCS#11 token (**non-production only**) | Dev/CI substitute for the production HSM/KMS |

**Terminology aliases:** *[fill in as decisions land]*

## Tech Stack

- **Language / runtime:** Java 21 (LTS)
- **Framework:** Quarkus (latest stable)
- **Build:** Maven (with the wrapper `mvnw`)
- **Test:** JUnit 5 + Quarkus test extensions (`@QuarkusTest`)
- **Persistence:** Oracle DB (containerised — `container-registry.oracle.com/database/free` or equivalent)
- **Containerisation:** Docker + Docker Compose (one container per component)
- **Crypto:** JDK 21 built-in (`java.security`), Bouncy Castle where CSR/PKCS handling requires it
- **Observability:** Quarkus SmallRye Health / Metrics (Micrometer + Prometheus format)

Native-image builds (GraalVM) are viable given Quarkus — a decision to be made at `/dr-prd` per component.

## Build Commands

```bash
# From the certificate-manager/ directory — commands become available once modules are scaffolded.
./mvnw verify                                  # compile + unit tests
./mvnw quarkus:dev -pl cert-api                # dev mode for one module
./mvnw -pl cert-api -am package                # package one module + its deps
docker compose up -d                           # bring up all containers
docker compose logs -f cert-renewer            # tail one component
```

*(These are placeholders until the Maven multi-module skeleton lands via `/dr-plan` → `/dr-do`.)*

## Conventions

- **One component = one container = one Maven module.** No shared bootstrap module.
- **No `latest` Docker tags** — pin every image (JDK, Quarkus base, Oracle DB) to a specific version.
- **Secrets never in git.** CA private keys, DB passwords, TLS material live only in mounted volumes / secret stores.
- **Test fixture certificates are allowed under `**/test-fixtures/**`** (the umbrella `.gitignore` whitelists them) — production material never.
- **Migration-first schema changes** — use Flyway or Liquibase; Oracle DDL changes go through migrations, not manual scripts.
- **Structured logging** — JSON via Quarkus logging; correlation ID on every request.

## Gotchas

> Hard-won lessons. Each one line, imperative, specific.

1. *[none yet — add as they surface]*

## Datarim Workflow

Workflow state is **shared across the umbrella** — this project has no local `datarim/`. All tasks for certificate-manager use the `CERT-*` prefix (e.g. `CERT-0001`) and land in the umbrella's `datarim/` and `documentation/archive/`.

- **Task prefix:** `CERT-` for all certificate-manager work
- **Start a task:** run `/dr-init <description>` from either this directory or the umbrella root — the framework resolves the git-toplevel and writes into the umbrella's `datarim/`

## Documentation Map

Following [Diátaxis](https://diataxis.fr). Cross-project material stays at the umbrella; project-specific material lives here.

| Document | Purpose |
|----------|---------|
| `documentation/tutorials/` | End-to-end walkthroughs (first cert issuance, first CSR flow) |
| `documentation/how-to/testing.md` | How to run unit / integration / container tests |
| `documentation/how-to/deployment.md` | How to bring up the stack, roll a release, roll back |
| `documentation/how-to/gotchas.md` | Discovered pitfalls |
| `documentation/reference/architecture.md` | Component map, data flow, storage schema |
| `documentation/explanation/` | Design rationale — CA topology, why per-component containers, native vs JVM |
| `documentation/ephemeral/` | Transient plans / research / QA reports |

## Key Files

*(seeded once scaffolding advances beyond docs)*

- `pom.xml` — parent POM (multi-module)
- `cert-api/`, `cert-signer/`, `cert-renewer/` — Maven modules (one per container)
- `docker-compose.yml` — local topology
- `db/migration/` — Flyway/Liquibase migrations

## Additional Rules

- **Cryptographic material is a first-class secret.** Never log a private key, never write one into a log-shipping surface, never commit one outside `**/test-fixtures/**`.
- **CSR validation is a security boundary.** Reject malformed / unexpected subjects before ever letting them near the signer.
- **Auto-rolling must be idempotent and reversible.** A failed swap must roll back to the previous cert without downtime.
