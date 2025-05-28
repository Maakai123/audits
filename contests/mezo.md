## Mezo
[Contest details](https://cantina.xyz/code/e757364c-1f68-4ec5-94f6-c6b3c2e80c6d/README.md)

### [Medium-01] Interest is double counted in borrowing capacity check in `BorrowerOperations`

**Description**

The `BorrowerOperations` contract’s `_requireHasBorrowingCapacity` function incorrectly double-counts the interest owed when checking if a debt increase is within a trove’s maximum borrowing capacity. The function checks

```solidity
require(
    _vars.maxBorrowingCapacity >=
        _vars.netDebtChange + _vars.debt + _vars.interestOwed,
    "BorrowerOps: An operation that exceeds maxBorrowingCapacity is not permitted"
);

```

Here, `_vars.debt` (from `troveManager.getTroveDebt(_borrower)`) already includes `_vars.interestOwed` (from `troveManager.getTroveInterestOwed(_borrower)`), as the total debt comprises principal, accrued interest, and gas compensation.

Adding `_vars.interestOwed` again overestimates the required borrowing capacity, making the check less restrictive than intended. 

This could allow debt increases closer to the maximum capacity than designed, reducing the safety margin

While other checks (e.g., `_requireICRisAboveMCR`) currently prevent actual over-borrowing beyond the Minimum Collateral Ratio (MCR, 110%), the incorrect logic introduces inconsistency and potential risks if combined with other vulnerabilities or future changes in debt calculation logic

**Recommendation**

Update `_requireHasBorrowingCapacity` to check only `vars.netDebtChange + vars.debt`

### [Medium-02] Delayed debt distribution in batch troves causes liquidation cascades

**Description**

The `batchLiquidateTroves()` function in the `TroveManager` contract accumulates bad debt (principal and interest) from liquidated troves during batch processing but only redistributes it to active troves after all troves in the batch are processed. This delayed redistribution causes two critical issues

1. **Stale ICR Calculations**: Individual Collateral Ratio (ICR) calculations for troves later in the batch do not account for the accumulating bad debt, leading to optimistic (higher) ICRs that do not reflect the pending debt redistribution.

2. **Sudden ICR Drop**: After the batch completes, the accumulated bad debt is redistributed in a single step, causing an unexpected and significant drop in ICR for all active troves, potentially pushing them below the Minimum Collateral Ratio (MCR, 110%) and triggering further liquidations.

This vulnerability undermines the fairness and stability of the liquidation mechanism, as users are unaware of the impending debt increase during batch processing, leading to unanticipated liquidations and potential cascading effects.

**Scenario**

1. A liquidator calls `batchLiquidateTroves()` with a list of troves, some of which have ICR < MCR due to a price drop or undercollateralization.

2. The function iterates through the trove array, liquidating eligible troves and accumulating bad debt (principal and interest) in the `LiquidationTotals` struct without updating the system’s per-unit-staked terms (`L_Principal`, `L_Interest`).
3. ICR calculations for subsequent troves in the batch (via `getCurrentICR`) use the stale `L_Principal` and `L_Interest`, excluding pending bad debt and resulting in higher-than-actual ICRs.
4. After processing all troves, the function redistributes the accumulated bad debt to active troves by updating `L_Principal` and `L_Interest`, significantly increasing each trove’s pending debt.
5. The sudden debt increase reduces the ICR of all active troves, potentially pushing them below MCR and making them eligible for liquidation in subsequent calls, leading to cascading liquidations.

**Recommendation**

Update `L_Collateral`, `L_Principal`, and `L_Interest` after each liquidation in the batch to reflect bad debt in subsequent ICR calculations

### [Medium-03] Instant trove closure allows users to dodge bad debt redistribution

**Description**
The `BorrowerOperations` contract allows users to instantly close and reopen troves without restrictions, enabling a Trove Front-Running or Liquidation Evasion attack.

This vulnerability permits a malicious user (e.g., Alice) to close their trove just before a liquidation event, avoid absorbing redistributed bad debt, and reopen a new trove afterward.

This unfairly shifts the bad debt burden onto other active troves, increasing their risk of liquidation and potentially destabilizing the system

**Attack Path**

1. Alice monitors the price feed (`priceFeed.fetchPrice()`) and trove statuses (`troveManager.getTroveColl()`, `troveManager.getTroveDebt()`) to detect an impending liquidation, triggered when a trove’s Individual Collateral Ratio (ICR) falls below the Minimum Collateral Ratio (MCR, 110%) or, in Recovery Mode, below the Critical Collateral Ratio (CCR, 150%)
2. Alice calls `closeTrove()` to close her trove, repaying her debt (minus `MUSD_GAS_COMPENSATION`, `200e18`) and recovering her collateral. This is permissionless and instantaneous in Normal Mode (TCR ≥ CCR).
3. The liquidation occurs, redistributing bad debt (uncollateralized debt from the liquidated trove) only to active troves. Since Alice’s trove is closed, it is excluded from this process.
4. Alice calls `openTrove()` to create a new trove using the recovered collateral, meeting the minimum net debt requirement (`minNetDeb`t, 1800e18) and ICR thresholds (MCR in Normal Mode, CCR in Recovery Mode). There are no cooldowns or penalties preventing immediate reopening
5. Other active troves absorb a larger share of bad debt, reducing their ICRs and increasing their liquidation risk. Repeated exploitation concentrates bad debt among honest users, potentially triggering cascading liquidations

**Recommendation**

Enforce a timelock on Trove closures.

### [Low-01] Incorrect EIP-712 struct hash in `BorrowOperationsSignatures`

**Description**

The `BorrowerOperationsSignatures` contract incorrectly constructs the EIP-712 struct hash in the `_verifySignature()` function by using abi.encodePacked with pre-encoded parameters (`_data`), combined with nonce and deadline.

According to the EIP-712 standard, the struct hash should be computed as `keccak256(abi.encode(typeHash, param1, param2, ..., nonce, deadline))`, encoding each parameter individually. Instead, the contract computes
```solidity
keccak256(abi.encodePacked(_typeHash, _data, nonces[_borrower], _deadline))
```
where `_data` is a pre-encoded blob (e.g., `abi.encode(params...)`). This misconstruction leads to invalid digests, causing all signature verifications to fail, even for valid signatures.

As a result, all signature-based functions (`addCollWithSignature`, `openTroveWithSignature`, `withdrawCollWithSignature`, `withdrawMUSDWithSignature`, `repayMUSDWithSignature`, `adjustTroveWithSignature`, `closeTroveWithSignature`, `refinanceWithSignature`, `claimCollateralWithSignature`) are unusable, rendering the contract’s core functionality for delegated trove operations broken.

**Recommendation**

Modify `_verifySignature()` to encode parameters individually, eliminating the use of `_data` and `abi.encodePacked`

### [Info-01] A timing attack can be carried out on `PreBlockHandler` sub-handler execution

**Description**

In `app/abci/preblock.go` the `PreBlockHandler` processes sub-handlers sequentially without timeouts or parallel execution, allowing an attacker to delay block finalization by submitting a malicious `InjectedTx` that causes one sub-handler to hang.

The `PreBlocker` iterates over sub-handlers (bridge and connect) in a fixed order (`preblock.go`), passing each the `InjectedTx` parts and regular transactions.

If a sub-handler (e.g., `bridge`) processes a malicious `InjectedTx` part that triggers a long-running operation (e.g., an external call to a slow Ethereum sidecar), the entire `FinalizeBlock` request stalls, as there’s no timeout or parallel execution. The `InjectedTx` struct (`proposal.pb.go`) allows arbitrary Parts data, which sub-handlers may not validate thoroughly

An attacker can submit a vote extension with a malicious Parts entry for `VoteExtensionPartBridge` that, when included in an `InjectedTx`, causes the bridge sub-handler to hang (e.g., by triggering a slow Ethereum sidecar query). This delays the `PreBlocker` execution, slowing block production.

If validators include this `InjectedTx` in proposals, the chain’s throughput drops significantly, enabling a DoS attack. Alternatively, an attacker could target the `connect` sub-handler by crafting a part that exploits the oracle client’s processing logic.

**Recommendation**

Implement per-sub-handler timeouts in `PreBlocker`. Run sub-handlers in parallel with a maximum wait time. Validate `InjectedTx` parts before processing to reject potentially malicious data.