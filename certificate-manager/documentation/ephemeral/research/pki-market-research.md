# PKI Market Research — certificate-manager

> **Task:** CERT-0001 · Phase 1 · captured 2026-07-31
> **Purpose:** evidence base for the Project Architecture Document. Every claim
> below carries a source. Decisions are **not** made here — they are made in
> `../../explanation/architecture-decisions.md` (Phase 2), citing the `F-n`
> findings in Part D.

## Scope and method

**Surveyed:** the standards corpus that governs X.509 certificate lifecycle
(RFC 5280, 8555, 7030, 8894, 6960, 6962/9162), NIST key-management guidance, the
CA/Browser Forum validity schedule, and six production implementations
(cert-manager, HashiCorp Vault PKI, step-ca, EJBCA, AWS Private CA,
Let's Encrypt/Boulder).

**Per-implementation capture is deliberately narrow** — lifecycle, rotation, and
CA hierarchy only, per the strategist's timebox. Full feature matrices were not
built.

**Excluded by operator decision (2026-07-31):** Java/Quarkus ecosystem specifics
— Bouncy Castle APIs, reactive patterns, Oracle schema tuning. These land in a
follow-up task.

**Source discipline.** Primary sources (IETF Datatracker, NIST, CA/B Forum,
vendor official documentation) are used wherever the claim is load-bearing.
Where only a secondary source was available, the finding is tagged
**`[secondary]`** and should be re-verified against primary documentation before
a decision rests on it.

**Criteria applied** come from
`datarim/creative/creative-CERT-0001-architecture-decision-framework.md`
(binding — Consilium condition 1).

---

## Part A — Standards corpus

### RFC 5280 — X.509 / PKIX certificate and CRL profile

Title: *Internet X.509 Public Key Infrastructure Certificate and Certificate
Revocation List (CRL) Profile*. May 2008. **Proposed Standard.** Obsoletes
RFC 3280, 4325, 4630. **Updated by** RFC 6818, 8398, 8399, 9549, 9598, 9608,
9618, 9925, 10007 — but **not obsoleted**: it remains the normative base for
certificate and CRL structure.

Specifies certificate/CRL structure, mandatory vs optional extensions (key
usage, basic constraints), and the path-validation algorithm. Includes name
constraints and path-length constraints — both directly relevant to bounding
what an intermediate CA is permitted to issue.

Source: <https://datatracker.ietf.org/doc/rfc5280/>

### RFC 8555 — ACME

Title: *Automatic Certificate Management Environment*. March 2019. **Proposed
Standard.** Not obsoleted, not updated.

Defines two challenge types — **http-01** (token served at a well-known URL) and
**dns-01** (token in a TXT record) — and a four-phase issuance flow: account
creation with an asymmetric key pair → order submission → challenge validation →
CSR finalization and issuance.

Source: <https://datatracker.ietf.org/doc/rfc8555/>

### ACME applicability to an internal CA

The standard challenge types assume public reachability. http-01 requires
inbound HTTP on port 80. dns-01 requires DNS records the validating server can
resolve, which collides with split-horizon DNS — a client may only see the
internal view while the ACME server validates against the public one.
**TLS-ALPN-01** is described as better suited to private environments (proof via
a temporary certificate in the TLS handshake, no DNS or HTTP routing needed) but
conflicts with load balancers or proxies that terminate TLS.

Note that for an internal CA, "prove you control this name" is a materially
weaker requirement than for public trust — the requester is typically already
authenticated by other means. **`[secondary]`**

Sources: <https://www.bastionxp.com/blog/acme-certificate-automation-internal-pki-complete-guide/>,
<https://cert-manager.io/docs/troubleshooting/acme/>

### RFC 7030 — EST (Enrollment over Secure Transport)

Title: *Enrollment over Secure Transport*. October 2013. **Proposed Standard.**
Updated by RFC 8951, 8996, 9908.

Profiles CMC messages over HTTPS (TLS 1.1+). Basic enrolment uses a PKCS#10 CSR
as a "Simple PKI Request" against distinct URIs — `/simpleenroll`,
`/simplereenroll`, `/cacerts`. Four client-authentication modes: existing TLS
client certificate; certificate-less TLS with a shared secret; HTTP
Basic/Digest over TLS; bootstrap via manual fingerprint verification.

`/simplereenroll` authenticated by the current certificate is a natural fit for
renewal in an mTLS environment — the client proves identity with the credential
it is replacing.

Source: <https://datatracker.ietf.org/doc/rfc7030/>

### RFC 8894 — SCEP

Title: *Simple Certificate Enrolment Protocol*. September 2020.
**Informational — not Standards Track.**

The specification documents its own limitations: most deployed devices default
to **SHA-1**; renewal carries **no proof of possession** of the new private key,
permitting certificate-substitution attacks; the `GetCACaps` response is
**unauthenticated**, enabling algorithm downgrade; and there is **no
confirmation** that an issued certificate was received.

Source: <https://datatracker.ietf.org/doc/rfc8894/>

### RFC 6960 — OCSP

Title: *X.509 Internet PKI Online Certificate Status Protocol*. June 2013.
**Standards Track.** Obsoletes RFC 2560, 6277. Updated by RFC 8954, 9654.

Real-time revocation status without distributing full CRLs. The specification
itself acknowledges: requests lack responder identification and are **replayable
with precomputed responses**; the protocol is **DoS-susceptible** through
signature-computation cost; and it suggests implementations "could implement CRL
processing logic as a fall-back position" when OCSP is unreachable.

Source: <https://datatracker.ietf.org/doc/rfc6960/>

### OCSP in practice — industry retirement

Let's Encrypt retired OCSP in staged fashion during 2025: 30 Jan 2025 —
Must-Staple issuance began failing; 7 May 2025 — CRL URLs added and OCSP URLs
removed from certificates; **6 Aug 2025 — OCSP responders shut down entirely**.
New Let's Encrypt certificates carry only a CRL Distribution Point.

Stated reasons: privacy (OCSP queries reveal which sites a visitor is
contacting, plus visitor IP, to the CA) and operational cost — roughly twelve
billion requests per day for little security benefit given soft-fail client
behaviour.

Sources: <https://letsencrypt.org/2024/12/05/ending-ocsp> (primary),
<https://www.feistyduck.com/newsletter/issue_121_the_slow_death_of_ocsp>

### RFC 6962 / RFC 9162 — Certificate Transparency

RFC 6962 (*Certificate Transparency*, June 2013, Experimental) is
**obsoleted by RFC 9162** (*Certificate Transparency Version 2.0*, December
2021, also Experimental). CT 2.0 moved algorithms to IANA registries, redesigned
precertificates as CMS objects, identifies logs by OID, introduces the
`TransItem` structure and a unified `submit-entry` endpoint, and replaces the
`signed_certificate_timestamp` TLS extension with `transparency_info`.

CT's threat model is **misissuance by a publicly-trusted CA**, detected after
the fact by third parties auditing public logs. It does not prevent misissuance;
it bounds the window before it becomes auditable (the Maximum Merge Delay).

Sources: <https://datatracker.ietf.org/doc/rfc6962/>,
<https://datatracker.ietf.org/doc/rfc9162/>

### NIST SP 800-57 Part 1 Rev. 5 — key management

*Recommendation for Key Management, Part 1: General*, May 2020. Table 1 gives
suggested cryptoperiods by key type; Table 5 gives protection requirements for
cryptographic keys. Rev. 5 adds emphasis on protecting key **metadata**, access
control, identity authentication, and key/certificate inventory management.

Search summaries report CA signing-key cryptoperiods of roughly 20 years (root)
and 5 years (intermediate); **this specific pairing was not confirmed against
the PDF and must be verified before any decision rests on it.** **`[secondary]`**

Source: <https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-57pt1r5.pdf>

### CA/Browser Forum — validity reduction schedule (SC-081v3)

Ballot **SC-081v3**, adopted April 2025 (25 in favour, 0 against, 5 abstentions;
Apple, Google, Mozilla and Microsoft in support), schedules maximum public TLS
certificate validity downward: **398 → 200 days (15 March 2026) → 100 days
(2027) → 47 days (2029)**. Domain-control-validation evidence reuse drops to 10
days by March 2029.

This binds **publicly-trusted** certificates. An internal CA is not bound by the
Baseline Requirements — but the schedule is the clearest available signal of
where the industry's operational assumptions are heading, and tooling will
increasingly assume short lifetimes.

Source: <https://cabforum.org/2025/04/11/ballot-sc081v3-introduce-schedule-of-reducing-validity-and-data-reuse-periods/>

---

## Part B — Implementation survey

Captured per the framework's criteria: lifecycle, rotation, CA hierarchy — plus
a borrow / do-not-borrow verdict.

### cert-manager (Kubernetes)

**Lifecycle.** Default certificate `duration` is **90 days**; minimum 1 hour.
Renewal fires by default at **2/3 of lifetime**. `renewBefore` sets an absolute
pre-expiry window (minimum 5 minutes); `renewBeforePercentage` expresses it as a
fraction and is **preferred**, because it adapts when the issued duration
differs from the requested one. The docs warn explicitly that setting
`renewBefore` close to `duration` risks a **renewal loop**. Renewal windows
(cron expressions) can further constrain when renewal is permitted.

**Rotation.** From v1.18.0 the default is `rotationPolicy: Always` — the private
key is regenerated on every reissuance. The documentation frames simultaneous
key+certificate rotation as the recommended practice: it reduces exposure and
ensures the rotation path is exercised routinely rather than first tested during
an emergency.

**Borrow:** the percentage-based renewal threshold, and key-rotation-on-renewal
as the default. **Do not borrow:** the CRD/controller model — it is Kubernetes-
shaped and this project is Docker Compose.

Source: <https://cert-manager.io/docs/usage/certificate/>

### HashiCorp Vault PKI

**Hierarchy.** Root and intermediate CA setup are separate documented paths, and
an explicit tutorial covers building a CA "with an offline Root" — offline root
is an endorsed operational strategy.

**Lifecycle.** Role-based issuance with TTL / max_ttl per role. The
documentation ties lifetime directly to revocation load: *"By keeping TTLs
relatively short, revocations are less likely to be needed, keeping CRLs short
and helping the secrets engine scale to large workloads."* It further promotes
**ephemeral certificates** fetched into memory at application startup and
discarded at shutdown, never written to disk.

**Borrow:** the role abstraction (issuance policy as named, reusable
configuration) and the short-TTL-shrinks-revocation argument — this is the
clearest statement of the ADR-004 ⊗ ADR-005 coupling found in the survey.
**Do not borrow:** Vault's own storage/seal model.

Source: <https://developer.hashicorp.com/vault/docs/secrets/pki>

### step-ca (Smallstep)

**Hierarchy.** Two-tier by design: *"The intermediate key is used by the CA to
sign certificates. The root key is not needed for day-to-day CA operation and
should be stored offline."* Root key in cold storage — HSM or air-gapped device
— with at least two copies in separate physical locations for durability.

**Lifecycle.** Default leaf lifetime **24 hours**; intermediate aligned to root
expiry; root 10 years. Stated position: *"User certificates should have the
lifespan of a mayfly: about a day or less"*; host/service-account certificates
*"one month or less"*. Short-lived certificates are explicitly preferred **over
active revocation**; CRL exists for cases needing immediate revocation or longer
lifetimes, but is described as introducing operational dependencies.

**Custody.** PKCS#11 HSMs and cloud KMS supported for the online intermediate
key.

**Borrow:** the two-tier topology with cold-storage root; the explicit framing
of short lifetimes as a *substitute* for revocation; durability-by-replication
of the offline root. **Do not borrow:** 24-hour leaves as a starting default
without first proving the reissue path is reliable at that cadence.

Source: <https://smallstep.com/docs/step-ca/certificate-authority-server-production/>

### EJBCA (Keyfactor)

**Custody abstraction.** A **Crypto Token** is where keys live, backed *either*
by a soft keystore (a file in the database) *or* an HSM PKCS#11 slot. The same
abstraction covers both, so dev and production custody differ by configuration
rather than by code path. Operational constraint: one crypto token per HSM
slot; a token may be shared across multiple CAs.

**Hierarchy.** Guidance is an offline (air-gapped) root as ultimate trust
anchor, with dedicated HSMs recommended not only for the root but also for
**issuing CA keys and OCSP signing keys**.

**Profiles.** Certificate profiles act as templates constraining certificate
shape — purpose, key algorithms and sizes, key usage — and can be restricted to
a subset of CAs. The page consulted did not cover validity configuration or end
entity profiles.

**Borrow:** the crypto-token abstraction — it is the cleanest answer found to
DevOps' concern that an HSM decision must remain locally testable. **Do not
borrow:** EJBCA's full profile system; it is enterprise-scaled well beyond this
project. **`[secondary]`** for hierarchy and HSM guidance.

Sources: <https://docs.keyfactor.com/ejbca/latest/certificate-profiles-overview> (primary, profiles only),
<https://docs.keyfactor.com/ejbca/latest/hardware-security-modules-hsm>,
<https://www.ejbca.org/resources/navigating-hsm-options-for-ejbca-pki-a-guide-for-product-development-engineers-and-product-owners/>

### AWS Private CA

**Hierarchy.** Root and subordinate CAs as a managed service.

**Constraint enforcement — the notable part.** AWS Private CA *enforces*:
`Not After` never later than the issuing CA's `Not After`; basic constraints
(must be present, marked critical, `CA=true`) and path length; and **name
constraints**. It explicitly does *not* enforce certificate policies, policy
constraints/mappings, inhibit anyPolicy, issuer alternative name, subject
directory attributes, or SKI/AKI. Critically, **SubjectPublicKeyInfo and Subject
Alternative Name are copied from the CSR without validation**.

**Cost.** Monthly charge per CA, plus a charge per certificate issued. Cost
therefore scales with the *number of CAs*, which penalises a per-environment
intermediate design.

**Borrow:** name constraints and path-length enforcement as a topology-level
control — a mechanism for bounding what an intermediate may issue that does not
depend on application logic. **Borrow as a warning:** SAN-copied-from-CSR-
without-validation is exactly the gap the project's own mandate ("CSR validation
is a security boundary") exists to close — validation must happen before the
signer, because the signing layer may not do it.

Source: <https://docs.aws.amazon.com/privateca/latest/userguide/PcaWelcome.html>

### Let's Encrypt / Boulder

**Short-lived certificates.** Six-day certificates (valid ~160 hours) went
generally available in **January 2026**, selected via **ACME certificate
profiles** — the client requests the `shortlived` profile. IP-address
certificates are also GA and, as a matter of policy, **must** be short-lived.

**Revocation.** As above: OCSP responders shut down 6 Aug 2025; CRL-only.

**Borrow:** the profile mechanism — a named issuance profile selected at request
time is a cleaner contract than per-request lifetime parameters, and it maps
well onto Vault's role abstraction. **Do not borrow:** public-trust operational
scale; the rate-limit and CT-submission machinery is not applicable to an
internal CA.

Sources: <https://letsencrypt.org/2026/01/15/6day-and-ip-general-availability>,
<https://letsencrypt.org/2025/01/16/6-day-and-ip-certs>

### Zero-downtime rotation patterns

The convergent pattern across sources is a **dual-certificate overlap window**
rather than an atomic swap: both old and new certificates are valid
simultaneously so in-flight sessions and clients transition gracefully. The
replacement is installed *while the existing certificate remains in place*. This
removes the instant at which no valid certificate is installed — the failure
mode that makes a swap non-reversible. **`[secondary]`**

Sources: <https://expiring.at/blog/zero-downtime-certificate-rotation-strategies-tools-best-practices/>,
<https://www.gravitee.io/blog/mtls-client-certificate-rotation-without-downtime>

---

## Part C — Kill criteria applied

Kill criteria taken from
`creative-CERT-0001-architecture-decision-framework.md` (Consilium condition 1).
Candidates eliminated by research evidence, **before** deep-dive.

| Candidate | Decision | Reason |
|---|---|---|
| **SCEP as primary enrolment** (ADR-002) | **KILLED** | RFC 8894 is Informational, not Standards Track, and self-documents SHA-1 defaults, no proof-of-possession on renewal, unauthenticated `GetCACaps` enabling downgrade, and no issuance confirmation (F-5). For a greenfield internal CA the framework's kill criterion — "legacy, weak-crypto history, superseded by EST/ACME" — is satisfied on the specification's own text. Not deep-dived. |
| **Single-tier root issuing leaves directly** (ADR-001) | **KILLED** | Framework criterion: killed if any rotation requirement exists. Every surveyed system that documents a hierarchy (Vault, step-ca, EJBCA, AWS Private CA) uses root + intermediate with the root offline (F-14, F-15, F-18, F-19). No survivor supports the single-tier model for a CA that must rotate. |
| **External corporate root** (ADR-001 option 4) | **KILLED (necessity)** | Framework criterion: killed if no corporate root is actually available. No such root is present in this workspace or referenced anywhere in project documentation. Out of scope until one exists. |
| **Full OCSP responder infrastructure** (ADR-005) | **PROVISIONALLY KILLED** | Conditional on ADR-004 landing on short lifetimes. Evidence is strong: the RFC itself concedes replay, DoS and fallback-to-CRL (F-6); Let's Encrypt shut its responders down entirely (F-7); soft-fail client behaviour means the security benefit was largely illusory. **Not final** — ADR-004 ⊗ ADR-005 are decided jointly in Phase 2. |
| **Certificate Transparency as a requirement** (ADR-005 adjacent) | **KILLED as a requirement; retained as optional** | CT's threat model is misissuance by a *publicly-trusted* CA detected by third parties auditing *public* logs (F-9). An internal CA with no public trust and no external relying parties does not inherit that model. Also: RFC 6962 is obsoleted by RFC 9162 (F-8), so any design citing 6962 as current is already stale. |
| **Very-short (hours) leaf lifetimes** (ADR-004) | **RETAINED, gated** | Framework criterion kills these if ADR-002 lands on a non-automated protocol. ACME and EST are both automated and both survive, so the gate is not yet triggered. Retained as a live candidate. |

### Candidates ADDED during research (anti-anchoring)

The framework's option lists are a floor, not a ceiling. Research surfaced four
candidates it did not enumerate:

1. **TLS-ALPN-01 challenge** (ADR-002) — not in the framework's ACME discussion;
   reported as better suited to private environments than http-01/dns-01,
   with a caveat about TLS-terminating proxies (F-3).
2. **Named issuance profiles** (ADR-002/ADR-004) — Let's Encrypt's ACME
   certificate profiles (F-21) and Vault's roles (F-14) are the same idea:
   lifetime and constraints selected by *name* at request time rather than
   passed as per-request parameters. This is a cleaner contract than either.
3. **Name constraints + path length as topology controls** (ADR-001) — AWS
   Private CA enforces both (F-19). They bound what an intermediate may issue
   at the certificate layer, independently of application logic. The framework
   framed ADR-001 purely as a hierarchy-shape question and missed this.
4. **Key rotation coupled to certificate renewal** (ADR-006 adjacent) —
   cert-manager's `rotationPolicy: Always` default (F-13). The framework treated
   rollout mechanics without asking whether the *key* rotates alongside the
   certificate. It is a distinct decision with its own blast radius.

---

## Part D — Findings

Each finding is a single falsifiable statement. Phase 2 ADRs cite these by ID.

| ID | Finding |
|---|---|
| **F-1** | RFC 5280 (May 2008, Proposed Standard) is updated by nine later RFCs but **not obsoleted**; it remains the normative profile for certificate and CRL structure, including name constraints and path-length constraints. |
| **F-2** | RFC 8555 (ACME, March 2019, Proposed Standard) is neither obsoleted nor updated, and defines exactly two challenge types: http-01 and dns-01. |
| **F-3** | ACME's standard challenges assume public reachability — http-01 needs inbound port 80, dns-01 collides with split-horizon DNS; TLS-ALPN-01 suits private environments but conflicts with TLS-terminating proxies. `[secondary]` |
| **F-4** | RFC 7030 (EST) authenticates re-enrolment via the client's existing TLS certificate at `/simplereenroll` — the client proves identity with the credential it is replacing. |
| **F-5** | RFC 8894 (SCEP) is **Informational, not Standards Track**, and its own text documents SHA-1 defaults, absent proof-of-possession on renewal, unauthenticated `GetCACaps` permitting downgrade, and no issuance confirmation. |
| **F-6** | RFC 6960 (OCSP) concedes in its own security considerations that responses are replayable, that the protocol is DoS-susceptible, and that implementations may need CRL fallback. |
| **F-7** | Let's Encrypt shut down its OCSP responders on **6 August 2025**; newly issued certificates carry only a CRL Distribution Point. Stated reasons: client privacy exposure and ~12 billion daily requests for little benefit under soft-fail. |
| **F-8** | RFC 6962 is **obsoleted by RFC 9162** (CT 2.0, December 2021). Both are Experimental. |
| **F-9** | CT's threat model is misissuance by a publicly-trusted CA, detected post-hoc via public logs; it presupposes public trust and third-party auditors, neither of which an internal-only CA has. |
| **F-10** | CA/B Forum ballot SC-081v3 (April 2025) schedules public TLS maximum validity 398 → **200 days (15 Mar 2026)** → 100 days (2027) → **47 days (2029)**; DCV reuse drops to 10 days by 2029. Binds public trust only. |
| **F-11** | **VERIFIED against the primary PDF** (2026-07-31, third attempt, after `pypdf` was installed). NIST SP 800-57 Part 1 Rev. 5, § 5.3.6 item 1 and Table 1 row 1: a **private signature key** has a recommended **maximum cryptoperiod (originator-usage period) of "about one to three years"**, and "a private signature key shall be destroyed at the end of its cryptoperiod". § 5.3.6 further states that when the corresponding public key is CA-certified, "the cryptoperiod for a private signature key ends when the `notAfter` date is reached on the last certificate issued for the public key". `[primary]` |
| **F-11b** | Same source, Table 1 row 2: a **public signature-verification key** has a cryptoperiod of **"several years (depends on key size)"** — materially longer than the private key's usage period. This is the distinction that separates *CA certificate validity* from *CA signing-key usage window*: the certificate may be long-lived while the private key's active signing period must not be. `[primary]` |
| **F-11c** | A CA issuing key **is** a private signature key in NIST's taxonomy, so F-11's 1–3 year bound applies to it directly. An issuing intermediate whose certificate is valid for 5 years therefore implies a 5-year signing window — exceeding the recommendation — unless the key is rotated mid-certificate. `[derived from F-11 + F-11b]` |
| **F-11d** | step-ca's production guidance states root CA validity of **10 years**, with the intermediate aligned to root expiry. This is a **primary** source and is what ADR-001's root-certificate figure rests on. It is consistent with F-11b: the root certificate carries a *public verification key*, for which "several years" is the guidance. |
| **F-12** | cert-manager defaults to a 90-day duration and renews at **2/3 of lifetime**; `renewBeforePercentage` is preferred over absolute `renewBefore` because it adapts when issued duration differs from requested, and a `renewBefore` too close to `duration` causes a renewal loop. |
| **F-13** | cert-manager v1.18.0+ defaults to `rotationPolicy: Always` — the private key is regenerated on every reissuance, so the rotation path is exercised routinely rather than first tested in an emergency. |
| **F-14** | Vault PKI states the lifetime↔revocation coupling directly: short TTLs make revocation less likely to be needed, keep CRLs small, and help the engine scale. It also promotes certificates held only in memory, never written to disk. |
| **F-15** | step-ca defaults leaf certificates to **24 hours**, recommends ≤1 month for host/service certificates, and explicitly prefers short lifetimes **over active revocation**, describing CRL as introducing operational dependencies. |
| **F-16** | step-ca keeps the root key in offline cold storage (HSM or air-gapped, ≥2 copies in separate locations) and supports PKCS#11 HSM or cloud KMS for the **online intermediate** signing key. |
| **F-17** | EJBCA's Crypto Token abstracts key storage over *either* a soft keystore (database file) *or* an HSM PKCS#11 slot — so development and production custody differ by configuration, not by code path. `[secondary]` |
| **F-18** | EJBCA guidance recommends dedicated HSMs not only for the root but also for issuing-CA keys **and OCSP signing keys**. `[secondary]` |
| **F-19** | AWS Private CA enforces `Not After` ≤ issuer's `Not After`, basic constraints, path length, and **name constraints** — but does **not** validate SubjectPublicKeyInfo or Subject Alternative Name, copying both from the CSR unvalidated. |
| **F-20** | AWS Private CA bills monthly **per CA** plus per certificate issued, so cost scales with the number of CAs — penalising per-environment intermediate designs. |
| **F-21** | Let's Encrypt made six-day certificates (~160 h) generally available in January 2026, selected via **ACME certificate profiles**; IP-address certificates must be short-lived by policy. |
| **F-22** | The convergent zero-downtime rotation pattern is a **dual-certificate overlap window** — the replacement is installed while the existing certificate remains valid, eliminating any instant with no valid certificate. `[secondary]` |

---

## Sources

**Primary — standards and specifications**

- RFC 5280 — <https://datatracker.ietf.org/doc/rfc5280/>
- RFC 8555 — <https://datatracker.ietf.org/doc/rfc8555/>
- RFC 7030 — <https://datatracker.ietf.org/doc/rfc7030/>
- RFC 8894 — <https://datatracker.ietf.org/doc/rfc8894/>
- RFC 6960 — <https://datatracker.ietf.org/doc/rfc6960/>
- RFC 6962 — <https://datatracker.ietf.org/doc/rfc6962/>
- RFC 9162 — <https://datatracker.ietf.org/doc/rfc9162/>
- NIST SP 800-57 Part 1 Rev. 5 — <https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-57pt1r5.pdf>
- CA/B Forum SC-081v3 — <https://cabforum.org/2025/04/11/ballot-sc081v3-introduce-schedule-of-reducing-validity-and-data-reuse-periods/>

**Primary — vendor documentation**

- cert-manager — <https://cert-manager.io/docs/usage/certificate/>
- HashiCorp Vault PKI — <https://developer.hashicorp.com/vault/docs/secrets/pki>
- step-ca production — <https://smallstep.com/docs/step-ca/certificate-authority-server-production/>
- EJBCA certificate profiles — <https://docs.keyfactor.com/ejbca/latest/certificate-profiles-overview>
- AWS Private CA — <https://docs.aws.amazon.com/privateca/latest/userguide/PcaWelcome.html>
- Let's Encrypt, ending OCSP — <https://letsencrypt.org/2024/12/05/ending-ocsp>
- Let's Encrypt, 6-day + IP GA — <https://letsencrypt.org/2026/01/15/6day-and-ip-general-availability>
- Let's Encrypt, 6-day announcement — <https://letsencrypt.org/2025/01/16/6-day-and-ip-certs>

**Secondary — flagged inline as `[secondary]`**

- EJBCA HSM guidance — <https://docs.keyfactor.com/ejbca/latest/hardware-security-modules-hsm>, <https://www.ejbca.org/resources/navigating-hsm-options-for-ejbca-pki-a-guide-for-product-development-engineers-and-product-owners/>
- ACME for internal PKI — <https://www.bastionxp.com/blog/acme-certificate-automation-internal-pki-complete-guide/>
- cert-manager ACME troubleshooting — <https://cert-manager.io/docs/troubleshooting/acme/>
- OCSP retirement commentary — <https://www.feistyduck.com/newsletter/issue_121_the_slow_death_of_ocsp>
- Zero-downtime rotation — <https://expiring.at/blog/zero-downtime-certificate-rotation-strategies-tools-best-practices/>, <https://www.gravitee.io/blog/mtls-client-certificate-rotation-without-downtime>
