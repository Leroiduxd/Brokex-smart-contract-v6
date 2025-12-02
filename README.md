
---

````markdown
# Brokex Perpetual Trading Contracts

This repository contains the core smart contracts of the **Brokex** protocol, a decentralized CFD engine that brings real-world liquidity (stocks, FX, commodities, indices, crypto) on-chain.

The system is built around four main contracts:

- `Calculator.sol` — Asset configuration, lot sizes, spreads & funding.
- `Trades.sol` — Main trading engine (market & limit orders, SL/TP, funding, liquidation).
- `BrokexMetaRelayer.sol` — EIP-712 meta-transaction relayer (gasless UX).
- `Vault.sol` — Margin custody & PnL settlement between traders and the protocol.

All contracts are written in **Solidity 0.8.24** and rely on **OpenZeppelin** libraries.

---

## Architecture

The high-level architecture and main interactions are shown below.

```mermaid
flowchart LR

    subgraph User Side
        U[User Wallet]
        R[Relayer / Paymaster]
    end

    subgraph Core Contracts
        M[BrokexMetaRelayer.sol]
        T[Trades.sol]
        C[Calculator.sol]
        V[Vault.sol]
    end

    subgraph External
        O[Supra Oracle<br/>verifyOracleProofV2]
        TK[ERC20 Token<br/>(USDT/USDC)]
    end

    U -->|EIP-712 Signatures| M
    M -->|open/close via relayer| T

    U -->|direct tx (no relayer)| T

    T -->|getLot, fee, fund,<br/>market status| C
    T -->|lock, unlock,<br/>settle PnL| V
    V -->|transferFrom / transfer| TK

    T -->|verifyOracleProofV2<br/>(price proof)| O
````

**Flows (simplified):**

1. User signs an **EIP-712** message (open/close/update).
2. Relayer calls `BrokexMetaRelayer`, which verifies the signature and forwards the call to `Trades`.
3. `Trades`:

   * Pulls price from **Supra Oracle** via a proof.
   * Uses `Calculator` for lot size, spread, funding, and market status.
   * Uses `Vault` to lock/unlock margin and settle PnL.
4. `Vault` holds the underlying ERC-20 token (e.g. USDT/USDC) and transfers value between the **owner (protocol)** and **traders**.

---

## Contracts Overview

### 1. `Calculator.sol`

#### Purpose

`Calculator` is a configuration contract that stores:

* Lot sizes for each asset.
* Market IDs (grouping assets by market type).
* Market open/closed status.
* Spread + commission per asset (`feeX6`, price × 1e6).
* Funding rate per asset (`fundX6`, price × 1e6 per interval).

It exposes simple **view** functions used by `Trades` to perform all arithmetic consistently.

#### Key Data Structures

```solidity
struct LotCfg {
    uint256 num;     // lot numerator
    uint256 den;     // lot denominator
    bool    listed;  // asset is known/active
    uint8   market;  // market ID (e.g. FX, stocks, crypto)
}

mapping(uint32 => LotCfg) public asset;
mapping(uint8  => bool)   public marketOpen;
mapping(uint32 => uint64) public feeX6;   // spread + commission (price × 1e6)
mapping(uint32 => int64)  public fundX6;  // funding per interval (price × 1e6)
```

#### Admin Functions

* `addAsset(uint32 assetId, uint8 marketId, uint256 num, uint256 den)`

  * Registers a new asset with its lot configuration and market.
* `setLot(uint32 assetId, uint256 num, uint256 den)`

  * Updates lot size for an existing asset.
* `setMarket(uint32 assetId, uint8 marketId)`

  * Changes the market ID for an asset.
* `setMarketOpen(uint8 marketId, bool open)`

  * Opens or closes trading for a given market.
* `setFee(uint32 assetId, uint64 valueX6)`

  * Sets **spread + commission** (in price × 1e6).
* `setFund(uint32 assetId, int64 valueX6)`

  * Sets **funding rate** per interval (in price × 1e6).

All admin functions are `onlyOwner` (OpenZeppelin `Ownable`).

#### View Functions

* `getLot(uint32 assetId) returns (uint256 num, uint256 den)`
* `getMarket(uint32 assetId) returns (uint8 marketId)`
* `isMarketOpen(uint32 assetId) returns (bool)`
* `fee(uint32 assetId) returns (uint64 feeX6)`
* `fund(uint32 assetId) returns (int64 fundX6)`
* `costs(uint32 assetId) returns (uint64 feeX6, int64 fundX6)`

---

### 2. `Trades.sol`

#### Purpose

`Trades` is the **core trading engine**. It manages:

* Market and limit orders.
* Long/short positions with leverage.
* Margin calculation based on lots and prices.
* Stop loss (SL), take profit (TP), and liquidation prices.
* Funding rate over time.
* Exposure tracking per asset.
* Settlement of PnL via the `Vault`.

It integrates directly with:

* `Calculator` (configuration & pricing parameters).
* `Vault` (custody and PnL).
* `ISupraOraclePull` (price proofs).
* `BrokexMetaRelayer` (via the `ITrades` interface).

#### Key Constants

```solidity
uint256 private constant WAD = 1e18;
uint256 private constant LIQ_LOSS_OF_MARGIN_WAD = 8e17; // 0.8
uint16  private constant TOL_BPS = 5;                   // 0.05%
uint256 private constant PROOF_MAX_AGE = 60;            // seconds
```

#### Trade Structure

```solidity
struct Trade {
    address owner;
    uint32  asset;
    uint16  lots;
    uint8   flags;       // bit0 = longSide ; bits4..7 = state

    int64   entryX6;     // 0 if pending LIMIT (not executed)
    int64   targetX6;    // LIMIT target; 0 for MARKET
    int64   slX6;        // Stop loss price; 0 if not set
    int64   tpX6;        // Take profit price; 0 if not set
    int64   liqX6;       // Liquidation price (fixed at open)

    uint16  leverageX;   // leverage factor
    uint64  openedAt;    // timestamp (position open)
    uint64  marginUsd6;  // locked margin in stablecoin (×1e6)
}
```

#### States

States are encoded in `flags`:

* `STATE_ORDER = 0` — Limit order (pending)
* `STATE_OPEN  = 1` — Position is open
* `STATE_CLOSED = 2` — Position closed and settled
* `STATE_CANCELLED = 3` — Limit order cancelled

Long/short side is stored in `flags` bit 0.

---

#### Oracle Integration (Supra)

Prices come from `verifyOracleProofV2(bytes proof)` which returns:

* asset IDs (`pairs`)
* prices
* decimals
* timestamps

`_priceX6FromProof`:

* Selects the right asset ID.
* Normalizes to **6 decimals** (×1e6).
* Checks timestamp freshness (`PROOF_MAX_AGE`).
* Ensures range safety (fits into `uint64`).

All order execution and closing is based on this **oracle mid-price**.

---

#### Spread & Funding

* At **open (MARKET)**:

  * Price = oracle mid ± spread (`feeX6`), using `_applySpread`.
* At **close** (market close or SL/TP/LIQ):

  * Funding is applied based on elapsed time in 45-minute intervals.
  * Spread is applied again in an antagonistic way (so open and close are symmetric).

```solidity
int64 pxClose = _closePxWithFundingAndSpread(t, oracleX6);
```

---

#### Margin & Exposure

Helper functions:

* `_qty1e18(assetId, lots)` — converts lots to **quantity (1e18)** using `Calculator.getLot`.
* `_notionalUsd6(qty, priceX6)` — notional in stablecoin ×1e6.
* `_marginUsd6(notional, leverage)` — required margin.
* `_liqPriceX6(entry, leverage, longSide)` — liquidation price at open.

Exposure is tracked per asset:

```solidity
mapping(uint32 => uint256) public longLots;
mapping(uint32 => uint256) public shortLots;
```

These are incremented/decremented whenever a position opens or closes.

---

#### Main Functions

##### Opening LIMIT Orders

* `openLimit(assetId, longSide, leverageX, lots, targetX6, slX6, tpX6)`

  * Locks margin in `Vault`.
  * Computes liquidation price from target.
  * Validates SL/TP ranges.
  * Creates a **pending LIMIT** with `STATE_ORDER`.

* `openLimitRelayer(...)` (same, but callable only by `relayer` address)

Execution of LIMITs:

* `executeLimit(id, proof)`

  * Reads oracle mid price.
  * Checks if price is within tolerance of `targetX6`.
  * Applies spread for actual execution price.
  * Sets `entryX6`, `STATE_OPEN`, and `openedAt`.
  * Adds exposure.

* `execLimits(assetId, ids[], proof)` — batch version.

##### Opening MARKET Orders

* `openMarket(proof, assetId, longSide, leverageX, lots, slX6, tpX6)`

  * Ensures market is open via `Calculator.isMarketOpen`.
  * Gets oracle mid price from proof.
  * Fetches spread via `Calculator.fee`.
  * Computes execution price (oracle ± spread).
  * Locks margin via `Vault.lock`.
  * Creates OPEN trade via `_createMarketTrade`.

* `openMarketRelayer(...)` — same, but for the `relayer`.

##### Updating SL / TP

* `updateStops(id, newSLx6, newTPx6)`
* `setSL(id, newSLx6)`
* `setTP(id, newTPx6)`

Relayer variants:

* `updateStopsRelayer(...)`
* `setSLRelayer(...)`
* `setTPRelayer(...)`

All validate that SL/TP are consistent with entry/target and liquidation price.

##### Closing Positions

* `close(id, reason, proof)`

  * `reason` is SL/TP/LIQ.
  * Uses oracle mid price from proof.
  * Checks that price has touched trigger (with tolerance).
  * Computes close price with funding + spread.
  * Finalizes PnL via `_finalizeClose`.

* `closeMarket(id, proof)`

  * User closes at market.
  * Uses oracle mid + funding + spread adjustments.

* `closeMarketRelayer(trader, id, proof)` — relayer version.

Batch closing:

* `closeBatch(assetId, reason, ids[], proof)`

##### Views

* `stateOf(id)` — returns state.
* `isLong(id)` — returns side.
* `getTrade(id)` — returns full struct.
* `getExposure(assetId)` — returns `(longLots, shortLots)`.
* `longExposure(assetId)` and `shortExposure(assetId)`.

---

### 3. `BrokexMetaRelayer.sol`

#### Purpose

The **MetaRelayer** enables **gasless** operations via **EIP-712 signatures**.
Users sign messages off-chain; a relayer submits them on-chain.

The relayer can:

* Open LIMIT orders.
* Cancel LIMIT orders.
* Update SL/TP.
* Open MARKET orders.
* Close MARKET orders.

But **cannot** move user funds directly; it only forwards calls to `Trades`.

#### Core Ideas

* Each trader has a `nonce` to prevent replay.
* Each signed message includes a `deadline`.
* Signatures are verified with EIP-712 typed data.
* The relayer calls methods on an `ITrades` interface.

#### Data Structures

```solidity
struct LimitOpenCall {
    address trader;
    uint32 assetId;
    bool   longSide;
    uint16 leverageX;
    uint16 lots;
    int64  targetX6;
    int64  slX6;
    int64  tpX6;
    uint256 nonce;
    uint256 deadline;
}

struct MarketOpenCall {
    address trader;
    uint32 assetId;
    bool   longSide;
    uint16 leverageX;
    uint16 lots;
    int64  slX6;
    int64  tpX6;
    uint256 nonce;
    uint256 deadline;
}

struct MarketCloseCall {
    address trader;
    uint32 id;
    uint256 nonce;
    uint256 deadline;
}
```

#### Main Functions

* `executeOpenLimit(...)`

  * Calls `trades.openLimitRelayer(...)` after verifying signature.
* `executeCancelLimit(...)`

  * Calls `trades.cancelRelayer(...)`.
* `executeUpdateStops(...)`

  * Calls `trades.updateStopsRelayer(...)`.
* `executeSetSL(...)`, `executeSetTP(...)`

  * Call the respective SL/TP relayer methods.
* `executeOpenMarket(...)`

  * Calls `trades.openMarketRelayer(...)`.
* `executeCloseMarket(...)`

  * Calls `trades.closeMarketRelayer(...)`.

`_verify(...)` handles the EIP-712 hashing and ECDSA recovery.

---

### 4. `Vault.sol`

#### Purpose

`Vault` is the **custody & settlement layer** of the protocol. It holds the stablecoin (ERC-20) and manages:

* User balances.
* Locked margin.
* Protocol (owner) liquidity.
* PnL settlement between traders and the protocol.

Only the **proxy** (the `Trades` contract) is allowed to call `lock`, `unlock`, and `settle`.

#### State

```solidity
IERC20 public immutable token;
address public immutable owner;
address public proxy;

mapping(address => uint256) public balance;
mapping(address => uint256) public locked;
```

#### Roles

* `owner`

  * Deploys the contract.
  * Provides initial liquidity and earns PnL from losing traders.
* `proxy`

  * Typically the `Trades` contract.
  * Controls `lock`, `unlock`, `settle`.
* `users`

  * Deposit and withdraw margin.

#### Admin Functions

* `setProxy(address _proxy)` — set the trading contract.
* `ownerDeposit(uint256 amount)` — owner adds liquidity.
* `ownerWithdraw(uint256 amount)` — owner withdraws free liquidity.

#### User Functions

* `deposit(uint256 amount)`

  * `transferFrom(msg.sender, address(this))`
  * Increases `balance[msg.sender]`.
* `withdraw(uint256 amount)`

  * Only from **free** funds (`balance - locked`).
  * Transfers token back to user.

#### Trading Functions (proxy only)

* `lock(address user, uint256 amount)`

  * Increments `locked[user]` after checking free balance.
* `unlock(address user, uint256 amount)`

  * Decrements `locked[user]`.
* `settle(address user, int256 pnl)`

  * If `pnl > 0`:

    * Trader wins: owner pays.
  * If `pnl < 0`:

    * Trader loses: owner receives.

This models a **broker style** setup where the protocol’s liquidity absorbs trader PnL.

#### Views

* `available(address user)` — `balance[user] - locked[user]` (or 0).
* `total(address user)` — full balance.
* `vaultBalance()` — ERC-20 balance of the contract.
* `ownerBalance()` — tracked balance of the owner.

---

## Development & Deployment

### Requirements

* Solidity `^0.8.24`
* OpenZeppelin Contracts (for `Ownable`, `EIP712`, `ECDSA`)
* A Supra Oracle Pull contract implementing:

  * `verifyOracleProofV2(bytes calldata)`

### Typical Deployment Order

1. Deploy the ERC-20 stablecoin (test token or real USDT/USDC).
2. Deploy `Vault` with the token address.
3. Deploy `Calculator`.
4. Deploy `Trades` with:

   * `Calculator` address
   * `Vault` address
   * `SupraOraclePull` address
5. Set `proxy` in `Vault` to the `Trades` contract.
6. Deploy `BrokexMetaRelayer` with the `Trades` address.
7. Set `relayer` address in `Trades`.
8. Configure assets & markets via `Calculator` (`addAsset`, `setFee`, `setFund`, `setMarketOpen`).

---

## Security Notes

* No external calls during PnL settlement except ERC-20 transfers.
* Oracle proofs checked for timestamp freshness and range.
* Funding and spread application is bounded to avoid overflow.
* Nonces + deadlines protect EIP-712 messages from replay.
* `onlyOwner` and `onlyProxy` modifiers enforce role separation.

---

## License

All contracts are released under the **MIT License**.

```


