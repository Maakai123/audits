## Primev
[Contest details](https://cantina.xyz/competitions/e92be0b9-b4f2-4bf2-9544-ae285fcfc02d)

### [Info-01] Unchecked growth of orphanedRewards mapping can lead to DOS or storage bloat

**Description**

Because any caller can invoke

```solidity
payProposer(pubkey, { value: 1 wei });
```

with a fresh, random 48-byte pubkey, each invocation creates a new nonzero slot in the `orphanedRewards` mapping.

Over time, the orphanedRewards mapping grows unboundedly each unique 48-byte key costs ~20 k gas for the first write. 

An attacker can cheaply land many tiny-value calls (e.g. 1 wei each) to force thousands of new storage slots.

Although the attacker pays ETH, this becomes a DoS: excessive storage slots increase future SSTORE/SLOAD gas costs and can even make the owner’s `claimOrphanedRewards()` loop (over a long list) run out of gas.

Eventually, `claimOrphanedRewards(bytes[] calldata pubkeys, address toPay)` becomes infeasible. If the owner wants to sweep a large set of orphaned keys but the array grows too big, the total gas to iterate and SSTORE‐clear them exceeds block limits.

As a result, any orphaned ETH (even if a real validator’s pubkey was used at some point) may become permanently locked, because the owner cannot efficiently iterate or prune the bloated mapping.

**Recommendation**

Disallow unbounded orphaned‐rewards entries by rejecting calls for non-opted-in pubkeys rather than storing them.