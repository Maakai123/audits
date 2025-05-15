## Gamma Strategies
[Contest Details](https://cantina.xyz/competitions/aaf79192-6ea7-4b1e-aed7-3d23212dd0f1)


### [Info-01] Incorrect Tick Rounding for Negative Ticks in `getRoundedTargetTick()` and `getRoundedCurrentTick()`

**Description**

In the `TickLibrary`, the `getRoundedTargetTick()` and `getRoundedCurrentTick()` functions incorrectly round ticks for `token0` orders (`isToken0 = true`) in negative tick ranges. Specifically:

- `getRoundedTargetTick` rounds negative `targetTick` downward (more negative), when it should round upward (less negative) to ensure it’s above the `currentTick`.
- `getRoundedCurrentTick` rounds negative `currentTick` upward (less negative), when it should round downward (more negative) to maintain `roundedCurrentTick < roundedTargetTick`. This causes `getValidTickRange` to reject valid `token0` limit orders, as `roundedTargetTick` may be less than or equal to `roundedCurrentTick`, triggering the `RoundedTargetTickLessThanRoundedCurrentTick` error.
- As a result, users cannot place limit orders in negative tick ranges for `token0`, leading to transaction failures and loss of functionality.

**Recommendation**

Adjust the rounding logic in `getRoundedTargetTick()` and `getRoundedCurrentTick()` to align with order direction