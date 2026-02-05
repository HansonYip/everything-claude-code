# LayerZero ULN302 Specification

> **Source:** [EVM Reference Implementation](https://github.com/LayerZero-Labs/LayerZero-v2/tree/main/packages/layerzero-v2/evm/messagelib/contracts/uln) > **Purpose:** ULN302 Message Library, DVN, Executor, and Options encoding specification

---

## Table of Contents

1. [ULN302 (Message Library Implementation)](#1-uln302-message-library-implementation)
    - [1.8 Public Interface](#18-public-interface)
2. [Options Encoding](#2-options-encoding)

---

## 1. ULN302 (Message Library Implementation)

### 1.1 Design Overview

ULN302 (Ultra Light Node 302) is the default message library implementing a decentralized verification network (DVN) based security model.

| Component         | Role                           | Key Responsibility                                     |
| ----------------- | ------------------------------ | ------------------------------------------------------ |
| **SendUln302**    | Send Message Library           | Packet encoding, fee collection, worker job assignment |
| **ReceiveUln302** | Receive Message Library        | DVN verification aggregation, quorum checking          |
| **DVN**           | Decentralized Verifier Network | Cross-chain message verification                       |
| **Executor**      | Execution Fee Quoter           | Quote execution fee for destination chain delivery     |
| **Treasury**      | Protocol Fee Collector         | Protocol fee calculation and collection                |

**Design Principles:**

1. **Flexible Security**: OApps can configure their own DVN sets and thresholds, or use protocol defaults
2. **Required + Optional DVNs**: Security model supports both "must-have" DVNs (all must verify) and "nice-to-have" DVNs (threshold-based)
3. **Confirmation Delay**: DVNs must wait for configurable block confirmations before verifying

### 1.2 ULN Configuration

#### Resolved Configuration

```
UlnConfig {
    confirmations: u64       // Required block confirmations before verification
    requiredDVNs: address[]  // Addresses of required DVNs (sorted, no duplicates)
    optionalDVNs: address[]  // Addresses of optional DVNs (sorted, no duplicates)
    optionalDVNThreshold: u8 // How many optional DVNs must verify [0, optionalDVNCount]
}
```

**DVN List Constraints:**

- Each list must be sorted in ascending order
- No duplicate addresses within a list
- Required and optional lists should not overlap
- Maximum combined DVN count: 127

#### Configuration Resolution Design

Each configuration field follows a **two-action model**:

| State               | Description                                                              |
| ------------------- | ------------------------------------------------------------------------ |
| **Unset (Default)** | Initial state - OApp has not configured this field, use protocol default |
| **Custom Value**    | OApp explicitly set a value (including explicitly setting to none/empty) |

**Resolution Logic:**

```
getUlnConfig(oapp, remoteEid):
    default = getDefaultConfig(remoteEid)
    custom = getOAppConfig(oapp, remoteEid)

    result.confirmations = isSet(custom.confirmations)
        ? custom.confirmations
        : default.confirmations

    result.requiredDVNs = isSet(custom.requiredDVNs)
        ? custom.requiredDVNs
        : default.requiredDVNs

    // optionalDVNs and optionalDVNThreshold must be selected together
    if isSet(custom.optionalDVNs):
        result.optionalDVNs = custom.optionalDVNs
        result.optionalDVNThreshold = custom.optionalDVNThreshold
    else:
        result.optionalDVNs = default.optionalDVNs
        result.optionalDVNThreshold = default.optionalDVNThreshold

    // Final validation: at least one verification source required
    assert(len(result.requiredDVNs) > 0 OR
           (result.optionalDVNThreshold > 0 AND len(result.optionalDVNs) >= result.optionalDVNThreshold))

    return result
```

### 1.3 Send Flow (SendUln302)

**Operation**: `send(packet, options, payInZro) → (fee, encodedPacket)`

**Process:**

1. Parse options into executor and DVN options
2. Encode packet (header + payload)
3. Get UlnConfig for sender/dstEid
4. Assign DVN jobs (for each required and optional DVN)
5. Assign Executor job
6. Return fee and encoded packet

**Events:**

- `DVNFeePaid(requiredDVNs[], optionalDVNs[], fees[])`
- `ExecutorFeePaid(executor, fee)`

### 1.4 Receive Flow (ReceiveUln302)

**Phase 1: DVN Verification**

Each DVN independently monitors the source chain and submits verification when ready.

**Storage Model:**

```
VerificationRecord {
    dvn: address              // Which DVN submitted this verification
    confirmations: u64        // Block confirmations at submission time
}

Storage: verifications[hash(packetHeader)][payloadHash][dvn] → VerificationRecord
```

**Phase 2: Commit to Endpoint**

After quorum is reached, anyone can commit the verification to Endpoint.

**Behavior:**

1. **Validate header**: length == 81, version == 1, dstEid == localEid
2. **Resolve config**: Get UlnConfig for (receiver, srcEid)
3. **Check quorum**: Verify DVN quorum met with sufficient confirmations
4. **Commit to endpoint**: Call `endpoint.verify(origin, receiver, payloadHash)`

**Quorum Logic:**

```
checkQuorum(config, headerHash, payloadHash):
    // ALL required DVNs must have verified
    for dvn in config.requiredDVNs:
        v = hashLookup[headerHash][payloadHash][dvn]
        if not v.submitted OR v.confirmations < config.confirmations:
            return false

    // If no optional DVNs configured, required is enough
    if config.optionalDVNCount == 0:
        return true

    // Count optional DVNs that have verified
    count = 0
    for dvn in config.optionalDVNs:
        v = hashLookup[headerHash][payloadHash][dvn]
        if v.submitted AND v.confirmations >= config.confirmations:
            count++
            if count >= config.optionalDVNThreshold:
                return true

    return false
```

### 1.5 Security Invariants

1. **DVN Quorum Requirement** - ALL required DVNs must verify with sufficient confirmations
2. **Block Confirmation Protection** - Each DVN records confirmations at verification time
3. **DVN List Validation** - Addresses must be sorted, no duplicates, max 127 each
4. **Packet V1 Validation** - Header must be exactly 81 bytes, version == 1

### 1.6 DVN (Decentralized Verifier Network)

**Core Responsibilities:**

- Monitor source chain for LayerZero messages
- Wait for required block confirmations
- Submit verification to destination chain's ReceiveLib
- Charge fees for verification services

**Interface:**

| Function    | Parameters                               | Purpose                          |
| ----------- | ---------------------------------------- | -------------------------------- |
| `assignJob` | params, options                          | Assigns verification job         |
| `getFee`    | dstEid, confirmations, sender, options   | Returns fee for verification     |
| `verify`    | packetHeader, payloadHash, confirmations | Submits verification attestation |

### 1.7 Executor

**Core Responsibilities:**

- Receives job assignments during message sending
- Monitors for verified messages
- Executes delivery with configured gas limits
- Handles native token airdrops

**Interface:**

| Function    | Parameters                            | Purpose                   |
| ----------- | ------------------------------------- | ------------------------- |
| `assignJob` | params, options                       | Assigns execution job     |
| `getFee`    | dstEid, sender, calldataSize, options | Returns fee for execution |

**Executor calls Endpoint:**

- `lzReceive(origin, receiver, guid, message, extraData)` - Delivers message
- `lzCompose(from, to, guid, index, message, extraData)` - Executes compose call

### 1.8 Public Interface

This section provides a complete list of all public functions exposed by SendUln302 and ReceiveUln302. The function names and signatures below are based on the EVM reference implementation. Other VM implementations (Stellar, Move, etc.) MUST provide equivalent functionality with naming conventions appropriate to their platform (e.g., `getUlnConfig` → `get_uln_config` in Rust/Move).

#### SendUln302 Functions

| Function               | Signature                                                    | Auth          | Description                  |
| ---------------------- | ------------------------------------------------------------ | ------------- | ---------------------------- |
| `send`                 | `send(packet, options, payInLzToken) → (fee, encodedPacket)` | Endpoint only | Send message and pay workers |
| `quote`                | `quote(packet, options, payInLzToken) → fee`                 | -             | Quote total fee              |
| `setConfig`            | `setConfig(oapp, params[])`                                  | Endpoint only | Set OApp ULN config          |
| `getConfig`            | `getConfig(eid, oapp, configType) → bytes`                   | -             | Get config bytes             |
| `setDefaultUlnConfigs` | `setDefaultUlnConfigs(params[])`                             | Owner         | Set default ULN configs      |
| `getUlnConfig`         | `getUlnConfig(oapp, remoteEid) → UlnConfig`                  | -             | Get resolved ULN config      |
| `getAppUlnConfig`      | `getAppUlnConfig(oapp, remoteEid) → UlnConfig`               | -             | Get OApp-specific config     |
| `setTreasury`          | `setTreasury(treasury)`                                      | Owner         | Set treasury address         |
| `withdrawFee`          | `withdrawFee(to, amount)`                                    | -             | Withdraw native fees         |
| `withdrawLzTokenFee`   | `withdrawLzTokenFee(lzToken, to, amount)`                    | -             | Withdraw LZ token fees       |
| `version`              | `version() → (major, minor, endpointVersion)`                | -             | Get version info             |
| `isSupportedEid`       | `isSupportedEid(eid) → bool`                                 | -             | Check EID support            |
| `messageLibType`       | `messageLibType() → MessageLibType`                          | -             | Get library type (Send)      |
| `supportsInterface`    | `supportsInterface(interfaceId) → bool`                      | -             | ERC165 support               |

#### ReceiveUln302 Functions

| Function               | Signature                                            | Auth          | Description                         |
| ---------------------- | ---------------------------------------------------- | ------------- | ----------------------------------- |
| `commitVerification`   | `commitVerification(packetHeader, payloadHash)`      | -             | Commit verified message to Endpoint |
| `verify`               | `verify(packetHeader, payloadHash, confirmations)`   | DVN only      | Submit DVN verification             |
| `verifiable`           | `verifiable(config, headerHash, payloadHash) → bool` | -             | Check if quorum reached             |
| `assertHeader`         | `assertHeader(packetHeader, localEid)`               | -             | Validate packet header              |
| `setConfig`            | `setConfig(oapp, params[])`                          | Endpoint only | Set OApp ULN config                 |
| `getConfig`            | `getConfig(eid, oapp, configType) → bytes`           | -             | Get config bytes                    |
| `setDefaultUlnConfigs` | `setDefaultUlnConfigs(params[])`                     | Owner         | Set default ULN configs             |
| `getUlnConfig`         | `getUlnConfig(oapp, remoteEid) → UlnConfig`          | -             | Get resolved ULN config             |
| `getAppUlnConfig`      | `getAppUlnConfig(oapp, remoteEid) → UlnConfig`       | -             | Get OApp-specific config            |
| `version`              | `version() → (major, minor, endpointVersion)`        | -             | Get version info                    |
| `isSupportedEid`       | `isSupportedEid(eid) → bool`                         | -             | Check EID support                   |
| `supportsInterface`    | `supportsInterface(interfaceId) → bool`              | -             | ERC165 support                      |

#### State Variables

| Variable     | Type      | Description                |
| ------------ | --------- | -------------------------- |
| `endpoint`   | `address` | LayerZero Endpoint address |
| `localEid`   | `uint32`  | Local endpoint ID          |
| `treasury`   | `address` | Treasury for protocol fees |
| `hashLookup` | `mapping` | DVN verification records   |

---

## 2. Options Encoding

### 2.1 Design Overview

Options are a flexible encoding mechanism for specifying execution parameters.

**Key Design Decisions:**

1. **Version Evolution**: Type 3 (modern) vs Type 1/2 (legacy V1 compatibility)
2. **Worker Multiplexing**: Options can include instructions for multiple workers
3. **Extensibility**: The `worker_id + size + data` format allows adding new workers

### 2.2 Type 3 Format (Modern)

#### Top-Level Structure

```
┌────────────┬──────────────────────────────────────────────────────────────┐
│  Offset    │  Field                                                       │
├────────────┼──────────────────────────────────────────────────────────────┤
│    0-1     │  options_type = 3 (uint16, big-endian)                       │
│    2+      │  worker_options[] (concatenated, variable length)            │
└────────────┴──────────────────────────────────────────────────────────────┘
```

#### Worker Option Format

```
┌────────────┬─────────────┬────────────────────────────────────────────────┐
│  Byte 0    │  Bytes 1-2  │  Bytes 3+                                      │
├────────────┼─────────────┼────────────────────────────────────────────────┤
│  worker_id │  size       │  option_data[size]                             │
│  (uint8)   │  (uint16)   │  (bytes)                                       │
└────────────┴─────────────┴────────────────────────────────────────────────┘

Worker IDs:
  1 = Executor
  2 = DVN
```

### 2.3 Executor Options

| Type | Name        | Size     | Format                            | Description                      |
| ---- | ----------- | -------- | --------------------------------- | -------------------------------- |
| 1    | LZRECEIVE   | 16 or 32 | `gas[16] + value?[16]`            | Gas/value for lzReceive callback |
| 2    | NATIVE_DROP | 48       | `amount[16] + receiver[32]`       | Airdrop native tokens to address |
| 3    | LZCOMPOSE   | 18 or 34 | `index[2] + gas[16] + value?[16]` | Gas/value for compose call       |
| 4    | ORDERED     | 0        | (empty)                           | Enforce in-order execution       |
| 5    | LZREAD      | 20 or 36 | `gas[16] + size[4] + value?[16]`  | Read operation parameters        |

### 2.4 DVN Options

```
┌────────────┬────────────┬──────────────────────────────────────────────────┐
│  Byte 0    │  Byte 1    │  Bytes 2+                                        │
├────────────┼────────────┼──────────────────────────────────────────────────┤
│  dvn_idx   │ option_type│  option_data                                     │
│  (uint8)   │  (uint8)   │                                                  │
└────────────┴────────────┴──────────────────────────────────────────────────┘

dvn_idx: Index into the combined (required + optional) DVN list (0-based)
```

| Type | Name     | Description                                   |
| ---- | -------- | --------------------------------------------- |
| 1    | PRECRIME | Enable precrime verification for this message |

### 2.5 Legacy Formats (V1 Compatibility)

**Type 1 (Gas Only):**

```
┌─────────────────┬────────────────────────────────────────────────────────┐
│  Bytes 0-1      │  Bytes 2-33                                            │
├─────────────────┼────────────────────────────────────────────────────────┤
│  type = 1       │  extraGas (uint256, 32 bytes)                          │
└─────────────────┴────────────────────────────────────────────────────────┘
```

**Type 2 (Gas + Native Airdrop):**

```
┌─────────────────┬────────────────┬───────────────┬─────────────────────────┐
│  Bytes 0-1      │  Bytes 2-33    │  Bytes 34-65  │  Bytes 66+              │
├─────────────────┼────────────────┼───────────────┼─────────────────────────┤
│  type = 2       │  extraGas      │  dstNativeAmt │  dstNativeAddress       │
│  (uint16)       │  (uint256)     │  (uint256)    │  (bytes, typically 32)  │
└─────────────────┴────────────────┴───────────────┴─────────────────────────┘
```
