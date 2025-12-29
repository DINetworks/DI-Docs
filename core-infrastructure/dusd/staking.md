# DUSD Staking

Stake DUSD to earn yield, participate in protocol governance, and help maintain stablecoin stability through the stability pool mechanism.

## Staking Overview

DUSD staking serves multiple purposes:
- Earn yield from protocol fees
- Participate in liquidation rewards
- Maintain DUSD price stability
- Gain governance voting power

## Staking Mechanics

### Stability Pool
DUSD stakers deposit into the stability pool which:
- Absorbs bad debt from liquidations
- Earns liquidation bonuses
- Maintains protocol solvency
- Provides DUSD price support

### Yield Sources
```
Total Yield = Base APY + Liquidation Rewards + Fee Share
```

**Yield Components:**
- **Base APY**: 5-8% from protocol operations
- **Liquidation Rewards**: Variable based on liquidation activity
- **Fee Share**: Portion of trading and borrowing fees

### Staking Process
```javascript
// Stake DUSD in stability pool
await dusdStaking.stake(parseEther("10000")) // 10,000 DUSD

// Check staking position
const position = await dusdStaking.getStakerInfo(userAddress)
console.log({
  stakedAmount: formatEther(position.amount),
  pendingRewards: formatEther(position.rewards),
  liquidationGains: formatEther(position.collateralGains)
})
```

## Liquidation Mechanism

### Liquidation Absorption
When positions are liquidated:
1. DUSD from stability pool covers bad debt
2. Stakers receive liquidated collateral at discount
3. Pool size decreases, individual stakes remain proportional

### Liquidation Rewards
```solidity
function liquidate(address borrower) external {
    uint256 debt = getBorrowerDebt(borrower);
    uint256 collateral = getBorrowerCollateral(borrower);
    
    // Burn DUSD from stability pool
    stabilityPool.offset(debt);
    
    // Distribute collateral to stakers
    stabilityPool.distributeCollateral(collateral);
}
```

### Example Liquidation
```
Initial Pool: 1,000,000 DUSD
Your Stake: 10,000 DUSD (1%)
Liquidation: 50,000 DUSD debt, 60,000 DUSD collateral

After Liquidation:
- Pool Size: 950,000 DUSD
- Your Stake: 9,500 DUSD
- Your Collateral Gain: 600 DUSD (1% of 60,000)
- Net Gain: 100 DUSD profit
```

## Yield Calculation

### APY Components
| Component | Range | Source |
|-----------|-------|--------|
| Base Yield | 5-8% | Protocol fees |
| Liquidation Bonus | 2-15% | Liquidation premiums |
| Fee Share | 1-3% | Trading/borrowing fees |
| **Total APY** | **8-26%** | **Combined sources** |

### Dynamic Yields
Yields fluctuate based on:
- Market volatility (affects liquidations)
- Protocol usage (affects fees)
- Pool utilization (affects base rate)
- Collateral types (affects liquidation frequency)

## Risk Considerations

### Liquidation Risk
- Pool absorbs bad debt during liquidations
- Temporary reduction in DUSD holdings
- Compensated by collateral gains
- Long-term profitable due to liquidation premiums

### Impermanent Loss
Unlike LP tokens, DUSD staking has no impermanent loss:
- Always denominated in DUSD
- Collateral gains are bonus rewards
- No price correlation risk

### Smart Contract Risk
- Audited stability pool contracts
- Gradual rollout with monitoring
- Insurance fund backstop

## Governance Integration

### Voting Power
Staked DUSD provides governance voting power:
```
Voting Power = Staked DUSD × Time Multiplier
```

### Time Multipliers
| Stake Duration | Voting Multiplier |
|----------------|-------------------|
| < 1 month | 1.0x |
| 1-3 months | 1.2x |
| 3-6 months | 1.5x |
| 6-12 months | 2.0x |
| > 12 months | 2.5x |

### Governance Rewards
Additional rewards for governance participation:
- Voting bonuses: +10% APY for active voters
- Proposal rewards: Bonus for successful proposals
- Delegation fees: Earn from delegated voting power

## Integration Examples

### Basic Staking
```javascript
import { DUSDStaking } from '@dinetwork/sdk'

const staking = new DUSDStaking({ provider, signer })

// Stake DUSD
await staking.stake(parseEther("5000"))

// Check rewards
const rewards = await staking.getPendingRewards(userAddress)
console.log(`Pending rewards: ${formatEther(rewards)} DUSD`)

// Claim rewards
await staking.claimRewards()
```

### Advanced Position Management
```javascript
// Get detailed staking info
const stakingInfo = await staking.getStakingInfo(userAddress)

const position = {
  stakedAmount: formatEther(stakingInfo.stakedAmount),
  dusdRewards: formatEther(stakingInfo.dusdRewards),
  collateralGains: stakingInfo.collateralGains.map(gain => ({
    token: gain.token,
    amount: formatEther(gain.amount)
  })),
  currentAPY: stakingInfo.currentAPY / 100,
  votingPower: formatEther(stakingInfo.votingPower)
}
```

### Liquidation Monitoring
```javascript
const useLiquidationMonitor = () => {
  const [liquidations, setLiquidations] = useState([])
  
  useEffect(() => {
    const contract = new ethers.Contract(STABILITY_POOL_ADDRESS, ABI, provider)
    
    const handleLiquidation = (liquidator, borrower, debtOffset, collateralSent) => {
      setLiquidations(prev => [{
        liquidator,
        borrower,
        debtOffset: formatEther(debtOffset),
        collateralSent: formatEther(collateralSent),
        timestamp: Date.now()
      }, ...prev.slice(0, 49)]) // Keep last 50
    }
    
    contract.on('Liquidation', handleLiquidation)
    return () => contract.off('Liquidation', handleLiquidation)
  }, [])
  
  return liquidations
}
```

## Staking Strategies

### Conservative Strategy
- **Stake Size**: 10-25% of DUSD holdings
- **Duration**: 3-6 months
- **Expected APY**: 8-12%
- **Risk Level**: Low

### Balanced Strategy
- **Stake Size**: 25-50% of DUSD holdings
- **Duration**: 6-12 months
- **Expected APY**: 12-18%
- **Risk Level**: Medium

### Aggressive Strategy
- **Stake Size**: 50-75% of DUSD holdings
- **Duration**: 12+ months
- **Expected APY**: 18-26%
- **Risk Level**: Higher

## Pool Dynamics

### Pool Size Impact
| Pool Size | Liquidation Share | Risk Level | Expected Returns |
|-----------|-------------------|------------|------------------|
| Small (<$1M) | High per staker | Higher | Higher APY |
| Medium ($1-10M) | Moderate | Balanced | Moderate APY |
| Large (>$10M) | Low per staker | Lower | Stable APY |

### Optimal Pool Size
The protocol targets optimal pool size through:
- Dynamic yield adjustments
- Incentive mechanisms
- Pool size-based multipliers

## Withdrawal Process

### Standard Withdrawal
```javascript
// Unstake DUSD (available immediately)
await staking.unstake(parseEther("1000"))

// Claim all rewards and collateral gains
await staking.claimAll()
```

### Emergency Withdrawal
```javascript
// Emergency unstake (may have penalties during high utilization)
await staking.emergencyUnstake(parseEther("5000"))
```

## Monitoring Tools

### Staking Dashboard
```javascript
const StakingDashboard = () => {
  const { data: stakingData } = useQuery(['staking', userAddress], getStakingData)
  
  return (
    <div className="staking-dashboard">
      <div className="pool-stats">
        <div>Total Pool Size: ${formatNumber(stakingData.poolSize)}</div>
        <div>Current APY: {stakingData.currentAPY}%</div>
        <div>Your Share: {stakingData.userShare}%</div>
      </div>
      
      <div className="user-position">
        <div>Staked: {formatNumber(stakingData.stakedAmount)} DUSD</div>
        <div>Rewards: {formatNumber(stakingData.pendingRewards)} DUSD</div>
        <div>Collateral Gains: {stakingData.collateralGains.map(gain => 
          `${formatNumber(gain.amount)} ${gain.symbol}`
        ).join(', ')}</div>
      </div>
    </div>
  )
}
```