---
name: stellar-coding-style
description: Enforces Soroban Stellar smart contract coding style and Rust best practices. Use when writing or reviewing Soroban Rust code for style compliance.
---

# Soroban Stellar Coding Style Guide

Apply these coding standards when writing or reviewing Soroban smart contracts:

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
// BAD - can cause DoS via storage bloat
#[instance(Vec<Address>)]
AllUsers,

// GOOD - use Persistent with pagination
#[persistent(Address)]
User { index: u32 },

#[instance(u32)]
UserCount,
```

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

## 6. Events

### Define events with appropriate topics

```rust
#[contractevent]
#[derive(Clone, Debug, Eq, PartialEq)]
pub struct Transfer {
    #[topic]  // Index for filtering
    pub from: Address,
    #[topic]  // Index for filtering
    pub to: Address,
    pub amount: i128,  // Not indexed, in payload only
}

// Publish events
Transfer { from, to, amount }.publish(env);
```

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

### Always use checked arithmetic for user inputs

```rust
// BAD - can overflow
let total = amount_a + amount_b;
let result = value * multiplier;

// GOOD - checked operations
let total = amount_a.checked_add(amount_b)
    .ok_or_else(|| MyError::Overflow)?;
let result = value.checked_mul(multiplier)
    .ok_or_else(|| MyError::Overflow)?;

// Also OK for internal constants
let constant_sum = FIXED_A + FIXED_B;  // Constants can't overflow
```

### Multiply before divide for precision

```rust
// BAD - loses precision
let result = amount / divisor * multiplier;

// GOOD - preserves precision
let result = amount * multiplier / divisor;
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

### Document public functions with Args/Returns/Panics

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
///
/// # Panics
/// * `InsufficientBalance` - If sender has insufficient funds
/// * `InvalidAmount` - If amount is zero or negative
pub fn transfer(env: &Env, from: &Address, to: &Address, amount: i128) -> i128 {
    // ...
}
```

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

### Keep functions small and focused

- Functions should do one thing well
- If a function is > 50 lines, consider splitting it
- Extract helper functions for reusable logic

## Output Format

When reviewing code, report style issues as:

1. **Category**: Organization/Naming/Error/Storage/Auth/etc.
2. **File**: path/to/file.rs:line
3. **Issue**: Description of the style violation
4. **Current**: The problematic code
5. **Suggested**: The corrected code
