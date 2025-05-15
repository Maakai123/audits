## Kinetiq
[Contest Details](https://code4rena.com/audits/2025-04-kinetiq)

### [Medium-01] Griefing Attack via L1 Operation Queue

**Finding description and impact**
The `StakingManager` contract is vulnerable to a greifing attack due to lack of limits on the size of `_pendingDeposits` and `_pendingWithdrawal` queues, which stores L1Operation structs for deposits and withdrawal operations respectively.

These queues are populated by public functions `stake()` and `queueWithdrawal()` accessible to any user(or whitelisted users if whitelistEnabled is true) and by operator admin functions like `queueL1Operations()`, and `redelegateWithdrawnHYPE()`.

The `processL1Operations()` called by operators, processes these queues by executing gas-intensive external calls to the L1Write contract (e.g., `sendTokenDelegate()`, `sendCDeposit()`, `sendCWithdrawal()`), with each operation costing approximately 25,000–40,000 gas due to external calls, state updates, and event emissions

The absence of a cap on queue size allows an attacker to flood the queues with numerous small operations, such as minimal stakes (e.g., `minStakeAmount`) or withdrawals, by repeatedly calling `stake()` and `queueWithdrawal()`

With no upper bound, an attacker can generate thousands of operations, significantly increasing the gas cost of `processL1Operations()` and causing a griefing attack and DOS. For example, processing 10,000 withdrawals (~40,000 gas each) requires ~400M gas, far exceeding Ethereum’s block gas limit (~30M gas as of April 2025)

While operators can process batches using the batchSize parameter, clearing large queues requires multiple transactions over hours, imposing a high cost and delaying critical operations.

**Attack Path**

1. The attacker repeatedly calls `stake()` with the minimum allowed amount (`minStakeAmount`) to queue numerous `UserDeposit` operations in `_pendingDeposits`.
2. Alternatively, the attacker queues multiple small withdrawals via `queueWithdrawal()`, adding UserWithdrawal operations to `_pendingWithdrawals`.
3. Each operation requires an L1 call (e.g., `sendCDeposit()` or `sendTokenDelegate()`), increasing the gas cost of `processL1Operations()`.
4. If the queue grows large enough, the gas cost exceeds the block gas limit (e.g., 30M on Ethereum), preventing operators from processing the queue, stalling the system.

**Recommended mitigation steps**
To address the griefing attack, the best approach combines a queue size limit with a dynamic minimum operation size to prevent unbounded queue growth and deter small, spammy operations while maintaining usability for legitimate users

