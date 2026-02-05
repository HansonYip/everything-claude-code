# LayerZero Treasury Specification

> **Source:** [EVM Reference Implementation](https://github.com/LayerZero-Labs/LayerZero-v2/blob/main/packages/layerzero-v2/evm/messagelib/contracts/Treasury.sol)
> **Purpose:** Treasury fee management specification

---

## 1. Design Overview

Treasury is the protocol-level fee management component responsible for calculating and collecting protocol fees on cross-chain messages. The Message Library invokes Treasury during the quote and send operations.

**Key Characteristics:**

| Aspect              | Description                                                      |
| ------------------- | ---------------------------------------------------------------- |
| **Fee Collection**  | Protocol fee collection point for LayerZero messaging            |
| **Integration**     | Called by Message Library (SendLib) during quote/send operations |
| **Payment Options** | Supports dual-token payment: Native token or ZRO token           |

---

## 2. Interface

**ILayerZeroTreasury:**

| Function | Parameters                                    | Returns | Description                    |
| -------- | --------------------------------------------- | ------- | ------------------------------ |
| `getFee` | sender, dstEid, totalNativeFee, payInZroToken | fee     | Query protocol fee (read-only) |
| `payFee` | sender, dstEid, totalNativeFee, payInZroToken | fee     | Execute fee payment            |

---

## 3. Fee Calculation

### Native Token Payment

Protocol fee is calculated as a percentage of the total worker fees (DVN + Executor):

```
protocolFee = totalNativeFee × nativeBP / 10000
```

Where `nativeBP` is the fee rate in basis points (e.g., 100 = 1%).

### ZRO Token Payment

When paying with ZRO token:

| Aspect       | Behavior                                       |
| ------------ | ---------------------------------------------- |
| Fee Amount   | Fixed fee: `zroTokenFee`                       |
| Requirement  | `zroTokenEnabled` must be `true`               |
| Failure Mode | Transaction reverts if ZRO payment is disabled |

---

## 4. Integration with Message Library

**Quote Flow:**

```
┌──────────┐        ┌─────────────┐        ┌──────────┐        ┌──────────┐
│   OApp   │        │ EndpointV2  │        │ SendLib  │        │ Treasury │
└────┬─────┘        └──────┬──────┘        └────┬─────┘        └────┬─────┘
     │  quote()            │                    │                   │
     │────────────────────▶│  quote()           │                   │
     │                     │───────────────────▶│                   │
     │                     │                    │                   │
     │                     │                    │  quoteWorkers()   │
     │                     │                    │  → workersFee     │
     │                     │                    │                   │
     │                     │                    │  getFee(workersFee)
     │                     │                    │──────────────────▶│
     │                     │                    │◀──────────────────│
     │                     │                    │  protocolFee      │
     │                     │                    │                   │
     │                     │◀───────────────────│                   │
     │◀────────────────────│  MessagingFee      │                   │
     │  (nativeFee,zroFee) │  nativeFee = workersFee + protocolFee  │
```

**Send Flow:**

```
┌──────────┐        ┌─────────────┐        ┌──────────┐        ┌──────────┐
│   OApp   │        │ EndpointV2  │        │ SendLib  │        │ Treasury │
└────┬─────┘        └──────┬──────┘        └────┬─────┘        └────┬─────┘
     │  send() + fee       │                    │                   │
     │────────────────────▶│  send()            │                   │
     │                     │───────────────────▶│                   │
     │                     │                    │                   │
     │                     │                    │  payWorkers()     │
     │                     │                    │  → workersFee     │
     │                     │                    │                   │
     │                     │                    │  payFee(workersFee)
     │                     │                    │──────────────────▶│
     │                     │                    │◀──────────────────│
     │                     │                    │  (fee collected)  │
```

---

## 5. Security Considerations

**Design Principle**: Treasury must never block user messages. LayerZero is designed to be permissionless - no admin action should cause DoS for OApps.

There are two approaches to achieve this guarantee:

### 5.1 Approach 1: Immutability

Prevent DoS by making Treasury unchangeable:

| Component         | Requirement                                      | Purpose                                          |
| ----------------- | ------------------------------------------------ | ------------------------------------------------ |
| Treasury Address  | Immutable in Message Library (set at deployment) | Admin cannot swap to malicious Treasury          |
| Treasury Contract | Non-upgradeable                                  | Treasury logic cannot be changed post-deployment |
| Fee Recipient     | Hardcoded to Treasury contract address           | Prevents invalid recipient configuration         |

**Pros**: Simple, no additional runtime protection needed
**Cons**: Cannot update Treasury logic or address; requires redeployment for changes

### 5.2 Approach 2: Fail-Safe Mechanisms

Allow Treasury configurability while ensuring messages never block:

| Protection | Mechanism                                                       | Purpose                                |
| ---------- | --------------------------------------------------------------- | -------------------------------------- |
| Gas Limit  | Treasury calls have configurable gas limit                      | Prevent DoS via gas exhaustion         |
| Fee Cap    | `treasuryNativeFeeCap` limits maximum native fee                | Prevent excessive fee extraction       |
| Fail-Safe  | Treasury call failure results in zero fee (message still sends) | Ensure message delivery is not blocked |

**Fee Cap Logic:**

```
maxNativeFee = max(totalNativeFee, treasuryNativeFeeCap)
actualFee = min(treasuryQuotedFee, maxNativeFee)
```

**Why Fail-Safe Works:**

Even if admin sets Treasury to a malicious or invalid address:

- Malicious contract consuming excessive gas → Gas Limit prevents DoS
- Contract charging unreasonable fees → Fee Cap limits extraction
- Invalid address causing call failure → Fail-Safe ensures zero fee, message proceeds

**Pros**: Flexible, Treasury can be updated
**Cons**: Requires correct implementation of fail-safe mechanisms in Message Library

### 5.3 Choosing an Approach

| Consideration             | Approach 1 (Immutability)    | Approach 2 (Fail-Safe)         |
| ------------------------- | ---------------------------- | ------------------------------ |
| Operational flexibility   | Low                          | High                           |
| Implementation complexity | Low                          | Medium                         |
| Trust assumption          | Trust deployment-time config | Trust fail-safe implementation |

**Invariant (both approaches)**: No administrative action on Treasury can prevent a user from sending a valid cross-chain message.

---

## 6. Configuration

**State Variables:**

| Variable          | Type    | Description                                    |
| ----------------- | ------- | ---------------------------------------------- |
| `nativeBP`        | uint256 | Native fee rate in basis points (10000 = 100%) |
| `zroTokenFee`     | uint256 | Fixed ZRO token fee amount                     |
| `zroTokenEnabled` | bool    | Whether ZRO token payment is enabled           |

**Administrative Operations:**

| Operation            | Description                      |
| -------------------- | -------------------------------- |
| `setNativeFeeBP`     | Configure native fee rate        |
| `setZroTokenFee`     | Configure fixed ZRO token fee    |
| `setZroTokenEnabled` | Enable/disable ZRO token payment |
