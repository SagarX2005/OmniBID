# OmniBid Protocol

**AI-assisted tender evaluation with private bid commitments, deterministic scoring and verifiable records on MST.**

> **Project status:** This README is the agreed project plan and documentation baseline. Implementation, integrations and deployment status must be confirmed against the repository and real test results. No contract address or transaction hash is published here until it has actually been deployed and verified.

## 1. What is OmniBid?

OmniBid is a Web2 + Web3 tender-evaluation platform for companies submitting bids for business or public-procurement projects.

The goal is to make bid submission more private before the deadline, make document review more consistent, show evidence for evaluation findings, and make the published result tamper-evident.

Its operating principle is:

**AI reads and explains. Deterministic Python validates and calculates. MST records and enforces the on-chain rules. A human authority makes the final procurement decision.**

OmniBid ranks valid bids and shortlists the top `N` bidders (default `N = 3`). It does **not** automatically appoint a winner.

## 2. Main features

- Company registration for tender authorities and bidders, with approval and role checks.
- SARAL as the intended simple onboarding path, subject to verification of the current SDK and wallet-signing capabilities.
- An option in the dashboard to connect an existing supported wallet, including BridgeKey where compatible.
- Tender creation with published rules, deadlines, bond amount and shortlist size.
- Commit-reveal bids that bind the tender, bidder wallet, price, salt and document fingerprint.
- First-stage ZIP hashing in the browser without uploading the ZIP to OmniBid's backend.
- Second-stage upload after the submission deadline, independent fingerprint verification and temporary processing.
- Three evaluation agents for document/eligibility review, financial/BOQ review and technical/compliance review.
- Deterministic Python validation, scoring and ranking; AI does not decide scores or rank bids.
- An evidence JSON document whose SHA-256 hash is published with the authorized result on MST.
- Authority-reviewed bidder feedback and auditable bond refunds/forfeiture under published rules.

## 3. End-to-end workflow

1. **Register:** A user signs in through the supported SARAL flow or connects an existing supported wallet. The user registers a company and requests the appropriate bidder/authority role.
2. **Approve:** The company is reviewed. Verified wallet addresses are associated with the company and authorized in the application and, where required, the smart contract.
3. **Create tender:** An approved authority enters tender details, eligibility rules, deadlines, bond amount, scoring rubric and `topN`. The full rules are kept in PostgreSQL; a canonical rules hash and essential tender parameters are recorded on MST.
4. **First ZIP selection:** The bidder selects the exact ZIP file in the browser. The browser calculates SHA-256 locally. The ZIP itself is **not** sent to the backend.
5. **Commit:** The bidder enters the price, the browser generates a cryptographically secure salt and computes a commitment binding the tender ID, bidder wallet, price, salt and document hash. PostgreSQL stores the restricted document hash and submission metadata. The bidder signs an MST transaction to submit the commitment and bond. The bidder downloads and securely keeps a receipt.
6. **Deadline passes:** New commitments close according to the smart contract's deadline.
7. **Second upload and reveal:** The bidder uploads the exact original ZIP. The backend independently calculates its SHA-256, compares it with the first-stage hash and rejects mismatches. After verification, the bidder reveals the original values on-chain; the contract checks the commitment. The backend verifies the reveal before evaluation.
8. **Evaluate temporarily:** The backend safely extracts the verified ZIP into a temporary processing area. Three agents produce evidence-backed findings. Python validates them, performs arithmetic, applies the published rubric and ranks valid bids.
9. **Review and publish:** The authority reviews the results and feedback drafts. The backend creates canonical evidence JSON and its SHA-256 hash. The authority publishes the ranked shortlist and evidence hash on MST.
10. **Clean up and settle:** Original ZIP and extracted files are deleted after processing. Evaluation evidence, scores, metadata and audit records remain. Revealing bidders can claim their bonds under the contract rules; non-reveal forfeitures follow the disclosed policy.

## 4. Data and privacy model

### During the first submission

| Data | Where it goes |
|---|---|
| Original ZIP bytes | Remain on the bidder's computer; never uploaded in first-stage submission |
| SHA-256 document hash | Stored in PostgreSQL with restricted access |
| Bid commitment | Recorded by the MST smart contract; a copy/reference may be stored in PostgreSQL |
| Unrevealed price and salt | Kept in the bidder's private submission receipt; not stored in PostgreSQL |
| Bond | Held by the smart contract |

The receipt contains sensitive reveal information. The bidder must keep it and the exact original ZIP safe. Browser storage is not a dependable long-term backup.

### During second upload and evaluation

The backend receives the ZIP only during reveal. It hashes the received bytes independently, validates the file and reveal, then processes the file temporarily. After evaluation, the original ZIP and extracted files are deleted; stale temporary files must also be cleaned up after crashes.

### Data retained after evaluation

PostgreSQL retains appropriate tender/submission metadata, revealed results, validated findings, score breakdowns, rankings, feedback-release decisions, evidence references and audit events. The evidence JSON is held in controlled application storage or the database. Its fingerprint is recorded on-chain.

**A hash is not encryption and does not prove that a document is truthful.** The actual document must be checked by the evaluation process. Do not put confidential bid documents, company KYC details, seed phrases or unrevealed prices on-chain.

## 5. Architecture

```text
Next.js + TypeScript frontend
   |-- SARAL onboarding (capabilities to verify)
   |-- Connect existing supported wallet (BridgeKey where compatible)
   |-- Browser-side first ZIP SHA-256 + commitment generation
   |
   +---- PostgreSQL through FastAPI ---- user/company/tender metadata
   |                                  
   +---- Wallet-signed MST transactions
                                         |
                                  OmniBid.sol
                          tender state, commitments, bonds,
                          reveals, result hash, refunds
                                         ^
                                         |
FastAPI backend -------------------------+
   |-- validate identity and roles
   |-- verify on-chain receipts/state
   |-- second-stage upload hash check
   |-- temporary safe ZIP/PDF/XLSX processing
   |-- three evaluation agents
   |-- deterministic Python scoring/ranking
   |-- canonical evidence JSON + SHA-256
   +-- human-reviewed feedback and audit trail
```

### Technology stack (target)

| Area | Technology |
|---|---|
| Frontend | Next.js, TypeScript, Tailwind CSS; shadcn/ui if useful |
| Onboarding | SARAL official integration, after SDK verification |
| External wallet | BridgeKey and other tested compatible wallet providers |
| Smart contract | Solidity |
| Contract development | Hardhat, OpenZeppelin components where appropriate |
| Backend | Python, FastAPI, Pydantic |
| Database | PostgreSQL, SQLAlchemy/Alembic if adopted |
| Document processing | PyMuPDF and openpyxl, plus safe ZIP handling |
| AI | A configured LLM API for extraction/explanation only |
| Scoring | Deterministic Python |
| Hashes | SHA-256 for ZIP/evidence fingerprints; Keccak-256 for commitment |

This table is the target architecture, not a claim that every dependency is installed or every integration is working. Select compatible versions after reviewing the repository and test environment.

## 6. MST's role

MST is an integral part of OmniBid, not just a wallet connection. The smart contract is intended to record tender identifiers and rules hashes, enforce submission/reveal deadlines, record bid commitments, hold bonds, verify revealed commitments, publish ranked bidder addresses and the evidence hash, and support bond refunds/forfeiture.

The contract does not independently verify private PDFs, AI findings or the off-chain scoring formula. The backend is an off-chain trust component; the system makes post-publication changes detectable, but a blockchain hash by itself cannot prove the result is substantively correct.

### Network configuration

The MST Buildathon guide provided these settings:

| Network | RPC URL | Chain ID | Symbol | Explorer |
|---|---|---:|---|---|
| MST Mainnet | `https://mariorpc.mstblockchain.com/` | `4646` | `MSTC` | <https://mstscan.com> |
| MST Testnet (guide values) | `https://testnetrpc.mstblockchain.com` | `91562037` | `tMSTC` | <https://testnet.mstscan.com> |

**Network warning:** The original Testnet explorer currently displays a notice that MST Chain Testnet is scheduled to shut down after **31 January 2027** and directs developers to MST Coral Testnet. Confirm with buildathon organizers which Testnet they accept for judging before deploying. Keep RPC URL and chain ID configurable; do not guess Coral settings or hard-code a network until verified.

Useful official references:

- MST developer docs: <https://docs.mstblockchain.com>
- MST website: <https://www.mstblockchain.com/>
- MST mainnet explorer: <https://mstscan.com>
- Original Testnet explorer/notice: <https://testnet.mstscan.com>
- Faucet listed in the guide: <https://faucet.mstblockchain.com>
- BridgeKey: <https://bridgekey.io>

### Deployment status

| Required item | Value |
|---|---|
| Accepted Testnet | **Confirm with organizers** |
| Testnet contract address | Not yet recorded — add only after real deployment |
| Required Testnet transaction hash | Not yet recorded — add only after a real transaction |
| Mainnet contract address | Not yet recorded — add only after real deployment |
| Frontend demo URL | Not yet recorded |
| Backend URL | Not yet recorded |

Never use sample/fake deployment details in the final submission. The buildathon requires real, verifiable MST transaction activity, a Mainnet contract address, a verifiable Testnet transaction hash and a public source repository. Check the latest official guide for the final submission rules.

## 7. Smart-contract responsibilities

The MVP is designed around one principal contract, provisionally `OmniBid.sol`. The candidate functions include:

- Role administration for multiple approved tender authorities and bidders.
- `createTender(...)` to record rules hash, deadlines, bond and `topN`.
- `submitBid(tenderId, commitment)` to record a commitment and hold the required bond.
- `revealBid(tenderId, price, salt, documentHash)` to check the original commitment and record revealed information.
- `publishResult(tenderId, rankedBidders, evidenceHash)` to record the approved shortlist and evidence fingerprint.
- `claimRefund(tenderId)` for each successful revealer's refundable bond.
- Non-reveal settlement to route forfeitures to a fixed treasury, not the tender lister.
- A no-result recovery path after `publishDeadline` so the authority cannot lock the process indefinitely.

The exact ABI must be finalized, implemented and tested before deployment. Critical safeguards include checks-effects-interactions, reentrancy protection for transfers, unique bidder/ranking checks, deadline checks, exact bond checks and prevention of duplicate refunds. The authority must not bid on its own tender address. A fixed treasury is the MVP default for non-reveal forfeitures.

## 8. Commitment and evidence rules

The intended bid commitment is:

```text
keccak256(abi.encode(tenderId, bidderWallet, bidPrice, salt, documentHash))
```

Expected Solidity types are `uint256`, `address`, `uint256`, `bytes32`, `bytes32`. The frontend and contract must use identical encoding and be covered by parity tests. Do not mix `abi.encode` with packed encoding.

The document fingerprint is SHA-256 of the exact raw ZIP bytes. It is kept in restricted PostgreSQL metadata before reveal and becomes publicly inspectable when included in a successful on-chain reveal.

The evidence JSON must use a documented, versioned canonical serialization before SHA-256. Test that the downloadable evidence reproduces the hash stored on-chain.

## 9. Scoring and feedback policy

- The tender's eligibility and scoring rules are published and frozen before bidding opens.
- Mandatory gates are separate from numerical scores and from `needs_review` cases.
- Python performs arithmetic and scoring; an LLM never assigns the final score or rank.
- Each finding and score deduction must have a reason and evidence reference where applicable.
- Price anomaly handling must be disclosed in advance; do not secretly change the formula after bids arrive.
- The authority reviews and may approve, edit or withhold feedback drafts. Feedback is only visible to the relevant bidder after release.
- A mismatch flag is not automatically a fraud finding or bond penalty.

Exact score weights and price formula are tender-specific and have not been finalized as universal constants in this README.

## 10. Security commitments

- No ZIP upload during the first-stage commitment.
- No unrevealed price or salt in PostgreSQL.
- No user private keys or seed phrases stored by OmniBid.
- Backend independently verifies the reveal upload; do not trust frontend-only checks.
- Temporary files have size/resource limits, safe extraction and cleanup after success, failure and restart.
- Company role checks are enforced server-side; relevant blockchain roles are enforced by the contract.
- Model outputs and evidence quotes are validated before scoring.
- Bond refunds do not depend on result publication; non-reveal forfeiture policy is disclosed.
- No fake transaction data, deployment addresses or results in source code, documentation or demo.

This is a proposed security design, not a claim that the implementation has passed an independent audit.

## 11. Scope

### MVP scope

- Company registration and approval.
- SARAL onboarding, once verified, and optional supported external wallet connection.
- Tender creation and discovery.
- One main smart contract for tender state, commitments, reveals, bonds and results.
- First-stage browser-only ZIP hashing and commitment.
- Second-stage upload, independent verification and temporary processing.
- Three evaluation agents and deterministic Python scoring.
- Evidence JSON with SHA-256 hash recorded on-chain.
- Human-reviewed bidder feedback.
- Bond refunds and documented non-reveal forfeiture.
- Testnet proof and Mainnet deployment as required by the buildathon.

### Out of scope unless explicitly approved

- ChainPay/UPI bond funding.
- VRF random draws.
- Advanced CAD drawing analysis.
- QCBS or extra scoring models outside the published rubric.
- Appeals, slashing or extra penalties.
- Non-refundable tender processing fees.
- Full IPFS document storage.
- Implementing all 25 possible tender checks.
- Automatically appointing a single winner.
- Permanent storage of the original ZIP.

## 12. Development checkpoints

| Checkpoint | Goal | Exit condition |
|---|---|---|
| CP1 | Requirements, repository audit and architecture docs | Facts, decisions, unknowns and scope are documented without confusing planned and implemented work. |
| CP2 | Verify MST network, SARAL SDK and wallet APIs | Accepted network is confirmed; minimal contract/wallet flow is verified or blockers are documented. |
| CP3 | Set up development environments | Frontend, backend and contract toolchain run with tested setup instructions. |
| CP4 | Database and backend APIs | Schema/migrations and foundational API tests pass. |
| CP5 | Smart contract | Contract implements roles, tender state, commitment, reveal, results, bond and recovery rules. |
| CP6 | Local contract testing | Success and security/failure tests pass. |
| CP7 | MST Testnet deployment | Real contract deployment and representative transactions are verifiable. |
| CP8 | Company registration and wallets | SARAL and/or external wallet path is tested; roles are enforced. |
| CP9 | Tender management | Tender can be created and its rules hash independently checked. |
| CP10 | Initial commitment | ZIP stays in browser; document hash and commitment are recorded correctly. |
| CP11 | Reveal upload | Backend hash check, on-chain reveal verification, evaluation handoff and cleanup work. |
| CP12 | Agents and scoring | Three structured findings pipelines and deterministic scoring pass tests. |
| CP13 | Results and bonds | Ranking matches evidence and chain; feedback and bond flows work. |
| CP14 | End-to-end/security review | Complete workflow and important failure cases pass. |
| CP15 | Mainnet and submission | Mainnet address, verifiable Testnet tx, working demo and public repo are documented. |

## 13. Repository organization

The actual repository may already differ; preserve existing working code and adapt this target structure rather than reorganizing blindly.

```text
omnibid/
├── contracts/          # Solidity, Hardhat config, deployment scripts and contract tests
├── frontend/           # Next.js application, components and wallet/client utilities
├── backend/            # FastAPI, schemas, database models, services, agents and scoring
├── docs/               # Architecture, decisions, checkpoints, threat model and demo notes
├── agent.md            # Instructions for AI coding assistants
├── README.md           # Public project overview, setup and truthful deployment details
├── .env.example        # Names of required environment variables only; no real secrets
└── .gitignore
```

## 14. Development workflow

1. Read `agent.md` and the relevant checkpoint document.
2. Inspect the repository and current Git changes.
3. State the checkpoint, intended changes and tests before implementing.
4. Make the smallest change that satisfies the requirement.
5. Run applicable formatting, type checks, unit tests and security checks.
6. Report exact files changed, commands run, results and unresolved issues.
7. Update documentation and checkpoint status only after verifying acceptance criteria.
8. Do not automatically proceed to the next checkpoint.

## 15. Recommended reading

- `agent.md` — detailed product, security and coding-agent rules.
- `docs/CP1_REQUIREMENTS_AND_ARCHITECTURE.md` — repository audit and architecture, once created.
- `docs/CP1_DECISIONS.md` — confirmed, proposed and unresolved decisions, once created.
- `docs/PROJECT_CHECKPOINTS.md` — checkpoint progress, once created.

If some of these files have not been created yet, treat them as planned documentation rather than existing artifacts.
