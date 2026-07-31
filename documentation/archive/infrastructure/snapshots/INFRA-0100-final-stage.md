---
task_id: INFRA-0100
artifact: stage-snapshot
schema_version: 1
stage: archive
command: /dr-archive
captured_at: 2026-07-31T14:15:59Z
captured_by: agent
recommended_next: /dr-init
options:
  - 1. /dr-init CERT-0006 — сборочный каркас проекта (рекомендуется)
  - 2. /dr-init CERT-0002 — контракт первичной выдачи (bootstrap credential)
  - 3. /dr-status — обзор состояния
size_bytes: 1700
truncated: false
---

ARCHIVED. L1 infra task, one symlink, verified by execution.

Root cause: symlink install with six of seven INSTALL_SCOPES linked; dev-tools
was already in the DEFAULT scope list but its link was never created. Framework
repo v2.59.0 already contained dev-tools/ (131 entries). Partial install, not a
version mismatch. Complexity revised L2 -> L1 on that evidence at /dr-init.

Fix: ln -s /home/ypodop/IdeaProjects/datarim/dev-tools ~/.claude/dev-tools
matching install.sh:524 semantics. Rollback: rm the link.

Proven by EXECUTION, not presence: next-free-id.sh -> INFRA-0101 exit 0;
check-expectations-checklist.sh --verify -> PASS exit 0; check-init-task-presence
-> exit 0; check-deferral-prose -> exit 0. provenance-gate.sh, which could never
run during CERT-0001, returned exit 1 with a real finding (uncommitted archive).

Three self-inflicted defects, all closed in-task: an exit code read through a
pipe (reported 0, true 1); a stray ID reservation left by a probe-style allocator
call (released); the do-stage snapshot never emitted (regenerated here).

Reflection promotes a RECURRING issue: stale stage snapshots hit both CERT-0001
and INFRA-0100. Three Class A proposals await operator approval.
