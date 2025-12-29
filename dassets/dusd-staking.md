# DUSD Staking System

The DUSD Staking system serves as the economic backbone of the DI Synthetic Asset Protocol, providing protocol security, revenue sharing, and risk backstop functionality for the entire DAssets ecosystem.

## Overview

DUSD Staking enables users to:
- Stake DUSD tokens for protocol rewards
- Provide economic security as protocol backstop
- Earn revenue from trading fees and liquidations
- Participate in governance decisions
- Access tiered reward multipliers based on lock periods

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    DUSD Staking System                      │
├─────────────────┬─────────────────┬─────────────────────────┤
│  Staking Vault  │ Reward Engine   │   Risk Management       │
│                 │                 │                         │
│ • Lock periods  │ • Fee collection│ • Insurance fund        │
│ • Multipliers   │ • Distribution  │ • Bad debt coverage     │
│ • Governance    │ • Compounding   │ • Loss waterfall        │
└─────────────────┴─────────────────┴─────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                   Revenue Sources                            │
├─────────────────┬─────────────────┬─────────────────────────┤
│  Trading Fees   │ Liquidation Fees│    Protocol Revenue     │
│ • Spot trading  │ • Perp liquidations│ • Asset listings     │
│ • Perp trading  │ • Liquidator bonus │ • Market maker fees  │
│ • Dynamic fees  │ • Insurance claims │ • Partnership fees   │
└─────────────────┴─────────────────┴─────────────────────────┘
```

## Core Components

### DUSDStaking Contract
**File:** `src/perps/DUSDStaking.sol`

Main staking contract managing user stakes and rewards:

```solidity
struct StakeInfo {
    uint256 amount;           // Staked DUSD amount
    uint256 lockPeriod;       // Lock duration in seconds
    uint256 startTime;        // Stake start timestamp
    uint256 multiplier;       // Reward multiplier (1e18 = 1x)
    uint256 lastClaimTime;    // Last reward claim timestamp
    bool isActive;            // Stake status
}
```

### RewardDistributor Contract
**File:** `src/perps/RewardDistributor.sol`

Manages revenue collection and distribution:

```solidity
struct RewardPool {
    uint256 totalRewards;     // Total accumulated rewards
    uint256 distributedRewards; // Already distributed rewards
    uint256 rewardRate;       // Rewards per second per staked token
    uint256 lastUpdateTime;   // Last reward calculation update
}
```

## Staking Mechanics

### Lock Periods and Multipliers

| Lock Period | Multiplier | Risk Level | Governance Weight |
|-------------|------------|------------|-------------------|
| **Flexible** | 1.0x | Low | 1x |
| **3 Months** | 1.1x | Low | 1.2x |
| **6 Months** | 1.25x | Medium | 1.5x |
| **12 Months** | 1.5x | High | 2x |
| **24 Months** | 2.0x | Very High | 3x |

### Staking Process
```javascript
// Stake DUSD with lock period
await dusdStaking.stake(
  parseEther("10000"),    // 10,000 DUSD
  365 * 24 * 60 * 60,     // 1 year lock period
  { value: executionFee } // Gas fee for keepers
)
```

### Reward Calculation
```solidity
// Effective staking power
effectiveStake = stakedAmount * multiplier

// User rewards
userRewards = (effectiveStake * rewardRate * timePeriod) / totalEffectiveStake
```

## Revenue Sources

### Trading Fee Revenue (60% of total fees)
- **Spot Trading**: 0.3-2% fees from DSwap operations
- **Perpetual Trading**: Maker/taker fees from order book
- **Dynamic Fees**: Stress-based burn fees during high utilization
- **Cross-Chain**: Bridge and meta-transaction fees

### Liquidation Revenue (25% of total fees)
- **Liquidation Penalties**: 0.5% of liquidated position value
- **Liquidator Bonuses**: Portion of liquidation fees
- **Insurance Claims**: Revenue from insurance fund operations
- **ADL Compensation**: Fees from auto-deleveraging events

### Protocol Revenue (15% of total fees)
- **Asset Listing Fees**: Revenue from new synthetic asset additions
- **Market Maker Fees**: Revenue from professional MM programs
- **Partnership Revenue**: Revenue sharing from integrations
- **Governance Fees**: Fees from governance proposal submissions

## Risk Management

### Insurance Fund Integration
DUSD staking provides the primary backstop for protocol risk:

```
Loss Waterfall:
1. Trading Losses → Insurance Fund (Protocol Reserves)
2. Excess Losses → Junior DUSD Stakers (High Multiplier)
3. Extreme Losses → Senior DUSD Stakers (Low Multiplier)
4. Catastrophic → Emergency Protocol Measures
```

### Risk Tranches
```solidity
enum RiskTranche {
    SENIOR,    // Low risk, low reward (flexible staking)
    MEZZANINE, // Medium risk, medium reward (3-6 month locks)
    JUNIOR     // High risk, high reward (12+ month locks)
}
```

### Loss Absorption Mechanism
```solidity
function absorbLoss(uint256 lossAmount) external onlyInsuranceFund {
    // 1. Try insurance fund first
    uint256 insuranceCover = Math.min(lossAmount, insuranceFund.balance);
    lossAmount -= insuranceCover;
    
    if (lossAmount > 0) {
        // 2. Absorb from junior stakers first
        uint256 juniorCover = _absorbFromTranche(JUNIOR, lossAmount);
        lossAmount -= juniorCover;
        
        if (lossAmount > 0) {
            // 3. Then mezzanine stakers
            uint256 mezzanineCover = _absorbFromTranche(MEZZANINE, lossAmount);
            lossAmount -= mezzanineCover;
            
            if (lossAmount > 0) {
                // 4. Finally senior stakers
                _absorbFromTranche(SENIOR, lossAmount);
            }
        }
    }
}
```

## Governance Integration

### Voting Power Calculation
```solidity
function getVotingPower(address user) external view returns (uint256) {
    StakeInfo memory stake = stakes[user];
    if (!stake.isActive) return 0;
    
    uint256 lockMultiplier = _getLockMultiplier(stake.lockPeriod);
    uint256 timeMultiplier = _getTimeMultiplier(stake.startTime);
    
    return stake.amount * lockMultiplier * timeMultiplier / 1e36;
}
```

### Governance Rights
Stakers can participate in:
- **Risk Parameter Updates**: Leverage limits, margin requirements
- **Asset Listings**: Adding new synthetic assets
- **Fee Structure Changes**: Trading and liquidation fee adjustments
- **Oracle Configuration**: Adding/removing price feed sources
- **Emergency Actions**: Protocol pause and recovery procedures

### Proposal Requirements
| Proposal Type | Min Voting Power | Quorum Required | Execution Delay |
|---------------|------------------|-----------------|-----------------|
| **Parameter Change** | 100K DUSD | 4% | 24 hours |
| **Asset Listing** | 500K DUSD | 6% | 48 hours |
| **Fee Changes** | 1M DUSD | 8% | 72 hours |
| **Emergency Action** | 2M DUSD | 12% | Immediate |

## Reward Distribution

### Distribution Schedule
- **Frequency**: Rewards distributed every 24 hours
- **Calculation**: Based on time-weighted average staking
- **Compounding**: Automatic reinvestment option available
- **Claiming**: Manual claim or auto-compound

### Reward Types
1. **Base Rewards**: Fixed APY based on protocol revenue
2. **Performance Bonuses**: Additional rewards during high activity
3. **Governance Rewards**: Bonuses for active governance participation
4. **Loyalty Rewards**: Increasing multipliers for long-term stakers

### Example Reward Calculation
```javascript
// User stakes 10,000 DUSD for 1 year
const stakeAmount = 10000
const lockPeriod = 365 * 24 * 60 * 60 // 1 year
const multiplier = 1.5 // 1.5x for 1-year lock

// Effective stake for rewards
const effectiveStake = stakeAmount * multiplier // 15,000 effective DUSD

// If protocol generates 1M DUSD in fees annually
// And user has 1% of total effective stake
const annualRewards = 1000000 * 0.01 // 10,000 DUSD rewards
const apy = annualRewards / stakeAmount // 100% APY
```

## Integration Examples

### Staking Operations
```javascript
import { DUSDStaking } from '@di-network/sdk'

const staking = new DUSDStaking({
  provider: web3Provider,
  network: 'mainnet'
})

// Stake DUSD
await staking.stake({
  amount: parseEther("10000"),
  lockPeriod: 365 * 24 * 60 * 60, // 1 year
  autoCompound: true
})

// Check rewards
const rewards = await staking.getPendingRewards(userAddress)
console.log('Pending rewards:', formatEther(rewards))

// Claim rewards
await staking.claimRewards()

// Unstake (after lock period)
await staking.unstake(stakeId)
```

### Governance Participation
```javascript
// Check voting power
const votingPower = await staking.getVotingPower(userAddress)

// Vote on proposal
await governance.vote(proposalId, true, votingPower)

// Create proposal (requires minimum voting power)
await governance.createProposal({
  title: "Increase BTC leverage limit to 75x",
  description: "Proposal to increase maximum leverage...",
  actions: [
    {
      target: marketRegistry.address,
      calldata: marketRegistry.interface.encodeFunctionData(
        "setMaxLeverage",
        [keccak256("BTC"), 75]
      )
    }
  ]
})
```

### Risk Monitoring
```javascript
// Monitor staking pool health
const poolHealth = await staking.getPoolHealth()
console.log('Insurance fund ratio:', poolHealth.insuranceRatio)
console.log('Total staked:', formatEther(poolHealth.totalStaked))
console.log('Risk exposure:', formatEther(poolHealth.riskExposure))

// Check individual stake risk
const stakeRisk = await staking.getStakeRisk(userAddress, stakeId)
console.log('Risk tranche:', stakeRisk.tranche)
console.log('Loss exposure:', formatEther(stakeRisk.maxLoss))
```

## Smart Contract Interface

```solidity
interface IDUSDStaking {
    // Staking operations
    function stake(uint256 amount, uint256 lockPeriod) external payable;
    function unstake(uint256 stakeId) external;
    function claimRewards(uint256 stakeId) external;
    function compoundRewards(uint256 stakeId) external;
    
    // View functions
    function getStakeInfo(address user, uint256 stakeId) external view returns (StakeInfo memory);
    function getPendingRewards(address user, uint256 stakeId) external view returns (uint256);
    function getVotingPower(address user) external view returns (uint256);
    function getTotalStaked() external view returns (uint256);
    
    // Risk management
    function getPoolHealth() external view returns (PoolHealth memory);
    function getStakeRisk(address user, uint256 stakeId) external view returns (StakeRisk memory);
    function absorbLoss(uint256 amount) external; // Only insurance fund
}

interface IRewardDistributor {
    function distributeRewards(uint256 amount, RewardType rewardType) external;
    function updateRewardRate() external;
    function getRewardRate() external view returns (uint256);
    function getTotalRewards() external view returns (uint256);
}
```

## Security Features

### Access Control
- **Role-Based Permissions**: Staker, Admin, Insurance Fund roles
- **Multi-Signature**: Critical functions require multiple signatures
- **Timelock**: Governance changes have execution delays
- **Emergency Pause**: Immediate staking halt capability

### Economic Security
- **Slashing Protection**: No slashing for honest stakers
- **Gradual Unstaking**: Prevents bank run scenarios
- **Insurance Backstop**: Multiple layers of loss protection
- **Diversified Revenue**: Multiple income streams reduce risk

### Operational Security
- **Oracle Integration**: Real-time protocol health monitoring
- **Automated Rebalancing**: Dynamic risk management
- **Audit Trail**: Complete transaction history
- **Upgrade Safety**: Backwards compatible upgrades

## Benefits

### For Stakers
- **Passive Income**: Earn from protocol revenue without active trading
- **Governance Rights**: Influence protocol development and parameters
- **Risk-Adjusted Returns**: Choose risk level based on lock period
- **Compounding Growth**: Automatic reward reinvestment options

### For the Protocol
- **Economic Security**: Aligned incentives for protocol health
- **Governance Participation**: Active community involvement
- **Capital Efficiency**: Staked capital provides insurance backstop
- **Long-Term Alignment**: Lock periods encourage long-term thinking

### For the Ecosystem
- **Stability**: Reduced protocol risk through insurance mechanism
- **Growth**: Revenue sharing attracts long-term capital
- **Decentralization**: Community governance and ownership
- **Innovation**: Funding for protocol development and expansion

## Risk Considerations

### Staker Risks
- **Loss Absorption**: Potential loss of staked capital in extreme scenarios
- **Lock Period Risk**: Cannot unstake during lock period
- **Governance Risk**: Poor governance decisions may affect returns
- **Smart Contract Risk**: Technical vulnerabilities in staking contracts

### Mitigation Strategies
- **Diversified Tranches**: Multiple risk levels for different risk appetites
- **Insurance Fund**: Primary loss absorption before staker funds
- **Gradual Implementation**: Phased rollout with conservative parameters
- **Continuous Monitoring**: Real-time risk assessment and management

## Future Enhancements

### Planned Features
- **Liquid Staking**: Tradeable staking derivatives (sDUSD tokens)
- **Cross-Chain Staking**: Stake DUSD from multiple chains
- **Advanced Strategies**: Automated yield optimization
- **Insurance Products**: Additional protection for stakers

### Integration Opportunities
- **DeFi Protocols**: Integration with lending and yield farming
- **Institutional Products**: Professional staking services
- **Mobile Apps**: User-friendly staking interfaces
- **Analytics Tools**: Advanced staking performance tracking