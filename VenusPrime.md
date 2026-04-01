# Venus Prime Audit

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

The Venus Prime system is an incentive mechanism built within the Venus Protocol that rewards long-term and high-value participants. Users stake XVS tokens to qualify for a non-transferable (soulbound) Prime token, which enables them to earn a share of protocol-generated revenue.

Rewards are distributed based on a score system that combines:

Stake (locked XVS)
Activity (borrow/supply usage in markets)

The system integrates multiple components such as:

Protocol income (via reserves)
Bootstrap liquidity (via PrimeLiquidityProvider)
External contracts (Comptroller, XVSVault)

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

The Venus Prime system consists of several interconnected modules responsible for eligibility, scoring, and reward distribution.

**1. Prime Token Mechanism**
Users must stake ≥ 1000 XVS for 90 days to mint a revocable Prime token
Tokens are soulbound (non-transferable)
Two types:
Revocable Token → burned if stake drops below threshold
Irrevocable Token (OG) → cannot be revoked (governance-controlled)

**2. Reward Distribution Model**

Rewards are distributed using a Cobb–Douglas-based scoring system, where:

Score depends on:
Stake (XVS locked)
Activity (borrow + supply)
Rewards are calculated via:
Global index (rewardIndex)
User score proportion

Mechanism:

Protocol generates income from lending markets
A portion is allocated to Prime users
Rewards are distributed per block

**3. Reward Sources**
 
(A) Protocol Share Reserve (PSR)
Collects protocol income (interest reserves)
Releases funds to Prime contract on demand
Ensures rewards availability during claims

(B) Prime Liquidity Provider
Provides bootstrap rewards
Emits tokens at a fixed per-block rate
Controlled by governance

**4. Core Integrations**

The Prime contract interacts with:

***Comptroller (PolicyFacet)***
→ Updates user score and interest on market actions
***XVSVault***
→ Tracks staking deposits and withdrawals
***Protocol Share Reserve (PSR)***
→ Supplies reward funds

These integrations ensure that:

User eligibility is tracked
Scores are updated dynamically
Rewards are distributed correctly

**5. Score Update Mechanism**
Score changes when users:
Stake / withdraw
Borrow / supply
Global parameter changes (like α or multipliers):
Cannot update all users at once (gas constraint)
Uses batch updates via updateScores(users[])

**6. Claiming Rewards**
Users accumulate rewards over time
On claim():
Rewards are calculated
Funds are pulled from PSR if needed
Claiming can be paused using a Pausable mechanism

### In-Scope Contracts
[Github](https://github.com/code-423n4/2023-09-venus)  
```
contracts/Tokens/Prime/Prime.sol ->  Souldbound token. Main contract of the feature. Prime holders will accrue rewards and they will be able to claim them when they want. 
contracts/Tokens/Prime/IPrime.sol -> Interface of the Prime contract
contracts/Tokens/Prime/PrimeStorage.sol -> Storage variables of the Prime contract
contracts/Tokens/Prime/PrimeLiquidityProvider.sol -> This contract provide liquidity to the Prime token, in an uniform way for a period of time. This liquidity is one of the two sources of funds supported by the Prime tokens.
contracts/Tokens/Prime/libs/Scores.sol -> Library with the function calculateScore, used to calculate the score associated with a Prime holder, that will finally define the rewards allocated to the user
contracts/Tokens/Prime/libs/FixedMath.sol -> Library with mathematical operations
contracts/Tokens/Prime/libs/FixedMath0x.sol -> Library with mathematical operations
```

### Contract Architecture

```
[2023-09-venus](https://github.com/code-423n4/2023-09-venus)
│
├── contracts/Tokens/Prime/Prime.sol
│   ← Soulbound token. Main contract of the feature. Prime holders accrue rewards and can claim them anytime.
│   ├── initialize()                  ← Initializes contract state and dependencies
│   ├── issue()                       ← Issues Prime tokens (revocable or irrevocable) to users
│   ├── burn()                        ← Burns Prime tokens if eligibility conditions are not met
│   ├── claimRewards()                ← Allows users to claim accrued rewards
│   ├── accrueInterest()              ← Accumulates rewards over time for users
│   ├── updateScores()                ← Updates user scores affecting reward distribution
│   └── isEligible()                  ← Checks whether a user satisfies staking requirements
│
├── contracts/Tokens/Prime/IPrime.sol
│   ← Interface of the Prime contract
│   ├── issue()                       ← Declares token issuance function
│   ├── claimRewards()                ← Declares reward claiming function
│   ├── burn()                        ← Declares token burning function
│   └── isEligible()                  ← Declares eligibility check function
│
├── contracts/Tokens/Prime/PrimeStorage.sol
│   ← Storage variables of the Prime contract
│   ├── tokens                        ← Stores user token data (existence, type, etc.)
│   ├── markets                       ← Stores market-related reward distribution data
│   ├── interests                     ← Tracks user scores and reward accumulation
│   ├── allMarkets                    ← List of all supported markets
│   └── governance                    ← Stores admin/governance-related configurations
│
├── contracts/Tokens/Prime/PrimeLiquidityProvider.sol
│   ← Provides liquidity to the Prime token uniformly over time (one of the reward sources)
│   ├── releaseFunds()                ← Releases liquidity gradually over a defined period
│   ├── getAvailableFunds()           ← Returns currently available liquidity
│   ├── fund()                        ← Allows adding funds to the liquidity pool
│   └── distribute()                  ← Sends funds to Prime contract for rewards
│
├── contracts/Tokens/Prime/libs/Scores.sol
│   ← Library for calculating user scores used in reward distribution
│   └── calculateScore()              ← Computes score based on stake, time, and other factors
│
├── contracts/Tokens/Prime/libs/FixedMath.sol
│   ← Library with mathematical operations
│   ├── mul()                         ← Fixed-point multiplication
│   ├── div()                         ← Fixed-point division
│   ├── add()                         ← Safe addition
│   └── sub()                         ← Safe subtraction
│
└── contracts/Tokens/Prime/libs/FixedMath0x.sol
    ← Library with mathematical operations (0x-style precision handling)
    ├── mulWad()                      ← Multiplication with WAD precision
    ├── divWad()                      ← Division with WAD precision
    ├── rpow()                        ← Exponentiation for fixed-point numbers
    └── sqrt()                        ← Square root calculation
```

### Data Flow

```
[User / XVS Staker]
        │
        │  stake / balance change
        ▼
[XVS Vault]
        │
        │  calls xvsUpdated(user)
        ▼
[Prime.sol] ──── updates ────► [PrimeStorage.sol (user score, eligibility)]
        │
        │  calls _updateScore()
        ▼
[Scores.sol] ──── calculates ────► [User Score]

------------------------------------------------------------

[User / Market Interaction]
        │
        │  supply / borrow
        ▼
[Comptroller]
        │
        │  triggers accrueInterest()
        ▼
[Prime.sol] ──── updates ────► [Interest Index / Rewards State]
        │
        │  calls _accrueInterestAndUpdateScore()
        ▼
[Scores.sol + FixedMath] ──── computes ────► [Updated Rewards]

------------------------------------------------------------

[Protocol Income / Liquidity Source]
        │
        │  releaseFunds()
        ▼
[PrimeLiquidityProvider.sol / PSR]
        │
        │  sends funds
        ▼
[Prime.sol] ──── stores ────► [Reward Pool]

------------------------------------------------------------

[User]
        │
        │  calls claim() / claimInterest()
        ▼
[Prime.sol]
        │
        │  calls _claimInterest()
        ▼
[PrimeStorage.sol] ──── reads ────► [Accrued Rewards]
        │
        │  transfers rewards
        ▼
[User Wallet]

------------------------------------------------------------

[Admin / Governance]
        │
        │  calls updateMultipliers() / addMarket() / setLimit()
        ▼
[Prime.sol] ──── updates ────► [Protocol Parameters]
```

### Trust Model & Roles

| Role | Address Type | Permissions |
| ---- | ------------ | ----------- |
| Owner / Governance | Multisig / Timelock | Can call `updateMultipliers()`, `addMarket()`, `setLimit()`, `togglePause()`, `updateAlpha()` |
| User | EOA | Can call `claim()`, `claimInterest()`, interact with markets affecting rewards |
| XVS Vault | Contract | Calls `xvsUpdated()` to update eligibility/score |
| Comptroller | Contract | Triggers `accrueInterest()` / `updateAssetsState()` |
| Liquidity Provider / PSR | Contract | Sends reward funds via `releaseFunds()` |
| Outsider | Any | Cannot mint/burn tokens or modify protocol parameters |

---

### Key State Variables

| Variable | Type | Visibility | Purpose |
| -------- | ---- | ---------- | ------- |
| `tokens` | `mapping(address => Token)` | internal | Stores user token state (exists, irrevocable status) |
| `markets` | `mapping(address => Market)` | internal | Stores market-level reward and score data |
| `interests` | `mapping(address => mapping(address => Interest))` | internal | Tracks user rewards and scores per market |
| `allMarkets` | `address[]` | public | List of all supported markets |
| `alphaNumerator` | `uint128` | public | Parameter for score calculation weighting |
| `alphaDenominator` | `uint128` | public | Parameter for score normalization |
| `irrevocableLimit` | `uint256` | public | Max number of irrevocable tokens |
| `revocableLimit` | `uint256` | public | Max number of revocable tokens |
| `paused` | `bool` | public | Controls pause state of critical functions |

---

### External Dependencies

| Dependency | Type | Used For |
| ---------- | ---- | -------- |
| `Scores.sol` | Library | Score calculation (Cobb-Douglas function) |
| `FixedMath.sol` | Library | Fixed-point arithmetic operations |
| `FixedMath0x.sol` | Library | Advanced precision math (WAD operations) |
| `XVS Vault` | Contract | Provides staking balance updates (`xvsUpdated`) |
| `Comptroller` | Contract | Market interaction and interest accrual |
| `PrimeLiquidityProvider` | Contract | Provides liquidity rewards over time |
| `Protocol Share Reserve (PSR)` | Contract | Additional reward funding source |
| `Resilient Oracle` | Oracle | Asset price feeds for APR and scoring |

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

---

*Report generated by [Srishti](https://SrishtiCode.io) · [April 2026]*
