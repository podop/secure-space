# Architecture Decisions — certificate-manager

> **Category:** *explanation* (understanding-oriented). This document holds the
> **why**. The resulting facts — component map, data flows, storage schema,
> policy values — live in [`../reference/architecture.md`](../reference/architecture.md).
>
> **Evidence base:** every decision cites `F-n` findings from
> `../ephemeral/research/pki-market-research.md` (CERT-0001 Phase 1). A decision
> with no cited finding is not a decision, it is a preference.

<!--
SUPERSESSION RULE (CERT-0001 design decision D-7).

These ADRs live in one file while none has been superseded. When an ADR is
superseded for the first time, split THAT ADR into its own file:

    explanation/adr-00N-<slug>.md   with frontmatter `supersedes: <prior>`

and leave a pointer row in the index table below. Do not edit a superseded
decision in place — supersede it. Rationale: ADR-calibre decisions are
immutable; the supersede path is a new file, not an edit.
-->

## Decision index

| ADR | Decision | Status | Research basis |
|-----|----------|--------|----------------|
| [ADR-001](#adr-001--ca-topology) | Two-tier CA: offline root (10 y) + per-environment issuing intermediates (3 y) | Accepted (validity corrected 2026-07-31) | F-1, F-11, F-11b, F-11c, F-11d, F-14, F-15, F-16, F-19, F-20 |
| [ADR-002](#adr-002--enrolment-protocol) | EST (RFC 7030) as primary enrolment; named issuance profiles | Accepted | F-2, F-3, F-3a, F-3b, F-4, F-5, F-14, F-21 |
| ~~[ADR-003](#adr-003--private-key-custody-and-the-cert-signer-trust-boundary)~~ | ~~PKCS#11 as the sole key interface; SoftHSM in dev, HSM/KMS in prod~~ | **Superseded 2026-07-31** → [`adr-003-key-custody-delegated-to-platform-services.md`](adr-003-key-custody-delegated-to-platform-services.md) | F-15, F-16, F-17, F-17a (F-18 retracted) |
| [ADR-003](adr-003-key-custody-delegated-to-platform-services.md) *(current)* | Key custody delegated to `key-management-service` + `hsm-management-service`; `cert-signer` becomes a client | Accepted | Same findings — superseded on platform scope, not on evidence |
| [ADR-004](#adr-004--adr-005--certificate-lifetime-and-revocation-one-decision) | 7-day leaf lifetime, renewal at 1/3 remaining, key rotated on every renewal | Accepted (joint with ADR-005) | F-7, F-10, F-12, F-13, F-14, F-15, F-21 |
| [ADR-005](#adr-004--adr-005--certificate-lifetime-and-revocation-one-decision) | No leaf revocation infrastructure; expiry is the mechanism; CRL for intermediates only | Accepted (joint with ADR-004) | F-6, F-7, F-9, F-14, F-15 |
| [ADR-006](#adr-006--rollout-and-rollback-mechanics) | Dual-certificate overlap window; rollback never calls the signer | Accepted | F-13, F-15, F-22, F-22a |

---

## ADR-001 — CA topology

### Context

The CA hierarchy determines the blast radius of a key compromise and the cost of
recovering from one. It is the hardest decision to reverse: changing the trust
anchor means redistributing it to every consumer.

### Options considered

1. Single self-signed root issuing leaves directly.
2. Offline root + one online issuing intermediate.
3. Offline root + per-environment issuing intermediates (dev / staging / prod).
4. External corporate root + locally-operated intermediate.

### Decision

**Option 3.** An offline root anchors the hierarchy; each environment gets its
own online issuing intermediate, constrained by X.509 **name constraints** and
**path length**.

- Root: **10-year certificate** validity (step-ca production guidance, F-11d),
  generated and stored offline, ≥2 copies in separate physical locations. This
  is consistent with NIST: a root certificate carries a *public
  signature-verification key*, whose cryptoperiod is "several years" (F-11b).
  The root's *private* key is used only during rare offline signing ceremonies.
- Issuing intermediates: **3-year certificate** validity, one per environment,
  online, holding the only keys used in day-to-day signing.

> **Why 3 and not 5.** An issuing CA key is, in NIST's taxonomy, a **private
> signature key** — and SP 800-57 Part 1 Rev. 5 § 5.3.6 recommends a maximum
> cryptoperiod of **one to three years** for one, requiring it to be destroyed
> at the end of that period (F-11). The distinction that matters is between
> *certificate validity* and *signing-key usage window*: NIST bounds the
> **key's** active signing period, not the certificate's shelf life (F-11b).
> A 5-year intermediate certificate implies a 5-year signing window unless the
> key is rotated mid-certificate (F-11c). Setting certificate validity equal to
> the NIST maximum makes the two windows coincide, so no mid-certificate re-key
> is needed and the constraint is satisfied by construction.
>
> This value was **corrected from ~5 years** after the primary source was read
> directly. The earlier figure was carried from a secondary gloss and recorded
> as a "project choice"; it conflicted with the standard this ADR cites.
>
> § 5.3.6 additionally notes that a CA-certified private signature key's
> cryptoperiod "ends when the `notAfter` date is reached on the last certificate
> issued for the public key". With 7-day leaves (ADR-004), the last leaf an
> intermediate issues expires at most 7 days after it stops signing, so that
> constraint is satisfied trivially.
- `pathLenConstraint: 0` on each intermediate — it may issue leaves and nothing
  else.
- Name constraints scope each intermediate to its environment's namespace.

### Consequences

- Compromise of an issuing intermediate is contained to one environment and is
  recovered by revoking that intermediate and re-issuing under a sibling — the
  root and every other environment stay valid.
- The root is used only to sign intermediates, so the offline ceremony is rare.
- Consumers must trust the root, not the intermediate, and must be served the
  full chain.
- Per-environment intermediates multiply the number of CAs to operate. This is
  the cost AWS Private CA's per-CA monthly billing makes prohibitive (F-20) —
  but that constraint is specific to the managed service and does not bind a
  self-hosted deployment, where an additional intermediate costs a keypair and a
  row in the database.
- `pathLenConstraint: 0` forecloses sub-intermediates. Reversing that later
  requires re-issuing the intermediate.

### Research basis

Option 1 was eliminated in Phase 1: **no surveyed system** supports a
single-tier root for a CA that must rotate. Vault documents an offline-root
tutorial as an endorsed strategy (F-14); step-ca states plainly that the root key
"is not needed for day-to-day CA operation and should be stored offline" and
keeps ≥2 copies in separate locations (F-15, F-16). Name constraints and path length are
enforceable per RFC 5280 (F-1) and AWS Private CA demonstrates them working as a
production topology control (F-19). Per-environment separation is affordable
here precisely because F-20's cost objection is AWS-specific.

**Validity figures.** The root's 10-year certificate follows step-ca's
production guidance (F-11d) and is consistent with NIST's "several years" for a
public signature-verification key (F-11b). The issuing intermediate's 3-year
validity is bounded by NIST SP 800-57 Part 1 Rev. 5 § 5.3.6 / Table 1 row 1,
which caps a private signature key's cryptoperiod at one to three years (F-11).
Because a CA issuing key *is* a private signature key (F-11c), the earlier
~5-year figure exceeded the recommendation and was corrected.

---

## ADR-002 — Enrolment protocol

### Context

Consumers need a protocol to request certificates and to renew them. The choice
determines what client software must exist, how a requester is authenticated,
and whether renewal can be fully automated — which in turn gates ADR-004.

### Options considered

1. ACME (RFC 8555).
2. EST (RFC 7030).
3. SCEP (RFC 8894).
4. Bespoke REST API over mTLS.

### Decision

**Option 2 — EST**, with **named issuance profiles** borrowed from ACME/Vault.

- `/simpleenroll` for first issuance, `/simplereenroll` for renewal, `/cacerts`
  for chain distribution.
- Renewal authenticated by the **current certificate** over mTLS.
- First issuance (bootstrap) authenticated out-of-band; the bootstrap credential
  is a separate concern, deliberately not resolved here.
- A request names a **profile** (e.g. `service-default`, `short-lived`) rather
  than passing lifetime and constraint parameters per request.

### Consequences

- Renewal needs no separate credential: the client proves identity with the
  certificate it is replacing, which is exactly the mTLS posture the system
  already assumes.
- EST clients are less common than ACME clients. Consumers without one need a
  small client shipped by this project.
- Profiles move issuance policy out of the request and into named, reviewable
  configuration — the requester chooses *which* policy, never *what* the policy
  says.
- ACME is **not** foreclosed. Adding an ACME front-end later for consumers that
  already speak it is additive, and the profile concept transfers directly.

### Research basis

SCEP was killed in Phase 1 on RFC 8894's own text: SHA-1 defaults, no proof of
possession on renewal, unauthenticated `GetCACaps` permitting downgrade, and no
issuance confirmation (F-5) — and it is Informational, not Standards Track.

ACME was rejected as *primary* on fit, not quality. Its two standard challenges
prove **domain control** (F-2), which is the wrong question internally: the
requester is already authenticated by mTLS, so re-proving name control adds a
dependency without adding assurance. Operationally both challenges are awkward
here — http-01 needs inbound port 80, dns-01 collides with split-horizon DNS
(F-3b). EST's `/simplereenroll` answers the actual question — *is this the same
party, holding the key we certified?* — with the credential already in hand
(F-4).

RFC 8555 § 8.3 / § 8.4 define exactly **two** challenge types, http-01 and
dns-01 (F-3). **TLS-ALPN-01 is not among them** — it is a separate standard,
RFC 8737 (F-3a). That matters for this decision: "just use ACME with
TLS-ALPN-01" is not a single-protocol adoption but two RFCs, and the operational
objections to http-01 and dns-01 in an internal network (F-3b) apply to the
RFC 8555 baseline as written.

The profile mechanism is taken from Let's Encrypt's ACME certificate profiles
(F-21) and Vault's roles (F-14), which are the same idea arrived at
independently. Both were surfaced as anti-anchoring additions in Phase 1.

---

## ADR-003 — Private-key custody and the `cert-signer` trust boundary

> **SUPERSEDED 2026-07-31** by
> [`adr-003-key-custody-delegated-to-platform-services.md`](adr-003-key-custody-delegated-to-platform-services.md).
>
> The text below is **left unedited on purpose** — it is the accepted record of
> what was decided on the evidence available at the time, and the supersession
> rule forbids editing a decision in place. It remains an accurate description
> of the **currently implemented** system: `cert-signer` still holds the PKCS#11
> session, because the platform services do not exist yet.
>
> The supersession was driven by **platform scope, not by an evidence
> correction** — its mechanism (PKCS#11-only, dev/prod parity by configuration,
> root key never online) is carried forward unchanged.

### Context

CA private keys are the system's crown jewels. The project mandate states that
cryptographic material is a first-class secret. The architecture already splits
`cert-signer` from the public API; this decision defines what that split
actually enforces.

### Options considered

1. Software keystore on the `cert-signer` container filesystem.
2. PKCS#11 against SoftHSM in a sidecar.
3. Cloud KMS (non-exportable keys, sign-as-a-service).
4. Network HSM.
5. Split: root in cold storage, issuing key in HSM/KMS.

### Decision

**Option 5, expressed through a PKCS#11 abstraction.**

- `cert-signer` reaches private keys **only** through PKCS#11. There is no code
  path that reads a key file directly.
- **Development and CI:** SoftHSM in a sidecar container, same PKCS#11
  interface.
- **Production:** a PKCS#11-capable HSM or cloud KMS. Selecting the specific
  product is out of scope; the interface is the decision.
- **Root key:** never online in any environment. Generated and used in an
  offline ceremony only.
- **Hard boundary:** `cert-api` has no credential, no network path, and no
  filesystem access that reaches key material. It calls `cert-signer` over mTLS
  and receives a signed certificate. This is non-negotiable and is the reason
  the two components are separate containers.

### Consequences

- Custody differs between dev and production by **configuration, not code**, so
  the signing path exercised in CI is the same path that runs in production.
- Signing throughput becomes bounded by the HSM/KMS, which feeds ADR-006's
  reissue-rate ceiling.
- A software keystore is never permitted, even in dev — SoftHSM costs little and
  keeps the interface honest.
- Adds a PKCS#11 dependency and an extra container in every environment.

### Research basis

EJBCA's Crypto Token is the pattern: key storage backed *either* by a soft
keystore *or* an HSM PKCS#11 slot behind one abstraction (F-17), which is what
makes dev/prod parity achievable. Two operational constraints come with it:
only **one crypto token per HSM slot** (F-17), and a PKCS#11 integration should
target **P11 NG** rather than the SunPKCS11 provider, which EJBCA deprecated in
9.4 (F-17a).

> **Correction.** An earlier draft of this ADR cited "EJBCA recommends dedicated
> HSMs for root **and** issuing keys" (F-18). That claim came from a vendor blog,
> and the primary EJBCA documentation does **not** designate which CA keys must
> be HSM-protected. F-18 is retracted; no per-key-type HSM mandate is attributed
> to EJBCA here. The decision is unaffected — it rests on the crypto-token
> abstraction (F-17, primary) and step-ca's split (F-15, F-16, primary).

step-ca supports PKCS#11 HSMs and cloud KMS
for the online intermediate while keeping the root in cold storage (F-15, F-16)
— exactly the split adopted here.

The `cert-api`-must-not-reach-keys boundary was fixed at design time as a kill
criterion, not derived from research: it follows from the existing component
split, and any option violating it was inadmissible.

---

## ADR-004 ⊗ ADR-005 — Certificate lifetime and revocation (one decision)

> **These are deliberately one deliberation.** Lifetime and revocation answer the
> same question — *how do we stop trusting a compromised key?* Deciding them
> separately is the canonical PKI failure: a lifetime policy that assumes
> revocation infrastructure nobody funded, or revocation infrastructure built for
> certificates that expire before it matters. They remain two numbered records
> for traceability only.

### Context

A compromised private key must stop being trusted. There are exactly two
mechanisms: wait for the certificate to expire, or actively revoke it and hope
relying parties check. The choice between them determines whether revocation
infrastructure is load-bearing or dead weight.

### Options considered

| Lifetime | Revocation implied |
|---|---|
| ~1 year | Mandatory: OCSP responder or CRL, actively checked |
| ~90 days | Strongly recommended |
| 1–7 days | Largely redundant — the expiry window is the exposure window |
| Hours | Redundant |

### Decision

**Short-lived leaves with no leaf revocation infrastructure.**

- **Leaf lifetime: 7 days.**
- **Renewal threshold: at 1/3 of lifetime remaining** (≈4.7 days of age),
  expressed as a **percentage of actual issued duration**, never as an absolute
  window. That leaves ≈56 hours of retry budget before hard expiry.
- **Private key rotated on every renewal** — a renewal produces a new keypair,
  not a re-signature of the old public key.
- **No OCSP responder. No CRL for leaves.** Expiry *is* the revocation
  mechanism.
- **CRL retained for intermediates only** — rare, high-impact, and small enough
  that a CRL is trivial to publish and check.
- **Compromise response for a leaf:** stop renewing it, and if the exposure
  window matters, roll the consumer immediately (ADR-006). Worst-case residual
  trust is 7 days; typical is under 5.

### Consequences

- The renewal path becomes the most-exercised code path in the system. That is
  the point: a rotation mechanism used every few days is one whose failures are
  found in daily operation, not during an incident.
- The system's availability now depends on renewal working. A signer outage
  longer than the retry budget causes expiry — this is the dominant operational
  risk and is why the threshold is a percentage with ≈56 hours of slack.
- Key rotation on renewal means a leaked key is invalidated by the next renewal
  regardless of whether the leak was noticed.
- No revocation infrastructure to build, operate, monitor, or keep available.
- 7 days is not the shortest defensible answer. It was chosen over 24 hours to
  give a comfortable retry budget before the reissue path is proven in
  production; it can be shortened later without re-architecting, and shortening
  is the expected direction.

### Research basis

The coupling is stated most directly by Vault: short TTLs mean "revocations are
less likely to be needed, keeping CRLs short" (F-14). step-ca goes further and
prefers short lifetimes **over** active revocation outright, describing CRL as
introducing operational dependencies (F-15).

The evidence against building OCSP is unusually strong. RFC 6960 concedes in its
own security considerations that responses are replayable, that the protocol is
DoS-susceptible, and that implementations may need CRL fallback (F-6). Let's
Encrypt then shut its responders down entirely on 6 August 2025, citing client
privacy exposure and ~12 billion daily requests for little benefit under
soft-fail behaviour (F-7). Building in 2026 what the largest CA in the world
just decommissioned would need a specific justification this project does not
have.

The 7-day figure sits inside the range the market has converged on: step-ca
defaults to 24-hour leaves and recommends ≤1 month for service certificates
(F-15); Let's Encrypt made 6-day certificates generally available in January 2026
(F-21). CA/B Forum SC-081v3 takes public TLS to 47 days by 2029 (F-10) — not
binding on an internal CA, but it is where tooling assumptions are heading.

The threshold mechanics are taken from cert-manager, which renews at 2/3 of
lifetime, prefers `renewBeforePercentage` over an absolute window because it
adapts when issued duration differs from requested, and warns that a threshold
too close to the duration causes a renewal loop (F-12). Key-rotation-on-renewal
is cert-manager's `rotationPolicy: Always` default, adopted for its stated
reason: it ensures the rotation path is exercised routinely rather than first
tested in an emergency (F-13).

Certificate Transparency is **not** adopted. Its threat model is misissuance by
a publicly-trusted CA detected by third parties auditing public logs (F-9) —
this CA has no public trust and no external relying parties. Note also that
RFC 6962 is obsoleted by RFC 9162 (F-8); any future CT work must cite 9162.

---

## ADR-006 — Rollout and rollback mechanics

### Context

The project mandate requires that automatic rolling be **idempotent** and that a
failed swap **roll back without downtime**. With 7-day certificates (ADR-004)
this path runs constantly, so its failure modes are operational, not theoretical.

### Options considered

1. Two-phase swap with health gate — write new material, probe consumer health,
   commit or restore.
2. Dual-certificate overlap — both old and new valid for a window; no atomic
   swap moment exists.
3. Blue/green trust bundle — rotate the bundle, not the leaf.
4. Consumer-pull — publish and let consumers fetch on their own schedule.

### Decision

**Option 2 — dual-certificate overlap**, with a health gate borrowed from
Option 1.

- The new certificate is installed **while the previous one remains valid**.
  There is no instant at which the consumer holds no valid certificate.
- The previous certificate **and its private key** are retained until the new
  one is confirmed healthy, then retained for one further renewal cycle before
  destruction.
- **Rollback = re-point the consumer at the retained previous material.** It
  performs no signing and makes no call to `cert-signer`.
- **Idempotency key:** `(consumer_id, renewal_request_id)`. A crashed-and-retried
  roll with the same key is a no-op if the target state is already reached; it
  never issues a second certificate.
- A roll is `pending` → `installed` → `verified` → `committed`, persisted at
  every transition, so a crash is recoverable by reading state rather than by
  inferring it.

### Consequences

- Two valid certificates exist per consumer during the overlap. Anything
  pinning a specific serial must tolerate this.
- Retaining previous private keys extends their lifetime — bounded to one
  renewal cycle, and they live under the same custody rules (ADR-003).
- Rollback works when `cert-signer` is down, which matters because signer
  unavailability is a plausible *cause* of a failed roll.
- Requires a per-consumer health probe. A consumer with no health signal cannot
  be safely auto-rolled, and must be explicitly marked as such.

### Research basis

The primary evidence establishes **retain-until-signed**: cert-manager "waits
until the Certificate object is correctly signed before overwriting the
`tls.key` file in the Secret", and constrains renewal windows so at least one
valid window exists before expiry (F-22). Existing material is never destroyed
before its replacement is confirmed.

> **Scope of the evidence.** cert-manager documents replace-on-success of a
> single Secret — **not** two simultaneously-valid certificates with the
> consumer choosing. The dual-valid framing rests on secondary sources only, and
> cert-manager's docs do not cover renewal-failure handling at all (F-22a).
>
> This decision therefore goes **deliberately further than the primary
> evidence**. Retain-until-signed removes the no-valid-material window; the
> overlap window additionally lets a consumer mid-rotation present either
> certificate, which is what makes the roll safe for consumers this system does
> not control the restart timing of. That extra property is a **project choice
> justified by the kill criteria** — no no-valid-certificate instant, and
> rollback that never calls the signer — not an industry pattern this research
> established.

Two framework kill criteria eliminated the alternatives directly. *Any option
requiring a re-sign to roll back is killed* — this removes naive two-phase swap
where rollback means re-issuing the old certificate, since it fails exactly when
the signer is unavailable. *Any option with a no-valid-certificate window is
killed* by the standing no-downtime mandate.

Retention of previous material is what makes rollback signer-independent, and
key rotation on every renewal (F-13) is what makes the retained key a bounded
rather than indefinite exposure.

---

## Open items

Deliberately unresolved here; each needs its own decision.

| Item | Why deferred |
|---|---|
| Bootstrap credential for first issuance | ADR-002 covers renewal; initial trust establishment is a separate problem with its own threat model. |
| HSM / KMS product selection | ADR-003 fixes the PKCS#11 interface; the product is a procurement decision. |
| Consumer health-probe contract | ADR-006 requires one; its shape depends on what consumers actually expose. |
| Audit log retention and schema | Referenced by the reference architecture; not yet specified. |
| ~~NIST SP 800-57 cryptoperiod confirmation~~ | **CLOSED 2026-07-31.** The primary PDF was read directly (§ 5.3.6 + Table 1). It did not confirm the prior figure — it **contradicted** it: a private signature key's maximum cryptoperiod is 1–3 years, so the issuing-intermediate validity was corrected from ~5 years to **3 years**. See F-11, F-11b, F-11c and ADR-001. |
