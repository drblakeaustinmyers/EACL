Estate Auction House (EACL) Prototype
Table of Contents

Executive Summary
Project Overview
Technical Architecture
Smart Contracts
Frontend
Backend
Setup Instructions
Testing
Deployment
Future Enhancements

Executive Summary
Estate Auction House (EACL) is a cutting-edge, Web3-enabled real estate blockchain ecosystem designed to revolutionize the purchase and sale of commercial and residential properties. Initially developed on the Binance Smart Chain, EACL aims to dominate the global real estate market through its decentralized platform, which maximizes seller returns and expands buyer options using an innovative auction model.
Key Features:

Decentralized real estate auctions on the blockchain
Silent bidding system with one-time bids
10% EACL token holding requirement for buyers
72-hour transaction finalization window
Integration with oracles for property valuations
Low listing fee for sellers ($100 worth of EACL tokens)
3% commission on both buyers and sellers

Project Overview
The EACL platform introduces a unique auction format that streamlines the real estate transaction process. Buyers place one-time silent bids, ensuring the highest possible offer for sellers. To participate, buyers must hold 10% of the property's value in EACL tokens in their MetaMask wallet, guaranteeing serious intent and financial backing.
The winning bidder has 72 hours to finalize the transaction, purchasing the remaining EACL tokens necessary to complete the property sale. In the event of non-completion, a 2% penalty of the initial 10% deposit is paid to the seller as a non-refundable deposit, safeguarding the seller's interests.
Sellers within the EACL ecosystem need only $100 worth of EACL tokens to list their properties, accompanied by a simple "sell as-is" contract, property photos, videos, and a brief description of any damages. EACL levies a 3% commission on both sellers and buyers, ensuring a transparent and efficient marketplace.
Technical Architecture
The EACL prototype consists of three main components:

Smart Contracts (Solidity)
Frontend (React.js)
Backend (Node.js with Express)

The system uses the Binance Smart Chain testnet for development and testing purposes.
Smart Contracts
The core functionality of the EACL platform is implemented in Solidity smart contracts. Here's the main auction contract:
solidityCopy// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";

contract EACLAuction is ReentrancyGuard {
    IERC20 public eaclToken;
    
    struct Auction {
        uint256 propertyId;
        address seller;
        uint256 startingPrice;
        uint256 endTime;
        address highestBidder;
        uint256 highestBid;
        bool ended;
    }
    
    mapping(uint256 => Auction) public auctions;
    mapping(uint256 => mapping(address => uint256)) public bids;
    
    event AuctionCreated(uint256 indexed propertyId, address indexed seller, uint256 startingPrice, uint256 endTime);
    event BidPlaced(uint256 indexed propertyId, address indexed bidder, uint256 amount);
    event AuctionEnded(uint256 indexed propertyId, address winner, uint256 amount);
    
    constructor(address _eaclTokenAddress) {
        eaclToken = IERC20(_eaclTokenAddress);
    }
    
    function createAuction(uint256 _propertyId, uint256 _startingPrice, uint256 _duration) external {
        require(eaclToken.transferFrom(msg.sender, address(this), 100 * 10**18), "Failed to transfer listing fee");
        
        auctions[_propertyId] = Auction({
            propertyId: _propertyId,
            seller: msg.sender,
            startingPrice: _startingPrice,
            endTime: block.timestamp + _duration,
            highestBidder: address(0),
            highestBid: 0,
            ended: false
        });
        
        emit AuctionCreated(_propertyId, msg.sender, _startingPrice, block.timestamp + _duration);
    }
    
    function placeBid(uint256 _propertyId, uint256 _bidAmount) external nonReentrant {
        Auction storage auction = auctions[_propertyId];
        require(block.timestamp < auction.endTime, "Auction has ended");
        require(_bidAmount > auction.highestBid, "Bid must be higher than current highest bid");
        require(eaclToken.balanceOf(msg.sender) >= _bidAmount * 10 / 100, "Insufficient EACL tokens for 10% deposit");
        
        if (auction.highestBidder != address(0)) {
            // Refund the previous highest bidder
            bids[_propertyId][auction.highestBidder] = 0;
        }
        
        auction.highestBidder = msg.sender;
        auction.highestBid = _bidAmount;
        bids[_propertyId][msg.sender] = _bidAmount;
        
        emit BidPlaced(_propertyId, msg.sender, _bidAmount);
    }
    
    function endAuction(uint256 _propertyId) external {
        Auction storage auction = auctions[_propertyId];
        require(block.timestamp >= auction.endTime, "Auction has not ended yet");
        require(!auction.ended, "Auction has already been ended");
        
        auction.ended = true;
        
        if (auction.highestBidder != address(0)) {
            // Transfer 90% of the bid amount to the seller
            uint256 sellerAmount = auction.highestBid * 90 / 100;
            require(eaclToken.transfer(auction.seller, sellerAmount), "Failed to transfer funds to seller");
            
            // Transfer 3% commission to the platform
            uint256 commissionAmount = auction.highestBid * 3 / 100;
            require(eaclToken.transfer(owner(), commissionAmount), "Failed to transfer commission");
        }
        
        emit AuctionEnded(_propertyId, auction.highestBidder, auction.highestBid);
    }
    
    function withdrawBid(uint256 _propertyId) external {
        require(auctions[_propertyId].ended, "Auction has not ended yet");
        require(msg.sender != auctions[_propertyId].highestBidder, "Winner cannot withdraw bid");
        
        uint256 bidAmount = bids[_propertyId][msg.sender];
        require(bidAmount > 0, "No bid to withdraw");
        
        bids[_propertyId][msg.sender] = 0;
        require(eaclToken.transfer(msg.sender, bidAmount), "Failed to transfer bid amount");
    }
}
This contract implements the core auction functionality, including creating auctions, placing bids, ending auctions, and withdrawing bids.
Frontend
The frontend is built using React.js. Here's an example component for displaying an auction listing:
jsxCopyimport React, { useState, useEffect } from 'react';
import { ethers } from 'ethers';
import { Card, Button, Form, Input, message } from 'antd';
import EACLAuctionABI from './EACLAuctionABI.json';

const AuctionListing = ({ propertyId, contractAddress }) => {
  const [auction, setAuction] = useState(null);
  const [bidAmount, setBidAmount] = useState('');

  useEffect(() => {
    fetchAuctionDetails();
  }, [propertyId]);

  const fetchAuctionDetails = async () => {
    try {
      const provider = new ethers.providers.Web3Provider(window.ethereum);
      const contract = new ethers.Contract(contractAddress, EACLAuctionABI, provider);
      const auctionDetails = await contract.auctions(propertyId);
      setAuction(auctionDetails);
    } catch (error) {
      console.error('Error fetching auction details:', error);
      message.error('Failed to fetch auction details');
    }
  };

  const handleBid = async () => {
    try {
      const provider = new ethers.providers.Web3Provider(window.ethereum);
      const signer = provider.getSigner();
      const contract = new ethers.Contract(contractAddress, EACLAuctionABI, signer);
      
      const tx = await contract.placeBid(propertyId, ethers.utils.parseEther(bidAmount));
      await tx.wait();
      
      message.success('Bid placed successfully');
      fetchAuctionDetails();
    } catch (error) {
      console.error('Error placing bid:', error);
      message.error('Failed to place bid');
    }
  };

  if (!auction) {
    return <div>Loading auction details...</div>;
  }

  return (
    <Card title={`Auction for Property #${propertyId}`}>
      <p>Seller: {auction.seller}</p>
      <p>Starting Price: {ethers.utils.formatEther(auction.startingPrice)} EACL</p>
      <p>End Time: {new Date(auction.endTime.toNumber() * 1000).toLocaleString()}</p>
      <p>Highest Bid: {ethers.utils.formatEther(auction.highestBid)} EACL</p>
      <p>Highest Bidder: {auction.highestBidder}</p>
      
      <Form onFinish={handleBid}>
        <Form.Item label="Bid Amount (EACL)">
          <Input
            value={bidAmount}
            onChange={(e) => setBidAmount(e.target.value)}
            placeholder="Enter bid amount"
          />
        </Form.Item>
        <Form.Item>
          <Button type="primary" htmlType="submit">
            Place Bid
          </Button>
        </Form.Item>
      </Form>
    </Card>
  );
};

export default AuctionListing;
This component displays auction details and allows users to place bids.
Backend
The backend is built using Node.js with Express. Here's a basic implementation:
javascriptCopyconst express = require('express');
const { ethers } = require('ethers');
const cors = require('cors');
const EACLAuctionABI = require('./EACLAuctionABI.json');

const app = express();
app.use(cors());
app.use(express.json());

const contractAddress = '0x...'; // Replace with your deployed contract address
const provider = new ethers.providers.JsonRpcProvider('https://data-seed-prebsc-1-s1.binance.org:8545/'); // Use Binance Smart Chain testnet
const contract = new ethers.Contract(contractAddress, EACLAuctionABI, provider);

app.post('/api/properties', async (req, res) => {
  try {
    const { address, description, imageUrls, startingPrice, duration } = req.body;
    
    // Store property details in your database
    const propertyId = await storePropertyInDatabase(address, description, imageUrls);
    
    // Create auction on the blockchain
    const signer = new ethers.Wallet(process.env.PRIVATE_KEY, provider);
    const tx = await contract.connect(signer).createAuction(propertyId, ethers.utils.parseEther(startingPrice), duration);
    await tx.wait();
    
    res.json({ success: true, propertyId });
  } catch (error) {
    console.error('Error creating property listing:', error);
    res.status(500).json({ success: false, error: 'Failed to create property listing' });
  }
});

app.get('/api/properties/:id', async (req, res) => {
  try {
    const propertyId = req.params.id;
    
    // Fetch property details from your database
    const propertyDetails = await getPropertyFromDatabase(propertyId);
    
    // Fetch auction details from the blockchain
    const auctionDetails = await contract.auctions(propertyId);
    
    res.json({
      ...propertyDetails,
      auction: {
        startingPrice: ethers.utils.formatEther(auctionDetails.startingPrice),
        endTime: auctionDetails.endTime.toNumber(),
        highestBid: ethers.utils.formatEther(auctionDetails.highestBid),
        highestBidder: auctionDetails.highestBidder,
      },
    });
  } catch (error) {
    console.error('Error fetching property details:', error);
    res.status(500).json({ success: false, error: 'Failed to fetch property details' });
  }
});

const PORT = process.env.PORT || 3001;
app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});

// Implement these functions to interact with your database
async function storePropertyInDatabase(address, description, imageUrls) {
  // TODO: Implement database storage logic
}

async function getPropertyFromDatabase(propertyId) {
  // TODO: Implement database retrieval logic
}
This backend provides API endpoints for creating property listings and fetching property details.
Setup Instructions

Clone the repository:
Copygit clone https://github.com/your-username/eacl-prototype.git
cd eacl-prototype

Install dependencies:
Copynpm install
cd frontend && npm install
cd ../backend && npm install

Create a .env file in the root directory with the following variables:
CopyPRIVATE_KEY=your_private_key
INFURA_PROJECT_ID=your_infura_project_id

Compile and deploy smart contracts:
Copytruffle compile
truffle migrate --network bsc_testnet

Update the contract address in frontend/src/config.js and backend/config.js with the deployed contract address.
Start the backend server:
Copycd backend
npm start

Start the frontend development server:
Copycd frontend
npm start

Open your browser and navigate to http://localhost:3000 to access the EACL prototype.

Testing
To run the smart contract tests:
Copytruffle test
To run the frontend tests:
Copycd frontend
npm test
Deployment

Deploy smart contracts to the Binance Smart Chain testnet:
Copytruffle migrate --network bsc_testnet

Deploy the backend API to your preferred hosting platform (e.g., Heroku, DigitalOcean).
Deploy the frontend to a static hosting service (e.g., Netlify, Vercel).

Future Enhancements

Implement a proprietary blockchain for improved speed, security, and scalability.
Integrate with oracles (e.g., Chainlink) for real-time property valuations from Zillow and LoopNet.
Develop a mobile application for easier access to the platform.
Implement a reputation system for buyers and sellers.
Add support for fractional property ownership and tokenization.
Integrate with virtual reality (VR) platforms for immersive property tours.
7
