# Stellar Review Guide

> VM-specific supplement for Stellar/Soroban smart contract review

## Platform Overview

| Attribute     | Value                                             |
| ------------- | ------------------------------------------------- |
| Language      | Rust                                              |
| Framework     | Soroban SDK                                       |
| Address Size  | Variable (normalized to 32 bytes for cross-chain) |
| Authorization | `require_auth()`                                  |
| Storage Model | TTL-based (instance/persistent/temporary)         |
| Upgrade Model | WASM upgrade                                      |

---

## 1. Language & Best Practices

### File Organization

Every contract should follow this structure:

```
contract-name/
├── Cargo.toml
└── src/
    ├── lib.rs              # Entry point with feature gates
    ├── errors.rs           # Error definitions
    ├── events.rs           # Event definitions
    ├── storage.rs          # Storage schema
    ├── [contract].rs       # Main contract implementation
    ├── interfaces/         # Trait definitions (if multiple)
    └── tests/              # Test modules by feature
```

#### lib.rs Pattern

```rust
#![no_std]

cfg_if::cfg_if! {
    if #[cfg(any(not(feature = "library"), feature = "testutils"))] {
        mod my_contract;
        pub use my_contract::{MyContract, MyContractClient};
    }
}

mod errors;
mod events;
mod storage;

pub use errors::*;
pub use events::*;
```

### Naming Conventions

| Element       | Convention             | Example                              |
| ------------- | ---------------------- | ------------------------------------ |
| Files         | snake_case             | `message_lib_manager.rs`             |
| Structs/Enums | PascalCase             | `EndpointV2`, `PacketSent`           |
| Functions     | snake_case             | `send_message`, `get_config`         |
| Constants     | SCREAMING_SNAKE_CASE   | `MAX_COMPOSE_INDEX`                  |
| Error Enums   | `{Component}Error`     | `EndpointError`                      |
| Storage Enums | `{Component}Storage`   | `EndpointStorage`                    |
| Events        | PascalCase noun phrase | `PacketSent`, `OwnershipTransferred` |

### Code Organization

Group related functions with section comments:

```rust
// ============================================================================
// View Functions
// ============================================================================

pub fn get_balance(env: &Env, user: &Address) -> i128 { ... }
pub fn get_total_supply(env: &Env) -> i128 { ... }

// ============================================================================
// Admin Functions
// ============================================================================

#[only_auth]
pub fn set_config(env: &Env, config: &Config) { ... }
```

### Soroban-Specific Patterns

- [ ] Use `#[contracttype]` for custom types
- [ ] Prefer `Symbol` for short strings (≤32 chars)
- [ ] Use `BytesN<32>` for fixed-size byte arrays
- [ ] Leverage `env.storage()` methods appropriately
- [ ] Keep contracts small and modular - favor composition over monolithic designs

---

## 2. Platform Constraints

### Storage Type Selection

| Type           | Use For                           | Key Characteristics                                  |
| -------------- | --------------------------------- | ---------------------------------------------------- |
| **Instance**   | Admin, config, small metadata     | Loaded every call, ~100KB limit, shares contract TTL |
| **Persistent** | User balances, large/keyed data   | Individual TTL, archived when expired, restorable    |
| **Temporary**  | Nonces, sessions, time-bound data | Deleted forever at expiry, lowest cost               |

### Use the #[storage] macro

```rust
#[storage]
pub enum MyStorage {
    /// Instance storage - loaded with every call, no archival
    #[instance(Address)]
    Admin,

    /// Persistent storage - can be archived, use for large/keyed data
    #[persistent(u64)]
    #[default(0)]
    UserBalance { user: Address },

    /// Temporary storage - for transient data
    #[temporary(Bytes)]
    PendingOperation { id: BytesN<32> },
}
```

### DON'T: Store unbounded data in Instance storage

```rust
// BAD - can cause DoS via storage bloat (Instance has ~100KB limit)
#[instance(Vec<Address>)]
AllUsers,

// GOOD - use Persistent with pagination, distribute across storage slots
#[persistent(Address)]
User { index: u32 },

#[instance(u32)]
UserCount,
```

### TTL Best Practices

**Key Rules:**

- Never rely on TTL expiration for security - entries can be extended by anyone
- Store absolute ledger/timestamp boundaries in data for time-based invariants
- Temporary storage is a cost optimization, not a timing enforcement mechanism

```rust
// DON'T rely on TTL for enforcing time bounds - anyone can extend TTL
// DO store expiration timestamps in the data itself

#[contracttype]
pub struct TimeBoundData {
    pub value: i128,
    pub expiry: u64,  // Always include expiry in data, not just TTL
}

// Validate expiry in code
assert_with_error!(env, data.expiry > env.ledger().timestamp(), MyError::Expired);
```

### Resource Limits

- CPU instructions: ~100M per transaction

---

## 3. Security Vulnerabilities

### Security Vulnerability Checklist

#### Critical

- [ ] **Unprotected Storage**: All `env.storage()` writes have `require_auth()` checks
- [ ] **Unprotected Upgrade**: `update_current_contract_wasm()` has admin-only access
- [ ] **Unrestricted Transfer**: `transfer_from` validates authorization

#### High

- [ ] **Unsafe Unwrap/Expect**: No `.unwrap()` on user input or optional data — See [Section 4](#4-error-handling)
- [ ] **Unsafe Map Get**: Using `.get().ok_or()` or proper error handling
- [ ] **Zero/Test Addresses**: No hardcoded addresses, validate admin/owner
- [ ] **Variant Function Consistency**: Functions with similar purposes but different access levels have consistent validation logic
- [ ] **Doc-to-Code Verification**: All doc comment constraints ("must be", "requires", etc.) are enforced in code

#### Medium

- [ ] **DoS via Unbounded Operations**: Loops have size limits, use pagination
- [ ] **Instance Storage Bloat**: No unbounded data in Instance storage — See [Section 2](#dont-store-unbounded-data-in-instance-storage)
- [ ] **State Archival**: Critical Persistent storage has `extend_ttl()` calls — See [Section 2 TTL Best Practices](#ttl-best-practices)

#### Low/Enhancement

- [ ] **Panic Usage**: Using `panic_with_error!()` not `panic!()`
- [ ] **Unsafe Blocks**: No `unsafe { }` blocks
- [ ] **Core Mem Forget**: No `core::mem::forget()` usage
- [ ] **Soroban Version**: Using current stable soroban-sdk
- [ ] **Stale Dependencies**: `contractimport!` references current versions

### Security Scan Patterns

Use these grep patterns to identify high-risk code areas.

**High Priority (MUST scan):**

| Category           | Grep Pattern                                        | Reason                               |
| ------------------ | --------------------------------------------------- | ------------------------------------ |
| Function signature | `pub fn`                                            | Identify public interfaces           |
| Permission check   | `require_auth`, `require_auth_for_args`             | Core permission mechanism            |
| Cross-contract     | `invoke_contract`, `Client::new`                    | External interaction, trust boundary |
| Storage ops        | `\.storage\(\)`, `\.set\(`, `\.get\(`, `\.remove\(` | State change risk                    |
| Dangerous unwrap   | `unwrap\(\)`, `expect\(`                            | Can cause panic                      |
| Assertions         | `assert`, `panic`, `require`                        | Verify logic correctness             |

**Medium Priority (Recommended scan):**

| Category        | Grep Pattern                                | Reason                           |
| --------------- | ------------------------------------------- | -------------------------------- |
| Type conversion | `as u32`, `as i128`, `try_into`             | Potential overflow/truncation    |
| Safe arithmetic | `checked_add`, `checked_sub`, `saturating_` | Check if using safe arithmetic   |
| Event publish   | `events\(\)\.publish`, `\.publish\(`        | Off-chain integration dependency |
| TODO comments   | `TODO`, `FIXME`, `HACK`, `SECURITY`         | Developer-marked risk points     |
| Error define    | `Error`, `#\[contract_error\]`              | Error handling completeness      |

### Authorization

**Standard Soroban auth (`require_auth`):**

```rust
// DO: Call require_auth early, before any state changes
pub fn transfer(env: &Env, from: &Address, to: &Address, amount: i128) {
    from.require_auth();  // Auth first
    // Then proceed with logic...
}

// DO: Use require_auth_for_args for custom argument sets (with care)
// Ensure deterministic, ledger-state-independent mapping
pub fn batch_transfer(env: &Env, from: &Address, transfers: &Vec<Transfer>) {
    from.require_auth_for_args((transfers.clone(),).into_val(env));
}
```

- Call `require_auth` before any state changes
- Sub-contract calls handle their own auth — no need to re-check
- The Soroban host handles signatures, authentication, and replay prevention automatically

**LayerZero macro auth (if using `lz-soroban-sdk`):**

```rust
// #[lz_contract] includes #[ownable] — provides owner management
#[lz_contract]
pub struct MyContract;

// #[only_auth] on a function restricts it to the owner
#[contract_impl]
impl MyContract {
    #[only_auth]
    pub fn admin_function(env: &Env) { ... }
}
```

When reviewing LZ contracts, verify that `#[only_auth]` is applied to all admin-only functions.

### Arithmetic Safety

The workspace has `overflow-checks = true` in the release profile, so plain `+`, `-`, `*`, `/` will panic on overflow.

```rust
// OK - panics on overflow due to overflow-checks = true
let total = amount_a + amount_b;
let result = value * multiplier;
let expiry = env.ledger().timestamp() + grace_period;
```

**Multiply before divide for precision:**

```rust
// BAD - loses precision
let result = amount / divisor * multiplier;

// GOOD - preserves precision
let result = amount * multiplier / divisor;
```

**Never use floating-point for financial calculations:**

```rust
// BAD - floating point precision issues
let fee = amount as f64 * 0.01;

// GOOD - use integer math with smallest unit
let fee = amount * FEE_BPS / 10000;  // basis points
```

### Cross-Contract Data Validation

```rust
// When receiving data from external contracts via Vec<T> or Map<K,V>,
// values converted from Val may not match expected types.
// Validate before storing or using.

pub fn receive_data(env: &Env, external_data: &Vec<i128>) {
    for item in external_data.iter() {
        // Validate each item before use
        assert_with_error!(env, item >= 0, MyError::InvalidData);
    }
}
```

### Variant Function Comparison

When functions have variants (e.g., admin vs user, default vs custom, internal vs external), compare them systematically:

1. **Identify variant pairs** by naming patterns or similar signatures
2. **Create comparison matrix:**

    | Aspect           | Variant A | Variant B | Consistent? |
    | ---------------- | --------- | --------- | ----------- |
    | Input validation |           |           |             |
    | Business logic   |           |           |             |
    | State changes    |           |           |             |
    | Events           |           |           |             |

3. **Flag inconsistencies** - validation in one but not the other is a bug

### Documentation & Comment Verification

**Every comment and doc constraint must accurately reflect the code.** This is a mandatory review step.

**1. Extract constraints** from doc comments — these are implicit specs:

| Keyword in doc        | Required enforcement     |
| --------------------- | ------------------------ |
| "must be/not be"      | assertion                |
| "requires"            | precondition check       |
| "only when/if"        | conditional guard        |
| "valid/invalid"       | validation function      |

**2. Verify all comments match code:**

| What to Verify                | Check                                                     |
| ----------------------------- | --------------------------------------------------------- |
| Function doc vs function body | Does the description match the actual logic?              |
| Parameter docs vs params      | All params listed? Descriptions correct?                  |
| Inline comments               | Does `// does X` match what the next line does?           |
| Code examples in docs         | Do examples match actual generated/expected code?         |
| Module docs                   | Does the module description match its contents?           |

**3. Flag gaps:**

- **❌ CRITICAL**: Comment omits security-critical behavior, describes wrong access control, or claims auth exists when it doesn't (or vice versa)
- **⚠️ WARNING**: Wrong name in comment, incomplete listing, stale comment after refactor, functionally wrong examples

---

## 4. Error Handling

### Use contract_error macro and assert_with_error

```rust
#[contract_error]
pub enum MyError {
    InvalidInput,
    Unauthorized,
    AlreadyExists,
}

// In contract code:
assert_with_error!(env, amount > 0, MyError::InvalidInput);
panic_with_error!(env, MyError::Unauthorized);
```

### DON'T: Use unwrap/expect/panic on user-provided or optional data

```rust
// BAD - can cause unexpected panics on user input or optional data
let value = user_provided_option.unwrap();
let data = some_option.expect("should exist");
panic!("something went wrong");

// GOOD - proper error handling for user input
let value = user_input.ok_or_else(|| MyError::NotFound)?;
assert_with_error!(env, some_option.is_some(), MyError::NotFound);
```

### OK: Use .unwrap() for constructor-initialized storage

Storage values that are **guaranteed to be set in `__constructor`** can safely use `.unwrap()`:

```rust
// OK - these values are always set in __constructor
fn eid(env: &Env) -> u32 {
    MyStorage::eid(env).unwrap()  // Set in constructor, will never be None
}

fn native_token(env: &Env) -> Address {
    MyStorage::native_token(env).unwrap()  // Set in constructor, will never be None
}

// BAD - optional storage that may not be set
fn zro(env: &Env) -> Option<Address> {
    MyStorage::zro(env)  // May not be set, return Option
}
```

When reviewing, verify that any `.unwrap()` on storage reads corresponds to a value initialized in `__constructor`.

---

## 5. Events

### Define events with #[contractevent]

```rust
#[contractevent]
#[derive(Clone, Debug, Eq, PartialEq)]
pub struct Transfer {
    #[topic]  // Indexed for filtering
    pub from: Address,
    #[topic]  // Indexed for filtering
    pub to: Address,
    pub amount: i128,  // Not indexed, in payload only
}

// Publish events
Transfer { from, to, amount }.publish(env);
```

### Event Best Practices

- Events are ephemeral - RPC providers keep ~1 week of history
- Include sufficient data for downstream processes (IDs, old/new values)
- Topics can be mixed types

### Alternative publish syntax

```rust
env.events().publish(
    (symbol_short!("packet"), symbol_short!("sent")),
    PacketSentEvent {
        encoded_packet,
        options,
        send_library,
    },
);
```

**Topic Guidelines:**

- Use `symbol_short!()` for topic keys
- First topic typically identifies event category
- Second topic identifies specific event
- Data payload contains full event details

---

## 6. Constructor Pattern

### Always use \_\_constructor with proper initialization

```rust
pub fn __constructor(
    env: &Env,
    owner: &Address,
    endpoint: &Address,
) {
    // Validate inputs
    assert_with_error!(env, owner != &Address::default(), MyError::InvalidAddress);

    // Initialize storage
    MyStorage::set_endpoint(env, endpoint);

    // Initialize owner (from #[ownable] macro)
    Self::init_owner(env, owner);
}
```

---

## 7. Documentation

### Document public functions with Args/Returns

```rust
/// Transfers tokens from one address to another.
///
/// # Arguments
/// * `from` - The sender address
/// * `to` - The recipient address
/// * `amount` - The amount to transfer
///
/// # Returns
/// The new balance of the recipient
pub fn transfer(env: &Env, from: &Address, to: &Address, amount: i128) -> i128 {
    // ...
}
```

**Note**: Error conditions are defined in `errors.rs` using the `#[contract_error]` macro. Do not duplicate error documentation in `# Panics` sections.

---

## 8. Logic Tracing

**CRITICAL: Never assume an assertion is correct. Trace both paths.**

For every `assert_with_error!`, `if`/`else`, and comparison operator, trace what happens with each possible input:

```
# assert_with_error!(env, payload_hash == empty_hash, Error)

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

---

## References

- [Stellar Docs - Storage Guide](https://developers.stellar.org/docs/build/guides/storage/choosing-the-right-storage)
- [Stellar Docs - Authorization](https://developers.stellar.org/docs/build/smart-contracts/example-contracts/auth)
- [Scout Soroban Detectors](https://github.com/CoinFabrik/scout-soroban)
- [Veridise Security Checklist](https://veridise.com/blog/audit-insights/building-on-stellar-soroban-grab-this-security-checklist-to-avoid-vulnerabilities/)
