# Architecture — certificate-manager

> **Last updated:** 2026-07-31 (CERT-0001)
> **Category:** *reference* (information-oriented) — facts for lookup. The
> reasoning behind every decision below lives in
> [`../explanation/architecture-decisions.md`](../explanation/architecture-decisions.md).

## Overview

certificate-manager operates an internal X.509 certificate authority for
service-to-service identity. It is not publicly trusted and issues no
publicly-trusted certificates.

The system runs as cooperating Quarkus services, one per container, over a
shared Oracle database. Signing key material is reachable only through a
PKCS#11 interface held by a single component; no other component has a path to
it. Leaf certificates are short-lived and continuously renewed, and there is no
leaf revocation infrastructure — expiry is the revocation mechanism.

## Components

| Component | Path | Runtime | Purpose |
|-----------|------|---------|---------|
| `cert-api` | `cert-api/` | Java 21 / Quarkus | Public HTTP API — EST endpoints, enrolment and status. Holds no key material. |
| `cert-signer` | `cert-signer/` | Java 21 / Quarkus | CA signing worker. Sole holder of the PKCS#11 session. Issues certificates; never exposed outside the internal network. |
| `cert-renewer` | `cert-renewer/` | Java 21 / Quarkus | Expiry scanning **and** automatic rolling. Detects certificates past the renewal threshold, drives reissue, performs the overlap swap and rollback. |
| `cert-db` | `db/` | Oracle DB (container) | Persistence — certificates, enrolment requests, roll state, audit log. |
| `softhsm` | `ops/softhsm/` | SoftHSM sidecar | **Non-production only.** Provides the PKCS#11 token in dev and CI. Replaced by an HSM or cloud KMS in production. |

### Change from the initial component sketch

The original sketch listed five components including separate `cert-monitor`
(expiry scanning) and `cert-roller` (renewal). These are **merged into
`cert-renewer`**.

Scanning and rolling share the same privileges, the same database access, and
the same trust level, so a container boundary between them isolates nothing. The
roller must independently re-verify expiry before acting — its rolls are
idempotent (ADR-006) — which makes the monitor's output advisory and duplicates
the check on both sides of an event hop that can itself fail. The surveyed
systems that perform both functions (cert-manager, step-ca) implement them as a
single controller.

`softhsm` is new, and appears only in non-production topologies.

## Trust boundaries

| Boundary | Enforced by |
|---|---|
| `cert-api` → key material | **No path exists.** `cert-api` holds no PKCS#11 credential, no token PIN, and no filesystem mount reaching key storage. |
| `cert-api` → `cert-signer` | mTLS, internal network only. `cert-signer` publishes no externally-routable port. |
| `cert-renewer` → `cert-signer` | mTLS, same internal network. |
| Any component → root key | **No path exists in any environment.** The root key is offline; it signs intermediates during an offline ceremony only. |
| Enrolment request → signer | CSR subject, SAN, key usage and extended key usage are validated against the requested profile **before** the request reaches `cert-signer`. |
| Environment → environment | Each environment has its own issuing intermediate, scoped by X.509 name constraints and `pathLenConstraint: 0`. |

## Certificate authority hierarchy

```
Root CA  (offline, 10 years, >=2 copies in separate physical locations)
 |-- Issuing CA - dev       (online, 3 years, pathLenConstraint: 0, name-constrained)
 |-- Issuing CA - staging   (online, 3 years, pathLenConstraint: 0, name-constrained)
 |-- Issuing CA - prod      (online, 3 years, pathLenConstraint: 0, name-constrained)
      |-- leaf certificates (7 days)
```

## Certificate policy

| Parameter | Value |
|---|---|
| Leaf validity | 7 days |
| Renewal threshold | 1/3 of issued lifetime remaining (≈4.7 days of age), expressed as a percentage of actual issued duration |
| Retry budget before hard expiry | ≈56 hours |
| Private key rotation | New keypair on every renewal |
| Issuing CA validity | 3 years (NIST SP 800-57 max cryptoperiod for a private signature key) |
| Root CA validity | 10 years (certificate; the root private key signs only during offline ceremonies) |
| Leaf revocation | None — expiry is the mechanism |
| Intermediate revocation | CRL |
| Certificate Transparency | Not used |

## Enrolment

Protocol: **EST (RFC 7030)** over HTTPS.

| Endpoint | Purpose | Authentication |
|---|---|---|
| `/simpleenroll` | First issuance | Out-of-band bootstrap credential (contract not yet specified) |
| `/simplereenroll` | Renewal | The requester's current certificate, over mTLS |
| `/cacerts` | Chain distribution | None required |

Requests name an **issuance profile** rather than passing policy parameters.
A profile fixes validity, key algorithm and size, key usage, extended key usage,
and the permitted subject/SAN namespace.

## Data flows

### Flow 1 — First issuance

1. Consumer generates a keypair and a PKCS#10 CSR.
2. Consumer POSTs the CSR to `cert-api` `/simpleenroll` with its bootstrap credential.
3. `cert-api` authenticates the requester, resolves the named profile, and validates the CSR's subject, SAN, key usage and EKU against it. A CSR failing validation is rejected here and never reaches the signer.
4. `cert-api` calls `cert-signer` over mTLS.
5. `cert-signer` signs via PKCS#11 using the environment's issuing key.
6. Certificate and metadata are persisted to `cert-db`; an audit record is written with the correlation ID.
7. `cert-api` returns the certificate and chain.

### Flow 2 — Renewal (EST re-enrolment)

1. Consumer generates a **new** keypair and CSR.
2. Consumer POSTs to `/simplereenroll` authenticated by its **current** certificate over mTLS.
3. `cert-api` verifies the presented certificate is currently valid and issued by this CA, then validates the new CSR against the same profile.
4. Steps 4–7 of Flow 1 follow unchanged.

### Flow 3 — Expiration detection

1. `cert-renewer` scans `cert-db` on a schedule for certificates at or past the renewal threshold.
2. Each due certificate becomes a roll with a `renewal_request_id`.
3. The roll is persisted `pending` before any action is taken.

### Flow 4 — Automatic rolling

1. `cert-renewer` requests reissue via `cert-signer` (state → `installed` once new material is written).
2. The new certificate is delivered to the consumer **while the previous one remains valid** — a dual-certificate overlap window.
3. `cert-renewer` probes the consumer's health endpoint (state → `verified`).
4. On success, the roll commits (state → `committed`); the previous certificate and key are retained for one further renewal cycle, then destroyed.
5. On failure, `cert-renewer` re-points the consumer at the **retained previous material**. Rollback performs no signing and makes no call to `cert-signer`.
6. Every transition is persisted. A crashed-and-retried roll carrying the same `(consumer_id, renewal_request_id)` is a no-op when the target state is already reached.

## Storage schema (cert-db)

Sketch — column types and migrations are not yet specified.

| Table | Key columns | Notes |
|---|---|---|
| `ca_authority` | `id`, `tier` (root/issuing), `environment`, `subject_dn`, `not_before`, `not_after`, `pkcs11_key_label` | Holds a key **label**, never key material. |
| `issuance_profile` | `id`, `name`, `validity_seconds`, `key_algorithm`, `key_size`, `key_usage`, `eku`, `name_constraint` | Named policy referenced by enrolment requests. |
| `certificate` | `id`, `serial`, `issuer_ca_id`, `profile_id`, `consumer_id`, `subject_dn`, `san`, `not_before`, `not_after`, `status`, `predecessor_id` | `predecessor_id` links a renewal to the certificate it replaces. |
| `enrolment_request` | `id`, `consumer_id`, `profile_id`, `csr_fingerprint`, `outcome`, `rejection_reason`, `correlation_id` | Records rejected requests as well as accepted ones. |
| `roll` | `id`, `renewal_request_id`, `consumer_id`, `from_certificate_id`, `to_certificate_id`, `state`, `state_changed_at` | `state` ∈ pending / installed / verified / committed / rolled_back. `(consumer_id, renewal_request_id)` is the idempotency key. |
| `audit_log` | `id`, `event_type`, `actor`, `subject_certificate_id`, `correlation_id`, `occurred_at` | Append-only. Never contains key material. |

## Key custody

| Environment | PKCS#11 provider |
|---|---|
| Development / CI | SoftHSM sidecar container |
| Production | HSM or cloud KMS (product not yet selected) |
| Root key (all environments) | Offline; never present in a running container |

`cert-signer` accesses private keys **only** through PKCS#11. No code path reads
a key file directly, in any environment.

## Decisions

Rationale, options considered, and consequences for every decision above are
recorded in
[`../explanation/architecture-decisions.md`](../explanation/architecture-decisions.md)
(ADR-001 … ADR-006), each traced to findings in
[`../ephemeral/research/pki-market-research.md`](../ephemeral/research/pki-market-research.md).

## Not yet specified

- Bootstrap credential for first issuance.
- HSM / KMS product selection.
- Consumer health-probe contract required by Flow 4.
- Audit log retention policy and full schema.
- Oracle column types and migration scripts.
