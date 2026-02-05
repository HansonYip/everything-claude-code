# LayerZero Endpoint Specification

> **Source:** [EVM Reference Implementation](https://github.com/LayerZero-Labs/LayerZero-v2/tree/main/packages/layerzero-v2/evm/protocol) > **Purpose:** EndpointV2 and MessageLib interface specification

---

## Table of Contents

1. [Overview & Architecture](#1-overview--architecture)
2. [EndpointV2](#2-endpointv2)
    - [2.8 Public Interface](#28-public-interface)
3. [Message Library Interface](#3-message-library-interface)

---

## 1. Overview & Architecture

### 1.1 Protocol Architecture

The core protocol consists of **Endpoint** and pluggable **Message Libraries**. The Endpoint handles message routing and state management, while Message Libraries handle encoding and verification.

**Key Design Principle**: The Endpoint is library-agnostic. Different Message Libraries can implement different security models (DVN quorum, trusted validator, etc.) while maintaining the same Endpoint interface.

**Send Flow (Source Chain)**

```
┌──────────┐            ┌─────────────┐            ┌──────────┐
│   OApp   │            │ EndpointV2  │            │ SendLib  │
└────┬─────┘            └──────┬──────┘            └────┬─────┘
     │                         │                        │
     │  send(params)           │                        │
     │────────────────────────▶│                        │
     │                         │                        │
     │                         │  increment nonce       │
     │                         │  generate GUID         │
     │                         │                        │
     │                         │  send(Packet)          │
     │                         │───────────────────────▶│
     │                         │                        │
     │                         │◀───────────────────────│
     │                         │  (fee, encodedPacket)  │
     │                         │                        │
     │◀────────────────────────│                        │
     │  MessagingReceipt       │                        │
     │                         │                        │
     │                         │  emit PacketSent       │
     └─────────────────────────┴────────────────────────┘
```

**Receive Flow (Destination Chain)**

```
┌────────────┐      ┌──────────┐      ┌─────────────┐      ┌──────────┐
│ ReceiveLib │      │ Executor │      │ EndpointV2  │      │   OApp   │
└─────┬──────┘      └────┬─────┘      └──────┬──────┘      └────┬─────┘
      │                  │                   │                  │
      │  verify(origin, receiver, hash)      │                  │
      │─────────────────────────────────────▶│                  │
      │                  │                   │                  │
      │                  │                   │ store hash       │
      │                  │                   │                  │
      │                  │      ... later ...                   │
      │                  │                   │                  │
      │                  │ lzReceive(origin, │                  │
      │                  │   receiver, ...)  │                  │
      │                  │──────────────────▶│                  │
      │                  │                   │                  │
      │                  │                   │ clear hash       │
      │                  │                   │ update nonce     │
      │                  │                   │                  │
      │                  │                   │ lzReceive(...)   │
      │                  │                   │─────────────────▶│
      │                  │                   │                  │
      │                  │                   │ emit Delivered   │
      └──────────────────┴───────────────────┴──────────────────┘
```

### 1.2 Component Roles Summary

| Component           | Role             | Key Responsibility                                   |
| ------------------- | ---------------- | ---------------------------------------------------- |
| **EndpointV2**      | Protocol Router  | Message routing, nonce management, library selection |
| **Message Library** | Send/Receive Lib | Packet encoding, verification, worker coordination   |
| **OApp**            | Application      | Cross-chain application using LayerZero messaging    |
| **Executor**        | Message Executor | Execute verified messages on destination chain       |
| **PriceFeed**       | Price Oracle     | Gas/token price feeds for fee calculation            |

### 1.3 Terminology

| V1 Term               | V2 Term         | Description                               |
| --------------------- | --------------- | ----------------------------------------- |
| chainId               | eid             | Endpoint ID (unique per chain deployment) |
| adapterParams         | options         | Execution options (gas, value, etc.)      |
| userApplication       | oapp            | Omnichain Application                     |
| srcAddress/dstAddress | sender/receiver | Generic identifier (supports non-EVM)     |
| payload               | message         | Application data within packet            |

---

## 2. EndpointV2

### 2.1 Design Overview

EndpointV2 is the central coordinator of the LayerZero protocol. It serves as the single entry point for all cross-chain messaging operations, managing the complete lifecycle of messages from sending to delivery.

**Core Responsibilities:**

| Module                         | Responsibility                                                                                                                                                                        |
| ------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Message Library Management** | Registering, selecting, and configuring message libraries (SendLib/ReceiveLib) for each pathway. Supports protocol-wide defaults and per-OApp overrides with migration grace periods. |
| **Messaging Channel**          | Managing message ordering through nonces. Outbound nonces ensure unique GUIDs; inbound nonces enable ordered delivery while allowing out-of-order verification.                       |
| **Messaging Composer**         | Handling multi-step message composition, where one received message can trigger additional cross-contract calls.                                                                      |

**Design Principles:**

1. **Library Abstraction**: EndpointV2 does not implement verification or fee logic directly. Instead, it delegates to pluggable Message Libraries, enabling protocol upgrades without endpoint changes.

2. **Lazy Inbound Nonce**: Messages can be verified out-of-order, but execution requires all prior nonces to be either delivered or explicitly skipped. This preserves censorship resistance while enabling flexible verification.

3. **Delegate Pattern**: OApps can authorize a delegate address to manage their LayerZero configuration, enabling separation of operational and asset management keys.

4. **Pull and Push Delivery**: Messages can be delivered via executor (push) or by the OApp itself (pull via `clear`), giving OApps flexibility in handling their messages.

### 2.2 Functional Modules

#### 2.2.1 Message Library Management

**Purpose**: Allow the protocol to evolve by supporting multiple message library implementations, while giving OApps control over which libraries handle their messages.

**Key Behaviors:**

| Operation                        | Description                                                                                | Authorization                    |
| -------------------------------- | ------------------------------------------------------------------------------------------ | -------------------------------- |
| Register Library                 | Add a new library to the protocol's registry                                               | Owner only                       |
| Set Default Send/Receive Library | Configure protocol-wide default for an endpoint pair                                       | Owner only                       |
| Set OApp Send/Receive Library    | Override the default for a specific OApp                                                   | OApp or delegate                 |
| Library Migration                | Transition to a new library with a grace period where both old and new libraries are valid | Owner (default) or OApp (custom) |

**Library Resolution Logic:**

```
getSendLibrary(sender, dstEid):
    if customLibrary[sender][dstEid] is set:
        return customLibrary[sender][dstEid]
    else:
        return defaultLibrary[dstEid]  // must exist

isValidReceiveLibrary(receiver, srcEid, actualLib):
    expectedLib = getReceiveLibrary(receiver, srcEid)
    if actualLib == expectedLib:
        return true
    // Check grace period
    timeout = getTimeout(receiver, srcEid)
    if timeout.lib == actualLib AND timeout.expiry > currentBlock:
        return true
    return false
```

#### 2.2.2 Messaging Channel

**Purpose**: Ensure message ordering and uniqueness while enabling flexible verification strategies.

**Nonce Model:**

| Nonce Type         | Behavior                           | Purpose                                                            |
| ------------------ | ---------------------------------- | ------------------------------------------------------------------ |
| **Outbound Nonce** | Eagerly incremented before sending | Guarantees unique GUID for each message                            |
| **Inbound Nonce**  | Lazily tracked (checkpoint-based)  | Enables out-of-order verification while ensuring ordered execution |

**Inbound Flow:**

```
                    ┌─────────────────────────────────────────────────┐
                    │              INBOUND MESSAGE STATES             │
                    └─────────────────────────────────────────────────┘

    ┌──────────┐     verify()      ┌──────────┐     lzReceive()    ┌──────────┐
    │  Empty   │ ─────────────────▶│ Verified │ ──────────────────▶│ Executed │
    │  (none)  │                   │(payloadH)│                    │  (none)  │
    └──────────┘                   └──────────┘                    └──────────┘
         │                              │
         │ skip()                       │ nilify()
         ▼                              ▼
    ┌──────────┐                   ┌──────────┐
    │ Skipped  │                   │ Nilified │ ◀──── Can be re-verified
    │  (none)  │                   │(0xFF..FF)│
    └──────────┘                   └──────────┘
                                        │
                                        │ burn()
                                        ▼
                                   ┌──────────┐
                                   │  Burnt   │ ◀──── Permanent, cannot recover
                                   │  (none)  │
                                   └──────────┘
```

**Nonce Array & lazyInboundNonce:**

```
    Nonces:     1      2      3      4      5      6      7      8
              ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐
    State:    │Exec│ │Exec│ │Nilf│ │Exec│ │Verf│ │Verf│ │Verf│ │Empt│
              └────┘ └────┘ └────┘ └────┘ └────┘ └────┘ └────┘ └────┘
                                     ▲                    ▲
                                     │                    │
                            lazyInboundNonce=4    inboundNonce()=7
                            (checkpoint of last   (scans from lazy, finds
                             consecutive execution) 5,6,7 verified/executed)
```

**Operation Constraints by Nonce Position:**

| Operation  | Constraint                                  | Allowed Nonces (in example) |
| ---------- | ------------------------------------------- | --------------------------- |
| `skip()`   | nonce = inboundNonce() + 1                  | Only nonce 8                |
| `nilify()` | nonce > lazyInboundNonce OR payload exists  | 3 (payload) or 5,6,7,8 (>4) |
| `burn()`   | nonce ≤ lazyInboundNonce AND payload exists | Only nonce 3 (Nilified, ≤4) |

**Payload Hash States:**

| State    | Value                       | Meaning                                 |
| -------- | --------------------------- | --------------------------------------- |
| Empty    | Storage default (None/null) | Not verified or already executed        |
| Verified | `hash(guid + message)`      | Verified, pending execution             |
| Nilified | `0xFFFF...FFFF`             | Temporarily blocked, can be re-verified |

#### 2.2.3 Messaging Composer

**Purpose**: Enable complex multi-step workflows where receiving a message triggers additional contract interactions.

**Compose Flow:**

```
    ┌─────────┐                    ┌──────────┐
    │  OApp   │  1. sendCompose()  │ Endpoint │
    │Receiver │ ──────────────────▶│          │
    └─────────┘                    └────┬─────┘
                                        │ 2. Queue compose
                                        │    with hash
                                        │
                                   ... later ...
                                        │
                                        ▲
    ┌──────────┐  3. lzCompose()   ┌────┴─────┐  4. lzCompose()   ┌──────────┐
    │ Executor │ ────────────────▶ │ Endpoint │ ────────────────▶ │ Composer │
    └──────────┘                   └──────────┘                   └──────────┘
```

**Compose Queue States:**

| State    | Hash Value              | Meaning                   |
| -------- | ----------------------- | ------------------------- |
| Empty    | Storage default (None)  | No compose message queued |
| Queued   | `hash(message)`         | Pending execution         |
| Executed | Sentinel (e.g., `0x01`) | Already delivered         |

**Security**: Compose messages can only be sent by the original message receiver (via `sendCompose`), and can only be executed once. |

### 2.3 Core Operations

#### 2.3.1 Sending Messages

**Operation**: `send(params, refundAddress) → receipt`

**Process:**

1. Validate fee payment method
2. Increment outbound nonce for the path (sender → dstEid → receiver)
3. Generate globally unique identifier (GUID)
4. Construct packet and delegate to send library
5. Validate supplied fees against required fees
6. Transfer fees to library, refund excess
7. Emit `PacketSent` event

#### 2.3.2 Verifying Messages

**Operation**: `verify(origin, receiver, payloadHash)`

**Authorization**: Only callable by a valid receive library for the receiver/srcEid pair.

**Process:**

1. Validate receive library authorization
2. Check path initialization (first message requires OApp approval)
3. Store payload hash at the nonce slot
4. Emit `PacketVerified` event

#### 2.3.3 Executing Messages

**Operation**: `lzReceive(origin, receiver, guid, message, extraData)` [Push Mode]

**Process:**

1. Compute payload hash from provided data
2. Verify hash matches stored value
3. Update lazy inbound nonce (fills gaps if consecutive)
4. Delete payload hash (prevents reentrancy)
5. Call receiver's `lzReceive` callback
6. Emit `PacketDelivered` event

**Operation**: `clear(oapp, origin, guid, message)` [Pull Mode]

Same as `lzReceive` but does not call the receiver callback.

#### 2.3.4 Message Management

- **Skip** - Marks a nonce as processed without verification
- **Nilify** - Marks a verified message as temporarily non-executable
- **Burn** - Permanently deletes a verified message

### 2.4 Data Structures

```
MessagingParams {
    dstEid: u32          // Destination endpoint ID
    receiver: bytes32    // Receiver address (32 bytes for cross-VM compatibility)
    message: bytes       // Application payload
    options: bytes       // Execution options (gas, value, etc.)
    payInZro: bool       // Pay fees in ZRO token vs native
}

MessagingReceipt {
    guid: bytes32        // Globally unique message identifier
    nonce: u64           // Outbound nonce for this path
    fee: MessagingFee    // Actual fees charged
}

MessagingFee {
    nativeFee: u256      // Fee in native token
    zroFee: u256         // Fee in ZRO token
}

Origin {
    srcEid: u32          // Source endpoint ID
    sender: bytes32      // Sender address (32 bytes)
    nonce: u64           // Message nonce
}

OutBoundPacket {
    nonce: u64
    srcEid: u32
    sender: bytes32
    dstEid: u32
    receiver: bytes32
    guid: bytes32
    message: bytes
}
```

### 2.5 Events

| Event                      | Trigger                             | Key Fields                                   |
| -------------------------- | ----------------------------------- | -------------------------------------------- |
| `PacketSent`               | Message sent                        | encodedPacket, options, sendLibrary          |
| `PacketVerified`           | Message verified by receive library | origin, receiver, payloadHash                |
| `PacketDelivered`          | Message executed or cleared         | origin, receiver                             |
| `ComposeSent`              | Compose message queued              | from, to, guid, index, message               |
| `ComposeDelivered`         | Compose message executed            | from, to, guid, index                        |
| `LzReceiveAlert`           | Execution failed                    | receiver, executor, origin, guid, gas, value |
| `LzComposeAlert`           | Compose execution failed            | from, to, executor, guid, index, gas, value  |
| `InboundNonceSkipped`      | Nonce explicitly skipped            | srcEid, sender, receiver, nonce              |
| `PacketNilified`           | Message marked non-executable       | srcEid, sender, receiver, nonce, payloadHash |
| `PacketBurnt`              | Message permanently deleted         | srcEid, sender, receiver, nonce, payloadHash |
| `LibraryRegistered`        | New library registered              | newLib                                       |
| `DefaultSendLibrarySet`    | Default send library changed        | eid, newLib                                  |
| `DefaultReceiveLibrarySet` | Default receive library changed     | eid, newLib                                  |
| `SendLibrarySet`           | OApp send library override          | sender, eid, newLib                          |
| `ReceiveLibrarySet`        | OApp receive library override       | receiver, eid, newLib                        |

### 2.6 Error Conditions

| Condition                       | Description                                          |
| ------------------------------- | ---------------------------------------------------- |
| ZRO Unavailable                 | Attempting to pay with ZRO token but address not set |
| Invalid Receive Library         | Caller is not a valid receive library                |
| Invalid Nonce                   | Nonce out of expected sequence                       |
| Unauthorized                    | Caller is not OApp or delegate                       |
| Default Send Lib Unavailable    | No default send library for destination              |
| Default Receive Lib Unavailable | No default receive library for source                |
| Path Not Initializable          | First message on path but OApp rejects               |
| Path Not Verifiable             | Attempting to verify invalid nonce                   |
| Invalid Payload Hash            | Zero payload hash                                    |
| Payload Hash Not Found          | Provided hash doesn't match stored                   |
| Compose Not Found               | Compose message hash mismatch                        |
| Compose Exists                  | Compose message already queued                       |
| Insufficient Fee                | Supplied fees less than required                     |

### 2.7 Security Invariants

1. **Nonce Integrity** - Outbound nonce is strictly increasing per path (no gaps, no reuse)
2. **Reentrancy Protection** - Payload hash deleted before calling receiver's `lzReceive`
3. **Library Authorization** - Only valid receive library can call `verify`
4. **Path Initialization** - First message on any path requires OApp's explicit approval
5. **Fee Validation** - Fees must be fully supplied before sending

### 2.8 Public Interface

This section provides a complete list of all public functions exposed by EndpointV2. The function names and signatures below are based on the EVM reference implementation. Other VM implementations (Stellar, Move, etc.) MUST provide equivalent functionality with naming conventions appropriate to their platform (e.g., `getSendLibrary` → `get_send_library` in Rust/Move).

#### Core Messaging

| Function         | Signature                                                                        | Auth            | Description                 |
| ---------------- | -------------------------------------------------------------------------------- | --------------- | --------------------------- |
| `quote`          | `quote(params, sender) → fee`                                                    | -               | Quote message fee           |
| `send`           | `send(params, refundAddress) → receipt`                                          | -               | Send cross-chain message    |
| `verify`         | `verify(origin, receiver, payloadHash)`                                          | ReceiveLib only | Verify message              |
| `verifiable`     | `verifiable(origin, receiver) → bool`                                            | -               | Check if verifiable         |
| `initializable`  | `initializable(origin, receiver) → bool`                                         | -               | Check if path initializable |
| `lzReceive`      | `lzReceive(origin, receiver, guid, message, extraData)`                          | -               | Execute message (push)      |
| `lzReceiveAlert` | `lzReceiveAlert(origin, receiver, guid, gas, value, message, extraData, reason)` | -               | Record failed execution     |
| `clear`          | `clear(oapp, origin, guid, message)`                                             | OApp/delegate   | Execute message (pull)      |

#### Messaging Channel

| Function       | Signature                                          | Auth          | Description         |
| -------------- | -------------------------------------------------- | ------------- | ------------------- |
| `inboundNonce` | `inboundNonce(receiver, srcEid, sender) → u64`     | -             | Get inbound nonce   |
| `nextGuid`     | `nextGuid(sender, dstEid, receiver) → bytes32`     | -             | Get next GUID       |
| `skip`         | `skip(oapp, srcEid, sender, nonce)`                | OApp/delegate | Skip nonce          |
| `nilify`       | `nilify(oapp, srcEid, sender, nonce, payloadHash)` | OApp/delegate | Mark non-executable |
| `burn`         | `burn(oapp, srcEid, sender, nonce, payloadHash)`   | OApp/delegate | Permanently delete  |

#### Messaging Composer

| Function         | Signature                                                                       | Auth          | Description           |
| ---------------- | ------------------------------------------------------------------------------- | ------------- | --------------------- |
| `sendCompose`    | `sendCompose(to, guid, index, message)`                                         | Receiver only | Queue compose message |
| `lzCompose`      | `lzCompose(from, to, guid, index, message, extraData)`                          | -             | Execute compose       |
| `lzComposeAlert` | `lzComposeAlert(from, to, guid, index, gas, value, message, extraData, reason)` | -             | Record failed compose |

#### Message Library Management

| Function                          | Signature                                               | Auth          | Description             |
| --------------------------------- | ------------------------------------------------------- | ------------- | ----------------------- |
| `registerLibrary`                 | `registerLibrary(lib)`                                  | Owner         | Register new library    |
| `getRegisteredLibraries`          | `getRegisteredLibraries() → address[]`                  | -             | List all libraries      |
| `setDefaultSendLibrary`           | `setDefaultSendLibrary(eid, newLib)`                    | Owner         | Set default send lib    |
| `setDefaultReceiveLibrary`        | `setDefaultReceiveLibrary(eid, newLib, gracePeriod)`    | Owner         | Set default receive lib |
| `setDefaultReceiveLibraryTimeout` | `setDefaultReceiveLibraryTimeout(eid, lib, expiry)`     | Owner         | Set timeout             |
| `getSendLibrary`                  | `getSendLibrary(sender, dstEid) → address`              | -             | Get send library        |
| `getReceiveLibrary`               | `getReceiveLibrary(receiver, srcEid) → (address, bool)` | -             | Get receive library     |
| `isDefaultSendLibrary`            | `isDefaultSendLibrary(sender, dstEid) → bool`           | -             | Check if default        |
| `isValidReceiveLibrary`           | `isValidReceiveLibrary(receiver, srcEid, lib) → bool`   | -             | Validate receive lib    |
| `isSupportedEid`                  | `isSupportedEid(eid) → bool`                            | -             | Check EID support       |
| `setSendLibrary`                  | `setSendLibrary(oapp, eid, newLib)`                     | OApp/delegate | Set OApp send lib       |
| `setReceiveLibrary`               | `setReceiveLibrary(oapp, eid, newLib, gracePeriod)`     | OApp/delegate | Set OApp receive lib    |
| `setReceiveLibraryTimeout`        | `setReceiveLibraryTimeout(oapp, eid, lib, expiry)`      | OApp/delegate | Set OApp timeout        |
| `setConfig`                       | `setConfig(oapp, lib, params[])`                        | OApp/delegate | Set library config      |
| `getConfig`                       | `getConfig(oapp, lib, eid, configType) → bytes`         | -             | Get library config      |

#### Admin & Utility

| Function       | Signature                         | Auth  | Description          |
| -------------- | --------------------------------- | ----- | -------------------- |
| `setDelegate`  | `setDelegate(delegate)`           | OApp  | Set delegate address |
| `setLzToken`   | `setLzToken(lzToken)`             | Owner | Set ZRO token        |
| `nativeToken`  | `nativeToken() → address`         | -     | Get native token     |
| `recoverToken` | `recoverToken(token, to, amount)` | Owner | Recover stuck tokens |

#### State Getters

| Function                       | Signature                                                       | Description           |
| ------------------------------ | --------------------------------------------------------------- | --------------------- |
| `eid`                          | `eid() → u32`                                                   | Endpoint ID           |
| `lzToken`                      | `lzToken() → address`                                           | ZRO token address     |
| `delegates`                    | `delegates(oapp) → address`                                     | OApp's delegate       |
| `isRegisteredLibrary`          | `isRegisteredLibrary(lib) → bool`                               | Check if registered   |
| `defaultSendLibrary`           | `defaultSendLibrary(dstEid) → address`                          | Default send lib      |
| `defaultReceiveLibrary`        | `defaultReceiveLibrary(srcEid) → address`                       | Default receive lib   |
| `defaultReceiveLibraryTimeout` | `defaultReceiveLibraryTimeout(srcEid) → Timeout`                | Default timeout       |
| `receiveLibraryTimeout`        | `receiveLibraryTimeout(receiver, srcEid) → Timeout`             | OApp timeout          |
| `lazyInboundNonce`             | `lazyInboundNonce(receiver, srcEid, sender) → u64`              | Lazy nonce checkpoint |
| `outboundNonce`                | `outboundNonce(sender, dstEid, receiver) → u64`                 | Outbound nonce        |
| `inboundPayloadHash`           | `inboundPayloadHash(receiver, srcEid, sender, nonce) → bytes32` | Stored payload hash   |
| `composeQueue`                 | `composeQueue(from, to, guid, index) → bytes32`                 | Compose message hash  |

#### Constants

| Constant             | Value                        | Description                   |
| -------------------- | ---------------------------- | ----------------------------- |
| `EMPTY_PAYLOAD_HASH` | `bytes32(0)`                 | Empty hash constant           |
| `NIL_PAYLOAD_HASH`   | `bytes32(type(uint256).max)` | Nil hash constant (0xFF...FF) |

> **Note**: Constants `EMPTY_PAYLOAD_HASH` and `NIL_PAYLOAD_HASH` may be exposed as public constants or getter functions depending on the VM implementation.

---

## 3. Message Library Interface

### 3.1 Overview

Message Libraries are pluggable modules that handle the actual cross-chain message encoding and verification.

### 3.2 SendLib Interface

**Operation**: `send(packet, options, payInZro) → (fee, encodedPacket)`

| Parameter  | Type           | Description                                                                   |
| ---------- | -------------- | ----------------------------------------------------------------------------- |
| `packet`   | OutboundPacket | Packet from Endpoint (nonce, srcEid, sender, dstEid, receiver, guid, message) |
| `options`  | bytes          | Worker options (executor gas, DVN options, etc.)                              |
| `payInZro` | bool           | Whether to pay fees in ZRO token                                              |

**Operation**: `quote(packet, options, payInZro) → fee`

Same parameters as `send`, returns fee estimate without executing.

### 3.3 ReceiveLib Interface

Each receive library implementation defines its own verification mechanism. The only requirement is that verified messages must be committed to the Endpoint by calling `endpoint.verify()`.

### 3.4 Config Interface

**Operation**: `setConfig(oapp, eid, configType, config)`

Called by Endpoint to set OApp-specific configuration.

### 3.5 Packet Encoding (PacketV1Codec)

**Packet Structure:**

```
┌──────────┬────────┬─────────────────────────────────────────────────────┐
│  Offset  │  Size  │  Field                                              │
├──────────┼────────┼─────────────────────────────────────────────────────┤
│    0     │    1   │  version       (always 1 for V1 packets)            │
│    1     │    8   │  nonce         (uint64, big-endian)                 │
│    9     │    4   │  srcEid        (uint32, big-endian)                 │
│   13     │   32   │  sender        (bytes32, EVM addresses left-padded) │
│   45     │    4   │  dstEid        (uint32, big-endian)                 │
│   49     │   32   │  receiver      (bytes32)                            │
├──────────┼────────┼─────────────────────────────────────────────────────┤
│   81     │   32   │  guid          (bytes32, computed from header)      │
│  113     │  var   │  message       (application payload, any length)    │
└──────────┴────────┴─────────────────────────────────────────────────────┘

Total Header Size: 81 bytes (offset 0-80)
```

**GUID Generation:**

```
GUID = keccak256(nonce || srcEid || sender || dstEid || receiver)
```

**Payload Hash:**

```
PayloadHash = keccak256(guid || message)
```
