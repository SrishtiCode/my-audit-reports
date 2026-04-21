# SukukFi Audit

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
The **WERC7575 system** is the blockchain settlement layer within a multi-tier telecom wholesale voice traffic settlement ecosystem. It works in conjunction with two off-chain platforms — **COMMTRADE** (smart contract engine) and **WRAPX** (settlement entity) — and carrier OSS/BSS systems to enable efficient, transparent, and gas-optimized settlement of inter-carrier voice traffic transactions.
 
The system is composed of two distinct but interdependent layers:
 
**1. Settlement Layer (WERC7575ShareToken + WERC7575Vault)**
The settlement layer is the operational core. Telecom carriers (e.g., Verizon Wholesale, AT&T Wholesale) hold settlement balances as ERC-20 share tokens (WUSD) backed by a USDC vault. Deposits are permissionless for KYC-verified carriers, while withdrawals require an off-chain permit issued by the WRAPX validator. The validator — acting as the trusted settlement platform — calls `batchTransfers()` to atomically net and settle inter-carrier obligations in a single on-chain transaction, reducing gas costs by up to 95%. The settlement layer is **non-upgradeable by design**, prioritizing stability and regulatory certainty.
 
**2. Investment Layer (ShareTokenUpgradeable + ERC7575VaultUpgradeable)**
The investment layer enables external investors to deploy capital into the settlement layer and earn yield from telecom traffic activity. Investors follow an asynchronous ERC-7540 Request → Fulfill → Claim flow, with an Investment Manager controlling fulfillment timing for capital efficiency. The Investment Manager deploys USDC into the WERC7575Vault, receiving WUSD shares on behalf of the ShareTokenUpgradeable contract. Yield is credited via `adjustrBalance()` adjustments. The investment layer is **upgradeable (UUPS)**, allowing regulatory adaptations and new investment strategies over time.
 
**Key Stakeholders:**
- **Telecom Carriers**: Fund wallets, receive/send settlement transfers
- **WRAPX (Validator)**: Signs batch settlements, issues withdrawal permits
- **COMMTRADE**: Tracks CDRs, calculates net positions, sends settlement instructions to WRAPX
- **Investors**: Deposit USDC, earn yield from telecom settlement activity
- **Investment Manager**: Deploys and redeems investor capital
**Underlying Asset:** USDC (stablecoin)
 
**Settlement Token:** WUSD (ERC-20 share token, non-transferable except via validator-controlled flows)
 
**Investment Token:** IUSD (ERC-7540/ERC-4626 share token)

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

The WERC7575 system is deployed as a four-tier architecture bridging legacy telecom infrastructure with a blockchain settlement layer:
 
- **Tier 1 — Carrier OSS/BSS Systems**: Generate Call Detail Records (CDRs) for every routed voice call and push them upstream.
- **Tier 2 — COMMTRADE Platform**: Ingests CDRs, enforces bilateral rate agreements, calculates net positions between carriers, and pushes individual settlement instructions to WRAPX.
- **Tier 3 — WRAPX Platform**: Receives settlement instructions, optimizes them into gas-efficient batches using a netting algorithm, issues withdrawal permits, and calls the on-chain `batchTransfers()` function as the trusted validator.
- **Tier 4 — WERC7575 Smart Contracts**: Execute atomic, on-chain settlement transfers between carrier wallets and maintain an immutable audit trail.
Sitting alongside the settlement layer is an **Investment Layer** (upgradeable) that allows third-party investors to provide working capital that is deployed into the settlement vault and earns yield from ongoing telecom traffic activity.

### Contract Architecture

```
WERC7575 System
│
├── WERC7575ShareToken.sol  (Settlement Layer — ERC-20 + ERC-2612)
│   ← Non-upgradeable share token representing carrier settlement balances (WUSD).
│   ← Permissionless deposits (KYC-verified carriers), permit-gated withdrawals.
│   ├── deposit()                  ← Carrier deposits USDC; mints WUSD shares
│   ├── transfer()                 ← Requires self-allowance permit from WRAPX validator
│   ├── transferFrom()             ← Dual-authorization: self-allowance (platform) + caller allowance (owner)
│   ├── batchTransfers()           ← Validator-only; atomically settles net positions across carriers
│   ├── permit()                   ← ERC-2612 permit; used by WRAPX to authorize carrier withdrawals
│   ├── adjustrBalance()           ← Adjusts investment contract's rBalance to reflect deployed capital & returns
│   └── mint() / burn()            ← Internal; controlled via deposit/withdraw flows
│
├── WERC7575Vault.sol  (Settlement Layer — ERC-4626-style Vault)
│   ← Non-upgradeable vault holding USDC collateral backing WUSD shares.
│   ├── deposit()                  ← Accepts USDC, mints WUSD to receiver (carrier or investment contract)
│   ├── withdraw()                 ← Burns WUSD, returns USDC (requires WRAPX permit)
│   ├── totalAssets()              ← Returns total USDC held in vault
│   └── convertToShares/Assets()  ← ERC-4626 share↔asset conversion functions
│
├── ShareTokenUpgradeable.sol  (Investment Layer — ERC-7540 + ERC-4626, UUPS)
│   ← Upgradeable vault for investor capital. Async deposit/redemption lifecycle.
│   ← Holds WUSD shares in the settlement layer on behalf of investors.
│   ├── requestDeposit()           ← Investor initiates deposit; assets transferred to vault
│   ├── fulfillDeposit()           ← Investment Manager converts pending assets to claimable shares
│   ├── deposit() / mint()         ← Investor claims IUSD shares after fulfillment
│   ├── requestRedeem()            ← Investor initiates redemption of IUSD shares
│   ├── fulfillRedeem()            ← Investment Manager fulfills redemption, withdraws from settlement
│   ├── withdraw() / redeem()      ← Investor claims USDC after redemption fulfilled
│   ├── investAssets()             ← Investment Manager deploys idle USDC into WERC7575Vault
│   └── totalAssets()              ← Returns value of WUSD shares held + pending amounts
│
└── ERC7575VaultUpgradeable.sol  (Investment Layer — Supporting Vault, UUPS)
    ← Upgradeable base vault logic for the investment layer.
    ├── initialize()               ← Sets up vault parameters and roles on deployment
    ├── _authorizeUpgrade()        ← UUPS upgrade guard (owner only)
    └── [ERC-4626 standard functions]
```


### Data Flow

```
[Telecom Carriers — OSS/BSS Systems]
        │
        │  Generate CDRs (Call Detail Records) per routed call
        ▼
[COMMTRADE Platform — Tier 2]
        │  • Ingests CDRs from all carriers
        │  • Enforces bilateral rate agreements
        │  • Aggregates transactions per settlement period
        │  • Calculates net positions (who owes whom, net amount)
        │  • Pushes individual settlement instructions to WRAPX
        ▼
[WRAPX Platform — Tier 3 / Validator]
        │  • Receives individual settlement instructions
        │  • Batches and applies netting algorithm
        │  • Validates carrier balances on-chain
        │  • Signs batch settlement transaction
        │  • Issues ERC-2612 permit signatures for authorized withdrawals
        ▼
[WERC7575ShareToken — On-Chain Settlement]
        │  • batchTransfers(debtors[], creditors[], amounts[]) called by WRAPX
        │  • Atomically debits/credits carrier WUSD balances
        │  • Immutable settlement record written to blockchain
        ▼
[Carrier Wallets — On-Chain Balances]
        │  • Net receivers: balance increases
        │  • Net payers: balance decreases
        │  • Withdrawals: carrier requests → WRAPX validates → permit issued → transfer()
 
─────────────────────────────────────────────────
 
[Investors]
        │
        │  1. requestDeposit(USDC) → assets held in ShareTokenUpgradeable
        ▼
[ShareTokenUpgradeable — Investment Layer]
        │  2. Investment Manager calls fulfillDeposit() when deal is available
        │  3. Investor claims IUSD shares via deposit()
        │
        │  4. Investment Manager calls investAssets(amount)
        ▼
[WERC7575Vault — Settlement Layer]
        │  • Mints WUSD shares to ShareTokenUpgradeable
        │  • USDC deployed as working capital for carrier deals
        ▼
[Telecom Traffic Activity — Yield Generation]
        │  • Settlement fees, voice traffic margins, prepayment interest
        │
        │  5. WRAPX calls adjustrBalance(ShareTokenUpgradeable, invested, returned)
        ▼
[ShareTokenUpgradeable — rBalance Updated]
        │  • WUSD position value increases (e.g., 100k → 110k)
        │  • IUSD share price appreciates proportionally
        │
        │  6. Investor calls requestRedeem() → fulfillRedeem() → redeem()
        ▼
[Investor Wallet]
        └── Receives USDC + profit
```

### Trust Model & Roles

| Role | Address Type | Permissions |
| ---- | ------------ | ----------- |
| **WRAPX Validator** | Trusted Off-Chain Platform (EOA or Multisig) | Calls `batchTransfers()` on settlement layer; issues ERC-2612 permit signatures authorizing carrier withdrawals; calls `adjustrBalance()` to credit yield; KYC enforcement |
| **Telecom Carrier** | KYC-verified EOA or Contract Wallet | Calls `deposit()` to fund settlement wallet (permissionless post-KYC); calls `transfer()` / `transferFrom()` to withdraw (requires WRAPX permit); cannot settle without validator |
| **Investment Manager** | Trusted EOA or Multisig | Calls `fulfillDeposit()`, `fulfillRedeem()`, `investAssets()` on investment layer; controls timing of capital deployment and redemption |
| **Investor** | EOA | Calls `requestDeposit()`, `deposit()` (after fulfillment), `requestRedeem()`, `redeem()` (after fulfillment); cannot force immediate withdrawal |
| **COMMTRADE Platform** | Off-Chain System | Sends settlement instructions to WRAPX; no direct on-chain permissions |
| **Contract Owner** | Multisig / Admin | Admin functions on upgradeable contracts (investment layer only); `_authorizeUpgrade()` for UUPS upgrades |
| **Outsider / Unauthenticated** | Any | Cannot deposit (no KYC), cannot call `batchTransfers()`, cannot issue permits; effectively excluded from all privileged operations |

---

### Key State Variables

| Variable | Contract | Type | Visibility | Purpose |
| -------- | -------- | ---- | ---------- | ------- |
| `_balances` | WERC7575ShareToken | `mapping(address => uint256)` | internal | Tracks each carrier's (and investment contract's) available WUSD share balance |
| `_rBalances` | WERC7575ShareToken | `mapping(address => uint256)` | internal | Tracks the investment contract's deployed capital in active deals (not available for withdrawal); used to calculate yield; carriers do NOT have rBalance entries |
| `_allowances` | WERC7575ShareToken | `mapping(address => mapping(address => uint256))` | internal | Standard ERC-20 allowances; self-allowance (`allowance[x][x]`) is repurposed as the WRAPX-issued withdrawal permit; caller allowance enables third-party delegations |
| `validator` | WERC7575ShareToken | `address` | public | Address of the WRAPX validator; only this address may call `batchTransfers()` and `adjustrBalance()` |
| `pendingDeposits` | ShareTokenUpgradeable | `mapping(address => uint256)` | internal | Tracks USDC amounts requested by investors awaiting Investment Manager fulfillment (Request phase of ERC-7540) |
| `claimableShares` | ShareTokenUpgradeable | `mapping(address => uint256)` | internal | Tracks IUSD shares available for investors to claim after fulfillment (Claim phase of ERC-7540) |
| `pendingRedemptions` | ShareTokenUpgradeable | `mapping(address => uint256)` | internal | Tracks IUSD shares submitted for redemption awaiting Investment Manager fulfillment |
| `claimableAssets` | ShareTokenUpgradeable | `mapping(address => uint256)` | internal | Tracks USDC amounts available for investors to claim after redemption fulfillment |
| `investmentVault` | ShareTokenUpgradeable | `address` | internal | Address of the WERC7575Vault into which the Investment Manager deploys capital |
| `shareToken` | ShareTokenUpgradeable | `address` | internal | Address of the ShareTokenUpgradeable itself, used as the receiver of WUSD shares when calling `investAssets()` |
| `totalInvested` | ShareTokenUpgradeable | `uint256` | internal | Cumulative USDC deployed into the settlement vault; used alongside rBalance to calculate net yield |

---


## Audit Methodology & Workflow

### Approach

This audit followed a structured, multi-phase process to maximize coverage within the allotted time. The methodology combined manual code review with automated tooling, with emphasis on [manual review / fuzz testing / formal verification — pick what applies].

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


## Audit Details

**Commit Hash**
```
[18fe2578cf1c6203dac7ff21513533010f3dda3e]
```

### Scope

```
./
└── src/DecimalConstants.sol
└── src/ERC7575VaultUpgradeable.sol
└── src/SafeTokenTransfers.sol
└── src/ShareTokenUpgradeable.sol
└── src/WERC7575ShareToken.sol
└── src/WERC7575Vault.sol

```
---

## Executive Summary

The SukukFi contract was reviewed for security vulnerabilities. The audit revealed [1] High-severity, [3] Medium, [9] Low/Informational findings.


### Issues Found

| Severity | Number of Issues |
| -------- | :--------------: |
| High | [1] |
| Medium | [3] |
| Low/Informational | [9] |
| **Total** | **[13]** |

---
## Findings

---

## High
