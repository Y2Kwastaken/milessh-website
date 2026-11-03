---
title: 'RPC Marketplace'
tech: ['React 19', 'Spring Boot 4.0', 'Java 21', 'Hedera Hashgraph', 'MySQL', 'Docker']
lead: 'A containerized, microservice-driven e-commerce platform where traditional web infrastructure meets distributed ledger technology.'
---

For this academic project, our team was tasked with building a full-stack Web3 cryptocurrency-driven marketplace. I focused heavily on backend and cryptocurrency orchestration, ensuring that user data, inventory states, and real-time cryptocurrency transactions remained perfectly synchronized across multiple independent systems.

## The Core Architecture

### The Client Layer

A responsive React interface focused on immediate user feedback. To reduce server load and improve perceived performance, I helped implement custom UI features like client-side image previews, ensuring users can validate their listings before any heavy network requests are made.

### The RESTful Layer

A Java Spring Boot backend acting as the central traffic controller. It handles stateless, token-based user authentication and strictly validates all incoming requests before communicating with the database or the external cryptocurrency network.

### The Ledger Integration

A dedicated Java service isolated from the main web traffic. It safely manages interactions with the Hedera Hashgraph network, provisioning secure user wallets behind the scenes and processing native $RPC token transfers without ever exposing private keys to the frontend.

## Engineering Challenge: Distributed State Management

The most significant architectural hurdle in this project was bridging a traditional relational database (MySQL) with a decentralized blockchain (Hedera).

In a standard web app, purchasing an item is a single database transaction. If it fails, everything rolls back. However, when a cryptocurrency network is involved, you run the risk of the database updating successfully but the blockchain transfer failing (or vice versa), resulting in lost items or lost funds.

### The Solution: Safe Transaction Flow

1. **Pre-Validation:** The Spring Boot backend first queries the MySQL database to verify the item is in stock, and concurrently queries the Hedera network to verify the buyer has sufficient $RPC funds.
2. **State Locking:** The item is temporarily marked as "Pending" in the database, preventing other users from purchasing it simultaneously (preventing race conditions).
3. **Ledger Execution:** The isolated Crypto Service constructs and submits the transaction to the Hedera network, awaiting a consensus finality receipt.
4. **Reconciliation:** Only upon receiving mathematical proof of the transfer from the blockchain does the database permanently transfer item ownership and record the order history. If the ledger fails, the database lock is released safely.

## Infrastructure & Reliability

To guarantee stability across these interacting services, the entire environment is fully containerized using Docker. Furthermore, I established an automated CI/CD pipeline. Every code change triggers isolated testing environments where the frontend is validated, the backend business logic is verified, and the crypto service executes mock transactions against a Hedera testnet, ensuring deployment confidence.
