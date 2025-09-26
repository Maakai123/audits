## GTE
[Contest Details](https://code4rena.com/audits/2025-07-gte-spot-clob-and-router)

### [Medium-01] Bitwise Logic Error in CLOB Zero-Cost Trade Protection Enables Value Extraction

**Finding description and impact**
The CLOB contract contains a critical vulnerability in its zero-cost trade protection mechanism within the _processLimitBidOrder() and _processLimitAskOrder() functions. The vulnerable code uses a bitwise AND operator (&) instead of a logical AND operator (&&) in the protection condition:

```rust
if (baseTokenAmountReceived != quoteTokenAmountSent && baseTokenAmountReceived & quoteTokenAmountSent == 0) {
    revert ZeroCostTrade();
}
```
The condition is intended to prevent zero-cost trades by reverting when:

The amounts are different (baseTokenAmountReceived != quoteTokenAmountSent)
One of the amounts is zero (intended logic)
However, due to the bitwise AND operation, the protection only triggers when the bitwise AND of the two amounts equals zero, meaning the amounts share no common set bits in their binary representation.

Protection Bypass: When two different token amounts share common bits in their binary representation, the protection mechanism is completely bypassed, allowing unfair trades.
Mathematical Proof:
Example: baseReceived = 3 (binary: 011) and quoteSent = 1 (binary: 001)
Bitwise operation: 3 & 1 = 1 ≠ 0
Result: Protection does NOT trigger, allowing 3 tokens to be received for only 1 token
Economic Impact:
Immediate Loss: Attackers can extract significant value per transaction


**Recommended mitigation steps**
Replace with Proper Zero-Amount Check
Replace the vulnerable bitwise condition with a direct zero-amount validation:

```rust
// Current vulnerable code:
if (baseTokenAmountReceived != quoteTokenAmountSent && baseTokenAmountReceived & quoteTokenAmountSent == 0) {
    revert ZeroCostTrade();
}

// Recommended fix:
if (baseTokenAmountReceived == 0 || quoteTokenAmountSent == 0) {
    revert ZeroCostTrade();
}

```
