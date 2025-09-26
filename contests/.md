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





### [MEDIUM-02] Oracle Staleness Vulnerability in Borrow Logic

**Summary**

The Aave borrow logic lacks staleness validation for oracle price feeds, allowing users to borrow against outdated collateral prices. This vulnerability enables over-borrowing and manipulation of health factors, posing a significant risk of protocol insolvency during periods of price volatility.


**Finding Description**

The vulnerability stems from the absence of timestamp validation in the oracle's price retrieval mechanism, specifically within the get_asset_price_internal() function. This function retrieves asset prices from Chainlink but does not check the freshness of the price data.

Entry Point: The borrow_logic::borrow() function initiates a borrow operation and calls internal_borrow().

Validation: internal_borrow() invokes validation_logic::validate_borrow(), which relies on oracle::get_asset_price() to assess collateral and borrow amounts.

Price Usage: Both validate_borrow() and generic_logic::calculate_user_account_data() (called within it) use these potentially stale prices to compute collateral values and health factors, enabling unsafe borrowing.



## Impact Explanation

Borrowing against the stale, inflated collateral value, exceeding safe limits.


## Recommendation
To mitigate this vulnerability, implement staleness validation in the oracle's price retrieval logic. The following steps are recommended:

Timestamp Validation: Check the price timestamp against the current time, rejecting prices beyond a staleness threshold (e.g., 1 hour).





### [LOW-01] Debt Ceiling Precision issues in validation_logic.move

**Summary**

A precision error in the Aave V3 protocol’s isolation mode debt ceiling calculation allows users to borrow slightly more than the intended debt ceiling due to integer division truncating fractional amounts, violating the specification’s requirement to round up. This affects validation and debt tracking logic, making the debt ceiling a “soft” limit.


**Finding Description**

The bug exists in the debt ceiling validation and debt tracking logic for isolation mode borrowing in the validation_logic.move The issue stems from using integer division (/) in Move, which rounds down, instead of ceiling division (rounding up), as required by the protocol specification. This allows borrowing amounts just above the debt ceiling to pass validation due to truncation of fractional amounts.

Affected Codes

1).Validation Logic (aave-core/sources/aave-logic/validation_logic.move:280-291):

let total_debt =
    isolation_mode_total_debt
        + (
            amount
                / math_utils::pow(
                    10,
                    (vars.reserve_decimals - reserve_config::get_debt_ceiling_decimals())
                )
        );
assert!(total_debt <= isolation_mode_debt_ceiling, error_config::get_edebt_ceiling_exceeded());
The amount is divided by 10^(reserve_decimals - debt_ceiling_decimals), and the result is truncated to an integer, underestimating the debt.

2). Debt Tracking Logic (aave-core/sources/aave-logic/borrow_logic.move:270-275):

let decimals = reserve_config::get_decimals(&reserve_configuration_map)
    - reserve_config::get_debt_ceiling_decimals();
let next_isolation_mode_total_debt =
    (isolation_mode_total_debt as u256)
        + (amount / math_utils::pow(10, decimals));
     ```
     The same integer division is used, consistently undertracking the debt.
     
 3). Debt Ceiling Decimals (aave-core/aave-config/sources/reserve_config.move:79):
 
 ```rust
 const DEBT_CEILING_DECIMALS: u256 = 2;
public fun get_debt_ceiling_decimals(): u256 {
    DEBT_CEILING_DECIMALS
}
The debt ceiling is expressed with 2 decimal places, so a ceiling of 100000 represents 1000.00 USD.

Debt Ceiling Enforcement: The specification requires that the debt ceiling be a hard limit, with calculations rounding up to prevent any excess borrowing. The use of integer division violates this by allowing borrowing up to 10^(reserve_decimals - 2) - 1 units per transaction without detection.

Precision Integrity: The protocol fails to accurately track debt in isolation mode, as fractional amounts are truncated, leading to underreporting of debt and incorrect validation.

A malicious user can exploit this by borrowing an amount slightly above the debt ceiling, crafted to exploit the truncation:

For an asset with 18 decimals (e.g., ETH), borrowing 1000.000000999999999999 USD (i.e., 1,000,000,000,999,999,999,999,999 wei) with a debt ceiling of 1000.00 USD (100000 in 2 decimals):

The calculation divides by 10^(18-2) = 10^16: 1,000,000,000,999,999,999,999,999 / 10^16 = 100.000000999999999999 (truncated to 100).

Validation checks total_debt <= 100000, which passes (100 <= 100000), allowing the borrow.

The actual borrowed amount (1000.000000999999999999 USD) exceeds the ceiling (1000.00 USD).

This can be repeated across multiple transactions, accumulating significant excess borrowing.



## Impact Explanation
Exploitable but Limited: The bug allows borrowing slightly above the debt ceiling (up to 10^(reserve_decimals - 2) - 1 units per transaction). For ETH (18 decimals), this is 10^16 - 1 wei ($0.01 at $3000/ETH); for BTC (8 decimals), it’s 10^6 - 1 satoshis ($25 at $50,000/BTC).

Cumulative Risk: Multiple transactions can accumulate significant excess (e.g., 10 transactions of 10^16 - 1 wei = 99,999,999,999,999,990 wei, or ~$0.10 for ETH).



## Recommendation

To fix the bug, replace integer division with ceiling division in both validation_logic.move and borrow_logic.move to ensure fractional amounts are rounded up, aligning with the specification. This prevents any excess borrowing and ensures accurate debt tracking.




### [LOW-02] Integer Division Truncation Bug in Isolation Mode Logic

**Summary**

A bug in the update_isolated_debt function of the isolation mode logic module causes small repayment amounts to be truncated to zero due to integer division, failing to reduce the isolation_mode_total_debt and compromising debt ceiling accuracy.


**Finding Description**
In the aave_pool::isolation_mode_logic module, the update_isolated_debt function calculates the repaid isolated debt with the line:

let isolated_debt_repaid = (repay_amount / math_utils::pow(10, debt_decimals));
Issue: Integer division (/) truncates the result to zero when repay_amount is less than math_utils::pow(10, debt_decimals). The debt_decimals value is computed as reserve_config::get_decimals(&reserve_config_map) - reserve_config::get_debt_ceiling_decimals(), where DEBT_CEILING_DECIMALS is hardcoded to 2 in reserve_config.move.

Security Guarantees Broken:

Debt Tracking Integrity: The isolation mode is designed to limit borrowing by tracking isolation_mode_total_debt against a debt ceiling (in units of 0.01, given 2 decimals). Truncation prevents small but valid repayments from reducing this debt, leading to inaccurate debt tracking.

Protocol Safety: Incorrect debt values undermine the debt ceiling’s role in restricting risky borrowing, potentially allowing the protocol to misjudge user positions during repayments or liquidations.

A user or malicious actor initiates a repayment via borrow_logic::update_isolated_debt_if_isolated or a liquidation via liquidation_logic, passing a repay_amount (e.g., 10^15 base units) to update_isolated_debt.

The function retrieves the reserve configuration and computes debt_decimals (e.g., 18 - 2 = 16 for an asset with 18 decimals).

It calculates isolated_debt_repaid = repay_amount / math_utils::pow(10, debt_decimals). If repay_amount < 10^16, the result truncates to 0.




## Impact Explanation

Systemic Scope: The bug affects both regular repayments (called via borrow_logic) and liquidations (called via liquidation_logic), impacting all users in isolation mode.

Debt Ceiling Misalignment: Small repayments (e.g., 0.009 ETH or 0.0001 USDC, depending on decimals) are ignored, causing isolation_mode_total_debt to overstate actual debt. This can mislead borrowing limits, liquidation thresholds, and risk assessments.

Protocol Risk: Inaccurate debt tracking weakens the isolation mode’s purpose of capping exposure, potentially increasing financial risk to the Aave protocol.



## Recommendation

To fix the bug, replace integer division with ceiling division in both validation_logic.move and borrow_logic.move to ensure fractional amounts are rounded up, aligning with the specification. This prevents any excess borrowing and ensures accurate debt tracking.




### [LOW-03] Oracle Staleness Issue in Aave Aptos V3 Oracle Implementation

**Summary**

The Aave Aptos V3 Oracle implementation lacks an Oracle Sentinel module, which is critical for handling oracle staleness and downtime protection. Despite being referenced in documentation and configuration, the absence of this module results in no validation of price feed freshness, no downtime detection, and no grace period after oracle recovery. This vulnerability could allow the protocol to use stale or unreliable price data, leading to incorrect asset valuations, improper liquidations, or unauthorized borrowing.


**Finding Description**

The Aave Oracle module (aave_oracle::oracle) is designed to provide price feeds for the Aave protocol using Chainlink data or custom prices. However, it lacks the Oracle Sentinel, a critical component documented to manage oracle health by enforcing staleness checks, downtime detection, and a grace period for user position adjustments after oracle recovery. The absence of this module breaks the following security guarantees:

Data Freshness: The protocol does not validate the timestamp of Chainlink price feed data, allowing the use of stale prices that may not reflect current market conditions.

Downtime Protection: There is no mechanism to detect oracle downtime or pause operations (e.g., borrowing or liquidation) when price data is unreliable.

User Protection: Without a grace period after oracle recovery, users may face immediate liquidations without time to adjust their positions, violating fairness guarantees.

How the Issue Manifests

The issue arises because the get_asset_price_internal function in the Oracle module retrieves prices without checking the freshness of Chainlink data:


## Impact Explanation

Financial Risk: Stale or unreliable price data could lead to incorrect asset valuations, resulting in:

Over-borrowing if prices are artificially high.

Unfair liquidations if prices are artificially low.




## Recommendation
Implement Oracle Sentinel Module: Create a new oracle_sentinel module with functions to validate oracle health and manage operations during downtime or recovery.

Add Staleness Checks: Modify get_asset_price_internal to validate Chainlink feed timestamps against a configurable staleness threshold.












