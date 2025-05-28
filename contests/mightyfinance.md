## Mighty Finance
[Contest details](https://cantina.xyz/competitions/616d8bb4-16ce-4ca9-9ce9-5b99d6e146ef)

### [High-01] Zero-Slippage Guard Lets Sandwich Bots Drain `reducePosition`

**Description**

Every invocation of `ShadowRangePositionImpl:reducePosition` uses a zero slippage bound `(amountOutMinimum = 0)`. This means the swap will never revert even if the pool price moves drastically against the user. An MEV-powered sandwich attack can:

1. Front-run the user’s `reducePosition` transaction by swapping a small amount to push the pool price one tick (or more).
2. Let the victim’s reducePosition execute (it won’t revert because `minOut = 0`).
3. Back-run by swapping back, pocketing the price difference.

Over time, each reducePosition call silently loses value, draining user funds under the guise of a normal rebalance.

**Recommendation**

Introduce slippage bounds on every swap


### [Low-01] Incorrect Reward Distribution Due to `xShadow`/`x33` Wrapping Lock in `ShadowRangePositionImpl`
**Description**

The `_claimRewards` function in `ShadowRangePositionImpl` skips `xShadow` reward distribution if the `x33` contract is locked (`IShadowX33(x33).isUnlocked() returns false`), as it neither wraps `xShadow` into `x33` nor transfers it directly.

During `x33`’s epoch-based lock periods (e.g., 12 hours), users calling `claimRewards` or `closePosition` receive no `xShadow`/`x33` rewards, despite earning them, with no error or queuing mechanism to ensure later delivery.

This silent failure leads to underpayment, and there’s no guaranteed way to recover rewards unless users retry during an unlocked period.

**Scenario**

1. `x33` enters a lock period, so `isUnlocked()` returns `false`.
2. A user triggers `claimRewards` (via the vault) or `closePosition`, which calls `_claimRewards`.
3. For xShadow rewards, the function skips processing, transferring no xShadow or x33 to the vault.
4. The user receives other rewards (if any) but misses `xShadow`/`x33`, with no notification of the loss.

**Recommendation**

To prevent reward loss during `x33` lock periods, the protocol should enhance _claimRewards handling queue unclaimed rewards. If `isUnlocked()` is `false`, store xShadow balances in a mapping or event log for later claiming

### [Low-02] Price oracle staleness in `PrimaryPriceOracle` can lead to incorrect debt ratios

**Description**

The `PrimaryPriceOracle` contract relies on Pyth price feeds with `maxPriceAge` of 1 hour, allowing prices within this window to be considered valid. 

The `ShadowRange` contract uses these prices to calculate debt ratios for critical operations like opening closing and liquidating positions.

During this high market volatility or oracle downtime, a price just within 1-hour window (e.g 59mins old) could be stale leading to incorrect debt ratio calculations. 

This can allow attackers to open over leveraged positions or avoid liquidations when positions are underwater, resulting in significant bad debt for the protocol

**Attack Scenario**

1. During high volatility period, the price oracle fails to update, leaving a stale high price for a token (for example 100usd, 59 minutes old)
2. An attacker opens a leveraged position in `ShadowRangeVault`, borrowing heavily. The stale price inflates the position value, keeping the debt ratio below the `liquidationDebtRatio` (86%)
3. The market price drops (e.g., to $50), but the stale oracle price delays liquidation
4. The attacker closes the position or withdraws profits, leaving the protocol with undercollateralized debt

As a result there are significant financial losses due to unliquidated, underwater positions, potentially causing protocol insolvency

**Recommendation**

Lower the `maxPriceAge` to a shorter window (e.g., 5-15 minutes) or adjust dynamically based on market volatility.


### [Info-01] Credit Ledger Inflation on Over-Repayment

**Description**

In `LendingPool.repay(address onBehalfOf, uint256 debtId, uint256 amount)`, the contract credits the caller’s borrowing power before clamping amount to their actual outstanding debt. Concretely, the code does:

```solidity
// 1) Grant full credit upfront
credits[reserveId][msg.sender] = credits[reserveId][msg.sender].add(amount);

// 2) Clamp repayment and reduce debt
if (amount > debtPosition.borrowed) {
    amount = debtPosition.borrowed;
}

debtPosition.borrowed = debtPosition.borrowed.sub(amount);
```

An attacker who owes, say, 1 ETH can call `repay(..., 2 ETH)`. Their debt is reduced to zero, but their credit balance increases by the full 2 ETH, leaving 1 ETH of “phantom” credit unbacked by any collateral.

Repeating this inflates borrowing power arbitrarily and allows over-borrowing, undermining the protocol’s core collateralization assumptions.

**Recommendation**

Only credit the actual amount paid. Move the credit‐update to after the clamp


### [Info-02] Lack of Access Control on `newDebtPosition()` in `LendingPool`

**Description**

The `newDebtPosition()` function in the `LendingPool` contract is external and lacks access control, allowing any user (EOA or contract) to create debt positions.

This permissionless access enables potential spam or misuse, leading to bloated storage, unnecessary gas costs, and the creation of invalid debt positions that cannot be used by whitelisted vault contracts for borrowing.

This disrupts the intended integration with vault contracts (e.g., `ShadowRangePositionImpl`) and could facilitate denial-of-service (DoS) attacks.

**Recommendation**

Restrict `newDebtPosition()` to be callable only by whitelisted vault contracts.


### [Info-03] Incorrect Debt Position Ownership Can Break Borrow Functionality

**Description**

The `LendingPool` contract contains a critical flaw in the `newDebtPosition()` function, which sets the debt position’s owner to `_msgSender()`. 

This allows any user to create a debt position, setting themselves as the owner. However, the borrow function requires the caller (`msg.sender`, expected to be a vault contract) to be the owner of the debt position. 

This ownership mismatch prevents vaults from borrowing on behalf of users, breaking the core borrowing functionality of the protocol.

**Scenario**

1. A user (EOA) calls `newDebtPosition(reserveId)`, creating a debt position with `debtId` and `owner = user_address`.
2. The user interacts with a vault (e.g., via `ShadowRangePositionImpl`) to open a leveraged position, which requires the vault to call `borrow(onBehalfOf, debtId, amount)`.
3. The borrow function reverts with `Errors.VL_INVALID_DEBT_OWNER` because `msg.sender` (vault address) does not match `debtPosition.owner` (user address).
4. As a result, the vault cannot borrow tokens, preventing the creation or management of leveraged positions.
5. Maliciously, an attacker could repeatedly call `newDebtPosition()` to create unusable debt positions, disrupting vault operations or front-end functionality.

**Recommendation**

Modify `newDebtPosition()` to set the owner as the vault contract address instead of the user. Ensure that only vault contracts can create debt positions for their users.