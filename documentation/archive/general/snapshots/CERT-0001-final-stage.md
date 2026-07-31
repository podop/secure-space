---
task_id: CERT-0001
artifact: stage-snapshot
schema_version: 1
stage: archive
command: /dr-archive
captured_at: 2026-07-31T13:34:01Z
captured_by: agent
recommended_next: /dr-init
options:
  - 1. /dr-init — начать следующую задачу (рекомендуется)
  - 2. Взять пункт из backlog (follow-up задачи CERT-*/INFRA-*)
  - 3. /dr-status — обзор состояния
size_bytes: 1751
truncated: false
---

ARCHIVED. Verdicts: QA v3 ALL_PASS, compliance v2 COMPLIANT_WITH_NOTES.
Certified commit e6242c18f3143ed4eb510cbdface4d54ed147619, pushed to origin/main.

Delivered 3 documents (1092 lines): market research (473 lines, 29 findings),
reference architecture (167), architecture decisions ADR-001..006 (452).

Key decisions: offline root 10y + per-environment issuing intermediates 3y with
name constraints; EST (RFC 7030) over ACME for enrolment; PKCS#11 as sole key
interface with SoftHSM in dev; 7-day leaves with NO leaf revocation (expiry is
the mechanism); dual-certificate overlap rollout with signer-independent rollback.
ADR-004 and ADR-005 authored as one deliberation.

AC-7 outcome: component split REVISED, not confirmed - cert-monitor + cert-roller
merged into cert-renewer; softhsm added as non-production sidecar.

Two gates fired and both were substantive: QA v1 blocked on a rollback instruction
contradicting ADR-005; compliance v1 blocked on 3 decisions resting on secondary
sources. Closing the second changed a shipped value (intermediate 5y -> 3y, NIST
cryptoperiod) and retracted one claim entirely. Primary-tagged findings 2 -> 8.

CAVEAT carried to archive: this install has no dev-tools/, so ~a dozen mandated
validators were performed by reading. Not-run, not passed.
