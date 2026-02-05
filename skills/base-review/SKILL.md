---
name: base-review
description: Universal smart contract security review covering language best practices, platform constraints, and common vulnerabilities. Use when reviewing any smart contract code regardless of protocol.
argument-hint: '[contract-path]'
allowed-tools:
    - Read
    - Glob
    - Grep
    - Task
---

# Universal Smart Contract Review

Performs a comprehensive security review applicable to **any** smart contract, regardless of protocol or business logic.

## Step 1: Detect VM & Load Guide

Detect target VM from file extensions and load the corresponding guide:

| Extension | VM      | Guide                    |
| --------- | ------- | ------------------------ |
| `.rs`     | stellar | `./vm/stellar-review.md` |
| `.sol`    | evm     | `./vm/evm-review.md`     |
| `.move`   | move    | `./vm/move-review.md`    |

**Note**: Currently only Stellar/Soroban is fully supported.

## Step 2: Execute VM-Specific Review

The VM guide contains everything needed for the review:

1. **Language & Best Practices** — File organization, naming, patterns
2. **Platform Constraints** — Storage model, resource limits, authorization
3. **Security Vulnerabilities** — VM-specific checklist and grep patterns
4. **Error Handling** — Proper error patterns for the platform
5. **Testing Requirements** — Platform-specific test patterns

**Follow the VM guide's Security Scan section** for grep patterns to identify high-risk code.

## Step 3: Logic Tracing

**CRITICAL: Never assume an assertion is correct. Trace both paths:**

```
# Example: assert_with_error!(env, payload_hash == empty_hash, Error)

Path A: empty hash [0x00; 32] → should be REJECTED
  payload_hash == empty_hash → TRUE
  assert_with_error! panics when FALSE → No panic
  Result: ACCEPTED ← BUG!

Path B: valid hash [0xab; 32] → should be ACCEPTED
  payload_hash == empty_hash → FALSE
  assert_with_error! panics when FALSE → PANIC
  Result: REJECTED ← BUG!
```

**High-Risk Patterns (Extra Scrutiny):**

| Pattern         | Risk              | Check               |
| --------------- | ----------------- | ------------------- |
| `==` vs `!=`    | Boolean inversion | Trace both paths    |
| `<` vs `<=`     | Boundary          | Test exact boundary |
| Comment vs Code | Mismatch          | Verify alignment    |

## Output Format

```markdown
# Base Review: [contract-name]

**VM:** [vm] | **Date:** [date]

## Critical Issues

[❌ MUST fix items]

## Warnings

[⚠️ Should fix items]

## Checklist Summary

| Category                  | ✅  | ❌  | ⚠️  |
| ------------------------- | --- | --- | --- |
| Language & Best Practices |     |     |     |
| Platform Constraints      |     |     |     |
| Security Vulnerabilities  |     |     |     |
| Error Handling            |     |     |     |

## Detailed Findings

[Findings organized by category from VM guide]
```
