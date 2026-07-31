---
adr: ADR-003
title: Private-key custody delegated to platform services
status: accepted
supersedes: architecture-decisions.md#adr-003
superseded_decision: "ADR-003 — Private-key custody and the `cert-signer` trust boundary"
date: 2026-07-31
driver: operator decision (platform scope), not new research evidence
related:
  - ../../../key-management-service/CLAUDE.md
  - ../../../hsm-management-service/CLAUDE.md
  - ../../../CLAUDE.md
---

# ADR-003 (superseding) — Private-key custody delegated to platform services

> **Supersedes** the ADR-003 recorded in
> [`architecture-decisions.md`](architecture-decisions.md#adr-003--private-key-custody-and-the-cert-signer-trust-boundary).
> That decision is **not edited** — it stands as the accepted record of what was
> decided on the evidence available at the time. This file replaces it going
> forward.

## Context

The original ADR-003 was made when `certificate-manager` was the only service in
the workspace. It made `cert-signer` the **sole** holder of the PKCS#11 session,
because there was nothing else that could hold one.

Two sibling services now exist:

- [`key-management-service`](../../../key-management-service/CLAUDE.md) — key lifecycle, policy, inventory.
- [`hsm-management-service`](../../../hsm-management-service/CLAUDE.md) — the HSM and the PKCS#11 provider.

The operator decided (2026-07-31) that these two **own key custody for the
platform**, and that `cert-signer` becomes a client of them rather than the
custodian.

### What drives this supersession — stated plainly

**This is an architectural scope decision, not an evidence correction.** No
research finding contradicted the original ADR-003. Its mechanism —
PKCS#11-only access, dev/prod parity by configuration, root key never online —
remains supported by the same primary sources (F-15, F-16, F-17, F-17a) and is
**carried forward unchanged** below.

This matters for how the two records should be read. Earlier in CERT-0001, the
issuing-CA validity was corrected from ~5 years to 3 because a primary source
*contradicted* the shipped value. That was a defect. This is not: the prior
ADR-003 was correct for a single-service workspace, and it is being replaced
because the workspace stopped being single-service.

## Options considered

1. **Keep custody in `cert-signer`; the new services serve other consumers.**
   The two services would each need their own PKCS#11 provider, giving the
   platform three independent custody implementations and three places to get
   HSM configuration wrong.
2. **Move custody wholesale to the platform services; `cert-signer` becomes a
   client.** One provider, one place HSM configuration lives, one audit surface.
3. **Split by key class** — CA keys stay in `cert-signer`, everything else moves.
   Preserves the CA's independence but keeps two provider implementations and
   makes "who owns this key?" a per-key question rather than a structural one.

## Decision

**Option 2.** Key custody moves to the platform services.

- **`hsm-management-service` holds the PKCS#11 session.** It is now the only
  component in the platform that talks to an HSM or loads a PKCS#11 provider.
- **`key-management-service` owns key lifecycle** — generation, rotation,
  retirement, cryptoperiod enforcement, and the inventory of what exists.
- **`cert-signer` becomes a client.** It requests signing operations over mTLS;
  it no longer holds a session, a PIN, or a slot credential.
- **`cert-signer` remains a distinct component.** It still owns CA semantics —
  profile enforcement, certificate assembly, serial allocation, chain
  construction. Delegating *custody* is not the same as deleting the signer.

### Carried forward from the superseded ADR-003 — unchanged and still binding

These were correct then and are correct now. Re-opening any of them needs a
reason, not an oversight:

1. **PKCS#11 is the sole key interface.** No code path reads a key file
   directly, in any environment. The interface moves; it is not relaxed.
2. **Dev/prod parity by configuration, not code** — SoftHSM in dev and CI, HSM
   or cloud KMS in production, same interface. CI must exercise the production
   signing path.
3. **The root key is never online** in any environment.
4. **A software keystore is never permitted**, even in development.
5. **One crypto token per HSM slot** (F-17); target **P11 NG**, not the
   SunPKCS11 provider EJBCA deprecated at 9.4 (F-17a).

### The trust boundary, restated for three components

The superseded ADR's boundary was simple because there were two components:
*`cert-api` has no path to key material.* With custody delegated, "no path" has
to be re-stated, because there are now more components that could constitute a
path.

| Boundary | Enforced by |
|---|---|
| `cert-api` → key material | **No path.** Unchanged and non-negotiable. |
| `cert-api` → `hsm-management-service` | **No path.** `cert-api` must not be able to reach the custody service directly — otherwise delegating custody would have *created* the bypass the original boundary existed to prevent. |
| `cert-signer` → `hsm-management-service` | mTLS, authenticated as a named client, authorised per key. This is the only route from certificate-manager to a signing operation. |
| `hsm-management-service` → key material | PKCS#11 only. It performs operations; it never returns key bytes. |
| Any component → root key | **No path in any environment.** Offline ceremony only. |

**The new risk this decision creates:** custody is now reachable over the
network, where before it was in-process behind a container boundary. The
original boundary was enforced by "there is no such path"; the new one is
enforced by authentication and authorisation, which is a weaker guarantee. That
is the price of the decision, and it should be named rather than glossed.

## Consequences

- **One custody implementation instead of three.** HSM configuration, PIN
  handling, and slot management live in one place with one audit surface.
- **Signing crosses a network hop.** The superseded ADR noted that signing
  throughput is bounded by the HSM and feeds ADR-006's reissue-rate ceiling.
  That ceiling now includes network latency and the availability of a second
  service. With 7-day leaves (ADR-004) and continuous rolling, `certificate-manager`
  now has a **runtime dependency on `hsm-management-service` for every issuance
  and every renewal**.
- **A new failure mode:** `hsm-management-service` being down stops all issuance
  *and* all renewal. ADR-006's rollback deliberately does not call the signer,
  so rollback survives this — but forward progress does not. The renewal retry
  budget (≈56 hours, ADR-004) is now also the outage budget for this dependency.
- **Cross-project versioning appears.** The signing interface becomes a contract
  between two independently deployable services and needs to be versioned as one.
- **CA independence is reduced.** `certificate-manager` can no longer issue
  without a sibling service. If CA availability must exceed platform
  availability, that is an argument for Option 3, and this decision should be
  revisited.

## Research basis

**No new findings.** The mechanism carried forward rests on the same primary
sources as the superseded decision:

- **F-17** (primary) — EJBCA's Crypto Token abstracts key storage over either a
  soft keystore or an HSM PKCS#11 slot, which is what makes dev/prod parity
  achievable; one crypto token per HSM slot.
- **F-17a** (primary) — P11 NG is the primary PKCS#11 implementation;
  SunPKCS11 deprecated at EJBCA 9.4.
- **F-15, F-16** (primary) — step-ca keeps the root in cold storage and supports
  PKCS#11 HSMs and cloud KMS for the online intermediate.
- **F-18** remains **retracted** — the primary EJBCA documentation does not
  designate which CA keys must be HSM-protected. It is not resurrected here.

Findings live in
[`../ephemeral/research/pki-market-research.md`](../ephemeral/research/pki-market-research.md).

**What has no research basis:** the delegation itself. No surveyed system was
examined for how it splits custody across services, because that question did
not exist when the survey was scoped. The decision rests on the operator's
platform architecture, and this ADR does not claim otherwise.

## Open items

Each needs its own decision; none is resolved here.

| Item | Why it is open |
|---|---|
| Does the CA root key move to `hsm-management-service`, or stay under certificate-manager's offline ceremony? | ADR-001 keeps it offline and that constraint outlives this change — but *whose* ceremony it is has not been decided. |
| How does `cert-signer` authenticate to `hsm-management-service`? | It needs a credential it did not previously require. A bootstrap problem: the certificate service may need a certificate to obtain certificates. |
| What prevents `cert-api` reaching `hsm-management-service` directly? | Network policy, service-mesh authorisation, or provider-side client allowlist. The boundary table above states the requirement; the mechanism is unspecified. |
| Signing interface contract and its versioning | Now a cross-service API, not an in-process call. |
| Does the availability coupling hold? | If CA availability must exceed platform availability, Option 3 (split by key class) is the better answer and this ADR should be revisited. |

## Implementation impact on existing records

The following are now **stale** with respect to this decision and need updating
when it is implemented. They are listed, not silently edited:

- [`../reference/architecture.md`](../reference/architecture.md) — § Components
  describes `cert-signer` as "sole holder of the PKCS#11 session"; § Trust
  boundaries and § Key custody both encode the two-component model; the
  `softhsm` row is scoped to this project.
- [`architecture-decisions.md`](architecture-decisions.md) — the index row for
  ADR-003 should point here (see the pointer row added at supersession time).
- [`../how-to/deployment.md`](../how-to/deployment.md) — rollback guidance
  assumes the signer is local.

These stay accurate for the *currently implemented* system, which still has
`cert-signer` holding the session. Updating them before the platform services
exist would make the documentation describe something that is not running.
