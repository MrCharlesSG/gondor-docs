# Blockchain Interaction Flow

End-to-end lifecycle of an investor in the Gondor Ecosystem across four sequential blockchain interaction flows.

---

## System Actors

| Actor | Role |
|---|---|
| **Gondor App (Frontend)** | Mobile app operating on two parallel channels: REST calls to backend for trading, RPC calls to Polygon for reputation reading |
| **Minas Tirith (Web2 Backend)** | Financial engine in Go — authentication, MySQL, portfolios, Redis-cached price APIs |
| **Pelargir (Web3 Coprocessor)** | Go module for intensive cryptography — signs receipts, builds Merkle trees, generates ZK-SNARK proofs, manages blockchain write transactions |
| **GondorSovereignIdentity (Smart Contract)** | Polygon contract — stores SBTs, manages Merkle root registry, acts as mathematical judge verifying Groth16 proofs before updating investor reputation |

---

## Flow 1: Onboarding and Identity (SBT Creation)

The goal is to assign a decentralized identity without exposing the user to Web3 friction. The flow requires atomic transactions to ensure the Web2 registration and blockchain identity are always in sync.

```mermaid
flowchart TD
    subgraph App [1. Frontend: Gondor App]
        Start([Start]) --> SendReg(Send Registration via REST)
        RecvSuccess(Receive 201 Created) --> End([Authenticated User])
        RecvFailure(Receive 500/503) --> EndFail([Warning: Retry])
    end

    subgraph MT [2. Web2: Minas Tirith & MySQL]
        SendReg --> StartTx{BEGIN MySQL Tx}
        StartTx --> InsertTemp(Insert Temporary User)
        InsertTemp --> RequestId(Request Identity from Pelargir)

        EvalResp{Blockchain Success?}
        EvalResp -->|Yes| CommitTx(Save TokenID and COMMIT)
        EvalResp -->|No| RollbackTx(MySQL ROLLBACK)

        CommitTx --> RecvSuccess
        RollbackTx --> RecvFailure
    end

    subgraph Pelargir [3. Web3: Pelargir Crypto]
        RequestId --> DeriveDID(Derive Keys and DID)
        DeriveDID -.->|Async RPC Call| PolygonMint(safeMint Request)

        RecvMint(Receives Tx Hash and TokenID) --> EvalResp
        FailMint(Gas or Network Failure) --> EvalResp
    end

    subgraph Polygon [4. Blockchain: Polygon Amoy]
        PolygonMint -->|Processing Block| ContractMint(Smart Contract: safeMint)
        ContractMint -.->|On-chain Confirmation| RecvMint
        ContractMint -.->|Revert / Timeout| FailMint
    end
```

---

## Flow 2: Trading and Data Sealing (Merkle Batching)

The user begins investing. Trade data must be sealed at execution time to prevent any retroactive manipulation by the platform. Asynchronous background processes are used to avoid blocking the user experience during batch assembly.

```mermaid
flowchart TD
    subgraph App [1. Frontend: Gondor App]
        Buy(Send Buy/Sell Order) --> RecvReceipt(Receive HTTP 200 + JSON Receipt)
    end

    subgraph MT [2. Web2: Minas Tirith]
        Buy --> ExecBuy(Execute and Persist Trade)
        ExecBuy --> RequestSig(Request Receipt from Pelargir)
        SendReceipt(Attach Receipt to HTTP Response) --> RecvReceipt
    end

    subgraph Pelargir [3. Web3: Pelargir Crypto]
        RequestSig --> CreateJSON(Generate Deterministic JSON)
        CreateJSON --> Sign(Sign with Server PrivKey EdDSA/Poseidon)

        Sign --> SendReceipt
        Sign -.->|Async Background Enqueue| Queue(Hash Queue Buffer)

        Queue --> Check{Batch of N transactions full?}
        Check -->|No| Wait(Wait for more trades)
        Check -->|Yes| CalcRoot(Build Poseidon Merkle Tree)

        CalcRoot -.->|Async RPC Call| Anchor(Call anchorRoot)
    end

    subgraph Polygon [4. Blockchain: Polygon Amoy]
        Anchor --> ContractAnchor(Store Public Merkle Root in Contract)
    end
```

---

## Flow 3: Reputation Update and Heartbeat (ZK Proofs)

This flow updates and certifies the investor's true ROI level on-chain while preserving privacy. As a secondary effect, it refreshes the on-chain Heartbeat to prevent loss concealment through inactivity. Triggered automatically by Pelargir when a transaction batch is complete (either naturally or forced by the HeartbeatCron).

```mermaid
flowchart TD
    subgraph Pelargir [1. Web3: Pelargir - Batch Saver & ZK]
        Saver((Trigger: Batch Complete)) --> GetTx(Fetch recent history from DB)
        GetTx --> CalcROI(Compute New Costs and Revenue)
        CalcROI --> StartCircuit(Launch SnarkJS Circuit)

        StartCircuit --> GatherInputs(Load Private and Public Inputs)

        GatherInputs --> ValidMerkle{Constraints: Validate Leaves vs Root}
        ValidMerkle -->|Hash OK| CalcZkROI(Simulate Trades in Fixed-Point)

        CalcZkROI --> ValidROI{Constraint: ZK_ROI == Local_ROI?}
        ValidROI -->|OK| GenProof(Generate ZK-SNARK Proof)

        GenProof -.->|Async RPC Call| UpdatePol(updateReputation Request)
    end

    subgraph Polygon [2. Blockchain: Polygon Smart Contract]
        UpdatePol --> VerifyGroth(Invoke on-chain Groth16 Verifier)
        VerifyGroth --> IsValid{Proof Valid?}

        IsValid -->|Yes| UpdateSBT(Update Reputation Levels and ROI)
        UpdateSBT --> UpdateHeartbeat(Refresh lastHeartbeat on SBT)

        IsValid -->|No| Revert(Transaction Reverted - Fraud Detected)
    end
```

---

## Flow 4: Transparent Consumption (Truth on the Frontend)

The user opens the app to view their public reputation. To guarantee Minas Tirith cannot censor or misrepresent on-chain data, the device splits requests across two independent channels.

```mermaid
flowchart TD
    subgraph App [Frontend: Gondor App]
        UserOpen(User Opens Dashboard) --> DoubleRequest[Generate Parallel Requests]

        DoubleRequest --> AskWeb2(REST Call to Gondor API)
        DoubleRequest --> AskWeb3(RPC Call to Polygon Node)

        ShowWeb2(Render Private Balance in USD and Holdings)
        ShowWeb3(Render Immutable On-chain Reputation and Level)
    end

    subgraph MT [Web2: Minas Tirith]
        AskWeb2 --> QueryMySQL(Read MySQL / Redis Backend)
        QueryMySQL --> ShowWeb2
    end

    subgraph Polygon [Blockchain: Polygon Amoy]
        AskWeb3 -.->|viem getLevel() direct contract call| ReadContract(Free View State Read)
        ReadContract -.->|Returns Uint and Heartbeat Date| ShowWeb3
    end
```

---

## Complete Sequential Lifecycle

Full end-to-end interaction sequence from registration to reputation display.

```mermaid
sequenceDiagram
    autonumber
    actor User as User (Gondor App)
    participant MT as Minas Tirith (Go Backend)
    participant Pel as Pelargir (Crypto Coprocessor)
    participant Polygon as Polygon (Smart Contract)

    Note over User, Polygon: Flow 1: Onboarding and Identity
    User->>MT: POST /register
    MT->>Pel: Request Identity
    Pel->>Polygon: RPC safeMint()
    Polygon-->>Pel: Emit SBT, return TokenID
    Pel-->>MT: Return TokenID
    MT-->>User: Registration OK (SBT Assigned)

    Note over User, Polygon: Flow 2: Trading and Merkle Sealing
    User->>MT: POST /buy AAPL
    MT->>Pel: Notify trade to Crypto layer
    Pel-->>MT: Return signed JSON Receipt
    MT-->>User: Trade OK + Immutable Receipt
    Note over Pel, Polygon: Worker accumulates N transactions...
    Pel->>Polygon: RPC anchorMerkleRoot(Root)

    Note over User, Polygon: Flow 3: Reputation Update and Heartbeat
    Note right of Pel: Saver detects batch complete.<br/>Pelargir calculates new ROI locally.
    Note right of Pel: Runs SnarkJS circuit.<br/>Validates history vs Merkle Root.<br/>Computes ZK-Proof.
    Pel->>Polygon: RPC updateReputation(TokenID, ROI, ZK-Proof)
    Note over Polygon: Groth16 Verifier evaluates proof.
    Polygon->>Polygon: Update Reputation and Heartbeat on SBT

    Note over User, Polygon: Flow 4: Transparent Consumption
    User->>MT: GET /portfolio (Private Data)
    MT-->>User: JSON Balance in USD and Holdings
    User->>Polygon: Direct RPC (viem): getLevel(TokenID)
    Polygon-->>User: Immutable On-chain Level
```
