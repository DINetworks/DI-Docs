# Getting Started for Users

Welcome to DI Network! This comprehensive guide will walk you through everything you need to know to start using the protocol, from setting up your wallet to making your first trade.

## Prerequisites

Before you begin, make sure you have:

- [ ] **Web3 Wallet**: MetaMask, WalletConnect, or similar
- [ ] **Supported Network**: Access to Ethereum, BSC, Polygon, Arbitrum, Base, or Crossfi
- [ ] **Base Tokens**: Native tokens (ETH, BNB, MATIC) for gas fees
- [ ] **Basic DeFi Knowledge**: Understanding of wallets, tokens, and transactions

## Step 1: Set Up Your Wallet

### Install MetaMask (Recommended)

1. Visit [metamask.io](https://metamask.io) and install the browser extension
2. Create a new wallet or import existing seed phrase
3. **Security**: Write down your seed phrase and store it safely
4. Add the networks you want to use

### Add Supported Networks

Add these networks to your MetaMask:

{% tabs %}
{% tab title="Ethereum" %}
- **Network Name**: Ethereum Mainnet
- **RPC URL**: https://mainnet.infura.io/v3/YOUR_KEY
- **Chain ID**: 1
- **Currency Symbol**: ETH
- **Block Explorer**: https://etherscan.io
{% endtab %}

{% tab title="BSC" %}
- **Network Name**: BNB Smart Chain
- **RPC URL**: https://bsc-dataseed.binance.org/
- **Chain ID**: 56
- **Currency Symbol**: BNB
- **Block Explorer**: https://bscscan.com
{% endtab %}

{% tab title="Polygon" %}
- **Network Name**: Polygon Mainnet
- **RPC URL**: https://polygon-rpc.com/
- **Chain ID**: 137
- **Currency Symbol**: MATIC
- **Block Explorer**: https://polygonscan.com
{% endtab %}

{% tab title="Arbitrum" %}
- **Network Name**: Arbitrum One
- **RPC URL**: https://arb1.arbitrum.io/rpc
- **Chain ID**: 42161
- **Currency Symbol**: ETH
- **Block Explorer**: https://arbiscan.io
{% endtab %}
{% endtabs %}

## Step 2: Connect to DI Network

### Access the Application

1. Visit [app.dinetwork.xyz](https://app.dinetwork.xyz)
2. Click "Connect Wallet" in the top right corner
3. Select your wallet (MetaMask, WalletConnect, etc.)
4. Approve the connection request
5. Select your preferred network

### Verify Connection

Once connected, you should see:
- Your wallet address in the top right
- Your current network
- Your token balances
- Available protocol features

## Step 3: Acquire DI Tokens

You need DI tokens to use most protocol features. Here are several ways to get them:

### Option A: Buy on Decentralized Exchanges

{% tabs %}
{% tab title="Ethereum" %}
**Uniswap V3**
1. Go to [app.uniswap.org](https://app.uniswap.org)
2. Connect your wallet
3. Select ETH → DI
4. Enter amount and swap
5. Confirm transaction
{% endtab %}

{% tab title="BSC" %}
**PancakeSwap**
1. Go to [pancakeswap.finance](https://pancakeswap.finance)
2. Connect your wallet
3. Select BNB → DI
4. Enter amount and swap
5. Confirm transaction
{% endtab %}

{% tab title="Polygon" %}
**QuickSwap**
1. Go to [quickswap.exchange](https://quickswap.exchange)
2. Connect your wallet
3. Select MATIC → DI
4. Enter amount and swap
5. Confirm transaction
{% endtab %}
{% endtabs %}

### Option B: Centralized Exchanges

If available, you can buy DI tokens on:
- **Binance**: DI/USDT trading pair
- **Coinbase**: DI/USD trading pair
- **OKX**: DI/USDT trading pair

### Option C: Participate in Token Sales

Check [dinetwork.xyz](https://dinetwork.xyz) for:
- Public token sales
- Community airdrops
- Liquidity mining programs
- Staking rewards

## Step 4: Mint DUSD Stablecoin

DUSD is the base currency for all trading activities. Here's how to mint it:

### Understanding Collateral

You can use these tokens as collateral:

| Token | Collateral Factor | Liquidation Threshold |
| --- | --- | --- |
| **DI** | 75% | 80% |
| **WBTC** | 70% | 75% |
| **WETH** | 70% | 75% |
| **USDT** | 90% | 95% |
| **USDC** | 90% | 95% |

### Minting Process

1. **Navigate to Mint Section**
   - Go to the "Mint" tab in the app
   - Select your collateral token (DI recommended)

2. **Deposit Collateral**
   - Enter the amount of collateral to deposit
   - Click "Approve" to allow the contract to use your tokens
   - Confirm the approval transaction

3. **Mint DUSD**
   - Enter the amount of DUSD to mint (up to 75% of collateral value)
   - Review the interest rate (5% APR)
   - Click "Mint DUSD"
   - Confirm the transaction

4. **Monitor Your Position**
   - Check your collateral ratio regularly
   - Ensure it stays above the liquidation threshold
   - Add more collateral if needed

### Example Calculation

```
Collateral: 1,000 DI tokens @ $2.00 each = $2,000
Collateral Factor: 75%
Max DUSD to Mint: $2,000 × 0.75 = $1,500 DUSD
Safe Amount: $1,200 DUSD (80% of max for safety buffer)
```

## Step 5: Start Trading

Now you can access all DI Network trading features:

### Synthetic Asset Trading (DSwap)

Trade synthetic versions of real-world assets:

1. **Navigate to DSwap**
   - Go to the "Trade" section
   - Select "Synthetic Assets"

2. **Choose Your Assets**
   - **Stocks**: xAAPL, xTSLA, xGOOG, xAMZN
   - **Commodities**: xGold, xSilver, xOil
   - **Crypto**: xBTC, xETH, xBNB
   - **Forex**: xEUR, xGBP, xJPY

3. **Execute Trades**
   - **Mint**: DUSD → Synthetic Asset
   - **Burn**: Synthetic Asset → DUSD
   - **Swap**: Synthetic Asset A → Synthetic Asset B

### Perpetual Trading (DPerp)

Trade with leverage up to 50x:

1. **Navigate to DPerp**
   - Go to the "Trade" section
   - Select "Perpetuals"

2. **Open Position**
   - Select asset (BTC, ETH, etc.)
   - Choose direction (Long/Short)
   - Set leverage (1.1x to 50x)
   - Enter position size
   - Set acceptable price
   - Confirm transaction

3. **Manage Position**
   - Monitor PnL in real-time
   - Add/remove collateral
   - Partially close positions
   - Set stop-loss/take-profit

## Step 6: Earn Rewards

Multiple ways to earn passive income:

### Staking DI Tokens

Earn 8-20% APY by staking DI tokens:

1. **Navigate to Staking**
   - Go to the "Earn" section
   - Select "Staking"

2. **Choose Lock Period**
   - No lock: 8% APY
   - 3 months: 9.6% APY
   - 6 months: 12% APY
   - 12 months: 16% APY
   - 24 months: 20% APY

3. **Stake Tokens**
   - Enter amount to stake
   - Select lock period
   - Approve and confirm transaction

### Liquidity Provision

Provide DUSD liquidity to earn fees:

1. **Navigate to Liquidity**
   - Go to the "Earn" section
   - Select "Liquidity"

2. **Add Liquidity**
   - Enter DUSD amount
   - Review APY and risks
   - Confirm transaction
   - Receive DLP tokens

3. **Monitor Performance**
   - Track earnings from trading fees
   - Monitor pool utilization
   - Withdraw after cooldown period (15 minutes)

## Step 7: Advanced Features

### Cross-Chain Operations

Transfer assets between supported chains:

1. **Navigate to Bridge**
   - Go to the "Bridge" section
   - Select source and destination chains

2. **Bridge Assets**
   - Choose token to bridge
   - Enter amount
   - Set destination address
   - Pay bridge fee
   - Confirm transaction

### Gasless Transactions

Use DUSD to pay for gas fees:

1. **Deposit Gas Credits**
   - Go to "Gas Credits" section
   - Deposit DUSD for gas credits
   - Credits work across all chains

2. **Execute Gasless Transactions**
   - Sign transactions without native tokens
   - Gas fees automatically deducted from credits
   - Works for all protocol operations

### Governance Participation

Participate in protocol governance:

1. **Delegate Voting Power**
   - Go to "Governance" section
   - Delegate to yourself or others
   - Voting power = DI token balance

2. **Vote on Proposals**
   - Review active proposals
   - Vote yes/no on changes
   - Earn governance rewards

## Risk Management

### Position Health Monitoring

Always monitor your positions:

- **Collateral Ratio**: Keep above liquidation threshold
- **Interest Accrual**: Track growing debt from interest
- **Market Conditions**: Watch for volatility
- **Liquidation Risk**: Add collateral if ratio drops

### Risk Mitigation Strategies

1. **Conservative Leverage**: Start with low leverage (2-5x)
2. **Diversification**: Don't put all funds in one position
3. **Stop Losses**: Set automatic exit points
4. **Regular Monitoring**: Check positions daily
5. **Emergency Funds**: Keep extra collateral available

### Common Mistakes to Avoid

- **Over-leveraging**: Using maximum leverage without experience
- **Ignoring Interest**: Forgetting about accruing interest on DUSD
- **Poor Timing**: Entering positions during high volatility
- **No Exit Strategy**: Not planning when to close positions
- **Insufficient Collateral**: Operating too close to liquidation threshold

## Troubleshooting

### Common Issues

**Transaction Fails**
- Check gas fees and network congestion
- Ensure sufficient token balance
- Verify contract approvals

**Can't Connect Wallet**
- Clear browser cache and cookies
- Try different browser or incognito mode
- Update wallet extension

**Position Liquidated**
- Collateral ratio fell below threshold
- Add more collateral next time
- Monitor positions more closely

**High Gas Fees**
- Use Layer 2 networks (Polygon, Arbitrum)
- Wait for lower network congestion
- Use gasless transactions with DUSD credits

### Getting Help

- **Discord**: Join [discord.gg/dinetwork](https://discord.gg/dinetwork)
- **Documentation**: Browse this documentation
- **FAQ**: Check [Frequently Asked Questions](../resources/faq.md)
- **Support**: Contact support through official channels

## Next Steps

Now that you're set up, explore these advanced topics:

{% content-ref url="getting-started.md" %}
[getting-started.md](getting-started.md)
{% endcontent-ref %}

---

{% hint style="success" %}
**Congratulations!** You're now ready to use DI Network. Start small, learn the system, and gradually increase your involvement as you become more comfortable with the protocol.
{% endhint %}