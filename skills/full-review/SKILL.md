---
name: full-review
description: Complete LayerZero protocol review covering universal smart contract checks and protocol-specific validation. Combines base-review and spec-review.
argument-hint: '[contract-path]'
allowed-tools:
    - Read
    - Glob
    - Grep
    - Task
    - Skill
---

# LayerZero Full Protocol Review

This skill orchestrates base-review and spec-review using parallel sub-agents.

## Execution

Launch **two Task agents in parallel**:

```
Task(subagent_type="general-purpose", description="Base review", prompt="""
Run /base-review {CONTRACT_PATH}
Return the complete report in markdown format.
""")

Task(subagent_type="general-purpose", description="Spec review", prompt="""
Run /spec-review {CONTRACT_PATH}
Return the complete report in markdown format.
If no spec is available, return "SKIP: No spec available for this project type."
""")
```

Where `CONTRACT_PATH` = `$ARGUMENTS[0]`

## Merge Reports

**Handle spec-review skip**: If spec-review returns "SKIP:", note in final report that spec review was skipped.

**Deduplication**: If both phases report the same issue, consolidate into single entry noting "Found in both phases".

After both complete, merge into unified report:

```markdown
# LayerZero Full Protocol Review

**Contract:** [path] | **Date:** [date]

## Summary

| Phase       | ✅  | ❌  | ⚠️  |
| ----------- | --- | --- | --- |
| Base Review |     |     |     |
| Spec Review |     |     |     |

## Critical Issues

[❌ from both phases, deduplicated]

## Warnings

[⚠️ from both phases, deduplicated]

## Phase 1: Base Review

[base-review findings]

## Phase 2: Spec Review

[spec-review findings or "Skipped - no spec available"]
```
