---
name: spec-review
description: Spec-driven review that compares implementation against spec requirements function-by-function.
argument-hint: '[contract-path]'
allowed-tools:
    - Read
    - Glob
    - Grep
    - Task
---

# Spec-Driven Protocol Review

Compare spec documentation against code implementation, validating compliance function-by-function.

## Usage

```bash
/spec-review contracts/oapp/
```

| Argument        | Description   | Example           |
| --------------- | ------------- | ----------------- |
| `$ARGUMENTS[0]` | Contract path | `contracts/oapp/` |

---

## Execution Flow

```
Step 0: Detect project type and match spec
Step 1: Extract function list from spec
Step 2: Compare each function in parallel
Step 3: Aggregate results
```

### Step 0: Detect Project Type and Locate Spec

First, locate the specs directory by finding this SKILL.md file's sibling `specs/` folder:

```
Use Glob to find `**/skills/spec-review/specs/endpoint-spec.md`
Extract the parent directory path → this is {SPECS_DIR}
```

Store `{SPECS_DIR}` as the **absolute path** to the specs directory. All spec file reads in subsequent steps MUST use `{SPECS_DIR}/{matched-spec}.md`.

Then analyze the contract path to determine which spec to use.

**Mapping Rules:**

| Path Pattern                         | Spec File           |
| ------------------------------------ | ------------------- |
| Contains `oapp`                      | `oapp-spec.md`      |
| Contains `endpoint`                  | `endpoint-spec.md`  |
| Contains `uln`                       | `uln302-spec.md`    |
| Contains `treasury`                  | `treasury-spec.md`  |
| Contains `pricefeed` or `price_feed` | `pricefeed-spec.md` |

**If no matching spec found:**

```
No spec available for this project

The contract path `{contract-path}` does not match any supported spec.

Available specs:
- oapp-spec.md       → OApp contracts (path contains "oapp")
- endpoint-spec.md   → Endpoint contracts (path contains "endpoint")
- uln302-spec.md     → MessageLib contracts (path contains "uln")
- treasury-spec.md   → Treasury contracts (path contains "treasury")
- pricefeed-spec.md  → PriceFeed contracts (path contains "pricefeed" or "price_feed")

To add support for a new project type:
1. Create a spec file in {SPECS_DIR}/{project}-spec.md
2. Follow the Spec Format Requirements below
```

**Then stop execution and wait for user action.**

### Step 1: Extract Function List

Read `{SPECS_DIR}/{matched-spec}.md` and extract all function definitions.

**Validation:** Check if spec contains `## Public Interface` section.

**If `## Public Interface` section not found:**

```
Spec file needs improvement

The spec file `{matched-spec}` does not contain a `## Public Interface` section.

Please add this section to the spec file before running spec-review.
See "Spec Format Requirements" below for the required format.
```

**Then stop execution and wait for user to update the spec.**

**Extraction Rules:**

| Spec Marker                 | Extracts      | Detection Method                  |
| --------------------------- | ------------- | --------------------------------- |
| `## Public Interface` table | Function list | Backtick names in Function column |
| `## Events` table           | Event list    | Backtick names in Event column    |

**Output Format:**

```
Function List:
1. {func_name} | {spec_file} | {requirements_summary}
2. ...
```

### Step 2: Parallel Comparison

Spawn a sub-agent for each function:

```
Task(subagent_type="general-purpose", prompt="""
## Function Comparison: {FUNC_NAME}

**Spec Requirements** (from `{SPECS_DIR}/{SPEC_FILE}`):
{SPEC_REQUIREMENTS}

**Tasks**:
1. Find {FUNC_NAME} implementation in {CODE_PATH}
2. If not found, mark as 🚫 Missing
3. If found, compare:
   - Signature: Do parameters/return values match spec?
   - Behavior: Does logic conform to spec description?
   - Error handling: Are all spec-defined errors handled?
   - Events: Are spec-defined events emitted correctly?
   - Doc-to-Code Verification: Verify that doc comment constraints ("must be", "requires", etc.) have corresponding enforcement in code

**Output Format**:
Function: {FUNC_NAME}
Location: {file}:{line} or "Not found"
Result: ✅ Compliant / ❌ Non-compliant / ⚠️ Partial / 🚫 Missing
Differences:
- [Specific differences, if any]
""")
```

**Parallel Strategy:** All function comparison agents launch simultaneously.

### Step 3: Aggregate Results

Collect all sub-agent outputs and generate report with:

1. **Header:** Contract path, spec file, date
2. **Summary table:** Count by status (✅ ❌ ⚠️ 🚫)
3. **Sections by priority:**
    - 🚫 Missing (Critical) — Required by spec but not implemented
    - ❌ Non-Compliant — Signature or behavior differs from spec
    - ⚠️ Partial — Partially matches spec
    - ✅ Compliant — Full list of passing functions

---

## Spec Format Requirements

Each spec document **MUST** contain a standardized Public Interface section to ensure accurate and complete function extraction.

### Required Sections

#### `## Public Interface`

List all public functions in table format:

```markdown
## Public Interface

| Function | Signature                               | Auth | Description              |
| -------- | --------------------------------------- | ---- | ------------------------ |
| `send`   | `send(params, refundAddress) → receipt` | -    | Send cross-chain message |
| `quote`  | `quote(params, sender) → fee`           | -    | Quote message fee        |
| ...      | ...                                     | ...  | ...                      |
```

**Table Columns:**

| Column      | Required | Description                                              |
| ----------- | -------- | -------------------------------------------------------- |
| Function    | ✅       | Function name wrapped in backticks                       |
| Signature   | ✅       | Function signature (parameters, return values)           |
| Auth        | Optional | Authorization (e.g., Owner, OApp/delegate, - for public) |
| Description | ✅       | Brief description                                        |

#### `## Events`

List all events in table format:

```markdown
## Events

| Event        | Trigger      | Key Fields             |
| ------------ | ------------ | ---------------------- |
| `PacketSent` | Message sent | encodedPacket, options |
| ...          | ...          | ...                    |
```

## Important Rules

1. **Dynamic function extraction** — Never hardcode function lists
2. **One agent per function** — Parallel execution, isolated context
3. **Reference line numbers** — e.g., `messaging_channel.rs:183`
4. **Trace logic paths** — Never assume assertions are correct; trace both paths
