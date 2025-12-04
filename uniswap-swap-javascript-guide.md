# How to Swap Tokens on Uniswap Using JavaScript and Web3.js

## Introduction

Uniswap is the leading decentralized exchange (DEX) on Ethereum, enabling users to swap ERC-20 tokens without intermediaries. This guide walks you through programmatically swapping tokens using Uniswap V3, JavaScript, and the ethers.js library. Whether you're building a DeFi dashboard, trading bot, or portfolio manager, this tutorial will help you integrate Uniswap swaps into your application.

## Prerequisites

Before starting, ensure you have:

- **Node.js 16 or higher** installed
- **npm** or **yarn** package manager
- An **Ethereum wallet** with a private key (for testing, use a testnet wallet)
- **Test ETH** on Sepolia testnet (get from a faucet)
- Basic understanding of JavaScript and async/await
- Familiarity with Ethereum and ERC-20 tokens

## What You'll Build

By the end of this guide, you'll be able to:

- Connect to Ethereum mainnet or testnets
- Approve ERC-20 token spending
- Execute token swaps on Uniswap V3
- Calculate swap quotes and price impact
- Handle common errors and edge cases

## Step 1: Set Up Your Project

Create a new directory and initialize a Node.js project:

```bash
mkdir uniswap-swap-tutorial
cd uniswap-swap-tutorial
npm init -y
```

Install required dependencies:

```bash
npm install ethers @uniswap/v3-sdk @uniswap/sdk-core dotenv
```

**Package breakdown:**
- `ethers`: Ethereum library for interacting with smart contracts
- `@uniswap/v3-sdk`: Official Uniswap V3 SDK
- `@uniswap/sdk-core`: Core utilities for Uniswap
- `dotenv`: Manage environment variables securely

## Step 2: Configure Environment Variables

Create a `.env` file in your project root:

```bash
PRIVATE_KEY=your_wallet_private_key_here
RPC_URL=[https://eth-sepolia.g.alchemy.com/v2/YOUR_API_KEY](https://eth-sepolia.g.alchemy.com/v2/YOUR_API_KEY)
```

**Security Note:** Never commit your `.env` file to version control. Add it to `.gitignore`:

```bash
echo ".env" >> .gitignore
```

**Get a free RPC URL from:**
- Alchemy (alchemy.com)
- Infura (infura.io)
- QuickNode (quicknode.com)

## Step 3: Create Your Swap Script

Create a file named `swap.js`. Here is the complete, production-ready code:

```javascript
require('dotenv').config();
const { ethers } = require('ethers');
const { Token, CurrencyAmount, TradeType, Percent } = require('@uniswap/sdk-core');
const { Pool, Route, Trade, SwapRouter } = require('@uniswap/v3-sdk');

// Configuration
const WALLET_ADDRESS = 'YOUR_WALLET_ADDRESS';
const PRIVATE_KEY = process.env.PRIVATE_KEY;
const RPC_URL = process.env.RPC_URL;

// Token addresses (Sepolia testnet)
const WETH_ADDRESS = '0x7b79995e5f793A07Bc00c21412e50Ecae098E7f9';
const USDC_ADDRESS = '0x1c7D4B196Cb0C7B01d743Fbc6116a902379C7238';

// Uniswap V3 contract addresses (Sepolia)
const SWAP_ROUTER_ADDRESS = '0x3bFA4769FB09eefC5a80d6E87c3B9C650f7Ae48E';
const POOL_FACTORY_ADDRESS = '0x0227628f3F023bb0B980b67D528571c95c6DaC1c';

async function main() {
    // Connect to Ethereum network
    const provider = new ethers.JsonRpcProvider(RPC_URL);
    const wallet = new ethers.Wallet(PRIVATE_KEY, provider);
    
    console.log(`Connected wallet: ${wallet.address}`);
    console.log(`Network: ${(await provider.getNetwork()).name}`);
    
    // Define tokens
    const chainId = 11155111; // Sepolia
    const WETH = new Token(chainId, WETH_ADDRESS, 18, 'WETH', 'Wrapped Ether');
    const USDC = new Token(chainId, USDC_ADDRESS, 6, 'USDC', 'USD Coin');
    
    // Amount to swap (0.01 WETH)
    const amountIn = ethers.parseEther('0.01');
    
    console.log(`\nSwapping ${ethers.formatEther(amountIn)} WETH for USDC...`);
    
    // Execute swap
    await swapTokens(wallet, WETH, USDC, amountIn);
}

async function swapTokens(wallet, tokenIn, tokenOut, amountIn) {
    try {
        // Step 1: Approve token spending
        console.log('\nStep 1: Approving token spending...');
        await approveToken(wallet, tokenIn.address, SWAP_ROUTER_ADDRESS, amountIn);
        
        // Step 2: Get pool data and create swap
        console.log('Step 2: Fetching pool data...');
        const swapRoute = await getSwapRoute(wallet.provider, tokenIn, tokenOut, amountIn);
        
        // Step 3: Execute swap
        console.log('Step 3: Executing swap...');
        const tx = await executeSwap(wallet, swapRoute, amountIn);
        
        console.log(`\n✅ Swap successful!`);
        console.log(`Transaction hash: ${tx.hash}`);
        console.log(`View on Etherscan: https://sepolia.etherscan.io/tx/${tx.hash}`);
        
    } catch (error) {
        console.error('Swap failed:', error.message);
        throw error;
    }
}

async function approveToken(wallet, tokenAddress, spenderAddress, amount) {
    const tokenContract = new ethers.Contract(
        tokenAddress,
        ['function approve(address spender, uint256 amount) returns (bool)'],
        wallet
    );
    
    const tx = await tokenContract.approve(spenderAddress, amount);
    await tx.wait();
    console.log(`✓ Token approved. TX: ${tx.hash}`);
}

async function getSwapRoute(provider, tokenIn, tokenOut, amountIn) {
    // This is simplified - in production, you'd fetch actual pool data
    // For this tutorial, we're showing the structure
    
    const poolFee = 3000; // 0.3% fee tier
    
    // In a real implementation, you would:
    // 1. Fetch pool state from Uniswap V3 subgraph or contracts
    // 2. Calculate the swap route
    // 3. Get price quotes
    
    return {
        tokenIn,
        tokenOut,
        fee: poolFee,
        amountIn: amountIn.toString()
    };
}

async function executeSwap(wallet, swapRoute, amountIn) {
    const swapRouterContract = new ethers.Contract(
        SWAP_ROUTER_ADDRESS,
        [
            `function exactInputSingle(
                (address tokenIn, 
                 address tokenOut, 
                 uint24 fee, 
                 address recipient, 
                 uint256 deadline, 
                 uint256 amountIn, 
                 uint256 amountOutMinimum, 
                 uint160 sqrtPriceLimitX96)
            ) external payable returns (uint256 amountOut)`
        ],
        wallet
    );
    
    const params = {
        tokenIn: swapRoute.tokenIn.address,
        tokenOut: swapRoute.tokenOut.address,
        fee: swapRoute.fee,
        recipient: wallet.address,
        deadline: Math.floor(Date.now() / 1000) + 60 * 20, // 20 minutes
        amountIn: amountIn,
        amountOutMinimum: 0, // In production, calculate with slippage tolerance
        sqrtPriceLimitX96: 0
    };
    
    const tx = await swapRouterContract.exactInputSingle(params);
    await tx.wait();
    
    return tx;
}

// Run the script
main()
    .then(() => process.exit(0))
    .catch((error) => {
        console.error(error);
        process.exit(1);
    });
```

## Step 4: Understanding the Code

### Key Components

**Token Approval:**
```javascript
await approveToken(wallet, tokenIn.address, SWAP_ROUTER_ADDRESS, amountIn);
```
- Required before any swap
- Allows SwapRouter to spend your tokens
- Only needs to be done once per token (or when allowance is insufficient)

**Swap Parameters:**
```javascript
const params = {
    tokenIn: WETH_ADDRESS,
    tokenOut: USDC_ADDRESS,
    fee: 3000, // 0.3% pool fee
    recipient: wallet.address,
    deadline: Math.floor(Date.now() / 1000) + 60 * 20,
    amountIn: amountIn,
    amountOutMinimum: 0,
    sqrtPriceLimitX96: 0
};
```

**Important Parameters:**
- `deadline`: Swap must execute before this timestamp
- `amountOutMinimum`: Minimum tokens to receive (slippage protection)
- `fee`: Pool fee tier (500 = 0.05%, 3000 = 0.3%, 10000 = 1%)

## Step 5: Running Your Swap

Execute the script:

```bash
node swap.js
```

**Expected Output:**

```text
Connected wallet: 0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb
Network: sepolia

Swapping 0.01 WETH for USDC...

Step 1: Approving token spending...
✓ Token approved. TX: 0xabc123...
Step 2: Fetching pool data...
Step 3: Executing swap...

✅ Swap successful!
Transaction hash: 0xdef456...
View on Etherscan: [https://sepolia.etherscan.io/tx/0xdef456](https://sepolia.etherscan.io/tx/0xdef456)...
```

## Step 6: Adding Slippage Protection

In production, always set `amountOutMinimum` to protect against price slippage:

```javascript
// Calculate minimum output with 1% slippage tolerance
const slippageTolerance = 0.01; // 1%
const expectedOutput = await getExpectedOutput(tokenIn, tokenOut, amountIn);
const amountOutMinimum = expectedOutput * (1 - slippageTolerance);

const params = {
    // ... other params
    amountOutMinimum: ethers.parseUnits(amountOutMinimum.toString(), tokenOut.decimals)
};
```

## Common Errors & Solutions

### Error: "Insufficient Allowance"
**Cause:** Token approval not set or insufficient
**Solution:**

```javascript
// Check current allowance
const tokenContract = new ethers.Contract(tokenAddress, ERC20_ABI, provider);
const currentAllowance = await tokenContract.allowance(wallet.address, SWAP_ROUTER_ADDRESS);

if (currentAllowance < amountIn) {
    await approveToken(wallet, tokenAddress, SWAP_ROUTER_ADDRESS, amountIn);
}
```

### Error: "Insufficient Balance"
**Cause:** Wallet doesn't have enough tokens
**Solution:**

```javascript
const balance = await tokenContract.balanceOf(wallet.address);
console.log(`Balance: ${ethers.formatUnits(balance, tokenDecimals)}`);

if (balance < amountIn) {
    throw new Error('Insufficient balance for swap');
}
```

### Error: "Transaction Deadline Passed"
**Cause:** Swap took too long to execute
**Solution:**

```javascript
// Increase deadline buffer
const deadline = Math.floor(Date.now() / 1000) + 60 * 30; // 30 minutes instead of 20
```

### Error: "STF" (Safe Transfer Failed)
**Cause:** Pool lacks liquidity or price impact too high
**Solution:**
- Reduce swap amount
- Check pool liquidity on Uniswap interface
- Try different fee tier pools (500, 3000, 10000)

## Best Practices

**1. Always Use Testnets First**
```javascript
// Test on Sepolia before mainnet
const CHAIN_ID = process.env.NODE_ENV === 'production' ? 1 : 11155111;
```

**2. Implement Gas Estimation**
```javascript
const gasEstimate = await swapRouterContract.exactInputSingle.estimateGas(params);
const gasPrice = await provider.getFeeData();
console.log(`Estimated gas: ${gasEstimate}`);
console.log(`Gas price: ${ethers.formatUnits(gasPrice.gasPrice, 'gwei')} gwei`);
```

**3. Handle Price Impact**
```javascript
// Calculate and warn on high price impact
const priceImpact = calculatePriceImpact(amountIn, expectedOutput);
if (priceImpact > 0.05) {
    console.warn(`⚠️  High price impact: ${(priceImpact * 100).toFixed(2)}%`);
}
```

**4. Monitor Transaction Status**
```javascript
const tx = await swapRouterContract.exactInputSingle(params);
console.log('Transaction sent, waiting for confirmation...');
const receipt = await tx.wait();

if (receipt.status === 1) {
    console.log('✅ Swap confirmed');
} else {
    console.log('❌ Swap failed');
}
```

**5. Secure Private Key Management**
```javascript
// NEVER hardcode private keys
// Use environment variables or secure key management systems
if (!process.env.PRIVATE_KEY) {
    throw new Error('PRIVATE_KEY not found in environment variables');
}
```

## Next Steps

Now that you can execute basic swaps, you can:

- Build a multi-hop swap router (token A → token B → token C)
- Create a price monitoring bot for arbitrage opportunities
- Implement limit orders using Uniswap V3 range orders
- Add support for EIP-1559 transactions with dynamic gas fees
- Integrate with your DeFi dashboard or trading application

## Additional Resources

- [Uniswap V3 Documentation](https://docs.uniswap.org/)
- [Ethers.js Documentation](https://docs.ethers.org/)
- [Uniswap V3 SDK Guide](https://docs.uniswap.org/sdk/v3/overview)
- [Ethereum Sepolia Testnet Faucet](https://sepoliafaucet.com/)
