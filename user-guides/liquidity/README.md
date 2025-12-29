# Liquidity Guide

Learn how to provide liquidity on DI Network, earn trading fees, participate in liquidity mining, and manage your liquidity provider positions effectively.

## Liquidity Overview

DI Network offers multiple liquidity provision opportunities:
- **DEX Liquidity**: Provide liquidity to DI/ETH, DI/USDC pairs
- **Stability Pool**: DUSD liquidity for liquidation absorption
- **Cross-Chain Liquidity**: Bridge liquidity across networks
- **Synthetic Asset Pools**: Liquidity for synthetic asset trading

## Getting Started

### Prerequisites
- Supported wallet (MetaMask, WalletConnect, etc.)
- Tokens for liquidity pairs (DI, ETH, USDC, DUSD)
- Understanding of impermanent loss
- Basic knowledge of AMM mechanics

### Choosing Your Strategy
| Strategy | Risk Level | Expected Returns | Time Commitment |
|----------|------------|------------------|-----------------|
| **Stable Pairs** | Low | 5-15% APY | Low |
| **DI Pairs** | Medium | 15-40% APY | Medium |
| **Exotic Pairs** | High | 30-80% APY | High |
| **Stability Pool** | Low-Medium | 8-26% APY | Low |

## Liquidity Provision Types

### AMM Liquidity Pools
**Traditional 50/50 Pools**:
- Equal value of two tokens
- Earn trading fees (0.3%)
- Subject to impermanent loss
- Liquidity mining rewards available

**Supported Pairs**:
- DI/ETH (Primary pair, highest rewards)
- DI/USDC (Stable exposure)
- DI/DUSD (Protocol synergy)
- DUSD/USDC (Stable pair, low IL)

### Stability Pool (DUSD)
**Mechanism**: Single-asset liquidity provision
- Deposit only DUSD
- No impermanent loss
- Earn liquidation bonuses
- Support protocol stability

## Liquidity Mining Program

### Reward Structure
```
Total Liquidity Mining Allocation: 300M DI (5 years)

Year 1 Distribution (90M DI):
├── DI/ETH Pool: 36M DI (40%)
├── DI/USDC Pool: 27M DI (30%)
├── DI/DUSD Pool: 18M DI (20%)
└── Other Pools: 9M DI (10%)
```

### APY Calculation
```
Pool APY = Trading Fees APY + Liquidity Mining APY

Example DI/ETH Pool:
Trading Volume: $1M daily
Pool TVL: $10M
Trading Fee APY: ($1M × 0.003 × 365) / $10M = 10.95%

Liquidity Mining: 36M DI rewards / $10M TVL = 3.6 DI per $ TVL
At $10/DI: 36% APY
Total APY: 10.95% + 36% = 46.95%
```

## Risk Management

### Impermanent Loss
Occurs when token prices diverge:
```
Initial: 1 ETH = $2000, 1 DI = $10
Pool: 100 ETH + 20,000 DI = $400,000

After price change: 1 ETH = $3000, 1 DI = $10
Your share: 81.65 ETH + 24,495 DI = $489,900
Hold strategy: 100 ETH + 20,000 DI = $500,000
Impermanent Loss: $10,100 (2.02%)
```

### Risk Mitigation
- Choose correlated pairs (DUSD/USDC)
- Use hedging strategies
- Focus on high-reward pools
- Monitor positions regularly

## Performance Tracking

### Key Metrics
```
LP Performance Dashboard:
- Initial Investment: $10,000
- Current Value: $12,500
- Impermanent Loss: -$200 (-1.6%)
- Fees Earned: $450
- LM Rewards: $2,250
- Net P&L: +$2,500 (25%)
- Annualized Return: 101.4%
```

## Best Practices

### Getting Started
- Start with stable pairs to learn
- Begin with small amounts
- Understand IL before large commitments
- Monitor positions regularly

### Risk Management
- Never provide more than 20% of portfolio
- Diversify across multiple pools
- Keep emergency funds liquid
- Set IL tolerance limits