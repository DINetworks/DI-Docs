# Fee Structure

The DI Network bridge implements a dynamic fee structure that balances network security, operational costs, and user accessibility while incentivizing optimal liquidity distribution.

## Fee Components

### Base Bridge Fee
Standard fee for all bridge operations:
- **Deposit Fee**: 0.1% of transfer amount
- **Withdrawal Fee**: 0.15% of transfer amount
- **Minimum Fee**: $1 equivalent in DI tokens
- **Maximum Fee**: $100 per transaction

### Network Gas Fees
Users pay gas fees on both chains:
- **Source Chain**: Gas for lock/burn transaction
- **Destination Chain**: Gas for mint/unlock transaction
- **Relayer Fee**: 10-20% markup on destination gas

## Dynamic Fee Calculation

### Liquidity-Based Adjustments
Fees adjust based on asset liquidity:
```javascript
// Fee calculation example
const baseFee = 0.001 // 0.1%
const liquidityRatio = availableLiquidity / totalSupply
const liquidityMultiplier = liquidityRatio < 0.1 ? 2.0 : 1.0
const finalFee = baseFee * liquidityMultiplier
```

### Volume Discounts
| Monthly Volume | Discount |
|----------------|----------|
| $10K - $50K    | 10%      |
| $50K - $200K   | 20%      |
| $200K - $1M    | 30%      |
| $1M+           | 40%      |

### Asset-Specific Fees
| Asset Category | Base Fee | Min Fee | Max Fee |
|----------------|----------|---------|---------|
| DI Token       | 0.05%    | $0.50   | $50     |
| DUSD           | 0.08%    | $0.80   | $80     |
| Major Crypto   | 0.10%    | $1.00   | $100    |
| Other Tokens   | 0.15%    | $1.50   | $150    |

## Fee Payment Options

### Payment Methods
Users can pay fees using:
1. **Source Asset**: Deducted from transfer amount
2. **DI Tokens**: 20% discount when paying with DI
3. **Gas Credits**: Pre-purchased credit system
4. **Fee Sponsorship**: Third-party fee coverage

### Payment Examples
```javascript
// Pay with source asset
await bridge.transfer({
  token: 'USDC',
  amount: '1000',
  feePayment: 'source' // Fee deducted from 1000 USDC
})

// Pay with DI tokens
await bridge.transfer({
  token: 'USDC',
  amount: '1000',
  feePayment: 'DI', // 20% discount applied
  feeToken: 'DI'
})
```

## Fee Distribution

### Revenue Allocation
Bridge fees are distributed as follows:
- **40%**: DI token stakers
- **25%**: Bridge validators and relayers
- **20%**: Protocol treasury
- **10%**: Liquidity incentives
- **5%**: Insurance fund

### Staker Rewards
DI token stakers receive proportional fee rewards:
```javascript
// Calculate staker rewards
const userStake = await staking.getStake(userAddress)
const totalStake = await staking.getTotalStake()
const feeRewards = totalFees * 0.4 * (userStake / totalStake)
```

## Special Fee Programs

### Fast Transfer Premium
Expedited transfers with higher fees:
- **Standard**: 5-15 minutes, normal fees
- **Fast**: 1-3 minutes, 2x fees
- **Instant**: <1 minute, 5x fees (limited availability)

### Liquidity Incentives
Reduced fees for transfers that improve liquidity balance:
- **Rebalancing Bonus**: 50% fee reduction
- **Low Liquidity Chains**: Up to 75% fee reduction
- **Large Transfers**: Graduated discounts for $100K+ transfers

## Fee Estimation

### SDK Integration
```javascript
// Estimate bridge fees
const feeEstimate = await bridge.estimateFees({
  token: 'USDC',
  amount: '1000',
  fromChain: 'ethereum',
  toChain: 'polygon',
  speed: 'standard'
})

// Returns: {
//   bridgeFee: '1.0', // 0.1% of 1000 USDC
//   gasFee: '15.0',   // Estimated gas costs
//   totalFee: '16.0'
// }
```

### Real-time Updates
Fees update based on:
- Network congestion levels
- Asset liquidity ratios
- Validator availability
- Market volatility

## Fee Optimization Strategies

### For Users
- **Batch Transfers**: Combine multiple transfers to reduce per-transaction costs
- **Off-Peak Timing**: Lower fees during low network usage
- **DI Token Payments**: 20% discount for fee payments in DI
- **Volume Discounts**: Increase monthly volume for better rates

### For Developers
```javascript
// Optimize transfer timing
const optimalTime = await bridge.getOptimalTransferTime('USDC')
// Returns timestamp when fees are expected to be lowest

// Batch multiple transfers
const batch = await bridge.createBatch([
  { token: 'USDC', amount: '500', to: 'polygon' },
  { token: 'ETH', amount: '1.0', to: 'arbitrum' }
])
```

## Governance and Fee Updates

### Parameter Adjustment
Governance can modify:
- Base fee percentages
- Minimum and maximum fee limits
- Volume discount tiers
- Fee distribution ratios

### Emergency Fee Controls
- Circuit breakers for excessive fees
- Emergency fee reductions during network issues
- Temporary fee waivers for critical operations

## Fee Transparency

### Real-time Monitoring
```javascript
// Get current fee rates
const feeRates = await bridge.getCurrentFeeRates()

// Get fee history
const feeHistory = await bridge.getFeeHistory('USDC', '7d')
```

### Public Dashboards
- Live fee rates across all assets
- Historical fee trends and analytics
- Volume and revenue statistics
- Liquidity-based fee adjustments

## Refund Policy

### Eligible Scenarios
- Failed transfers due to bridge errors
- Excessive delays beyond SLA
- Technical issues preventing completion
- Governance-approved emergency situations

### Refund Process
```javascript
// Request fee refund
const refund = await bridge.requestRefund({
  transferId: 'failed_transfer_id',
  reason: 'technical_failure',
  evidence: proofOfFailure
})
```