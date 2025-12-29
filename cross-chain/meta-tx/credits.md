# Gas Credits

Gas credits provide a prepaid system for covering transaction fees on DI Network. Users can purchase credits using DI tokens or other supported assets, enabling seamless gasless transactions.

## Overview

Gas credits are tokenized gas allowances that:
- Pre-fund transaction fees
- Enable predictable gas costs
- Support bulk purchasing at discounts
- Work across all supported networks

## Credit System

### Credit Units
- 1 Gas Credit = ~21,000 gas units (standard transfer)
- Credits automatically adjust for network gas prices
- Unused credits never expire

### Purchasing Credits
```javascript
// Purchase with DI tokens
await di.gasCredits.purchase({
  amount: 100, // 100 credits
  paymentToken: 'DI',
  recipient: userAddress
})

// Purchase with DUSD
await di.gasCredits.purchase({
  amount: 50,
  paymentToken: 'DUSD',
  recipient: userAddress
})
```

## Pricing Model

### Base Pricing
- 1 Credit = 0.1 DI tokens (at launch)
- Dynamic pricing based on network congestion
- Volume discounts for bulk purchases

### Discount Tiers
| Credits Purchased | Discount |
|------------------|----------|
| 100-499          | 5%       |
| 500-999          | 10%      |
| 1000-4999        | 15%      |
| 5000+            | 20%      |

### Network-Specific Rates
Credits are automatically converted based on target network gas costs:
- Ethereum: 1 credit = ~21,000 gas
- Polygon: 1 credit = ~100,000 gas
- Arbitrum: 1 credit = ~50,000 gas

## Credit Management

### Balance Checking
```javascript
const balance = await di.gasCredits.getBalance(userAddress)
console.log(`Available credits: ${balance}`)
```

### Usage Tracking
```javascript
const usage = await di.gasCredits.getUsageHistory(userAddress)
// Returns array of credit consumption events
```

### Auto-Refill
Users can set up automatic credit purchases:
```javascript
await di.gasCredits.setAutoRefill({
  threshold: 10, // Refill when below 10 credits
  amount: 100,   // Purchase 100 credits
  paymentToken: 'DI'
})
```

## Credit Transfers

### Gifting Credits
```javascript
await di.gasCredits.transfer({
  to: recipientAddress,
  amount: 25,
  from: senderAddress
})
```

### Corporate Accounts
Organizations can distribute credits to users:
- Bulk credit allocation
- Usage monitoring and limits
- Departmental credit pools

## Integration Examples

### Wallet Integration
```javascript
// Check if user has sufficient credits
const requiredCredits = await di.estimateCredits(transaction)
const userCredits = await di.gasCredits.getBalance(userAddress)

if (userCredits < requiredCredits) {
  // Prompt user to purchase credits
  await promptCreditPurchase(requiredCredits - userCredits)
}
```

### DApp Integration
```javascript
// Sponsor user transactions
await di.gasCredits.sponsorTransaction({
  user: userAddress,
  transaction: txData,
  maxCredits: 5 // Maximum credits to spend
})
```

## Credit Economics

### Supply Management
- Credits are minted when purchased
- Credits are burned when used for gas
- No inflation - 1:1 gas consumption ratio

### Revenue Distribution
Credit purchase fees are distributed to:
- 60% - DI token stakers
- 25% - Protocol treasury
- 15% - Relayer network incentives

## Security Features

### Credit Protection
- Credits are non-transferable by default (unless explicitly enabled)
- Multi-signature support for corporate accounts
- Spending limits and approval workflows

### Fraud Prevention
- Rate limiting on credit purchases
- KYC requirements for large purchases
- Monitoring for unusual usage patterns