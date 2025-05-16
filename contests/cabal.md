## Cabal Liquid Staking Token
[Contest Details](https://code4rena.com/audits/2025-04-cabal-liquid-staking-token)

### [Medium-01] Insufficient slippage protection in LP reward compounding

**Finding description and impact**

The `compound_lp_pool_rewards` function in `cabal.move` uses a minimal slippage parameter (`option::some<u64>(1)`) in `dex::single_asset_provide_liquidity`, making it vulnerable to front-running attacks.

Attackers can manipulate the DEX pool’s price via sandwich attacks, reducing the LP tokens minted and diminishing compounded rewards for users

The `dex::single_asset_provide_liquidity` call sets `min_liquidity = 1`, allowing the transaction to succeed with minimal LP tokens minted, even under severe price manipulation

```move
let lp = dex::single_asset_provide_liquidity(
    object::convert<Metadata, Config>(m_store.stake_token_metadata[pool_index]),
    reward_fa,
    option::some<u64>(1)
);
```

**Recommended mitigation steps**

Implement dynamic slippage protection by calculating an expected min_liquidity based on the current pool reserves and reward_amount. This ensures `dex::single_asset_provide_liquidity` reverts if the minted LP tokens fall below a reasonable threshold, preventing front-running losses