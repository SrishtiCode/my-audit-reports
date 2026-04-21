# Panoptic Audit

**Prepared by:** [Srishti](https://SrishtiCode.io)

**Date:** April, 2026

**Version:** 1.0

---

## Table of Contents

- [Protocol Summary](#protocol-summary)
- [Disclaimer](#disclaimer)
- [Risk Classification](#risk-classification)
- [Architecture Overview](#architecture-overview)
- [Audit Methodology & Workflow](#audit-methodology--workflow)
- [Audit Details](#audit-details)
- [Executive Summary](#executive-summary)
- [Findings](#findings)

---

## Protocol Summary

**Panoptic** is a fully on-chain, permissionless options trading protocol built on top of Uniswap V3 and V4. It enables users to buy and sell options on any Uniswap token pair without expiries, without order books, and without oracles for pricing — instead leveraging Uniswap's existing liquidity infrastructure as the underlying options engine.

The protocol is composed of four core contracts that work in concert:

**1. SemiFungiblePositionManager (SFPM)**
The SFPM is the low-level "engine" of Panoptic — a gas-efficient alternative to Uniswap's NonFungiblePositionManager. It manages complex, multi-leg Uniswap positions encoded in ERC-1155 tokenIds, performs single-token swaps to allow users to mint positions with only one asset, and critically supports both standard LP positions (liquidity added to Uniswap) and "long" positions (liquidity removed/burnt from Uniswap). Separate implementations exist for Uniswap V3 (`SemiFungiblePositionManagerV3`) and V4 (`SemiFungiblePositionManagerV4`).

**2. PanopticPool**
The "conductor" of the protocol. All user-facing interactions — minting/burning positions, liquidations, force exercises, premium accumulation — originate here via a unified `dispatch()` / `dispatchFrom()` entry point. It orchestrates calls to the SFPM, CollateralTracker, and RiskEngine.

**3. RiskEngine**
The central risk and solvency calculator. It holds no funds or user state, but computes all collateral requirements, liquidation bonuses, force exercise costs, dynamic interest rates (via a PID controller), and manages the internal pricing oracle with EMA and median-filter safeguards against manipulation.

**4. CollateralTracker**
An ERC-4626 vault for each constituent token (token0 and token1) that holds Passive Liquidity Provider (PLP) deposits and option position collateral. It implements compound interest accrual, commission collection and distribution, premium settlement between buyers and sellers, and liquidation settlement.

**Key Actors:**
- **Passive PLPs**: Deposit tokens into CollateralTracker vaults; earn commissions and interest
- **Option Sellers**: Add liquidity to Uniswap via Panoptic; earn fee multipliers; pay interest to PLPs
- **Option Buyers**: Remove liquidity from Uniswap via Panoptic; pay premia to sellers
- **Liquidators**: Close distressed accounts below maintenance margin; earn liquidation bonuses
- **Force Exercisors**: Forcefully close out-of-range long positions; pay a fee to the exercised user

**Underlying Infrastructure:** Uniswap V3 / V4

**Position Encoding:** ERC-1155 tokenIds (up to four legs per position)

**Collateral Vaults:** ERC-4626 (one per token per Panoptic instance)

**Deployment:** Permissionless — any Uniswap pool can have a Panoptic instance deployed on top of it

---

## Disclaimer

> **Independent Security Review**
> Every effort was made to identify vulnerabilities within the allotted time. However, this report does not constitute an endorsement of the underlying business or product. The audit was time-boxed and focused solely on the security aspects of the Solidity implementation. The auditor assumes no responsibility for the findings presented herein.

---

## Risk Classification

|                        |     | Impact: High | Impact: Medium | Impact: Low |
| ---------------------- | --- | :----------: | :------------: | :---------: |
| **Likelihood: High**   |     |      H       |      H/M       |      M      |
| **Likelihood: Medium** |     |     H/M      |       M        |     M/L     |
| **Likelihood: Low**    |     |      M       |      M/L       |      L      |

Severity is determined using the [CodeHawks severity matrix](https://docs.codehawks.com/hawks-auditors/how-to-evaluate-a-finding-severity).

---

## Architecture Overview

### Protocol Description

Each instance of the Panoptic protocol is deployed on top of a single Uniswap pool and consists of:

- **One PanopticPool** — orchestrates all user interactions
- **One RiskEngine** — computes collateral requirements, solvency, liquidation parameters, interest rates, and oracle state
- **Two CollateralTrackers** — one ERC-4626 vault per token (token0 and token1) tracking PLP deposits and option collateral
- **One canonical SFPM** — shared across all Panoptic pools; manages Uniswap liquidity positions as ERC-1155 tokens

Deployment of new Panoptic instances is handled permissionlessly via `PanopticFactoryV3` (for Uniswap V3) or `PanopticFactoryV4` (for Uniswap V4).

### Contract Architecture

```
Panoptic Protocol
│
├── SemiFungiblePositionManagerV3.sol / SemiFungiblePositionManagerV4.sol
│   ← The "engine." ERC-1155 token manager for multi-leg Uniswap positions.
│   ← Shared singleton across all Panoptic pool instances.
│   ├── mintTokenizedPosition()    ← Mints an ERC-1155 tokenId; adds or removes Uniswap liquidity
│   ├── burnTokenizedPosition()    ← Burns a tokenId; reverses the Uniswap liquidity operation
│   ├── swapInAMM()                ← Internal; performs single-token swap to enable one-sided minting
│   └── getAccountLiquidity()      ← Returns net and removed liquidity for a position chunk
│
├── PanopticPool.sol
│   ← The "conductor." Unified entry point for all protocol interactions.
│   ├── dispatch()                 ← Primary entry point; user executes actions on own account
│   ├── dispatchFrom()             ← Operator executes actions on behalf of an approved user
│   ├── [mint action]              ← Calls SFPM to open position; RiskEngine checks post-mint solvency
│   ├── [burn action]              ← Calls SFPM to close position; RiskEngine checks collateral
│   ├── [liquidate action]         ← Closes all legs of a distressed account; distributes bonus via CT
│   ├── [forceExercise action]     ← Force-closes an out-of-range long position; pays fee to victim
│   ├── [settlePremium action]     ← Forces solvent option buyers to pay outstanding premium to sellers
│   └── [pokeOracle action]        ← Inserts a new tick observation into the RiskEngine median buffer
│
├── RiskEngine.sol
│   ← Central risk calculator. Holds no funds or user balances.
│   ├── isAccountSolvent()         ← Returns whether an account meets maintenance margin requirements
│   ├── getCollateralRequirements()← Computes required collateral for a set of position legs
│   ├── getLiquidationBonus()      ← Computes liquidator bonus and protocol loss for a distressed account
│   ├── exerciseCost()             ← Computes fee paid to force-exercised user (exponential decay by strike distance)
│   ├── getBorrowRate()            ← Returns current dynamic borrow rate based on pool utilization (PID model)
│   ├── updateOracle()             ← Inserts new tick observation; updates EMA and median ring buffer
│   └── getSafePrice()             ← Returns manipulation-resistant price with volatility safeguards
│
├── CollateralTracker.sol  (×2 per Panoptic instance — one per token)
│   ← ERC-4626 vault for PLP deposits and option collateral.
│   ├── deposit() / mint()         ← PLP deposits tokens; receives shares
│   ├── withdraw() / redeem()      ← PLP withdraws tokens; burns shares
│   ├── takeCommission()           ← Collects commission on option mint/burn; splits to protocol/builder/PLPs
│   ├── settlePremium()            ← Transfers premia between option buyers and sellers
│   ├── accrueInterest()           ← Compounds interest on borrowed liquidity using global borrow index
│   ├── delegate() / revoke()      ← Mints/burns virtual shares for active option positions
│   └── settleLiquidation()        ← Mints shares to liquidator; absorbs protocol loss if account insolvent
│
├── PanopticFactoryV3.sol / PanopticFactoryV4.sol
│   ← Permissionless deployer of new Panoptic instances on Uniswap pools.
│   ├── deployNewPool()            ← Deploys PanopticPool + two CollateralTrackers for a Uniswap pool
│   ├── minePoolAddress()          ← On-chain CREATE3 salt mining for vanity pool addresses
│   └── [NFT reward]               ← Issues FactoryNFT to the deployer as an incentive
│
└── libraries/
    ├── PanopticMath               ← TWAP, price conversions, position sizing, tick math
    ├── FeesCalc                   ← Up-to-date swap fee calculation for liquidity chunks
    ├── Math                       ← Generic math: abs(), mulDiv(), etc.
    ├── TokenId                    ← Encode/decode up to 4 option legs in a 256-bit ERC-1155 tokenId
    ├── LiquidityChunk             ← Encode tickLower, tickUpper, liquidity into a packed type
    ├── LeftRight                  ← Packed type holding two 128-bit token0/token1 values
    ├── SafeTransferLib            ← Safe ERC-20 transfers with missing-return-value handling
    ├── CallbackLib                ← Verify and decode Uniswap mint/swap callbacks
    └── TransientReentrancyGuard   ← Reentrancy protection via transient storage (EIP-1153)
```

### Data Flow

```
════════════════════════════════════════════════════════
  FLOW 1: Option Seller mints a short position
════════════════════════════════════════════════════════

[Option Seller]
        │
        │  1. deposit(token0/token1) → becomes a PLP, receives CT shares
        ▼
[CollateralTracker (token0 + token1)]
        │  Collateral is held; delegation mints virtual shares for the position
        │
        │  2. dispatch(mintAction, tokenId, positionSize)
        ▼
[PanopticPool]
        │  • Decodes tokenId to identify up to 4 legs
        │  • Calls RiskEngine to check post-mint solvency
        ├──────────────────────────────────────────────────►[RiskEngine]
        │                                                    • getCollateralRequirements()
        │                                                    • isAccountSolvent() ✓
        │◄─────────────────────────────────────────────────
        │
        │  • Calls SFPM to execute Uniswap liquidity addition
        ▼
[SemiFungiblePositionManager]
        │  • Adds liquidity to Uniswap V3/V4 at specified tick range
        │  • Mints ERC-1155 tokenId to seller
        ▼
[Uniswap V3/V4 Pool]
        └── Liquidity now active; earns swap fees while in range

════════════════════════════════════════════════════════
  FLOW 2: Option Buyer purchases the position
════════════════════════════════════════════════════════

[Option Buyer]
        │
        │  1. deposit() → becomes PLP; posts collateral in CollateralTracker
        │
        │  2. dispatch(mintAction, long tokenId, positionSize)
        ▼
[PanopticPool]
        │  • Identifies leg as "long" (remove liquidity from Uniswap)
        │  • Calls RiskEngine to verify buyer solvency
        ├──────────────────────────────────────────────────►[RiskEngine]
        │                                                    • isAccountSolvent() ✓
        │◄─────────────────────────────────────────────────
        │
        │  • Calls SFPM to remove seller's liquidity from Uniswap
        ▼
[SemiFungiblePositionManager]
        │  • Burns liquidity chunk from Uniswap; moves tokens back to Panoptic
        │  • Mints long ERC-1155 tokenId to buyer
        ▼
[CollateralTracker]
        └── settlePremium() tracks premia owed from buyer to seller over time

════════════════════════════════════════════════════════
  FLOW 3: Liquidation of a distressed account
════════════════════════════════════════════════════════

[Liquidator]
        │
        │  dispatch(liquidateAction, distressedAccount)
        ▼
[PanopticPool]
        │  • Calls RiskEngine: isAccountSolvent() → false
        │  • Calls RiskEngine: getLiquidationBonus() → bonus amount
        │  • Calls SFPM to close all position legs
        │  • Calls CollateralTracker to settle liquidation
        ▼
[CollateralTracker]
        │  • Mints bonus shares to liquidator
        │  • If account insolvent: absorbs protocol loss from PLP pool
        ▼
[Liquidator Wallet]
        └── Receives liquidation bonus in CT shares

════════════════════════════════════════════════════════
  FLOW 4: Force Exercise of out-of-range long position
════════════════════════════════════════════════════════

[Force Exercisor (typically an Option Seller)]
        │
        │  dispatch(forceExerciseAction, victim, tokenId)
        ▼
[PanopticPool]
        │  • Calls RiskEngine: exerciseCost() → fee owed to victim
        │    (exponentially higher the closer position is to current price)
        │  • Calls SFPM to close victim's long position
        │  • Transfers exercise fee from exercisor to victim
        ▼
[Uniswap V3/V4 Pool]
        └── Liquidity re-added; seller can now close their short position
```

### Trust Model & Roles

| Role | Address Type | Permissions |
| ---- | ------------ | ----------- |
| **Passive PLP** | EOA or Contract | Calls `deposit()` / `withdraw()` on CollateralTracker; no trading privileges; earns commissions and interest passively |
| **Option Seller** | EOA or Contract | Calls `dispatch()` on PanopticPool to mint short (add liquidity) positions; must maintain collateral above RiskEngine requirements; pays interest to PLPs |
| **Option Buyer** | EOA or Contract | Calls `dispatch()` on PanopticPool to mint long (remove liquidity) positions; pays premia to sellers; must remain solvent per RiskEngine |
| **Liquidator** | EOA or Contract | Calls `dispatch(liquidateAction)` on distressed accounts; provides tokens to close all legs; receives bonus from RiskEngine formula; permissionless role |
| **Force Exercisor** | EOA or Contract (typically a Seller) | Calls `dispatch(forceExerciseAction)` to close out-of-range long positions held by others; pays `exerciseCost()` fee to the exercised user; permissionless role |
| **Approved Operator** | EOA or Contract | Calls `dispatchFrom()` on PanopticPool to act on behalf of an approved user; must be pre-approved by the account owner |
| **Guardian** | Privileged EOA or Multisig | Can override safe-mode settings and lock/unlock pools in emergency situations via RiskEngine guardian controls |
| **Pool Deployer** | EOA or Contract | Calls `PanopticFactoryV3/V4.deployNewPool()` to permissionlessly create a new Panoptic instance on any Uniswap pool; receives FactoryNFT reward |
| **Protocol / Builder** | Address set at deployment | Receives a share of commission fees collected by CollateralTracker on every option mint/burn |
| **Outsider** | Any | Cannot open positions (no collateral deposited); cannot liquidate or force-exercise without providing required tokens; no privileged access |

---

### Key State Variables

| Variable | Contract | Type | Visibility | Purpose |
| -------- | -------- | ---- | ---------- | ------- |
| `s_positionBalance` | PanopticPool | `mapping(address => mapping(TokenId => PositionBalance))` | internal | Tracks each user's open positions: size, pool utilizations at mint, and oracle ticks at mint time |
| `s_options` | PanopticPool | `mapping(address => mapping(TokenId => mapping(uint256 => LeftRight)))` | internal | Accumulated premia (settled and gross) per user per position leg, stored as packed token0/token1 LeftRight values |
| `s_grossPremiumLast` | PanopticPool | `mapping(TokenId => mapping(uint256 => LeftRight))` | internal | Last recorded gross premium per liquidity chunk; used to calculate premia owed since last interaction |
| `s_settledTokens` | PanopticPool | `mapping(TokenId => mapping(uint256 => LeftRight))` | internal | Total settled premium tokens per chunk; ensures buyers pay sellers for removed liquidity |
| `marketState` | CollateralTracker | `MarketState` | internal | Packed type holding the global borrow index and last interest accrual timestamp; used to compound interest for all borrowers |
| `s_userInterestState` | CollateralTracker | `mapping(address => InterestState)` | internal | Per-user net borrows and snapshot of the global borrow index at last interaction; used to calculate interest owed |
| `s_poolAssets` | CollateralTracker | `uint128` | internal | Total USDC/token assets deposited by PLPs (not including deployed AMM assets or credited long positions) |
| `s_inAMM` | CollateralTracker | `uint128` | internal | Total token assets currently deployed in Uniswap via option seller positions; used for utilization calculations |
| `riskParameters` | RiskEngine | `RiskParameters` | internal | Packed type holding protocol-wide parameters: seller/buyer collateral ratios, commission fees, VEGOID, target utilization, min/max borrow rates |
| `oraclePack` | RiskEngine | `OraclePack` | internal | Packed oracle state: recent tick observations, EMA accumulators, and median ring buffer; used by `getSafePrice()` to return manipulation-resistant prices |
| `s_accountLiquidity` | SFPM | `mapping(bytes32 => LeftRight)` | internal | Per (account, pool, tokenId chunk): net liquidity added and total removed liquidity; the core accounting for long vs. short positions |
| `s_accountPremiumOwed` | SFPM | `mapping(bytes32 => LeftRight)` | internal | Accumulated fees per unit liquidity owed to option buyers from the Uniswap pool; updated on every mint/burn |
| `s_accountPremiumGross` | SFPM | `mapping(bytes32 => LeftRight)` | internal | Accumulated gross fees per unit liquidity for a chunk; used with `s_accountPremiumOwed` to compute the spread multiplier premia owed to sellers |

---

## Audit Methodology & Workflow

### Approach

This audit followed a structured, multi-phase process to maximize coverage within the allotted time. The methodology combined manual code review with automated tooling, with emphasis on manual review and proof-of-concept development for all confirmed findings.

### Phase 1 — Reconnaissance & Scoping

- Read all protocol documentation, README, and natspec comments
- Mapped the full contract architecture (see [Architecture Overview](#architecture-overview))
- Identified entry points, trust boundaries, and privileged roles
- Noted discrepancies between specification and implementation

### Phase 2 — Static Analysis

Automated tools were run to surface common vulnerability classes before manual review began.

| Tool | Purpose | Findings Surfaced |
| ---- | ------- | ----------------- |
| [Slither] | Static analyzer for common Solidity bugs | [e.g., 3 informational] |
| [Aderyn] | Rust-based static analyzer | [e.g., 1 low] |
| [Solhint] | Linter / code style | [e.g., 2 warnings] |

> All automated findings were manually triaged. Only confirmed, exploitable issues are included in this report.

### Phase 3 — Manual Review

Each in-scope contract was reviewed line-by-line with focus on:

- **Access control** — Are all state-changing functions properly gated?
- **Input validation** — Are inputs sanitized? Can edge cases cause unexpected behavior?
- **State transitions** — Are state changes atomic and consistent?
- **Business logic** — Does the implementation match the specification?
- **Trust assumptions** — Are privileged roles minimized and clearly documented?
- **Data visibility** — Is sensitive data exposed on-chain or through events?

### Phase 4 — Dynamic Testing & PoC Development

For each suspected vulnerability, a proof-of-concept test was written to confirm exploitability.

```bash
# Run the test suite
forge test

# Run fuzz tests with increased runs
forge test --fuzz-runs 10000

# Run a specific test
forge test --match-test test_[testName] -vvvv
```

All findings in this report include a passing PoC test or reproducible steps.

---

## Audit Details

**Commit Hash**
```
[29980a740b67f3e5d9df9d96264a246b51fc7b6b]
```

### Scope

```
contracts/
├── CollateralTracker.sol
├── PanopticFactoryV3.sol
├── PanopticFactoryV4.sol
├── PanopticPool.sol
├── RiskEngine.sol
├── SemiFungiblePositionManagerV3.sol
├── SemiFungiblePositionManagerV4.sol
├── base/
│   ├── FactoryNFT.sol
│   ├── MetadataStore.sol
│   └── Multicall.sol
├── tokens/
│   ├── ERC1155Minimal.sol
│   └── ERC20Minimal.sol
├── types/
│   ├── LeftRight.sol
│   ├── LiquidityChunk.sol
│   ├── MarketState.sol
│   ├── OraclePack.sol
│   ├── PositionBalance.sol
│   ├── RiskParameters.sol
│   └── TokenId.sol
└── libraries/
    ├── CallbackLib.sol
    ├── FeesCalc.sol
    ├── Math.sol
    ├── PanopticMath.sol
    ├── SafeTransferLib.sol
    └── TransientReentrancyGuard.sol
```

---

## Executive Summary

The Panoptic contract suite was reviewed for security vulnerabilities. The audit revealed [3] High-severity, [19] Medium, [11] Low/Informational findings.

### Issues Found

| Severity | Number of Issues |
| -------- | :--------------: |
| High | [3] |
| Medium | [19] |
| Low/Informational | [11] |
| **Total** | **[33]** |

---

## Findings

---

## High
