# Architecture — hsm-management-service

> Last updated: 2026-07-31
> **Category:** *reference* (information-oriented) — facts for lookup. Rationale
> belongs in [`../explanation/`](../explanation/), not here.

## Overview

hsm-management-service owns the HSM and the PKCS#11 provider for the secure-space platform.

*[TODO: describe the system once decisions land. Do not write rationale here —
this is the reference surface; ADRs go to `../explanation/`.]*

## Components

*[TODO — one row per container.]*

| Component | Path | Runtime | Purpose |
|-----------|------|---------|---------|

## Trust boundaries

*[TODO — state what may reach key material and what may not.]*

## Data flows

*[TODO.]*

## Storage schema

*[TODO.]*

## Decisions

Rationale lives in [`../explanation/`](../explanation/).
See also the pending supersession of certificate-manager's ADR-003, recorded in
[`../../CLAUDE.md`](../../CLAUDE.md) § Boundary with certificate-manager.
