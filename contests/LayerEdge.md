## LayerEdge
[Contest details](https://audits.sherlock.xyz/contests/952)

### [Medium-01] Unbounded tier-event growth enables account-Level DoS


**Summary**

Every time `_checkBoundariesAndRecord()` detects a stake or unstake crossing a tier boundary, it calls `_findAndRecordTierChange()`, which in turn invokes `_recordTierChange()`. That function unconditionally does:

```solidity
stakerTierHistory[user].push(TierEvent({ … }));  
```

for each affected boundary user. Meanwhile, `calculateUnclaimedInterest()` loops over that same stakerTierHistory[user] array on every `claimInterest()` or `compoundInterest()` call:

```solidity
for (uint256 i = 0; i < userTierHistory.length; i++) { … }
```

As a result, a malicious actor toggling a boundary every block can bloat any victim’s history array without interacting with them—and eventually their interest‐claim transactions run out of gas and revert

**Root Cause**

Neither the boundary‐checking logic nor the tier‐history recorder enforces any cap on `stakerTierHistory[user].length`. Every boundary‐crossing event appends new entries, and `calculateUnclaimedInterest()`’s linear scan over that unbounded array makes future claims O(n) in gas.

**Internal Pre-conditions**

- `_checkBoundariesAndRecord()` is called on every boundary shift and loops over all newly promoted/demoted ranks, pushing one `TierEvent` per user touched

- `calculateUnclaimedInterest()` always iterates over the full `stakerTierHistory[user]` array before computing interest

**External Pre-conditions**

- An attacker stakes or unstakes just around one of the tier thresholds in successive blocks, causing that boundary to flip each time.

- The victim’s address happens to fall on the moving boundary, so each toggle appends a new TierEvent for the victim.

**Attack Path**
1. Attacker alternates stakes/unstakes at the Tier 1/2 cutoff every block.
2. The victim’s rank lies on that cutoff, so each toggle pushes a new TierEvent into their `stakerTierHistory`.
3. After thousands of blocks, the victim’s history array grows so large that any future `claimInterest()` or `compoundInterest()` call exceeds block gas limits and reverts.
4. The victim is permanently blocked from claiming or compounding interest—even though they never interacted with the attacker.

**Impact**

DOS on interest claims for the victim, locking their earned rewards indefinitely.