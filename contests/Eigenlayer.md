## EigenLayer
[Contest Details](https://cantina.xyz/competitions/e7af4986-183d-4764-8bd2-1d6b47f87d99)

### [Info-01] Potential DOS on `getAllocatedSets()` function

**Summary**

The `getAllocatedSets()` and related view functions are vulnerable to DOS attacks due to unbounded loops over state variables that can be manipulated by operators using the `modifyAllocation()` function. A malicious operator can inflate the allocatedSets array, causing legitimate calls to these functions to run out of gas.

```solidity
function getAllocatedSets(address operator) external view returns (OperatorSet[] memory) {
    uint256 length = allocatedSets[operator].length();  // @audit - Can be manipulated
    OperatorSet[] memory operatorSets = new OperatorSet[](length);
    for (uint256 i = 0; i < length; i++) {  // @auditt - Gas-intensive loop
        operatorSets[i] = OperatorSetLib.decode(allocatedSets[operator].at(i));
    }
    return operatorSets;
}
```

**Attack Path**

1. An malicious operator calls `modifyAllocations` repeatedly, adding many allocations to different operator sets. Each allocation increases the size of `allocatedSets[operator]`
2. The allocatedSets array grows arbitrarily large (e.g., thousands of entries).
3. When a legitimate user calls `getAllocatedSets(operator)`, the function loops through the inflated array, consuming excessive gas and reverting.

**Impact Explanation**

DOS on `getAllocatedSets()`

**Recommendation**

Add pagination to limit loop iterations per call. Restrict how often operators can call `modifyAllocations()` (e.g., cooldown periods).