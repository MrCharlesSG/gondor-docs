# ADR-005: Privacy-Preserving ZK over Transparent On-Chain Anchoring

**Status:** Accepted

---

## Context

When designing the reputation certification layer, an alternative architecture was considered: **transparent anchoring**, where each transaction is recorded on-chain in full (symbol, amount, price, timestamp). This would be simpler to implement and easier to audit.

Three reasons led to rejecting it:

1. **Strategic alpha destruction**: Publishing the full detail of trades (asset, price, moment) enables parasitic replication and front-running — a structural failure documented in Spectral Finance and ARCx. An investor's competitive advantage lies precisely in *when* and *what* they trade. Making this public destroys it.

2. **Convergence with existing solutions**: A transparent architecture with price oracles would be functionally similar to existing Web3 platforms already analyzed in the State of the Art, without addressing the privacy gap that differentiates Gondor.

3. **Regulatory implications**: Permanently publishing detailed financial data on an immutable blockchain creates tensions with GDPR Article 17 (right to erasure). ZK-SNARK proofs reveal only derived aggregates (total revenue, total cost), not personal transaction records.

---

## Decision

Use **ZK-SNARKs (Groth16)** to prove derived financial aggregates to the blockchain **without revealing transaction details**. The public signals visible on-chain are:

- `tokenId` — the investor's SBT identifier
- `oldTotalRevenue`, `oldTotalCostInvested`, `oldTotalCostRealized` — previous state
- `newTotalRevenue`, `newTotalCostInvested`, `newTotalCostRealized` — new state after the batch
- `root[5]` — Merkle roots of the 5 transactions in the batch (already anchored on-chain independently)

The private inputs — individual symbols, units, prices, amounts, timestamps, EdDSA signatures, and Merkle proofs — are processed entirely off-chain by the Pelargir prover and never appear on the blockchain or in any public log.

---

## Consequences

**Positive:**
- Transaction details remain completely private. No amount of blockchain analysis can recover the investor's specific trades.
- Any third party can independently verify that the published ROI is mathematically consistent with the anchored Merkle roots, without seeing the underlying data.
- Regulatory compliance: only aggregate state transitions appear on-chain, not personal financial records.

**Negative / Trade-offs:**
- **Executed transactions only**: On-chain reputation reflects only closed trades (`txType=1` sells), not the current mark-to-market value of open positions. An investor holding a losing position without selling will not have it reflected in their on-chain reputation until it closes. The **Heartbeat mechanism** and **dual reputation metric** (`reputationLevels` vs `reputationLevelsRealized`) partially mitigate this.
- **Fixed batch size**: The current circuit processes exactly N=5 transactions per ZK proof. This is a consequence of the Groth16 static circuit compilation. Incremental proving (future work) would remove this constraint.
- **Proving latency**: ~14.5 seconds per proof on CPU. This is asynchronous and invisible to the user, but it means reputation updates have a lag proportional to how many batches have been completed.
