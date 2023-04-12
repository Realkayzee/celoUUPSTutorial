# Building an Upgradeable Contract-

---

## Introduction:

The concept of Smart Contract being immutable is a great idea that creates enough trust to the users interacting with a deployed implementation. This concept has its downside with creator/developer not having the capability to fix any vulnerability when it arises (since developer can not change anything on a deployed contract), giving attacker an unfair advantage over the developer. Upgradeable Contract provides solution to this Smart Contract problem while giving developers a good experience to maintain the immutable nature of smart contracts and logic of their implementation.

## Table of Contents:

- [Building an Upgradeable Contract](#building-an-upgradeable-contract)
    - [Introduction](#introduction)
    - [Table of Content](#table-of-contents)
    - [Objective](#Objective)
    - [Prerequisite](#Prerequisites)
    - [Requirement](#Requirements)
    - [Tutorial](#Tutorial)
        - [Step 1 - Spin off hardhat environment](#STEP-1---Spin-off-Hardhat-Environment)
        - [Step 2 - Create a simple upgradeable smart contract](#STEP-2---Create-a-simple-upgradeable-bank-contract)
        - [Step 3 - Simple bank contract with withdraw defect](#STEP-3---Simple-bank-contract-with-Withdraw-defect)
        - [Step 4 - Interacting with our deployed contract](#STEP-4---Interacting-with-our-deployed-Contract)
        - [Step 5 - Upgrade Celo Bank contract to solve bug](#STEP-5---Upgrade-Celo-Bank-contract-to-solve-bug)
    - [Conclusion](#conclusion)
    
## Objective:

By the end of this tutorial, you will be able to write an upgradeable smart contract using the **Universal Upgradeable Proxy Standard (UUPS)**

## Pre-requisites:

* Knowledge of using [Hardhat](https://hardhat.org/). 
* Intermediate/advance knowledge in Solidity.
* Basic knowledge of using the command line.
* Understanding  of delegateCall.
* Understanding of [vscode](https://code.visualstudio.com/).

## Requirements:

* Have [Node.js](https://nodejs.org/en/download) installed from version V10. or higher.
* Have [npm](https://docs.npmjs.com/cli/v8/commands/npm-install) or [yarn](https://yarnpkg.com/cli/install) installed.
* [vscode](https://code.visualstudio.com/)

## Tutorial:

### **STEP 1** - Spin off Hardhat Environment:

The first thing we will be doing is to create a folder for our implementation and spin off hardhat environment, go to your terminal and follow this processes below to create an environment for our implementation

```
mkdir uupsPractice
```

This command create a new folder for us, we then need to initialize and open the folder either through the terminal using the command
```
 npm init -y
 cd uupsPractice
 code .
```

or from our vscode.
N.B: I'm using a linux base operating system but this should also work for windows (They may be a slight differences)
![](https://i.imgur.com/c5L69ZQ.png)

After opening the folder in our vscode, we should have something similar to what we have below.
![](https://i.imgur.com/UVWJOm6.png)

Now let's complete our initialization process by spinning off our hardhat environment, I will be using npm.
```
 npm install hardhat
 npx hardhat
```
after the command `npx hardhat` we should see something like what we have below.
![](https://i.imgur.com/o3XT3Pi.png)

For this tutorial we will be using "create a typescript project", confirm this and other prompt options.

### **STEP 2** - Setup Upgradeable Contract:

We will be using a simple bank contract for this tutorial as it will give a better understanding on implementing the upgradeable standard.

[Click here](https://eips.ethereum.org/EIPS/eip-1822) to know more about UUPS.

Within our contract folder, we will create two files named:
1. proxy.sol (The proxy contract that delegate call to our implementation contract)
2. proxiable.sol (Responsible for upgrading contract)

**What is Proxy contract?**

Proxy contract is a contract (Contract A) that delegates call to another contract (Contract B) while maintaining the same storage layout(state variables).
![](https://i.imgur.com/pcKYn2W.png)

in our proxy.sol contract, copy and paste the code below:
```
// SPDX-License-Identifier: MIT
pragma solidity 0.8.0;

contract Proxy {
    // Code position in storage is keccak256("PROXIABLE") = "0xc5f16f0fcc639fa48a6947836d9850f504798523bf8c9a3a87d5876cf622bcf7"
    address private contractLogic; // Address variable to store the contract logic

    constructor(address _contractLogic) {
        // Save the code address during contract deployment
        contractLogic = _contractLogic;
    }

    fallback() external payable {
        assembly {
            // Load the contract logic address from storage
            let logic := sload(0xc5f16f0fcc639fa48a6947836d9850f504798523bf8c9a3a87d5876cf622bcf7)
            // Copy the calldata (input data) to memory
            calldatacopy(0x0, 0x0, calldatasize())
            // Delegatecall to the contract logic address with the copied calldata
            // and limited gas (subtracting 10000 from the current gas)
            let success := delegatecall(sub(gas(), 10000), logic, 0x0, calldatasize(), 0, 0)
            // Get the size of the returndata (output data)
            let retSz := returndatasize()
            // Copy the returndata to memory
            returndatacopy(0, 0, retSz)
            // Check the success of the delegatecall
            switch success
            case 0 {
                // If delegatecall fails, revert with the returndata
                revert(0, retSz)
            }
            default {
                // Otherwise, return the returndata
                return(0, retSz)
            }
        }
    }

    receive() external payable {}
}

```
![](https://i.imgur.com/Ho4OTfu.png)

Before explaining this code it's important to have some understanding on delegate call, [this article gives a better explanation on delegate call](https://medium.com/coinmonks/delegatecall-calling-another-contract-function-in-solidity-b579f804178c).

The constrcutor has two parameters, *constructData* and *contractLogic*. The *constructData* takes in  the initialization calldata/payload (representing a constructor in our implementation), the *contractLogic* takes in implemetation contract address (address to delegatecall to).

**A fallback function** is triggered when a non-existence function is called in a contract. Whenever, a user interacts with proxy contract by calling a function that doesn't exist in the contract the fallback function will be triggered, and the logic within the fallback is being computed, giving us the ability to delegatecall to our implementation contract.

in our *proxiable.sol*, copy and paste the code below:
```
// SPDX-License-Identifier: MIT

pragma solidity 0.8.18;

contract Proxiable {
    // Code position in storage is keccak256("PROXIABLE") = "0xc5f16f0fcc639fa48a6947836d9850f504798523bf8c9a3a87d5876cf622bcf7"

    function updateCodeAddress(address newAddress) internal {
        // Check if the given newAddress is compatible with the current contract
        require(
            bytes32(0xc5f16f0fcc639fa48a6947836d9850f504798523bf8c9a3a87d5876cf622bcf7) == Proxiable(newAddress).proxiableUUID(),
            "Not compatible"
        );
        // Store the new code address in storage
        assembly {
            sstore(0xc5f16f0fcc639fa48a6947836d9850f504798523bf8c9a3a87d5876cf622bcf7, newAddress)
        }
    }

    function proxiableUUID() public pure returns (bytes32) {
        // Return the UUID (unique identifier) for this contract
        return 0xc5f16f0fcc639fa48a6947836d9850f504798523bf8c9a3a87d5876cf622bcf7;
    }
}

```
*proxiable.sol contract* will be inherited by our logic contract and it is responsible for upgradeability.

### **STEP 3** - Simple bank contract with Withdraw defect:

We will create our simple bank contract file using the name *celoBank.sol*, then copy & paste what we have below.
```
// SPDX-License-Identifier: MIT
pragma solidity 0.8.18;

import {Proxiable} from "../contracts/proxiable.sol"; // Importing Proxiable contract

contract celoBank is Proxiable {
    // State variables
    mapping(address => uint256) usersDeposit; // Mapping to store users' deposit amount
    address owner; // Address of the contract owner
    bool initialized; // Boolean flag to track contract initialization status

    // Function to initialize the contract
    function init(address _addr) external {
        require(!initialized, "Already initialized"); // Check if contract is not already initialized
        owner = _addr; // Set the contract owner
        initialized = true; // Mark the contract as initialized
    }

    // Function to get the contract owner
    function admin() external view returns(address) {
        return owner; // Return the address of the contract owner
    }

    // Function to deposit funds into the contract
    function deposit() external payable {
        require(msg.value != 0, "Amount can't be zero"); // Check if deposit amount is not zero
        usersDeposit[msg.sender] += msg.value; // Add deposited amount to user's deposit balance
    }

    // Function to get user's deposit balance
    function userBalance(address _addr) external view returns(uint256){
        return usersDeposit[_addr]; // Return the deposit balance of the specified address
    }

    // Function to upgrade the contract to a new address
    function upgradeContract(address _newAddress) external {
        require(msg.sender == owner, "You are not an admin"); // Check if the caller is the contract owner
        updateCodeAddress(_newAddress); // Call the Proxiable contract's updateCodeAddress function to upgrade the contract
    }
}

```
Next is to setup our hardhat config. We need to install dotenv by pasting the code below in our terminal:

> npm install dotenv

We can then, create a file named *.env* and paste our private key in our *.env*. It should look like the below code:

![](https://i.imgur.com/oyfaIXR.png)

Now, to the main hardhat setup for our celo alfajores testnet, copy and paste the code below in hardhat.config.ts:

```
import "@nomicfoundation/hardhat-toolbox"; // Importing Hardhat Toolbox for additional functionality
require("dotenv").config({path: ".env"}); // Loading environment variables from .env file

const PRIVATE_KEY = process.env.PRIVATE_KEY // Storing the private key from environment variable

module.exports = {
  solidity: "0.8.18", // Specifying the version of Solidity to be used
  networks: {
    alfajores: {
      url: "https://alfajores-forno.celo-testnet.org", // URL of the Alfajores Celo testnet node
      accounts: [PRIVATE_KEY] // Specifying the private key for the accounts to be used for deployment
    }
  }
}


```
Before moving to writing a script for our contract, we need to compile the code with this command in our terminal:
> npx hardhat compile

let's go on to deploy and interact. In our *deploy.ts* copy and paste this code to deploy our contract on celo Alfajores testnet:

```
import { ethers } from "hardhat"; // Importing ethers library for interacting with Ethereum contracts

async function main() {
  const ABI = [
    "function init(address _addr)" // ABI of the 'init' function in the contract
  ];

  const iProxy = new ethers.utils.Interface(ABI); // Creating an instance of the ethers Interface with the ABI

  const constructData = iProxy.encodeFunctionData("init", ["0x5DE9d9C1dC9b407a9873E2F428c54b74c325b82b"]); // Encoding the 'init' function call data with the arguments

  console.log(constructData, "construct data"); // Logging the encoded function call data

  // Deploying the implementation/logic contract
  const CeloBank = await ethers.getContractFactory("celoBank"); // Getting the contract factory for 'celoBank'
  const celoBank = await CeloBank.deploy(); // Deploying 'celoBank' contract

  await celoBank.deployed(); // Waiting for the contract deployment to be confirmed
  console.log(`Celo bank deployed to ${celoBank.address}`); // Logging the deployed contract's address

  // Deploying the proxy contract
  const Proxy = await ethers.getContractFactory("Proxy"); // Getting the contract factory for 'Proxy'
  const proxy = await Proxy.deploy(constructData, celoBank.address); // Deploying 'Proxy' contract with the encoded function call data and 'celoBank' contract address as arguments

  await proxy.deployed(); // Waiting for the contract deployment to be confirmed
  console.log(`Proxy contract deployed to ${celoBank.address}`); // Logging the deployed contract's address
}

// Using async/await pattern to handle asynchronous tasks and errors properly
main().catch((error) => {
  console.error(error); // Logging any errors that occurred during contract deployment
  process.exitCode = 1; // Setting the exit code to 1 to indicate an error occurred
});

```
Since we have our script fully written, we need to paste this command below in our terminal:
> npx hardhat run scripts/deploy.ts --network alfajores

**Note:** You must have test celo in your metamask wallet, before this operation can be succesful. You can get a test celo [here](https://faucet.celo.org/).

You should have something like what I have below:

![](https://i.imgur.com/Pt9Wp70.png)

### **STEP 4** - Interacting with our deployed Contract:

We need to checkup our deployed contract on celo testnet explorer [here](https://alfajores.celoscan.io/) to confirm our deployment.
![](https://i.imgur.com/RACLxV5.png)

After deployment is confirmed, it is necessary for us to verify, so that we can interact on the explorer and make our deployment readable to others.

To verify, click on the contract tab and open the link to *verify and publish* as shown below:

![](https://i.imgur.com/N7yLCvP.png)

Follow the procedures to verify (It's straight forward by following the guildlines). Verify both the implementation/logic contract (celoBank contract) and the proxy contract deployment.

When you are done with the proxy contract verification, you should see something like this below:
![](https://i.imgur.com/MBzUcM6.png)

Select more options & make the explorer see it as a proxy contract. At the end it should look like what we have below:

![](https://i.imgur.com/6EMTQan.png)

One thing you will notice is that we now have all the functions available on our implementation contract here, even though those functions aren't written in our *proxy.sol*. changing the implementation contract address to a new contract address will change the available functions in proxy with a persistent storage.

Let's interact with our deployed proxy by depositing some celo and view the user balance after.
- we connect our explorer to our wallet
- we deposit some celo to the celoBank contract
![](https://i.imgur.com/M8XI0oj.png)
- Lastly, we view user's balance
![](https://i.imgur.com/Omxsrer.png)

### **STEP 5** - Upgrade Celo Bank contract to solve bug:

While interacting we noticed users can deposit but cannot withdraw their funds. Traditionally, the money will be locked in the contract forever, but since this is an upgradeable contract, we can upgrade our contract to implement withdrawal.

This step, will guide us through on how to upgrade our contract.

NOTE: The upgrade can only be done by the admin. Without the admin, the upgrade is not possible.

let's go on to create another implementation with withdrawal option. name it celoBankUpgrade.sol and paste the code below:
```
// SPDX-License-Identifier: MIT

pragma solidity 0.8.18;

import { Proxiable } from "../contracts/proxiable.sol"; // Importing the 'Proxiable' contract from a separate file

contract celoBankUpgrade is Proxiable {
    mapping(address => uint256) usersDeposit; // Mapping to store users' deposits

    address owner; // Address of the contract owner
    bool initialized; // Flag to track contract initialization status

    // Function to initialize the contract with the owner's address
    function init(address _addr) external {
        require(!initialized, "Already initialized"); // Allowing initialization only once
        owner = _addr; // Setting the owner's address
        initialized = true; // Marking the contract as initialized
    }

    // Function to get the address of the contract owner
    function admin() external view returns (address) {
        return owner;
    }

    // Function to deposit Ether into the contract
    function deposit() external payable {
        require(msg.value != 0, "Amount can't be zero"); // Checking that the deposited amount is not zero
        usersDeposit[msg.sender] += msg.value; // Adding the deposited amount to the user's deposit balance
    }

    // Function to withdraw Ether from the contract
    function withdraw() external {
        uint256 myBalance = usersDeposit[msg.sender]; // Storing the user's deposit balance
        usersDeposit[msg.sender] = 0; // Setting the user's deposit balance to zero
        (bool success, ) = payable(msg.sender).call{ value: myBalance }(""); // Transferring the withdrawn amount to the user's address
        require(success, "Invalid"); // Checking if the transfer was successful
    }

    // Function to get the deposit balance of a user
    function userBalance(address _addr) external view returns (uint256) {
        return usersDeposit[_addr]; // Returning the deposit balance of the specified user
    }

    // Function to upgrade the contract to a new address by the contract owner
    function upgradeContract(address _newAddress) external {
        require(msg.sender == owner, "You are not an admin"); // Checking if the caller is the contract owner
        updateCodeAddress(_newAddress); // Updating the contract's code address to the new address
    }
}

```

Now that we have created a new contract with withdraw function, we need to deploy our *celoBankUpgrade.sol*. Create a deploy file with a name *deployUpgrade.ts* and paste the following code below:

```
import { ethers } from "hardhat"; // Importing ethers library for interacting with Ethereum contracts

async function main() {
    const CeloBankUpgrade = await ethers.getContractFactory("celoBankUpgrade"); // Getting the contract factory for "celoBankUpgrade" contract
    const celoBankUpgrade = await CeloBankUpgrade.deploy(); // Deploying the "celoBankUpgrade" contract

    await celoBankUpgrade.deployed(); // Waiting for the contract deployment to complete

    console.log(`celo bank upgrade deployed to ${celoBankUpgrade.address}`); // Logging the deployed contract's address
}

// We recommend this pattern to be able to use async/await everywhere
// and properly handle errors.
main().catch((error) => {
    console.error(error); // Logging any errors that occur during contract deployment
    process.exitCode = 1; // Setting the exit code to 1 to indicate an error occurred
});

```
If everything is right, should look like what we have below
![](https://i.imgur.com/GrnT2Lp.png)

paste the deployment command below in your terminal
```
npx hardhat run scripts/deployUpgrade.ts --network alfajores
```
We can go on to verify as we did for celoBank contract and proxy contract.

Now, let's upgrade our proxy contract by passing celoBankUpgrade contract address as a parameter to upgradeContract function in proxy contract.

![](https://i.imgur.com/kLANPxI.png)

After upgrading, you can see that we now have our withdraw function available which will give us the opportunity to withdraw our deposited funds (Remeber, we already deposited 1 celo before the upgrade).

Now we can withdraw it through the power of upgradeable contract.

![](https://i.imgur.com/syOwpEc.png)

### Conclusion:

Therefore, upgradeable contract gives us the flexibility to fix bugs in an immutable contract as we see in this tutorial. It is necessary to take such practices beyond creating simple bank contract and use it for real life solution.

UUPS is not the only upgradeable contract standard. We have:
- Diamond Standard (little bit advanced than UUPS)
- Transparent Upgradeable Proxy

Thank you for following up till the end of this tutorial, I hope you have learned something new. Good learning!!

Here's a link to the project [UUPS for celo blockchain](https://)
