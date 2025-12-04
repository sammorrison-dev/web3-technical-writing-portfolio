# SimpleStaking Contract Documentation

## Overview

SimpleStaking is a Solidity smart contract that enables users to stake ERC-20 tokens and earn rewards over time. This contract implements a time-weighted staking mechanism where rewards accumulate based on stake duration and amount.

- **Contract Address (Sepolia):** `0x...` (deploy your own)
- **Solidity Version:** `^0.8.19`
- **License:** MIT

### Key Features
- Stake any ERC-20 token
- Time-weighted reward distribution
- No minimum stake amount
- Flexible unstaking (with optional time lock)
- Emergency withdrawal function
- Owner-controlled reward rate adjustment

## Contract Architecture

```text
SimpleStaking
├── Staking Functions
│   ├── stake()
│   ├── unstake()
│   └── claimRewards()
├── View Functions
│   ├── getStakedBalance()
│   ├── getPendingRewards()
│   └── getStakeInfo()
├── Admin Functions
│   ├── setRewardRate()
│   ├── depositRewards()
│   └── emergencyWithdraw()
└── Events
    ├── Staked
    ├── Unstaked
    └── RewardsClaimed
```

## Installation

Install OpenZeppelin contracts:

```bash
npm install @openzeppelin/contracts
```

## Full Contract Code

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

/**
 * @title SimpleStaking
 * @notice A staking contract for ERC-20 tokens with time-weighted rewards
 * @dev Implements ReentrancyGuard for security against reentrancy attacks
 */
contract SimpleStaking is ReentrancyGuard, Ownable {
    using SafeERC20 for IERC20;

    // ============ State Variables ============

    /// @notice The token users stake
    IERC20 public immutable stakingToken;

    /// @notice The token distributed as rewards
    IERC20 public immutable rewardToken;

    /// @notice Reward rate per second per token staked (in wei)
    uint256 public rewardRate;

    /// @notice Total tokens currently staked
    uint256 public totalStaked;

    /// @notice Timestamp of last reward calculation
    uint256 public lastUpdateTime;

    /// @notice Accumulated rewards per token
    uint256 public rewardPerTokenStored;

    /// @notice Minimum staking duration (optional, 0 = no lock)
    uint256 public lockDuration;

    // ============ Structs ============

    /// @notice Information about a user's stake
    struct StakeInfo {
        uint256 amount;
        uint256 startTime;
        uint256 rewardDebt;
        uint256 pendingRewards;
    }

    // ============ Mappings ============

    /// @notice User address => stake information
    mapping(address => StakeInfo) public stakes;

    /// @notice User address => reward per token paid
    mapping(address => uint256) public userRewardPerTokenPaid;

    // ============ Events ============

    /**
     * @notice Emitted when a user stakes tokens
     * @param user Address of the staker
     * @param amount Amount of tokens staked
     * @param timestamp Time of stake
     */
    event Staked(address indexed user, uint256 amount, uint256 timestamp);

    /**
     * @notice Emitted when a user unstakes tokens
     * @param user Address of the unstaker
     * @param amount Amount of tokens unstaked
     * @param timestamp Time of unstake
     */
    event Unstaked(address indexed user, uint256 amount, uint256 timestamp);

    /**
     * @notice Emitted when a user claims rewards
     * @param user Address of the claimer
     * @param amount Amount of rewards claimed
     */
    event RewardsClaimed(address indexed user, uint256 amount);

    /**
     * @notice Emitted when reward rate is updated
     * @param oldRate Previous reward rate
     * @param newRate New reward rate
     */
    event RewardRateUpdated(uint256 oldRate, uint256 newRate);

    // ============ Errors ============

    /// @notice Thrown when stake amount is zero
    error ZeroAmount();
    /// @notice Thrown when user has insufficient staked balance
    error InsufficientBalance();
    /// @notice Thrown when tokens are still locked
    error TokensLocked();
    /// @notice Thrown when no rewards are available
    error NoRewardsAvailable();

    // ============ Constructor ============

    /**
     * @notice Initializes the staking contract
     * @param _stakingToken Address of the ERC-20 token to stake
     * @param _rewardToken Address of the ERC-20 token for rewards
     * @param _rewardRate Initial reward rate per second per token
     * @param _lockDuration Minimum stake duration in seconds (0 for no lock)
     */
    constructor(
        address _stakingToken,
        address _rewardToken,
        uint256 _rewardRate,
        uint256 _lockDuration
    ) Ownable(msg.sender) {
        stakingToken = IERC20(_stakingToken);
        rewardToken = IERC20(_rewardToken);
        rewardRate = _rewardRate;
        lockDuration = _lockDuration;
        lastUpdateTime = block.timestamp;
    }

    // ============ Modifiers ============

    /**
     * @notice Updates reward calculations before function execution
     * @param _account Address to update rewards for
     */
    modifier updateReward(address _account) {
        rewardPerTokenStored = rewardPerToken();
        lastUpdateTime = block.timestamp;
        if (_account != address(0)) {
            stakes[_account].pendingRewards = earned(_account);
            userRewardPerTokenPaid[_account] = rewardPerTokenStored;
        }
        _;
    }

    // ============ External Functions ============

    /**
     * @notice Stake tokens to earn rewards
     * @param _amount Amount of tokens to stake
     * @dev Requires prior approval of staking token
     */
    function stake(uint256 _amount) external nonReentrant updateReward(msg.sender) {
        if (_amount == 0) revert ZeroAmount();
        stakes[msg.sender].amount += _amount;
        stakes[msg.sender].startTime = block.timestamp;
        totalStaked += _amount;
        stakingToken.safeTransferFrom(msg.sender, address(this), _amount);
        emit Staked(msg.sender, _amount, block.timestamp);
    }

    /**
     * @notice Unstake tokens and receive them back
     * @param _amount Amount of tokens to unstake
     */
    function unstake(uint256 _amount) external nonReentrant updateReward(msg.sender) {
        if (_amount == 0) revert ZeroAmount();
        if (stakes[msg.sender].amount < _amount) revert InsufficientBalance();
        
        if (lockDuration > 0) {
            if (block.timestamp < stakes[msg.sender].startTime + lockDuration) {
                revert TokensLocked();
            }
        }
        
        stakes[msg.sender].amount -= _amount;
        totalStaked -= _amount;
        stakingToken.safeTransfer(msg.sender, _amount);
        emit Unstaked(msg.sender, _amount, block.timestamp);
    }

    /**
     * @notice Claim accumulated rewards
     * @dev Transfers all pending rewards to the caller
     */
    function claimRewards() external nonReentrant updateReward(msg.sender) {
        uint256 reward = stakes[msg.sender].pendingRewards;
        if (reward == 0) revert NoRewardsAvailable();
        
        stakes[msg.sender].pendingRewards = 0;
        rewardToken.safeTransfer(msg.sender, reward);
        emit RewardsClaimed(msg.sender, reward);
    }

    /**
     * @notice Unstake all tokens and claim rewards in one transaction
     */
    function exit() external {
        unstake(stakes[msg.sender].amount);
        claimRewards();
    }

    // ============ View Functions ============

    /**
     * @notice Calculate current reward per token
     * @return Current accumulated reward per token (in wei)
     */
    function rewardPerToken() public view returns (uint256) {
        if (totalStaked == 0) {
            return rewardPerTokenStored;
        }
        return rewardPerTokenStored + (
            (block.timestamp - lastUpdateTime) * rewardRate * 1e18 / totalStaked
        );
    }

    /**
     * @notice Calculate pending rewards for an account
     * @param _account Address to check rewards for
     * @return Amount of pending rewards (in wei)
     */
    function earned(address _account) public view returns (uint256) {
        return (
            stakes[_account].amount *
            (rewardPerToken() - userRewardPerTokenPaid[_account]) / 1e18
        ) + stakes[_account].pendingRewards;
    }

    /**
     * @notice Get staked balance for an account
     * @param _account Address to check
     * @return Amount of tokens staked
     */
    function getStakedBalance(address _account) external view returns (uint256) {
        return stakes[_account].amount;
    }

    /**
     * @notice Get complete stake information for an account
     * @param _account Address to check
     */
    function getStakeInfo(address _account) external view returns (
        uint256 amount,
        uint256 startTime,
        uint256 pendingRewards,
        uint256 timeUntilUnlock
    ) {
        StakeInfo memory info = stakes[_account];
        uint256 unlockTime = info.startTime + lockDuration;
        
        return (
            info.amount,
            info.startTime,
            earned(_account),
            block.timestamp >= unlockTime ? 0 : unlockTime - block.timestamp
        );
    }

    // ============ Admin Functions ============

    /**
     * @notice Update the reward rate (owner only)
     * @param _newRate New reward rate per second per token
     */
    function setRewardRate(uint256 _newRate) external onlyOwner updateReward(address(0)) {
        uint256 oldRate = rewardRate;
        rewardRate = _newRate;
        emit RewardRateUpdated(oldRate, _newRate);
    }

    /**
     * @notice Deposit reward tokens into the contract (owner only)
     * @param _amount Amount of reward tokens to deposit
     */
    function depositRewards(uint256 _amount) external onlyOwner {
        rewardToken.safeTransferFrom(msg.sender, address(this), _amount);
    }

    /**
     * @notice Emergency withdraw of stuck tokens (owner only)
     * @param _token Address of token to withdraw
     * @param _amount Amount to withdraw
     */
    function emergencyWithdraw(address _token, uint256 _amount) external onlyOwner {
        require(
            _token != address(stakingToken) || _amount <= IERC20(_token).balanceOf(address(this)) - totalStaked,
            "Cannot withdraw staked tokens"
        );
        IERC20(_token).safeTransfer(msg.sender, _amount);
    }
}
```

## Function Reference

### User Functions

| Function | Parameters | Returns | Description |
| :--- | :--- | :--- | :--- |
| `stake` | `_amount`: tokens to stake | - | Stake tokens to earn rewards |
| `unstake` | `_amount`: tokens to unstake | - | Withdraw staked tokens |
| `claimRewards` | - | - | Claim accumulated rewards |
| `exit` | - | - | Unstake all and claim rewards |

### View Functions

| Function | Parameters | Returns | Description |
| :--- | :--- | :--- | :--- |
| `getStakedBalance` | `_account`: user address | `uint256` | Get user's staked amount |
| `earned` | `_account`: user address | `uint256` | Get pending rewards |
| `getStakeInfo` | `_account`: user address | `(uint256, uint256, uint256, uint256)` | Get complete stake details |
| `rewardPerToken` | - | `uint256` | Current reward per token |

### Admin Functions

| Function | Parameters | Access | Description |
| :--- | :--- | :--- | :--- |
| `setRewardRate` | `_newRate`: new rate | Owner | Update reward rate |
| `depositRewards` | `_amount`: tokens | Owner | Add reward tokens |
| `emergencyWithdraw` | `_token`, `_amount` | Owner | Rescue stuck tokens |

## Events Reference

### Staked
```solidity
event Staked(address indexed user, uint256 amount, uint256 timestamp);
```
Emitted when a user stakes tokens.
- **user**: Address of the staker
- **amount**: Amount of tokens staked
- **timestamp**: Block timestamp of the stake

### Unstaked
```solidity
event Unstaked(address indexed user, uint256 amount, uint256 timestamp);
```
Emitted when a user withdraws staked tokens.
- **user**: Address of the user
- **amount**: Amount of tokens withdrawn
- **timestamp**: Block timestamp of the withdrawal

### RewardsClaimed
```solidity
event RewardsClaimed(address indexed user, uint256 amount);
```
Emitted when a user claims their rewards.
- **user**: Address of the claimer
- **amount**: Amount of rewards claimed

## Error Reference

| Error | Cause | Solution |
| :--- | :--- | :--- |
| `ZeroAmount()` | Attempting to stake/unstake 0 tokens | Use amount > 0 |
| `InsufficientBalance()` | Unstaking more than staked | Check `getStakedBalance()` first |
| `TokensLocked()` | Unstaking before lock period ends | Wait for lock duration |
| `NoRewardsAvailable()` | Claiming with 0 pending rewards | Check `earned()` first |

## Integration Examples

### JavaScript (ethers.js)

```javascript
const { ethers } = require('ethers');

// Connect to contract
const stakingContract = new ethers.Contract(
    STAKING_ADDRESS,
    STAKING_ABI,
    signer
);

// Approve and stake
async function stakeTokens(amount) {
    const tokenContract = new ethers.Contract(TOKEN_ADDRESS, ERC20_ABI, signer);
    
    // Approve spending
    const approveTx = await tokenContract.approve(STAKING_ADDRESS, amount);
    await approveTx.wait();
    
    // Stake tokens
    const stakeTx = await stakingContract.stake(amount);
    await stakeTx.wait();
    console.log('Staked successfully!');
}

// Check rewards and claim
async function checkAndClaimRewards(userAddress) {
    const pending = await stakingContract.earned(userAddress);
    console.log(`Pending rewards: ${ethers.formatEther(pending)}`);
    
    if (pending > 0) {
        const claimTx = await stakingContract.claimRewards();
        await claimTx.wait();
        console.log('Rewards claimed!');
    }
}

// Get complete stake info
async function getStakeDetails(userAddress) {
    const [amount, startTime, pending, lockTime] = await stakingContract.getStakeInfo(userAddress);
    
    return {
        stakedAmount: ethers.formatEther(amount),
        stakedSince: new Date(Number(startTime) * 1000),
        pendingRewards: ethers.formatEther(pending),
        unlockIn: Number(lockTime)
    };
}
```

### React Hook Example

```javascript
import { useState, useEffect } from 'react';
import { useContract, useSigner } from 'wagmi';

function useStaking(stakingAddress) {
    const [stakeInfo, setStakeInfo] = useState(null);
    const { data: signer } = useSigner();
    
    const contract = useContract({
        address: stakingAddress,
        abi: STAKING_ABI,
        signerOrProvider: signer
    });

    const fetchStakeInfo = async (userAddress) => {
        const info = await contract.getStakeInfo(userAddress);
        setStakeInfo({
            amount: info[0],
            startTime: info[1],
            pending: info[2],
            lockTime: info[3]
        });
    };

    const stake = async (amount) => {
        const tx = await contract.stake(amount);
        await tx.wait();
        await fetchStakeInfo(signer.address);
    };

    const claimRewards = async () => {
        const tx = await contract.claimRewards();
        await tx.wait();
        await fetchStakeInfo(signer.address);
    };

    return { stakeInfo, stake, claimRewards, fetchStakeInfo };
}
```

## Security Considerations

### Audited Dependencies
This contract uses OpenZeppelin's audited libraries:
- **ReentrancyGuard:** Prevents reentrancy attacks
- **SafeERC20:** Safe token transfers
- **Ownable:** Access control

### Known Risks
- **Reward Token Depletion:** If reward token balance is insufficient, claims will fail. Monitor with `rewardToken.balanceOf(stakingContract)`.
- **Centralization Risk:** Owner can change reward rate and emergency withdraw. Consider adding timelock or multisig.
- **Front-running:** Large stakes/unstakes may be front-run. Consider adding commit-reveal scheme for large amounts.

## Deployment

### Using Hardhat

```javascript
// scripts/deploy.js
const { ethers } = require("hardhat");

async function main() {
    const SimpleStaking = await ethers.getContractFactory("SimpleStaking");
    
    const stakingContract = await SimpleStaking.deploy(
        "0x...", // staking token address
        "0x...", // reward token address
        ethers.parseEther("0.001"), // reward rate: 0.001 per second per token
        86400 * 7 // lock duration: 7 days
    );

    await stakingContract.waitForDeployment();
    console.log("SimpleStaking deployed to:", await stakingContract.getAddress());
}

main().catch(console.error);
```

### Verification

```bash
npx hardhat verify --network sepolia DEPLOYED_ADDRESS \
    "STAKING_TOKEN" "REWARD_TOKEN" "REWARD_RATE" "LOCK_DURATION"
```
## Additional Resources

- [OpenZeppelin Contracts Documentation](https://docs.openzeppelin.com/contracts/)
- [Solidity Documentation](https://docs.soliditylang.org/)
- [Hardhat Documentation](https://hardhat.org/docs)
- [ERC-20 Token Standard](https://eips.ethereum.org/EIPS/eip-20)
