## Mighty Finance
[Contest details](https://cantina.xyz/competitions/616d8bb4-16ce-4ca9-9ce9-5b99d6e146ef)

### [High-01] In setReward(), block.timestamp is used instead of startTime at lastUpdate

**Description**

In setReward rewardData[rewardToken].lastUpdateTime is set to block.timestamp instead of startTime. This causes the reward calculation to include the period between block.timestamp and startTime when rewards should not be distributed.

```rust
rewardData[rewardToken].lastUpdateTime = block.timestamp;
```

**Recommendation**
To fix the bug, set lastUpdateTime to startTime to ensure reward calculations begin at the intended start time:

```rust
rewardData[rewardToken].lastUpdateTime = startTime;
```



### [High-02] Lack of Slippage Protection in Swaps leads to financial losses

**Description**
Inside the ShadowRangePositionImpl contract the _swapTokenExactInput function performs token swaps via a Uniswap V3-style router (IShadowSwapRouter). The function accepts an amountOutMinimum parameter, which specifies the minimum amount of output tokens expected from the swap. If the actual output is less than amountOutMinimum, the swap reverts, providing slippage protection. However, in critical functions like reducePosition, amountOutMinimum is explicitly set to 0, disabling this protection. This allows swaps to execute even if the output is negligible, exposing the contract to high slippage in low-liquidity pools or manipulated markets.

Relevant Code (in reducePosition)

```rust
if (amount0ToSwap > 0) {
    _swapTokenExactInput(token0, token1, amount0ToSwap, 0);
}
if (amount1ToSwap > 0) {
    _swapTokenExactInput(token1, token0, amount1ToSwap, 0);
}
(_swapTokenExactInput)
```


```rust
function _swapTokenExactInput(address tokenIn, address tokenOut, uint256 amountIn, uint256 amountOutMinimum)
    internal
    returns (uint256 amountOut)
{
    address router = IAddressRegistry(IVault(vault).addressProvider()).getAddress(AddressId.ADDRESS_ID_SHADOW_ROUTER);
    IERC20(tokenIn).approve(router, amountIn);
    amountOut = IShadowSwapRouter(router).exactInputSingle(
        IShadowSwapRouter.ExactInputSingleParams({
            tokenIn: tokenIn,
            tokenOut: tokenOut,
            tickSpacing: tickSpacing,
            recipient: address(this),
            deadline: block.timestamp,
            amountIn: amountIn,
            amountOutMinimum: amountOutMinimum,
            sqrtPriceLimitX96: 0
        })
    );
}
```

**Recommendation**
Set amountOutMinimum to a value based on an expected output, adjusted for a reasonable slippage tolerance (e.g., 1-2%).






### [Medium-01] Config does not update the currentBorrowingRate LendingPool::setBorrowingRateConfig


**Description**

The setBorrowingRateConfig does not call updateState and updateInterestRates the new borrowing rate does not take effect immediately and only updates when another action (e.g., deposit, borrow, or repay) triggers a state update.

The setBorrowingRateConfigfunction in the LendingPool contract is responsible for updating the borrowing rate configuration, which defines how interest rates scale with the reserve’s utilization rate . The function sets parameters such as utilizationA, borrowingRateA, utilizationB, borrowingRateB, and maxBorrowingRate to create a piecewise linear curve for the borrowing rate. However, the function does not call updateState or updateInterestRates, which are necessary to apply the new configuration to the reserve’s currentBorrowingRate. As a result, the new borrowing rate does not take effect immediately and only updates when another action (e.g., deposit, borrow, or repay) triggers a state update. This delay can lead to the protocol using outdated interest rates, potentially causing financial inefficiencies or misaligned incentives.

**Impact Explanation**
The protocol will actually have no control of when the new borrowingRateConfig will be actually activated. For the meantime between the s etBorrowingConfigRate until the next action on the LendingPool which will update the interest rates (which is indefinite), the currentBorrowingRate that will be used will be the old one which was based on the previous borrowingRateConfig. This means that the protocol can lose funds since they will want for example for the current utilization rate to have 25% interest rate but instead they will have the previous 15% interest rate until someone update the state by deposit/borrow...

**Recommendation**

To ensure the new borrowing rate configuration takes effect immediately, the setBorrowingRateConfig function should call updateState before updating the configuration and updateInterestRates after updating it. This ensures the reserve’s state (e.g., utilization rate) is current before applying the new config and that the currentBorrowingRate reflects the updated configuration immediately.







### [Low-01] Liquidation Fee Calculation Uses Integer Division, Leading to Precision Loss

**Description**
The liquidatePosition function calculates fees to be paid to a liquidationFeeRecipient and the caller (e.g., a liquidator). These fees are a percentage of the tokens reduced from the position (token0Reduced and token1Reduced).

The fee calculations use integer division, which rounds down to the nearest whole number. This can lead to precision loss, where small fee amounts are rounded to zero, especially for low token amounts or low fee percentages. This results in:

No fees being collected for small positions or low-value tokens. Reduced incentives for liquidators, as caller fees may be zero. Potential accounting errors if the protocol expects non-zero fees.


```rust

function liquidatePosition(address caller)
    external
    onlyVault
    returns (
        uint128 liquidity,
        uint256 token0Reduced,
        uint256 token1Reduced,
        uint256 token0Fees,
        uint256 token1Fees,
        uint256 token0Left,
        uint256 token1Left
    )
{
    // ... existing logic ...
    LiquidationFeeVars memory vars = LiquidationFeeVars({
        liquidationFee: IVault(vault).liquidationFee(), // e.g., 500 bps (5%)
        liquidationCallerFee: IVault(vault).liquidationCallerFee(), // e.g., 100 bps (1%)
        liquidationFeeRecipient: IVault(vault).liquidationFeeRecipient()
    });

    // Handle liquidation fees
    if (token0Reduced > 0) {
        token0Fees = token0Reduced * vars.liquidationFee / 10000; // Integer division
        pay(token0, address(this), vars.liquidationFeeRecipient, token0Fees);

        uint256 token0CallerFees = token0Fees * vars.liquidationCallerFee / 10000; // Integer division
        if (token0CallerFees > 0) {
            pay(token0, address(this), caller, token0CallerFees);
        }

        token0Reduced = token0Reduced - token0Fees - token0CallerFees;
    }

    if (token1Reduced > 0) {
        token1Fees = token1Reduced * vars.liquidationFee / 10000; // Integer division
        pay(token1, address(this), vars.liquidationFeeRecipient, token1Fees);

        uint256 token1CallerFees = token1Fees * vars.liquidationCallerFee / 10000; // Integer division
        if (token1CallerFees > 0) {
            pay(token1, address(this), caller, token1CallerFees);
        }

        token1Reduced = token1Reduced - token1Fees - token1CallerFees;
    }
 .
}

```




**Recommendation**
erform calculations with higher precision by scaling intermediate results before division, reducing rounding errors.

```rust
if (token0Reduced > 0) {
    token0Fees = (token0Reduced * vars.liquidationFee * 1e18) / 10000 / 1e18; // Scale up
    token0Fees = token0Fees > 0 ? token0Fees : 1;
    pay(token0, address(this), vars.liquidationFeeRecipient, token0Fees);

    token0CallerFees = (token0Fees * vars.liquidationCallerFee * 1e18) / 10000 / 1e18;
    token0CallerFees = token0CallerFees > 0 ? token0CallerFees : 1;
    if (token0CallerFees > 0) {
        pay(token0, address(this), caller, token0CallerFees);
    }

    token0Reduced = token0Reduced - token0Fees - token0CallerFees;
}
```
