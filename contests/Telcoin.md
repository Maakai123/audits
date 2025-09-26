## telcoin-network
[Contest Details]([https://cantina.xyz/competitions/aaf79192-6ea7-4b1e-aed7-3d23212dd0f1](https://cantina.xyz/code/26d5255b-6f68-46cf-be55-81dd565d9d16/overview))


### [Medium-01] Base Fee Validation Bypass

**Description**

Base fee validation bypass vulnerability in BatchValidator::validate_basefee() allows attackers to submit batches with None base fees, completely bypassing economic validation and potentially enabling free transaction processing.

The vulnerability exists in the batch validation system where the validate_basefee() method validator.rs:193-201 only performs validation when base_fee_per_gas is Some(value) but silently accepts None values without any validation.

This breaks the economic security guarantee that all batches must pay appropriate base fees. The attack propagates as follows:

Attacker constructs a malicious Batch with base_fee_per_gas: None During batch validation, validate_basefee() is called validator.rs:76 The method's conditional logic skips validation entirely for None values and returns Ok(()) The batch passes validation and gets processed with zero base fee via unwrap_or_default() This undermines the fee mechanism designed to prevent spam and ensure economic sustainability of the network.

**Impact**
This vulnerability directly compromises the economic security model of the network. Attackers can:

Submit unlimited transactions without paying base fees Spam the network with zero-cost batches Undermine the fee market mechanism Potentially cause economic attacks against validators who expect fee revenue The impact is systemic as it affects the core economic incentive structure that keeps the network secure and sustainable.

**Recommendation**
Modify validate_basefee() to require base fees to always be present and valid:

```rust
fn validate_basefee(&self, base_fee: Option<u64>) -> BatchValidationResult<()> {  
    let base_fee = base_fee.ok_or(BatchValidationError::MissingBaseFee)?;  
    let expected_base_fee = self.base_fee.base_fee();  
    if base_fee != expected_base_fee {  
        return Err(BatchValidationError::InvalidBaseFee { expected_base_fee, base_fee });  
    }  
    Ok(())  
}
```
