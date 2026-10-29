# OmniBid Protocol — AI Coding Assistant Guide

> **Read this file before inspecting or changing code.** This document describes the current agreed product design and the rules for implementing it. It is a specification, not proof that every feature already exists.

**Project:** OmniBid Protocol  
**Blockchain ecosystem:** MST  
**Document status:** Product and architecture baseline; integration details and implementation status must be verified against the actual repository.  
**Primary principle:** **AI reads and explains. Deterministic code validates and calculates. The blockchain records and enforces on-chain rules. Humans retain the final procurement decision.**

---

## 1. Your role

You are the coding assistant for OmniBid Protocol. Help build a secure, understandable, demonstrable Web2 + Web3 application for the MST Buildathon.

The project owner is learning blockchain and tender-evaluation concepts. Explain technical terms in plain language when they first appear. Code must be understandable enough for the owner to explain to judges.

### Working rules

1. **Inspect before editing.** Read the relevant source files, tests, configuration, current Git status and related documentation before changing anything.
2. **Work one checkpoint at a time.** Determine the current checkpoint from the user's instruction and `docs/PROJECT_CHECKPOINTS.md`, if it exists. Do not implement a later checkpoint without approval.
3. **Build the smallest complete working slice.** Prefer a narrow, tested feature over speculative abstractions or a large untested implementation.
4. **Preserve existing work.** Never overwrite user changes, reset branches, delete files, run destructive commands or reorganize the repository without explicit approval.
5. **Explain non-trivial code.** Add useful comments where they improve understanding and give a short explanation of important changes, trade-offs and commands.
6. **Test honestly.** Do not say tests, deployments, SDK integrations or transactions work unless you actually verified them. Report commands run and their real outcomes.
7. **Never fabricate external details.** Do not invent MST SDK methods, SARAL capabilities, wallet-provider APIs, RPC behavior, contract addresses, explorer links, transaction hashes or deployment results. Verify them through official documentation and real test calls.
8. **Protect secrets.** Never print, commit or expose private keys, seed phrases, recovery phrases, API secrets or production credentials. Keep secrets out of browser bundles and Git history.
9. **Do not silently expand scope.** Ask before large structural changes. If time is limited, explain options and trade-offs rather than quietly removing agreed requirements.
10. **Be accurate about security.** A hash proves that data matches a fingerprint; it does not prove the data is true. Blockchain records do not independently prove that an off-chain AI evaluation was correct.
11. **Keep the MVP explainable.** Avoid microservices, unnecessary frameworks, excessive smart contracts, speculative infrastructure and autonomous AI decision-making unless an approved requirement needs them.
12. **Keep documentation synchronized.** Mark incomplete features as planned or not verified. Update the README only with facts supported by the repository and actual tests.

### Source-of-truth order

When information conflicts, use this order:

1. The user's latest explicit decision.
2. This `agent.md` and any newer, explicitly approved decision record.
3. The current implementation and tests, which establish what is actually implemented—not necessarily what should be implemented.
4. The MST Buildathon guide and official, current provider documentation for external requirements and APIs.
5. Older notes and earlier drafts.

If an important conflict remains, document it and ask for a decision. Do not silently resolve material policy, financial or security questions.

---

## 2. What OmniBid is

OmniBid is a tender-evaluation platform for companies participating in business or public-procurement tenders. It combines a conventional web application with blockchain-enforced rules and publicly verifiable records.

It is designed to reduce the risk of premature bid disclosure, inconsistent document review, unexplained ranking changes and disputes about whether published evidence changed later.

OmniBid ranks valid bids under rules published before bidding opens and shortlists the top `N` bidders (default `N = 3`). **It does not automatically appoint a winner.** The tender authority remains responsible for the final decision.

### Core responsibilities

- **Web2 application:** registration, company profiles, tender details, document processing, evaluation, evidence, results and feedback.
- **Cryptography:** document fingerprints and secret bid commitments.
- **Smart contract:** tender lifecycle, approved wallet roles, commitment recording, reveal verification, bond accounting, result publication and refund/forfeiture rules.
- **AI agents:** extract, organize and explain evidence-backed findings.
- **Deterministic Python:** validate findings, perform arithmetic, apply published eligibility/scoring rules and rank bids.
- **Human review:** approve company registrations where required, inspect uncertain findings, review feedback and make the procurement decision.

---

## 3. Finalized product decisions

### 3.1 Web2 + Web3 design

OmniBid is intentionally a hybrid application, not a website that merely displays a wallet button.

- PostgreSQL holds structured application data and evaluation records.
- The FastAPI backend validates data, verifies blockchain state and coordinates document evaluation.
- The Solidity contract enforces on-chain state transitions, commitments and bond rules.
- User wallets sign their own blockchain transactions.
- MST transactions provide verifiable evidence that core tender actions occurred.

The database is not a substitute for contract verification. Whenever a security-sensitive state is asserted to be on-chain, the backend must verify the relevant contract state, event or transaction receipt. Database copies of chain state are operational records or caches, not the blockchain's source of truth.

### 3.2 Registration, SARAL and optional external wallets

The product must offer:

1. **SARAL onboarding** as the intended easy sign-in/onboarding path.
2. **An option in the dashboard to connect an existing supported wallet**, including BridgeKey where compatible.
3. Company registration for tender authorities and bidders.
4. Company review/approval and role assignment.
5. Association of verified wallet addresses with the correct user and company.

A company may have multiple members and may have more than one associated wallet. The data model must not assume that every company has exactly one user or exactly one wallet.

**Integration caveat:** Do not assume SARAL automatically creates an EVM wallet, exposes a compatible signer, or can sign every smart-contract transaction. During CP2, verify its current official SDK and supported authentication, key-management, account and transaction-signing flows. If SARAL provides a usable signer, integrate that documented path. If it supplies login but not an EVM signer, retain SARAL for login and require a supported external wallet for on-chain actions. Do not invent SARAL methods or claim the wallet flow works before it has been tested.

For existing wallets, verify control with the documented wallet-signature flow. Never ask the user to paste a private key or seed phrase into OmniBid.

The dashboard must clearly show the **active wallet** and network. A commitment is bound to the original bidder address; changing the active wallet must not silently change the identity of an existing bid. Use the same wallet for reveal unless an explicitly reviewed contract design supports delegated reveal.

Company approval is distinct from wallet creation or login. Company documents, GST/PAN details and private account information must never be written to the blockchain.

### 3.3 Authorization

Authorization must exist in both relevant layers:

- FastAPI checks the authenticated user, company membership, company approval and role before performing application actions.
- The contract uses an on-chain allowlist or equivalent authorization for actions such as tender creation and bid submission. Frontend-only checks are not security controls.

The contract should support multiple approved authorities and bidders, rather than a single hard-coded tender-lister wallet. An administrator/deployer can grant or revoke approved wallet roles. Do not put personal or company documents on-chain.

A contract allowlist does not stop one company from using an unrelated second wallet unless identity is tied to wallets by off-chain review. Explain this limitation; do not claim that the contract alone prevents Sybil identities.

### 3.4 Tender creation

An approved authority creates a tender with:

- Title and description.
- Published tender rules and eligibility requirements.
- Required document list.
- Bid submission deadline.
- Reveal deadline.
- Result-publication deadline (`publishDeadline`) to prevent an authority from locking the process indefinitely.
- Required bond amount.
- Shortlist size (`topN`, default 3).
- Scoring criteria and rule/version information.

The full tender configuration is stored in PostgreSQL. The backend produces a deterministic canonical representation of the rules and computes a SHA-256 `rulesHash`. The contract stores the rules hash and the essential on-chain parameters.

Once bidding opens, the published rules must not be silently changed. If cancellation or amendments are needed, design an explicit, auditable procedure that does not retroactively change the rules governing existing commitments.

### 3.5 First ZIP selection and bid commitment

**The original ZIP must not be uploaded to the OmniBid backend during the first submission.**

1. The bidder selects a ZIP file in the browser.
2. Browser code calculates SHA-256 over the exact raw ZIP bytes. This is `documentHash`.
3. The bidder enters a bid price.
4. The browser generates a cryptographically secure random 32-byte `salt`.
5. The browser computes the commitment described in Section 4.
6. The browser sends only the document hash and required non-secret submission metadata to FastAPI. PostgreSQL stores the hash under restricted access; the ZIP bytes do not leave the browser.
7. The bidder's active wallet submits the commitment with the required bond to the smart contract.
8. The user downloads and safely keeps a private submission receipt containing the reveal information.

Before reveal, PostgreSQL must not store the plaintext bid price or salt. Do not send either to logs, analytics, error reports or the backend during the initial commitment stage. Do not publish the document hash separately on-chain before reveal; only the commitment is submitted.

The browser must not rely on the file remaining in browser memory or storage until the deadline. The bidder is responsible for retaining the exact original ZIP and the receipt.

### 3.6 Reveal and second ZIP upload

After the submission deadline, the bidder must reselect the same original ZIP and provide the reveal data from their saved receipt.

Recommended sequence:

1. The file is uploaded to a private temporary-processing area during the reveal window.
2. The backend hashes the **received bytes** independently with SHA-256.
3. The backend compares that hash against the stored first-stage fingerprint and rejects a mismatch.
4. The frontend/backend verifies that the reveal inputs are consistent with the original commitment; the contract remains the final authority on commitment correctness.
5. Once the file check passes, the bidder signs the on-chain reveal transaction before the reveal deadline.
6. The application verifies the successful reveal transaction/contract state before evaluating the file.
7. The verified ZIP and extracted files are held only in temporary processing during evaluation.
8. On completion or terminal failure, temporary files are deleted; a stale-file cleanup process handles server crashes.

Do not evaluate a file before its hash has passed the backend check and the corresponding reveal is verified on-chain. Do not trust the browser-only hash comparison because the browser can be modified or bypassed.

For the MVP, require byte-for-byte equality with the original ZIP. Recreating a ZIP from the same folder can change the ZIP bytes and therefore the hash. The UI must clearly tell bidders to retain the exact original ZIP.

If hashes mismatch, reject that file for evaluation, delete its temporary copy, record an audit event and flag it for review. Notify the bidder and permit a retry while the reveal window remains open. A mismatch by itself is not proof of fraud and must not automatically ban a bidder or confiscate a bond. The published non-reveal rules govern the consequences if the bidder does not complete a valid reveal before the deadline.

### 3.7 Temporary processing and retention

The original ZIP is never stored during first submission. It is uploaded only after the submission deadline for reveal verification and temporary evaluation.

Use a private temporary directory or equivalent transient processing environment. Do not add permanent ZIP storage or public IPFS storage to the MVP. Cleanup must run in `finally`-style handling and through a separate stale-file sweeper because the process can crash before cleanup executes.

Apply upload-size, file-count, decompressed-size, processing-time and allowed-file-type limits. Safely extract archives without path traversal, symlink abuse or ZIP-bomb exposure. Treat filenames and file contents as untrusted.

After processing, retain findings, scores, ranking, evidence and audit metadata—not the original ZIP or extracted source files. Application logs must not contain confidential document contents, bid receipts, salts or private keys.

### 3.8 Three-agent evaluation

Use three clearly separated evaluation components. They can be Python modules orchestrated by FastAPI; a complex autonomous multi-agent framework is not required for the MVP.

**Agent 1 — Document and eligibility review**
- Check whether required documents are present.
- Extract relevant facts from company, registration and eligibility documents.
- Produce structured findings with document/page evidence.

**Agent 2 — Financial and BOQ review**
- Inspect financial requirements and bill-of-quantities data.
- Recalculate quantity × rate and totals using deterministic Python.
- Compare declared and calculated totals and flag discrepancies.
- Check defined financial thresholds and any bond/EMD requirements that are part of the published tender rules.

**Agent 3 — Technical and compliance review**
- Compare the proposal against published technical requirements.
- Identify missing sections, mismatches and cases needing human review.
- Produce evidence-backed findings rather than unsupported conclusions.

The names/precise scope can be implemented incrementally, but all three responsibilities must be represented in the MVP unless the project owner explicitly approves a scope change.

LLMs may extract and explain; they must never calculate final arithmetic, decide scores or directly decide the ranking. Validate all model outputs with Pydantic or equivalent schemas. Validate evidence quotes against extracted document text. If a quote cannot be found, the model output is invalid or `needs_review`. Store relevant model/provider/version and prompt/schema version metadata so an evaluation can be audited. Do not claim perfect reproducibility of LLM extraction.

Treat the uploaded documents as untrusted input. Instructions embedded in tender or bid documents must never override system/developer evaluation instructions or change the scoring rubric.

### 3.9 Deterministic eligibility, scoring and ranking

The backend validates findings and uses plain Python to apply the rules published with the tender.

Separate:

- **Mandatory eligibility gates:** published pass/fail criteria.
- **Scored criteria:** published numerical rubric.
- **Needs-review cases:** ambiguous, low-confidence or conflicting evidence that requires a documented human decision.

Each tender's rule version and scoring parameters must be fixed before bidding opens. Do not silently change a formula after seeing submitted prices.

No universal final scoring formula has been approved in this specification. The system must support a versioned, published rule configuration. A 100-point rubric with technical/financial/compliance categories can be configured as a tender template only after its exact definitions and weights are reviewed.

A price formula based on the lowest valid price can be manipulated by an implausibly low bid. If used, the tender rules must state any plausibility thresholds, price gates or human-review rules before opening. Do not secretly replace the formula with a median or other method after bids arrive. Price anomaly checks must be transparent and must not become an undisclosed penalty.

Rank valid bids and shortlist the top `topN` (default 3). The contract can verify that a published list consists of unique revealed bidder addresses and respects `topN`; it cannot independently verify off-chain AI findings or recalculate the complete scoring rubric. Make that boundary explicit in the UI and README.

### 3.10 Evidence bundle and result publication

Create an evidence JSON document containing the data necessary to explain and audit the evaluation, including:

- Tender ID and rules/scoring version.
- Commitment/reveal references.
- Validated findings and evidence references.
- Deterministic calculations.
- Score breakdowns and eligibility outcomes.
- Ranked list and shortlist.
- Relevant model/version/prompt/schema information.
- Review decisions and feedback-release events.

Define one canonical serialization procedure and version it. The exact serialized bytes must be reproducible before computing SHA-256. Prefer an established canonical JSON scheme or a small, explicitly tested project serializer; do not assume ordinary JSON serialization across languages always produces the same bytes.

Store the evidence file in controlled application storage or PostgreSQL if practical. Publish the result only after the authority has reviewed it. The authority signs the on-chain result-publication transaction containing the ordered bidder list and `evidenceHash`.

Before/after publication, the application must verify that the ranking shown to users matches the ranking in the evidence JSON and the on-chain result. A mismatched ranking/evidence pair must be flagged and not presented as verified.

A hash establishes that evidence matches a published fingerprint, not that the evidence is true or the evaluation correct. The backend is a trusted off-chain component; the system makes later tampering detectable, not impossible.

### 3.11 Human-reviewed feedback

Generate a short draft (normally 2–3 lines) for each bidder based only on validated results. It may include rank, score, principal criteria affecting the result, evidence/page references and gap to the top-N cutoff when meaningful.

- A human authority can approve, edit or withhold each draft.
- Feedback is not sent automatically.
- A bidder can see only their own released feedback.
- A disqualified bidder should see the failed published criterion and an explanation; a flag is not automatically a penalty.
- Log the release decision for audit.

An LLM may rewrite validated facts into plain language but must not invent reasons, numerical values or evidence references. If generated feedback fails validation, use a deterministic template.

### 3.12 Bonds, refunds and non-reveal forfeiture

The initial design uses the MST native coin as the bond asset, subject to verification on the selected network. Bond amount and transaction gas fee are separate. The bond is held by the contract; gas is paid for transactions and is not refunded as part of the bond.

MVP rules:

- A bidder who successfully reveals can claim the full bond back, regardless of score or eligibility outcome.
- A low score, disqualification or document mismatch flag alone must not forfeit the bond.
- A bidder who fails to complete a valid reveal before the published reveal deadline forfeits the bond according to the published contract rules.
- Forfeited bonds go to a fixed treasury address configured at deployment, not to the tender lister. The policy and recipient must be disclosed in the tender terms/README.
- Refund eligibility must not depend on the authority publishing the result.
- A `publishDeadline` and a documented no-result recovery path must prevent the authority from keeping a tender and its funds locked forever.
- Use pull-based refunds, checks-effects-interactions and reentrancy protection. Prevent double claims and unauthorized withdrawals.

If the deployed network/contract design requires a different token or treasury arrangement, document it and get approval before implementation. Do not make a financial policy change silently.

---

## 4. Cryptographic and on-chain requirements

### 4.1 Document hash

`documentHash = SHA-256(exact raw ZIP bytes)`.

- Browser: use the Web Crypto API where supported.
- Backend: recompute independently using a standard SHA-256 implementation.
- Treat the 32-byte output as `bytes32` when it enters Solidity, with exactly documented byte conversion.
- Store it privately in PostgreSQL before reveal. Do not emit it as a separate public value before reveal.
- On reveal, passing it to the contract makes it publicly inspectable on-chain.

A hash is a fingerprint, not encryption. A person who already has a suspected file can calculate its hash and compare it.

### 4.2 Commitment

Use a domain-bound commitment:

```text
commitment = keccak256(
  abi.encode(
    tenderId,
    bidderWallet,
    bidPrice,
    salt,
    documentHash
  )
)
```

The definitive Solidity types are expected to be `uint256`, `address`, `uint256`, `bytes32`, `bytes32`, respectively. If implementation changes them, update the frontend and cross-language tests together. `bidderWallet` in Solidity reveal logic must be derived from `msg.sender`, not trusted as a caller-supplied identity.

Do not use `solidityPackedKeccak256` for this commitment while the contract uses `abi.encode`. The frontend and contract must encode exactly the same fields in the same order. Add tests with known sample inputs to prove frontend/contract parity.

Include tender ID and bidder address to prevent cross-tender replay and commitment copying. The secret salt must be generated using a cryptographically secure random source and must remain unchanged until reveal.

### 4.3 Receipt and secret handling

The downloadable receipt should include the tender ID, bidder address, bid price, salt, document hash, commitment hash and transaction reference after confirmation.

- Never store the salt or unrevealed price in PostgreSQL.
- Do not rely on browser `localStorage` as the only backup. Persistent browser storage can be cleared and may expose secrets to an XSS vulnerability.
- Tell the bidder the receipt is sensitive and must be retained securely and not shared.
- If encrypted backups are added, design them separately and do not make them a prerequisite for the first working slice.
- A lost salt or original ZIP may make a valid reveal impossible; clearly warn the bidder before commitment.

### 4.4 Evidence hash

Canonicalize evidence JSON with a versioned procedure, then calculate SHA-256 over the exact bytes that are made available for audit. Test that the public/downloaded evidence can be re-hashed to obtain the on-chain value.

### 4.5 Avoid unsupported confidentiality claims

A commitment conceals bid details before reveal only under the assumptions of the commitment design and secret salt. The blockchain itself is public. Do not publish private bid documents, plaintext prices before reveal, secrets, company KYC data or full evaluation documents on-chain.

---

## 5. Smart contract specification

The MVP should use one main Solidity contract, provisionally `OmniBid.sol`, unless a reviewed technical requirement justifies another contract. Use a supported Solidity version and dependencies only after testnet compatibility is checked.

### 5.1 Candidate functions

| Function | Responsibility |
|---|---|
| Role-management functions | Allow the admin to approve/revoke authority and bidder wallet addresses. |
| `createTender(...)` | Store rules hash, deadlines, bond amount and `topN`; require approved authority and valid timing/parameters. Include `publishDeadline`. |
| `submitBid(tenderId, commitment)` payable | Require approved bidder, open submission window, exact bond and no duplicate submission. Reject the tender authority's own wallet. |
| `revealBid(tenderId, price, salt, documentHash)` | After submission closes and before reveal closes, recompute the domain-bound commitment, verify it and record the reveal. |
| `publishResult(tenderId, rankedBidders, evidenceHash)` | Require the proper authority and publication window; require unique revealed bidder addresses and list length within `topN`; store ranking and evidence hash. |
| `claimRefund(tenderId)` | Allow each successful revealer to pull the full refundable bond once, without depending on result publication. |
| Non-reveal settlement | After reveal deadline, route forfeitures according to the fixed-treasury policy without permitting the tender authority to profit from non-reveals. |
| No-result recovery | After `publishDeadline`, allow the tender to leave the publication-pending state without locking revealed bidders' refunds indefinitely. |

These are design requirements, not final ABI signatures. Final names, events, mappings and state machine must be validated with the tests before deployment.

### 5.2 State and deadline rules

The state model should make these conditions explicit:

- Tender created/open for commitments.
- Commitments closed and reveal window open.
- Reveal window closed.
- Result published or publication deadline missed.
- Settlement/forfeiture accounting finalized where needed.

On-chain `block.timestamp` is the source of truth for contract deadlines. Do not rely only on the browser or backend clock.

Ensure that:

- `bidDeadline < revealDeadline < publishDeadline`.
- Commit submission and reveal are only accepted inside their permitted windows.
- Publishing is permitted only after the reveal phase closes and before `publishDeadline`.
- The authority cannot indefinitely prevent refunds by refusing to publish.
- Each bid can only be committed/revealed/refunded/forfeited according to its allowed state.
- A wallet cannot bid on a tender it created. Explain that this does not prevent the same company from using a different wallet unless off-chain approval controls it.

### 5.3 Bond safety

Prefer an immutable/non-zero treasury address at deployment for the MVP. Never pay non-reveal forfeitures to the tender lister by default. Avoid a treasury-setting function unless its access controls and implications have been reviewed.

Follow checks-effects-interactions, use a reentrancy guard for external transfers where appropriate and mark claim state before transferring. Refunds must be individually claimable and idempotent. Avoid a design in which one failing recipient blocks others' refunds.

### 5.4 Events

At minimum, design clear events for tender creation, authority/bidder role changes, bid commitments, reveals, result publication, refunds, forfeitures and no-result recovery. Events should contain enough public identifiers to verify the workflow but must not contain unrevealed bid prices, salt or confidential document data.

### 5.5 Limits of on-chain validation

The contract can verify the commitment, deadlines, role allowlist, bond, uniqueness of bidders and basic structure of the published list. It cannot directly validate private PDF evidence, AI accuracy or an off-chain scoring calculation. The README and demo must describe this honestly.

---

## 6. Architecture and recommended stack

Use the existing repository and dependencies where appropriate. The following is the target stack, not a claim about what has already been installed.

| Layer | Target technology | Responsibility |
|---|---|---|
| Frontend | Next.js + TypeScript | Registration, dashboards, tenders, uploads, results and feedback. |
| UI | Tailwind CSS; shadcn/ui if useful | Accessible, responsive components. |
| Onboarding | SARAL official SDK/integration | Simplified onboarding, subject to verified capabilities. |
| Existing wallet | BridgeKey and compatible wallet providers | User-controlled contract signing. |
| Contracts | Solidity | On-chain tender lifecycle, commitment, bond and result rules. |
| Contract tooling | Hardhat | Compile, local tests and deployment. |
| Contract safety | OpenZeppelin components where appropriate | Standard access-control and reentrancy protections. |
| Backend | Python + FastAPI | APIs, file checks, orchestration, evidence and scoring. |
| Data access | SQLAlchemy + Alembic if adopted | Database models and migrations. |
| Validation | Pydantic | Request and agent-output validation. |
| Database | PostgreSQL | Persistent structured application records. |
| PDF parsing | PyMuPDF or an appropriate tested equivalent | Extract PDF text with page information. |
| Excel parsing | openpyxl | Read spreadsheet/BOQ data. |
| AI | A supported LLM API, configured outside source code | Extraction and explanation only. |
| Backend chain access | A tested EVM JSON-RPC library/SDK | Verify transactions and query contract state. |
| Frontend chain access | A tested provider integration and ethers.js if compatible | Encode commitments and request user signatures. |

Do not install all candidates blindly. Review the existing repository, choose compatible versions and verify the selected MST network/tooling during CP2.

### Component boundaries

**Frontend** handles UI, local first-stage ZIP hashing, commitment construction, wallet prompts and result presentation. It is untrusted and cannot be the final security validator.

**FastAPI backend** checks identity/roles, stores metadata, independently hashes second-stage uploads, verifies blockchain facts, safely processes temporary files, coordinates agents, performs scoring and builds evidence.

**PostgreSQL** stores users, companies, wallet associations, tender metadata, hashes, submission status, validated findings, results, feedback and audit events. Do not store original ZIPs, unrevealed plaintext prices or salts.

**Smart contract** stores public on-chain state, commitments, bonds, revealed information and published result fingerprints. Do not store private documents or company data.

**Temporary processing area** holds reveal uploads and extracted files only while they are being verified and evaluated; cleanup is mandatory.

---

## 7. Data/storage rules

### Before the first commitment

The application may store account/company profile data and the published tender configuration. There is no bid file yet.

### First ZIP selection / commitment

| Item | Location |
|---|---|
| Original ZIP bytes | Bidder's computer only; never sent to backend |
| SHA-256 document hash | PostgreSQL with restricted access |
| Commitment hash | MST contract after transaction; optional DB copy/reference |
| Bid price and salt | Private client-side submission receipt; not in PostgreSQL |
| Bond | Locked in the smart contract |
| Transaction reference | PostgreSQL after transaction plus public chain record |

### Second upload / evaluation

The ZIP is temporarily uploaded only after the submission deadline. The backend computes its hash independently, validates it, confirms the on-chain reveal and runs evaluation. The original ZIP and extracted files are deleted after processing, with stale-temp cleanup for crashes.

### After evaluation

Persist the evidence JSON or controlled reference, validated findings, scoring breakdown, ranking, feedback-release records, audit events and transaction references. Persist the revealed price only after successful reveal and protect it according to the product's access-control policy. The salt should not be retained in PostgreSQL unless a separately approved security design requires it.

### Candidate PostgreSQL entities

Use the minimum normalized data model needed for the existing code and APIs. Candidate entities:

- `users`
- `companies`
- `company_members`
- `wallets`
- `tenders`
- `bid_submissions`
- `evaluation_runs`
- `evaluation_findings`
- `bid_results`
- `feedback`
- `audit_events`

This is a logical list, not a requirement to create every table immediately. Review foreign keys, unique constraints, indexes, lifecycle fields and retention policy before migrations.

### Audit and operational records

Log status changes and transaction identifiers, not secrets or document contents. Keep an audit trail for company approval, wallet authorization, mismatches, evaluation runs, human review and feedback release. Store timestamps consistently in UTC and display them in the user's local timezone.

---

## 8. API design guidance

Inspect the existing API before adding routes. Use consistent naming, request/response schemas, authorization checks and clear error codes. The following are logical API areas, not a mandatory list of exact routes:

- Authentication/session and wallet verification.
- Company registration, membership, role and approval.
- Tender creation, discovery and details.
- Initial bid metadata registration (no ZIP body).
- Commitment transaction confirmation/state sync.
- Reveal-stage temporary ZIP upload and hash verification.
- Reveal transaction confirmation.
- Evaluation start/status/results.
- Evidence retrieval and evidence-hash verification.
- Authority-only feedback review/release.
- Bidder-only released feedback.
- Audit/health endpoints appropriate for development.

Security requirements:

- Every company-specific action verifies authenticated membership and role.
- A bidder sees only their own confidential submission and released feedback.
- A lister sees only the bids and evaluation data they are authorized to view under the published procurement workflow.
- Do not expose unrevealed price/salt in APIs.
- Enforce content-length, file-type and archive limits before expensive parsing.
- Use idempotency/duplicate handling around transactions and evaluation requests.
- Separate application login identity from the active wallet identity and verify their association.
- Treat on-chain transaction receipts and contract state as facts to verify, not data to trust because a frontend reports success.

---

## 9. MST networks and integration verification

The MST Buildathon guide provided these values for its stated network configuration:

| Network | RPC | Chain ID | Currency symbol | Explorer |
|---|---|---:|---|---|
| MST Mainnet | `https://mariorpc.mstblockchain.com/` | `4646` | `MSTC` | `https://mstscan.com` |
| MST Testnet (guide values) | `https://testnetrpc.mstblockchain.com` | `91562037` | `tMSTC` | `https://testnet.mstscan.com` |

The original Testnet explorer currently displays a deprecation notice saying that the MST Chain Testnet is scheduled to shut down after **31 January 2027** and directs developers to MST Coral Testnet. The buildathon guide's Testnet values and the current notice therefore need to be reconciled before deployment. **Confirm with the organizers which test network is accepted for judging and use configuration rather than hard-coded network assumptions.** Do not guess Coral RPC/chain ID from memory.

Official references to check during CP2:

- MST developer documentation: <https://docs.mstblockchain.com>
- MST website: <https://www.mstblockchain.com/> (also check the currently linked official site)
- Mainnet explorer: <https://mstscan.com>
- Original Testnet explorer/deprecation notice: <https://testnet.mstscan.com>
- Testnet faucet listed in the guide: <https://faucet.mstblockchain.com>
- SARAL/SDK documentation linked by the current official docs.
- BridgeKey official website and wallet/provider documentation.

MST describes its current stack as EVM-compatible and supports Solidity, Hardhat and MetaMask, but actual deployment, wallet provider and contract calls still must be tested on the accepted network. Before the full contract, deploy a minimal contract and test reads, writes, events and wallet signing.

**Never publish fabricated addresses or hashes.** Store actual deployment details in a deployment record and README only after verification.

---

## 10. Testing requirements

Testing should be staged and proportionate to each checkpoint.

### Smart-contract tests

At minimum test:

- Approved authority can create a valid tender.
- Unapproved authority cannot create one.
- Invalid deadlines/parameters are rejected.
- Approved bidder can submit within the window with the exact bond.
- Missing/incorrect bond, duplicate commitment and late submission are rejected.
- Tender authority cannot bid on its own tender.
- Reveal before/after invalid window is rejected.
- Correct commitment reveals; wrong price, salt, bidder, tender ID or document hash fails.
- Result cannot publish before reveal closes or after `publishDeadline`.
- Result publisher must be authorized tender authority.
- Ranked addresses are unique, revealed, and no more than `topN`.
- One bidder cannot claim a refund twice.
- Revealed bidders can claim the full refund regardless of score and without result publication.
- Non-reveal forfeiture cannot be captured by the tender lister.
- No-result recovery works.
- Reentrancy/failed transfer cases do not lock or duplicate funds.

### Frontend/backend commitment parity

Use fixed sample vectors to confirm the browser and Solidity produce the same commitment bytes/hash. Test a raw ZIP fixture so browser and backend SHA-256 agree exactly.

### File-processing tests

Test matching/mismatching files, corrupt ZIPs, path traversal, oversized archive, too many entries, decompression bombs, unsupported types and cleanup after exceptions/restarts.

### Evaluation tests

Test malformed LLM output, quote not found in source text, missing evidence, arithmetic edge cases, gate vs needs-review separation, reproducible scoring, tie behavior as published, ranking/evidence mismatch and feedback authorization.

### End-to-end test

Run: company registration/approval → tender creation → first-stage hash/commitment → deadline transition → second upload and verification → on-chain reveal → three agents → deterministic scoring → human review → result publication/evidence hash → refund/forfeiture.

Use real Testnet transactions for the required proof. Local testnet simulations do not count as real MST transactions.

---

## 11. Security/privacy checklist

- [ ] `.env` and secrets are ignored by Git.
- [ ] No private key/seed phrase enters frontend code, database or logs.
- [ ] No original ZIP is uploaded in first-stage submission.
- [ ] No pre-reveal price or salt is stored server-side.
- [ ] Document hash fields are access-controlled before reveal.
- [ ] Commitment binds tender ID and bidder address.
- [ ] Frontend and contract encoding parity is tested.
- [ ] The contract, not just the frontend, checks roles and deadlines.
- [ ] Treasury does not benefit the tender authority from non-reveals.
- [ ] A missing authority result cannot lock revealed bidders' refunds forever.
- [ ] Refunds use safe accounting and cannot be claimed twice.
- [ ] File extraction is path-safe and resource-limited.
- [ ] Temporary files are deleted after processing and stale temp cleanup exists.
- [ ] LLM text is treated as untrusted and all findings are validated.
- [ ] Scoring rules are fixed before bidding opens.
- [ ] Results published on-chain match the retained evidence JSON.
- [ ] README clearly describes off-chain trust assumptions and limitations.
- [ ] No fake address, transaction hash or test result appears in docs/demo.

---

## 12. Project checkpoints

Work sequentially. A checkpoint is complete only when its acceptance tests are demonstrated and documented. A generated file or written function is not, by itself, proof of completion.

| Checkpoint | Objective | Acceptance evidence |
|---|---|---|
| **CP1 — Requirements and repository foundation** | Audit the repository and reconcile the approved product design, documentation and scope. | Requirements/architecture, decisions and checkpoint docs exist; current implementation is distinguished from planned work. |
| **CP2 — MST, SARAL and wallet feasibility** | Verify accepted test network, SDK capabilities and wallet transaction flow. | Official docs reviewed; minimal contract/wallet proof tested or all blockers precisely documented; network selection confirmed with organizers. |
| **CP3 — Development environments** | Establish frontend, backend, contract toolchain and local configuration. | Each existing app starts/builds with documented commands; secrets are excluded. |
| **CP4 — Database and API foundation** | Build or align schema, migrations, identity, company and tender APIs. | Migrations and API tests pass. |
| **CP5 — Smart contract implementation** | Implement roles, tender lifecycle, commit/reveal, bonds, results and recovery. | Contract compiles; code and state machine reviewed. |
| **CP6 — Local contract tests** | Test success and failure cases, including money flows. | Automated tests pass and important security cases are covered. |
| **CP7 — MST Testnet deployment** | Deploy to the network accepted by the buildathon. | Real address and verifiable transaction records; end-to-end contract workflow tested on that network. |
| **CP8 — Registration and wallets** | Implement SARAL path and optional external wallet, company approval and wallet linking. | Supported sign-in/wallet flow tested; roles enforced in backend and contract. |
| **CP9 — Tender management** | Create/list tenders and freeze/hash published rules. | Test authority creates an on-chain tender; hash corresponds to canonical rules. |
| **CP10 — First ZIP submission** | Browser-only file fingerprint, submission receipt and commitment transaction. | No ZIP bytes reach backend during the first submission; on-chain commitment and DB metadata agree. |
| **CP11 — Reveal and temporary file verification** | Second upload, independent SHA-256 check, on-chain reveal and cleanup. | Valid file passes; mismatch is rejected/audited; cleanup verified. |
| **CP12 — Agents and scoring** | Structured evidence findings and deterministic evaluation. | Agents produce validated findings; Python scoring is tested and reproducible from stored findings/rules. |
| **CP13 — Results and settlement** | Human review, evidence hash publication, feedback and bond outcomes. | Published ranking matches evidence and chain; refund/forfeiture tests pass. |
| **CP14 — End-to-end/security review** | Verify the entire application and important abuse/failure cases. | End-to-end test and security checklist recorded. |
| **CP15 — Mainnet and submission** | Deploy frozen contract to Mainnet and prepare public submission. | Real Mainnet address, required verifiable Testnet transaction, working demo/link where applicable, public source and accurate README. |

Do not treat CP2, CP7 or CP15 as complete merely because network settings have been written down.

---

## 13. Out of scope for the initial MVP

Do not implement these unless explicitly approved:

- ChainPay/UPI bond funding.
- Verifiable-random-function (VRF) draws.
- Advanced CAD drawing interpretation.
- QCBS or additional scoring models beyond the approved published rules.
- Appeals, slashing or penalties beyond the documented bond/non-reveal policy.
- A non-refundable tender processing fee.
- Full IPFS storage of tender documents.
- All 25 evaluation elements from the broader concept brief.
- Automatic appointment of a single winning bidder.
- Permanent storage of the original bid ZIP.

SARAL onboarding and optional external wallet connection are part of the intended product design; they are **not** to be silently removed as an old draft proposed. Their precise capability and fallback must be resolved during CP2.

---

## 14. README and demo honesty requirements

The root README must distinguish `Implemented`, `In progress`, `Planned` and `Not verified` features. It must include MST integration details, network/contract data once real, setup instructions that have been tested, and a clear account of what is off-chain versus on-chain.

Never state that a contract is deployed until the real address and explorer record exist. Never fabricate a Testnet transaction hash, Mainnet address, live website or agent output. If a feature uses mock data or a local chain, label it.

When presenting to judges, explain that MST is integral because it records commitment hashes, enforces deadlines/roles/bonds and publishes ranking/evidence fingerprints. A wallet connection alone is not the core blockchain contribution.

---

## 15. Glossary

- **Tender:** A request for companies to submit offers for a job or procurement contract.
- **Bid:** A company's offer, including price and required proposal documents.
- **Tender authority/lister:** The approved company or organization that creates a tender and reviews results.
- **Bidder:** An approved company submitting an offer.
- **Wallet:** An account used to authorize blockchain transactions; it is not the same as a company registration.
- **Smart contract:** Code deployed to a blockchain that enforces defined rules.
- **Hash/fingerprint:** A fixed-size digest of data. It helps detect changes but is not encryption and does not prove truth.
- **SHA-256:** A hash algorithm used here to fingerprint ZIP bytes and the evidence JSON.
- **Keccak-256:** The hash used by Ethereum-compatible Solidity `keccak256` for the bid commitment.
- **Salt:** A random secret included in a commitment so the bid cannot be trivially guessed from the commitment.
- **Commit-reveal:** Submit a commitment first, then reveal the underlying values after the submission deadline.
- **Bond/EMD:** A deposit held under the published rules. It is separate from transaction gas fees.
- **BOQ:** Bill of quantities; a list of items, quantities, rates and totals.
- **Rubric:** The published criteria and formula used to score bids.
- **Eligibility gate:** A mandatory published pass/fail requirement.
- **Evidence bundle:** A structured JSON document of validated findings, calculations and result metadata.
- **`topN`:** The number of highest-ranked valid bids to shortlist, default 3.
- **Gas fee:** A transaction fee paid for blockchain execution. It is not the tender bond.

---

## 16. Before changing code: required assistant response

At the start of a new coding task, briefly report:

1. The checkpoint being worked on.
2. Relevant files inspected.
3. The smallest intended change.
4. The tests or verification to run.
5. Any blocker requiring a decision.

After the task, report files changed, commands/tests run, results, limitations and the next checkpoint. Do not auto-advance to the next checkpoint.
