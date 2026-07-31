# Testing — certificate-manager

> Last updated: 2026-07-31

## Strategy

Test pyramid, per component:
- **Unit** — pure logic (CSR validators, policy checks, date/expiry math)
- **Integration** — component + Oracle DB (Testcontainers) + Quarkus test extension
- **Container / smoke** — Docker Compose up, hit HTTP endpoints, verify issuance flow end-to-end

Each Maven module owns its unit and integration tests. Smoke tests live at the project root.

## Test Structure

| Type | Location | Runner | Purpose |
|------|----------|--------|---------|
| Unit | `<module>/src/test/java` | JUnit 5 (via Surefire) | Pure logic |
| Integration | `<module>/src/test/java` (`@QuarkusTest`) | Failsafe + Testcontainers | Component + real Oracle DB |
| Smoke | `smoke/` *(planned)* | Bash + `curl` / `jq` | End-to-end after `docker compose up` |

## How to Run

```bash
# Unit tests (fast, no containers)
./mvnw test

# Full verify — unit + integration
./mvnw verify

# Single module
./mvnw -pl cert-api verify

# End-to-end smoke (once smoke/ exists)
docker compose up -d
./smoke/run.sh
docker compose down -v
```

## Coverage Expectations

- Signer, validator, and expiry-math code paths: **>90%** line coverage.
- HTTP surface: contract tests over full-path assertions.
- Never mock the Oracle DB in integration tests — use Testcontainers.
