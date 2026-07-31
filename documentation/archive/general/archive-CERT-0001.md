---
id: CERT-0001
title: Market research on PKI best practices + Project Architecture Document
status: archived
completed_date: 2026-07-31
complexity: L3
type: architecture
project: certificate-manager
related: []
archive_doc: documentation/archive/general/archive-CERT-0001.md
verification_outcome:
  caught_by_verify: 0
  missed_by_verify: 0
  false_positive: 0
  n_a: true
  dogfood_window: "none"
---

# Архив: CERT-0001 — Исследование практик PKI и архитектурный документ проекта

## Начальная задача

Сначала изучить рыночные практики, и уже на их основе написать архитектурный документ проекта.

## Как решили

- **«start with the research of the best practices on the market».** Выполнено. Собран отдельный документ с результатами исследования на 473 строки: свод стандартов (шесть RFC, рекомендации по управлению ключами, отраслевой график сокращения сроков действия) и разбор шести работающих систем. Каждая находка пронумерована, чтобы на неё можно было сослаться из решения.
- **«Create architectural documentation based on best practices».** Выполнено. Архитектура разложена на два документа: справочник фактов и документ с шестью решениями. У каждого решения указана находка исследования, на которой оно стоит.
- **«create Project Architecture Document after research».** Выполнено. Порядок соблюдён и проверяем: документ исследования создан раньше архитектурного, и вторая часть работы не начиналась, пока первая не прошла свои проверки.
- **«Сначала — исследование рынка, и только потом документ» (уточнение брифа).** Выполнено. Порядок подтверждён временем создания файлов, а не на словах.
- **«Каждое утверждение подкреплено источником» (уточнение брифа).** Выполнено. Три утверждения, на которых стояли решения, были доведены до первоисточников. Одно подтвердилось, одно оказалось у́же записанного, одно первоисточник не подтвердил вовсе — оно отозвано. Доля утверждений с первичными источниками выросла с двух до восьми.
- **«Project Architecture Document создан и опирается на найденные практики» (уточнение брифа).** Выполнено. В справочнике есть карта частей системы, четыре потока работы, набросок хранения и границы доверия; каждое решение сопровождается ссылкой на находку.
- **«Существующий набросок из пяти компонентов проверен, а не принят на веру» (уточнение брифа).** Выполнено. Набросок пересмотрен: две части, отвечавшие за слежение за сроками и за перевыпуск, слиты в одну, потому что у них одинаковые права и одинаковый доступ к данным — граница между ними ничего не изолировала. Добавлена вспомогательная часть для хранения ключей вне рабочего окружения.
- **«Документация разложена по правилам Diátaxis» (уточнение брифа).** Выполнено. Факты и обоснования разведены по разным разделам, документы ссылаются друг на друга.

## Артефакты задачи

Заверено против коммита `e6242c1`, отправленного в общее хранилище. Рабочее дерево чистое, содержимое коммита совпадает с проверенным.

Создано:

- `certificate-manager/documentation/ephemeral/research/pki-market-research.md` — результаты исследования, 473 строки, 29 пронумерованных находок.
- `certificate-manager/documentation/explanation/architecture-decisions.md` — шесть решений с обоснованиями, 452 строки.

Изменено:

- `certificate-manager/documentation/reference/architecture.md` — заготовка на 36 строк заменена справочником на 167 строк.
- `certificate-manager/documentation/how-to/deployment.md` — переписан раздел отката.
- `certificate-manager/CLAUDE.md` — состав частей системы приведён в соответствие.

Рабочие записи (вне общего хранилища): план, два документа проработки, три отчёта проверки качества, два отчёта соответствия, документ разбора итогов, запись о заверенном коммите.

## Следующие шаги

Задача закрыта. Дальнейшая работа вынесена в отдельные пункты — см. раздел о передаче ниже.

---

## Дополнительно для аудита

### verification_outcome

- `caught_by_verify: 0`
- `missed_by_verify: 0`
- `false_positive: 0`
- `n_a: true` — `/dr-verify` was never invoked as a standalone command. The
  automatic post-step self-verification hook could not run either: this install
  has no `dev-tools/dr-verify-floor.sh`. Gaps were caught by `/dr-qa` and
  `/dr-compliance` instead (2 blocking findings, both closed).
- `dogfood_window: "none"` — no prospective-measurement window active.

### Acceptance Criteria

<!-- gate:literal -->
| AC | Status | Evidence |
|---|---|---|
| AC-1 — research artefact covers the standards corpus | Met | `pki-market-research.md` Part A; V-AC-1 greps RFC 5280/8555/7030/8894/6960/6962 — all present |
| AC-2 — survey covers six implementations | Met | Part B: cert-manager, Vault PKI, step-ca, EJBCA, AWS Private CA, Let's Encrypt/Boulder + kill-criteria section |
| AC-3 — every claim carries a source | Met | V-AC-3 `awk` probe over all `###` subsections → empty output. 8 findings `[primary]`, 1 `[retracted]` |
| AC-4 — PAD at `reference/architecture.md` | Met | 36-line stub replaced; both deferral markers gone (V-AC-4) |
| AC-5 — six decisions resolved with traced rationale | Met | ADR-001..006; V-AC-7 ungrounded-ADR probe empty — every ADR cites `F-n` |
| AC-6 — component map, four flows, schema sketch, trust boundaries | Met | V-AC-4 + V-AC-5 |
| AC-7 — split confirmed or revised with reason | Met (**revised**) | `cert-monitor` + `cert-roller` → `cert-renewer`; `softhsm` added. V-AC-8: active component rows identical across `architecture.md` and `CLAUDE.md` |
| AC-8 — Diátaxis placement | Met | V-AC-9: 0 rationale headings in the reference document |
<!-- /gate:literal -->

**Architecture decisions recorded:**

<!-- gate:literal -->
| ADR | Decision |
|---|---|
| ADR-001 | Offline root (10 y) + per-environment issuing intermediates (3 y), `pathLenConstraint: 0`, name-constrained |
| ADR-002 | EST (RFC 7030) primary; renewal authenticated by the current certificate; named issuance profiles |
| ADR-003 | PKCS#11 as the sole key interface; SoftHSM in dev/CI, HSM/KMS in prod; root never online |
| ADR-004 ⊗ 005 | 7-day leaves, renew at 1/3 remaining (≈56 h retry budget), key rotated every renewal, **no leaf revocation** — authored as one deliberation |
| ADR-006 | Dual-certificate overlap; rollback re-points to retained material and never calls the signer |
<!-- /gate:literal -->

### Lessons Learned

Full text in `datarim/reflection/reflection-CERT-0001.md`.

- **Removing a citation does not make a number correct.** Round 3 dropped the
  dependency on an unverified claim and called the risk closed while still
  shipping the unverified figure; round 4 read the source and found it
  contradicted the standard (~5 y vs a 1–3 y maximum).
- **When one sampled secondary claim proves wrong, the rest become suspect.**
  Checking the remaining three found only one correct as written.
- **A rename is not a rewrite.** The `deployment.md` defect had two halves — a
  stale component name and advice to perform revocation the architecture does
  not provide. A name-oriented sweep fixes the first and leaves the second.

### Operator Handoff

1. **No `dev-tools/` in this install.** Roughly a dozen mandated validators were
   unavailable across every stage — provenance gate, spec-graph gate,
   expectations validator, deferral scanner, live-evidence, deploy-class,
   Playwright detector, session-handoff validator, `dr-verify-floor`. Every
   corresponding check was performed by reading and recorded as such. **They are
   not-run, not passed.** This is the sole reason the compliance verdict is
   COMPLIANT_WITH_NOTES. Follow-up proposed as an `INFRA-` item.
2. **No Task Prefix Registry declared.** `CERT` resolves to `general/` because
   neither `CLAUDE.md` declares a `## Task Prefix Registry` section. Declaring
   one would route future `CERT-*` archives to a dedicated area. This archive is
   correctly placed per the current canonical resolution.
3. **Test-environment gate: SKIP** (documentation-only, ships no runtime
   behaviour). Recorded verbatim per contract — a SKIP is not a verification
   pass.
4. **Two `[secondary]` findings remain** — F-3b (internal-deployment operational
   claims) and F-22a (dual-valid overlap framing). Both are *supporting*
   statements beside a primary sibling that carries the decision, and both are
   labelled deployment experience rather than standards claims. Accepted, low.
5. **Open architecture items, deliberately unresolved**, each named in the
   decisions document § Open items with its own decision: bootstrap credential
   for first issuance, HSM/KMS product selection, consumer health-probe
   contract, audit-log retention and full schema.
6. **Three Class A evolution proposals await approval** (source-tier tagging;
   identifier-sweep guidance; deferral-row blocker requirement). None applied —
   framework-runtime writes require operator approval.

### Related

- Task description: `datarim/tasks/CERT-0001-task-description.md`
- PRD: none — waived at `/dr-init` (operator decision, 2026-07-31)
- Plan: `datarim/plans/CERT-0001-plan.md` (deleted at archive; this document supersedes)
- Design: `datarim/creative/creative-CERT-0001-architecture-documentation-structure.md`, `datarim/creative/creative-CERT-0001-architecture-decision-framework.md`
- QA: `datarim/qa/qa-report-CERT-0001.md` (v1 BLOCKED), `-v2.md` (CONDITIONAL_PASS), `-v3.md` (ALL_PASS)
- Compliance: `datarim/reports/compliance-report-CERT-0001.md` (v1 NON-COMPLIANT), `-v2.md` (COMPLIANT_WITH_NOTES)
- Reflection: `datarim/reflection/reflection-CERT-0001.md`
- Final stage card: `documentation/archive/general/snapshots/CERT-0001-final-stage.md`
- Provenance: `datarim/provenance/CERT-0001.sha` → `e6242c18f3143ed4eb510cbdface4d54ed147619`
- Follow-ups: see § Operator Handoff item 5 and the reflection's follow-up table
