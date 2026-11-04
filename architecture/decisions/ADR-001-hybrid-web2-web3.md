# ADR-001: Hybrid Web2/Web3 Architecture

**Status:** Accepted

---

## Context

The core problem being solved — certifying financial reputation — requires two conflicting properties simultaneously:

1. **Performance**: Trade execution, portfolio valuation, and real-time price serving must operate with sub-millisecond latency for a viable user experience.
2. **Immutability**: The track record must be mathematically verifiable by any third party without trusting the platform operator.

A fully on-chain approach (all data stored on a smart contract) fails on performance — writing each trade to a public blockchain costs gas, takes 1–2 seconds, and exposes private transaction data publicly. A fully off-chain approach (traditional Web2 database) fails on immutability — any administrator with MySQL access can alter historical trades.

---

## Decision

Build a **hybrid Web2/Web3** system where:

- **Data management** lives in traditional Web2 infrastructure (Go, MySQL, Redis) for operational performance.
- **Truth certification** lives on a public blockchain (Polygon) for immutability and autonomous verifiability.

The two worlds are connected by a cryptographic coprocessor (**Pelargir**) that:
1. Creates unforgeable fingerprints of each trade at execution time (EdDSA/Poseidon signed receipts).
2. Periodically anchors batched fingerprints on-chain (Merkle root anchoring).
3. Generates Zero-Knowledge proofs that allow the blockchain to verify ROI without seeing trade details.

---

## Consequences

**Positive:**
- Operational latency remains in the microsecond range (Redis) for normal user interactions.
- Any third party can independently verify the integrity of the full historical track record by comparing Merkle roots.
- Private trade details (symbols, amounts, timing) never appear on the public blockchain.
- The system is cost-efficient: gas is only consumed for batch anchoring and ZK proof verification, not for every individual trade.

**Negative / Trade-offs:**
- The system requires trusting the Pelargir server to sign receipts honestly at trade execution time. This is mitigated by the fact that signed receipts are returned to users immediately and can be independently verified.
- On-chain reputation reflects only **executed transactions**, not the current mark-to-market value of open positions. This limitation is noted as future work (Chainlink oracle integration).
- Operational complexity is higher than a pure Web2 or pure Web3 solution.
