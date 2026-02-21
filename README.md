# recguard

Policy enforcement and compliance verification for LevitateOS installed-system
mutability models.

## Canonical Specification

`REQUIREMENTS.md` is the canonical source of truth for behavior, interfaces,
and conformance requirements.

- Spec file: `tools/recguard/REQUIREMENTS.md`
- Current version: `0.1.0` (Draft)

## Scope (Summary)

`recguard` verifies that observed system state conforms to declared policy for
`ab` and `mutable` modes. It does not partition disks, install payloads, or
perform slot transitions.

## Tooling Boundary

- `recpart`: partition/layout orchestration
- `recstrap`/install wrapper: payload installation
- `recab`: slot selection and commit/rollback
- `recguard`: policy assertion evaluation and enforcement
