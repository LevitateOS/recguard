# recguard

Immutability and integrity policy guard for LevitateOS installed systems.

## Scope

`recguard` enforces installed-system immutability policy.
It is separate from partitioning (`recpart`) and slot state transitions (`recab`).

## Goals

- Validate and enforce immutable runtime expectations.
- Provide explicit policy checks with actionable failures.
- Integrate with A/B slot workflows as a guardrail, not a slot manager.

## Responsibility Boundary

- `recpart`: partition/layout orchestration
- `recstrap`/install wrapper: payload installation
- `recab`: slot select, commit, rollback
- `recguard`: integrity and immutability policy enforcement

## Planned Policy Areas

- Root mount policy validation (expected read-only behavior).
- Writable-path policy validation (`/etc`, `/var`, `/home`, etc. by mode).
- Boot artifact policy checks (expected files and topology).
- A/B target safety checks (prevent mutating active slot by mistake).

## Future Hardening Hooks

- dm-verity presence and config checks.
- UKI/signature policy checks.
- Secure boot state checks where available.
- Optional measured-boot/attestation hooks.

## Planned Interfaces

- `recguard check --mode <ab|mutable> --root <path>`
- `recguard check-live` for running system checks.
- Machine-readable output mode for CI/harness integration.

## Non-Goals

- Creating partitions.
- Installing payload files.
- Setting boot target for next reboot.

## Design Principles

- Fail fast, no silent fallback.
- Deterministic checks and stable diagnostics.
- Clear remediation commands in every failure path.
