# Product Requirements Document (PRD)

## 1. Summary
An **on-chain policy backstop for institutional stablecoin treasuries**, built on top of existing custody (Safe owned by custodian MPC keys). It enforces **outgoing** spending rules, **clears incoming** invoice payments through an escrow, applies **one rulebook across chains** (hub-and-spoke via Chainlink CCIP), and lets **AI agents propose** payments without being able to spend.

## 2. Problem
- Custodian policy engines are **off-chain**: they can be bypassed (e.g., recovery paths) and hide rules from counterparties and auditors.
- **Incoming** checks stop at screening: custodians like Fireblocks screen and auto-freeze suspicious inbound funds, but nothing ties a payment to an invoice, holds it in escrow or refunds it by rule, so unmatched payments ("unapplied cash") and sanctions exposure stay manual.
- Treasuries span **multiple chains**. Custodian caps can span chains but only see what the custodian signs; on-chain wallet rules (e.g., Safe guards) work one chain at a time; and bridges are a top attack target (KelpDAO $292M, 2026).
- Multisigs check **who** signed, not **what** was signed (Bybit $1.5B, 2025).
- Teams want AI agents in finance ops but can't safely give them spending power.

## 3. Goals & non-goals

**Goals (MVP)**
1. Enforce USD-denominated outgoing limits, allowlists, tiered approvals and time-locked rule changes **on-chain**, on a Safe.
2. Receive invoice payments into an **escrow** with automatic checks, human clearance, settlement and refunds.
3. Pay out and receive **across two chains** (Sepolia hub ↔ Base Sepolia spoke) with rules decided on the hub.
4. Offer **gasless** operations for staff and payers (relayer).
5. Expose a **propose-only MCP server** for AI agents.
6. Meet every capstone grading item (security patterns, oracles, full test suite incl. Echidna, multichain deployment, docs).

**Non-goals (MVP)**
- Replacing custodians or MPC; real custodian API integrations.
- Mainnet production use; handling real customer funds.
- AI invoice reconciliation, privacy L2s, receivables financing, more than 2 chains, spoke float.

## 4. Personas

| Persona | Needs | Maps to role |
|---|---|---|
| **Treasury Operator** (finance ops) | Pay approved vendors quickly within limits | OPERATOR |
| **Approver / Controller** | Review and sign larger payments; clear flagged receipts | APPROVER |
| **Treasury Owner / Custodian signer** | Final authority for large moves; govern policy | Safe owners (POLICY_ADMIN via Safe) |
| **Risk / Security Officer** | Freeze fast; veto risky rule changes | GUARDIAN |
| **Payer (customer)** | Pay an invoice easily, from either chain, ideally gasless | External |
| **AI Agent** (via MCP) | Read state, simulate and propose payments | AGENT |
| **Auditor / Instructor** | Verify rules and history publicly | Read-only |

## 5. User stories & acceptance criteria (by epic)

### E1 — Policy configuration & governance
| ID | Story | Acceptance criteria |
|---|---|---|
| E1.1 | As an Owner, I set per-role limits (auto, tier-2, per-tx), buckets and the hard cap | Values stored on-chain; events emitted; UI shows current values |
| E1.2 | As an Owner, loosening changes wait for a time-lock | Loosening is queued with an ETA; executes only after the ETA; the UI shows a countdown |
| E1.3 | As a Guardian, I can veto a queued loosening change | A cancelled change never takes effect; event emitted |
| E1.4 | Tightening changes apply instantly | Lowering a limit or removing a payee takes effect in the same transaction |
| E1.5 | As a Guardian, I can pause the engine or a lane instantly | While paused, only the minimal owner path + recovery works; unpause needs the admin quorum |
| E1.6 | Payees are added with an activation delay | A payment to a payee before `activatesAt` is DENIED |

### E2 — Outgoing payments
| ID | Story | Acceptance criteria |
|---|---|---|
| E2.1 | As an Operator, I pay an allowlisted vendor ≤ my auto limit without extra approval | Executes via PaymentModule; bucket decreases by the USD value (rounded up) |
| E2.2 | Above the auto limit, K-of-N approvers must sign the exact intent | Execution fails without K distinct, valid, unexpired, unused signatures; the proposer can't approve |
| E2.3 | Above tier-2, only owners can execute | Module path DENIES; an owner-signed Safe tx succeeds ≤ hard cap |
| E2.4 | Nothing exceeds the hard cap or buckets | DENY for any path; the UI explains why |
| E2.5 | Dangerous call types are blocked | `delegatecall` to non-allowlisted targets, unknown contract calls, unlimited `approve`, and self-calls outside the admin path are DENIED |
| E2.6 | Stale/invalid prices never auto-approve | Payment falls to NEEDS_APPROVAL; UI shows "price unavailable" |
| E2.7 | Pay a vendor on Base from the hub treasury | `ccipSend` is policy-checked (payee allowlisted for Base, fee counted); vendor receives funds on Base; UI shows Sent → In transit → Delivered |

### E3 — Incoming payments & clearance
| ID | Story | Acceptance criteria |
|---|---|---|
| E3.1 | As an Operator, I create an invoice and share a pay link | Invoice stored on-chain (key fields + `docHash`); the document stored off-chain; public pay page works |
| E3.2 | As a Payer, I pay an invoice on the hub | Escrow entry created; the receiver never reverts on business-rule failures |
| E3.3 | As a Payer on Base, I pay cross-chain | Token + invoiceId arrive atomically on the hub; same checks as E3.2 |
| E3.4 | Auto-checks clear good payments | Exact match + allowed payer + sanctions OK → CLEARED; otherwise FLAGGED with a reason |
| E3.5 | As an Approver, I clear or reject flagged entries | Maker-checker enforced; state changes + events |
| E3.6 | Cleared funds are pulled into the Safe | `settle` moves only CLEARED entries; SETTLED once |
| E3.7 | Rejected/overpaid funds are refundable | Same chain: payer `claimRefund`; cross-chain: approver-triggered REFUND; never refunded twice |
| E3.8 | Direct transfers and spam are handled | Untracked balance recorded as UNMATCHED; unsupported tokens never swept |
| E3.9 | Sanctioned payers are frozen, not refunded | Entry FLAGGED "sanctions"; no automatic refund |
| E3.10 | Reminders for overdue invoices and pending reviews | Jobs send notifications on due date / review deadline |

### E4 — Gasless UX
| ID | Story | Acceptance criteria |
|---|---|---|
| E4.1 | Staff never need ETH to propose/approve | Proposals and approvals are EIP-712 signatures; the relayer submits executions |
| E4.2 | Relayer can't be abused | Only role-signed payloads accepted; simulated before sending; rate limits; gas budget alerts |
| E4.3 | (Should) USDC payers pay with one signature | `receiveWithAuthorization` flow works; front-running doesn't break invoice linking |
| E4.4 | (Should) 4337 sponsored ops still pass the guard | Spike S3 passes; otherwise documented as roadmap |

### E5 — AI agent (MCP)
| ID | Story | Acceptance criteria |
|---|---|---|
| E5.1 | As an Agent, I can read policy, balances, payees, invoices | MCP tools return data; no secrets exposed |
| E5.2 | As an Agent, I can simulate a payment | Returns ALLOW / NEEDS_APPROVAL / DENY + reason via `eth_call` |
| E5.3 | As an Agent, I can propose but never execute | Proposals land in the queue with NEEDS_APPROVAL; no tool can approve/execute/change policy; AGENT auto limit = $0 |
| E5.4 | Agents can't add payees or spam | Payee must pre-exist; rate limit enforced |

### E6 — Visibility & audit
| ID | Story | Acceptance criteria |
|---|---|---|
| E6.1 | Dashboard of balances (USD), buckets, pending items | Loads from chain + subgraph; refreshes on events |
| E6.2 | Full history of payments, approvals, escrow and policy changes | Subgraph-backed; each item links to the explorer / CCIP explorer |
| E6.3 | Export an audit log | CSV/JSON export from the API |

### E7 — Engineering & grading
| ID | Story | Acceptance criteria |
|---|---|---|
| E7.1 | Comprehensive tests | Unit, fuzz, invariant (19 invariants), fork, Echidna, E2E; ≥ 90% line coverage on core contracts |
| E7.2 | Static analysis | Slither + Aderyn in CI; no unresolved High findings; `SECURITY.md` triage table |
| E7.3 | Multichain deploy + verify | Foundry scripts deploy hub + spoke; contracts verified on Etherscan/Basescan |
| E7.4 | README + roadmap | Local spin-up via Docker Compose in ≤ 15 min; roadmap section |

## 6. Functional requirements
_Defined in `01-requirements-analysis.md` §3.1 (from the instructor meeting); repeated here so the PRD stands alone._

| ID | Requirement | Priority | Covered by |
|---|---|---|---|
| F1 | Outgoing spend rules: periodic (velocity) limits and approved (allowlisted) recipients | Must | E1, E2 |
| F2 | Receiving contract: allowlist of senders, allowed amount ranges, invoice verification hook | Must | E3.2–E3.4, E3.9 |
| F3 | Payment clearance only after invoice match + compliance checks (hold → clear → release) | Must | E3.4–E3.8 |
| F4 | Works alongside an existing custodian (we don't hold keys; custodian MPC keys = Safe owners) | Must (design) / Should (real integration → roadmap) | E2.3 |
| F5 | Cross-chain payments via hub-and-spoke routing | Must (1 lane) | E2.7, E3.3 |
| F6 | Transparent routing/UI fees (CCIP fee quoted, shown, counted toward limits) | Should | E2.7 |
| F7 | Fee modes: payer pays vs. sponsored (gasless) transactions | Should | E4 |
| F8 | API abstraction layer + MCP server for AI agents (propose-only) | Should | E5 |
| F9 | Off-chain reminders / settlement deadlines | Should | E3.10 |
| F10 | AI invoice reconciliation | Won't (roadmap) | — |

## 7. Non-functional requirements
| Area | Requirement |
|---|---|
| Security | Threat model T1–T16 mitigated (R9); receivers never revert on business rules; anti-bricking guarantee; all loosening time-locked |
| Correctness | 19 invariants hold under Foundry invariant tests and Echidna |
| Gas | Guard overhead target < 60k gas per simple transfer (measure; document) |
| Latency | Same-chain ops: 1 block. Cross-chain: minutes (CCIP finality), clearly shown in the UI |
| Availability | If the backend/relayer is down, owners can still execute signed Safe transactions directly (no loss of control) |
| Auditability | Every decision/state change emits an event; public on-chain rules (coarse caps) |
| Privacy | Only coarse caps + allowlist on-chain; documents off-chain; Merkle allowlist as a Should |
| Usability | Every DENY / NEEDS_APPROVAL has a human-readable reason in the UI |
| Portability | Engine takes a normalized `Intent` (ERC-7579-ready); bridge behind `ICrossChainAdapter`; relayer behind `IRelayer` |

## 8. Success metrics (demo / evaluation)
- **All Must stories demoed end-to-end on testnet** (Sepolia + Base Sepolia).
- **Attack-replay demo:** Bybit-style `delegatecall` blocked; forged CCIP sender quarantined; agent proposal can't execute.
- Tests: 19/19 invariants green; ≥ 90% core coverage; Slither/Aderyn clean of Highs.
- Local setup from README in ≤ 15 minutes.
- (Stretch) Gasless 4337 op passing the module guard.

## 9. Scope (MoSCoW) — see `04-tech-discovery.md` §10
Must: E1, E2 (incl. E2.7), E3 (except CREATE2), E4.1–E4.2, E5, E6.1–E6.2, E7. Should: E4.3, E4.4, EAS KYB, Merkle allowlist, CREATE2 deposit addresses, E6.3. Could: CRE verdict, `propose_invoice`, ERC-7579 wrapper, MultiSend decoding.

## 10. Release plan (Modules 13–16, ~4 weeks)

| Week | Module | Focus | Exit |
|---|---|---|---|
| **W1** | M13 Oracles | Spikes S1–S8; repo + CI skeleton; PolicyEngine core (buckets, tiers, allowlist, oracle adapter); Go ramp-up | Spikes decided; engine unit/fuzz tests green; CI running |
| **W2** | M14 Static/dynamic analysis | PolicyGuard + PaymentModule on Safe v1.5; governance/time-lock/pause; InvoiceRegistry + Escrow; invariant + Echidna harnesses | Same-chain outgoing + incoming flows work on Anvil; 19 invariants written |
| **W3** | M15 | CCIP adapter + SpokeGateway; Go API, relayer, listener/jobs; subgraph; Next.js core screens | Cross-chain payout + incoming work on testnet; UI covers E1–E3 |
| **W4** | M16 + Final Evaluation | MCP server; Should items as time allows; E2E tests; deploy + verify; README, SECURITY.md, roadmap; demo rehearsal | All Musts demoed; docs complete |

## 11. Dependencies
Safe v1.5 deployments (Sepolia/Base Sepolia) · Chainlink CCIP lane + Data Feeds on Sepolia · Circle USDC faucet · EAS (Should) · Subgraph Studio · Safe Transaction Service · RPC providers.

## 12. Risks (top) — see `04-tech-discovery.md` §12
Scope vs one month · Go/The Graph learning · USDC on the testnet lane · testnet feed staleness · security tests squeezed · guard bricking.

## 13. Pitch questions — answered by Pops (2026-09-29)
| # | Question | Answer | Consequence |
|---|---|---|---|
| 1 | CCIP acceptable, or LayerZero preferred? | **CCIP for now**; switch only if the instructor asks | Keep `ICrossChainAdapter` so a LayerZero adapter is a contained change (≥2 DVNs, pinned configs, Composer escrow) |
| 2 | Does M13 still expect Chainlink Functions? | **No** | No Functions dependency; CRE verdict stays a Could |
| 3 | Safe v1.5 base vs ERC-7579 modular account? | To be raised in the pitch with the explanation below | Engine stays account-agnostic; the 7579 hook wrapper is a stretch |
| 4 | Testnet-only demo acceptable? | **Yes** | D15 unchanged (L2 mainnet fallback only if rehearsal fails) |

**Q3 explained (for the pitch):**
- *Safe v1.5 (our choice):* the most-used institutional multisig. Our rules plug in as a **guard** (checks every transaction, including module/4337 ones since v1.5) plus a **module** (proposals/approvals). Lowest security risk for a solo build; custodians already sign for Safes.
- *ERC-7579 modular account:* a newer standard where plug-ins ("hooks") run on any compliant wallet brand (Kernel, Nexus, Safe via adapter). More portable and modern, but the standard is still **Draft**, and the Safe adapter is younger and less used (its 2024 audit found a critical wallet-takeover bug, since **fixed**; code changed after the fix review wasn't re-audited). It also adds a layer to learn and test.
- *Our bridge between both:* the PolicyEngine takes a wallet-neutral `Intent`, so wrapping it as an ERC-7579 hook later is a thin adapter, not a rewrite. **Ask:** "Is Safe + guard acceptable, or do you want the 7579 version in scope?"