# secure-space

Umbrella workspace for security-focused services. Each project below is independently deployable and containerised.

## Projects

| Project | Status | Description |
|---------|--------|-------------|
| [`certificate-manager/`](certificate-manager/) | architecture documented (no code yet) | Full X.509 certificate lifecycle — issuance, CSR, expiration monitoring, automatic renewal. Java 21 / Quarkus / Maven / Oracle / Docker. |
| [`key-management-service/`](key-management-service/) | scaffolded (no code yet) | Cryptographic key lifecycle, policy, and inventory. Java 21 / Quarkus / Maven / Docker. |
| [`hsm-management-service/`](hsm-management-service/) | scaffolded (no code yet) | HSM and PKCS#11 provider ownership — the platform's key-custody boundary. Java 21 / Quarkus / Maven / Docker. |

Key custody is being delegated from `certificate-manager`'s `cert-signer` to the
two platform services — see the superseding decision in
[`certificate-manager/documentation/explanation/adr-003-key-custody-delegated-to-platform-services.md`](certificate-manager/documentation/explanation/adr-003-key-custody-delegated-to-platform-services.md).

## Layout

```
secure-space/
├── CLAUDE.md                      # umbrella entry-point (read first)
├── datarim/                        # workflow state (gitignored)
├── documentation/                  # cross-project docs (Diátaxis-split)
├── certificate-manager/            # X.509 certificate lifecycle
│   ├── CLAUDE.md                   # project entry-point
│   └── documentation/              # project docs (Diátaxis-split)
├── key-management-service/         # key lifecycle, policy, inventory
│   ├── CLAUDE.md
│   └── documentation/
└── hsm-management-service/         # HSM / PKCS#11 provider
    ├── CLAUDE.md
    └── documentation/
```

## Getting started

Read [`CLAUDE.md`](CLAUDE.md) first — it explains the umbrella conventions, task-prefix scheme, and Datarim workflow. Then enter the project you want to work on.

## License

TBD
