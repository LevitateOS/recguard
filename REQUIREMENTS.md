# recguard Requirements

**Version:** 0.1.0  
**Status:** Draft  
**Last Updated:** 2026-02-21

This document defines the requirements for `recguard`: policy enforcement and
compliance verification for LevitateOS installed-system mutability models.
`recguard` validates that observed runtime/disk state matches declared policy
for `ab` or `mutable` mode and fails with actionable diagnostics when it does
not.

---

## Table of Contents

1. [Overview](#1-overview)
2. [Terms](#2-terms)
3. [Design Principles](#3-design-principles)
4. [Policy Source and Ownership](#4-policy-source-and-ownership)
5. [Mode Assertions](#5-mode-assertions)
6. [Evidence and Evaluation Model](#6-evidence-and-evaluation-model)
7. [Failure Semantics](#7-failure-semantics)
8. [CLI Interface](#8-cli-interface)
9. [Security Requirements](#9-security-requirements)
10. [Conformance and Traceability](#10-conformance-and-traceability)

---

## 1. Overview

### 1.1 Purpose

`recguard` enforces installed-system policy conformance:

- Verifies mode-specific mutability expectations (`ab` vs `mutable`)
- Verifies slot safety constraints for A/B workflows
- Produces deterministic, actionable failure output for operators/automation

### 1.2 Scope

This specification covers:

- Policy assertion evaluation for installed systems and offline sysroots
- Mode-specific pass/fail rules
- Error classification, exit codes, and diagnostics
- Machine-readable output for harness/CI

This specification does not cover:

- Partition creation (`recpart`)
- Payload extraction (`recstrap`)
- Slot selection/commit/rollback (`recab`)
- Package composition/update planning (`recipe`)

### 1.3 Responsibility Boundary

**REQ-SCOPE-001**: `recguard` MUST be a policy verifier/enforcer and MUST NOT
perform partitioning or slot-transition operations directly.

**REQ-SCOPE-002**: `recguard` MUST support both live-system checks and offline
sysroot checks.

---

## 2. Terms

| Term | Definition |
|------|------------|
| Policy source | Canonical policy artifact/schema produced and owned outside `recguard` (primarily `recipe`-owned policy declarations). |
| Assertion | A single required condition evaluated by `recguard` that yields pass/fail. |
| Mode | Installation mutability model: `ab` or `mutable`. |
| Active slot | Currently booted slot in an A/B model. |
| Inactive slot | Non-booted slot targeted for next composition/update. |
| Evidence | Observed facts used to evaluate assertions (mount data, labels, boot metadata, slot state, files). |

---

## 3. Design Principles

### 3.1 Policy Fidelity

**REQ-DESIGN-001**: `recguard` MUST evaluate declared policy as written and
MUST NOT silently reinterpret assertions.

**REQ-DESIGN-002**: `recguard` MUST fail on unresolved policy ambiguity instead
of guessing.

### 3.2 Determinism

**REQ-DESIGN-010**: Given identical policy input and identical observed system
state, `recguard` MUST produce identical pass/fail results.

**REQ-DESIGN-011**: Assertion evaluation order MUST be deterministic.

### 3.3 Fail Fast

**REQ-DESIGN-020**: Critical policy violations MUST hard-fail immediately.

**REQ-DESIGN-021**: `recguard` MUST NOT downgrade critical failures to warnings
in default operation.

### 3.4 Automation Ready

**REQ-DESIGN-030**: `recguard` MUST be non-interactive in default operation.

**REQ-DESIGN-031**: `recguard` MUST provide a machine-readable output mode.

---

## 4. Policy Source and Ownership

### 4.1 Source of Truth

**REQ-POLICY-001**: Runtime mutability assertions MUST be sourced from a
canonical policy artifact owned by policy-authoring tooling (primarily `recipe`
policy declarations).

**REQ-POLICY-002**: `recguard` MUST NOT define a parallel default policy that
conflicts with canonical policy artifacts.

### 4.2 Schema Compatibility

**REQ-POLICY-010**: `recguard` MUST validate policy schema version before
evaluation and fail with a clear compatibility error on mismatch.

**REQ-POLICY-011**: Unknown required policy fields MUST fail closed.

### 4.3 Minimal Local Defaults

**REQ-POLICY-020**: Any local fallback policy (if supported) MUST be explicitly
opt-in and MUST be marked non-canonical in output.

---

## 5. Mode Assertions

### 5.1 Shared Assertions

**REQ-MODE-001**: Every run MUST evaluate:

- mount-policy assertions
- writable-path assertions
- boot-artifact assertions
- slot-safety assertions when mode is `ab`

### 5.2 `ab` Mode Assertions

**REQ-MODE-010**: In `ab` mode, `recguard` MUST enforce active/inactive slot
safety: destructive or composition-target assertions MUST resolve to inactive
slot by default.

**REQ-MODE-011**: In `ab` mode, assertions MUST confirm that required immutable
roots and policy-declared read-only paths are not writable under normal
operation.

**REQ-MODE-012**: In `ab` mode, assertions MUST confirm policy-declared
writable state paths exist and are writable where expected.

### 5.3 `mutable` Mode Assertions

**REQ-MODE-020**: In `mutable` mode, assertions MUST permit writable root
semantics according to policy.

**REQ-MODE-021**: `mutable` mode MUST still enforce critical safety assertions
that prevent accidental target ambiguity or invalid mount topology.

### 5.4 Cross-Mode Consistency

**REQ-MODE-030**: If observed state simultaneously violates both mode contracts
or cannot be disambiguated, `recguard` MUST fail as ambiguous-policy-state.

---

## 6. Evidence and Evaluation Model

### 6.1 Evidence Sources

**REQ-EVID-001**: `recguard` MUST explicitly declare and collect evidence from
stable system interfaces (for example: mount tables, block metadata, slot state
metadata, boot artifact paths).

**REQ-EVID-002**: Missing required evidence MUST be a hard failure unless
policy explicitly marks it optional.

### 6.2 Evaluation Pipeline

**REQ-EVID-010**: Evaluation MUST follow deterministic phases:

1. Load/validate policy
2. Collect evidence
3. Evaluate assertions
4. Emit result

**REQ-EVID-011**: Assertion results MUST include assertion identifier, expected
condition, observed evidence summary, and verdict.

---

## 7. Failure Semantics

### 7.1 Severity Classes

**REQ-FAIL-001**: `recguard` MUST classify assertion failures with stable
severity classes (`CRITICAL`, `HIGH`, `MEDIUM`, `LOW`).

**REQ-FAIL-002**: `CRITICAL` failures MUST cause non-zero exit and overall fail.

### 7.2 Exit Codes

**REQ-FAIL-010**: CLI MUST publish stable exit codes for:

- policy load/validation failure
- evidence collection failure
- assertion failure (policy non-conformance)
- internal execution error

### 7.3 Diagnostics

**REQ-FAIL-020**: Every failure output MUST include:

- component (`policy`, `evidence`, `assertion`, `runtime`)
- expectation
- observed state summary
- concrete remediation guidance

**REQ-FAIL-021**: Output MUST never claim success if any `CRITICAL` assertion
failed.

---

## 8. CLI Interface

### 8.1 Commands

**REQ-CLI-001**: `recguard` MUST provide:

- `recguard check --mode <ab|mutable> --root <path>`
- `recguard check-live --mode <ab|mutable>`

### 8.2 Output Modes

**REQ-CLI-010**: CLI MUST support a machine-readable output format (for
example `--json`) suitable for CI and stage harnesses.

**REQ-CLI-011**: Default success output SHOULD be concise; failure output MUST
be explicit and actionable.

### 8.3 Quiet and Strict Operation

**REQ-CLI-020**: CLI MUST avoid interactive prompts in default paths.

**REQ-CLI-021**: A strict mode (default behavior) MUST fail on unresolved
policy ambiguity.

---

## 9. Security Requirements

### 9.1 No Silent Policy Bypass

**REQ-SEC-001**: `recguard` MUST NOT silently bypass failing assertions.

**REQ-SEC-002**: Optional compatibility exceptions MUST be explicitly flagged
in output and MUST be disabled by default.

### 9.2 Integrity-Hook Awareness

**REQ-SEC-010**: `recguard` SHOULD support policy assertions for integrity
signals (for example dm-verity presence, signed-boot expectations) where
declared by policy source.

**REQ-SEC-011**: If integrity assertions are policy-required but evidence is
missing, `recguard` MUST fail closed.

---

## 10. Conformance and Traceability

### 10.1 Requirement IDs

**REQ-CONF-001**: Conformance tests SHOULD reference one or more `REQ-*`
identifiers from this specification.

### 10.2 Minimum Coverage

Implementations SHOULD include tests for:

- policy source validation (`REQ-POLICY-001`, `REQ-POLICY-010`)
- mode assertion behavior (`REQ-MODE-010`, `REQ-MODE-020`)
- ambiguity fail-closed behavior (`REQ-MODE-030`, `REQ-CLI-021`)
- deterministic evaluation (`REQ-DESIGN-010`, `REQ-EVID-010`)
- failure semantics (`REQ-FAIL-001`, `REQ-FAIL-021`)

### 10.3 Traceability Matrix

| Requirement | Planned Test Target | Notes |
|-------------|---------------------|-------|
| `REQ-POLICY-001`, `REQ-POLICY-010`, `REQ-POLICY-011` | `tests/policy_load.rs` | Canonical policy loading, schema/version checks, fail-closed on unknown required fields. |
| `REQ-MODE-010`, `REQ-MODE-011`, `REQ-MODE-012` | `tests/mode_ab.rs` | A/B slot-safety, read-only path expectations, writable-state expectations. |
| `REQ-MODE-020`, `REQ-MODE-021` | `tests/mode_mutable.rs` | Mutable semantics with mandatory safety checks still enforced. |
| `REQ-MODE-030`, `REQ-CLI-021` | `tests/ambiguity.rs` | Ambiguous state handling must fail closed. |
| `REQ-EVID-001`, `REQ-EVID-002`, `REQ-EVID-010` | `tests/evidence_pipeline.rs` | Evidence source collection and deterministic evaluation pipeline. |
| `REQ-FAIL-001`, `REQ-FAIL-010`, `REQ-FAIL-020`, `REQ-FAIL-021` | `tests/failures.rs` | Severity classes, stable exit codes, and mandatory diagnostic fields. |
| `REQ-CLI-001`, `REQ-CLI-010`, `REQ-CLI-011`, `REQ-CLI-020` | `tests/cli.rs` | Command shape, JSON output contract, and non-interactive behavior. |
| `REQ-SEC-001`, `REQ-SEC-002`, `REQ-SEC-011` | `tests/security.rs` | No silent bypass, explicit exception signaling, fail-closed for required integrity evidence. |

