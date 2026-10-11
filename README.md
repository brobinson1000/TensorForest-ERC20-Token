<p align="center">
  <img src="assets/tsaki-logo.svg" alt="TFRST logo" width="720" />
</p>

# TFRST (Tensor Forest) ERC-20 Governance Token

TFRST is an ERC-20 governance token built on Ethereum. I made it as a learning project to understand how cryptocurrencies work, how smart contracts are written and deployed, and how on-chain governance works in the style of tokens like Uniswap's UNI.

Holders of TFRST can delegate voting power, create proposals, vote, and have passed proposals executed through a timelock. No single person controls the token once deployment is finished.

> **Disclaimer:** TFRST is an educational project. It has no promised value, is not an investment, and has not been professionally audited. Do not use this code with real funds without a full security review.

---

## Table of Contents

- [Features](#features)
- [Token Details](#token-details)
- [How Governance Works](#how-governance-works)
- [Governance Parameters](#governance-parameters)
- [Contracts](#contracts)
- [Tech Stack](#tech-stack)
- [Project Structure](#project-structure)
- [Getting Started](#getting-started)
- [Compile and Test](#compile-and-test)
- [Deploy](#deploy)
- [Using TFRST](#using-tski)
- [Design Decisions](#design-decisions)
- [What I Learned](#what-i-learned)
- [License](#license)

---

## Features

- **Standard ERC-20:** `transfer`, `approve`, `transferFrom`, `balanceOf`, `allowance`, `totalSupply`
- **Voting power (`ERC20Votes`):** holders delegate votes, and balances are checkpointed per block so past votes can't be manipulated
- **Gasless approvals (`ERC20Permit`, EIP-2612):** approve spending with a signed message instead of a transaction
- **Rate-limited minting:** only the timelock can mint, at most once every 365 days, and no more than 2% of total supply per mint
- **On-chain governance (`TsakiGovernor`):** proposals, voting, and quorum, built on OpenZeppelin's Governor
- **Timelock (`TsakiTimelock`):** every passed proposal waits 7 days before it can run, giving holders time to react
- **No admin backdoors:** no pause, no blacklist, no burn restriction, no rescue function, and no upgradeable proxy
- **Ownership by governance:** the timelock owns the token, and the deployer gives up all control after setup

## Token Details

| Property         | Value                                                              |
| ---------------- | ------------------------------------------------------------------ |
| Name             | Tensor Forest                                                      |
| Symbol           | TFRST                                                               |
| Decimals         | 18                                                                 |
| Initial Supply   | `YOUR_INITIAL_SUPPLY`                                              |
| Network          | Ethereum (`YOUR_NETWORK`)                                          |
| Token Address    | `0xYOUR_TOKEN_ADDRESS`                                             |
| Governor Address | `0xYOUR_GOVERNOR_ADDRESS`                                          |
| Timelock Address | `0xYOUR_TIMELOCK_ADDRESS`                                          |
| Etherscan        | [View token](https://etherscan.io/address/0xYOUR_TOKEN_ADDRESS)    |

## How Governance Works

1. **Delegate.** A holder calls `delegate(address)` on the token, usually delegating to themselves. Tokens have no voting power until this is done.
2. **Propose.** An address with enough delegated voting power calls `propose(...)` on the Governor, describing an action such as minting tokens.
3. **Vote.** After the voting delay, delegated holders call `castVote(proposalId, support)` with `0` = Against, `1` = For, `2` = Abstain. Votes are weighted by tokens.
4. **Pass.** The proposal succeeds if quorum is reached and there are more For votes than Against votes.
5. **Queue.** Anyone queues a successful proposal in the timelock.
6. **Wait.** The timelock delay must pass (7 days).
7. **Execute.** Anyone executes the proposal, and the timelock calls the token.

Voting power is read from a snapshot taken when voting opens, so holders must delegate **before** a proposal is created for their tokens to count on it.

## Governance Parameters

| Setting             | Value                                                      |
| ------------------- | ---------------------------------------------------------- |
| Voting delay        | 7,200 blocks (about 1 day)                                 |
| Voting period       | 7,200 blocks (about 1 day)                                 |
| Proposal threshold  | 5,000,000 TFRST of voting power (5% of a 100M supply)       |
| Quorum              | 5% of total supply (For + Abstain votes)                   |
| Timelock delay      | 604,800 seconds (7 days)                                   |
| Mint interval       | At least 365 days between mints                            |
| Mint size limit     | At most 2% of total supply per mint                        |

## Contracts

| Contract         | File                          | Purpose                                                      |
| ---------------- | ----------------------------- | ------------------------------------------------------------ |
| `TFRST`          | `contracts/Tsaki.sol`         | The ERC-20 token with voting, permit, and rate-limited mint  |
| `TFRSTGovernor`  | `contracts/TsakiGovernor.sol` | Proposals, voting, and quorum                                |
| `TFRSTTimelock`  | `contracts/TsakiTimelock.sol` | Delays and executes approved actions; owns the token         |

## Tech Stack

- **Solidity** (`^0.8.20`): smart contracts
- **TypeScript**: deployment scripts and tests
- **Hardhat 3**: compile, test, and deploy
- **viem**: contract interaction from TypeScript
- **OpenZeppelin Contracts v5**: ERC-20, Votes, Permit, Governor, and TimelockController
- **Node.js**: runtime

## Project Structure

```
.
├── assets/
│   └── tfrst-logo.svg
├── contracts/
│   ├── TFRST.sol
│   ├── TFRSTGovernor.sol
│   └── TFRSTTimelock.sol
├── scripts/
│   ├── deployTimelock.ts
│   └── deployGovernance.ts
├── test/
├── hardhat.config.ts
├── package.json
├── tsconfig.json
└── README.md
```

## Getting Started

### Prerequisites

- [Node.js](https://nodejs.org/) (a version supported by Hardhat 3, 22 LTS or newer recommended)
- npm
- A wallet with ETH for gas (I used testnet ETH for Sepolia)
- An RPC provider URL (I used Alchemy or you can use Infura, etc.)
- An Etherscan API key (optional, for verification)

### Installation

```bash
git clone https://github.com/YOUR_USERNAME/YOUR_REPO.git
cd YOUR_REPO
npm install
```

### Configuration

Hardhat 3 supports an encrypted keystore, so secrets don't need to live in a plain `.env` file:

```bash
npx hardhat keystore set SEPOLIA_RPC_URL
npx hardhat keystore set SEPOLIA_PRIVATE_KEY
npx hardhat keystore set ETHERSCAN_API_KEY
```

## Compile and Test

```bash
npx hardhat build
npx hardhat test
```

## Deploy

Recommended order:

1. Deploy the **timelock** (you are temporary proposer and admin).
2. Deploy the **token** with the timelock as owner.
3. Deploy the **Governor**, passing the token and timelock addresses.
4. Grant the Governor the **proposer** and **canceller** roles on the timelock.
5. Revoke your own timelock roles and renounce admin.

```bash
npx hardhat run scripts/deployGovernance.ts --network sepolia
```

Verify contracts on Etherscan:

```bash
npx hardhat verify --network sepolia 0xYOUR_TOKEN_ADDRESS <constructor args>
```

## Using TFRST

### Add TFRST to MetaMask

1. Switch MetaMask to the network where TFRST is deployed.
2. Click **Import tokens**.
3. Paste the token address: `0xYOUR_TOKEN_ADDRESS`.
4. The symbol (`TFRST`) and decimals (`18`) should fill in automatically.

### Delegate your votes

On the token's Etherscan page, open **Contract**, then **Write Contract**, connect your wallet, and call `delegate` with your own address. Verify with `getVotes(yourAddress)` under **Read Contract**.

### Vote on a proposal

On the Governor's Etherscan page, open **Write Contract** and call `castVote(proposalId, support)` while the proposal is Active (`0` = Against, `1` = For, `2` = Abstain).

## Design Decisions

TFRST is modeled on the simplicity of Uniswap's UNI token. I originally wanted to design a token modeled after USDT, however with further research I realized USDT limits a true decentralization model. One example of this in USDT is allowing freezing or blacklist of wallet addresses. TFRST deliberately leaves out features that give any single party power over holders:

- **No pause function:** nobody can freeze transfers.
- **No blacklist or fees:** transfers behave exactly like a standard ERC-20.
- **No rescue function:** tokens sent to a contract by mistake stay there, which keeps the admin surface minimal.
- **No upgradeable proxy:** the code cannot be changed after deployment.
- **Minting is bounded:** the only admin power is minting, and only the timelock can use it, after a governance vote, a 7-day delay, a 365-day interval, and a 2% size limit.

### Future Implementations

- **Burn and Mint Mechanism:** Later I want to try to implement a burn and mint mechanism rewarding holders with another token for burning TFRST token. 
- **Hardhat Tests:** Very soon I will implement hardhat test for every component in solidity contracts.

## What I Learned

- How the ERC-20 standard defines a common interface for fungible tokens
- How `ERC20Votes` checkpoints balances so voting power can't be faked
- How delegation, snapshots, quorum, and thresholds work in a Governor
- Why a timelock separates deciding from doing, and why that protects holders
- How role management works in `AccessControl` and why renouncing admin matters
- How inheritance and `override` resolve conflicts between multiple parent contracts
- How custom errors and events reduce gas and help apps track activity
- How to write, compile, test, and deploy with Hardhat 3, TypeScript, and viem
- Why distribution matters: token-weighted voting favors whoever holds the supply
- Decrentralized crypto versus centralized crypto, and its trade offs
- Solidity basics and best practices

## License

This project is licensed under the MIT License. See the `LICENSE` file for details.

---

Built to learn. Feedback and suggestions are always welcomed!
