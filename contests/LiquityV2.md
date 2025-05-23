## LiquityV2
[Contest details](https://cantina.xyz/competitions/d86632df-ab33-4448-8198-64955eae6712)

### [Low-01] Instant trove closure allows users to dodge bad debt redistribution

**Description**

The current system allows users to instantly close and reopen troves to avoid absorbing bad debt during liquidation during liquidations. This unfairly shifts the redistribution burden onto other troves, increasing their risk of liquidation

**Attack path**

- Alice detects an upcoming liquidation
- Alice closes the trove (`onCloseTrove()`) before liquidation occurs
- Liquidation executes, generating bad debt that is only redistributed to active Troves(Alice's closed Trove is excluded)
- Alice reopens a new Trove after the liquidation, avoiding his share of the bad debt

As a result, the other troves absorb more bad debt than they should. Alice avoids losses at the expense of other users and repeated abuse can destabilize the system increasing the liquidation risks for honest participants

### [Low-02] Batch shares are miscalculated during debt decrease

**Description**

The `_updateBatchShares` function in TroveManager.sol miscalculates batchDebtShares during partial debt decreases (e.g., via `BorrowerOperations.repayBold` or `redeemCollateral`) due to Solidity’s integer division rounding.

The calculation `batchDebtSharesDelta = currentBatchDebtShares * debtDecrease / _batchDebt` truncates decimal results, causing `Troves[_troveId].batchDebtShares` to be slightly overestimated.

This leads to users overpaying interest and batch fees proportional to their shares. The issue is mitigated when `_newTroveDebt == 0` (shares set to 0), but for partial decreases, errors accumulate over multiple adjustments, skewing the batch’s debt-to-shares ratio

**Recommendation**

Use higher precision in batchDebtSharesDelta calculation to reduce rounding errors

### [Low-03] Users can front run liquidations to evade losses in stability pool

**Description**

The StabilityPool contract allows depositors to front-run liquidations or price drops by withdrawing their deposits before losses can occur. The evasion is possible because the system lacks restrictions on withdrawals during period when liquidations will result in depositor losses(e.g vaults liquidated below 100% collateralization)

The code does not check if the vaults ICR < 110% exists during withdrawals. Depositors can freely withdraw even when liquidations are imminent, evading losses from "bad" liquidations (vaults liquidated below 100% collateralization)

```solidity
function withdrawFromSP(uint256 _amount, bool _doClaim) external override {
    // No check for existing liquidatable vaults (ICR < 110%) 
    // ...
    uint256 boldToWithdraw = LiquityMath._min(_amount, compoundedBoldDeposit);
    // ...
    require(newTotalBoldDeposits >= MIN_BOLD_IN_SP, "Withdrawal must leave totalBoldDeposits >= MIN_BOLD_IN_SP");
}
```

Depositors can monitor external price feeds and front run oracle updates that would trigger liquidations. Depositors can also direcly monitor incoming liquidation tx that would cause them a net loss and frontrun with `withdrawFromSP()` to evade losses

Depositors can withdraw funds before unprofitable liquidations, shifting losses to remaining depositors

**Recommendation**

Modify `withdrawFromSP()` to block withdrawals if there are vaults with ICR < 110%

### [Info-01] Liquidation rewards can be manipulated through stake inflation

**Description**

The `TroveManager` contract’s liquidation reward redistribution mechanism is vulnerable to manipulation via stake inflation.

An attacker can exploit the stake calculation logic (`_computeNewStake`) to disproportionately claim liquidation redistribution rewards by opening a high-collateral, minimal-debt trove just before a liquidation event.

This drains collateral from the DefaultPool and unfairly reduces rewards for honest troves

```solidity
// TroveManager.sol
function _computeNewStake(uint256 _coll) internal view returns (uint256) {  
    if (totalCollateralSnapshot == 0) {  
        return _coll;  
    } else {  
        return (_coll * totalStakesSnapshot) / totalCollateralSnapshot;  
    }  
}
```

A trove’s stake is proportional to its collateral relative to the total collateral snapshot (`totalCollateralSnapshot`). If `totalCollateralSnapshot` is low (e.g., during a liquidation wave), a large collateral deposit inflates the attacker’s stake

```solidity
// TroveManager.sol
function _redistributeDebtAndColl(...) internal {  
    collRewardPerUnitStaked = collNumerator / totalStakes;  
    L_coll += collRewardPerUnitStaked;  
    // ...  
}
```

Redistribution rewards are distributed per-unit-staked. An inflated stake allows the attacker to claim a larger share

**Attack Path**

1. Attacker watches the mempool or PriceFeed for troves nearing liquidation (e.g., ICR < MCR).
2. Uses a flash loan to open a trove deposit 100 ETH as collateral with minimal debt (e.g., MIN_DEBT = 2000 BOLD).
3. If `totalCollateralSnapshot = 1000 ETH`, attacker’s stake = `(100 ETH * totalStakesSnapshot) / 1000 ETH = 10% of totalStakes`.
4. A vulnerable trove (e.g., 10 ETH collateral, 20,000 BOLD debt) is liquidated
5. During `_redistributeDebtAndColl`, rewards are distributed per-unit-staked
6. Attacker’s rewards:
- `collGain = 10% * 10 ETH = 1 ETH`
- `boldDebtGain = 10% * 20,000 BOLD = 2,000 BOLD`

7. Attacker calls `withdrawColl` or `closeTrove` to exit, repaying 2000 BOLD debt and keeping 1 ETH profit

### [info-02] Impossible to Liquidate Last Remaining Trove

**Description**

The `TroveManager` contract enforces a check in `_closeTrove()` that prevents liquidation when there is only one active trove left in the system. This is due to the following logic

When liquidating a trove, the function `_liquidate()` calls `_closeTrove()`. Inside `_closeTrove()`, there is a check

```solidity
function _closeTrove(
    uint256 _troveId,
    TroveChange memory _troveChange,
    address _batchAddress,
    uint256 _newBatchColl,
    uint256 _newBatchDebt,
    Status closedStatus
) internal {
    uint256 TroveIdsArrayLength = TroveIds.length;
    _requireMoreThanOneTroveInSystem(TroveIdsArrayLength); // @audit - reverts if only 1 trove left
    ...
}
```

```solidity
function _requireMoreThanOneTroveInSystem(uint256 TroveIdsArrayLength) internal view {
    require(TroveIdsArrayLength > 1, "TroveManager: Only one trove in the system");
}
```
If a `TroveManager` instance has only one active borrower, it becomes impossible to liquidate that borrower. This can lead to loss of funds if the last trove is undercollateralized but cannot be closed

**Recommendation**

Modify `_closeTrove()` to skip the check when the trove is being liquidated (`closedStatus == Status.closedByLiquidation`)