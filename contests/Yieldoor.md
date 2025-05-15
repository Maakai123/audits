## Yieldoor
[Contest Details](https://audits.sherlock.xyz/contests/791/report)

### [Medium-01] Incorrect use of token0 balance for token1 Debt repayment

**Summary**

In the Leverager contract's `withdraw()` function, there's a typo that causes the wrong token balance to be used when repaying a debt denominated in `token1`. Instead of using `amountOut1` (the `token1` balance), the code mistakenly uses `amountOut0`.

```solidity
else if (borrowed == up.token1) {  
    uint256 repayFromWithdraw = amountOut1 < owedAmount ? amountOut0 : owedAmount; // ❌ Typo  
    // ...  
} 
```

**Vulnerability Impact**

The bug arises from an incorrect conditional check in the token1 repayment branch. When the borrowed token is `token1`, the code erroneously references `amountOut0` instead of `amountOut1`, which leads to an inaccurate calculation of the repayment amount. This mistake can result in a misalignment between the actual token balances and the amount of debt being repaid.

**Impact**

Borrowers might end up repaying too little or too much debt relative to their actual `token1` balance.

**Recommendation**

```solidity
else if (borrowed == up.token1) {  
    uint256 repayFromWithdraw = amountOut1 < owedAmount ? amountOut1 : owedAmount; // ✅  
    // ...  
}  
```
**Code Snippets**

https://github.com/sherlock-audit/2025-02-yieldoor/blob/main/yieldoor/src/Leverager.sol#L226-L235