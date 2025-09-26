## solaxy
[Contest details](https://cantina.xyz/code/50d38b86-80a0-49af-9df8-70d7d601b7d7/overview)

### [High-01] Missing Account-Owner Index Synchronization in storage.rs


**Summary**
Account storage operations in storage.rs (set_account, remove_account) don't update the owner index, causing data inconsistency between the accounts map and owner index that can lead to incorrect program account enumeration and potential access control issues.

**Finding Description**

The AccountsDB implementation maintains two separate data structures: an accounts map (BTreeMap<Pubkey, Account>) and an owner index (BTreeMap<Pubkey, Vec>). The owner index tracks which accounts belong to each owner for efficient lookups via get_program_accounts(). However, the set_account() and remove_account() methods only modify the accounts map without updating the corresponding owner index entries. This breaks the fundamental invariant that the owner index should accurately reflect the current state of account ownership.

**Impact**
This affects core data integrity guarantees. The owner index is used by get_program_accounts() which is likely consumed by other components for access control, account discovery, or program management. Stale index data could:

Return non-existent or incorrect accounts to dependent systems Miss legitimate accounts that should be included in operations Enable privilege escalation if access control relies on owner enumeration Cause runtime failures when attempting to access accounts that no longer exist


**Recommendation**
Implement transactional updates that maintain consistency between accounts and owner index:
