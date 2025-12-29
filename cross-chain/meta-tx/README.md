# Meta Transactions

Meta transactions enable users to interact with DI Network without holding native tokens for gas fees. This system provides a seamless user experience by abstracting away gas complexity.

## Overview

Meta transactions allow users to:
- Execute transactions without ETH/native tokens
- Batch multiple operations in a single transaction
- Use gas credits for transaction fees
- Delegate transaction execution to relayers

## Key Features

### Gasless Transactions
Users can perform operations without holding native tokens by using relayers who pay gas fees on their behalf.

### Gas Credits System
A credit-based system where users can pre-purchase gas credits using DI tokens or other supported assets.

### Batch Operations
Multiple operations can be bundled into a single meta transaction, reducing overall gas costs and improving efficiency.

## How It Works

1. **User Signs Transaction**: User creates and signs a meta transaction off-chain
2. **Relayer Submission**: A relayer submits the transaction to the blockchain
3. **Gas Payment**: Gas fees are paid through credits or by the relayer
4. **Execution**: The transaction is executed on behalf of the user

## Benefits

- **Lower Barrier to Entry**: New users don't need native tokens to start using DI Network
- **Cost Efficiency**: Batch operations reduce per-transaction costs
- **Better UX**: Simplified transaction flow for end users
- **Flexible Payment**: Multiple options for covering transaction costs

## Security

Meta transactions maintain the same security guarantees as regular transactions through:
- Cryptographic signature verification
- Nonce management to prevent replay attacks
- Expiration timestamps for time-bound validity
- Relayer reputation and bonding mechanisms