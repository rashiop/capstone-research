# R4 — Receiving Side: Invoices, Escrow, Clearance
  
## 1. Decisions & requirements served
- **Decisions:** (DP1) invoice data model; (DP2) how a payment is linked to an invoice; (DP3) escrow + clearance state machine; (DP4) compliance/verification mechanism; (DP5) payer UX; (DP6) handling unexpected payments.
- **Serves:** F2 (receiving rules), F3 (hold → clear → release), F9 (deadlines), and the graded **pull-over-push** + **oracle** items.
- This is the project's **main differentiator** (R1: custodians' policy engines focus on outgoing transfers).

### Terms
- **Payment reference:** a short ID derived from the invoice and included with the payment so it can be matched automatically.
- **Unapplied cash:** money received that can't be matched to an invoice (R5 §9.1).
- **Clearance:** the checks after money arrives and before the treasury accepts it.
- **Attestation:** a signed on-chain statement by a known party (e.g., "Company X passed KYB").
- **KYB (Know Your Business):** the company version of KYC. Checks that a business is real, who owns it, and that it isn't sanctioned.
- **Verdict:** the result of a check written back on-chain: `CLEAR` or `FLAG` (+ reason code), for one escrow entry.
- **AR / ERP:** Accounts Receivable, the finance team's record of who owes us money, usually inside an ERP system (SAP, NetSuite, etc.). (AP, Accounts Payable, is the *payer's* side.)
- **CRE:** Chainlink Runtime Environment (see §9.2).

---

## 2. Key findings

### 2.1 Prior art
| Source | What it does | Take-away |
|---|---|---|
| **Request Network** | The payer derives a **paymentReference** from the invoice; pays via a proxy contract that **emits an event** `(token, to, amount, paymentReference)`; a subgraph indexes events to compute the invoice balance; invoice content on IPFS. Supports partial payments and multiple chains. **V** | Reuse the **payment-reference + event + indexer** pattern. But Request only *detects* payments, it doesn't *enforce* rules on arrival. Our escrow adds enforcement |
| **ERC-3009 (USDC)** | Signed transfers. **`receiveWithAuthorization`** requires the **caller to be the payee**, which prevents front-running when a *contract* pulls the funds; random 32-byte nonces; `validAfter`/`validBefore` windows. **V** | Best "pay invoice with one signature" UX for USDC |
| **Chainalysis Sanctions Oracle** | `isSanctioned(address) → bool` (US, EU, UN lists); free. **Mainnets only** (Ethereum, Base, Arbitrum, OP, Polygon, …); no testnet listed. **V** | Code against its interface; use a **mock with the same interface** on Sepolia |
| **Ethereum Attestation Service (EAS)** | On-chain attestations against schemas; deployed on **Sepolia** and **Base Sepolia** (explorer + contracts). **V** | "Counterparty passed KYB" attestations from a trusted attester, checked on arrival |
| **Chainlink Functions** | ⚠️ **Sunset: June 30, 2026 (testnet: June 15, 2026)**. Replaced by the **Chainlink Runtime Environment (CRE)**. **V** | **Don't build on Functions.** M13 may still teach it; the concepts transfer to CRE |
| **Chainlink CRE** | Workflows (TypeScript/Go → WebAssembly) with triggers (cron, HTTP, EVM log) → HTTP fetch → signed **report** → `KeystoneForwarder` → our contract's **`onReport`** (via `ReceiverTemplate`). Local simulation with `cre workflow simulate`. Security: validate the forwarder (mandatory) + optionally workflow ID/owner; embed chain selector + timestamp against replay. **V** | The async "off-chain check → on-chain verdict" path. Supported testnet list: check at build time **U** |
 
---
 
## 3. Decision points
 
### DP1 — Invoice data model (moderately one-way)
 
| | A. On-chain InvoiceRegistry (hash + key fields) | B. Off-chain only + payment reference (Request-style) | C. Invoice as NFT (ERC-721 receivable) |
|---|---|---|---|
| What's on-chain | `invoiceId → {issuer, payer (chain,addr) or open, token, amount, dueDate, docHash, status, paid}` | Nothing but payment events | Same as A, as a transferable token |
| Pros | Contract can **enforce** match/amount/due date on arrival; simple to test | Cheapest; private invoice data | Enables factoring/financing later (sell the receivable); nice portfolio item |
| Cons | Key fields are public (amount, payer) | No on-chain enforcement → no escrow logic | Extra complexity; transfer semantics (who gets paid if the NFT moves) |
| D7 score | **4.50** | 3.51 | 3.94 |
 
**Pick: A.**
- The full invoice document (PDF/JSON) lives **off-chain** in the backend (D6); only `docHash` (keccak of the document) goes on-chain for integrity.
- **Privacy option:** store only `docHash` + amount + token; keep the payer's identity off-chain when the invoice is "open to any allowlisted payer".
- **C (receivable NFT / factoring) → roadmap.**
**Invoice status:** `Open → PartiallyPaid → Paid` | `Overpaid` | `Expired` (past due + grace) | `Cancelled` (by the issuer, only if unpaid).
 
### DP2 — Linking a payment to its invoice
 
| Method | How | When | MVP |
|---|---|---|---|
| **`payInvoice(invoiceId, amount)`** on the hub | Payer approves + calls, or uses a permit / 3009 signature (DP5) | Same-chain payers with a normal wallet | **Must** |
| **Cross-chain `INVOICE_PAYMENT`** | SpokeGateway sends token + `invoiceId` via CCIP (atomic, R5) | Payer on Base Sepolia | **Must** |
| **Payment reference in events** | Every accepted payment emits `PaymentReceived(invoiceId, paymentRef, payer, chain, token, amount)`; the indexer builds the history (Request-style) | Always | **Must** |
| **Per-invoice deposit address** (CREATE2) | Each invoice gets a deterministic address; a **plain token transfer** to it is enough; a sweeper forwards funds + invoiceId to the escrow | Payers whose custodian only allows **simple transfers** (common for institutions, e.g., through Fireblocks/Coinbase address books) | Should |
 
### DP3 — Escrow + clearance state machine (pull-over-push)
 
```
                 payment arrives (same-chain or CCIP)  — receiver NEVER reverts on business rules
                                   │
                                   ▼
                              [PENDING]
                                   │  auto-checks (sync): invoice exists & open, token matches,
                                   │  amount within range, payer allowed (invoice-bound or allowlisted
                                   │  for its chain), sanctions hook OK, not expired
                 ┌─────────────────┼───────────────────────────┐
         all pass│      async check needed (CRE/EAS pending)   │ any fail / unknown
                 ▼                 ▼                           ▼
            [CLEARED]      [AWAITING_VERDICT] ──verdict──► [FLAGGED] ◄── timeout (no verdict by T)
                 │                 │ ok                         │
                 │                 └──────────► [CLEARED] ◄─────┤ approver clears (maker-checker, EIP-712)
                 │                                              │
                 ▼                                              ▼
   Safe pulls funds: settle(entryId)                      [REJECTED] ── approver rejects
   → [SETTLED], invoice.paid += amount                          │
                                                                ▼
                                             refund entitlement recorded for the original payer
                                             same-chain: payer calls claimRefund()  → [REFUNDED]
                                             cross-chain: approver triggers REFUND via CCIP → [REFUNDED]
```
 
**Rules**
- **Pull, not push:** the escrow never sends funds out automatically. The **Safe pulls** cleared funds (`settle`), and **payers pull** refunds (`claimRefund`). Cross-chain refunds are the exception: a CCIP message must be *sent*, which is triggered by an approver, and the fee policy is decided in R6.
- **Partial payments:** accumulate until `amount`; each partial payment is its own escrow entry.
- **Overpayment:** the exact amount is cleared; the **excess becomes a refund entitlement** (never silently kept).
- **Deadlines (F9):** `FLAGGED` entries past the review deadline trigger backend reminders. `REJECTED` refunds have a claim window, after which the admin can re-route them per policy (and must document it).
- **Who can clear/reject:** `APPROVER` role, maker-checker (the approver ≠ the invoice issuer), via the same EIP-712 approval flow as R3 DP4.
- **Reentrancy:** `settle` / `claimRefund` use checks-effects-interactions + `ReentrancyGuard` + `SafeERC20`.
### DP4 — Compliance / verification mechanism
All options plug into R3's **`IPolicyHook`** slot (staticcall, gas cap, deny-overrides), or report asynchronously into the escrow.
 
| Option | How | Sync/async | Testnet reality | MVP |
|---|---|---|---|---|
| **Sanctions check** | `isSanctioned(payer)` via the Chainalysis interface; **mock contract** with the same interface on Sepolia (admin can flag test addresses) | Sync | Real oracle is mainnet-only **V** | **Must** |
| **Counterparty attestation (EAS)** | Require a valid, unrevoked attestation "KYB-passed" from our trusted attester for the payer (or payer org) | Sync | EAS on Sepolia + Base Sepolia **V** | **Should** |
| **Off-chain verdict via Chainlink CRE** | A CRE workflow triggered by the `PaymentReceived` log → calls the backend/compliance API (e.g., "invoice confirmed in our AR/ERP system? payer KYC ok?") → writes the verdict via `onReport` → `CLEARED`/`FLAGGED` | Async | CRE testnet support: verify **U**; `cre workflow simulate` works locally **V** | **Could** (stretch). Strong showcase, Chainlink's current product |
| **Trusted backend "clearer" role** | The backend signs an EIP-712 verdict (like an approver) | Async | Always works | Fallback if CRE isn't available |
 
**Why this split:** sync checks keep the demo fast and testable; the async path shows a realistic bank-style "fraud/compliance review" without blocking the MVP.
 
### DP5 — Payer UX
 
| Payer situation | Flow | MVP |
|---|---|---|
| Same chain, any ERC-20 | `approve` + `payInvoice` (2 txs) | Must |
| Same chain, token with EIP-2612 permit | Sign permit → `payInvoiceWithPermit` (1 tx) | Should |
| **Same chain, USDC** | Sign **ERC-3009 `receiveWithAuthorization`** (payee = escrow contract, so it can't be front-run) → a relayer or the payer submits `payInvoiceWithAuthorization`. **Gasless for the payer** if our relayer submits (R6 fee modes) | Should (showcase) |
| Payer on Base Sepolia | `SpokeGateway.payInvoice(invoiceId)`; the payer pays the CCIP fee in native ETH; optional pre-check on the spoke | Must |
| Payer can only do plain transfers | Per-invoice CREATE2 deposit address (DP2) | Should |
 
### DP6 — Unexpected / unwanted payments
 
| Case | Handling |
|---|---|
| Payment for an unknown or cancelled invoice | Escrow entry `FLAGGED` (never lost, never auto-accepted) |
| Payer not allowed for this invoice/chain | `FLAGGED` → approver decides → usually `REJECTED` → refund |
| Wrong token | `FLAGGED`; refund in the same token |
| **Direct ERC-20 `transfer` to the escrow** (bypasses `payInvoice`) | Contract tracks `accounted[token]`; `recordUnmatched(token)` books `balanceOf − accounted` as an **UNMATCHED** entry (the on-chain "unapplied cash" queue) → approver assigns it to an invoice or rejects it |
| Unsupported / spam / dust tokens (incl. **address-poisoning** transfers) | Never auto-swept into the Safe; ignored unless explicitly assigned; UI hides them by default |
| Sanctioned payer | `FLAGGED` with reason "sanctions". Don't auto-refund: sending funds back to a sanctioned address can itself be a violation. Freeze pending legal review (documented policy). **L** (general compliance practice) |
 
---
 
## 4. Invariants for R9 (Echidna / Foundry)
1. **Conservation:** for each token, `balance(escrow) ≥ Σ(open entry amounts)`, and received = settled + refunded + open.
2. No funds leave the escrow except `settle` (to the Safe, only `CLEARED` entries) or a refund (only `REJECTED`/overpay entitlements, only to the original payer/chain).
3. An entry never goes `SETTLED` or `REFUNDED` twice (no double spend).
4. An invoice's `paid` never exceeds its `amount` (the excess always becomes a refund entitlement).
5. A payment for an unknown, expired or cancelled invoice never reaches `CLEARED` without an approver action.
6. The receiver never reverts on business-rule failures (fuzz: random invoice IDs, tokens and senders → always recorded).
7. A CRE / clearer verdict is accepted only from the configured forwarder + workflow ID, once per entry (replay-safe).
---
 
## 5. Recommendations (summary)
- **DP1:** on-chain InvoiceRegistry with `docHash`; documents off-chain. NFT receivables → roadmap.
- **DP2:** `payInvoice` + cross-chain `INVOICE_PAYMENT` + events for the indexer (Must); CREATE2 deposit addresses (Should).
- **DP3:** state machine above; pull-based settle/refund; partial + overpay handling; deadlines drive backend reminders.
- **DP4:** sanctions hook with a mock (Must) → EAS KYB attestation (Should) → **Chainlink CRE** verdict (Could/stretch). **Not Chainlink Functions (sunset).**
- **DP5:** approve+pay (Must); EIP-2612 permit and **USDC ERC-3009 `receiveWithAuthorization`** (Should).
- **DP6:** never lose, never auto-accept: FLAGGED / UNMATCHED queues; spam never swept; sanctioned funds frozen, not returned.
## 6. Important finding outside R4
**Chainlink Functions was sunset on June 30, 2026 (testnet June 15, 2026)**, replaced by **CRE**. **V**
- Affects: R3 (hook ideas), R5 §9.3 ("Functions" in the one-stack story → read as "CRE"), and the requirements analysis (M13 coverage).
- **Action:** M13 concepts (DON-executed off-chain computation) still apply, but build on **CRE**. Mention in the pitch: "uses Chainlink's current compliance/automation runtime (CRE)". Check whether the bootcamp assignment for M13 still expects Functions (ask the instructor).
## 7. Risks, unknowns, revisit triggers
- **CRE testnet availability (U):** fall back to the trusted backend clearer (EIP-712 verdict).
- **Public invoice fields:** amounts/payers visible on-chain → minimize fields (docHash-only mode).
- **Cross-chain refund fees:** who pays (treasury vs deducted) → R6.
- **Sanctions handling** is a legal policy question. The MVP freezes and flags; a real deployment needs counsel.
- **Gas:** escrow entries per payment; fine on the hub for demo volumes.
## 8. Status
Approved
 
## 9. Deeper notes
 
### 9.1 Attestations, KYB and who the "trusted attester" is
- **Analogy:** an attestation is a **digital stamp**. "I, Party X, confirm that address 0xABC belongs to Company Y, which passed KYB on 2026-09-01. Valid until 2027-09-01." It's signed by X and stored on-chain via **EAS**, so any contract can check it. X can also **revoke** it.
- **Who is trusted?** Whoever *our* institution chooses. The contract stores a list of accepted attester addresses (changing it is a loosening change → time-locked).
  - **MVP:** our own **compliance-officer key** (a role in our backend) issues "KYB-passed" attestations after reviewing the counterparty off-chain, like a bank's onboarding team approving a new client.
  - **Real world:** a specialist KYB/identity provider acts as attester. Example of the pattern: Coinbase Verifications publishes "verified account" attestations on Base via EAS. **L** (as far as known; verify before citing in the pitch)
- **What the check does:** "Is there a valid, unrevoked, unexpired attestation from an accepted attester, with schema *KYB-passed*, for this payer's (chain, address)?" Yes → pass; no → FLAGGED (not rejected), so a human can decide.
### 9.2 CRE and "verdict"
- **CRE (Chainlink Runtime Environment):** Chainlink's platform for running **your code on Chainlink's decentralized node network**.
  1. You write a small workflow (TypeScript/Go).
  2. Many independent nodes each run it: e.g., call our compliance API.
  3. They must **agree** on the answer, then **sign** it.
  4. The signed result (a "report") is delivered to our contract's `onReport` function through Chainlink's forwarder contract.
  - Think "serverless function whose output is verified by many independent computers", rather than trusting one server.
  - It replaces Chainlink Functions (sunset 2026-06-30).
- **Verdict:** the answer the workflow writes on-chain for one escrow entry: `CLEAR` or `FLAG` (+ reason code). The escrow moves `AWAITING_VERDICT → CLEARED` or `→ FLAGGED`.
- **AR/ERP check:** e.g., "does our accounting system (ERP) agree this invoice is real and still open?". (Earlier drafts said "AP system"; the correct term on our receiving side is **AR**, accounts receivable.)
- **Why not just our backend?** A single backend key is one point of trust (and one hack away from approving anything). CRE spreads that trust across many nodes. The **fallback** (backend signs an EIP-712 verdict) is simpler and fine for the MVP.
### 9.3 Why `receiveWithAuthorization` can't be front-run
**Setup:** the payer signs "move 1,000 USDC from me to the Escrow, nonce N". Our relayer submits `payInvoiceWithAuthorization(invoiceId, signature)`. That transaction sits in the public mempool before it's mined, so **anyone can see the signature**.
 
| | `transferWithAuthorization` | `receiveWithAuthorization` |
|---|---|---|
| Who may execute the signed transfer | **Anyone** | **Only the payee** (`msg.sender` must equal `to`, i.e., the Escrow contract) **V** |
| Attack | An attacker copies the signature and calls `USDC.transferWithAuthorization` **directly**, first. The USDC still lands in the Escrow, but **our `payInvoice` logic never runs**: no invoice link, no escrow entry, and our transaction then fails because the nonce is used | The attacker can't call it; only the Escrow can, and it only does so inside `payInvoice…`, which records the invoice link in the **same** transaction |
| Result | Not theft, but **griefing**: the payment becomes "unmatched cash" (manual work) and the payer's UX breaks | Payment and invoice record are always atomic |
 
That's why ERC-3009 itself recommends `receiveWithAuthorization` when a **contract** is the payee. **V**
 
### 9.4 Why flag/reject a payer we don't recognise? (We'd still have the money)
Having the money isn't the same as being allowed to keep it. Institutions treat unexplained incoming money as a **risk**, not a gain:
1. **Anti-money-laundering (AML):** criminals "clean" money by paying it into legitimate businesses (e.g., paying someone else's invoice). Accepting it can make the institution part of the laundering chain.
2. **Sanctions:** money from a sanctioned party is illegal to handle, even if we did nothing else.
3. **Tainted funds:** stolen crypto can later be traced, frozen or seized. The treasury could lose it anyway, plus its reputation.
4. **Mistaken payments must be returned:** if someone pays by mistake, the law in most places requires returning it ("unjust enrichment"). Silently keeping it creates a liability.
5. **Accounting:** revenue can't be booked without knowing *who* paid *for what*. Unknown money sits as a liability until resolved.
That's why banks also hold and investigate, or return, payments from unexpected senders.
**Our design doesn't auto-reject.** An unknown payer → **FLAGGED** → a human decides: clear it (e.g., after KYB), reject and refund, or freeze (sanctions).
 
### 9.5 What is Echidna?
- A **fuzzer** from Trail of Bits: a tool that attacks your contracts with **huge numbers of random call sequences** to try to break rules you declare.
- You write **properties**: functions that must always return `true` (e.g., `echidna_no_double_refund()`). Echidna calls your functions in random orders, with random values and random senders, thousands to millions of times. If any sequence makes a property false, it shows the exact steps.
- **Compared with Foundry:**
  - Foundry **fuzz tests** randomize inputs to **one** function call.
  - Foundry **invariant tests** and Echidna randomize **sequences** of calls.
  - Echidna is the dedicated, more powerful tool (property modes, corpus, shrinking). The guide names it explicitly, and M14 teaches it.
- We'll run the same invariants in Foundry (fast, daily) and Echidna (deeper, CI/nightly).
### 9.6 "Open entry amount" (invariant 1)
- An **escrow entry** is one received payment being tracked (§3 DP3).
- It's **open** while it's not finished, i.e., the escrow still owes that money to someone: `PENDING`, `AWAITING_VERDICT`, `FLAGGED`, `CLEARED` (not yet pulled by the Safe), `REJECTED` (not yet refunded), `UNMATCHED`.
- It's **closed** once `SETTLED` (the Safe took it) or `REFUNDED` (the payer got it back).
- **Invariant:** the escrow's token balance must always be **≥ the sum of open entries**, so it can always pay everyone it owes.
- **Example:** received 3 payments (100 + 50 + 20 = 170 USDC). 100 settled to the Safe, 20 refunded → open = 50 → the escrow must hold ≥ 50 USDC. If it ever held 40, money leaked, which is a bug.
## 10. Sources
- Request Network — how payment networks work: https://docs.request.network/advanced/protocol-overview/how-payment-networks-work
- Request Network — ERC20 fee proxy spec: https://github.com/RequestNetwork/requestNetwork/blob/master/packages/advanced-logic/specs/payment-network-erc20-fee-proxy-contract-0.1.0.md
- ERC-3009 (Transfer With Authorization): https://eips.ethereum.org/EIPS/eip-3009
- Circle — 4 ways to authorize USDC interactions: https://www.circle.com/blog/four-ways-to-authorize-usdc-smart-contract-interactions-with-circle-sdk
- Chainalysis sanctions oracle docs: https://go.chainalysis.com/chainalysis-oracle-docs.html
- EAS contracts (Sepolia deployment): https://github.com/ethereum-attestation-service/eas-contracts/blob/master/deployments/sepolia/EAS.json
- EAS explorer — Base Sepolia: https://base-sepolia.easscan.org/schemas
- Chainlink Functions getting started (sunset notice): https://docs.chain.link/chainlink-functions/getting-started
- Migrate from Chainlink Functions to CRE: https://docs.chain.link/cre/reference/clf-migration-ts
- CRE — building consumer contracts (onReport, forwarder): https://docs.chain.link/cre/guides/workflow/using-evm-client/onchain-write/building-consumer-contracts
