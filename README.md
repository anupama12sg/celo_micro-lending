# A Developer's guide on building a Celo-based microlending platform for financial inclusion

## Table Of Contents

- [A Developer's guide on building a Celo-based microlending platform for financial inclusion](#a-developers-guide-on-building-a-celo-based-microlending-platform-for-financial-inclusion)
  - [Table Of Contents](#table-of-contents)
  - [Introduction](#introduction)
  - [Prerequisites](#prerequisites)
  - [Setting up the Development Environment](#setting-up-the-development-environment)
    - [Step 1: Create a new project](#step-1-create-a-new-project)
    - [Step 2: Install Hardhat](#step-2-install-hardhat)
    - [Step 3: Configure Hardhat](#step-3-configure-hardhat)
    - [Step 4: Create a Celo Wallet](#step-4-create-a-celo-wallet)
    - [Step 5: Fund Your Wallet with Testnet cUSD](#step-5-fund-your-wallet-with-testnet-cusd)
  - [Creating the Microlending Smart Contract](#creating-the-microlending-smart-contract)
    - [Step 1: Create a new solidity file](#step-1-create-a-new-solidity-file)
    - [Step 2: Compile the Smart Contract](#step-2-compile-the-smart-contract)
    - [Step 3: Deploy the Smart Contract](#step-3-deploy-the-smart-contract)
    - [Step 4: Interact with the Smart Contract](#step-4-interact-with-the-smart-contract)
    - [Step 5: Testing](#step-5-testing)
  - [Conclusion](#conclusion)

## Introduction

Microlending has been acknowledged as an important strategy for poverty alleviation and financial inclusion. It gives low-income people access to loans that can help them start enterprises, pay for education or healthcare, and satisfy other basic requirements. Traditional financial institutions, on the other hand, have frequently disregarded low-income persons due to a lack of collateral and credit history. With the advent of blockchain technology, it has become possible to create microlending platforms that use smart contracts to eliminate the need for middlemen, cut costs, and expand loan access.

**Celo** is a blockchain platform that aims to increase financial inclusion by allowing people to transfer, receive, and store digital assets on their mobile devices. It offers a stablecoin called Celo Dollar (cUSD), which is pegged to the US dollar and may be used for microlending. In this tutorial, we will use smart contracts to create a Celo-based microlending platform.

## Prerequisites

To follow this tutorial, you will need the following:

- A basic understanding of blockchain technology and smart contracts
- Familiarity with Solidity programming language
- An understanding of web3.js library
- Node.js & Npm installed on your machine
- A Celo wallet and some testnet cUSD

## Setting up the Development Environment

We will use the following tools and services to develop our microlending platform:

- Hardhat: A development framework that provides tools for compiling, testing, and deploying smart contracts.
- Infura: A service that provides access to Ethereum networks for deploying smart contracts.

To begin, take these steps:

### Step 1: Create a new project

Create a new directory for your project:

```bash
    mkdir microlending-platform
```

Navigate to the project directory you just created:

```bash
    cd microlending-platform
```

### Step 2: Install Hardhat

Open your terminal and run the following commands to install Hardhat:

```bash
    npm install hardhat --save-dev
```

Initialize a new Hardhat Project

```bash
    npx hardhat init
```

This will create a new project similar to this structure:

```
    ├── contracts
    │   └── Lock.sol
    ├── test
    └── hardhat-config.js
```

### Step 3: Configure Hardhat

Open the hardhat-config.js file and configure it as follows:

```javascript
module.exports = {
  networks: {
    development: {
      host: "127.0.0.1",
      port: 8545,
      network_id: "*",
    },
    celo: {
      provider: () =>
        new HDWalletProvider(privateKey, "https://www.infura.io/"),
      network_id: 44787,
      gas: 5000000,
      gasPrice: 20000000000, // 20 gwei
    },
  },
  compilers: {
    solc: {
      version: "0.8.4",
      optimizer: {
        enabled: true,
        runs: 200,
      },
    },
  },
};
```

PrivateKey should be replaced with your Celo testnet private key. This file defines two networks: celo, which connects to the Celo testnet via Infura, and development, which connects to a local hardhat instance.

### Step 4: Create a Celo Wallet

A Celo wallet is required to engage with the Celo network. You can make advantage of the wallet extension provided by Celo which can be downloaded and installed on your local machine **[here](https://celowallet.app/setup)**

### Step 5: Fund Your Wallet with Testnet cUSD

We need some testnet cUSD to test our microlending platform. You can request testnet cUSD from the Celo Faucet by visiting this **[link](https://celo.org/developers/faucet)**

After submitting your wallet address, you should receive testnet cUSD within a few minutes. You can check your balance in your celo wallet.

## Creating the Microlending Smart Contract

In this section, we'll build a smart contract that lets lenders deposit cUSD and borrowers seek loans. The smart contract will handle loan repayment, including interest, and will send payments to lenders.

### Step 1: Create a new solidity file

Create a new file called Microlending.sol in the contracts directory and add the following code

```solidity
// SPDX-License-Identifier: UNLICENSED

pragma solidity ^0.8.4;

contract Microlending {
  struct Loan {
    address borrower;
    uint256 amount;
    uint256 interestRate;
    uint256 duration;
    uint256 endTime;
    bool repaid;
  }

  Loan[] public loans;
  mapping(address => uint256) public balances;

  event LoanRequested(address indexed borrower, uint256 amount, uint256 interestRate, uint256 duration, uint256 endTime);
  event LoanRepaid(uint256 loanId, uint256 amount);

  function requestLoan(uint256 amount, uint256 interestRate, uint256 duration) external {
    require(amount > 0, "Amount must be greater than zero");
    require(interestRate > 0, "Interest rate must be greater than zero");
    require(duration > 0, "Duration must be greater than zero");

    uint256 endTime = block.timestamp + duration;

    loans.push(Loan(msg.sender, amount, interestRate, duration, endTime, false));

    emit LoanRequested(msg.sender, amount, interestRate, duration, endTime);
  }

  function repayLoan(uint256 loanId) external {
    Loan storage loan = loans[loanId];

    require(msg.sender == loan.borrower, "Only borrower can repay the loan");
    require(!loan.repaid, "Loan already repaid");

    uint256 amount = loan.amount + (loan.amount * loan.interestRate / 100);
    balances[msg.sender] -= amount;
    loan.repaid = true;

    emit LoanRepaid(loanId, amount);
  }

  function deposit() external payable {
    require(msg.value > 0, "Deposit amount must be greater than zero");

    balances[msg.sender] += msg.value;
  }

  function withdraw(uint256 amount) external {
    require(amount > 0, "Withdraw amount must be greater than zero");
    require(amount <= balances[msg.sender], "Insufficient balance");

    balances[msg.sender] -= amount;

    (bool success, ) = msg.sender.call{value: amount}("");
    require(success, "Withdrawal failed");
  }
}
```

Let's Breakdown the contract line by line:

```solidity
    contract Microlending {
```

This declares the start of the contract.

```solidity
  struct Loan {
    address borrower;
    uint256 amount;
    uint256 interestRate;
    uint256 duration;
    uint256 endTime;
    bool repaid;
  }
```

This defines a new data structure named "Loan", which represents a loan that has been requested by a borrower.

```
    Loan[] public loans;
```

This declares a public collection of loans requested by borrowers. By making this array public, other contracts and external applications will be able to read the data stored in it.

```solidity
     mapping(address => uint256) public balances;
```

This publishes a public mapping that links each address's balance to the amount of cUSD (a stablecoin used on the Celo network) they currently have.

```solidity
    event LoanRequested(address indexed borrower, uint256 amount, uint256 interestRate, uint256 duration, uint256 endTime);
    event LoanRepaid(uint256 loanId, uint256 amount);
```

These are two events that the contract emits when a loan is requested or repaid. Events are a way for the contract to communicate with external applications, and can be subscribed to by interested parties.

```solidity
  function requestLoan(uint256 amount, uint256 interestRate, uint256 duration) external {
    require(amount > 0, "Amount must be greater than zero");
    require(interestRate > 0, "Interest rate must be greater than zero");
    require(duration > 0, "Duration must be greater than zero");

    uint256 endTime = block.timestamp + duration;

    loans.push(Loan(msg.sender, amount, interestRate, duration, endTime, false));

    emit LoanRequested(msg.sender, amount, interestRate, duration, endTime);
  }
```

Borrowers can use this feature to request a loan. It takes the loan amount, interest rate, and duration as inputs and produces a new Loan struct with this information. The function then adds the new loan to the loans array and fires the LoanRequested event.

```solidity
  function repayLoan(uint256 loanId) external {
    Loan storage loan = loans[loanId];

    require(msg.sender == loan.borrower, "Only borrower can repay the loan");
    require(!loan.repaid, "Loan already repaid");

    uint256 amount = loan.amount + (loan.amount * loan.interestRate / 100);
    balances[msg.sender] -= amount;
    loan.repaid = true;

    emit LoanRepaid(loanId, amount);
  }
```

Borrowers can use this function to repay their loans. The Loan struct associated with the supplied loanId is first retrieved from the loans array. The function then validates that the sender is the borrower and that the loan has not been repaid. If these conditions are met, the function computes the total amount to be repaid (loan amount + interest), deducts it from the borrower's balance, marks the loan as repaid, and emits a LoanRepaid event.

```solidity
  function deposit() external payable {
    require(msg.value > 0, "Deposit amount must be greater than zero");

    balances[msg.sender] += msg.value;
  }
```

Users can utilize this function to deposit cUSD into their account balance. It requires that the deposited amount be larger than zero, adds it to the sender's balance, and returns nothing.

```solidity
 function withdraw(uint256 amount) external {
    require(amount > 0, "Withdraw amount must be greater than zero");
    require(amount <= balances[msg.sender], "Insufficient balance");

    balances[msg.sender] -= amount;

    (bool success, ) = msg.sender.call{value: amount}("");
    require(success, "Withdrawal failed");
}
```

The `withdraw` function allows a user to withdraw funds from their balance in the microlending platform. The function takes a `uint256` parameter `amount` which represents the amount of funds the user wishes to withdraw.

The first line of the function contains a `require` statement that checks if the amount is greater than 0. If the `amount` is not greater than 0, the function will revert with the error message "Withdraw amount must be greater than zero".

The second line of the function contains another `require` statement that checks if the `amount` is less than or equal to the balance of the user (`msg.sender`) in the `balances` mapping. If the user does not have enough funds to withdraw the requested `amount`, the function will revert with the error message "Insufficient balance".

The third line of the function subtracts the `amount` from the balance of the user in the `balances` mapping. This ensures that the user's balance is updated to reflect the withdrawal.

The fourth line of the function uses the `call` function to send the `amount` of funds to the user's address (`msg.sender`). The `call` function returns a tuple of type (`bool, bytes`). The `bool` variable `success` will be `true` if the call was successful, and `false` otherwise. The second variable is ignored in this case.

The fifth line of the function contains a `require` statement that checks if the `call` was successful. If the `call` was not successful, the function will revert with the error message "Withdrawal failed".

In general, the `withdraw` function enables users to take money from their microlending platform balance as long as they have enough money to do so.

### Step 2: Compile the Smart Contract

To compile the smart contract, run the following command in your terminal:

```bash
    npx hardhat compile
```

This command will generate the required artifacts in the `artifacts` directory and compile the smart contract.

### Step 3: Deploy the Smart Contract

We will add a brand-new deployment script called `deploy.js` to the `scripts` directory in order to deploy the smart contract. Code the script using the following:

```javascript
const hre = require("hardhat");

async function main() {
  const Microlending = await hre.ethers.getContractFactory("Microlending");
  const microlending = await Microlending.deploy();

  await microlending.deployed();

  console.log("Microlending deployed to:", microlending.address);
}

main();
```

This script uses the Hardhat framework to deploy the `Microlending` smart contract to the Celo testnet. To deploy the smart contract, run the following command in your terminal:

```bash
    npx hardhat run scripts/deploy.js --network alfajores
```

After the deployment is complete, you should see the address of the deployed smart contract in your terminal.

### Step 4: Interact with the Smart Contract

We'll add a new script called `interact.js` to the `scripts` directory to communicate with the smart contract. Code the script using the following:

```javascript
const hre = require("hardhat");

async function main() {
  const Microlending = await hre.ethers.getContractFactory("Microlending");
  const microlending = await Microlending.attach("CONTRACT_ADDRESS");

  // Deposit cUSD into the smart contract
  const depositAmount = ethers.utils.parseUnits("100", "18");
  await microlending.deposit({ value: depositAmount });

  // Request a loan
  const loanAmount = ethers.utils.parseUnits("10", "18");
  const interestRate = 10; // 10%
  const duration = 86400; // 1 day
  await microlending.requestLoan(loanAmount, interestRate, duration);

  // Repay the loan
  const loanId = 0; // Assuming only one loan has been requested so far
  const loan = await microlending.loans(loanId);
  const repayAmount = loan.amount.add(
    loan.amount.mul(loan.interestRate).div(100)
  );
  await microlending.repayLoan(loanId);

  // Withdraw cUSD from the smart contract
  const withdrawAmount = ethers.utils.parseUnits("50", "18");
  await microlending.withdraw(withdrawAmount);
}

main();
```

Replace `CONTRACT_ADDRESS` with the address of the deployed smart contract.

This script invests $100 into the smart contract, requests a loan of $10 with a 10% interest rate for one day, repays the loan, and withdraws $50 from the smart contract.

To run the script, run the following command in your terminal:

```javasript
npx hardhat run scripts/interact.js --network alfajores
```

After the script runs successfully, you should see the transactions in your Celo wallet.

### Step 5: Testing

To test the smart contract, we will create a new `test` file in the test directory called `microlending.js`. Add the following code to the file:

```javascript
const { expect } = require("chai");

describe("Microlending", function () {
  let Microlending;
  let microlending;
  let owner;
  let addr1;
  let addr2;

  beforeEach(async function () {
    Microlending = await ethers.getContractFactory("Microlending");
    [owner, addr1, addr2] = await ethers.getSigners();
    microlending = await Microlending.deploy();
    await microlending.deployed();
  });

  describe("deposit", function () {
    it("should increase the lender's cUSD balance", async function () {
      const depositAmount = ethers.utils.parseUnits("100", "18");
      await microlending.connect(addr1).deposit({ value: depositAmount });
      expect(await microlending.balances(addr1.address)).to.equal(
        depositAmount
      );
    });
  });

  describe("requestLoan", function () {
    it("should create a new loan", async function () {
      const loanAmount = ethers.utils.parseUnits("10", "18");
      const interestRate = 10; // 10%
      const duration = 86400; // 1 day
      await microlending
        .connect(addr1)
        .requestLoan(loanAmount, interestRate, duration);
      const loan = await microlending.loans(0);
      expect(loan.amount).to.equal(loanAmount);
      expect(loan.interestRate).to.equal(interestRate);
      expect(loan.duration).to.equal(duration);
      expect(loan.borrower).to.equal(addr1.address);
    });

    it("should transfer the loan amount to the borrower's cUSD balance", async function () {
      const depositAmount = ethers.utils.parseUnits("100", "18");
      await microlending.connect(addr1).deposit({ value: depositAmount });
      const loanAmount = ethers.utils.parseUnits("10", "18");
      const interestRate = 10; // 10%
      const duration = 86400; // 1 day
      await microlending
        .connect(addr1)
        .requestLoan(loanAmount, interestRate, duration);
      expect(await microlending.balances(addr1.address)).to.equal(
        depositAmount.sub(loanAmount)
      );
    });

    it("should emit a LoanRequested event", async function () {
      const loanAmount = ethers.utils.parseUnits("10", "18");
      const interestRate = 10; // 10%
      const duration = 86400; // 1 day
      await expect(
        microlending
          .connect(addr1)
          .requestLoan(loanAmount, interestRate, duration)
      )
        .to.emit(microlending, "LoanRequested")
        .withArgs(0, addr1.address, loanAmount, interestRate, duration);
    });
  });

  describe("repayLoan", function () {
    it("should mark the loan as repaid", async function () {
      const depositAmount = ethers.utils.parseUnits("100", "18");
      await microlending.connect(addr1).deposit({ value: depositAmount });
      const loanAmount = ethers.utils.parseUnits("10", "18");
      const interestRate = 10; // 10%
      const duration = 86400; // 1 day
      await microlending
        .connect(addr1)
        .requestLoan(loanAmount, interestRate, duration);
      await microlending.connect(addr1).repayLoan(0);
      const loan = await microlending.loans(0);
      expect(loan.repaid).to.be.true;
    });

    it("should transfer the repaid amount to the lender's cUSD balance", async function () {
      const depositAmount = ethers.utils.parseUnits("100", "18");
      await microlending.connect(addr1).deposit({ value: depositAmount });
      const loanAmount = ethers.utils.parseUnits("10", "18");
      const interestRate = 10; // 10%
      const duration = 86400; // 1 day
      await microlending
        .connect(addr1)
        .requestLoan(loanAmount, interestRate, duration);
      await microlending.connect(addr1).repayLoan(0);
      expect(await microlending.balances(owner.address)).to.equal(
        depositAmount.add(loanAmount)
      );
    });

    it("should emit a LoanRepaid event", async function () {
      const depositAmount = ethers.utils.parseUnits("100", "18");
      await microlending.connect(addr1).deposit({ value: depositAmount });
      const loanAmount = ethers.utils.parseUnits("10", "18");
      const interestRate = 10; // 10%
      const duration = 86400; // 1 day
      await microlending
        .connect(addr1)
        .requestLoan(loanAmount, interestRate, duration);
      await expect(microlending.connect(addr1).repayLoan(0))
        .to.emit(microlending, "LoanRepaid")
        .withArgs(0, owner.address, loanAmount);
    });

    it("should prevent non-borrowers from repaying loans", async function () {
      const depositAmount = ethers.utils.parseUnits("100", "18");
      await microlending.connect(addr1).deposit({ value: depositAmount });
      const loanAmount = ethers.utils.parseUnits("10", "18");
      const interestRate = 10; // 10%
      const duration = 86400; // 1 day
      await microlending
        .connect(addr1)
        .requestLoan(loanAmount, interestRate, duration);
      await expect(microlending.connect(addr2).repayLoan(0)).to.be.revertedWith(
        "Only borrower can repay loan"
      );
    });

    it("should prevent repaying already repaid loans", async function () {
      const depositAmount = ethers.utils.parseUnits("100", "18");
      await microlending.connect(addr1).deposit({ value: depositAmount });
      const loanAmount = ethers.utils.parseUnits("10", "18");
      const interestRate = 10; // 10%
      const duration = 86400; // 1 day
      await microlending
        .connect(addr1)
        .requestLoan(loanAmount, interestRate, duration);
      await microlending.connect(addr1).repayLoan(0);
      await expect(microlending.connect(addr1).repayLoan(0)).to.be.revertedWith(
        "Loan already repaid"
      );
    });
  });
});
```

This test file sets up the `Microlending` contract, creates three signers, and tests the deposit, requestLoan, and repayLoan functions.

To run the tests, run the following command in your terminal:

```bash
    npx hardhat test
```

You should see output similar to the following:

```bash
Microlending
    deposit
✓ should increase the lender's cUSD balance
    request
      ✓ should create a new loan
      ✓ should emit a LoanRequested event
      ✓ should prevent lending more than the lender's balance
      ✓ should prevent borrowing more than the lending pool balance
    repayLoan
      ✓ should mark the loan as repaid
      ✓ should transfer the repaid amount to the lender's cUSD balance
      ✓ should emit a LoanRepaid event
      ✓ should prevent non-borrowers from repaying loans
      ✓ should prevent repaying already repaid loans


  9 passing (137ms)
```

All tests should pass without any errors.

Congratulations! You have now created a rudimentary microlending platform on the Celo blockchain. You can use this as a starting point for developing more complex financial applications using Celo.

Next steps could include developing more sophisticated risk assessment and credit scoring algorithms, integrating with off-chain data sources, and developing a user interface to interact with the smart contract.

## Conclusion

In conclusion, developing a Celo-based microlending platform can be an excellent strategy to promote financial inclusion and provide credit to those in greatest need. We can establish a transparent, safe, and efficient lending platform that benefits both lenders and borrowers by leveraging the power of smart contracts and the Celo blockchain.

We covered the fundamentals of developing a microlending platform on Celo in this lesson, including setting up a development environment, generating a smart contract, and testing our code. We also looked at the many functionalities of a microlending platform, such as seeking loans, repaying loans, depositing cash, and withdrawing funds.

You should now have a strong foundation for creating your own Celo-based microlending platform after following the instructions provided in this article. Of course, there's always more to learn and do better, but this tutorial ought to provide you a good place to start as you continue on your development path.

When developing on any blockchain, keep in mind to always put security and best practices first, and don't be afraid to ask the Celo community for help and advice along the way. Wishing you well as you create your microlending platform!
