# Frequently Asked Questions

Common questions and answers about DI Network, its features, and how to use the protocol.

## General Questions

### What is DI Network?
DI Network is a comprehensive DeFi ecosystem that enables cross-chain interoperability, synthetic asset trading, and perpetual trading. It consists of multiple interconnected protocols working together to provide a unified multi-chain DeFi experience.

### What makes DI Network different?
- **True Multi-Chain**: Unified experience across 6 blockchain networks
- **Zero Slippage Trading**: Oracle-based pricing eliminates slippage
- **Gasless Transactions**: Pay fees in DUSD across all networks
- **Comprehensive Coverage**: 50+ synthetic assets including stocks, commodities, and crypto

### Which networks are supported?
DI Network is deployed on:
- Ethereum (Chain ID: 1)
- BSC (Chain ID: 56)
- Polygon (Chain ID: 137)
- Arbitrum (Chain ID: 42161)
- Base (Chain ID: 8453)
- Crossfi (Chain ID: 4157)

## DI Token Questions

### What is the DI token used for?
DI token serves four main purposes:
1. **Governance**: Vote on protocol parameters and upgrades
2. **Collateral**: Primary collateral for minting DUSD (75% factor)
3. **Staking**: Earn 8-20% APY by staking with lock periods
4. **Fee Sharing**: Receive 50% of all protocol fees

### What is the total supply of DI tokens?
The total supply is fixed at 1 billion DI tokens. No additional tokens can be minted, making it a deflationary asset through buyback and burn mechanisms.

### How can I acquire DI tokens?
You can acquire DI tokens through:
- **DEXs**: Uniswap, PancakeSwap, QuickSwap
- **CEXs**: Binance, Coinbase, OKX (when listed)
- **Token Sales**: Participate in public or private sales
- **Staking Rewards**: Earn DI by staking existing tokens

## DUSD Stablecoin Questions

### What is DUSD?
DUSD is DI Network's over-collateralized algorithmic stablecoin pegged to $1 USD. It serves as the base currency for all protocol operations including trading, cross-chain transfers, and gas payments.

### How is DUSD backed?
DUSD is backed by over-collateralization with multiple asset types:
- DI tokens (75% collateral factor)
- WBTC (70% collateral factor)
- WETH (70% collateral factor)
- USDT/USDC (90% collateral factor)

### What interest rate do I pay on borrowed DUSD?
The interest rate is fixed at 5% APR, which accrues continuously. This rate can be adjusted through governance voting.

### When can my DUSD position be liquidated?
Your position can be liquidated when:
- Your collateral ratio falls below 80%
- Your remaining collateral is insufficient to cover fees
- Your position becomes insolvent due to interest accrual

## Trading Questions

### What's the difference between DSwap and DPerp?
- **DSwap**: Spot trading of synthetic assets with zero slippage using oracle pricing
- **DPerp**: Perpetual trading with leverage up to 50x using GMX-style mechanics

### What assets can I trade?
You can trade 50+ synthetic assets including:
- **Cryptocurrencies**: xBTC, xETH, xBNB, xADA, xSOL
- **Stocks**: xAAPL, xTSLA, xGOOG, xAMZN, xMSFT
- **Commodities**: xGold, xSilver, xOil, xGas
- **Forex**: xEUR, xGBP, xJPY, xCHF

### What are the trading fees?
- **Spot Trading**: 0.1% for minting/burning, 0.3% for swapping
- **Perpetual Trading**: 0.1% for opening/closing positions
- **Funding Rates**: Variable based on utilization

### Can I trade 24/7?
Yes! Unlike traditional markets, DI Network operates 24/7, allowing you to trade stocks and commodities even when traditional markets are closed.

## Cross-Chain Questions

### How do cross-chain transfers work?
DI Network uses a decentralized relayer network to validate and execute cross-chain messages. The process is:
1. Initiate transfer on source chain
2. Relayers validate the message
3. Execute transfer on destination chain
4. Confirmation and completion

### What are gasless transactions?
Gasless transactions allow you to pay transaction fees in DUSD instead of native tokens (ETH, BNB, etc.). You deposit DUSD as gas credits and the system automatically deducts fees.

### How long do cross-chain transfers take?
- **Typical Time**: 2-10 minutes depending on network congestion
- **Confirmation Time**: 1-3 block confirmations on each chain
- **Status Tracking**: Real-time status updates available

## Staking Questions

### How much can I earn by staking DI tokens?
Staking rewards depend on your lock period:
- No lock: 8% APY
- 3 months: 9.6% APY
- 6 months: 12% APY
- 12 months: 16% APY
- 24 months: 20% APY

### Can I unstake early?
Yes, but with penalties:
- **Penalty**: 50% of accrued rewards forfeited
- **Emergency Only**: Available for emergency situations
- **Governance Control**: Community can enable/disable early unstaking

### Do staked tokens have voting power?
Yes, staked tokens receive enhanced voting power:
- Base voting power: 1 DI = 1 vote
- Staking multiplier: Up to 2x for 24-month locks
- Participation bonus: +20% for active voters

## Security Questions

### Is DI Network audited?
Yes, DI Network has undergone multiple security audits by leading firms:
- Trail of Bits
- ConsenSys Diligence
- Quantstamp
- All audit reports are publicly available

### Is there a bug bounty program?
Yes, we have an active bug bounty program with:
- **Total Rewards**: $500,000 allocated
- **Scope**: All smart contracts and infrastructure
- **Platform**: HackerOne
- **Response Time**: <24 hours for critical issues

### What happens if there's a security issue?
We have comprehensive emergency procedures:
- **Emergency Pause**: 5-of-9 multi-sig can pause the protocol
- **Incident Response**: <2 hours response time for critical issues
- **Communication**: Transparent updates to the community
- **Resolution**: Fix implementation and thorough testing

## Technical Questions

### What wallets are supported?
DI Network supports all major Web3 wallets:
- MetaMask
- WalletConnect
- Coinbase Wallet
- Trust Wallet
- Hardware wallets (Ledger, Trezor)

### Is there an API for developers?
Yes, we provide comprehensive APIs:
- **GraphQL API**: Query protocol data
- **REST API**: Simple HTTP endpoints
- **WebSocket API**: Real-time data streaming
- **SDK**: JavaScript/TypeScript SDK for easy integration

### Can I integrate DI Network into my dApp?
Absolutely! We provide:
- **Comprehensive SDK**: Easy integration tools
- **Documentation**: Detailed integration guides
- **Examples**: Sample code and tutorials
- **Support**: Developer support channels

## Troubleshooting

### My transaction failed. What should I do?
1. **Check Gas**: Ensure sufficient gas for the transaction
2. **Check Allowances**: Verify token approvals are sufficient
3. **Check Slippage**: Increase slippage tolerance if needed
4. **Try Again**: Network congestion may cause temporary failures

### I can't see my tokens after a cross-chain transfer
1. **Check Network**: Ensure you're on the correct destination network
2. **Add Token**: Manually add the token contract to your wallet
3. **Wait**: Cross-chain transfers can take 2-10 minutes
4. **Check Status**: Use the bridge status tracker

### My position was liquidated. Why?
Positions are liquidated when:
- Collateral ratio falls below the liquidation threshold
- Remaining collateral insufficient to cover fees
- Position becomes insolvent due to interest or funding fees

## Getting Help

### Where can I get support?
- **Discord**: Join our community for real-time help
- **Telegram**: Official announcements and support
- **Documentation**: Comprehensive guides and tutorials
- **GitHub**: Technical issues and feature requests

### How do I report a bug?
1. **Security Issues**: Email security@dinetwork.xyz
2. **General Bugs**: Create a GitHub issue
3. **Bug Bounty**: Submit through HackerOne
4. **Community**: Report in Discord #bug-reports

### Where can I find the latest updates?
- **Twitter**: @dinetwork for announcements
- **Discord**: Community discussions and updates
- **Blog**: Medium blog for detailed updates
- **GitHub**: Code updates and releases

---

Can't find your question? Join our [Discord community](https://discord.gg/dinetwork) for real-time support!