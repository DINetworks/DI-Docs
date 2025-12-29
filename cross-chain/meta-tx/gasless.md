# Gasless Transactions

Gasless transactions allow users to interact with DI Network without holding native blockchain tokens for gas fees. Relayers execute transactions on behalf of users while covering the gas costs.

## How Gasless Transactions Work

### 1. Transaction Creation
```javascript
const metaTx = {
  from: userAddress,
  to: contractAddress,
  data: encodedFunctionCall,
  nonce: userNonce,
  gasLimit: estimatedGas,
  expiration: timestamp + 3600 // 1 hour validity
}
```

### 2. User Signature
Users sign the meta transaction with their private key:
```javascript
const signature = await user.signMessage(metaTx)
```

### 3. Relayer Execution
Relayers submit the signed transaction to the blockchain and pay gas fees.

## Relayer Network

### Relayer Selection
- **Reputation-based**: Higher reputation relayers get priority
- **Fee-based**: Competitive fee structure for relayer services
- **Geographic**: Regional relayers for faster execution

### Relayer Requirements
- Minimum DI token stake for bonding
- Sufficient native tokens for gas payments
- Reliable infrastructure and uptime

## Fee Structure

### Relayer Fees
- Base fee: 0.1% of transaction value
- Gas markup: 10-20% above network gas price
- Priority fees for faster execution

### Payment Methods
- DI tokens from user balance
- Gas credits (pre-purchased)
- Cross-subsidization for new users

## Security Measures

### Signature Verification
All meta transactions require valid user signatures verified on-chain.

### Nonce Management
Sequential nonces prevent replay attacks and ensure transaction ordering.

### Expiration Timestamps
Transactions have limited validity periods to prevent stale execution.

### Relayer Bonding
Relayers stake DI tokens that can be slashed for malicious behavior.

## Integration

### SDK Usage
```javascript
import { DINetwork } from '@di-network/sdk'

const di = new DINetwork({ gasless: true })

// Execute gasless transaction
await di.swap({
  from: 'DI',
  to: 'DUSD',
  amount: '100',
  gasless: true
})
```

### Smart Contract Integration
Contracts must inherit from `MetaTransactionReceiver` to support gasless execution.

## Limitations

- Slightly higher latency due to relayer processing
- Dependency on relayer network availability
- Additional fees compared to direct transactions
- Limited to supported contract functions