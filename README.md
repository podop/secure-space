# secure-space

Umbrella workspace for security-focused services. Each project below is independently deployable and containerised.

## Projects

| Project | Status | Description |
|---------|--------|-------------|
| [`certificate-manager/`](certificate-manager/) | scaffolded (no code yet) | Full X.509 certificate lifecycle — issuance, CSR, expiration monitoring, automatic renewal. Java 21 / Quarkus / Maven / Oracle / Docker. |

## Layout

```
secure-space/
├── CLAUDE.md                      # umbrella entry-point (read first)
├── datarim/                        # workflow state (gitignored)
├── documentation/                  # cross-project docs (Diátaxis-split)
└── certificate-manager/            # first project
    ├── CLAUDE.md                   # project entry-point
    └── documentation/              # project docs (Diátaxis-split)
```

## Getting started

Read [`CLAUDE.md`](CLAUDE.md) first — it explains the umbrella conventions, task-prefix scheme, and Datarim workflow. Then enter the project you want to work on.

## License

TBD
