# Bridge Assets

Learn how to safely and efficiently bridge assets between different blockchains on DI Network.

## Bridging Overview

### What is Asset Bridging?
Asset bridging allows you to move tokens between different blockchains while maintaining their value and functionality. DI Network's bridge uses a lock-and-mint mechanism for security.

### Supported Assets
- **DI Token**: Native governance token
- **DUSD**: Algorithmic stablecoin  
- **Major Cryptocurrencies**: ETH, BTC, USDC, USDT
- **LP Tokens**: Liquidity provider tokens
- **Synthetic Assets**: xBTC, xETH, xGOLD, etc.

## Bridge Process

### Step-by-Step Bridging
1. **Select Source Chain**: Choose where your assets currently are
2. **Select Destination Chain**: Choose where you want to send them
3. **Choose Asset**: Select which token to bridge
4. **Enter Amount**: Specify how much to bridge
5. **Review Details**: Check fees, time estimates, and addresses
6. **Execute Transaction**: Confirm and wait for completion

### Bridge Interface
```
Bridge Assets Interface:
┌─────────────────────────────────────┐
│ From: Ethereum                      │
│ To: Polygon                         │
│ Asset: DI Token                     │
│ Amount: 1,000 DI                    │
│ Bridge Fee: 5 DI (0.5%)            │
│ Gas Fee: ~$15                      │
│ Estimated Time: 5-10 minutes       │
│ You Will Receive: ~995 DI          │
└─────────────────────────────────────┘
```

## Bridge Mechanics

### Lock-and-Mint System
**On Source Chain (e.g., Ethereum)**:
1. Your tokens are locked in bridge contract
2. Bridge validators verify the lock transaction
3. Cross-chain message sent to destination

**On Destination Chain (e.g., Polygon)**:
1. Validators verify the cross-chain message
2. Equivalent tokens are minted to your address
3. You receive bridged tokens (e.g., DI on Polygon)

### Security Model
- **Multi-Signature Validation**: Multiple validators must agree
- **Time Delays**: Large transfers have additional security delays
- **Emergency Pause**: Bridge can be paused if issues detected
- **Insurance Fund**: Protocol insurance covers bridge risks

## Bridge Fees

### Fee Structure
| Transfer Size | Bridge Fee | Gas Estimate | Total Cost |
|---------------|------------|--------------|------------|
| **< $1,000** | 0.5% | $10-20 | $15-25 |
| **$1,000 - $10,000** | 0.3% | $15-25 | $18-35 |
| **$10,000 - $100,000** | 0.2% | $20-30 | $40-230 |
| **> $100,000** | 0.1% | $25-35 | $125-1,035 |

### Fee Optimization
**Timing Strategies**:
- Bridge during low network congestion
- Use gas price trackers for optimal timing
- Consider batching multiple transfers

**Size Optimization**:
- Larger transfers have lower percentage fees
- Consider consolidating smaller amounts
- Factor in gas costs vs bridge fees

## Supported Routes

### Primary Routes
**Ethereum ↔ Polygon**:
- Fastest route (5-10 minutes)
- Lowest fees
- Highest liquidity

**Ethereum ↔ Arbitrum**:
- Medium speed (10-15 minutes)
- Moderate fees
- Good liquidity

**Polygon ↔ BSC**:
- Cross-ecosystem bridging
- Higher fees but unique access
- Moderate liquidity

### Route Selection
```javascript
// Example route optimization
const selectOptimalRoute = (fromChain, toChain, amount) => {
  const routes = getAvailableRoutes(fromChain, toChain)
  
  return routes.map(route => ({
    ...route,
    totalCost: route.bridgeFee + route.gasFee,
    timeEstimate: route.avgTime,
    score: calculateRouteScore(route, amount)
  })).sort((a, b) => b.score - a.score)[0]
}
```

## Bridge Safety

### Security Best Practices
**Before Bridging**:
- [ ] Verify you're on the official bridge interface
- [ ] Double-check destination chain and address
- [ ] Start with small test amounts
- [ ] Ensure you have gas tokens on destination chain

**During Bridging**:
- [ ] Don't close browser during transaction
- [ ] Save transaction hash for tracking
- [ ] Monitor bridge status dashboard
- [ ] Be patient - bridges take time

**After Bridging**:
- [ ] Verify tokens arrived at destination
- [ ] Check token contract addresses
- [ ] Add tokens to wallet if needed
- [ ] Keep transaction records

### Common Mistakes to Avoid
**Wrong Network Selection**:
- Always verify source and destination chains
- Check wallet network before confirming
- Understand bridged token contracts

**Insufficient Gas**:
- Keep native tokens on both chains
- Factor in gas price fluctuations
- Have backup funds for emergencies

**Impatience**:
- Bridges take time (5-30 minutes typical)
- Don't attempt duplicate transactions
- Check bridge status before panicking

## Bridge Monitoring

### Transaction Tracking
**Bridge Status Dashboard**:
```
Transaction Status:
┌─────────────────────────────────────┐
│ Status: In Progress                 │
│ From: Ethereum (Confirmed)          │
│ To: Polygon (Pending)              │
│ Asset: 1,000 DI                    │
│ Progress: ████████░░ 80%           │
│ ETA: 3 minutes                     │
│ TX Hash: 0x123...                  │
└─────────────────────────────────────┘
```

**Tracking Tools**:
- Official bridge explorer
- Cross-chain transaction trackers
- Wallet transaction history
- Block explorer verification

### Troubleshooting Delays
**Common Causes**:
- Network congestion on source/destination
- Validator delays or maintenance
- Large transfer security delays
- Bridge liquidity constraints

**Resolution Steps**:
1. Check bridge status page for issues
2. Verify transaction was confirmed on source
3. Wait for standard processing time
4. Contact support if stuck >2 hours

## Advanced Bridging

### Batch Bridging
**Multiple Asset Bridging**:
```javascript
// Bridge multiple assets in one transaction
const batchBridge = async (transfers) => {
  const batch = transfers.map(transfer => ({
    asset: transfer.asset,
    amount: transfer.amount,
    destination: transfer.toChain
  }))
  
  return await bridge.executeBatch(batch)
}
```

### Programmatic Bridging
**SDK Integration**:
```javascript
import { DIBridge } from '@di-network/sdk'

const bridge = new DIBridge({
  fromChain: 'ethereum',
  toChain: 'polygon',
  provider: web3Provider
})

// Execute bridge transfer
const result = await bridge.transfer({
  asset: 'DI',
  amount: parseEther('1000'),
  recipient: userAddress
})
```

### Bridge Arbitrage
**Cross-Chain Price Differences**:
```
Arbitrage Opportunity:
Ethereum DI Price: $10.05
Polygon DI Price: $9.95
Bridge Fee: 0.3% ($3.02)
Gas Costs: $15
Profit per 1000 DI: $100 - $3.02 - $15 = $81.98
ROI: 8.2% per bridge cycle
```

## Bridge Economics

### Liquidity Provision
**Bridge Liquidity Rewards**:
- Provide liquidity to bridge pools
- Earn fees from bridge transactions
- Higher yields but bridge-specific risks
- Lock periods may apply

### Fee Revenue Sharing
**Bridge Fee Distribution**:
- 40% to bridge liquidity providers
- 30% to protocol treasury
- 20% to bridge validators
- 10% to insurance fund

## Emergency Procedures

### Bridge Pauses
**When Bridges Pause**:
- Security incidents or exploits
- Major network issues
- Validator consensus problems
- Planned maintenance windows

**During Pause**:
- No new bridge transactions accepted
- Existing transactions may complete
- Monitor official channels for updates
- Plan alternative routes if needed

### Recovery Procedures
**Stuck Transactions**:
1. Gather transaction details and hashes
2. Check bridge status and known issues
3. Contact support with full information
4. Wait for manual intervention if needed
5. Keep records for potential compensation

## Integration Examples

### Bridge Widget Integration
```javascript
const BridgeWidget = () => {
  const [fromChain, setFromChain] = useState('ethereum')
  const [toChain, setToChain] = useState('polygon')
  const [asset, setAsset] = useState('DI')
  const [amount, setAmount] = useState('')
  
  const { data: quote } = useBridgeQuote({
    fromChain,
    toChain,
    asset,
    amount
  })
  
  return (
    <div className="bridge-widget">
      <ChainSelector value={fromChain} onChange={setFromChain} />
      <AssetSelector value={asset} onChange={setAsset} />
      <AmountInput value={amount} onChange={setAmount} />
      <BridgeQuote quote={quote} />
      <BridgeButton onClick={() => executeBridge(quote)} />
    </div>
  )
}
```

### Bridge Status Monitor
```javascript
const useBridgeStatus = (txHash) => {
  return useQuery(
    ['bridgeStatus', txHash],
    () => getBridgeTransactionStatus(txHash),
    {
      refetchInterval: 10000, // Check every 10 seconds
      enabled: !!txHash
    }
  )
}
```

## Best Practices

### Planning Your Bridge Strategy
- [ ] Understand destination chain requirements
- [ ] Plan for gas costs on both chains
- [ ] Consider timing and market conditions
- [ ] Have contingency plans for delays

### Cost Optimization
- [ ] Compare routes and fees
- [ ] Time transactions for low gas
- [ ] Batch multiple transfers when possible
- [ ] Consider bridge fee vs speed tradeoffs

### Security and Safety
- [ ] Always use official bridge interfaces
- [ ] Verify all addresses and amounts
- [ ] Start with small test transactions
- [ ] Keep detailed records of all bridges
- [ ] Stay informed about bridge updates