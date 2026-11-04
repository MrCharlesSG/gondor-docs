# Global Architecture Diagram

Full system component interaction diagram for the Gondor Ecosystem.

```mermaid
graph TD
    subgraph Frontend [Presentation Layer]
        App[Mobile App<br>React Native / Expo]
        Portal[Audit Web Portal<br>React]
    end

    subgraph Backend [Gondor Go Modular Monolith]
        subgraph MT [Minas Tirith - Web2 Core]
            API[REST API<br>Gin-gonic]
            TS[TransactionService]
            SS[StockService]
            AS[AuthService]
            HS[HoldingService]
            PS[PortfolioService]
        end

        subgraph PEL [Pelargir - Web3 Coprocessor]
            RS[RegisterService]
            BATCH[TransactionBatcher]
            ZK[ReputationUpdater<br>ZK-SNARKs]
            CRON[HeartbeatCron]
        end

        API --> TS
        API --> AS
        API --> SS
        API --> PS

        TS -. "IdentityBridge" .-> RS
        TS -. "TransactionReceipt" .-> BATCH
        BATCH --> ZK
        CRON --> ZK
    end

    subgraph Persistence [Data Infrastructure]
        MySQL[(MySQL<br>Transactional ACID)]
        Redis[(Redis<br>High-Speed Cache µs)]
    end

    subgraph External [External Integrations]
        ExtAPI[Price Providers<br>Finnhub / Alpaca]
        Polygon[Polygon Amoy<br>Blockchain Network]
        SC{GondorSovereignIdentity<br>Smart Contract}
    end

    App -->|HTTP REST| API
    App -->|RPC viem| Polygon
    Portal -->|RPC viem| Polygon

    SS <--> Redis
    SS <--> ExtAPI

    TS <--> MySQL
    AS <--> MySQL
    BATCH <--> MySQL

    RS -->|safeMint SBT| SC
    BATCH -->|anchorRoot| SC
    ZK -->|updateReputationWithProof| SC

    Polygon --- SC
```

## Component Responsibility Summary

| Component | Module | Primary Responsibility |
|---|---|---|
| `REST API` | Minas Tirith | HTTP routing, JWT middleware, request/response mapping |
| `TransactionService` | Minas Tirith | Trade execution, ROI calculation, holding updates |
| `StockService` | Minas Tirith | Real-time stock price serving (Redis-first) |
| `AuthService` | Minas Tirith | User registration, login, JWT issuance |
| `HoldingService` | Minas Tirith | Aggregate position management per user/symbol |
| `PortfolioService` | Minas Tirith | Consolidated portfolio valuation |
| `RegisterService` | Pelargir | Wallet generation + SBT minting (IdentityBridge implementor) |
| `TransactionBatcher` | Pelargir | Signed receipt buffering, Merkle tree construction, `anchorRoot` |
| `ReputationUpdater` | Pelargir | ZK proof generation (SnarkJS CLI), `updateReputationWithProof` |
| `HeartbeatCron` | Pelargir | Forces batch completion with dummy transactions for inactive users |
| `MySQL` | Infrastructure | Persistent storage for users, transactions, holdings, batches |
| `Redis` | Infrastructure | In-memory cache for stock prices (µs read latency) |
| `GondorSovereignIdentity` | Blockchain | ERC-721 SBT, Merkle root registry, ZK proof verifier interface |
