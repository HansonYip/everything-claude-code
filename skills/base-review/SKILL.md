---
name: base-review
description: Reviews smart contract code for language best practices, platform constraints, common security vulnerabilities, and documentation accuracy — independent of protocol-specific business logic.
argument-hint: '[contract-path]'
allowed-tools:
    - Read
    - Glob
    - Grep
    - Task
---

# Universal Smart Contract Review

## Workflow

### Step 0: Validate Input

If no contract path argument is provided, **stop immediately** and return:

> ❌ **Missing argument**: Please provide a contract path. Usage: `/base-review [contract-path]`

Do not proceed with the review.

### Step 1: Detect VM & Load Guide

Detect target VM from file extensions and load the corresponding guide:

| Extension | VM      | Guide                    | Verification                              |
| --------- | ------- | ------------------------ | ----------------------------------------- |
| `.rs`     | stellar | `./vm/stellar-review.md` | `Cargo.toml` contains `soroban-sdk` dep   |

**Detection steps:**
1. Match the file extension against the table above.
2. If matched, run the verification check (e.g., look for `soroban-sdk` in the nearest `Cargo.toml`).
3. If verification fails, **stop immediately** and return:

> ❌ **VM verification failed**: File is `.rs` but no `soroban-sdk` dependency found in `Cargo.toml`. This does not appear to be a Soroban contract.

If the file extension does not match any row in the table, **stop immediately** and return:

> ❌ **Unsupported VM**: Could not detect a supported VM from the file extension `.[ext]`. Supported extensions: `.rs` (Stellar/Soroban).

Do not proceed with the review.

### Step 2: Execute VM-Specific Review

The VM guide contains everything needed for the review:

1. **Language & Best Practices** — File organization, naming, patterns
2. **Platform Constraints** — Storage model, resource limits, authorization
3. **Security Vulnerabilities** — VM-specific checklist and grep patterns
4. **Error Handling** — Proper error patterns for the platform
5. **Events** — Event emission and structure
6. **Constructor Pattern** — Initialization and setup patterns
7. **Documentation** — Doc comments and inline comment accuracy
8. **Logic Tracing** — Trace every assertion and conditional through both paths

**Follow the VM guide's Security Scan section** for grep patterns to identify high-risk code.

## Output Format

```markdown
# Base Review: [contract-name]

**VM:** [vm] | **Date:** [date]

## Critical Issues

[❌ MUST fix items]

## Warnings

[⚠️ Should fix items]

## Info

[ℹ️ Low-severity / enhancement items]

## Checklist Summary

| Category                  | ✅  | ❌  | ⚠️  |
| ------------------------- | --- | --- | --- |
| Language & Best Practices |     |     |     |
| Platform Constraints      |     |     |     |
| Security Vulnerabilities  |     |     |     |
| Error Handling            |     |     |     |
| Events                    |     |     |     |
| Constructor Pattern       |     |     |     |
| Documentation             |     |     |     |

## Detailed Findings

[Findings organized by category from VM guide]
```
