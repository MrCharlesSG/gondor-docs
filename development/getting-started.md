# Getting Started

> This guide walks you through setting up the complete Gondor Ecosystem locally.  
> For the full Makefile reference, see the [gondor-backend README](https://github.com/MrCharlesSG/gondor-backend#readme).

---

## Repositories

Clone the relevant repositories:

```bash
git clone https://github.com/MrCharlesSG/gondor-backend
git clone https://github.com/MrCharlesSG/gondor-blockchain
git clone https://github.com/MrCharlesSG/gondor-app
```

---

## Prerequisites

| Tool | Version | Purpose |
|---|---|---|
| **Go** | ≥ 1.22 | Backend runtime |
| **Docker & Docker Compose** | Any recent | MySQL + Redis infrastructure |
| **Node.js + npm** | LTS (v20) | Hardhat smart contract deployment + `abigen` tooling |
| **Circom + SnarkJS** | circom 2.1.x, snarkjs 0.7.x | Only if recompiling ZK circuits from scratch |

---

## Local Setup

Follow this exact order — each step depends on the previous one being healthy.

### Step 1 — Start Infrastructure (MySQL + Redis)

```bash
cd gondor-backend
make docker
```

Starts MySQL and Redis via Docker Compose. MySQL is automatically initialized with the schema from `db/schema.sql`.

### Step 2 — Start Local Blockchain and Deploy Contracts

```bash
make hard-local
```

This automated command:
1. Starts a local Hardhat/Anvil node (Chain ID: 31337)
2. Deploys `GondorSovereignIdentity.sol` and the Groth16 `Verifier.sol` from `gondor-blockchain`
3. Updates the contract addresses in your local config (`manifests/config-local.yaml`)
4. Resets the MySQL schema

Alternatively, run steps manually:
```bash
make local-node    # Start Hardhat node in a new terminal
make deploy-local  # Deploy contracts
```

### Step 3 — Run the Go Backend

```bash
make run-local
```

Starts the Gondor monolith with `APP_ENV=local`. The server listens at `http://localhost:8080`.

### Step 4 — Run the Stress Simulator (Optional)

```bash
make simulate
```

Registers multiple users, waits for their SBTs to be minted on-chain, then submits a stream of interleaved buy and sell orders. This is the fastest way to trigger the full pipeline: signing receipts → Merkle batching → ZK proof generation → on-chain reputation update.

---

## Configuration

The backend uses **Viper** for configuration, loaded from `manifests/config-local.yaml` when `APP_ENV=local`.

Key sections to configure for a local setup:

```yaml
minastirith:
  mysql:
    user: "app_user"
    password: "root"
    host: "127.0.0.1"
    port: 3306
    dbname: "minastirith_db"
  redis:
    host: "localhost"
    port: 6379
  clients:
    alpaca:
      apiKey: "YOUR_ALPACA_KEY"       # Get from alpaca.markets
      apiSecret: "YOUR_ALPACA_SECRET"
    finnhub:
      apiKey: "YOUR_FINNHUB_KEY"      # Get from finnhub.io (free tier available)

pelargir:
  credentials:
    privateKey: "..."                  # Server BabyJubJub private key for EdDSA signing
  polygon:
    rpcURL: "http://127.0.0.1:8545"   # Local Hardhat node
    contractAddress: "0x..."           # Auto-updated by make hard-local
    chainID: 31337
  batcher:
    maxBatchSize: 5                    # Transactions per Merkle batch
  circuitDir: "C:\\...\\gondor-backend\\circuit"
  heartbeat:
    cronIntervalSeconds: 30
    updateThresholdSeconds: 3073600
```

---

## Makefile Command Reference

| Command | Action |
|---|---|
| `make docker` | Start MySQL and Redis via Docker Compose |
| `make docker-down` | Stop Docker Compose services |
| `make docker-down-h` | Stop and remove persistent MySQL/Redis volumes (hard reset) |
| `make hard-local` | Full automated setup: Hardhat node + contract deploy + DB reset + backend start |
| `make run-local` | Start the Go backend with `APP_ENV=local` |
| `make simulate` | Run the trade simulator for full pipeline load testing |
| `make generate` | Regenerate mock files and pull latest Solidity ABI bindings |
| `make build-circuit` | Compile the ZK circuit (circom), generate proof artifacts, update on-chain Verifier.sol |
| `make test` | Run Go integration tests (cache disabled) |
| `make acceptance` | Run Testcontainers-based E2E acceptance tests |
| `make check` | Run integration + acceptance tests sequentially |
| `make clean` | Clean test cache, generated mocks, and coverage outputs |

---

## Verifying the Full Pipeline

After running `make simulate`, you can verify the complete flow worked by:

1. **Checking MySQL** — The `transaction` table should contain signed receipt hashes for each simulated trade.
2. **Checking Polygon (local Hardhat)** — Query `canonicalRoots` on the deployed contract to confirm Merkle roots were anchored.
3. **Checking reputation levels** — Call `reputationLevels(tokenId)` on the contract — levels should have updated from 1 to a new value after ZK proof verification.

Or simply run `make check` to validate the full test suite.
