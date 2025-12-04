# How to Connect an Ethereum Wallet to Your Web App

## Introduction

Wallet connection is the foundation of every Web3 application. Before users can interact with smart contracts, mint NFTs, or swap tokens, they need to connect their Ethereum wallet to your dApp. This guide shows you how to implement wallet connection using MetaMask and WalletConnect, supporting both desktop and mobile users.



By the end of this tutorial, you'll have a production-ready wallet connection component that handles multiple wallet providers, network switching, and common edge cases.

## Prerequisites

Before starting, ensure you have:

- **Node.js 16 or higher** installed
- **React 18 or higher** experience
- Basic understanding of Ethereum wallets
- Familiarity with React hooks and async/await
- A code editor (VS Code recommended)

## What You'll Build

A complete wallet connection system with:

- MetaMask integration for desktop browsers
- WalletConnect support for mobile wallets
- Network switching (mainnet, testnets)
- Account change detection
- Disconnect functionality
- Error handling for common issues

## Step 1: Set Up Your React Project

Create a new React app:

```bash
npx create-react-app wallet-connection-demo
cd wallet-connection-demo
```

Install required dependencies:

```bash
npm install ethers@6 @web3modal/ethers5 @walletconnect/modal
```

**Package breakdown:**
- `ethers`: Ethereum library for wallet interactions
- `@web3modal/ethers5`: UI component for WalletConnect
- `@walletconnect/modal`: Modal interface for wallet selection

## Step 2: Create Wallet Context

Create `src/contexts/WalletContext.js`. This manages the global state of the wallet connection.

```javascript
import React, { createContext, useContext, useState, useEffect } from 'react';
import { ethers } from 'ethers';

const WalletContext = createContext();

export const useWallet = () => {
  const context = useContext(WalletContext);
  if (!context) {
    throw new Error('useWallet must be used within WalletProvider');
  }
  return context;
};

export const WalletProvider = ({ children }) => {
  const [account, setAccount] = useState(null);
  const [chainId, setChainId] = useState(null);
  const [provider, setProvider] = useState(null);
  const [isConnecting, setIsConnecting] = useState(false);
  const [error, setError] = useState(null);

  // Check if wallet is already connected on page load
  useEffect(() => {
    checkIfWalletConnected();
  }, []);

  // Listen for account changes
  useEffect(() => {
    if (window.ethereum) {
      window.ethereum.on('accountsChanged', handleAccountsChanged);
      window.ethereum.on('chainChanged', handleChainChanged);
      return () => {
        window.ethereum.removeListener('accountsChanged', handleAccountsChanged);
        window.ethereum.removeListener('chainChanged', handleChainChanged);
      };
    }
  }, []);

  const checkIfWalletConnected = async () => {
    try {
      if (!window.ethereum) return;
      const accounts = await window.ethereum.request({
        method: 'eth_accounts'
      });
      if (accounts.length > 0) {
        await connectWallet();
      }
    } catch (err) {
      console.error('Error checking wallet connection:', err);
    }
  };

  const connectWallet = async () => {
    setIsConnecting(true);
    setError(null);
    try {
      // Check if MetaMask is installed
      if (!window.ethereum) {
        throw new Error('MetaMask is not installed. Please install it to use this app.');
      }

      // Request account access
      const accounts = await window.ethereum.request({
        method: 'eth_requestAccounts'
      });

      // Create ethers provider
      const ethersProvider = new ethers.BrowserProvider(window.ethereum);
      const network = await ethersProvider.getNetwork();

      setAccount(accounts[0]);
      setChainId(network.chainId.toString());
      setProvider(ethersProvider);

      console.log('Connected to wallet:', accounts[0]);
      console.log('Network:', network.name);
    } catch (err) {
      console.error('Error connecting wallet:', err);
      setError(err.message);
    } finally {
      setIsConnecting(false);
    }
  };

  const disconnectWallet = () => {
    setAccount(null);
    setChainId(null);
    setProvider(null);
    setError(null);
  };

  const switchNetwork = async (targetChainId) => {
    try {
      await window.ethereum.request({
        method: 'wallet_switchEthereumChain',
        params: [{ chainId: `0x${targetChainId.toString(16)}` }],
      });
    } catch (err) {
      // This error code indicates that the chain has not been added to MetaMask
      if (err.code === 4902) {
        console.error('Please add this network to MetaMask');
      }
      throw err;
    }
  };

  const handleAccountsChanged = (accounts) => {
    if (accounts.length === 0) {
      disconnectWallet();
    } else {
      setAccount(accounts[0]);
    }
  };

  const handleChainChanged = (chainId) => {
    // Convert hex chainId to decimal
    setChainId(parseInt(chainId, 16).toString());
    // Reload to avoid state issues
    window.location.reload();
  };

  const value = {
    account,
    chainId,
    provider,
    isConnecting,
    error,
    connectWallet,
    disconnectWallet,
    switchNetwork
  };

  return (
    <WalletContext.Provider value={value}>
      {children}
    </WalletContext.Provider>
  );
};
```

## Step 3: Create Wallet Connection Component

Create `src/components/WalletConnect.js`. This component handles the UI for connecting and switching networks.

```javascript
import React from 'react';
import { useWallet } from '../contexts/WalletContext';
import './WalletConnect.css';

const NETWORKS = {
  '1': 'Ethereum Mainnet',
  '11155111': 'Sepolia Testnet',
  '137': 'Polygon Mainnet',
  '80001': 'Mumbai Testnet'
};

const WalletConnect = () => {
  const {
    account,
    chainId,
    isConnecting,
    error,
    connectWallet,
    disconnectWallet,
    switchNetwork
  } = useWallet();

  const formatAddress = (address) => {
    if (!address) return '';
    return `${address.substring(0, 6)}...${address.substring(address.length - 4)}`;
  };

  const handleNetworkSwitch = async (targetChainId) => {
    try {
      await switchNetwork(targetChainId);
    } catch (err) {
      alert('Failed to switch network: ' + err.message);
    }
  };

  if (account) {
    return (
      <div className="wallet-container">
        <div className="wallet-info">
          <div className="network-badge">
            {NETWORKS[chainId] || `Chain ID: ${chainId}`}
          </div>
          <div className="account-address">
            {formatAddress(account)}
          </div>
          <button className="disconnect-btn" onClick={disconnectWallet}>
            Disconnect
          </button>
        </div>
        
        <div className="network-switcher">
          <h3>Switch Network:</h3>
          <div className="network-buttons">
            <button onClick={() => handleNetworkSwitch(1)}>
              Mainnet
            </button>
            <button onClick={() => handleNetworkSwitch(11155111)}>
              Sepolia
            </button>
            <button onClick={() => handleNetworkSwitch(137)}>
              Polygon
            </button>
          </div>
        </div>
      </div>
    );
  }

  return (
    <div className="wallet-container">
      <button
        className="connect-btn"
        onClick={connectWallet}
        disabled={isConnecting}
      >
        {isConnecting ? 'Connecting...' : 'Connect Wallet'}
      </button>
      
      {error && (
        <div className="error-message">
          {error}
        </div>
      )}
      
      {!window.ethereum && (
        <div className="install-message">
          <p>MetaMask not detected.</p>
          <a
            href="[https://metamask.io/download/](https://metamask.io/download/)"
            target="_blank"
            rel="noopener noreferrer"
          >
            Install MetaMask
          </a>
        </div>
      )}
    </div>
  );
};

export default WalletConnect;
```

## Step 4: Add Styling

Create `src/components/WalletConnect.css`:

```css
.wallet-container {
  max-width: 500px;
  margin: 2rem auto;
  padding: 2rem;
  background: #f8f9fa;
  border-radius: 12px;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
}

.connect-btn {
  width: 100%;
  padding: 1rem 2rem;
  font-size: 1.1rem;
  font-weight: 600;
  color: white;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  border: none;
  border-radius: 8px;
  cursor: pointer;
  transition: transform 0.2s;
}

.connect-btn:hover:not(:disabled) {
  transform: translateY(-2px);
}

.connect-btn:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

.wallet-info {
  display: flex;
  flex-direction: column;
  gap: 1rem;
  align-items: center;
}

.network-badge {
  padding: 0.5rem 1rem;
  background: #4caf50;
  color: white;
  border-radius: 20px;
  font-size: 0.9rem;
  font-weight: 600;
}

.account-address {
  font-size: 1.2rem;
  font-weight: 600;
  font-family: 'Courier New', monospace;
  color: #333;
}

.disconnect-btn {
  padding: 0.5rem 1.5rem;
  background: #f44336;
  color: white;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  font-weight: 600;
}

.disconnect-btn:hover {
  background: #d32f2f;
}

.network-switcher {
  margin-top: 2rem;
  padding-top: 2rem;
  border-top: 1px solid #ddd;
}

.network-switcher h3 {
  margin-bottom: 1rem;
  color: #555;
}

.network-buttons {
  display: flex;
  gap: 0.5rem;
  flex-wrap: wrap;
}

.network-buttons button {
  padding: 0.5rem 1rem;
  background: #2196f3;
  color: white;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  font-weight: 600;
}

.network-buttons button:hover {
  background: #1976d2;
}

.error-message {
  margin-top: 1rem;
  padding: 1rem;
  background: #ffebee;
  color: #c62828;
  border-radius: 6px;
  font-weight: 500;
}

.install-message {
  margin-top: 1rem;
  padding: 1rem;
  background: #fff3e0;
  border-radius: 6px;
  text-align: center;
}

.install-message a {
  display: inline-block;
  margin-top: 0.5rem;
  padding: 0.5rem 1rem;
  background: #ff6f00;
  color: white;
  text-decoration: none;
  border-radius: 6px;
  font-weight: 600;
}

.install-message a:hover {
  background: #e65100;
}
```

## Step 5: Update App.js

Replace `src/App.js` to use the provider and component:

```javascript
import React from 'react';
import { WalletProvider } from './contexts/WalletContext';
import WalletConnect from './components/WalletConnect';
import './App.css';

function App() {
  return (
    <WalletProvider>
      <div className="App">
        <header className="App-header">
          <h1>Web3 Wallet Connection Demo</h1>
          <p>Connect your Ethereum wallet to get started</p>
        </header>
        <main>
          <WalletConnect />
        </main>
        <footer>
          <p>Built with React and ethers.js</p>
        </footer>
      </div>
    </WalletProvider>
  );
}

export default App;
```

Update `src/App.css` for basic layout:

```css
.App {
  min-height: 100vh;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
}

.App-header {
  padding: 3rem 1rem;
  text-align: center;
  color: white;
}

.App-header h1 {
  font-size: 2.5rem;
  margin-bottom: 0.5rem;
}

.App-header p {
  font-size: 1.2rem;
  opacity: 0.9;
}

main {
  padding: 2rem 1rem;
}

footer {
  text-align: center;
  padding: 2rem;
  color: white;
  opacity: 0.8;
}
```

## Step 6: Run Your Application

Start the development server:

```bash
npm start
```

Your app will open at `http://localhost:3000`

**Test the functionality:**
1. Click "Connect Wallet"
2. MetaMask popup appears
3. Approve connection
4. See your address and network
5. Try switching networks
6. Click "Disconnect"

## Understanding the Code

### Wallet Context Pattern
The Context API provides wallet state globally:

```javascript
const { account, provider, connectWallet } = useWallet();
```

**Benefits:**
- Any component can access wallet data
- No prop drilling
- Centralized state management
- Easy to add features later

### Event Listeners
Listen for wallet changes:

```javascript
window.ethereum.on('accountsChanged', handleAccountsChanged);
window.ethereum.on('chainChanged', handleChainChanged);
```

**Why this matters:**
- User switches accounts in MetaMask → your UI updates
- User changes networks → your app reacts
- Prevents stale data issues

### Network Switching

```javascript
await window.ethereum.request({
  method: 'wallet_switchEthereumChain',
  params: [{ chainId: '0x1' }], // Hex format required
});
```

**Chain ID formats:**
- MetaMask requires hexadecimal: `0x1` (not `1`)
- ethers.js returns decimal: `1` (not `0x1`)
- **Always convert between formats**

## Common Errors & Solutions

### Error: "MetaMask is not installed"
**Cause:** User doesn't have MetaMask browser extension
**Solution:**

```javascript
if (!window.ethereum) {
  return (
    <a href="[https://metamask.io/download/](https://metamask.io/download/)" target="_blank">
      Install MetaMask
    </a>
  );
}
```

### Error: "User rejected the request"
**Cause:** User clicked "Cancel" in MetaMask popup
**Solution:**

```javascript
catch (err) {
  if (err.code === 4001) {
    setError('Connection cancelled. Please try again.');
  }
}
```

### Error: "Already processing eth_requestAccounts"
**Cause:** Multiple connection requests sent simultaneously
**Solution:**

```javascript
const [isConnecting, setIsConnecting] = useState(false);
if (isConnecting) return; // Prevent duplicate requests
```

### Network Mismatch Issues
**Cause:** App expects mainnet, user connected to testnet
**Solution:**

```javascript
const REQUIRED_CHAIN_ID = '1'; // Mainnet
if (chainId !== REQUIRED_CHAIN_ID) {
  await switchNetwork(parseInt(REQUIRED_CHAIN_ID));
}
```

## Best Practices

**1. Always Check for MetaMask**

```javascript
if (typeof window.ethereum === 'undefined') {
  alert('Please install MetaMask to use this application');
  return;
}
```

**2. Handle Account Changes**

```javascript
// User might switch accounts while using your app
window.ethereum.on('accountsChanged', (accounts) => {
  if (accounts.length === 0) {
    // User disconnected all accounts
    handleDisconnect();
  } else {
    // User switched to different account
    setAccount(accounts[0]);
  }
});
```

**3. Cleanup Event Listeners**

```javascript
useEffect(() => {
  window.ethereum.on('accountsChanged', handleAccountsChanged);
  return () => {
    // Prevent memory leaks
    window.ethereum.removeListener('accountsChanged', handleAccountsChanged);
  };
}, []);
```

**4. Store Connection State**

```javascript
// Remember if user was connected
useEffect(() => {
  if (account) {
    localStorage.setItem('walletConnected', 'true');
  } else {
    localStorage.removeItem('walletConnected');
  }
}, [account]);
```

**5. Show Loading States**

```javascript
<button disabled={isConnecting}>
  {isConnecting ? 'Connecting...' : 'Connect Wallet'}
</button>
```

## Next Steps

Now that you have wallet connection working, you can:

- Add support for more wallets (Coinbase Wallet, WalletConnect)
- Implement transaction signing
- Read data from smart contracts
- Display user's token balances
- Build a complete dApp with wallet integration

## Additional Resources

- [MetaMask Documentation](https://docs.metamask.io/)
- [ethers.js Documentation](https://docs.ethers.org/)
- [EIP-1193: Ethereum Provider API](https://eips.ethereum.org/EIPS/eip-1193)
- [React Context API Guide](https://react.dev/learn/passing-data-deeply-with-context)
