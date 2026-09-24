# Architecture — Proof of Aid (Milestone Escrow & Multi-Verifier Evidence)

This document describes the complete system architecture for our **Proof of Aid** platform. It outlines the overall vision, actor roles, milestone lifecycle (including partial funding and refunds), multi-verifier mechanics, system components, onchain/offchain boundaries, and critical architectural trade-offs.

---

## Vision and Actors

### Vision
Our vision is to build a transparent, automated aid distribution mechanism that bridges donor intent with verifiable impact on the ground. By combining **Milestone-Based Escrow**, **Offchain Cryptographic Evidence Integrity**, and a **Multi-Verifier Consensus with Dispute Windows**, we ensure funds are only released when aid is genuinely delivered and independently validated.

### Actors
* **Donor:** Contributes crypto/stablecoins to specific milestone escrows (supports partial or full funding).
* **NGO (Recipient):** Creates aid projects/milestones, executes work upon funding, submits progress evidence (receipts, photos), and requests milestone releases.
* **Verifiers (Independent Auditing Entities):** Authorized third parties (institutional auditors, local observers, community representatives) who inspect offchain evidence and vote onchain.
* **Beneficiaries:** Final recipients of aid (represented anonymously/aggregate in this implementation phase).

---

## End-to-End Flow & Milestone State Lifecycle

### State Transition Diagram

```text
 [ Milestone Created ]
          │
          ▼
  ( Pending_Funding ) ◄─────── [ Donors Deposit Funds ]
          │                      (Partial deposits allowed: e.g., $500 / $1,000)
          ├─────────────────────────────────────────────┐
          │ (Goal reached within deadline)              │ (Deadline expires & Goal not met)
          ▼                                             ▼
      ( Funded )                                  ( Expired )
          │                                             │
          │ [ NGO Executes Work & Uploads Evidence ]    └─► [ Donors Trigger Reclaim / Refund ]
          ▼
 ( Evidence_Submitted )
          │
          │ [ Verifiers Inspect & Cast Approval Votes ]
          ▼
     ( Verified ) ── (Threshold Met: e.g., 2-of-3)
          │
          │ [ 48h Dispute Window Opens & Pass Without Claims ]
          ▼
     ( Released ) ──► [ Funds Transferred to NGO Wallet ]
```
## Verifier Framework (Roles, Integrity & Disputes)

### 1. Verifier Profiles
* **Institutional Auditors:** Professional NGOs/audit firms managing high-reputation credentials onchain.
* **Local Ground Observers:** Verified entities operating in the direct geographic location of aid delivery.
* **Community Delegates:** Designated donor representatives reviewing high-value releases.

### 2. Verification Protocol
* **Cryptographic Validation:** The verifier app calculates `SHA256(Downloaded_File)` and asserts it matches the registered `onchainHash`. If hashes mismatch, the submission is flagged as tampered.
* **Substantive Audit:** Verifiers validate invoice amounts, supplier legitimacy, and metadata alignment (GPS/timestamps).
* **Consensus Trigger:** Once $K$-of-$N$ authorized verifiers (e.g., 2-of-3) submit affirmative votes, the contract advances to `Verified`.

### 3. Anti-Collusion & Dispute Mechanism
* **48-Hour Dispute Window:** Prevents instant releases. During this window, any actor can freeze the release by submitting a dispute bond and evidence of fraud.
* **Slashing & Banning:** Verifiers engaged in malicious voting or collusion lose their registered status and staked incentives.

---

## Core Components

| Component | Responsibility | Technology / Approach | Rationale |
| :--- | :--- | :--- | :--- |
| **AidEscrow Contract** | Holds locked deposits per milestone, tracks donor balance contributions, calculates threshold voting, records cryptographic hashes, handles expiration refunds, and executes releases. | Solidity / OpenZeppelin | Complete programmatic trust without relying on intermediaries. |
| **Verifier Registry** | Manages whitelist and authorization of independent audit addresses. | Solidity (AccessControl) | Ensures only authorized auditors can influence milestone releases. |
| **Evidence Hasher** | Generates deterministic SHA-256 digests of offchain documents in the browser/client. | SHA-256 (Web Crypto API) | Guarantees tamper detection with zero file bloat onchain. |
| **Offchain Storage** | Stores raw delivery receipts, site photos, and logistics invoices. | Local Server / S3 / IPFS | Cost savings, high scalability, and compliance with data privacy. |
| **Indexer & Timeline** | Listens to onchain events (`Deposited`, `Funded`, `EvidenceSubmitted`, `Released`, `Refunded`) to provide real-time updates. | Node.js + PostgreSQL | Fast UI queries without spamming Web3 RPC endpoints. |
| **Web Dashboard** | Interface for Donors to fund/refund, NGOs to submit progress, and Verifiers to audit. | React / Next.js + viem/wagmi | Direct wallet interactions with intuitive state feedback. |

---

## Key Decisions, Trade-Offs, and Edge Cases

| Area | Decision & Design Choice | Trade-Off / Alternative |
| :--- | :--- | :--- |
| **Partial Funding** | Allow incremental contributions per milestone with deadline-based refund triggers. | **Trade-off:** Adds logic complexity to donor accounting.<br>**Alternative:** All-or-nothing single donor model (Limits accessibility for micro-donors). |
| **Evidence Storage** | Offchain file storage + Onchain cryptographic SHA-256 hash. | **Trade-off:** Requires offchain storage persistence.<br>**Alternative:** Onchain storage (Prohibitively expensive gas fees). |
| **Consensus Model** | Multi-verifier threshold ($k$-of-$n$) with a 48h dispute window. | **Trade-off:** Introduces release latency.<br>**Alternative:** Single-verifier auto-release (High risk of collusion or single point of failure). |
| **Refund Mechanism** | Pull-over-push refund strategy where donors call `reclaimFunds()` if milestone expires. | **Trade-off:** Requires donor action to recover funds.<br>**Alternative:** Automatic push refunds (Extremely unsafe due to unbounded loop gas limits). |
