# Testing — hsm-management-service

> Last updated: 2026-07-31

## Strategy

*[TODO: fill at `/dr-plan`.]* Test pyramid, per component:
- **Unit** — pure logic (policy checks, state transitions)
- **Integration** — component + backing store + Quarkus test extension
- **Container / smoke** — Docker Compose up, exercise the real path end-to-end

## Running

```bash
./mvnw verify        # unit + integration
```
