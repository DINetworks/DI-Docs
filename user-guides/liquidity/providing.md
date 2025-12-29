# Providing Liquidity

Step-by-step guide to providing liquidity on DI Network, from choosing pools to managing your positions effectively.

## Getting Started

### Step 1: Choose Your Pool
Select from available liquidity pools:
- **DI/ETH**: Highest rewards, moderate risk
- **DI/USDC**: Balanced risk/reward
- **DUSD/USDC**: Lowest risk, stable returns
- **DI/DUSD**: Protocol synergy benefits

### Step 2: Prepare Your Assets
Ensure you have both tokens in the pair:
```
For DI/ETH Pool:
- Equal USD value of DI and ETH
- Example: $1000 DI + $1000 ETH
- Plus ETH for gas fees
```

### Step 3: Connect Wallet
1. Visit DI Network liquidity interface
2. Connect your wallet (MetaMask, WalletConnect)
3. Ensure you're on the correct network
4. Verify your token balances

## Adding Liquidity Process

### Single-Sided Deposits
Add one token and let the protocol balance:
```
1. Select "Add Liquidity"
2. Choose your pool (e.g., DI/ETH)
3. Enter amount of one token
4. Protocol calculates required amount of other token
5. Approve both token spending
6. Confirm liquidity addition
```

### Balanced Deposits
Add both tokens in correct ratio:
```
Current Pool Ratio: 1 ETH = 4000 DI
Your Deposit: 1 ETH + 4000 DI
LP Tokens Received: Based on pool share
```

### Transaction Steps
1. **Approve Tokens**: Allow contract to spend your tokens
2. **Add Liquidity**: Execute the liquidity addition
3. **Receive LP Tokens**: Get proof of your pool share
4. **Stake LP Tokens**: Stake for additional rewards (optional)

## Pool Selection Strategy

### Risk Assessment
| Pool | Impermanent Loss Risk | Reward Potential | Volatility |
|------|----------------------|------------------|------------|
| **DI/ETH** | High | Very High | High |
| **DI/USDC** | Medium | High | Medium |
| **DUSD/USDC** | Very Low | Low-Medium | Very Low |
| **DI/DUSD** | Medium | High | Medium |

### Reward Comparison
```
Current APY Breakdown (Example):

DI/ETH Pool:
- Trading Fees: 12%
- Liquidity Mining: 45%
- Total APY: 57%

DI/USDC Pool:
- Trading Fees: 8%
- Liquidity Mining: 35%
- Total APY: 43%

DUSD/USDC Pool:
- Trading Fees: 3%
- Liquidity Mining: 15%
- Total APY: 18%
```

## Liquidity Mining

### Staking LP Tokens
After providing liquidity, stake LP tokens for rewards:
```
1. Receive LP tokens from liquidity provision
2. Navigate to "Liquidity Mining" section
3. Select your pool
4. Stake LP tokens
5. Start earning DI rewards
```

### Reward Calculation
```
Your Pool Share = Your LP Tokens / Total LP Tokens
Daily Rewards = Pool Daily Rewards × Your Pool Share

Example:
Pool Daily Rewards: 1000 DI
Your Pool Share: 2%
Your Daily Rewards: 20 DI
```

### Boost Mechanisms
**DI Token Boost**:
- Hold DI tokens for up to 2.5x reward boost
- Boost = min(2.5, DI_held / LP_value × 0.4)

**Loyalty Boost**:
- Longer liquidity provision gets bonus
- 1% bonus per month, up to 12%

## Managing Your Position

### Monitoring Tools
Track key metrics:
```
Position Dashboard:
- LP Token Balance: 0.5 LP tokens
- Pool Share: 2.1%
- Current Value: $2,150
- Impermanent Loss: -$45 (-2.1%)
- Fees Earned: $125
- Mining Rewards: $340
- Net P&L: +$420 (+19.5%)
```

### Rebalancing Strategy
**When to Rebalance**:
- Significant price divergence (>20%)
- Better opportunities in other pools
- Risk tolerance changes
- Reward rate changes

**Rebalancing Process**:
1. Calculate current position value
2. Assess impermanent loss
3. Compare with holding strategy
4. Factor in switching costs
5. Execute if net benefit > 2%

### Adding More Liquidity
**Dollar-Cost Averaging**:
```
Strategy: Add liquidity regularly
Week 1: Add $500 to DI/ETH
Week 2: Add $500 more
Week 3: Add $500 more
Benefit: Reduces timing risk
```

## Advanced Strategies

### Range Orders
For concentrated liquidity (when available):
```
Set Price Range:
Lower Bound: $8 per DI
Upper Bound: $12 per DI
Current Price: $10 per DI
Benefit: Higher fees within range
Risk: Out-of-range = no fees
```

### Yield Farming Rotation
Move between pools for optimal returns:
```
Decision Framework:
1. Monitor APY changes weekly
2. Calculate switching costs
3. Estimate time to break-even
4. Execute if benefit > 3% APY
```

### Hedging Strategies
**Perpetual Hedge**:
```
LP Position: $10,000 DI/ETH (50% DI exposure)
Hedge: Short $5,000 DI perpetual
Result: Reduced IL, maintain fee earnings
Cost: Funding fees on perpetual
```

## Risk Management

### Position Sizing
**Conservative Approach**:
- Maximum 10% of portfolio in LP
- Focus on stable pairs
- Regular monitoring and rebalancing

**Aggressive Approach**:
- Up to 30% of portfolio in LP
- Higher risk/reward pairs
- Active management required

### Stop-Loss Strategies
**IL-Based Stops**:
```
Set IL Threshold: 5%
Monitor Daily: Check IL vs rewards
Exit Trigger: IL > 5% and rewards < IL
Action: Remove liquidity, reassess
```

**Time-Based Stops**:
- Set maximum holding period
- Reassess every 30-90 days
- Exit if objectives not met

## Tax Implications

### LP Token Creation
Generally not a taxable event:
- No immediate gain/loss recognition
- Cost basis tracks underlying assets
- LP tokens represent pool ownership

### Reward Taxation
**Liquidity Mining Rewards**:
- Taxable as income when earned
- Fair market value at receipt
- Daily accrual creates daily events

**Trading Fee Rewards**:
- Also taxable as income
- Compounding creates additional events
- Track for accurate reporting

### Exit Taxation
**Removing Liquidity**:
```
Tax Calculation:
Initial Deposit: 1 ETH + 4000 DI
Exit Received: 1.1 ETH + 3800 DI
ETH Gain: 0.1 ETH (taxable)
DI Loss: 200 DI (deductible)
Net: Calculate based on USD values
```

## Troubleshooting

### Common Issues
**High Gas Fees**:
- Time transactions during low congestion
- Use gas trackers for optimal timing
- Consider L2 solutions when available

**Slippage Errors**:
- Increase slippage tolerance
- Break large orders into smaller ones
- Check for sufficient liquidity

**Reward Discrepancies**:
- Verify LP tokens are staked
- Check reward distribution schedule
- Confirm you're in correct pool

### Getting Help
**Support Channels**:
- Discord community support
- Documentation and FAQs
- Support ticket system
- Community forums

## Best Practices

### Before Providing Liquidity
- [ ] Research pool mechanics thoroughly
- [ ] Understand impermanent loss
- [ ] Calculate potential returns
- [ ] Assess risk tolerance
- [ ] Start with small amounts

### During Liquidity Provision
- [ ] Monitor position daily
- [ ] Track IL vs rewards
- [ ] Stay informed about program changes
- [ ] Participate in governance
- [ ] Keep detailed records

### Optimization Tips
- [ ] Compound rewards regularly
- [ ] Time entries during high volatility
- [ ] Diversify across multiple pools
- [ ] Use boost mechanisms
- [ ] Plan exit strategies

### Security Considerations
- [ ] Verify contract addresses
- [ ] Use hardware wallets for large amounts
- [ ] Keep private keys secure
- [ ] Monitor for protocol updates
- [ ] Understand emergency procedures