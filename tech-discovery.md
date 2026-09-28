# Step 4 — Tech Discovery (consolidated)

_Written 2026-09-27. Consolidates Step 3 research (R1–R13) and decisions D1–D21 into one reference. Input for Step 5 (PRD, solution design, system architecture) and the pitch deck. Source of truth for decisions: `00-decisions-log.md`._

**Working name:** *Treasury Policy Layer*

---

## 1. Problem → solution in brief

**Problem.** Institutions hold stablecoins across several chains through custodians (Fireblocks, BitGo, Coinbase Prime) whose rules live **off-chain**. Off-chain rules can be bypassed (recovery paths, compromised UIs), don't cover **incoming** payments, and aren't consistent **across chains**. Recent losses show that the contracts can be fine while configuration and infrastructure fail:
- **Bybit, Feb 2025, ~$1.5B:** compromised UI + blind signing → `delegatecall` wallet takeover.
- **KelpDAO, Apr 2026, ~$292M:** single-verifier bridge config + poisoned RPC → forged cross-chain message.

**Solution.** An **on-chain policy backstop** on top of existing custody:
1. **Outgoing:** USD-priced limits, allowlists, tiered approvals and time-locked governance enforced by a Safe guard.
2. **Incoming:** invoice-linked **escrow with clearance** (sanctions/KYB checks, refunds, an unmatched-cash queue).
3. **Cross-chain:** one rulebook on a **hub chain**, with bridge messages treated as untrusted.
4. **AI-agent-safe:** agents can **propose** via an MCP server but never spend.

**Pitch lines**
- "Custodians protect the keys. We protect the transactions."
- "A multisig checks *who* signed. Our guard checks *what* they signed."
- "Others guard what goes out. We also clear what comes in, keep one rulebook across chains, and let AI agents propose but never spend."

---

## 2. Requirements → decisions traceability

| Req | Requirement | Decision(s) | Where |
|---|---|---|---|
| F1 | Outgoing spend rules (limits, allowlist) | D9, D12 | R2, R3 |
| F2 | Receiving rules (sender, amount range, invoice hook) | D13 | R4 |
| F3 | Hold → clear → release | D13 | R4 |
| F4 | Work alongside a custodian | D9 (custodian MPC key = Safe owner) | R1, R2 |
| F5 | Cross-chain via hub | D2, D11 | R5 |
| F6 | Transparent fees | D11 (fees count toward limits), D18 (fee modes) | R5, R6 |
| F7 | Sponsored transactions | D18 | R6 |
| F8 | API + MCP (propose-only) | D5, D17 | R8 |
| F9 | Reminders / deadlines | D13, D20 (jobs) | R4, R11 |
| F10 | AI reconciliation | Roadmap | — |
| Guide: Solidity + audited libs | OZ + Safe + Chainlink | D9, D12, D16 | R2, R3, R9 |
| Guide: frontend | Next.js + viem/wagmi | D19 | R10 |
| Guide: access control / reentrancy / pull-over-push | Roles, CEI + ReentrancyGuard, pull settle/refund | D12, D13, D16 | R3, R4, R9 |
| Guide: oracles | Chainlink Data Feeds (+ CRE Could) | D12, D14 | R3, R7 |
| Guide: Slither + tests (unit/E2E/fork/fuzz/Echidna) | Full toolchain + 19 invariants | D16 | R9 |
| Guide: multichain deploy + verification | Foundry scripts per chain, Etherscan/Basescan verification | D11 (R12 deferred to dev) | — |
| Guide: README + roadmap | §12 roadmap; Docker Compose local setup | D20 | R11 |

---

## 3. Architecture overview

```
                                OFF-CHAIN (our services)
 ┌───────────────────────────────────────────────────────────────────────────────┐
 │  Next.js dApp (TS)          Go API (Gin, REST/OpenAPI)         MCP server (Go)│
 │  - dashboard, payments,     - SIWE auth, proposals queue      - read/simulate/ │
 │    invoices, escrow queue,  - relayer (IRelayer)                propose tools │
 │    policy, payer page       - event listener + Asynq jobs      (AGENT key)    │
 │                             - compliance clearer (fallback)                   │
 │  The Graph subgraph (history)   PostgreSQL   Redis                            │
 └───────────────────────────────────────────────────────────────────────────────┘
             │ reads/writes                    │ submits signed txs
             ▼                                 ▼
 ┌──────────────────────── HUB: Ethereum Sepolia ────────────────────────────────┐
 │  Safe v1.5 (treasury; owners = custodian MPC keys / demo EOAs)                │
 │    ├─ PolicyGuard (tx guard + module guard)  ──►  PolicyEngine (UUPS)         │
 │    │                                              rules, buckets, tiers,     │
 │    │                                              governance, ≤3 hooks        │
 │    ├─ PaymentModule (proposals, K-of-N EIP-712 approvals)                     │
 │    └─ (Should) Safe4337Module ─► also checked by the module guard              │
 │  PriceOracle adapter ──► Chainlink Data Feeds (ETH/USD, USDC/USD) / mocks     │
 │  InvoiceRegistry + Escrow (PENDING→CLEARED/FLAGGED→SETTLED/REFUNDED)          │
 │    hooks: SanctionsHook (Chainalysis iface mock), EAS KYB hook, CRE receiver  │
 │  CcipAdapter (ICrossChainAdapter) ◄──► Chainlink CCIP Router                  │
 └──────────────────────────────────────┬────────────────────────────────────────┘
                                        │ CCIP lane (PAYOUT / INVOICE_PAYMENT / REFUND)
 ┌──────────────────────── SPOKE: Base Sepolia ──────────────────────────────────┐
 │  SpokeGateway: authenticity checks only (router, source chain, hub sender),   │
 │  per-lane caps, pause; payer entry point payInvoice(); payout to payees       │
 └───────────────────────────────────────────────────────────────────────────────┘
```

**Principles:** policy decided on the hub · authenticity checked everywhere · receivers never revert on business rules · funds live on the hub (Model A) · every loosening change is time-locked · everything off-chain can fail without breaking the on-chain safety guarantees.

---

## 4. Component catalogue

### 4.1 On-chain (Solidity, Foundry)

| Component | Responsibility | Upgradeable? | Reuses |
|---|---|---|---|
| **PolicyEngine** | Normalized `Intent` checks; ALLOW / NEEDS_APPROVAL / DENY; token buckets (global + per role); allowlist + activation delay; tiers + hard cap; asymmetric governance (tighten instantly / loosen via time-lock + veto); ≤3 `IPolicyHook`s (staticcall, gas cap, deny-overrides) | **UUPS** behind time-lock (EIP-7201 storage) | OZ AccessControlDefaultAdminRules, UUPS |
| **PolicyGuard** | Safe transaction guard + module guard; decodes shapes (ETH, ERC-20, `approve`, `ccipSend`, self-calls, `delegatecall`, MultiSend blocked in v1); anti-bricking path | Immutable | Safe guard interfaces |
| **PaymentModule** | Operator/agent proposals; K-of-N EIP-712 approvals (maker-checker, nonce, deadline, intent hash); executes via `execTransactionFromModule` | Immutable | OZ EIP712, SignatureChecker |
| **PriceOracle** | `usdValue(token, amount) → (usd18, status)`; staleness per feed, $1 floor for stables, round up, deviation check | Config via engine | Chainlink AggregatorV3 |
| **InvoiceRegistry + Escrow** | Invoices (key fields + `docHash`); payments; escrow state machine; pull-based `settle` / `claimRefund`; UNMATCHED queue; verdict intake | **Immutable** (holds funds) | OZ ReentrancyGuard, SafeERC20 |
| **Hooks** | SanctionsHook (Chainalysis interface; mock on testnet), EAS KYB hook (Should), CRE receiver (Could) | Replaceable via time-lock | EAS, CRE `ReceiverTemplate` |
| **CcipAdapter** | `ICrossChainAdapter` impl; encode/decode message types; quote fees; hub-side receive | Immutable | CCIP Router, CCIPReceiver |
| **SpokeGateway** | Authenticity checks, per-lane caps, pause, payer entry point, payouts | Immutable | CCIP |
| **Mocks** | MockV3Aggregator, mock sanctions oracle, fixed-price feed | — | — |

### 4.2 Off-chain

| Component | Responsibility | Stack |
|---|---|---|
| **Web app** | Dashboard, payments, payees, invoices, escrow queue, policy (time-lock countdown + veto), payer page | Next.js 16, viem/wagmi, RainbowKit, Safe Kits, Tailwind/shadcn |
| **API** | SIWE auth; proposal/approval collection; invoice documents; payee labels; notifications | Go, Gin, OpenAPI, Postgres (sqlc/pgx) |
| **Relayer** | Submits signed payloads (simulate → send → track → bump) | Go, go-ethereum |
| **Listener + jobs** | Chain events → actions (reminders, CCIP status, stuck-message alerts, gas budget alerts) | Go, Asynq + Redis |
| **Subgraph** | Payment/invoice/escrow history for the UI | The Graph (Subgraph Studio) |
| **MCP server** | Read / simulate / propose tools for AI agents | Go MCP SDK |
| **Compliance clearer (fallback)** | Signs EIP-712 verdicts if CRE isn't used | Go |

---

## 5. Key flows (summary; detailed sequences go into Step 5)

| # | Flow | Steps |
|---|---|---|
| 1 | **Operator payment ≤ auto limit** | Operator signs intent → API → relayer → `PaymentModule.execute` → module guard → PolicyEngine ALLOW → transfer → bucket consumed |
| 2 | **Payment needing approval** | Proposal → engine says NEEDS_APPROVAL(tier) → K approvers sign the exact intent hash (EIP-712) → relayer executes → guard re-checks → transfer |
| 3 | **Agent proposal (MCP)** | Agent `simulate_payment` → `propose_payment` (AGENT key) → always NEEDS_APPROVAL → humans approve → flow 2 |
| 4 | **Owner-path (large) payment** | Owners sign a Safe tx (Safe Kit) → `execTransaction` → tx guard → ALLOW up to the hard cap |
| 5 | **Cross-chain payout** | Flow 1/2/4 with `ccipSend` (decoded by the guard; fee counted) → CCIP → SpokeGateway checks authenticity + lane cap → payee paid |
| 6 | **Incoming invoice payment (same chain)** | Payer `payInvoice` (approve / permit / `receiveWithAuthorization`) → escrow PENDING → auto-checks → CLEARED (or FLAGGED / AWAITING_VERDICT) → Safe pulls via `settle` |
| 7 | **Incoming cross-chain payment** | Payer on Base → SpokeGateway → CCIP (token + invoiceId, atomic) → hub Escrow (never reverts) → flow 6 checks |
| 8 | **Reject + refund** | Approver rejects FLAGGED → refund entitlement → payer `claimRefund` (same chain) or approver-triggered CCIP REFUND |
| 9 | **Policy change** | Tighten → immediate. Loosen → queued (event + alert) → veto window (guardian/owners) → execute after ETA |
| 10 | **Emergency** | Guardian `pause()` (engine and/or a lane) → only minimal owner-path + recovery; unpause needs the admin quorum |

---

## 6. Data model (summary)

**On-chain (hub):**
- **PolicyEngine:** `payees[(chainSelector, address)]` → {active, activatesAt, labelHash}; per-role limits {auto, tier2, perTx}; buckets {capacity, rate, available, lastUpdate}; hard cap; token config {feed, heartbeat, isStable, fixedPrice}; pending changes {id, payload, eta}; hooks[≤3].
- **PaymentModule:** proposals {id, intentHash, proposer, status}; used nonces.
- **InvoiceRegistry / Escrow:** invoices {id, issuer, payer(opt), token, amount, dueDate, docHash, status, paid}; entries {id, invoiceId, payer, sourceChain, token, amount, state, reason}; `accounted[token]`.

**Off-chain (Postgres):** users/roles (SIWE); proposals + signatures; invoice documents + metadata; payee labels; notifications/reminders; relayer tx log; CCIP message tracking; audit export.
**Subgraph entities:** Payment, Proposal, Approval, Invoice, EscrowEntry, PolicyChange, CrossChainMessage.

---

## 7. Roles & permissions

| Role | Can | Can't |
|---|---|---|
| **Safe owners** (custodian MPC keys) | Owner-path payments up to the hard cap; admin quorum (`POLICY_ADMIN` = the Safe) for policy changes | Exceed the hard cap; bypass time-locks |
| **GUARDIAN** | Pause instantly; veto queued loosening changes | Unpause alone; move funds |
| **OPERATOR** | Propose; auto-execute ≤ auto limit | Approve own proposals; add payees without the time-lock |
| **APPROVER** | Sign approvals (K-of-N); clear/reject escrow entries | Approve own proposals (maker-checker) |
| **AGENT** (MCP) | Read, simulate, propose | Auto-execute ($0), approve, add payees, change policy |
| **Relayer** | Submit already-signed payloads | Anything else (no role) |
| **Payer** (external) | Pay invoices; claim refunds | — |

---

## 8. Security summary
- **Threat model:** 16 threats (R9 §2); top 5 = Bybit-style UI/blind signing, KelpDAO-style forged message, agent prompt injection, admin compromise, guard bricking.
- **Invariants:** 19 (R9 §6) across the engine/guard, escrow, cross-chain and upgrades.
- **Toolchain:** Foundry (unit/fuzz/invariant/fork, coverage ≥ 90% core, gas snapshots) · Slither + Aderyn every push · Echidna + fork tests nightly · Chainlink Local · Playwright E2E · OZ upgrade validation.
- **Attack-replay tests:** Bybit-style `delegatecall`, KelpDAO-style forged sender, replayed approval, reentrant token callback.

---

## 9. Tech stack summary

| Layer | Stack |
|---|---|
| Contracts | Solidity, Foundry, OpenZeppelin, Safe v1.5, Chainlink (Feeds, CCIP, Local, CRE optional), EAS |
| Frontend | Next.js 16, TypeScript, viem + wagmi, RainbowKit, Safe Protocol/API Kit, Tailwind + shadcn/ui, TanStack Query, RHF + Zod, Vitest, Playwright |
| Backend | **Go** (Gin), OpenAPI (+ generated TS client), go-ethereum/abigen, PostgreSQL + sqlc + pgx, Asynq + Redis, SIWE, Go MCP SDK |
| Indexing | The Graph (Subgraph Studio) + Go event listener |
| Infra | Docker Compose (Postgres, Redis, Anvil×2), GitHub Actions CI, Vercel (web), Railway/Fly (Go services) |
| Networks | Hub: Ethereum Sepolia · Spoke: Base Sepolia · Fallback: L2 mainnet (D15) |
| Tokens | USDC, WETH/ETH, CCIP-BnM (fallback) |

---

## 10. MVP scope (MoSCoW)

| Must | Should | Could | Won't (roadmap) |
|---|---|---|---|
| PolicyEngine + PolicyGuard + PaymentModule on Safe v1.5 | Merkle "hidden until used" allowlist | Chainlink **CRE** verdict workflow | AI invoice reconciliation (F10) |
| Token buckets, tiers, hard cap, allowlist + activation delay, asymmetric governance + pause | EAS KYB hook | `propose_invoice` MCP tool | Real custodian API adapters |
| Chainlink USD pricing (ETH/USD, USDC/USD or mock) | USDC `receiveWithAuthorization` + permit payer flows | ERC-7579 hook wrapper (stretch) | Privacy L2 / ZK allowlists |
| InvoiceRegistry + Escrow (state machine, pull settle/refund, UNMATCHED) | CREATE2 deposit addresses | Synpress smoke test | LayerZero adapter (USDT0/PYUSD) |
| Sanctions hook (mock) | ERC-4337 via Pimlico (after spike) | MultiSend batch decoding | Receivable NFTs / invoice financing |
| CCIP hub ↔ Base Sepolia (PAYOUT, INVOICE_PAYMENT, REFUND) + SpokeGateway | | | Spoke float, 3rd chain, mainnet |
| Own relayer (Go) | | | OAuth for MCP |
| MCP server (read/simulate/propose) | | | |
| Next.js dApp (7 screens) + Go API + subgraph + jobs | | | |
| Full test suite + Slither/Aderyn/Echidna CI | | | |
| Multichain deploy scripts + verification, README | | | |

---

## 11. Dev week 1: spikes & verifications (with exits)

| # | Spike / check | Exit criteria | If it fails |
|---|---|---|---|
| S1 | **CCIP hub ↔ spoke** with Programmable Token Transfer (Chainlink Local → real testnet); USDC lane check | Token + invoiceId delivered; receiver never reverts; latency/fee measured | CCIP-BnM demo token; if CCIP fails entirely → single-chain MVP (D10) |
| S2 | **Safe v1.5 + PolicyGuard** on Sepolia (tx guard + module guard, anti-bricking removal) | Guard blocks/permits as designed; removal after delay works | Safe v1.4.1 + tx guard; PaymentModule self-checks |
| S3 | **Safe v1.5 + `Safe4337Module` + Pimlico/Candide** with custom addresses | Sponsored user op executes **and** hits our module guard | Keep relayer-only (Must already covers F7) |
| S4 | **Price feeds** on Sepolia (USDC/USD, ETH/USD addresses, heartbeats, liveness) | Addresses + heartbeats recorded in config | Fixed-$1 mock for USDC (time-locked switch) |
| S5 | **Go ramp-up**: Gin + sqlc + go-ethereum "hello relayer" (sign + send a tx on Anvil) | Relayer submits a signed Safe tx locally | TS API fallback (same OpenAPI) |
| S6 | **Subgraph** on Subgraph Studio for Sepolia (one entity) | Deployed + queried from Next.js | Go listener serves history from Postgres |
| S7 | **CI skeleton**: Foundry tests + Slither + Aderyn + first Echidna property | Green pipeline | — |
| S8 | Ask the instructor: M13 still expects Chainlink Functions (sunset)? | Answer recorded | Use CRE or skip (Could) |

---

## 12. Risk register (top)

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Scope too big for one month solo | High | High | Strict MoSCoW; spikes in week 1; Should/Could only after Musts |
| Go + The Graph learning load | Medium | Medium | Timeboxed ramp-up (S5/S6), TS fallback, subgraph read-only |
| USDC not on the testnet CCIP lane | Medium | Low | CCIP-BnM + mock feed |
| Testnet feed staleness during the demo | Medium | Medium | Fail-closed to approval tier; mock fallback; rehearse; L2 mainnet fallback (D15) |
| Safe v1.5 + 4337 SDK incompatibility | Medium | Low | Relayer covers F7 |
| Security tests squeezed at the end | Medium | High | 25–30% of time reserved; CI from week 1 |
| Guard bricking bug | Low | High | Anti-bricking invariant + test from day 1 |
| Instructor prefers LayerZero / ERC-7579 | Low | Medium | Adapter interfaces (`ICrossChainAdapter`, account-agnostic engine) |

---

## 13. Post-bootcamp roadmap
1. **Production hardening:** audit, OZ Relayer or managed relayer, KMS keys, OAuth for MCP, monitoring (OpenTelemetry).
2. **More chains + spoke float** (pre-funded spoke allowances); an L2 or permissioned hub option.
3. **Multi-bridge:** LayerZero adapter for USDT0/PYUSD (≥2 DVNs, pinned configs), CCTP V2 direct.
4. **ERC-7579 hook** wrapper + ERC-7484 attestation-gated hook registry.
5. **Privacy:** Merkle allowlists everywhere; privacy L2 (Aztec) research; stealth addresses.
6. **Receivables:** invoice NFTs, invoice financing/factoring.
7. **AI:** invoice reconciliation (F10), anomaly detection feeding guardian alerts.
8. **Real custodian adapters** (Fireblocks/BitGo signer integration), ERP/AR connectors.

---

## 14. Document index
| Doc | Content |
|---|---|
| `00-decisions-log.md` | D1–D21 + schedule |
| `01-requirements-analysis.md` | Goals, requirements, risks, curriculum coverage |
| `02-research-plan.md` / `02a-research-method.md` | Plan, method, weights |
| `r1-institutional-custody-landscape.md` | Custodian policy models, on-/off-chain split, Bybit |
| `r2-smart-account.md` | Safe v1.5 choice, integration notes, privacy |
| `r3-policy-engine.md` | Buckets, hybrid engine, tiers, governance, oracles rules |
| `r4-invoices-escrow.md` | Invoices, escrow state machine, compliance, payer UX |
| `r5-cross-chain.md` | CCIP choice, hub design, adoption, KelpDAO, faucets |
| `r7-oracles-tokens.md` | Token list, feeds, oracle interface, mainnet cost |
| `r9-security-testing.md` | Threat model, upgradeability, toolchain, invariants |
| `r6 Gas, r8 MCP, r10 FE, r11 BE, r13 Competitors.md` | Gas, MCP, frontend, backend (Go), competitors, build-vs-buy |
| **`04-tech-discovery.md`** | **This consolidation** |