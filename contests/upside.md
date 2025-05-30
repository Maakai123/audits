## Upside
[Contest details](https://code4rena.com/audits/2025-05-upside)

### [Low-01] Rounding down of tiny swaps yields zero fee but alters reserves

**Finding description and impact**

In the `processSwapFee()` function, the swap fee is computed using integer division on Solidity’s default `uint256` type:

```solidity
uint256 fee = (_tokenAmount * swapFeeBp) / 10000;
uint256 tokenAmountAfterFee = _tokenAmount - fee;
```

When `_tokenAmount * swapFeeBp` is smaller than 10,000, the division truncates to zero. As a result:

1. `fee == 0` for all swaps below the “dust” threshold.
2. `tokenAmountAfterFee == _tokenAmount`, so the user pays no fee.
3. Nevertheless, the full `_tokenAmount` (i.e. `amountInAfterFee`) is credited to the liquidity reserves, altering both liquidityTokenReserves and metaCoinReserves as if a fee-bearing swap occurred.

Attackers can repeatedly execute tiny “micro-swaps” below the fee threshold, each time increasing the protocol’s reserves without contributing any fee revenue. 

Over many iterations, this can meaningfully skew the reserve ratios and manipulate the implied price curve without cost.

**Recommended mitigation steps**

Round fees up instead of down by replacing the current truncating division with a ceiling operation, ensuring that any non-zero basis point setting always yields at least a 1-unit fee on small swaps