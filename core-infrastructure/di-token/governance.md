# Governance

DI Network operates as a decentralized autonomous organization (DAO) where DI token holders collectively govern the protocol through on-chain voting and proposal mechanisms.

## Governance Overview

### Governance Principles
- **Decentralization**: Community-driven decision making
- **Transparency**: All proposals and votes are public
- **Security**: Time delays and emergency procedures
- **Participation**: Incentivized community engagement

### Governance Scope
- Protocol parameter adjustments
- Smart contract upgrades
- Treasury fund allocation
- New asset listings
- Fee structure changes
- Cross-chain expansion

## Voting System

### Voting Power Calculation
```
Voting Power = DI Balance × Staking Multiplier × Delegation Factor
```

### Staking Multipliers
| Stake Duration | Voting Multiplier |
|----------------|-------------------|
| No Staking | 1.0x |
| 3 months | 1.2x |
| 6 months | 1.5x |
| 12 months | 2.0x |
| 24 months | 2.5x |

### Delegation System
```javascript
// Delegate voting power
await governance.delegate(delegateAddress)

// Self-delegate to vote directly
await governance.delegate(userAddress)

// Check delegation
const delegate = await governance.delegates(userAddress)
```

## Proposal Process

### Proposal Types
1. **Parameter Changes**: Adjust protocol parameters
2. **Contract Upgrades**: Deploy new contract versions
3. **Treasury Spending**: Allocate community funds
4. **Asset Listings**: Add new synthetic assets
5. **Emergency Actions**: Critical protocol changes

### Proposal Lifecycle
```
1. Discussion (7 days) → 2. Proposal (3 days) → 3. Voting (7 days) → 4. Execution (2 days)
```

### Proposal Requirements
- **Minimum Tokens**: 100,000 DI to create proposal
- **Quorum**: 10% of circulating supply must vote
- **Approval**: >50% of votes must be "Yes"
- **Timelock**: 48-hour delay before execution

## Governance Contracts

### Governor Contract
```solidity
interface IGovernor {
    function propose(
        address[] memory targets,
        uint256[] memory values,
        bytes[] memory calldatas,
        string memory description
    ) external returns (uint256 proposalId);
    
    function castVote(uint256 proposalId, uint8 support) external;
    function execute(uint256 proposalId) external;
    function cancel(uint256 proposalId) external;
}
```

### Timelock Controller
```solidity
interface ITimelock {
    function schedule(
        address target,
        uint256 value,
        bytes calldata data,
        bytes32 predecessor,
        bytes32 salt,
        uint256 delay
    ) external;
    
    function execute(
        address target,
        uint256 value,
        bytes calldata data,
        bytes32 predecessor,
        bytes32 salt
    ) external;
}
```

## Proposal Examples

### Parameter Change Proposal
```javascript
// Propose to change trading fee from 0.3% to 0.25%
const targets = [DSWAP_ADDRESS]
const values = [0]
const calldatas = [
  dswap.interface.encodeFunctionData("setSwapFee", [25]) // 0.25%
]
const description = "Reduce DSwap trading fee to 0.25% to increase competitiveness"

await governor.propose(targets, values, calldatas, description)
```

### Treasury Spending Proposal
```javascript
// Propose to fund development grant
const targets = [TREASURY_ADDRESS]
const values = [parseEther("50000")] // 50,000 DI
const calldatas = [
  treasury.interface.encodeFunctionData("transfer", [
    grantRecipient,
    parseEther("50000")
  ])
]
const description = "Fund Q2 development grant for cross-chain expansion"

await governor.propose(targets, values, calldatas, description)
```

## Voting Process

### Casting Votes
```javascript
// Vote on proposal
await governor.castVote(proposalId, 1) // 0=Against, 1=For, 2=Abstain

// Vote with reason
await governor.castVoteWithReason(
  proposalId, 
  1, 
  "Supporting fee reduction to improve adoption"
)

// Check vote receipt
const receipt = await governor.getReceipt(proposalId, voterAddress)
```

### Vote Delegation
```javascript
// Delegate to expert voter
await diToken.delegate(expertVoterAddress)

// Check current delegate
const currentDelegate = await diToken.delegates(userAddress)

// Get voting power
const votingPower = await diToken.getVotes(userAddress)
```

## Governance Incentives

### Participation Rewards
- **Voting Rewards**: 0.1% APY bonus for active voters
- **Proposal Rewards**: 1000 DI for successful proposals
- **Delegation Fees**: Delegates earn 5% of delegator rewards

### Participation Tracking
```javascript
const getGovernanceStats = async (userAddress) => {
  const stats = await governance.getUserStats(userAddress)
  
  return {
    proposalsCreated: stats.proposalsCreated,
    votescast: stats.votesCast,
    participationRate: stats.participationRate,
    rewardsEarned: formatEther(stats.rewardsEarned),
    votingPower: formatEther(stats.votingPower)
  }
}
```

## Emergency Procedures

### Guardian System
- **Emergency Guardian**: Can pause protocol in emergencies
- **Guardian Council**: 5-member multisig for critical decisions
- **Community Override**: Governance can remove guardian powers

### Emergency Actions
```solidity
// Emergency pause (Guardian only)
function emergencyPause() external onlyGuardian {
    _pause();
    emit EmergencyPause(msg.sender, block.timestamp);
}

// Community override (Governance only)
function removeGuardian() external onlyGovernance {
    guardian = address(0);
    emit GuardianRemoved(block.timestamp);
}
```

## Governance Analytics

### Proposal Metrics
```javascript
const GovernanceAnalytics = () => {
  const { data: proposals } = useQuery(['proposals'], getProposals)
  
  const metrics = {
    totalProposals: proposals.length,
    passedProposals: proposals.filter(p => p.status === 'Executed').length,
    averageParticipation: proposals.reduce((sum, p) => sum + p.participation, 0) / proposals.length,
    activeVoters: new Set(proposals.flatMap(p => p.votes.map(v => v.voter))).size
  }
  
  return (
    <div className="governance-analytics">
      <div>Total Proposals: {metrics.totalProposals}</div>
      <div>Success Rate: {(metrics.passedProposals / metrics.totalProposals * 100).toFixed(1)}%</div>
      <div>Avg Participation: {metrics.averageParticipation.toFixed(1)}%</div>
      <div>Active Voters: {metrics.activeVoters}</div>
    </div>
  )
}
```

### Voting History
```javascript
const VotingHistory = ({ userAddress }) => {
  const { data: votes } = useQuery(['votes', userAddress], () => getUserVotes(userAddress))
  
  return (
    <div className="voting-history">
      {votes?.map(vote => (
        <div key={vote.proposalId} className="vote-item">
          <div>Proposal #{vote.proposalId}</div>
          <div className={`vote-${vote.support}`}>
            {vote.support === 1 ? 'FOR' : vote.support === 0 ? 'AGAINST' : 'ABSTAIN'}
          </div>
          <div>Power: {formatNumber(vote.votingPower)}</div>
        </div>
      ))}
    </div>
  )
}
```

## Integration Examples

### Governance Dashboard
```javascript
const GovernanceDashboard = () => {
  const { data: activeProposals } = useQuery(['activeProposals'], getActiveProposals)
  const { data: userStats } = useQuery(['userStats'], getUserGovernanceStats)
  
  return (
    <div className="governance-dashboard">
      <div className="user-stats">
        <div>Voting Power: {formatNumber(userStats.votingPower)} DI</div>
        <div>Participation Rate: {userStats.participationRate}%</div>
        <div>Rewards Earned: {formatNumber(userStats.rewards)} DI</div>
      </div>
      
      <div className="active-proposals">
        {activeProposals?.map(proposal => (
          <ProposalCard key={proposal.id} proposal={proposal} />
        ))}
      </div>
    </div>
  )
}
```

### Proposal Creation
```javascript
const CreateProposal = () => {
  const [targets, setTargets] = useState([''])
  const [values, setValues] = useState(['0'])
  const [calldatas, setCalldatas] = useState([''])
  const [description, setDescription] = useState('')
  
  const submitProposal = async () => {
    await governor.propose(targets, values, calldatas, description)
  }
  
  return (
    <form onSubmit={submitProposal}>
      <input 
        placeholder="Proposal description"
        value={description}
        onChange={(e) => setDescription(e.target.value)}
      />
      {/* Target contracts, values, and calldata inputs */}
      <button type="submit">Submit Proposal</button>
    </form>
  )
}
```

## Best Practices

### For Token Holders
- Stay informed about proposals
- Participate in community discussions
- Vote on all relevant proposals
- Consider delegation if unable to participate actively

### For Delegates
- Maintain transparency about voting decisions
- Engage with delegators regularly
- Provide reasoning for votes
- Act in the best interest of the protocol

### For Proposal Creators
- Engage community before formal proposal
- Provide detailed implementation plans
- Consider economic impact
- Allow sufficient discussion time