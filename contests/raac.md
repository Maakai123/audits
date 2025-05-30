## RAAC
[Contest details](https://codehawks.cyfrin.io/c/2025-02-raac)

### [High-01] Overestimation of Borrower Debt Leading to Excess Liquidation Costs in `liquidateBorrower()` function

**Summary**

A critical bug in the `liquidateBorrower()` function of the `StabilityPool.sol` contract causes borrower debt to be overestimated due to double application of the `usageIndex` scaling factor. 

This leads to incorrectly high liquidation amounts, resulting in Stability Pool depositors overpaying for liquidations and potential premature draining of Stability Pool funds

**Vulnerability Details**

Within `liquidateBorrower()`, the user's debt is first retrieved correctly using

```Solidity
uint256 userDebt = lendingPool.getUserDebt(userAddress);
```

However, `getUserDebt()` in `LendingPool.sol` already applies the interest rate multiplier **(`usageIndex`)**, as seen in:

```solidity
return user.scaledDebtBalance.rayMul(reserve.usageIndex);
```

Yet, the function incorrectly scales the debt again by calling:

```solidity
uint256 scaledUserDebt = WadRayMath.rayMul(userDebt, lendingPool.getNormalizedDebt());
```

Since `lendingPool.getNormalizedDebt()` is also equivalent to `usageIndex`, this results in:

```solidity
scaledUserDebt = user.scaledDebtBalance * usageIndex * usageIndex;
```

This incorrectly applies `usageIndex` twice, making liquidation amounts artificially high.

**Impact**

Since `scaledUserDebt` is larger than the real debt, the Stability Pool pays more than required to cover liquidations. Depositors lose more RTokens (rCRVUSD) than necessary, leading to faster depletion of the Stability Pool

**Recommendations**

Remove Redundant Scaling in `liquidateBorrower()`


### [Medium-01] `MIN_VOTE_DELAY` not used in `vote()` function enabling unrestricted vote spamming

**Summary**

The `GuageController:vote()` function doesn't enforce `MIN_VOTE_DELAY`

## Vulnerability Details

The contract defines `MIN_VOTE_DELAY`(10 days) but fails to enforce it in the contract. 

```Solidity
    uint256 public constant MIN_VOTE_DELAY = 1 days;
```

The `vote` function contains no checks against `lastUpdatedTime[msg.sender]`, allowing unlimited voting frequency. Attackers can manipulate gauge weights through rapid vote spamming and negligible micro-votes, destabilizing reward distributions

**Impact**

Attackers can DOS legitimate voters through gas price wars for vote inclusion


**Recommendations**

Enforce `MIN_VOTE_DELAY`


### [Medium-02] Users cannot repay when the `LendingPool` contract is paused causing forced and unfair liquidations

**Summary**

The RAAC protocol freezes all user actions when the contract is paused, including repaying loans. 

This creates a critical issue where borrowers cannot protect themselves from liquidation during a pause, leading to unfair liquidations that forcefully seize user assets.

In a properly designed system, users should always be able to repay loans or add collateral, even if the protocol is paused for security reasons. 

Blocking these actions results in forced liquidations, even when a user has the funds to prevent it.

**Vulnerability Details**

The pause mechanism is designed to stop protocol interactions during emergencies. However, it does not distinguish between harmful actions (e.g., borrowing more funds) and protective actions (e.g., repaying debt).

The key issue lies in how the `whenNotPaused` modifier is applied to repayment functions

When the contract is paused, users are blocked from calling:

* `repay(uint256 amount, address user, address onBehalfOf)` – prevents users from clearing their debt, even if they have the funds.

* `repayOnBehalf(uint256 amount, address onBehalfOf)repayOnBehalf(uint256 amount, address onBehalfOf)` -  prevents users from helping other's clear debt

As a result, borrowers get liquidated even when they have the ability to prevent it.

**Impact**

Users cannot repay leading to forced and unfair liquidations


**Recommendations**

Allow repayments  even when contract is paused
