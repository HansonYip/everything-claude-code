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

### General Rust Best Practices

**Prefer immutability:**

```rust
// Prefer let over let mut
let value = calculate_value();

// Only use mut when necessary
let mut accumulator = 0;
for item in items {
    accumulator += item.value;
}
```

**Use pattern matching over if-else chains:**

```rust
// GOOD - match extracts value directly
match option {
    Some(value) => { /* use value */ }
    None => { /* handle none */ }
}

// Also GOOD - if-let for single variant
if let Some(value) = option {
    // use value
}
```

**Use exhaustive pattern matching:**

```rust
// BAD - wildcard hides new variants
match status {
    Status::Active => { ... }
    _ => { ... }  // Dangerous: new variants silently fall through
}

// GOOD - explicit handling
match status {
    Status::Active => { ... }
    Status::Pending => { ... }
    Status::Closed => { ... }
}
```

**Keep functions small and focused:**

- Functions should do one thing well
- If a function is > 50 lines, consider splitting it
- Extract helper functions for reusable logic
- Separate pure logic from effectful operations

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
- Memory: Limited by Soroban VM
- Storage reads/writes: Counted and limited

---

## 3. Security Vulnerabilities

### Security Concerns

| Concern                | Description                                 |
| ---------------------- | ------------------------------------------- |
| TTL expiration attacks | Expired data requires extra gas to restore  |
| Auth confusion         | Missing or incorrect `require_auth()` calls |
| Address type mismatch  | Confusing `Address` with `BytesN<32>`       |

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

**Use the three-tier auth system:**

```rust
// Tier 1: Contract uses #[lz_contract] (includes #[ownable])
#[lz_contract]
pub struct MyContract;

// Tier 2: Protected functions use #[only_auth]
#[contract_impl]
impl MyContract {
    #[only_auth]
    pub fn admin_function(env: &Env) {
        // Only owner can call
    }
}

// Tier 3: User auth with require_auth
pub fn user_action(env: &Env, user: &Address, amount: i128) {
    user.require_auth();
    // User authorized this call
}
```

**require_auth Best Practices:**

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

// DON'T: Forget that sub-contract calls handle their own auth
// Just calling a sub-contract that uses require_auth is sufficient
```

**Security Notes:**

- The Soroban host handles signatures, authentication, and replay prevention automatically
- The `Address` type works uniformly for Stellar accounts, contracts, and contract accounts
- Test auth logic with `env.mock_all_auths()` and verify with `env.auths()`

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

**Use correct operators:**

```rust
// BAD - ^ is XOR, not exponentiation
let result = base ^ exponent;

// GOOD - use pow() for exponentiation
let result = base.pow(exponent);
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

### Doc-to-Code Verification

Documentation constraints are specifications that code must enforce:

1. **Extract constraints** from doc comments:

    - "must be/not be" → assertion required
    - "requires" → precondition check required
    - "only when/if" → conditional guard required
    - "valid/invalid" → validation function required

2. **Verify enforcement** - for each constraint, locate corresponding code check

3. **Flag gaps** - documented constraints without code enforcement are bugs

### Comment Accuracy Verification

**Every comment must accurately reflect the code it describes.** This is a mandatory review step.

For each file under review:

1. **Read every comment** (doc comments `///` `//!`, inline comments `//`)
2. **Read the code** it describes
3. **Verify alignment** — does the comment match what the code actually does?

**Check these specifically:**

| What to Verify | How |
| --- | --- |
| Function doc vs function body | Does the doc description match the actual logic? |
| Parameter docs vs actual params | Are all params listed? Are descriptions correct? |
| Method/field listings | Are all items listed? Any omissions? |
| Code examples in docs | Do examples match actual generated/expected code? |
| Inline comments | Does `// does X` actually describe what the next line does? |
| Module docs | Does the module description match its actual contents? |
| Trait method counts | If doc says "both A and B", does code really only produce A and B? |

**Flag as ⚠️ WARNING:**
- Wrong method/function name in comment
- Incomplete listing (e.g., lists 3 of 6 methods)
- Stale comment after refactor (describes old logic)
- "Conceptual" code examples that are functionally wrong

**Flag as ❌ CRITICAL:**
- Comment omits security-critical behavior (e.g., missing `freeze()` from upgrade docs)
- Comment describes wrong access control model
- Comment says auth is required but code doesn't enforce it (or vice versa)

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

### Error Mapping (EVM → Stellar)

```
EVM                    Stellar/Rust
─────────────────────────────────────
LZ_InvalidNonce     →  Error::InvalidNonce
LZ_Unauthorized     →  Error::Unauthorized
LZ_NotRegistered    →  Error::NotRegistered
LZ_InvalidEid       →  Error::InvalidEndpointId
LZ_InsufficientFee  →  Error::InsufficientFee
```

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

## 7. Testing

### Use setup helpers and descriptive test names

```rust
struct TestSetup<'a> {
    env: Env,
    admin: Address,
    contract: MyContractClient<'a>,
}

fn setup() -> TestSetup<'static> {
    let env = Env::default();
    let admin = Address::generate(&env);
    let contract_id = env.register(MyContract, (&admin,));
    let contract = MyContractClient::new(&env, &contract_id);
    TestSetup { env, admin, contract }
}

#[test]
fn test_transfer_succeeds_with_sufficient_balance() {
    let setup = setup();
    // Arrange
    // Act
    // Assert
}

#[test]
#[should_panic(expected = "InsufficientBalance")]
fn test_transfer_fails_with_insufficient_balance() {
    // ...
}
```

### Test authorization logic

```rust
#[test]
fn test_auth_required() {
    let setup = setup();
    setup.env.mock_all_auths();

    // Call function
    setup.contract.transfer(&from, &to, &100);

    // Verify auth was required
    assert_eq!(
        setup.env.auths(),
        [(from.clone(), AuthorizedInvocation { ... })]
    );
}
```

---

## 8. Documentation

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

## 9. Review Workflow

### Review Scope

When reviewing code:

- **Skip test code**: Do not review files in `tests/` directories or code inside `#[cfg(test)]` modules
- Focus on production contract code only

### Build & Test

```bash
# 1. Format and lint
cargo fmt --check
cargo clippy -- -D warnings

# 2. Build
stellar contract build

# 3. Test
cargo test
```

---

## References

- [Stellar Docs - Storage Guide](https://developers.stellar.org/docs/build/guides/storage/choosing-the-right-storage)
- [Stellar Docs - Authorization](https://developers.stellar.org/docs/build/smart-contracts/example-contracts/auth)
- [Scout Soroban Detectors](https://github.com/CoinFabrik/scout-soroban)
- [Veridise Security Checklist](https://veridise.com/blog/audit-insights/building-on-stellar-soroban-grab-this-security-checklist-to-avoid-vulnerabilities/)
