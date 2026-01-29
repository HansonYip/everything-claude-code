---
name: stellar-coding-style
description: Enforces Soroban Stellar smart contract coding style and Rust best practices. Use when writing or reviewing Soroban Rust code for style compliance.
---

# Soroban Stellar Coding Style Guide

Apply these coding standards when writing or reviewing Soroban smart contracts.

## 1. File Organization

### Module Structure

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

### lib.rs Pattern

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

### Keep Contracts Small and Modular

Favor composing smaller, reusable contracts over monolithic designs. This makes code easier to reason about and contain potential vulnerabilities.

## 2. Naming Conventions

| Element       | Convention             | Example                              |
| ------------- | ---------------------- | ------------------------------------ |
| Files         | snake_case             | `message_lib_manager.rs`             |
| Structs/Enums | PascalCase             | `EndpointV2`, `PacketSent`           |
| Functions     | snake_case             | `send_message`, `get_config`         |
| Constants     | SCREAMING_SNAKE_CASE   | `MAX_COMPOSE_INDEX`                  |
| Error Enums   | `{Component}Error`     | `EndpointError`                      |
| Storage Enums | `{Component}Storage`   | `EndpointStorage`                    |
| Events        | PascalCase noun phrase | `PacketSent`, `OwnershipTransferred` |

## 3. Error Handling

### DO: Use contract_error macro and assert_with_error

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

## 4. Storage Patterns

### Storage Type Selection

| Type           | Use For                           | Key Characteristics                                  |
| -------------- | --------------------------------- | ---------------------------------------------------- |
| **Instance**   | Admin, config, small metadata     | Loaded every call, ~100KB limit, shares contract TTL |
| **Persistent** | User balances, large/keyed data   | Individual TTL, archived when expired, restorable    |
| **Temporary**  | Nonces, sessions, time-bound data | Deleted forever at expiry, lowest cost               |

### Use the #[storage] macro with appropriate types

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

**Key Rules:**

- Never rely on TTL expiration for security - entries can be extended by anyone
- Store absolute ledger/timestamp boundaries in data for time-based invariants
- Temporary storage is a cost optimization, not a timing enforcement mechanism

## 5. Authorization

### Use the three-tier auth system

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

### require_auth Best Practices

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

## 6. Events

### Define events with appropriate topics

```rust
#[contractevent]
#[derive(Clone, Debug, Eq, PartialEq)]
pub struct Transfer {
    #[topic]  // Index for filtering (max 4 topics)
    pub from: Address,
    #[topic]  // Index for filtering
    pub to: Address,
    pub amount: i128,  // Not indexed, in payload only
}

// Publish events
Transfer { from, to, amount }.publish(env);
```

**Event Best Practices:**

- Maximum 4 topics per event for RPC filtering
- Events are ephemeral - RPC providers keep ~1 week of history
- Include sufficient data for downstream processes (IDs, old/new values)
- Topics can be mixed types

## 7. Constructor Pattern

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

## 8. Arithmetic Safety

### Use plain arithmetic operators - overflow panics automatically

The workspace has `overflow-checks = true` in the release profile, so plain `+`, `-`, `*`, `/` will panic on overflow. This is the desired behavior for invalid inputs.

```rust
// OK - panics on overflow due to overflow-checks = true
let total = amount_a + amount_b;
let result = value * multiplier;
let expiry = env.ledger().timestamp() + grace_period;
```

### Multiply before divide for precision

```rust
// BAD - loses precision
let result = amount / divisor * multiplier;

// GOOD - preserves precision
let result = amount * multiplier / divisor;
```

### Never use floating-point for financial calculations

```rust
// BAD - floating point precision issues
let fee = amount as f64 * 0.01;

// GOOD - use integer math with smallest unit
let fee = amount * FEE_BPS / 10000;  // basis points
```

## 9. Code Organization

### Group related functions with section comments

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

## 10. Documentation

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

## 11. Testing

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

## 12. General Rust Best Practices

### Prefer immutability

```rust
// Prefer let over let mut
let value = calculate_value();

// Only use mut when necessary
let mut accumulator = 0;
for item in items {
    accumulator += item.value;
}
```

### Use pattern matching over if-else chains

```rust
// BAD
if option.is_some() {
    let value = option.unwrap();
    // use value
} else {
    // handle none
}

// GOOD
match option {
    Some(value) => { /* use value */ }
    None => { /* handle none */ }
}

// Also GOOD
if let Some(value) = option {
    // use value
}
```

### Use exhaustive pattern matching

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

### Keep functions small and focused

- Functions should do one thing well
- If a function is > 50 lines, consider splitting it
- Extract helper functions for reusable logic
- Separate pure logic from effectful operations

## 13. Security Best Practices

### Never use unsafe blocks

```rust
// BAD - bypasses Rust's memory safety guarantees
unsafe {
    // dangerous code
}

// GOOD - use safe Rust patterns
// If unsafe seems needed, reconsider the design
```

### Avoid unbounded operations (DoS prevention)

```rust
// BAD - unbounded loop can exhaust resources
pub fn process_all(env: &Env, items: &Vec<Item>) {
    for item in items.iter() {
        process(item);
    }
}

// GOOD - bounded iteration with pagination
pub fn process_batch(env: &Env, items: &Vec<Item>, start: u32, limit: u32) {
    let end = (start + limit).min(items.len() as u32);
    for i in start..end {
        process(&items.get(i).unwrap());
    }
}
```

### Validate cross-contract data

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

### Protect contract upgrades

```rust
// Contract WASM updates should be protected by admin authorization
#[only_auth]
pub fn upgrade(env: &Env, new_wasm_hash: &BytesN<32>) {
    env.deployer().update_current_contract_wasm(new_wasm_hash);
}
```

### Use correct operators

```rust
// BAD - ^ is XOR, not exponentiation
let result = base ^ exponent;

// GOOD - use pow() for exponentiation
let result = base.pow(exponent);
```

## Review Scope

When reviewing code:

- **Skip test code**: Do not review files in `tests/` directories or code inside `#[cfg(test)]` modules
- Focus on production contract code only

## Output Format

When reviewing code, report style issues as:

1. **Category**: Organization/Naming/Error/Storage/Auth/Security/etc.
2. **File**: path/to/file.rs:line
3. **Issue**: Description of the style violation
4. **Current**: The problematic code
5. **Suggested**: The corrected code

## References

- [Stellar Docs - Storage Guide](https://developers.stellar.org/docs/build/guides/storage/choosing-the-right-storage)
- [Stellar Docs - Authorization](https://developers.stellar.org/docs/build/smart-contracts/example-contracts/auth)
- [Scout Soroban Detectors](https://github.com/CoinFabrik/scout-soroban)
- [Veridise Security Checklist](https://veridise.com/blog/audit-insights/building-on-stellar-soroban-grab-this-security-checklist-to-avoid-vulnerabilities/)
