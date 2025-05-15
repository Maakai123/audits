## Tally
[Contest details](https://cantina.xyz/competitions/8cb347d2-53b5-4fd0-bbbd-3f67537f2bb0)

### [Low-01] quorum in `_getProposalDetails()` is using current blocktimestamp gotten from `clock()` instead of proposal's snapshot

**Summary**
The `AutoDelegateOpenZeppelinGovernor` contract incorrectly calculates the quorum for governance proposals by using the current blocktimestamp gotten from `clock()` value instead of the proposal's snapshot timepoint. This allows attackers to manipulate governance outcomes by minting/burning tokens or changing their delegation after a proposal is created

**Finding Description**
The `_getProposalDetails()` function calculates the quorum for a proposal using the current `clock()` value instead of the proposal's snapshot timepoint. This violates the governance system's assumption that quorum calculations should be based on the state of the system at the time the proposal was created

```solidity
function _getProposalDetails(address _governor, uint256 _proposalId)
  internal
  view
  virtual
  returns (uint256 _proposalDeadline, uint256 _forVotes, uint256 _againstVotes, uint256 _quorumVotes)
{
  _proposalDeadline = IGovernor(_governor).proposalDeadline(_proposalId);
  (_againstVotes, _forVotes,) = IGovernorCountingExtensions(_governor).proposalVotes(_proposalId);
  _quorumVotes = IGovernor(_governor).quorum(clock()); // <-- BUG: Uses current clock()
}
```
The quorum should reflect the voting power at the time of the proposal's creation. Using the current state allows attackers to manipulate the quorum by changing token balances or delegations after the proposal is created. Proposals could be wrongly approved or rejected based on incorrect quorum calculations, which could affect the fairness of the governance process.

**Attack Scenario**

Alice creates a proposal at block `1000`. The snapshot of voting power is taken at block `1000`:

Total voting power: `150 (Alice: 100, Bob: 50)`.
Quorum requirement: 50% of total voting power = 75.
After the proposal is created, Bob mints 100 new tokens at block 1001. Total voting power is now `250 (Alice: 100, Bob: 150)`.

The autoDelegate calls `_getProposalDetails` at block `1002`. The quorum is calculated using clock(), which returns the current block (`1002`):

Quorum = `50% of 250 = 125`. -The proposal only has 100 "For" votes (Alice’s votes), so it fails the quorum check.

The proposal is wrongly rejected because the quorum check was based on the current state (`250` voting power) instead of the snapshot state (`150` voting power).

**Recommendation**
Modify the quorum calculation to use the proposal's snapshot block instead of the current block returned by `clock()`. Update the `_getProposalDetails()` function to pass the proposal's snapshot block to `quorum()`