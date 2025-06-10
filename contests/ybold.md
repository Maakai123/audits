## Yearn yBOLD
[Contest details](https://audits.sherlock.xyz/contests/977/report)

### [Medium-01] `estimatedTotalAssets()` omits on-hand `COLL`, enabling pps arbitrage drain

**Summary**

The view function `estimatedTotalAssets()` only sums BOLD on-hand and SP deposits, ignoring any collateral (`COLL`) that’s been claimed but not yet auctioned.

During the “claim → auction” window, `harvest()` records a temporary loss (PPS < 1). An attacker can deposit when PPS is depressed, wait for the auction to complete (PPS ≈ 1), then withdraw and pocket the spread.

**Root Cause**

`estimatedTotalAssets()` is implemented as
```solidity
return  
  asset.balanceOf(address(this))              // @audit only BOLD balance  
  + SP.getCompoundedBoldDeposit(address(this)); // @audit only SP deposit  
```

No term accounts for `COLL.balanceOf(address(this))` after `claim()` but before being swapped.

**Internal Pre-conditions**

- Strategy has claimed collateral gain via `claim()` (so `COLL.balanceOf(this) > 0`).
- `harvest()` runs before `tend()`, triggering a report that uses `estimatedTotalAssets()`.

**External Pre-conditions**

None

**Attack Path**

1. Keeper/attacker calls `harvest()` (or `report()`), which under the hood invokes `estimatedTotalAssets()` → sees only BOLD + SP deposit, omitting on-contract `COLL`.
2. Vault records a “loss” since totalAssets dips below the previous snapshot → share-price (PPS) drops below 1 BOLD.
3. Attacker deposits BOLD at PPS < 1, receiving more shares per BOLD.
4. Keeper subsequently calls `tend()`, auctions off the claimed `COLL`, converting it back to BOLD and raising `totalAssets` → PPS returns to ≈1.
5. Attacker withdraws at PPS ≈ 1, exchanging shares for more BOLD than they originally deposited, pocketing the delta.

**Impact**

**Economic drain:** Attackers extract surplus value on each collateral-release cycle without any on-chain keys or exotic conditions.
**Relevant loss:** Easily exceeds 0.01% and $10 per cycle, and can be repeated indefinitely.