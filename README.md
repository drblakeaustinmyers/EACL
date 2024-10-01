Estate Auction House (EACL) Platform Development Guide
Table of Contents
Executive Summary
Project Overview
Technical Requirements
Project Structure
Smart Contracts Development
EACLToken Contract
AuctionHouse Contract
Frontend Development
ReactJS Setup
Integrating Smart Contracts
Deployment and Testing
Binance Smart Chain Testnet Deployment
Running the Frontend Application
README Files and Documentation
Next Steps and Recommendations
Copying and Pasting Instructions
Executive Summary
Estate Auction House (EACL) is an innovative, decentralized real estate auction platform built on blockchain technology. Aimed at transforming the traditional real estate market, EACL leverages the Binance Smart Chain to facilitate secure, transparent, and efficient property transactions. The platform introduces a unique auction model where buyers place one-time silent bids, ensuring sellers receive the highest possible offers while expanding options for buyers globally.

Key features include:

EACLToken: A BEP20 token used for transactions within the ecosystem.
Silent Auctions: Buyers must hold 10% of the property's value in EACL tokens to place a bid.
Seller Protections: Non-refundable deposits and penalties safeguard sellers against non-completion.
Low Listing Fees: Sellers can list properties with only $100 worth of EACL tokens.
Global Reach: The platform taps into both domestic and international markets, often yielding 20-30% premiums for sellers.
Oracle Integration: Incorporates data from Zillow and LoopNet for accurate property valuations.
Project Overview
The goal is to develop a prototype of the EACL platform that includes:

Smart Contracts: Implementing the EACLToken and AuctionHouse contracts using Solidity.
Frontend Application: Building a ReactJS application that interacts with the smart contracts.
Test Network Deployment: Deploying contracts on the Binance Smart Chain Testnet for testing purposes.
This prototype will serve as a starting point for further development and provide a foundation for your programming team to build upon.

Technical Requirements
Prerequisites
Node.js and npm: Ensure Node.js (v14 or above) and npm are installed.
Truffle or Hardhat: For smart contract development and deployment.
MetaMask: Browser extension for managing blockchain accounts.
Git: For version control.
IDE/Text Editor: Such as Visual Studio Code or Sublime Text.
Technologies Used
Blockchain Network: Binance Smart Chain (BSC) Testnet.
Smart Contracts Language: Solidity.
Frontend Framework: ReactJS.
Token Standard: BEP20 (similar to ERC20).
Project Structure
java
Copy code
EACL-Platform/
├── contracts/
│   ├── EACLToken.sol
│   ├── AuctionHouse.sol
│   └── Migrations.sol
├── migrations/
│   ├── 1_initial_migration.js
│   └── 2_deploy_contracts.js
├── frontend/
│   ├── src/
│   │   ├── components/
│   │   │   ├── PropertyList.js
│   │   │   ├── PropertyDetails.js
│   │   │   └── BidForm.js
│   │   ├── App.js
│   │   ├── index.js
│   │   └── contracts/
│   │       ├── EACLToken.json
│   │       └── AuctionHouse.json
│   ├── public/
│   └── package.json
├── test/
│   ├── EACLToken.test.js
│   └── AuctionHouse.test.js
├── truffle-config.js
├── package.json
└── README.md
Smart Contracts Development
1. Setting Up the Development Environment
Initialize the Truffle Project

bash
Copy code
mkdir EACL-Platform
cd EACL-Platform
truffle init
Install Dependencies

bash
Copy code
npm init -y
npm install @openzeppelin/contracts
2. EACLToken Contract
This contract represents the BEP20 token used within the platform. It inherits from OpenZeppelin's ERC20 implementation.

File: contracts/EACLToken.sol

solidity
Copy code
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";

contract EACLToken is ERC20 {
    constructor(uint256 initialSupply) ERC20("Estate Auction Coin", "EACL") {
        _mint(msg.sender, initialSupply * (10 ** decimals()));
    }
}
Key Features:

Token Name: Estate Auction Coin
Symbol: EACL
Initial Supply: Set during deployment.
3. AuctionHouse Contract
Handles property listings, bidding processes, and transaction settlements.

File: contracts/AuctionHouse.sol

solidity
Copy code
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "./EACLToken.sol";

contract AuctionHouse {
    EACLToken public eaclToken;

    uint256 public propertyCount = 0;

    struct Property {
        uint256 id;
        address payable seller;
        string details;
        uint256 minPrice;
        bool sold;
    }

    struct Bid {
        address payable bidder;
        uint256 amount;
        bool accepted;
    }

    mapping(uint256 => Property) public properties;
    mapping(uint256 => Bid[]) public propertyBids;

    event PropertyListed(uint256 id, address seller, uint256 minPrice);
    event BidPlaced(uint256 propertyId, address bidder, uint256 amount);
    event BidAccepted(uint256 propertyId, address seller, address bidder, uint256 amount);
    event BidderPenalized(uint256 propertyId, address bidder, uint256 penaltyAmount);

    constructor(EACLToken _eaclToken) {
        eaclToken = _eaclToken;
    }

    // Function to list a property
    function listProperty(string memory _details, uint256 _minPrice) public {
        require(
            eaclToken.balanceOf(msg.sender) >= 100 * (10 ** eaclToken.decimals()),
            "You need $100 worth of EACL tokens to list a property."
        );

        propertyCount++;
        properties[propertyCount] = Property(
            propertyCount,
            payable(msg.sender),
            _details,
            _minPrice,
            false
        );

        emit PropertyListed(propertyCount, msg.sender, _minPrice);
    }

    // Function to place a bid
    function placeBid(uint256 _propertyId, uint256 _bidAmount) public {
        Property storage property = properties[_propertyId];
        require(!property.sold, "Property already sold.");
        require(
            eaclToken.balanceOf(msg.sender) >= (_bidAmount * 10) / 100,
            "You must hold 10% of the bid amount in EACL tokens."
        );

        propertyBids[_propertyId].push(
            Bid(payable(msg.sender), _bidAmount, false)
        );

        emit BidPlaced(_propertyId, msg.sender, _bidAmount);
    }

    // Function to accept a bid
    function acceptBid(uint256 _propertyId, uint256 _bidIndex) public {
        Property storage property = properties[_propertyId];
        require(msg.sender == property.seller, "Only the seller can accept bids.");
        require(!property.sold, "Property already sold.");

        Bid storage bid = propertyBids[_propertyId][_bidIndex];
        require(!bid.accepted, "Bid already accepted.");

        // Transfer 3% commission from buyer to platform
        uint256 commission = (bid.amount * 3) / 100;
        eaclToken.transferFrom(bid.bidder, address(this), commission);

        // Transfer remaining amount to seller
        uint256 sellerAmount = bid.amount - commission;
        eaclToken.transferFrom(bid.bidder, property.seller, sellerAmount);

        bid.accepted = true;
        property.sold = true;

        emit BidAccepted(_propertyId, property.seller, bid.bidder, bid.amount);
    }

    // Function to penalize a bidder who fails to complete the transaction
    function penalizeBidder(uint256 _propertyId, uint256 _bidIndex) public {
        Property storage property = properties[_propertyId];
        require(msg.sender == property.seller, "Only the seller can penalize.");
        require(!property.sold, "Property already sold.");

        Bid storage bid = propertyBids[_propertyId][_bidIndex];
        require(!bid.accepted, "Bid already accepted.");

        // Calculate 2% penalty
        uint256 penaltyAmount = (bid.amount * 2) / 100;
        eaclToken.transferFrom(bid.bidder, property.seller, penaltyAmount);

        // Remove the bid
        delete propertyBids[_propertyId][_bidIndex];

        emit BidderPenalized(_propertyId, bid.bidder, penaltyAmount);
    }
}
Key Features:

Property Listing: Sellers can list properties by staking $100 worth of EACL tokens.
Bidding Mechanism: Buyers place bids and must hold 10% of the bid amount in EACL tokens.
Bid Acceptance: Sellers can accept bids, triggering token transfers and commission deductions.
Penalties: Sellers can penalize bidders who fail to complete transactions.
Frontend Development
1. ReactJS Setup
Initialize the React App

bash
Copy code
cd EACL-Platform
npx create-react-app frontend
Install Dependencies

bash
Copy code
cd frontend
npm install web3 @openzeppelin/contracts react-router-dom
2. Project Structure
java
Copy code
frontend/
├── src/
│   ├── components/
│   │   ├── Navbar.js
│   │   ├── PropertyList.js
│   │   ├── PropertyDetails.js
│   │   ├── ListProperty.js
│   │   └── BidForm.js
│   ├── App.js
│   ├── index.js
│   └── contracts/
│       ├── EACLToken.json
│       └── AuctionHouse.json
├── public/
├── package.json
3. Integrating Smart Contracts
Note: After deploying your contracts, copy the EACLToken.json and AuctionHouse.json files generated by Truffle into the frontend/src/contracts/ directory.

File: frontend/src/App.js

jsx
Copy code
import React, { useEffect, useState } from 'react';
import Web3 from 'web3';
import { BrowserRouter as Router, Route, Routes } from 'react-router-dom';
import AuctionHouseContract from './contracts/AuctionHouse.json';
import EACLTokenContract from './contracts/EACLToken.json';
import Navbar from './components/Navbar';
import PropertyList from './components/PropertyList';
import PropertyDetails from './components/PropertyDetails';
import ListProperty from './components/ListProperty';

function App() {
  const [account, setAccount] = useState('');
  const [auctionHouse, setAuctionHouse] = useState(null);
  const [eaclToken, setEaclToken] = useState(null);
  const [properties, setProperties] = useState([]);

  useEffect(() => {
    loadBlockchainData();
  }, []);

  const loadBlockchainData = async () => {
    if (window.ethereum) {
      window.web3 = new Web3(window.ethereum);
      await window.ethereum.request({ method: 'eth_requestAccounts' });
    } else {
      window.alert('Please install MetaMask!');
      return;
    }

    const web3 = window.web3;
    const accounts = await web3.eth.getAccounts();
    setAccount(accounts[0]);

    const networkId = await web3.eth.net.getId();

    // Load AuctionHouse Contract
    const auctionHouseData = AuctionHouseContract.networks[networkId];
    if (auctionHouseData) {
      const auctionHouseInstance = new web3.eth.Contract(
        AuctionHouseContract.abi,
        auctionHouseData.address
      );
      setAuctionHouse(auctionHouseInstance);

      // Load Properties
      const propertyCount = await auctionHouseInstance.methods.propertyCount().call();
      const loadedProperties = [];
      for (let i = 1; i <= propertyCount; i++) {
        const property = await auctionHouseInstance.methods.properties(i).call();
        loadedProperties.push(property);
      }
      setProperties(loadedProperties);
    } else {
      window.alert('AuctionHouse contract not deployed to detected network.');
    }

    // Load EACLToken Contract
    const eaclTokenData = EACLTokenContract.networks[networkId];
    if (eaclTokenData) {
      const eaclTokenInstance = new web3.eth.Contract(
        EACLTokenContract.abi,
        eaclTokenData.address
      );
      setEaclToken(eaclTokenInstance);
    } else {
      window.alert('EACLToken contract not deployed to detected network.');
    }
  };

  return (
    <Router>
      <div>
        <Navbar account={account} />
        <Routes>
          <Route
            path="/"
            element={<PropertyList properties={properties} />}
          />
          <Route
            path="/property/:id"
            element={<PropertyDetails properties={properties} />}
          />
          <Route
            path="/list-property"
            element={<ListProperty auctionHouse={auctionHouse} account={account} />}
          />
        </Routes>
      </div>
    </Router>
  );
}

export default App;
Component Breakdown:

Navbar: Displays navigation links and the user's account.
PropertyList: Shows a list of available properties.
PropertyDetails: Displays details of a selected property and allows bidding.
ListProperty: Form for sellers to list new properties.
Deployment and Testing
1. Binance Smart Chain Testnet Deployment
Configure Truffle for BSC Testnet

Install HDWalletProvider

bash
Copy code
npm install @truffle/hdwallet-provider
File: truffle-config.js

javascript
Copy code
const HDWalletProvider = require('@truffle/hdwallet-provider');
const mnemonic = 'your mnemonic here'; // Replace with your wallet's mnemonic

module.exports = {
  networks: {
    bscTestnet: {
      provider: () =>
        new HDWalletProvider(
          mnemonic,
          'https://data-seed-prebsc-1-s1.binance.org:8545/'
        ),
      network_id: 97,
      confirmations: 10,
      timeoutBlocks: 200,
      skipDryRun: true,
    },
  },
  compilers: {
    solc: {
      version: '0.8.0',
    },
  },
};
Deploy Contracts

bash
Copy code
truffle migrate --network bscTestnet
2. Running the Frontend Application
Start the React App

bash
Copy code
cd frontend
npm start
Connect MetaMask to BSC Testnet

Open MetaMask.
Click on the network dropdown and select "Custom RPC".
Enter the following details:
Network Name: BSC Testnet
New RPC URL: https://data-seed-prebsc-1-s1.binance.org:8545/
Chain ID: 97
Currency Symbol: BNB
Block Explorer URL: https://testnet.bscscan.com
Import Test EACL Tokens

In MetaMask, add the EACLToken contract address to view token balances.
Click on "Import Tokens" and enter the contract address.
README Files and Documentation
Root README.md
File: README.md

markdown
Copy code
# Estate Auction House (EACL) Platform

A decentralized real estate auction platform built on the Binance Smart Chain Testnet.

## Overview

Estate Auction House (EACL) is a blockchain-based platform aiming to revolutionize real estate transactions by leveraging decentralized technologies.

## Features

- Property Listings and Auctions
- Secure Bidding Mechanism
- Integration with EACLToken (BEP20)
- Penalty Enforcement for Non-Completion
- Oracle Integration for Property Data (Future Implementation)

## Getting Started

### Prerequisites

- Node.js (v14 or above)
- npm
- Truffle
- MetaMask Extension
- Git

### Installation

1. **Clone the Repository**

   ```bash
   git clone <repository_url>
Install Backend Dependencies

bash
Copy code
cd EACL-Platform
npm install
Install Frontend Dependencies

bash
Copy code
cd frontend
npm install
Deployment
Compile Smart Contracts

bash
Copy code
truffle compile
Deploy to BSC Testnet

bash
Copy code
truffle migrate --network bscTestnet
Running the Application
Start the Frontend

bash
Copy code
cd frontend
npm start
Open in Browser

Navigate to http://localhost:3000 in your web browser.

Testing
Smart Contract Tests

bash
Copy code
truffle test
Next Steps and Recommendations
Implement Silent Auction Logic: Modify the auction contract to handle one-time silent bids.
Enforce 72-Hour Completion Window: Add functionality to enforce the 72-hour transaction completion requirement.
Integrate Oracles for Property Data: Use Chainlink or similar services to fetch external property data.
Enhance UI/UX: Improve the frontend design for better user experience.
Security Audits: Conduct comprehensive security audits of the smart contracts.
Expand Testing: Write additional tests to cover edge cases and ensure robustness.
Documentation: Create detailed documentation for developers and end-users.
