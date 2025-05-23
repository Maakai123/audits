## Infinifi
[Contest details](https://cantina.xyz/competitions/2ac7f906-1661-47eb-bfd6-519f5db0d36b)

### [Low-01] Overstated loss allocation in `LockingController.applyLosses()` due to `mulDivUp`

**Description**

The `applyLosses()` contains a bug where it uses `mulDivUp` to calculate loss allocations for each bucket. This function is designed to burn a specified amount of receipt token (`_amount`) and distribute the loss proportionally across buckets based on their share of total receipt tokens (`epochTotalReceiptToken` relative to `globalRecieptToken`).

```solidity
uint256 allocation = epochTotalReceiptToken.mulDivUp(_amount, _globalReceiptToken);
```

However `mulDivUp` rounds up each bucket's allocation, which can cause the sum of allocations to exceed _amount. As a result `globalReceiptToken` is decremented by more than the actual tokens burned, leading to an inconsistency between contract's internal accounting and it's actual token balance.


**Scenario**

- `_globalReceiptToken = 1000`
- Two buckets, each with `epochTotalReceiptToken = 500`
- `_amount = 99`(tokens to burn)
- For each bucket: `allocation = 500 * 99 / 1000 = 49.5`, but `mulDivUp` rounds to 50
- Total allocation: `50 + 50 = 100`
- Tokens burned: `99` (via `ERC20Burnable(receiptToken).burn(_amount)`)
- `globalReceiptToken` decremented by `100` (via `globalReceiptTokenDecrement`)
- New `globalReceiptToken = 1000 - 100 = 900`
- Actual token balance should be `1000 - 99 = 901`
- The contract understates the token balance by 1 unit

**Recommendation**

To resolve the bug, replace `mulDivUp` with `mulDivDown` in the loss allocation calculation within `applyLosses()`. This ensures that the sum of allocations across buckets does not exceed `_amount`, aligning the decrement of `globalReceiptToken` with the actual tokens burned.