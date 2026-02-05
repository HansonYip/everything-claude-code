# OApp Specification

> **Version:** 1.0.0
> **Purpose:** Application-layer interface specification for LayerZero cross-chain messaging

---

## Table of Contents

1. [Overview](#1-overview)
2. [Receiver Interface](#2-receiver-interface)
    - [2.1 lzReceive](#21-lzreceive)
    - [2.2 allowInitializePath](#22-allowinitializepath)
    - [2.3 nextNonce](#23-nextnonce)
3. [Composer Interface](#3-composer-interface)
    - [3.1 lzCompose](#31-lzcompose)
4. [Implementation Guidelines](#4-implementation-guidelines)
    - [4.1 Peer Configuration](#41-peer-configuration)
    - [4.2 Sending Messages](#42-sending-messages)
    - [4.3 Error Handling](#43-error-handling)
    - [4.4 Security Checklist](#44-security-checklist)

---

## 1. Overview

OApp (Omnichain Application) is the application-layer contract that uses LayerZero for cross-chain messaging. OApps must implement specific interfaces to receive messages and compose calls from the Endpoint.

**Interface Summary:**

| Interface              | Purpose                                      | Required |
| ---------------------- | -------------------------------------------- | -------- |
| **Receiver Interface** | Receive cross-chain messages via `lzReceive` | Yes      |
| **Composer Interface** | Receive compose calls via `lzCompose`        | Optional |

**Interaction Flow:**

**Push Mode (Default):**

In push mode, the Executor calls the Endpoint, which then calls the OApp's `lzReceive`. The Endpoint verifies and clears the message hash before calling the OApp.

```
┌──────────┐                    ┌─────────────┐                    ┌──────────┐
│ Executor │                    │ EndpointV2  │                    │   OApp   │
└────┬─────┘                    └──────┬──────┘                    └────┬─────┘
     │                                 │                                │
     │  lzReceive(origin, receiver,    │                                │
     │    guid, message, extraData)    │                                │
     │────────────────────────────────▶│                                │
     │                                 │                                │
     │                                 │  lzReceive(origin, guid,       │
     │                                 │    message, executor, extraData)
     │                                 │───────────────────────────────▶│
     │                                 │                                │
     │                                 │                    ┌───────────┤
     │                                 │                    │ Process   │
     │                                 │                    │ message   │
     │                                 │                    └───────────┤
     │                                 │                                │
     │                                 │◀───────────────────────────────│
     │◀────────────────────────────────│                                │
```

**Pull Mode:**

In pull mode, the Executor calls the OApp directly. The OApp calls `Endpoint.clear()` to verify and mark the message as delivered, then processes its business logic.

```
┌──────────┐                    ┌──────────┐                    ┌─────────────┐
│ Executor │                    │   OApp   │                    │ EndpointV2  │
└────┬─────┘                    └────┬─────┘                    └──────┬──────┘
     │                               │                                 │
     │  lzReceive(origin, guid,      │                                 │
     │    message, executor, extra)  │                                 │
     │──────────────────────────────▶│                                 │
     │                               │                                 │
     │                               │  clear(origin, guid, message)   │
     │                               │────────────────────────────────▶│
     │                               │                                 │
     │                               │                     ┌───────────┤
     │                               │                     │ Verify &  │
     │                               │                     │ Clear hash│
     │                               │                     └───────────┤
     │                               │                                 │
     │                               │◀────────────────────────────────│
     │                               │                                 │
     │                   ┌───────────┤                                 │
     │                   │ Process   │                                 │
     │                   │ business  │                                 │
     │                   │ logic     │                                 │
     │                   └───────────┤                                 │
     │                               │                                 │
     │◀──────────────────────────────│                                 │
```

---

## 2. Receiver Interface

The primary interface for receiving cross-chain messages. All OApps must implement this interface.

### 2.1 lzReceive

**Operation:** `lzReceive(origin, guid, message, executor, extraData)`

| Parameter   | Type    | Description                               |
| ----------- | ------- | ----------------------------------------- |
| `origin`    | Origin  | Source chain info (srcEid, sender, nonce) |
| `guid`      | bytes32 | Globally unique message identifier        |
| `message`   | bytes   | Application payload                       |
| `executor`  | bytes32 | Address that triggered delivery           |
| `extraData` | bytes   | Additional data from executor             |

**Behavior:**

1. Called by Endpoint after successful payload hash verification
2. Must process the message or revert
3. Reversion causes `LzReceiveAlert` event (message can be retried)

**Security Requirements:**

| Requirement           | Description                                                     |
| --------------------- | --------------------------------------------------------------- |
| Caller Validation     | Push mode: Only accept calls from the Endpoint contract         |
| Origin Validation     | Verify `origin.sender` is a trusted peer on the source chain    |
| Reentrancy Protection | Endpoint clears hash before calling, but OApp should add guards |

### 2.2 allowInitializePath

**Operation:** `allowInitializePath(origin) → bool`

**Purpose:** Controls whether the OApp accepts messages from a new (srcEid, sender) path.

**Behavior:**

- Called by Endpoint when first message arrives on a new path
- Return `true` to accept messages from this sender
- Return `false` to reject (message cannot be verified)

**Implementation Patterns:**

| Pattern        | Logic                                   | Use Case               |
| -------------- | --------------------------------------- | ---------------------- |
| Permissionless | Always return `true`                    | Open protocols         |
| Peer-gated     | `peers[origin.srcEid] == origin.sender` | Controlled deployments |

### 2.3 nextNonce

**Operation:** `nextNonce(srcEid, sender) → u64`

**Purpose:** Returns the next expected nonce for a given path.

**Behavior:**

- Called by Endpoint to determine message ordering requirements
- Return `0` to accept any nonce (unordered)
- Return specific nonce for ordered execution

**Implementation Patterns:**

| Pattern   | Logic                                   | Use Case                 |
| --------- | --------------------------------------- | ------------------------ |
| Unordered | Always return `0`                       | Independent messages     |
| Ordered   | `lastReceivedNonce[srcEid][sender] + 1` | State-dependent messages |

---

## 3. Composer Interface

Optional interface for receiving compose calls (multi-step message processing).

### 3.1 lzCompose

**Operation:** `lzCompose(from, guid, message, executor, extraData)`

| Parameter   | Type    | Description                        |
| ----------- | ------- | ---------------------------------- |
| `from`      | bytes32 | OApp that sent the compose message |
| `guid`      | bytes32 | Original message GUID              |
| `message`   | bytes   | Compose message payload            |
| `executor`  | bytes32 | Address that triggered compose     |
| `extraData` | bytes   | Additional data from executor      |

**Security Requirements:**

| Requirement       | Description                                              |
| ----------------- | -------------------------------------------------------- |
| Caller Validation | Push mode: Only accept calls from the Endpoint contract  |
| From Validation   | Verify `from` is a trusted OApp that can trigger compose |
| Index Tracking    | Each (guid, index) pair can only be composed once        |

---

## 4. Implementation Guidelines

### 4.1 Peer Configuration

OApps should maintain a peer registry to validate trusted senders:

```
Storage: peers[remoteEid] → bytes32  // Trusted sender address per chain

setPeer(remoteEid, peer):
    assert caller is owner
    peers[remoteEid] = peer

// In lzReceive:
assert caller == endpoint
assert peers[origin.srcEid] == origin.sender
```

### 4.2 Sending Messages

OApps send messages through the Endpoint:

```
send(dstEid, message, fee, options):
    peer = peers[dstEid]
    assert peer != 0

    params = MessagingParams {
        dstEid: dstEid,
        receiver: peer,
        message: message,
        options: options,
        payInZro: false
    }

    refund_address = sender
    receipt = endpoint.send(params, refund_address, fee.nativeFee)
    return receipt
```

### 4.3 Error Handling

**Execution Failures:**

- If `lzReceive` reverts, Endpoint emits `LzReceiveAlert`
- Message remains verified (not cleared)
- Can be retried by anyone calling `lzReceive` again with same parameters

**Recovery Options:**

| Scenario          | Recovery Method                                |
| ----------------- | ---------------------------------------------- |
| Bug in OApp logic | Fix OApp, retry execution                      |
| Insufficient gas  | Retry with higher gas limit                    |
| Malicious message | OApp owner can `nilify` or `burn` via Endpoint |
| Blocked sender    | OApp owner can `skip` future nonces            |

### 4.4 Security Checklist

| Check                     | Description                                                 |
| ------------------------- | ----------------------------------------------------------- |
| Verify Endpoint caller    | Only accept calls where caller == endpoint                  |
| Verify trusted peer       | Check origin.sender against configured peers                |
| Prevent reentrancy        | Use reentrancy guard or checks-effects-interactions pattern |
| Validate message format   | Decode and validate message structure before processing     |
| Handle execution failures | Implement recovery mechanism or admin intervention          |
| Set reasonable gas limits | Configure adequate gas in options for destination call      |
