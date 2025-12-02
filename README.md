
---

# ✅ **README.md — Brokex Smart Contracts**

````markdown
# Brokex — Perpetual CFD Smart Contracts

This repository contains the core smart contracts powering **Brokex**, a decentralized CFD broker that brings real-world liquidity (stocks, FX, indices, commodities, crypto) on-chain.

The system is composed of four main contracts:

- **Calculator.sol** – Asset configuration, lot sizes, spreads & funding.
- **Trades.sol** – Trading engine (market/limit orders, SL/TP, liquidation, funding).
- **BrokexMetaRelayer.sol** – EIP-712 meta-transaction relayer for gasless UX.
- **Vault.sol** – Margin custody & PnL settlement between traders and protocol.

All contracts are written in **Solidity 0.8.24**.

---

# 🧩 Architecture

Below is the high-level architecture of the Brokex protocol:

```mermaid
flowchart LR
    U[User Wallet] -->|Signs EIP-712| M[BrokexMetaRelayer]
    R[Relayer] -->|Submits Tx| M

    M -->|Forward Validated Calls| T[Trades Contract]

    T -->|Get lot, fee, funding,\nmarket status| C[Calculator]
    T -->|lock/unlock/settle| V[Vault]

    V -->|ERC20 transfers| TK[Stablecoin Token]

    T -->|verifyOracleProofV2| O[Supra Oracle]

    subgraph External Systems
        O
        TK
    end
````

---

# 📘 Contract Overview

## 1. **Calculator.sol**

### Purpose

Stores all asset configuration:

* lot sizes
* market grouping
* market open/closed
* spread + commission (`feeX6`)
* funding rate (`fundX6`)

### Key Mappings

* `asset[assetId] → LotCfg`
* `marketOpen[marketId] → bool`
* `feeX6[assetId] → uint64`
* `fundX6[assetId] → int64`

### Admin Functions

* `addAsset(assetId, marketId, num, den)`
* `setLot(assetId, num, den)`
* `setMarket(assetId, marketId)`
* `setMarketOpen(marketId, bool)`
* `setFee(assetId, valueX6)`
* `setFund(assetId, valueX6)`

### Views

* `getLot(assetId)`
* `isMarketOpen(assetId)`
* `fee(assetId)`
* `fund(assetId)`
* `costs(assetId)`

---

## 2. **Trades.sol**

### Purpose

Main engine of Brokex:

* Opens limit & market positions
* Applies spread and funding
* Handles liquidation
* Manages SL/TP
* Validates price via **Supra Oracle Proof V2**
* Settles PnL in the Vault
* Tracks exposure per asset

### Position Lifecycle

1. **Open Limit**
2. **Execute Limit (via Oracle Proof)**
3. **Open Market (via Oracle + Spread)**
4. **Funding accrues every 45 min**
5. **Close via:**

   * Market Close
   * Stop Loss
   * Take Profit
   * Liquidation

### Key Features

* Oracle price validation via:

  * `_priceX6FromProof()`
* Spread application:

  * long → price + fee
  * short → price - fee
* Funding applied at close
* Exposure tracking:

  * `longLots[assetId]`
  * `shortLots[assetId]`
* Meta-relayer integration

### Important Functions

#### Open Orders

* `openMarket(proof, ...)`
* `openLimit(...)`
* `openMarketRelayer(...)`
* `openLimitRelayer(...)`

#### Stop Management

* `updateStops(id, sl, tp)`
* `setSL(id, sl)`
* `setTP(id, tp)`
* Relayer equivalents

#### Close Orders

* `close(id, reason, proof)`
* `closeMarket(id, proof)`
* `closeMarketRelayer(...)`
* `closeBatch(...)`

---

## 3. **BrokexMetaRelayer.sol**

### Purpose

Allows **gasless trading** through **EIP-712 signature execution**.

The relayer verifies:

* user signature
* nonce (anti-replay)
* deadline
* typed data hashing

Then forwards calls to the `Trades` contract.

### Supports:

* Limit open
* Limit cancel
* SL/TP update
* Market open
* Market close

### Security

✔ Signatures validated
✔ Replay-protected
✔ User cannot be impersonated
✔ Relayer cannot alter prices

---

## 4. **Vault.sol**

### Purpose

Holds user balances & protocol liquidity.

Core functions:

* `deposit()`
* `withdraw()`
* `lock()` — margin lock at trade open
* `unlock()` — released at close
* `settle()` — transfers PnL:

  * **Trader wins → owner pays**
  * **Trader loses → owner receives**

### Logic Model

This replicates a **real CFD broker model**:

* protocol = market maker
* user margin = isolated
* PnL flows between user and protocol

---

# 🚀 Deployment Overview

1. Deploy stablecoin (USDT/USDC test token)
2. Deploy **Vault**
3. Deploy **Calculator**
4. Deploy **Trades** with links to:

   * Calculator
   * Vault
   * Supra Oracle Pull
5. Set `proxy` in Vault → Trades address
6. Deploy **BrokexMetaRelayer**
7. Set relayer address in Trades
8. Configure assets in Calculator:

   * `addAsset`
   * `setFee`
   * `setFund`
   * `setMarketOpen`

---

# 🔐 Security Highlights

* Strict Oracle proof timestamp validation
* Funding bounded and safe
* Spread applied safely on both sides
* EIP-712 signatures verified on-chain
* Full replay protection
* Role separation:

  * owner
  * relayer
  * proxy (Trades)
  * users

---

# 📄 License

MIT License.

---

```



