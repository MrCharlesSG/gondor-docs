# ADR-004: Soulbound Token on Polygon Amoy

**Status:** Accepted

---

## Context

The investor's identity must be:
1. **Non-transferable** — reputation cannot be bought or sold.
2. **Publicly verifiable** — any third party can check an investor's reputation level without trusting the platform.
3. **Low-cost** — the system must be viable for a large number of users.
4. **Frictionless for the end user** — no MetaMask, no seed phrases, no manual wallet management.

The standard NFT (ERC-721) allows transfers, so it was unsuitable on its own. The concept of a **Soulbound Token (SBT)** — an ERC-721 with transfers permanently blocked — was proposed by Vitalik Buterin (2022) as a primitive for non-transferable identity.

Blockchain options evaluated:
- **Ethereum mainnet**: ~$5–10 per mint at 30 Gwei. Prohibitive for user registration.
- **Polygon PoS (Amoy testnet)**: ~$0.003 per mint. Fast block confirmation (~2s). EVM-compatible.
- **Arbitrum/Optimism**: Similar cost profile to Polygon but less mature ZK tooling ecosystem at time of development.

---

## Decision

Implement the `GondorSovereignIdentity` contract as an **ERC-721 Soulbound Token** on **Polygon Amoy** (Chain ID 80002):

- Extends OpenZeppelin v5 `ERC721` and `Ownable`.
- Overrides `_update()` to revert any transfer not initiated by the contract owner (Pelargir backend), making it soulbound.
- Stores reputation state on-chain: `reputationLevels`, `reputationLevelsRealized`, `lastHeartbeat`, `identityStates`, `canonicalRoots`.
- The backend (Pelargir) generates the user's Ethereum wallet transparently using `go-ethereum/crypto`, assumes gas costs, and mints the SBT during user registration — completely invisible to the user.

---

## Consequences

**Positive:**
- Total cost of a complete user lifecycle (registration + 5 trades + first ZK proof) is ~$0.0085 on Polygon mainnet.
- The `sovereignHandover` function allows migrating custody from the backend-managed wallet to a user's personal wallet, achieving true sovereignty.
- Contract source code is verified on PolygonScan (Amoy), enabling any auditor to inspect the on-chain logic.

**Negative / Trade-offs:**
- **Testnet dependency**: The current deployment is on Amoy testnet. Mainnet deployment requires additional gas cost analysis and a production-grade key management solution for the backend-held wallets.
- **Custodial by default**: Until a user exercises `sovereignHandover`, their SBT wallet is managed by the Pelargir backend. This is a deliberate UX trade-off — custody can always be transferred, but zero friction at registration is prioritized.
