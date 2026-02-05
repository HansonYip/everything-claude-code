# LayerZero PriceFeed Specification

> **Source:** [EVM Reference Implementation](https://github.com/LayerZero-Labs/LayerZero-v2/blob/main/packages/layerzero-v2/evm/messagelib/contracts/PriceFeed.sol) > **Purpose:** Price oracle for gas prices, token exchange rates, and fee calculation

---

## Table of Contents

1. [Design Overview](#1-design-overview)
2. [Fee Models](#2-fee-models)
3. [Data Structures](#3-data-structures)
4. [Operations](#4-operations)
5. [Security Considerations](#5-security-considerations)

---

## 1. Design Overview

The PriceFeed is a price oracle that provides gas prices and token exchange rates for destination chains.

**Core Responsibilities:**

| Aspect          | Description                                                                                 |
| --------------- | ------------------------------------------------------------------------------------------- |
| **Price Data**  | Maintain gas prices, token price ratios, and per-byte costs for each destination            |
| **Fee Models**  | Support different gas models for various chain architectures (standard, Arbitrum, Optimism) |
| **USD Pricing** | Provide native token price in USD for margin calculations                                   |

## 2. Fee Models

### 2.1 Default Model (Standard EVM Chains)

```
calldataGas = callDataSize × gasPerByte
totalGas = calldataGas + gas
remoteFee = totalGas × gasPriceInUnit
fee = remoteFee × priceRatio / DENOMINATOR
```

### 2.2 Arbitrum Model (L2 with Compressed L1 Data)

```
L1 FEE (compressed calldata):
  compressedSize = callDataSize × COMPRESSION_PERCENT / 100
  l1Gas = compressedSize × gasPerL1CallDataByte

L2 FEE:
  l2CallDataGas = callDataSize × gasPerByte
  l2ExecutionGas = gas + gasPerL2Tx

TOTAL:
  totalGas = l1Gas + l2CallDataGas + l2ExecutionGas
  fee = totalGas × gasPriceInUnit × priceRatio / DENOMINATOR
```

### 2.3 Optimism Model (L1 + L2 with Separate Pricing)

```
L1 FEE (Ethereum mainnet pricing):
  l1Gas = callDataSize × ethereumPrice.gasPerByte + L1_OVERHEAD
  l1Fee = l1Gas × ethereumPrice.gasPriceInUnit × ethereumPrice.ratio

L2 FEE (Optimism pricing):
  l2Gas = callDataSize × optimismPrice.gasPerByte + gas
  l2Fee = l2Gas × optimismPrice.gasPriceInUnit × optimismPrice.ratio

TOTAL:
  fee = (l1Fee + l2Fee) / DENOMINATOR
```

## 3. Data Structures

```
Price {
    priceRatio: u128        // Token price ratio: destination/source
    gasPriceInUnit: u64     // Gas price in destination chain's smallest unit
    gasPerByte: u32         // Gas cost per byte of calldata
}

ModelType {
    DEFAULT = 0     // Standard EVM gas model
    ARB_STACK = 1   // Arbitrum-style L1+L2 model
    OP_STACK = 2    // Optimism-style L1+L2 model
}
```

## 4. Operations

| Operation         | Authorization | Description                                      |
| ----------------- | ------------- | ------------------------------------------------ |
| Set Price Updater | Owner         | Add/remove addresses authorized to update prices |
| Set Price         | Price Updater | Update gas prices and ratios for destinations    |
| Set Model Type    | Owner         | Configure which fee model applies                |
| Get Price         | Anyone        | Query current price data for a destination       |
| Estimate Fee      | Anyone        | Calculate fee for given calldata size and gas    |

## 5. Security Considerations

1. **Updater Trust**: Price updaters can influence all fee calculations
2. **Stale Prices**: No on-chain staleness check; off-chain systems must ensure regular updates
3. **Price Manipulation**: Extreme price ratios could result in excessive or insufficient fees
