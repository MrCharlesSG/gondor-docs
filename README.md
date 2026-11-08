# Gondor Ecosystem — Technical Documentation

> A hybrid Web2/Web3 platform for sovereign, verifiable, and privacy-preserving financial reputation.

---

## The Problem

Social Trading platforms (eToro, Darwinex, ZuluTrade, Wikifolio) promised to democratize financial markets but introduced three structural failures rooted in their centralized architecture:

1. **Risk amplification** — Copy-trading mechanics generate herding behavior, cascading liquidations, and irrational risk-taking driven by social exposure rather than strategy (Apesteguia et al., 2020).
2. **Reputation lock-in** — An investor's performance history is stored in proprietary CRUD databases. If they leave the platform, they lose all verifiable track record. Legal portability (GDPR Art. 20) exists; cryptographic portability does not.
3. **Survivorship bias** — Platforms depublish accounts of failed traders, making visible only the survivors and systematically overstating expected returns (Elton et al., 1996).

These are not bugs — they are consequences of mutable, centralized databases controlled by a single administrator.

## The Solution

The **Gondor Ecosystem** is a functional prototype of a hybrid **Web2/Web3** architecture that allows an investor to build, certify, and prove their financial reputation in a **sovereign, verifiable, and privacy-preserving** manner.

- Data is managed in **Web2 infrastructure** (Go, MySQL, Redis) for operational performance.
- Its truthfulness is certified on a **public blockchain** (Polygon Amoy) through cryptographic receipts, Merkle Tree anchoring, and Zero-Knowledge Proofs (ZK-SNARKs).
- The investor's identity is materialized as a **non-transferable Soulbound Token (SBT)**.
- A **Heartbeat mechanism** prevents concealing losses through inactivity.

---

## Architecture at a Glance

<img width="1335" height="580" alt="image" src="https://github.com/user-attachments/assets/7e81facb-6ce3-4704-bafa-7ba38cebb8b8" />


| Component | Module | Role |
|---|---|---|
| **Minas Tirith** | Web2 Core | Portfolio management engine — trades, ROI, real-time prices via Redis |
| **Pelargir I** | Web3 Coprocessor | Sovereign identity — Soulbound Token minting on Polygon |
| **Pelargir II** | Web3 Coprocessor | Cryptographic certification — signed receipts, Merkle batching, ZK-SNARKs |
| **Frontend** | Presentation | React Native mobile app + React audit portal (dual Web2/Web3 channel) |

---

## Repositories

| Repository | Description | Language |
|---|---|---|
| [`gondor-backend`](https://github.com/MrCharlesSG/gondor-backend) | Go monolith — Minas Tirith + Pelargir | Go |
| [`gondor-blockchain`](https://github.com/MrCharlesSG/gondor-blockchain) | Smart contracts, Hardhat deployment | Solidity |
| [`gondor-app`](https://github.com/MrCharlesSG/gondor-app) | Mobile app (React Native/Expo) + Audit portal (React) | TypeScript |

---

## Documentation Index

### Architecture
- [Architecture Overview](./architecture/overview.md) — Global system design, the hybrid Web2/Web3 validation model, and technology stack rationale.
- [Architecture Decisions (ADRs)](./architecture/decisions/README.md) — Why each key technical decision was made.
  - [ADR-001: Hybrid Web2/Web3](./architecture/decisions/ADR-001-hybrid-web2-web3.md)
  - [ADR-002: Hexagonal Architecture](./architecture/decisions/ADR-002-hexagonal-architecture.md)
  - [ADR-003: ZK-SNARKs (Groth16/BN254)](./architecture/decisions/ADR-003-zk-snark-groth16-bn254.md)
  - [ADR-004: Soulbound Token on Polygon](./architecture/decisions/ADR-004-soulbound-token-polygon.md)
  - [ADR-005: Privacy vs. Transparency](./architecture/decisions/ADR-005-privacy-vs-transparency.md)
- [Global Architecture Diagram](./architecture/diagrams/global-architecture.md) — Full system Mermaid diagram with all component interactions.
- [Blockchain Interaction Flow](./architecture/diagrams/blockchain-flow.md) — End-to-end lifecycle of an investor across all four blockchain flows.
- [ZK-SNARK Pipeline](./architecture/diagrams/zk-pipeline.md) — Zero-Knowledge proof generation and on-chain verification pipeline.

### Components
- [Minas Tirith](./components/minas-tirith.md) — Web2 financial engine: hexagonal architecture, Fan-in/Fan-out concurrency, Redis cache strategy, API reference.
- [Pelargir I — Sovereign Identity](./components/pelargir-i.md) — Soulbound Token design, IdentityBridge pattern, atomic Web2+Web3 registration.
- [Pelargir II — Cryptographic Certification](./components/pelargir-ii.md) — EdDSA/Poseidon signed receipts, Merkle Batching, ZK-SNARK circuit, Heartbeat mechanism.
- [Frontend](./components/frontend.md) — Mobile app and audit portal: dual Web2/Web3 channel design.

### Development
- [Getting Started](./development/getting-started.md) — Prerequisites, local setup, and `make` command reference.
- [Testing Strategy](./development/testing.md) — Unit tests, Hardhat smart contract tests, Testcontainers E2E, stress simulator, and experimental results.

### Thesis (Archival Reference)
- [Thesis Documents](./thesis/README.md) — Original Master's thesis in Spanish and auto-translated English.

---

## Quick Start

```bash
# 1. Start infrastructure (MySQL + Redis)
make docker

# 2. Start local blockchain and deploy contracts
make hard-local

# 3. Run the Go backend
make run-local
```

→ Full setup guide: [development/getting-started.md](./development/getting-started.md)

---

## Future Work

The following extensions have been identified as natural next steps for the Gondor Ecosystem:

- **Microservices decoupling** — The Hexagonal Architecture and Go interfaces (`IdentityBridge`) are ready for evolution to microservices with gRPC contracts and RabbitMQ queues.
- **Mainnet deployment** — Migration to Polygon PoS mainnet with gas cost analysis and multi-root batching optimizations for `anchorRoot`.
- **Real broker integration** — Connect with regulated broker APIs (Interactive Brokers, Alpaca Trading) to ingest real executed orders rather than manually recorded ones.
- **Distributed Trusted Setup** — Replace the single-participant Groth16 ceremony with a public multi-party computation ceremony, eliminating the trust assumption on the key generator.
- **Recursive circuits and proof aggregation** — Apply IVC techniques (Nova/SuperNova) for continuous incremental updates, reducing gas costs by aggregating multiple Groth16 proofs.
- **W3C Verifiable Credentials** — Package the verified performance as an interoperable credential for other decentralized identity ecosystems (EBSI, eIDAS 2.0).
- **Expanded reputation model** — Incorporate risk-adjusted metrics (Sharpe Ratio, max drawdown) into the ZK circuit, mark-to-market valuation via Chainlink Price Feeds, and multi-asset class support.
