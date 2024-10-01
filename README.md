# Estate Auction House (EACL) - Project Overview and Setup Guide

*Instructions for the Lead Programmer*

---

## Introduction

Estate Auction House (EACL) is a decentralized platform designed to revolutionize real estate auctions by leveraging blockchain technology. This document provides all the necessary code, instructions, and guidelines to set up the project, including smart contracts, front-end integration, and deployment steps.

---

## Table of Contents

1. Technologies Used
2. Project Structure
3. Setup Instructions
   - Prerequisites
   - Project Initialization
4. Smart Contracts
   - EACLToken.sol
   - AuctionHouse.sol
   - Deployment Script
5. Front-End Application
   - WalletConnector.js
   - ContractInteraction.js
   - App.js
6. Deployment and Testing
7. Additional Notes
8. License

---

## Technologies Used

- **Blockchain Platform**: Binance Smart Chain (BSC) Testnet
- **Smart Contract Language**: Solidity (version ^0.8.0)
- **Front-End Framework**: React.js
- **Blockchain Interaction**: Ethers.js
- **Development Environment**: Hardhat
- **Wallet Integration**: MetaMask
- **Node.js and NPM**: For package management
- **Version Control**: Git and GitHub

---

## Project Structure

EACL/
  - frontend/
    - src/
      - components/
        - WalletConnector.js
        - ContractInteraction.js
      - App.js
      - index.js
    - package.json
    - ...
  - smart-contracts/
    - contracts/
      - EACLToken.sol
      - AuctionHouse.sol
    - scripts/
      - deploy.js
    - hardhat.config.js
    - package.json
    - ...
  - README.md

---

## Setup Instructions

### Prerequisites

- **Node.js and NPM**: Install from the Node.js Official Website.
- **Git**: Install from the Git Official Website.
- **MetaMask**: Install the browser extension from the MetaMask Website.
- **Binance Smart Chain Testnet Account**: Set up an account and obtain testnet BNB.
- **Hardhat**: For smart contract development.

### Project Initialization

1. **Clone the Repository**

   git clone https://github.com/yourusername/EACL.git
   cd EACL

2. **Initialize Git (if not already initialized)**

   git init

3. **Create Project Directories**

   mkdir smart-contracts
   mkdir frontend

---

## Smart Contracts

Navigate to the `smart-contracts` directory to set up the smart contracts.

   cd smart-contracts

Initialize a Node.js project:

   npm init -y

Install Hardhat and dependencies:

   npm install --save-dev hardhat @nomiclabs/hardhat-ethers ethers dotenv
   npm install @openzeppelin/contracts

Initialize Hardhat:

   npx hardhat

Select "Create an empty hardhat.config.js" and follow the prompts.

### EACLToken.sol

Create `contracts/EACLToken.sol` with the following content:

// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";

contract EACLToken is ERC20 {
    constructor(uint256 initialSupply) ERC20("Estate Auction House Token", "EACL") {
        _mint(msg.sender, initialSupply);
    }
}

### AuctionHouse.sol

Create `contracts/AuctionHouse.sol` with the following content:

// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "./EACLToken.sol";

contract AuctionHouse {
    struct Property {
        uint256 id;
        address payable seller;
        uint256 minPrice;
        string details;
        bool isSold;
    }

    struct Bid {
        address bidder;
        uint256 amount;
    }

    EACLToken public token;
    uint256 public propertyCount;
    mapping(uint256 => Property) public properties;
    mapping(uint256 => Bid[]) public bids;

    event PropertyListed(uint256 id, address seller, uint256 minPrice, string details);
    event BidPlaced(uint256 propertyId, address bidder, uint256 amount);
    event PropertySold(uint256 propertyId, address buyer, uint256 amount);

    constructor(EACLToken _token) {
        token = _token;
    }

    function listProperty(uint256 minPrice, string memory details) public {
        require(token.balanceOf(msg.sender) >= 100 * (10 ** 18), "Need $100 worth of EACL tokens to list");

        propertyCount++;
        properties[propertyCount] = Property({
            id: propertyCount,
            seller: payable(msg.sender),
            minPrice: minPrice,
            details: details,
            isSold: false
        });

        emit PropertyListed(propertyCount, msg.sender, minPrice, details);
    }

    function placeBid(uint256 propertyId, uint256 bidAmount) public {
        Property storage property = properties[propertyId];
        require(!property.isSold, "Property already sold");
        require(bidAmount >= property.minPrice, "Bid amount too low");
        require(token.balanceOf(msg.sender) >= bidAmount / 10, "Need 10% of bid in EACL tokens");

        bids[propertyId].push(Bid({
            bidder: msg.sender,
            amount: bidAmount
        }));

        emit BidPlaced(propertyId, msg.sender, bidAmount);
    }

    function acceptBid(uint256 propertyId, uint256 bidIndex) public {
        Property storage property = properties[propertyId];
        require(msg.sender == property.seller, "Only seller can accept bids");
        require(!property.isSold, "Property already sold");

        Bid storage winningBid = bids[propertyId][bidIndex];

        // Transfer tokens as per the business logic
        uint256 sellerProceeds = winningBid.amount * 97 / 100; // 3% commission
        uint256 commission = winningBid.amount - sellerProceeds;

        // Implement token transfers here...

        property.isSold = true;

        emit PropertySold(propertyId, winningBid.bidder, winningBid.amount);
    }
}

### Deployment Script

Create `scripts/deploy.js` with the following content:

const hre = require("hardhat");

async function main() {
  // Deploy EACLToken
  const EACLToken = await hre.ethers.getContractFactory("EACLToken");
  const eaclToken = await EACLToken.deploy(ethers.utils.parseEther("1000000")); // 1,000,000 tokens
  await eaclToken.deployed();
  console.log("EACLToken deployed to:", eaclToken.address);

  // Deploy AuctionHouse
  const AuctionHouse = await hre.ethers.getContractFactory("AuctionHouse");
  const auctionHouse = await AuctionHouse.deploy(eaclToken.address);
  await auctionHouse.deployed();
  console.log("AuctionHouse deployed to:", auctionHouse.address);
}

main()
  .then(() => process.exit(0))
  .catch((error) => {
    console.error(error);
    process.exit(1);
  });

### Configuration

Update `hardhat.config.js`:

require("@nomiclabs/hardhat-ethers");
require("dotenv").config();

module.exports = {
  defaultNetwork: "bsctestnet",
  networks: {
    hardhat: {},
    bsctestnet: {
      url: "https://data-seed-prebsc-1-s1.binance.org:8545/",
      accounts: [`0x${process.env.PRIVATE_KEY}`]
    },
  },
  solidity: "0.8.0",
};

Create a `.env` file in the `smart-contracts` directory and add your private key:

PRIVATE_KEY=your_private_key_here

**Important:** Ensure `.env` is added to `.gitignore` to prevent sensitive information from being committed.

---

## Front-End Application

Navigate to the `frontend` directory:

cd ../frontend

Initialize a React app:

npx create-react-app .

Install necessary dependencies:

npm install ethers react-router-dom dotenv

### Project Structure

frontend/
  - src/
    - components/
      - WalletConnector.js
      - ContractInteraction.js
    - App.js
    - index.js
  - package.json
  - ...

### WalletConnector.js

Create `src/components/WalletConnector.js`:

import React, { useState } from 'react';
import { ethers } from 'ethers';

function WalletConnector() {
  const [address, setAddress] = useState(null);

  const connectWallet = async () => {
    if (window.ethereum) {
      try {
        await window.ethereum.request({ method: 'eth_requestAccounts' });
        const provider = new ethers.providers.Web3Provider(window.ethereum);
        const signer = provider.getSigner();
        setAddress(await signer.getAddress());
      } catch (error) {
        console.error("User rejected request");
      }
    } else {
      alert("MetaMask not detected");
    }
  };

  return (
    <div>
      {address ? (
        <p>Connected: {address}</p>
      ) : (
        <button onClick={connectWallet}>Connect MetaMask</button>
      )}
    </div>
  );
}

export default WalletConnector;

### ContractInteraction.js

Create `src/components/ContractInteraction.js`:

import React, { useEffect, useState } from 'react';
import { ethers } from 'ethers';
import AuctionHouse from '../artifacts/contracts/AuctionHouse.sol/AuctionHouse.json';

function ContractInteraction() {
  const [auctionHouse, setAuctionHouse] = useState(null);
  const [properties, setProperties] = useState([]);

  useEffect(() => {
    const init = async () => {
      if (window.ethereum) {
        const address = "YOUR_AUCTION_HOUSE_CONTRACT_ADDRESS"; // Replace with your deployed contract address
        const provider = new ethers.providers.Web3Provider(window.ethereum);
        const contract = new ethers.Contract(address, AuctionHouse.abi, provider);
        setAuctionHouse(contract);

        // Fetch properties
        const propertyCount = await contract.propertyCount();
        const props = [];
        for (let i = 1; i <= propertyCount; i++) {
          const property = await contract.properties(i);
          props.push(property);
        }
        setProperties(props);
      }
    };
    init();
  }, []);

  return (
    <div>
      <h2>Properties</h2>
      {properties.map((property, index) => (
        <div key={index}>
          <p>ID: {property.id.toString()}</p>
          <p>Seller: {property.seller}</p>
          <p>Min Price: {ethers.utils.formatEther(property.minPrice)} EACL</p>
          <p>Details: {property.details}</p>
          <p>Is Sold: {property.isSold ? 'Yes' : 'No'}</p>
        </div>
      ))}
    </div>
  );
}

export default ContractInteraction;

### App.js

Update `src/App.js`:

import React from 'react';
import WalletConnector from './components/WalletConnector';
import ContractInteraction from './components/ContractInteraction';

function App() {
  return (
    <div>
      <h1>Estate Auction House (EACL)</h1>
      <WalletConnector />
      <ContractInteraction />
    </div>
  );
}

export default App;

---

## Deployment and Testing

### Compile and Deploy Smart Contracts

Navigate to the `smart-contracts` directory:

cd ../smart-contracts

Compile the contracts:

npx hardhat compile

Deploy the contracts to the BSC testnet:

npx hardhat run scripts/deploy.js --network bsctestnet

**Note:** Ensure you have testnet BNB in your account for gas fees.

Record the deployed contract addresses. You'll need them for the front-end.

### Update Front-End with Contract Addresses

In `src/components/ContractInteraction.js`, replace `"YOUR_AUCTION_HOUSE_CONTRACT_ADDRESS"` with the actual address of the deployed `AuctionHouse` contract.

### Copy Contract Artifacts to Front-End

After compiling the contracts, copy the `artifacts` directory to the front-end `src` directory:

cp -r artifacts ../frontend/src/

### Run the Front-End Application

Navigate to the `frontend` directory:

cd ../frontend

Start the development server:

npm start

Access the application at `http://localhost:3000`.

---

## Additional Notes

- **Security Considerations**: Before deploying to the mainnet, perform a thorough security audit of the smart contracts.
- **Testing**: Implement unit tests for both smart contracts and front-end components.
- **UI/UX Enhancements**: The current front-end is a basic prototype. Consider improving the user interface.
- **Integration with Oracles**: Future iterations should include integration with property data sources like Zillow and LoopNet.
- **Legal Compliance**: Ensure all legal requirements for real estate transactions are met.

---

## License

This project is licensed under the MIT License.

---

*End of README*
