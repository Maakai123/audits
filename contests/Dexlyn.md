## Dexlyn
[Contest Details](https://dashboard.hackenproof.com/user/reports/DEXLYNCA-60)

### [Criticsl-01]  `repay_flash_swap` token-identity bypass (direct theft)

### Vulnerability details

Affected function: repay_flash_swap in DexlynClmm/sources/pool.move

The repay_flash_swap function verifies only the repaid amounts and never validates the token identity (metadata) of the repayment. It then deposits the repayment into a primary store at the pool address that accepts any FungibleAsset metadata.

An attacker can flash-borrow real tokens from the pool and “repay” with a different, worthless token bearing wrong metadata. The pool’s internal counters increase, but the legitimate store for the real asset remains short.

Impact: Direct theft of the flash-borrowed real asset; subsequent legitimate withdrawals fail due to insufficient real balance despite inflated accounting, effectively bricking the pool until fixed.

Root cause

repay_flash_swap verifies only the pay_amount but not the identity of the repaid token.
It calls primary_fungible_store::deposit(pool_address, …) which can create/use a primary store for any FA metadata at the pool’s address. This credits the wrong asset while the legitimate asset store remains underfunded.

Vulnerable code (no metadata check, unconditional deposit):

pool.move

Branch when a2b is true (repay in asset_a):

```rust
assert!(fungible_asset::amount(&asset_a) == pay_amount, EAMOUNT_INCORRECT);
primary_fungible_store::deposit(pool_address, asset_a);
pool.asset_a = pool.asset_a + pay_amount - ref_fee_amount;
```

Branch when a2b is false (repay in asset_b):


```rust
assert!(fungible_asset::amount(&asset_b) == pay_amount, EAMOUNT_INCORRECT);
primary_fungible_store::deposit(pool_address, asset_b);
pool.asset_b = pool.asset_b + pay_amount - ref_fee_amount;
There is no assertion tying asset_a/asset_b metadata to pool.asset_a_addr / pool.asset_b_addr before deposit.
```
Impact

Direct theft: Attacker can keep the real asset received during flash_swap while repaying with a worthless token.

Pool lock: Later legitimate withdrawals relying on the real store will fail, potentially bricking liquidity.

Attacker calls flash_swap with a2b=false to receive real asset A from the pool.
The pool signer’s legitimate store for A is reduced by amount_out.

Attacker keeps the real asset A (deposits into their own primary store).

Attacker mints a fake asset “WrongB” under arbitrary metadata and repays via repay_flash_swap with this wrong token.

Amount equals swap_pay_amount(&receipt).

repay_flash_swap accepts the wrong token (only checks amount), deposits it at pool_address, and increases pool.asset_b.
The real store for A remains short; the fake B store is credited instead.

A later attempt to withdraw/flash-swap real A will fail or cause the next legitimate withdraw to fail due to insufficient real balance in the pool signer’s store. The POC asserts this by trying to withdraw remaining+1 from the real A store and expecting an abort.


### [Criticsl-02] Arbitrary asset withdrawal via unvalidated asset_addr in collect_rewarder

Vulnerability details

dexlyn_clmm::clmm_router::collect_rewarder passes a caller-supplied asset_addr through to pool.

dexlyn_clmm::pool::collect_rewarder withdraws from the pool’s primary fungible store using the unvalidated asset_addr.

Root cause:

Missing binding between rewarder_index and the configured rewarder asset address. The function trusts the user-provided asset_addr instead of resolving the rewarder’s asset from pool state.

clmm_router: asset_addr is accepted from the user and forwarded to pool::collect_rewarder.

```rust
    let rewarder_asset = pool::collect_rewarder(
        account,
        pool_address,
        pos_index,
        rewarder_index,
        true,
        asset_addr
    );
```
pool: caller-provided asset_addr determines which asset is withdrawn; amount_owed is reset after withdrawal.

```rust
 let amount = &mut vector::borrow_mut(&mut position.rewarder_infos, (rewarder_index as u64)).amount_owed;
    let asset_metadata = object::address_to_object<Metadata>(asset_addr);
    let rewarder_asset = primary_fungible_store::withdraw(
        &pool_signer, asset_metadata, *amount
    );
    *amount = 0;
```
The rewarder subsystem accrues an amount_owed per position, per rewarder index. When collecting, pool::collect_rewarder uses a user-provided asset_addr to choose which asset to withdraw from the pool’s primary fungible store, rather than deriving the asset from the rewarder configuration. Because the router forwards the asset_addr unchecked, a user can select any pool-held asset (e.g., the pool’s asset A) and receive that asset as the “reward,” even if the rewarder at rewarder_index is configured for another asset (e.g., asset B). The function then zeroes amount_owed, finalizing the diversion.

Impact:

An LP can collect “rewards” in any token currently held by the pool’s primary store by choosing that token’s address as asset_addr, diverting funds that are not the configured rewarder asset. This can drain pool-held principal (e.g., asset A/B) up to amount_owed and desynchronize accounting.


### [Criticsl-03] `repay_add_liquidity` never verifies the token type leading to DOS 

Vulnerability details
Function affected:
repay_add_liquidity in DexlynClmm/sources/pool.move

The function asserts only the repayment amounts and never verifies the token type (metadata) being repaid matches the pool’s expected assets.

The pool accepts the repayment into any fungible-asset store under its address (even a brand new, unrelated token). Internal counters (pool.asset_a / pool.asset_b) increase, but the pool’s real token stores remain underfunded.

Impact:
Withdrawals/collects that try to withdraw the legitimate tokens abort due to insufficient balance and Liquidity or fees can be stuck.

In repay_add_liquidity the code checks amounts but not the metadata of asset_a/asset_b.

It then calls primary_fungible_store::deposit(pool_address, ...), which will create or use a primary store for whatever metadata the caller supplied.

Result: Counters updated for “A” and “B”, but the deposited tokens could be a different FA, leaving the legitimate pools’ stores short.

Issue from pool.move

```rust
assert!(fungible_asset::amount(&asset_a) == amount_a, EAMOUNT_INCORRECT);
primary_fungible_store::deposit(pool_address, asset_a);
pool.asset_a = pool.asset_a + amount_a;
```
Note:
Attackers can brick a pool (DoS) so LPs cannot withdraw. No direct token payout via this path; it’s a lock/grief attack.
Cost to attacker: Essentially gas-only.
They mint their own worthless FA and repay with exactly the required amounts from the receipt.
They don’t have to acquire real tokens.

Immediately after wrong-token repay: pool.asset_a/pool.asset_b go up as if funds arrived.

When removing liquidity or collecting with the real asset metadata, the pool attempts to withdraw from its legitimate FA store and aborts due to insufficient balance.
