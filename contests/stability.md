## Stability DAO
[Contest details](https://cantina.xyz/competitions/e1c0be8d-0c3d-485a-a446-a582beb120b1)

### [Medium-01] New vault addition could freeze all operations

**Description**

When a new child vault is added via `addVault`, its `totalSupply` remains zero. The MetaVault.currentProportions() function divides by each vault’s `totalSupply` to compute USD values:

```solidity
vaultUsdValue[i] = vaultSharesBalance × vaultTvl / vaultTotalSupply;
```
Because v`aultTotalSupply == 0` for the fresh vault, every call to `currentProportions()` reverts with a divide-by-zero error. 

As a result, all higher-level actions (deposits, withdrawals, rebalances) immediately fail, trapping user funds and halting the protocol until someone manually “seeds” the new vault outside the normal UI paths.

**Recommendation**

In c`urrentProportions()`, skip or zero out any vault whose totalSupply is zero