## Metropolis
[Contest details](https://cantina.xyz/competitions/076935b1-2706-48c6-bf0a-b3656aa24194)

### [Low-01] Admin can brick all queued withdrawals by changing strategy

**Description**

The `BaseVault` contract transfers vault shares to the current strategy's address when user queue withdrawals through `queuewithdrawal()` function. But if the strategy is changed through `setStrategy()`, the new strategy does not receive these shares and the old strategy retains them.

When the new strategy calls `executeQueuedWithdrawals()`, it attempts to burn shares from its own address which holds none, causing the transaction to revert. This design flaw breaks the queued withdrawal mechanism after a strategy change leaving users unable to withdraw their queued withdrawals

```solidity
//code snippets

function queueWithdrawal(uint256 shares, address recipient) {
    _transfer(msg.sender, address(strategy), shares); // Shares sent to strategy
    // ... records withdrawal ...
}

function executeQueuedWithdrawals() {
    // Only callable by current strategy
    _burn(address(strategy), totalQueuedShares); // @audit - fails if shares aren't here!
}
```

**Recommendation**

To fix this, track queued shares internally within the vault instead of transferring them to the strategy. This decouples share management from the strategy’s address, ensuring that strategy changes do not disrupt the withdrawal process