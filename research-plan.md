# Step 2 — Research Plan

## Goal of the research
Gather enough facts to make each architecture decision with evidence, and to show the instructor that the design builds on existing standards instead of reinventing them.

Every research track ends with: **findings → options table → recommendation → open risks**. These feed Step 4 (tech discovery docs).

## Priority order
P0 = blocks the architecture. P1 = shapes features. P2 = nice to know / roadmap.

---

## R1 — Institutional custody landscape (P0)
**Why:** Align with real custody expectations for better portfolio; defines what "institutional-grade" means.

Questions:
1. How do Fireblocks, BitGo, Coinbase Prime (and Copper, Anchorage) model policies? (transaction authorization policy, approval quorums, whitelists, velocity limits)
2. What's done off-chain (MPC signing, policy engine) vs on-chain? Where does an on-chain policy layer add value?
3. Do any of them sign for smart-contract wallets (e.g., Fireblocks + Safe)? What would an "adapter" to them look like?
4. Common policy primitives to copy: roles, quorum, time-locks, allow/deny lists, per-asset limits, travel rule/compliance hooks.

Output: policy-primitive list + "on-chain vs off-chain" responsibility split.

## R2 — Smart account base: "custodian stand-in" (P0)
**Why:** Decision D4. The policy layer's shape depends on the wallet it attaches to.

Options to compare:
- Safe{Wallet} + Guards (transaction guard, module guard) + Modules
- Safe + Zodiac Roles Modifier (existing permissions system — competitor/reference)
- ERC-7579 modular accounts (Rhinestone, Kernel/ZeroDev, Biconomy Nexus, Safe7579 adapter) — validators, executors, hooks
- ERC-6900 (Alchemy modular accounts)
- Plain custom vault contract (no smart account)

Compare on: maturity/audits, testnet availability, 4337 compatibility, 7702 compatibility, how hooks/guards can block a tx, dev tooling in Foundry, portfolio signal.

Output: recommendation + where our policy code plugs in (guard vs hook vs module).

## R3 — Policy engine design patterns (P0)
**Why:** Core of the product (F1).

Questions:
1. How to implement rolling/periodic spending limits cheaply (fixed window vs sliding window vs token bucket)?
2. USD-denominated limits across multiple tokens — how to price with Chainlink; stale-price handling; fallback.
3. Pluggable policy design: one contract with all rules vs registry of policy modules (e.g., `IPolicy.check(tx) → allow/deny/needsApproval`).
4. Approval tiers: auto-approve under X, require N approvers above X, time-lock above Y.
5. **Off-chain approvals with EIP-712 signatures** (covered in M12): approvers sign typed data instead of each sending a tx — gas-free for approvers, replay protection via nonce + chainId + deadline.
6. **Admin safety:** OpenZeppelin `AccessControlDefaultAdminRules` (used in M05) — 2-step admin transfer with delay; fits institutional change control.
7. Existing references: Zodiac Roles v2 (allowances), Safe Allowance Module, ERC-7579 spending-limit hooks, Coinbase Spend Permissions.

Output: policy interface sketch (no code), list of v1 policies.

## R4 — Receiving side: invoices, escrow, clearance (P0)
**Why:** Main differentiator (F2, F3). Must be concrete.

Questions:
1. Invoice standards/prior art: Request Network, ERC-3643 (compliance for permissioned tokens), ERC-7521, on-chain invoice NFTs, payment reference patterns (memo field, invoice hash).
2. How to tie a payment to an invoice on-chain (invoice ID in calldata, CREATE2 per-invoice deposit addresses, payment references).
3. Escrow + pull-over-push: states (Pending → Cleared → Withdrawable / Rejected → Refundable), timeouts, who can clear.
4. Handling unexpected payments (unknown sender, wrong amount, direct ERC-20 transfers that bypass the contract).
5. Compliance hooks: sanctions/allowlist oracles (e.g., Chainalysis sanctions oracle), attestations (EAS).
6. **Chainlink Functions (taught in M13)** as the invoice-verification / compliance hook: contract asks an off-chain API "is invoice #X approved / is sender sanctioned?" and clears on callback. Compare vs. trusted "clearer" role vs. EAS attestation. Check testnet availability, cost, latency.
7. **Payer UX with signatures (M12):** EIP-2612 `permit` and EIP-3009 `transferWithAuthorization` (supported by USDC) — payer signs, relayer submits "pay invoice" in one tx.

Output: state machine + invoice data model + chosen verification mechanism.

## R5 — Cross-chain: CCIP vs LayerZero V2 (P0)
**Why:** F5; decides token list (D3) and where policy is enforced.

Known from M10: sending a LayerZero message between two chains. Focus research on the gaps.

Questions:
1. Token transfer model: CCIP (lock/burn-mint token pools, Programmable Token Transfers, CCT standard) vs LayerZero (OFT/OFT Adapter, composed messages).
2. Which tokens can actually move on testnet Sepolia ↔ Base Sepolia (CCIP-BnM/LnM, test USDC; OFT we deploy ourselves)?
3. Fees: how they're paid (native vs LINK), how to quote them in-contract, how to pass/absorb them (F6, F7).
4. Security model: DON/RMN (CCIP) vs DVNs (LayerZero); rate limits; failure/retry handling (manual execution, lzRetry).
5. Hub-and-spoke: can the hub receive, check policy, and forward in one flow? Latency and cost of two hops.
6. **Where to enforce policy**: source, hub, destination — trade-offs.
7. Circle CCTP as a USDC-specific alternative.
8. Local testing: Chainlink Local, LayerZero TestHelper / devtools in Foundry; fork tests.
9. Stack coherence: CCIP + Chainlink Data Feeds + Chainlink Functions = one vendor/one mental model (M13 support) vs. LayerZero familiarity (M10). Score honestly.

Output: comparison table + chosen protocol + message flow diagram + spike plan.

## R6 — Gas sponsorship: ERC-4337 & EIP-7702 (P1, light)
**Why:** F7. Concepts covered in M12 (4337, 7702, 2771, 3009, permit); assignment used a **MockEntryPoint**, so the gap is the real network path.

Questions:
1. Real EntryPoint version (v0.7 / v0.8) on Sepolia/Base Sepolia and which one the chosen account (R2) supports.
2. Bundler + paymaster providers with free testnet tiers: Pimlico, Alchemy, Coinbase CDP, Biconomy, Candide — incl. ERC-20 paymasters (pay gas in USDC).
3. 7702 fit: does an institutional flow need it (EOA operators upgrading) or is a smart account enough? Known 7702 vulnerabilities (M12) to design around.
4. Fee modes: sponsor all / pass to payer / deduct from transfer — how each is implemented and shown in UI.
5. Build own verifying paymaster vs use provider (portfolio vs effort).

Output: fee-mode design + provider choice.

## R7 — Oracles & multi-token support (P1, pre-study for M13)
**Why:** Guide requires oracles; D3. M13 starts with development, so design must be settled first.

Questions:
1. Chainlink Data Feeds available on Sepolia and Base Sepolia (USDC/USD, ETH/USD, LINK/USD, EUR/USD…).
2. Staleness, decimals normalization, L2 sequencer uptime feed (Base).
3. Final token list = intersection of (bridgeable on testnet) ∩ (has price feed).

Output: supported token matrix + price-check interface.

## R8 — Agent access: API + MCP server (P1)
**Why:** F8, decision D5 (propose-only).

Questions:
1. MCP spec basics: tools, resources, auth; TypeScript SDK.
2. Existing wallet/crypto MCP servers (Coinbase AgentKit, Safe, GOAT, Alchemy) — patterns to copy.
3. Propose-only mechanics: agent gets a restricted "proposer" role / session key; proposal = EIP-712 signed intent stored off-chain or on-chain; policy pre-check via simulation (`eth_call`) before submit.
4. Security: prompt-injection limits, rate limits, agent identity.

Output: MCP tool list + permission model.

## R9 — Security & testing toolchain (P1, pre-study for M14)
**Why:** Grading items; M14 (Slither, Echidna, Advanced Foundry) comes mid-build, so invariants and test structure must be designed now.

Questions:
1. Common vulnerabilities in guards/modules/cross-chain receivers (known Safe guard bricking, CCIP receiver validation, reentrancy in escrow, signature replay).
2. Invariants to fuzz with Foundry + Echidna (list them now).
3. Slither/Aderyn setup in CI; fork testing for CCIP/LayerZero.
4. Upgradeability: UUPS vs immutable + migration — what's appropriate for a policy layer? EIP-7201 namespaced storage (M12) if upgradeable.

Output: threat model outline + invariant list + test plan skeleton.

## R10 — Frontend stack refresh (P1, lighter after M05)
**Why:** D6. M05 covered React + Tailwind + wagmi + viem + MetaMask/WalletConnect + auto network switching (ERC-20 payment gateway pattern). The gaps are Next.js and multichain/smart-account UX.

Questions:
1. Next.js current version: App Router, Server Components, Server Actions, caching — what's stable and recommended now?
2. React current version: new APIs (`use`, actions, compiler) — what matters for a dApp.
3. Wallet connection libraries (RainbowKit, ConnectKit, Reown/AppKit) and smart-account UX (Safe Apps SDK / Protocol Kit, permissionless.js for 4337).
4. Multichain UI: showing balances and payment status across Sepolia + Base Sepolia; cross-chain message tracking (CCIP Explorer / LayerZero Scan links).
5. UI kit & forms: shadcn/ui, Tailwind current version, react-hook-form + Zod, TanStack Query.
6. Testing: Playwright for E2E with a wallet (Synpress or mocked connector), Vitest.

Output: FE stack choice + "what changed" cheat sheet.

## R11 — Backend & off-chain services (P1, from D6)
**Why:** D6 — make the backend a real service. It also carries F6 (fee quotes), F8 (API + MCP) and F9 (reminders/settlement deadlines).

Questions:
1. What must live off-chain: invoice documents/metadata (only hash on-chain), payment history, reminders, fee quotes, proposal queue for MCP, relayer for signed payments/approvals.
2. Runtime & framework: Node.js vs Bun; Hono / Fastify / NestJS / Next.js route handlers — trade-offs for a small typed API. tRPC vs REST vs GraphQL.
3. Database & ORM: Postgres + Drizzle vs Prisma; hosted options (Neon, Supabase).
4. Indexing on-chain events: Ponder vs Envio vs The Graph vs own viem listener — multichain support, reorg handling.
5. Background jobs: settlement deadline reminders, retrying failed cross-chain messages — BullMQ/Redis, Trigger.dev, Inngest, cron.
6. Auth: Sign-In with Ethereum (SIWE, EIP-4361) + roles mirrored from on-chain.
7. Monorepo layout: Turborepo/pnpm workspaces with `contracts/`, `web/`, `api/`, `indexer/`, `mcp/`, shared ABIs/types (wagmi CLI codegen).
8. Deployment: Vercel/Railway/Fly for API + indexer; local dev with Docker Compose + Anvil.

Output: BE architecture sketch + stack choice + monorepo layout.

## R12 — Deployment tooling (P2)
Questions: multichain Foundry scripts (CREATE2/CreateX for same addresses), verification on Etherscan/Basescan (Etherscan V2 multichain API), CI with GitHub Actions.

Output: tooling choices.

## R13 — Competitive scan (P2)
Safe + Zodiac, Fireblocks policies, Den, Squads (Solana), Coinbase Smart Wallet spend permissions, Rhinestone, Brahma Console, Request Finance. Goal: one slide on "why this is different".

---

## Curriculum alignment (what the bootcamp gives us, and when)

| Module | Status | Feeds |
|---|---|---|
| M05 Frontend (React, Tailwind, wagmi, viem, wallets) | Done | R10 (lighter) |
| M07 DAOs / governance | Done | R3 approval tiers, time-locks |
| M08 Upgradeable contracts | Done | R9 upgrade strategy |
| M10 L2 + LayerZero messaging | Done | R5 (gaps only) |
| M12 EIP-712, permit, 3009, 2771, 4337, 7702, 7201 | Done | R3 signed approvals, R4 payer UX, R6 (light), R8 signed intents |
| M13 Oracles + **Chainlink Functions** | Upcoming (week 1 of dev) | R7 pre-study, R4 verification hook |
| M14 Slither, Echidna, Advanced Foundry | Upcoming (mid-build) | R9 pre-study — invariants designed now |
| M15 PoS, M16 Restaking | Upcoming | Not in scope; roadmap only |
| M16 Capstone Final Evaluation | End | MVP demo |

## Method
See `02a-research-method.md` (gates, weighted scoring per D7, confidence tags).

## Execution order
1. Wave 1 (P0): R1 → R2 + R5 → R3 + R4.
2. Wave 2 (P1): R7, R9 first (pre-study for M13/M14), then R6, R8, R10, R11.
3. Wave 3 (P2): R12, R13.

## Exit criteria for Step 3
Each P0/P1 track has a recommendation the user has accepted and logged in `00-decisions-log.md`.