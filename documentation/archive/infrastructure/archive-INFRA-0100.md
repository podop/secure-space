---
id: INFRA-0100
title: Install the missing dev-tools helper set in this Datarim runtime
status: archived
completed_date: 2026-07-31
complexity: L1
type: infra
project: Datarim runtime
related: [CERT-0001]
archive_doc: documentation/archive/infrastructure/archive-INFRA-0100.md
verification_outcome:
  caught_by_verify: 0
  missed_by_verify: 0
  false_positive: 0
  n_a: true
  dogfood_window: "none"
---

# Архив: INFRA-0100 — Установка недостающего набора служебных скриптов

## Начальная задача

Поставить в рабочее окружение набор служебных проверяющих скриптов, которого не хватало всю предыдущую задачу.

## Как решили

- **«Install the missing dev-tools helper set in this Datarim runtime».** Выполнено. Причина оказалась мелкой: окружение собрано из ссылок на репозиторий фреймворка, и шесть из семи областей были связаны правильно — не хватало одной-единственной ссылки. Сам набор скриптов в репозитории уже лежал, и в списке областей установки он тоже уже был указан. То есть установка была неполной, а не устаревшей. Оценка сложности снижена со второго уровня до первого ещё на этапе постановки задачи, по собранным доказательствам.
- **«Служебные проверяющие скрипты доступны, и это подтверждено запуском, а не наличием файла» (уточнение брифа).** Выполнено. Проверялось запуском: распределитель номеров задач вернул следующий свободный номер, проверяльщик ожиданий вернул «пройдено», проверка наличия постановочного файла отработала без ошибок. Отдельно показательно, что проверка происхождения — та самая, которая ни разу не смогла запуститься в предыдущей задаче — сразу нашла настоящую недоработку: архив предыдущей задачи лежал незакоммиченным.

## Артефакты задачи

Изменение — одна ссылка в рабочем окружении, за пределами репозитория:

```
~/.claude/dev-tools -> /home/ypodop/IdeaProjects/datarim/dev-tools
```

Создана тем же способом, каким установщик создаёт остальные шесть. Открыто 131 запись, из них 117 исполняемых. Откат — удаление ссылки; репозиторий фреймворка не затронут.

Рабочие записи (вне общего хранилища): постановка задачи, описание с журналом работ и доказательствами, список ожиданий, документ разбора итогов, карточка итогового состояния.

Задача ничего не добавляет в сам репозиторий: её результат живёт в рабочем окружении, а рабочие записи не отслеживаются системой контроля версий.

## Следующие шаги

Задача закрыта. Три предложения по улучшению самого фреймворка ждут решения оператора — см. раздел о передаче ниже.

---

## Дополнительно для аудита

### verification_outcome

- `caught_by_verify: 0`
- `missed_by_verify: 0`
- `false_positive: 0`
- `n_a: true` — `/dr-verify` was not invoked; L1 routes `init → do → archive`
  with the self-verification hook OFF at this complexity by contract.
- `dogfood_window: "none"`

### Acceptance Criteria

<!-- gate:literal -->
| AC | Status | Evidence |
|---|---|---|
| AC-1 — `$HOME/.claude/dev-tools` resolves to the framework repo's `dev-tools/` | Met | `readlink` → `/home/ypodop/IdeaProjects/datarim/dev-tools` |
| AC-2 — all seven `INSTALL_SCOPES` consistently symlinked | Met | Per-scope audit: agents, skills, commands, templates, scripts, tests, dev-tools all symlinks |
| AC-3 — the six helpers CERT-0001 found absent are present and executable | Met | `test -x` on provenance-gate, spec-graph-gate, check-expectations-checklist, check-deferral-prose, dr-verify-floor, next-free-id |
| AC-4 — at least one helper **executed** with sensible output | Met | 4 helpers run: `next-free-id.sh INFRA` → `INFRA-0101` exit 0; `check-expectations-checklist.sh --verify` → `PASS` exit 0; `check-init-task-presence.sh` → exit 0; `check-deferral-prose.sh` → exit 0 with documented fail-open warnings |
<!-- /gate:literal -->

**Archive-gate results** — the first task in this workspace where every gate ran
mechanically rather than by reading:

<!-- gate:literal -->
| Gate | Result |
|---|---|
| 0.1 clean-git (`pre-archive-check.sh`) | `OK: 1 repo(s) clean` exit 0 |
| 0.12 unpushed (`check-unpushed-commits.sh`) | `clean` exit 0 |
| 0.13 closure reachability (`closure-gate.sh`) | `PASS — all content from 'main' is present in 'origin/main'` exit 0 |
| 0.35 / 0.4 db-relocation / deploy-class classifiers | exit 1 → SKIP (not these classes) |
| 0.45(a) expectations (`check-expectations-checklist.sh --verify`) | `PASS` exit 0 |
| 0.45(b) anti-deferral (`check-deferral-prose.sh`) | exit 0, no findings |
| 0.5 reflection freshness (`reflection-freshness.sh`) | exit 1 → generate (no compliance report for L1) |
| 0.2 / 0.23 / 0.2.5 / 0.3 / 0.43 | SKIP — no VERSION change, no Actions AC, no `requires_runtime_probe`, no networking surface, docs/infra-only |
<!-- /gate:literal -->

### Lessons Learned

Full text in `datarim/reflection/reflection-INFRA-0100.md`.

- **A partial install degrades gates silently.** Twelve absent helpers across
  nine CERT-0001 stages were one missing symlink. Check install topology once,
  not once per stage.
- **A probe that writes is not a probe.** `next-free-id.sh` reserves before it
  prints; a speculative call consumed an ID until released.
- **Never read an exit code through a pipe.** `cmd | tail; echo $?` reported a
  failing gate as passing.

### Operator Handoff

1. **Three Class A evolution proposals await approval** — none applied:
   - **Promoted as a recurrence:** stage-snapshot emission is skipped in
     practice. This is the **second consecutive task** to ship a stale snapshot
     (CERT-0001 at `stage: plan`, INFRA-0100 at `stage: init`). CERT-0001's
     reflection never stamped it, so a key-grep could not detect the repeat —
     the semantic recurrence check did. Proposal adds a **consumer-side
     staleness advisory**, because the producer-side "mandatory" wording has now
     failed twice and nothing detects the miss.
   - Install-completeness probe at session start.
   - Document that `next-free-id.sh` mutates before it prints.
2. **The archived deliverable is not under version control.** The result is a
   symlink in `$HOME/.claude`, which is not a git repository. Neither the
   closure gate nor any commit can attest it — the attestation is the executed-
   helper evidence recorded above. Re-running `install.sh` or `update.sh` on a
   fresh machine reproduces it.
3. **CERT-0001's not-run caveat stands.** Its gates were genuinely not executed
   and its compliance verdict records that honestly. This task fixes the tooling
   forward; it does not retroactively validate that archive.
4. **`/dr-verify` has still never been exercised** in this workspace —
   `dr-verify-floor.sh` is now present but unrun. The first L2+ task to reach QA
   will be its real trial.

### Related

- Task description: `datarim/tasks/INFRA-0100-task-description.md`
- Init task: `datarim/tasks/INFRA-0100-init-task.md`
- Expectations: `datarim/tasks/INFRA-0100-expectations.md`
- Reflection: `datarim/reflection/reflection-INFRA-0100.md`
- Final stage card: `documentation/archive/infrastructure/snapshots/INFRA-0100-final-stage.md`
- PRD / Plan / QA / Compliance: none — L1 routes `init → do → archive`
- Spawned from: `documentation/archive/general/archive-CERT-0001.md` § Operator Handoff item 1
- Follow-ups: none — the task is self-contained
