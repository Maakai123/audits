## aave-aptos
[Contest Details](https://cantina.xyz/code/ad445d42-9d39-4bcf-becb-0c6c8689b767/overview)

### [MEDIUM-01] Lack of Freshness Check Allows Stale Oracle Data in "Liquidation Logic"

**Summary**

The aave_pool::liquidation_logic module lacks staleness detection for asset prices retrieved from the oracle::get_asset_price function, allowing the use of outdated prices during liquidation calculations. This could lead to incorrect collateral and debt valuations, potentially causing unfair liquidations or financial losses.



The liquidation_logic module in the Aave protocol relies on the oracle::get_asset_price function to fetch prices for collateral and debt assets during the liquidation_call function. These prices are used to calculate the amount of collateral to liquidate and debt to cover, as well as to assess the user's health factor. However, the oracle does not validate the freshness of price data, accepting prices that are significantly outdated (e.g., 24 hours old).

Liquidations rely on accurate, up-to-date asset prices to ensure that the correct amount of collateral is seized to cover the debt. Stale prices can lead to over- or under-liquidation, misrepresenting the financial state of the protocol and users.

Users may be unfairly liquidated based on outdated market conditions, or liquidators could exploit stale prices to gain excessive collateral at a lower cost.



## Impact Explanation

Financial Loss: Incorrect price data can lead to over-liquidation, where users lose more collateral than necessary, or under-liquidation, where the protocol fails to recover sufficient debt, potentially creating bad debt. Exploitation Potential: Liquidators could exploit stale prices during market volatility to liquidate positions at a profit, undermining user trust and protocol fairness.


## Recommendation
The liquidation_logic module should implement staleness detection for price data. The recommended approach is to add timestamp validation in the oracle::get_asset_price function to ensure prices are fresh.

DOS on `getAllocatedSets()`

**Recommendation**


Add pagination to limit loop iterations per call. Restrict how often operators can call `modifyAllocations()` (e.g., cooldown periods).
