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


## Audit Details

**Commit Hash**
```
[9dcc9c72514da5fb99a4b66d93938372209e85de]
```

### Scope

```
./contracts/Tokens/Prime/
└── Prime.sol
└── IPrime.sol
└── PrimeStorage.sol
└── PrimeLiquidityProvider.sol
└── libs/Scores.sol
└── libs/FixedMath.sol

```
---

## Executive Summary

The Venus Prime contract was reviewed for security vulnerabilities. The audit revealed [3] High-severity, [3] Medium, [7] Low, and [7] Informational findings.


### Issues Found

| Severity | Number of Issues |
| -------- | :--------------: |
| High | [3] |
| Medium | [3] |
| Low/Informational | [8] |
| **Total** | **[14]** |

---
## Findings

---

## High

### [H-1] [Irrevocable Token Issuance Does Not Reset `stakedAt`, Allowing Claim Without Active Stake]

**Severity:** `HIGH`
**Impact:** HIGH · **Likelihood:** HIGH

#### Description

Whenever a new Prime token is created, the users `stakedAt` is reset to 0. This happens when the user claim a revocable token and when he is `issue` a revocable token, but it does not happen when a user is `issue` an irrevocable token.

```javascript
    //If we want to issue the irrevocable token(cannot be revoke by admin or anyone else) then -> check if the token exist and 
    //and revocable and upgrade it else mint the irrevocable token and initialize market. And if we want to issue revocable 
    //token then mint revocable token and initialize market and reset staking time.          

    function issue(bool isIrrevocable, address[] calldata users) external {
        _checkAccessAllowed("issue(bool,address[])");

        if (isIrrevocable) {
            for (uint256 i = 0; i < users.length; ) {
                Token storage userToken = tokens[users[i]];//loop through the each user
                if (userToken.exists && !userToken.isIrrevocable) {//if the user has a token and it is revocable
                    _upgrade(users[i]);//upgrade it
                } else {
                    _mint(true, users[i]);
                    _initializeMarkets(users[i]);//initialize associated market
                    // @audit - here `stakedAt` is not being reset
                }

                unchecked {
                    i++;
                }
            }
        } else {//if revocable
            for (uint256 i = 0; i < users.length; ) {
                _mint(false, users[i]);
                _initializeMarkets(users[i]);
                delete stakedAt[users[i]];

                unchecked {
                    i++;
                }
            }
        }
    }
```

#### Impact

A user can claim a Prime token without having any XVS staked.

Due to `stakedAt` not being reset when an irrevocable token is issued, a user can later satisfy the staking time condition even after withdrawing all funds. This breaks the core requirement that users must actively stake XVS to claim a Prime token.

As a result, users can gain Prime token benefits without locking any tokens, leading to incorrect reward distribution and potential loss of protocol funds.

#### Proof of Concept

**Step-by-step reproduction:**

**1. Stake XVS to initialize `stakedAt`:**
```bash
Alice stakes ≥1000 XVS
```

**2. Issue an irrevocable Prime token (stakedAt NOT reset):**
```bash
Admin calls: issue(true, [Alice])
```

**3. Withdraw all staked XVS:**
```bash
Alice withdraws all XVS (stake becomes 0)
```

**4. Burn the irrevocable token:**
```bash
Admin calls: burn(Alice)

```
**5. Wait for staking period and call claim without staking:**
```bash
Alice calls: claim()
```

**Expected output:**
```
[Claim succeeds and a Prime token is minted even though Alice has 0 XVS staked.]
```

*Or, add the following test to `Test.t.sol`:*

```javascript
function test_claimWithoutStake_dueToIrrevocableIssue() public {
    address alice = address(1);

    // Step 1: Alice stakes
    vm.prank(alice);
    prime.stake{value: 10 ether}();

    uint256 initialStakedAt = prime.stakedAt(alice);
    assertTrue(initialStakedAt != 0);

    // Step 2: Admin issues irrevocable token (no reset)
    address;
    users[0] = alice;
    prime.issue(true, users);

    // stakedAt remains unchanged
    assertEq(prime.stakedAt(alice), initialStakedAt);

    // Step 3: Alice withdraws all stake
    vm.prank(alice);
    prime.withdrawAll();

    assertEq(prime.balanceOfStake(alice), 0);

    // Step 4: Admin burns token
    prime.burn(alice);

    // Step 5: Fast forward time
    vm.warp(block.timestamp + prime.STAKING_PERIOD());

    // Step 6: Alice claims without staking
    vm.prank(alice);
    prime.claim();

    // Unexpected behavior: claim succeeds
    assertTrue(prime.tokens(alice).exists);
}

```

#### Recommended Mitigation

Reset the user's stakedAt whenever he is issued an irrevocable token.

```diff
function issue(bool isIrrevocable, address[] calldata users) external {
        _checkAccessAllowed("issue(bool,address[])");

        if (isIrrevocable) {
            for (uint256 i = 0; i < users.length; ) {
                Token storage userToken = tokens[users[i]];
                if (userToken.exists && !userToken.isIrrevocable) {
                    _upgrade(users[i]);
                } else {
                    _mint(true, users[i]);
                    _initializeMarkets(users[i]);
+                    delete stakedAt[users[i]]; 
                }
          ...
```
---

### [H-2] [Manipulation of `pendingScoreUpdates` Allows Attacker to Freeze Score and Gain Excess Rewards]

**Severity:** `HIGH`
**Impact:** HIGH · **Likelihood:** HIGH

#### Description

The protocol relies on `pendingScoreUpdates` to track how many users still need score updates after changes to parameters like `alpha` or multipliers. However, this variable can be **incorrectly reduced when a user’s token is burned**, even if their score was never updated.  

Additionally, minting a new token via `claim()` does **not increase `pendingScoreUpdates`**, creating an imbalance. An attacker can exploit this by minting and burning tokens to artificially reduce `pendingScoreUpdates` to zero, preventing further score updates and allowing their own score to remain outdated and inflated.

```javascript
function _updateRoundAfterTokenBurned(address user) internal { 
    // @audit - decreases pendingScoreUpdates without actual score update
    if (pendingScoreUpdates > 0 && !isScoreUpdated[nextScoreUpdateRoundId][user]) {
        pendingScoreUpdates--;
    }
}

function claim() external {
    // @audit - minting does not increase pendingScoreUpdates
    _mint(false, msg.sender);
}
```

#### Impact

An attacker can avoid unfavorable score updates and retain an artificially higher score, allowing them to earn disproportionately higher rewards at the expense of other users.

#### Proof of Concept

**Step-by-step reproduction:**

**1. Trigger score update requirement:**
```bash
Call updateAlpha() or updateMultipliers()
```

**2. Manipulate pendingScoreUpdates:**
```bash
Attacker mints a token (claim)
Attacker burns token (withdraw)
Repeat to reduce pendingScoreUpdates
```

**3. Exhaust updates:**
```bash
Call updateScores() on other users until pendingScoreUpdates == 0
```


**Expected output:**
```
Further calls to updateScores() revert with NoScoreUpdatesRequired,
leaving attacker’s score unupdated (higher than intended).
```

*Or, add the following test to `Test.t.sol`:*

```javascript
function test_freezeAttackerScore() public {
    // Setup users and staking...

    // Step 1: Change alpha → requires updates
    prime.updateAlpha(1, 5);

    // Step 2: Attacker manipulates counter
    vm.prank(attacker1);
    prime.claim();
    vm.prank(attacker1);
    xvsVault.requestWithdrawal(...);

    vm.prank(attacker2);
    prime.claim();
    vm.prank(attacker2);
    xvsVault.requestWithdrawal(...);

    // Step 3: Update other users
    prime.updateScores([user3]);

    // Step 4: Further updates blocked
    vm.expectRevert();
    prime.updateScores([attacker1, attacker2]);

    // Attacker score remains unupdated
}
```

#### Recommended Mitigation

Ensure pendingScoreUpdates cannot be manipulated via mint/burn imbalance.
Fix: Increment `pendingScoreUpdates` on mint if updates are pending

```diff
function _mint(bool isIrrevocable, address user) internal {
    if (tokens[user].exists) revert IneligibleToClaim();

    tokens[user].exists = true;
    tokens[user].isIrrevocable = isIrrevocable;

    if (isIrrevocable) {
        totalIrrevocable++;
    } else {
        totalRevocable++;
    }

    if (totalIrrevocable > irrevocableLimit || totalRevocable > revocableLimit) revert InvalidLimit();

+   if (pendingScoreUpdates != 0) {
+       unchecked { ++pendingScoreUpdates; }
+   }

    emit Mint(user, isIrrevocable);
}
```
Additionally:

Avoid decreasing pendingScoreUpdates unless an actual score update occurs
Ensure mint/burn operations do not affect update accounting

---

### [H-3] [Incorrect Decimal Normalization Causes Zero or Miscalculated Rewards]

**Severity:** `HIGH`
**Impact:** HIGH · **Likelihood:** HIGH

#### Description

The `Prime._calculateScore` function incorrectly normalizes `capital` using `vToken.decimals()` instead of the underlying token’s decimals. Since `capital` is derived from underlying assets, this results in inflated precision (e.g., 28 decimals instead of 18). This incorrect scaling propagates into `sumOfMembersScore`, causing the `delta` calculation in `Prime.accrueInterest` to become extremely small or zero. As a result, `rewardIndex` may not increase, preventing users from accruing rewards.

```javascript
function _calculateScore(...) internal view returns (uint256) {
    (uint256 capital, , ) = _capitalForScore(...);

    // @audit - Incorrect decimals used; should use underlying token decimals instead of vToken.decimals()
    capital = capital * (10 ** (18 - vToken.decimals()));

    return Scores.calculateScore(..., capital, ...);
}
```

#### Impact

Users may receive zero or significantly reduced rewards despite valid participation, as rewardIndex fails to increase due to inflated sumOfMembersScore, effectively halting reward distribution.

#### Proof of Concept

**Step-by-step reproduction:**

**1. Simulate a market with mismatched decimals (e.g., vToken = 8 decimals, underlying = 18):**
```bash
Setup vToken.decimals() = 8
Underlying.decimals() = 18
```

**2. Call `_calculateScore` and `accrueInterest`:**
```bash
capital scaled = capital * 10^(18 - 8) = capital * 1e10
sumOfMembersScore becomes ~1e28
```

**Expected output:**
```
delta = (distributionIncome * 1e18) / 1e28 = 0
rewardIndex does not increase
users receive 0 rewards
```

*Or, add the following test to `Test.t.sol`:*

```javascript
function test_rewardIndexNotIncreasing_dueToWrongDecimals() public {
    uint256 distributionIncome = 1e18;

    // Simulate correct vs incorrect scaling
    uint256 correctSum = 1e18;
    uint256 wrongSum = 1e28;

    uint256 correctDelta = (distributionIncome * 1e18) / correctSum;
    uint256 wrongDelta = (distributionIncome * 1e18) / wrongSum;

    // Correct behavior
    assertGt(correctDelta, 0);

    // Buggy behavior
    assertEq(wrongDelta, 0);
}
```

#### Recommended Mitigation

Use the underlying token decimals instead of vToken.decimals() when normalizing capital.

```diff
function _calculateScore(...) internal view returns (uint256) {
    (uint256 capital, , ) = _capitalForScore(xvsBalanceForScore, borrow, supply, market);

-   capital = capital * (10 ** (18 - vToken.decimals()));
+   capital = capital * (10 ** (18 - IERC20Upgradeable(_getUnderlying(market)).decimals()));

    return Scores.calculateScore(..., capital, ...);
}
```

---

## Medium

### [M-1] [Scores.sol : Incorrect computation of user's score when alpha is 1]

**Severity:** `MEDIUM`
**Impact:** MEDIUM · **Likelihood:** MEDIUM

#### Description

The formula for a user’s score depends on the xvs staked and the capital. One core variable in the calculation of a user’s score is alpha which represents the weight for xvs and capital. It has been stated in the documentation that alpha can range from 0 to 1 depending on what kind of incentive the protocol wants to drive (XVS Staking or supply/borrow). Please review Significance of α subsection.

When alpha is 1, xvs has all the weight when calculating a user’s score, and capital has no weight. If we see the Cobb-Douglas function, the value of capital doesn’t matter, it will always return 1 since capital^0 is always 1. So, a user does not need to borrow or lend in a given market since capital does not have any weight in the score calculation.

The issue is an inconsistency in the implementation of the Cobb-Douglas function.

Developers have added an exception: if (xvs == 0 || capital == 0) return 0;
Because of this the code will always return 0 if either the xvs or the capital is zero, but we know this should not be the case when the alpha value is 1.

#### Proof of Concept

To check how it works:

In describe('boosted yield') add the following code:

```bash
it.only("calculate score only staking", async () => {
      const xvsBalance = bigNumber18.mul(5000);
      const capital = bigNumber18.mul(0);

      await prime.updateAlpha(1, 1); // 1

      //  5000^1 * 0^0 = 5000
      // BUT IS RETURNING 0!!!
      expect((await prime.calculateScore(xvsBalance, capital)).toString()).to.be.equal("0");
});
```

#### Recommended Mitigation

Only return 0 when (xvs = 0 or capital = 0) * and alpha is not 1.
---

### [M-2] [DoS and Gas Griefing of Calls to `Prime.updateScores()`]

**Severity:** `MEDIUM`
**Impact:** MEDIUM · **Likelihood:** MEDIUM

#### Description

`updateScores()` is meant to be called to update the scores of many users after reward alpha is changed or reward multipliers are changed. An attacker can cause calls to `Prime.updateScores()` to out-of-gas revert, delaying score updates. Rewards will be distributed incorrectly until scores are properly updated.

The root cause is a misplaced `i++` increment combined with a `continue` statement. When a user has already been updated, `continue` is hit before `i` is incremented, causing the loop to re-process the same index indefinitely until the transaction runs out of gas:


```javascript
function updateScores(address[] memory users) external {
    if (pendingScoreUpdates == 0) revert NoScoreUpdatesRequired();
    if (nextScoreUpdateRoundId == 0) revert NoScoreUpdatesRequired();

    for (uint256 i = 0; i < users.length; ) {
        address user = users[i];

        if (!tokens[user].exists) revert UserHasNoPrimeToken();
        if (isScoreUpdated[nextScoreUpdateRoundId][user]) continue; // i never increments
        ...
        unchecked {
            i++; // increment is never reached if `continue` is hit
        }

        emit UserScoreUpdated(user);
    }
}
```

#### Impact

An attacker can frontrun any call to `updateScores()` by pre-updating a single address that appears in the target call's argument array. When the frontrun call is then executed, the loop gets stuck infinitely on that already-updated address and reverts out of gas. Only one user gets updated per griefing attempt, while the legitimate batch update fails entirely. Rewards will be distributed based on stale, incorrect scores until all users are eventually updated.

#### Proof of Concept

An attacker frontruns `updateScores([user1, user2, user3])` with `updateScores([user3])`. When the original call executes and reaches `user3`, `isScoreUpdated` is already `true`, `continue` is hit, `i` is never incremented, and the transaction loops on `user3` until it runs out of gas and reverts.

Paste and run the following test in the `'mint and burn'` scenario in `Prime.ts` (line 302):

```javascript
it("dos_updateScores", async () => {
    // Setup 3 users
    await prime.issue(true, [
        user1.getAddress(),
        user2.getAddress(),
        user3.getAddress()
    ]);

    await xvs.connect(user1).approve(xvsVault.address, bigNumber18.mul(1000));
    await xvsVault.connect(user1).deposit(xvs.address, 0, bigNumber18.mul(1000));

    await xvs.connect(user2).approve(xvsVault.address, bigNumber18.mul(1000));
    await xvsVault.connect(user2).deposit(xvs.address, 0, bigNumber18.mul(1000));

    await xvs.connect(user3).approve(xvsVault.address, bigNumber18.mul(1000));
    await xvsVault.connect(user3).deposit(xvs.address, 0, bigNumber18.mul(1000));

    // Change alpha to trigger score update round
    await prime.updateAlpha(1, 5);

    // Attacker frontruns the batch update with a single-user call for user3
    await prime.connect(user1).updateScores([user3.getAddress()]);

    // Original batch call now reverts due to infinite loop on user3
    await expect(
        prime.updateScores([
            user1.getAddress(),
            user2.getAddress(),
            user3.getAddress()
        ])
    ).to.be.reverted;
});
```

#### Recommended Mitigation

Move the increment to the for loop header so that continue no longer prevents i from advancing:

```diff
-   for (uint256 i = 0; i < users.length; ) {
+   for (uint256 i = 0; i < users.length; ++i) {
        address user = users[i];

        if (!tokens[user].exists) revert UserHasNoPrimeToken();
        if (isScoreUpdated[nextScoreUpdateRoundId][user]) continue;
        ...
-       unchecked {
-           i++;
-       }

        emit UserScoreUpdated(user);
    }
```

This ensures that already-updated users are simply skipped rather than causing an infinite loop, making the function resilient to griefing.

---

### [M-3] [The `owner` is a Single Point of Failure and a Centralization Risk]

**Severity:** `MEDIUM`
**Impact:** MEDIUM · **Likelihood:** MEDIUM

#### Description

Having a single EOA as the only owner of contracts is a large centralization risk and a single point of failure. A single private key may be taken in a hack, or the sole holder of the key may become unable to retrieve the key when necessary. Critical protocol functions gated behind `onlyOwner` would then be permanently inaccessible or, worse, fall under the control of a malicious actor.

There are 3 instances of this issue in `PrimeLiquidityProvider.sol`:

```javascript
File: contracts/Tokens/Prime/PrimeLiquidityProvider.sol

118:    function initializeTokens(address[] calldata tokens_) external onlyOwner {

177:    function setPrimeToken(address prime_) external onlyOwner {

216:    function sweepToken(
            IERC20Upgradeable token_,
            address to_,
            uint256 amount_
        ) external onlyOwner {
```

#### Impact

If the owner's private key is compromised or lost, an attacker or no one at all would have exclusive control over the following critical operations:

- `initializeTokens()` — ability to add supported tokens to the liquidity provider, blocking protocol bootstrapping.
- `setPrimeToken()` — ability to set the Prime token address, a one-time configuration step essential for the protocol to function.
- `sweepToken()` — ability to recover tokens from the contract, potentially draining all funds if the key is stolen.

#### Proof of Concept

All three functions are gated behind the same single-owner check with no fallback or recovery mechanism:

```javascript
modifier onlyOwner() {
    require(owner() == msg.sender, "Ownable: caller is not the owner");
    _;
}
```

A single compromised or lost key grants or removes access to all three functions simultaneously, with no recourse.

#### Recommended Mitigation

Consider one or more of the following approaches:

1. Use a multi-signature wallet (e.g. Gnosis Safe) as the owner address, requiring M-of-N signers to authorize sensitive calls.
2. Adopt a role-based access control model using OpenZeppelin's AccessControl, separating privileges so no single key controls all critical functions:

```diff
- function initializeTokens(address[] calldata tokens_) external onlyOwner {
+ function initializeTokens(address[] calldata tokens_) external onlyRole(TOKEN_ADMIN_ROLE) {

- function setPrimeToken(address prime_) external onlyOwner {
+ function setPrimeToken(address prime_) external onlyRole(CONFIG_ROLE) {

- function sweepToken(...) external onlyOwner {
+ function sweepToken(...) external onlyRole(SWEEP_ROLE) {
```

3. Implement a timelock controller so that sensitive owner actions have a mandatory delay, giving users time to react to malicious or erroneous changes before they take effect.

---

## LOW

### [L-1] [`Prime.sol`'s `_claimInterest()` Should Apply Slippage or Allow Users to Partially Claim Their Interest]

**Severity:** `LOW`
**Impact:** LOW · **Likelihood:** LOW

#### Description

The `_claimInterest` function hardcodes a 0% slippage tolerance and does not perform a final balance check after attempting to release funds from its two reserve sources. If the contract's balance remains insufficient after both release attempts, the subsequent `safeTransfer` call will revert unconditionally, preventing users from claiming any of their accrued interest.

```javascript
function _claimInterest(address vToken, address user) internal returns (uint256) {
    uint256 amount = getInterestAccrued(vToken, user);
    amount += interests[vToken][user].accrued;

    interests[vToken][user].rewardIndex = markets[vToken].rewardIndex;
    interests[vToken][user].accrued = 0;

    address underlying = _getUnderlying(vToken);
    IERC20Upgradeable asset = IERC20Upgradeable(underlying);

    if (amount > asset.balanceOf(address(this))) {
        address[] memory assets = new address[](1);
        assets[0] = address(asset);
        IProtocolShareReserve(protocolShareReserve).releaseFunds(comptroller, assets);
        if (amount > asset.balanceOf(address(this))) {
            IPrimeLiquidityProvider(primeLiquidityProvider).releaseFunds(address(asset));
            // @audit - no final check; if balance is still insufficient,
            // safeTransfer will always revert and user receives nothing
            unreleasedPLPIncome[underlying] = 0;
        }
    }

    asset.safeTransfer(user, amount); // reverts if balance < amount

    emit InterestClaimed(user, vToken, amount);
    return amount;
}
```
The contract attempts to top up its balance in two steps:

1. Release funds from `IProtocolShareReserve`.
2. If still insufficient, release funds from `IPrimeLiquidityProvider`.

However, there is no final guard before `safeTransfer` to confirm the balance is now sufficient. If both releases fail to cover the full `amount`, the transfer reverts and the user walks away with nothing, even if a partial payment was possible.

#### Impact

If both reserve sources are temporarily underfunded, users are completely blocked from claiming any accrued interest. This leads to user frustration, erodes trust in the platform, and effectively locks user funds until the contract happens to accumulate sufficient balance — with no mechanism to notify users or queue the deficit for later.

#### Proof of Concept

Consider the following scenario:

1. A user accrues 1000 USDC in Prime interest.
2. _claimInterest() is called — the contract balance is 200 USDC.
3. releaseFunds() from ProtocolShareReserve adds 300 USDC → balance is now 500 USDC.
4. releaseFunds() from PrimeLiquidityProvider adds 400 USDC → balance is now 900 USDC.
5. safeTransfer(user, 1000) is called — reverts because 900 < 1000.
6. The user receives nothing, despite 900 USDC being available and their accrued balance having already been reset to 0.

#### Recommended Mitigation

Add a final balance check after both release attempts. Transfer only what is available, and store any deficit so it can be claimed later once the contract has sufficient funds:

```diff
function _claimInterest(address vToken, address user) internal returns (uint256) {
    ...
    if (amount > asset.balanceOf(address(this))) {
        address[] memory assets = new address[](1);
        assets[0] = address(asset);
        IProtocolShareReserve(protocolShareReserve).releaseFunds(comptroller, assets);
        if (amount > asset.balanceOf(address(this))) {
            IPrimeLiquidityProvider(primeLiquidityProvider).releaseFunds(address(asset));
            unreleasedPLPIncome[underlying] = 0;
        }
    }

+   uint256 availableBalance = asset.balanceOf(address(this));
+   uint256 transferAmount = (amount <= availableBalance) ? amount : availableBalance;

+   // Store any deficit so the user can claim the remainder later
+   if (transferAmount < amount) {
+       interests[vToken][user].accrued += amount - transferAmount;
+   }

-   asset.safeTransfer(user, amount);
+   asset.safeTransfer(user, transferAmount);

-   emit InterestClaimed(user, vToken, amount);
-   return amount;
+   emit InterestClaimed(user, vToken, transferAmount);
+   return transferAmount;
}
```

This ensures users always receive whatever is currently available rather than nothing, and any remaining deficit is preserved in accrued for a future claim rather than silently lost.

---

### [L-2][`getEffectiveDistributionSpeed()` Should Prevent Reversion and Return 0 if Accrued Exceeds Balance]

**Severity:** `LOW`
**Impact:** LOW · **Likelihood:** LOW

#### Description

The `getEffectiveDistributionSpeed()` function assumes that `balance >= accrued` is always true, but this invariant can be broken. The `sweepToken()` function allows the owner to withdraw tokens from the contract without updating `tokenAmountAccrued`, meaning the accrued amount can exceed the actual balance. When this happens, the subtraction `balance - accrued` underflows and the function reverts.

```javascript
function getEffectiveDistributionSpeed(address token_) external view returns (uint256) {
    uint256 distributionSpeed = tokenDistributionSpeeds[token_];
    uint256 balance = IERC20Upgradeable(token_).balanceOf(address(this));
    uint256 accrued = tokenAmountAccrued[token_];

    if (balance - accrued > 0) { // underflows if accrued > balance
        return distributionSpeed;
    }

    return 0;
}
```
Contrast `sweepToken()` with `releaseFunds()` — the former transfers tokens without resetting `tokenAmountAccrued`, while the latter correctly zeroes it out:

```javascript
// sweepToken - does NOT update tokenAmountAccrued
function sweepToken(IERC20Upgradeable token_, address to_, uint256 amount_) external onlyOwner {
    uint256 balance = token_.balanceOf(address(this));
    if (amount_ > balance) revert InsufficientBalance(amount_, balance);
    token_.safeTransfer(to_, amount_);
}

// releaseFunds - correctly resets tokenAmountAccrued
function releaseFunds(address token_) external {
    accrueTokens(token_);
    uint256 accruedAmount = tokenAmountAccrued[token_];
    tokenAmountAccrued[token_] = 0; // ✅ reset
    IERC20Upgradeable(token_).safeTransfer(prime, accruedAmount);
}
```

#### Impact

After a `sweepToken()` call that reduces the contract balance below the accrued amount, `getEffectiveDistributionSpeed()` will revert on every call for that token due to an arithmetic underflow. Any off-chain or on-chain caller relying on this view function to check distribution speeds will be broken until the balance is replenished.

#### Proof of Concept

Consider the following scenario:

1. Contract holds 500 USDC; `tokenAmountAccrued[USDC] = 300`.
2. Owner calls `sweepToken(USDC, owner, 400)` — balance drops to 100 USDC, accrued remains 300.
3. `getEffectiveDistributionSpeed(USDC)` is called — `balance - accrued = 100 - 300` underflows and reverts.

#### Recommended Mitigation

Replace the unsafe subtraction with an explicit comparison:

```diff
function getEffectiveDistributionSpeed(address token_) external view returns (uint256) {
    uint256 distributionSpeed = tokenDistributionSpeeds[token_];
    uint256 balance = IERC20Upgradeable(token_).balanceOf(address(this));
    uint256 accrued = tokenAmountAccrued[token_];

-   if (balance - accrued > 0) {
+   if (balance > accrued) {
        return distributionSpeed;
    }

    return 0;
}
```

This single-character change eliminates the underflow risk and correctly returns `0` whenever accrued meets or exceeds the available balance.

---

### [L-3][Inconsistency in Parameter Validations Between Sister Functions]

**Severity:** `LOW`
**Impact:** LOW · **Likelihood:** LOW

#### Description

The `updateAlpha()` function validates its inputs through `_checkAlphaArguments()`, which ensures the denominator is non-zero and the numerator does not exceed the denominator. However, the logically analogous `updateMultipliers()` function has no equivalent validation for its `supplyMultiplier` and `borrowMultiplier` parameters. This inconsistency means invalid or illogical multiplier values can be set without any on-chain revert.

```javascript
function _checkAlphaArguments(uint128 _alphaNumerator, uint128 _alphaDenominator) internal {
    if (_alphaDenominator == 0 || _alphaNumerator > _alphaDenominator) {
        revert InvalidAlphaArguments();
    }
}

// updateAlpha validates inputs ✅
function updateAlpha(uint128 _alphaNumerator, uint128 _alphaDenominator) external {
    _checkAlphaArguments(_alphaNumerator, _alphaDenominator);
    ...
}

// updateMultipliers has no equivalent validation ❌
function updateMultipliers(address market, uint256 supplyMultiplier, uint256 borrowMultiplier) external {
    if (!markets[market].exists) revert MarketNotSupported();
    ...
    markets[market].supplyMultiplier = supplyMultiplier;
    markets[market].borrowMultiplier = borrowMultiplier;
    _startScoreUpdateRound();
}
```

#### Impact

Without validation, zero or otherwise illogical multiplier values can be written to a market's configuration. This could result in incorrect reward calculations or unintended score update rounds being triggered with invalid state.


#### Recommended Mitigation

Introduce a `_checkMultipliersArguments()` validation function and apply it inside `updateMultipliers()`, mirroring the pattern already established by `updateAlpha()`:

```diff
+ function _checkMultipliersArguments(uint256 supplyMultiplier, uint256 borrowMultiplier) internal pure {
+     if (supplyMultiplier == 0 || borrowMultiplier == 0) {
+         revert InvalidMultiplierArguments();
+     }
+ }

function updateMultipliers(address market, uint256 supplyMultiplier, uint256 borrowMultiplier) external {
+   _checkMultipliersArguments(supplyMultiplier, borrowMultiplier);
    if (!markets[market].exists) revert MarketNotSupported();
    ...
}
```
---

### [L-4][The `lastAccruedBlock` Naming Needs to Be Refactored]

**Severity:** `LOW`
**Impact:** LOW · **Likelihood:** LOW

#### Description

The NatSpec comment above the `lastAccruedBlock` mapping in `PrimeLiquidityProvider.sol` incorrectly describes it as storing the rate at which a token is distributed to the Prime contract. In reality, this mapping stores the last block number at which an accrual was made for a given token — a fundamentally different concept.

```javascript
File: contracts/Tokens/Prime/PrimeLiquidityProvider.sol

/// @notice The rate at which token is distributed to the Prime contract
mapping(address => uint256) public lastAccruedBlock; // comment is wrong
```
The comment describes `tokenDistributionSpeeds`, not `lastAccruedBlock`. This mismatch between documentation and implementation creates confusion for developers integrating with or auditing the protocol.

#### Impact

Misleading NatSpec causes developer confusion and increases the risk of misuse or incorrect integrations that rely on documentation rather than implementation.

#### Recommended Mitigation

Replace the unsafe subtraction with an explicit comparison:

```diff
- /// @notice The rate at which token is distributed to the Prime contract
+ /// @notice The last block at which tokens were accrued for a given address
mapping(address => uint256) public lastAccruedBlock;
```

---

### [L-5][More Sanity Checks Should Be Applied to `calculateScore()`]

**Severity:** `LOW`
**Impact:** LOW · **Likelihood:** LOW

#### Description

The `calculateScore()` function accepts alpha numerator and denominator as `uint256` values and later casts them to `int256` when computing the exponentiation. If either value exceeds `type(int256).max`, the cast will overflow and cause a revert, with no user-friendly error message.

#### Impact

While passing a value above `type(int256).max` is unlikely in practice, the absence of an explicit check means such a scenario produces an opaque panic revert rather than a descriptive protocol error. Adding bounds checks improves robustness and developer experience.

#### Recommended Mitigation

Validate that numerator and denominator values are within safe int256 bounds before use:

```diff
function calculateScore(...) public view returns (uint256) {
+   require(alphaNumerator <= uint256(type(int256).max), "alphaNumerator overflow");
+   require(alphaDenominator <= uint256(type(int256).max), "alphaDenominator overflow");
    ...
}
```

---

### [L-6][`getPendingInterests()` Should Handle Cases Where a Market Is Unavailable]

**Severity:** `LOW`
**Impact:** LOW · **Likelihood:** LOW

#### Description

`getPendingInterests()` iterates over all markets to fetch accrued interest for a user. If any market becomes unavailable or reverts internally (e.g., due to a paused or deprecated market), the entire loop is interrupted and the function reverts, making pending interest data for all subsequent markets inaccessible.

```javascript
function getPendingInterests(address user) external returns (PendingInterest[] memory pendingInterests) {
    address[] storage _allMarkets = allMarkets;
    PendingInterest[] memory pendingInterests = new PendingInterest[](_allMarkets.length);

    for (uint256 i = 0; i < _allMarkets.length; ) {
        address market = _allMarkets[i];
        uint256 interestAccrued = getInterestAccrued(market, user); // ❌ can revert
        uint256 accrued = interests[market][user].accrued;

        pendingInterests[i] = PendingInterest({
            market: IVToken(market).underlying(),
            amount: interestAccrued + accrued
        });

        unchecked { i++; }
    }

    return pendingInterests;
}
```

#### Impact

A single failing market blocks retrieval of pending interest data for all other markets. This can mislead users into thinking they have no pending rewards and may affect user decisions regarding claiming or staking.

#### Recommended Mitigation

Wrap the inner calls in a try/catch block and emit an event for failed markets, allowing the loop to continue:

```diff
+ event MarketCallFailed(address indexed market);

for (uint256 i = 0; i < _allMarkets.length; ) {
    address market = _allMarkets[i];

+   try this.getInterestAccrued(market, user) returns (uint256 interestAccrued) {
        uint256 accrued = interests[market][user].accrued;
        pendingInterests[i] = PendingInterest({
            market: IVToken(market).underlying(),
            amount: interestAccrued + accrued
        });
+   } catch {
+       emit MarketCallFailed(market);
+   }

    unchecked { i++; }
}
```

---

### [L-7][Missing Zero/`isContract` Checks in Important Functions]

**Severity:** `LOW`
**Impact:** LOW · **Likelihood:** LOW

#### Description

The protocol defines `_ensureZeroAddress()` as a standard guard against zero-address inputs, and uses it in some functions such as `setPrime()`. However, this check is not consistently applied across all functions that accept address parameters, leaving several entry points vulnerable to misconfiguration. Additionally, no `isContract()` check exists for cases where the provided address must be a deployed contract.

```javascript
function _ensureZeroAddress(address address_) internal pure {
    if (address_ == address(0)) revert InvalidArguments();
}
```

#### Impact

An admin error passing a zero address or an EOA where a contract is expected could misconfigure the protocol silently, potentially causing downstream failures when the address is later called as a contract.

#### Recommended Mitigation

Apply `_ensureZeroAddress()` to all functions accepting address parameters. For addresses that must be contracts, add an `isContract()` helper:

```diff
+ event MarketCallFailed(address indexed market);
+ function isContract(address addr) internal view returns (bool) {
+     uint256 size;
+     assembly { size := extcodesize(addr) }
+     return size > 0;
+ }
```
Then apply both checks at all relevant entry points where a valid contract address is required.

---

### [L-8][The `uintDiv()` Function from `FixedMath.sol` Should Protect Against Division by Zero]

**Severity:** `LOW`
**Impact:** LOW · **Likelihood:** LOW

#### Description

The `uintDiv()` function guards against negative divisors but does not guard against a divisor of `0`. Passing `f = 0` bypasses the existing check and causes a panic error due to division by zero, rather than a clean protocol revert with a descriptive error.

```javascript
function uintDiv(uint256 u, int256 f) internal pure returns (uint256) {
    if (f < 0) revert InvalidFixedPoint(); // guards negative
    // f == 0 is accepted and causes a panic
    return uint256((u.toInt256() * FixedMath0x.FIXED_1) / f);
}
```

#### Impact

A zero divisor produces an unhandled panic revert rather than a descriptive `InvalidFixedPoint` revert, making debugging harder and the protocol's error handling inconsistent.

#### Recommended Mitigation

Extend the existing guard to also reject zero:

```diff
function uintDiv(uint256 u, int256 f) internal pure returns (uint256) {
-   if (f < 0) revert InvalidFixedPoint();
+   if (f <= 0) revert InvalidFixedPoint();
    return uint256((u.toInt256() * FixedMath0x.FIXED_1) / f);
}
```

---

*Report generated by [Srishti](https://SrishtiCode.io) · [April 2026]*
