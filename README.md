# Optimistic Rollup Example: ERC20

![](https://i.imgur.com/jJKCZ84.png)

## Introduction

This example demonstrates how to deploy and interact with ERC20 contract using Optimistic Rollup, including the following operations:
- **Deposit** ERC20 from L1 to L2
- **Transfer** ERC20 in L2
- **Withdraw** ERC20 from L2 to L1

Here are two other great examples:
- [optimism-tutorial](https://github.com/ethereum-optimism/optimism-tutorial)
- [l1-l2-deposit-withdrawal](https://github.com/ethereum-optimism/l1-l2-deposit-withdrawal)

## Prerequisite Software

- [Git](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git)
- [Node.js](https://nodejs.org/en/download/)
- [Yarn](https://classic.yarnpkg.com/en/docs/install#mac-stable)
- [Docker](https://docs.docker.com/engine/install/)

## Build and Run Optimistic Ethereum

Everything we need to build and run Optimistic Rollup is in [Optimistic Ethereum](https://github.com/ethereum-optimism/optimism/tree/develop/ops):

```bash
$ git clone https://github.com/ethereum-optimism/optimism.git
$ cd optimism
$ yarn
$ yarn build
```

Run it when the build is done:

```bash
$ cd ops
$ docker-compose build
$ docker-compose up
```

A few services will be up, including：
- L1 Ethereum Node (EVM)
- L2 Optimistic Ethereum Node (OVM)
- Batch Submitter
- Data Transport Layer
- **Deployer**
- Relayer
- Verifier

> The environment variable `FRAUD_PROOF_WINDOW_SECONDS` in the deployer service defines how much long the user has to wait when withdrawing. Its default value is `0` second.

> Clean the docker volume when you need to restart the service. Ex: `docker-compose down -v`

### Test Optimistic Ethereum

Before we continue, we need to make sure that the Optimistic Ethereum operates normally, Especially the relayer. Use integration test to check its functionality:

```bash
$ cd optimism/integration-tests
$ yarn build:integration
$ yarn test:integration
```

Make sure all the tests related to `L1 <--> L2 Communication` passed before you continue.

> It might take a while (~ 120s) for Optimistic Ethereum to be fully operational. If you fail all the test, try again later or rebuild Optimistic Ethereum from the source again.

## Deploy ERC20 and Gateway Contracts

Next, let's deploy the contract:

```bash
$ git clone https://github.com/ccniuj/optimistic-rollup-example-erc20.git
$ cd optimistic-rollup-example-erc20
$ yarn install
$ yarn compile
```

There are 3 contracts to be deployed:
- `ERC20`, L1
- `L2DepositedEERC20`, L2
- `OVM_L1ERC20Gateway`, L1

> `OVM_L1ERC20Gateway` is depoyed on L1 only. As its name suggests, it works as a "Gateway" which provides `deposit` and `withdraw` functions. User needs to use a gateway to move funds.

> In this example, we use `OVM_L1ERC20Gateway` developed by official team. However, developers can custimize the gateway based on what they need.

Next, deploy these contracts by the script:

```bash
$ node ./deploy.js
Deploying L1 ERC20...
Deploying L1 ERC20...
L1 ERC20 Contract Address:  0xFD471836031dc5108809D173A067e8486B9047A3
Deploying L2 ERC20...
L2 ERC20 Contract Address:  0x09635F643e140090A9A8Dcd712eD6285858ceBef
Deploying L1 ERC20 Gateway...
L1 ERC20 Gateway Contract Address:  0xcbEAF3BDe82155F56486Fb5a1072cb8baAf547cc
Initializing L2 ERC20...
```

## ERC20 Deposit, Transfer and Withdraw

### Deposit ERC20 （L1 => L2）

Here are the current balances:

| L1/L2 | Account | Balance |
| - | - | - |
| L1 | Deployer | 10000 |
| L1 | User | 0 |
| L2 | Deployer | 0 |
| L2 | User | 0 |

After the L1 ERC20 is deployed, Deployer is the only account that is funded. Next, let's move the fund from L1 to L2.

First, enter ETH (L1) console:

```bash
$ npx hardhat console --network eth
Welcome to Node.js v16.1.0.
Type ".help" for more information.
> 
```

Initialize Deployer and User:

```javascript
// In Hardhat ETH Console

> let accounts = await ethers.getSigners()
> let deployer = accounts[0]
> let user = accounts[1]
```

Instantiate `ERC20` and `OVM_L1ERC20Gateway` contracts. Their contract addresses are in the outputs of the deploy script:

```javascript
// In Hardhat ETH Console

> let ERC20_abi = await artifacts.readArtifact("ERC20").then(c => c.abi)
> let ERC20 = new ethers.Contract("0xFD471836031dc5108809D173A067e8486B9047A3", ERC20_abi)
> let Gateway_abi = await artifacts.readArtifact("OVM_L1ERC20Gateway").then(c => c.abi)
> let Gateway = new ethers.Contract("0xcbEAF3BDe82155F56486Fb5a1072cb8baAf547cc", Gateway_abi)
```

Confirm the balance of Deployer:

```javascript
> await ERC20.connect(deployer).balanceOf(deployer.address)
BigNumber { _hex: '0x2710', _isBigNumber: true } // 10000
```

Next, approve `OVM_L1ERC20Gateway` to spend the ERC20:

```javascript
// In Hardhat ETH Console

> await ERC20.connect(deployer).approve("0xcbEAF3BDe82155F56486Fb5a1072cb8baAf547cc", 10000)
{
  hash: "...",
  ...
}
> await ERC20.connect(user).approve("0xcbEAF3BDe82155F56486Fb5a1072cb8baAf547cc", 10000)
{
  hash: "...",
  ...
}
```

> Both Deployer and User have to approve `OVM_L1ERC20Gateway` in ERC20 contract, otherwise the relayer service might fail when withdrawing in the following steps.

Call `deposit` at `OVM_L1ERC20Gateway` contract:

```javascript
// In Hardhat ETH Console

> await Gateway.connect(deployer).deposit(1000)
{
  hash: "...",
  ...
}

```

Confirm if the deposit is successful from Optimistic Ethereum (L2) console:

```bash
$ npx hardhat console --network optimism
Welcome to Node.js v16.1.0.
Type ".help" for more information.
> 
```

Initialize Deployer and User:

```javascript
// In Hardhat Optimism Console

> let accounts = await ethers.getSigners()
> let deployer = accounts[0]
> let user = accounts[1]
```

Instantiate `L2DepositedERC20` contract. Its contract address is in the outputs of the deploy script:

```javascript
// In Hardhat Optimism Console

> let L2ERC20_abi = await artifacts.readArtifact("L2DepositedERC20").then(c => c.abi)
> let L2DepositedERC20 = new ethers.Contract("0x09635F643e140090A9A8Dcd712eD6285858ceBef", L2ERC20_abi)
```

Confirm if the deposit is successful:

```javascript
// In Hardhat Optimism Console

> await L2DepositedERC20.connect(deployer).balanceOf(deployer.address)
BigNumber { _hex: '0x03e8', _isBigNumber: true } // 1000
```

### Transfer ERC20 （L2 <=> L2）

Here are the current balances:

| L1/L2 | Account | Balance |
| - | - | - |
| L1 | Deployer | 9000 |
| L1 | User | 0 |
| L2 | Deployer | 1000 |
| L2 | User | 0 |

Next, let's transfer some funds from Deployer to User:

```javascript
// In Hardhat Optimism Console

> await L2DepositedERC20.connect(user).balanceOf(user.address)
BigNumber { _hex: '0x00', _isBigNumber: true } // 0
> await L2DepositedERC20.connect(deployer).transfer(user.address, 1000)
{
  hash: "..."
  ...
}
> await L2DepositedERC20.connect(user).balanceOf(user.address)
BigNumber { _hex: '0x03e8', _isBigNumber: true } // 1000
```

### Withdraw ERC20 （L2 => L1）

Here are the current balances:

| L1/L2 | Account | Balance |
| - | - | - |
| L1 | Deployer | 9000 |
| L1 | User | 0 |
| L2 | Deployer | 0 |
| L2 | User | 1000 |

Next, let's withdraw the funds via account User. Call `withdraw` at `L2DepositedERC20` contract:

```javascript
// In Hardhat Optimism Console

> await L2DepositedERC20.connect(user).withdraw(1000)
{
  hash: "..."
  ...
}
> await L2DepositedERC20.connect(user).balanceOf(user.address)
BigNumber { _hex: '0x00', _isBigNumber: true }
```

Finally, let's confirm if the withdrawal is successful on L1:

```javascript
// In Hardhat ETH Console

> await ERC20.connect(user).balanceOf(user.address)
BigNumber { _hex: '0x03e8', _isBigNumber: true } // 1000
> await ERC20.connect(deployer).balanceOf(deployer.address)
BigNumber { _hex: '0x2328', _isBigNumber: true } // 9000
```

> Since the `FRAUD_PROOF_WINDOW_SECONDS` is set to be `0` second, you don't need to wait too long before the fund is withdrawn back to L1.

After all the operations, here is the final balances:

| L1/L2 | Account | Balance |
| - | - | - |
| L1 | Deployer | 9000 |
| L1 | User | 1000 |
| L2 | Deployer | 0 |
| L2 | User | 0 |

## Reference

- [OVM Deep Dive](https://medium.com/ethereum-optimism/ovm-deep-dive-a300d1085f52)
- [(Almost) Everything you need to know about Optimistic Rollup](https://research.paradigm.xyz/rollups)
- [How does Optimism's Rollup really work?](https://research.paradigm.xyz/optimism)
- [Optimistic Rollup Official Documentation](https://community.optimism.io/docs/)
- [Ethers Documentation (v5)](https://docs.ethers.io/v5/)
- [Optimism (Github)](https://github.com/ethereum-optimism/optimism)
- [optimism-tutorial (Github)](https://github.com/ethereum-optimism/optimism-tutorial)
- [l1-l2-deposit-withdrawal (Github)](https://github.com/ethereum-optimism/l1-l2-deposit-withdrawal)
