# Architecture Overview

The Gondor Ecosystem is built around a single governing design principle: **separate what must be fast from what must be trustworthy**.

Data management happens in traditional Web2 infrastructure (Go, MySQL, Redis) for operational performance. Truth certification happens on a public blockchain (Polygon) for immutability and autonomous verifiability. These two worlds are bridged by a cryptographic coprocessor — **Pelargir** — that lives inside the same monolith as the financial engine **Minas Tirith** but operates as an independent bounded context.

---

## The Hybrid Validation Model

The core innovation of Gondor is not the use of blockchain per se — it is the **hybrid validation model** that makes it viable:

```
Web2 (fast, private)          Web3 (immutable, public)
─────────────────────         ───────────────────────────
Trade execution               Signed receipt (EdDSA/Poseidon)
Portfolio calculation         Merkle root anchored on-chain
User session (JWT)            Soulbound Token (non-transferable SBT)
ROI computation               ZK-SNARK proof of ROI without data exposure
```

This model solves a fundamental tension: proving financial performance to a public ledger without exposing the transaction details that constitute an investor's competitive advantage (strategy, timing, position sizes).

---

## System Components

### Minas Tirith (Web2 Core)

The high-speed portfolio management engine written in **Go**, following **Hexagonal Architecture (Ports and Adapters)**. It handles:

- Trade execution (buy/sell by units or by invested amount)
- Real-time portfolio valuation
- Stock price retrieval with **Redis caching** and multi-provider rotation (Finnhub, Alpaca)
- User authentication via JWT
- Concurrent price updates using the **Fan-in/Fan-out** pattern

### Pelargir — Phase I: Sovereign Identity

The blockchain coprocessor module handling **identity materialization**:

- Generates an Ethereum wallet for each user transparently (no MetaMask, no seed phrases)
- Mints a **Soulbound Token (SBT)** on Polygon Amoy via the `GondorSovereignIdentity.sol` contract
- Ensures atomic registration across MySQL and blockchain via the `IdentityBridge` pattern with 5-retry fallback

### Pelargir — Phase II: Cryptographic Certification

The cryptographic certification layer implementing the *Verify-me* architecture:

- **Signed receipts**: Each trade is sealed with an EdDSA signature over a Poseidon hash at the moment of execution
- **Merkle Batching**: N receipts are aggregated into a Poseidon Merkle Tree; the root is anchored on-chain via `anchorRoot`
- **ZK-SNARK proofs**: The `BatchTxVerifier` circuit (Circom/Groth16/BN254) proves accumulated ROI to the blockchain without revealing private transaction details
- **Heartbeat**: On-chain and off-chain mechanisms prevent concealing losses through inactivity

### Frontend

Two client applications with a dual-channel design:
- **`gondor-app`** (React Native/Expo): Web2 channel for trading via REST API + Web3 channel for reading SBT reputation directly from Polygon via `viem`
- **Audit Portal** (React): Read-only Web3 view of any investor's on-chain reputation

---

## Technology Stack

| Layer | Technology | Rationale |
|---|---|---|
| Backend language | Go 1.22 | Native goroutines for Fan-in/Fan-out concurrency; strong typing |
| Web framework | Gin-gonic | Lightweight HTTP router with middleware support |
| Database | MySQL 8 | ACID transactions required for financial data consistency |
| Cache | Redis | Microsecond latency for stock price serving |
| Smart contracts | Solidity 0.8.24 + OpenZeppelin v5 | Industry-standard ERC-721 base for SBT implementation |
| Blockchain | Polygon Amoy (testnet) | ~$0.001 per tx vs ~$0.03 on Ethereum mainnet; 2s block time |
| ZK framework | Circom 2.1.6 + SnarkJS | Native support for Poseidon hash and EdDSA BabyJubJub in-circuit |
| ZK proof scheme | Groth16 over BN254 | Smallest proof size (3 EC points = ~256 bytes); constant-time verification |
| Hash function (ZK) | Poseidon | SNARK-friendly hash — significantly fewer constraints than SHA-256 or Keccak |
| Signature scheme | EdDSA over BabyJubJub | Native to ZK circuits; eliminates the need for in-circuit signature verifier adapters |
| Mobile app | React Native + Expo | Cross-platform iOS/Android; `viem` for direct Polygon RPC calls |
| Contract testing | Hardhat + Chai | EVM-in-memory environment for deterministic smart contract testing |
| Integration testing | Testcontainers-Go | Docker-based E2E tests without mocking the blockchain layer |

---

## Architectural Decision Records

Key design decisions are documented as ADRs in [`architecture/decisions/`](./decisions/README.md):

| ADR | Decision |
|---|---|
| [ADR-001](./decisions/ADR-001-hybrid-web2-web3.md) | Hybrid Web2/Web3 instead of fully on-chain |
| [ADR-002](./decisions/ADR-002-hexagonal-architecture.md) | Hexagonal Architecture for the Go monolith |
| [ADR-003](./decisions/ADR-003-zk-snark-groth16-bn254.md) | Groth16/BN254 for ZK-SNARK proof scheme |
| [ADR-004](./decisions/ADR-004-soulbound-token-polygon.md) | Soulbound Token on Polygon Amoy |
| [ADR-005](./decisions/ADR-005-privacy-vs-transparency.md) | Privacy-preserving ZK over transparent anchoring |
