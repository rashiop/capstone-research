
# Wave 2 — R6 Gas, R8 MCP, R10 Frontend, R11 Backend, R13 Competitors

_Step 3 research, Wave 2. v1 2026-09-27 (quick picks); **v2 2026-09-27: R6, R10, R11 and R13 expanded after Pops' review; R8 approved (D17).** Confidence: **V** Verified · **L** Likely · **U** Unverified._

---

## R6 — Gas sponsorship (F7) — expanded

### R6.1 Why Safe v1.5 matters (and why v1.4.1 isn't enough for us)
- A Safe can execute transactions in two ways: **owner-signed** (`execTransaction`) or **through an enabled module** (`execTransactionFromModule`).
- **Safe ≤ v1.4.1:** the transaction guard only checks owner-signed transactions. **Anything executed by a module bypasses the guard.** **V** (R2 §2A)
- **Safe v1.5.0 (July 2025):** adds the **Module Guard** (`checkModuleTransaction`), so module transactions are checked too. **V**
- **Why that's decisive for us:**
  - Our **PaymentModule** is a module. On v1.4.1 it would bypass the guard (we could make it self-check, but that's weaker).
  - **The ERC-4337 module (`Safe4337Module`) is itself a module.** It executes user operations via `execTransactionFromModule`. **V** (source code)
  - So on v1.4.1, **every gas-sponsored 4337 transaction would skip our policy guard entirely**. That's exactly the kind of bypass the project exists to prevent.
  - On v1.5, the same 4337 transactions **pass through our Module Guard**.
- **Conclusion:** v1.4.1 + 4337 = gasless but unguarded. v1.5 (+ 4337 if it works) = gasless **and** guarded. Keep **D9 (Safe v1.5)**; no need to revisit R2.

### R6.2 Is there a 4337 module/SDK that supports Safe v1.5?

| Piece | Status | Confidence |
|---|---|---|
| `Safe4337Module` contract | Requires **Safe ≥ 1.4.1**; executes via `execTransactionFromModule` → on v1.5 it works under the Module Guard | Requirement **V**; works on 1.5 **L** |
| Candide `Safe4337MultiChainSignatureModule` (EntryPoint **v0.9**) | Preflight requires **Safe ≥ 1.4.1** | **V** (requirement); v1.5 not explicitly listed **U** |
| Pimlico `permissionless.js` `toSafeSmartAccount` | Officially **safeVersion "1.4.1"**, EntryPoint v0.7 default; **accepts custom contract addresses** (factory, singleton, modules) | **V**; passing v1.5 addresses **U** → spike |
| Safe7579 adapter (R2 option B) | 4337 via the adapter; our policy would then be a 7579 hook | **V** (exists), not our path (D9) |

**Honest answer:** no provider advertises "Safe v1.5 + 4337" as a supported preset today. The contracts should be compatible (the 4337 module only needs ≥ 1.4.1), but the SDKs are pinned to 1.4.1 addresses. **Needs a half-day spike** (dev week 1): deploy Safe v1.5 + `Safe4337Module` on Sepolia → send a sponsored user operation via Pimlico or Candide with custom addresses → confirm our Module Guard runs.

### R6.3 4337 provider options (if the spike works)

| Provider | What it is | Safe support | Pros | Cons / limits |
|---|---|---|---|---|
| **Pimlico** (permissionless.js) | Bundler + paymaster + viem-based TS SDK | ✅ `toSafeSmartAccount` (1.4.1; custom addresses) | **viem-native** (same stack as wagmi); very good docs; supports many account types; free testnet tier **L** | Safe preset pinned to 1.4.1 |
| **Candide** (AbstractionKit) | Bundler + paymaster, **Safe-specialist** (co-built the Safe 4337 module) | ✅ several Safe account versions (EntryPoint v0.6/0.7/0.9) | Deepest Safe focus; newest EntryPoint v0.9 multichain-signature module | Own SDK (not viem-native); smaller community **L** |
| **Alchemy** (Account Kit / Gas Manager) | Bundler + paymaster + own **Modular Account (ERC-6900)** | Partial / other accounts first **L** | Big brand; strong infra | SDK centred on their account type |
| **Biconomy** (Nexus) | Bundler + paymaster + own **ERC-7579** account | Not Safe-first **L** | 7579 ecosystem | Pushes their account |
| **ZeroDev** (Kernel) | 7579 account + infra | Not Safe-first **L** | Strong 7579 tooling | Pushes their account |
| **Coinbase CDP Paymaster** | Paymaster (ERC-7677), Base-centric | Works with standard 4337 accounts **L** | Great on Base; generous credits **L** | Base-focused; hub is Sepolia |

**Why Pimlico first:** it uses **viem** (the same library as the frontend and backend), so one mental model. It has the most flexible Safe support (custom addresses) and the widest docs. **Candide is the close alternative** (Safe specialists). Try Candide if Pimlico's Safe preset fights us in the spike.

### R6.4 Relayer options (sponsorship without 4337)

| Option | What it is | Pros | Cons / limits |
|---|---|---|---|
| **A. Build our own relayer** (backend wallet submits already-signed payloads with viem) | ~200–400 lines in `apps/api`: queue → simulate (`eth_call`) → send → track → retry | Works with **Safe v1.5 today**; no vendor or EntryPoint version issues; full control (simulate before paying gas, per-role rate limits); **no special permissions** (it can only submit what others signed); strong backend portfolio piece; cheap | **We run a hot key** (theft = loss of gas funds + DoS, not treasury funds); must handle **nonces, stuck/under-priced txs, gas bumping, reorgs**; monitoring + gas budget alerts; **griefing** (spam requests that fail on-chain cost us gas → simulate first, only accept payloads signed by known roles, rate limit); not standard 4337 (other bundlers can't pick up our ops); liveness depends on us (mitigation: anyone can still submit a signed Safe tx manually) |
| **B. OpenZeppelin Relayer** (open-source, self-hosted) | Successor to Defender Relayer (**Defender sunset July 1, 2026**) **V** | Battle-tested design (nonce mgmt, gas bumping, key backends); any EVM chain **V** | Another service to run (container + config); learning curve; less to show as "our code" |
| **C. Gelato Relay** (hosted) | `sponsoredCall` / ERC-2771 relaying, paid via prepaid balance; Sepolia supported **V** | No infra; mature | Vendor + billing; ERC-2771 needs `_msgSender()` handling in our contracts (extra attack surface); less control |
| **D. ERC-4337 paymaster** (R6.3) | Standard account abstraction | Industry standard; strongest "AA" signal | Safe v1.5 SDK support unconfirmed (spike) |

### R6.5 Recommendation
1. **Must: A, our own minimal relayer**, behind an `IRelayer` interface in the backend (so B or C can replace it). Guardrails:
   - simulate-before-send
   - only role-signed payloads
   - per-role rate limits
   - gas budget + alerts
   - a separate low-balance hot wallet
   - nonce manager with gas bumping
   Scope ≈ 1.5–2 days.
2. **Should: D, 4337 via Pimlico** (Candide as the alternative) **if** the dev-week-1 spike shows Safe v1.5 + `Safe4337Module` works under our Module Guard. That becomes a strong demo: "gasless AND still policy-checked".
3. **Not planned:** C (ERC-2771 adds `_msgSender` complexity to security-critical contracts), B (extra ops work for a solo MVP; mention as the production path).

---

## R8 — Agent access: API + MCP server — **approved (D17)**
(Design unchanged from v1.)
- Tools: read (`get_policy_summary`, `get_balances`, `list_payees`, `get_invoice`, `list_open_invoices`), `simulate_payment`, `propose_payment` (+ `propose_invoice` as Could).
- No approve/execute/policy tools. AGENT key: $0 auto limit, can't add payees; rate-limited.
- MCP spec 2026-07-28 (stateless, OAuth hardening, JSON Schema 2020-12) **V**; official TypeScript SDK **V**.
- MVP transport: stdio or HTTP + API key; OAuth = roadmap.

---

## R10 — Frontend stack (options, trade-offs, job-market signal)

**Job-market signal** (for a "Fullstack Web3 Product Engineer" target):
- Stack Overflow 2025 (professional devs): **Node.js 49.1%, React 46.9%, Next.js 21.5%, Express 20.3%**. **V**
- Web3 frontend guides describe **React + Next.js + TypeScript + viem + wagmi** as the standard dApp stack, with **RainbowKit** as the most popular wallet UI and The Graph for indexing. **V** (gm.careers)
- Job-board data is qualitative (no reliable percentages). **L**

| Concern | Options (brief) | Pros / cons | Recommendation |
|---|---|---|---|
| **Framework** | **Next.js 16** (App Router, React Server Components) · **Vite + React SPA** (+ TanStack Router) · React Router v7 (ex-Remix) | Next: the job-market default, SSR for public pages (payer invoice links), API routes; but RSC + wallet state = client/server boundary care. Vite SPA: simplest for wallet-heavy apps, fast dev; less "full-stack" signal. RR7: good, smaller market | **Next.js 16** (16.3 LTS **V**). Keep wallet/Safe code in client components; use server components for public invoice pages |
| **Web3 library** | **viem + wagmi** · ethers v6 · thirdweb SDK | viem/wagmi: standard, typed, taught in M05. ethers: legacy but common in older codebases. thirdweb: fast, but vendor-shaped | **viem + wagmi** (confirm v2 vs v3 on day 1; examples exist for both **L**) |
| **Wallet UI** | **RainbowKit** · **Reown AppKit** (WalletConnect's official kit; email/social login) · ConnectKit · Privy/Dynamic (embedded wallets, SaaS) | RainbowKit: most popular, clean. AppKit: official WalletConnect; wagmi 3 examples exist **V**. Embedded wallets: great consumer UX but institutions use their own wallets/custodians | **RainbowKit** if it supports our wagmi version on day 1; otherwise **AppKit**. Two-way door |
| **Safe integration** | **Safe Protocol Kit + API Kit** (+ Safe Transaction Service, available on Sepolia & Base Sepolia **V**) · raw viem calls | Kit: signature collection, tx building, service integration. Raw: more control, more code | **Protocol Kit + API Kit** for owner-path flows |
| **UI kit** | **Tailwind + shadcn/ui** · MUI · Mantine · Chakra | Tailwind/shadcn: most in-demand in modern React roles, full control. MUI: enterprise look, heavier | **Tailwind + shadcn/ui** |
| **Server state** | TanStack Query (already inside wagmi) · SWR | One cache for chain + API data | **TanStack Query** |
| **Client state** | Zustand · Redux Toolkit · React context | Minimal need | **Zustand** (only if needed) |
| **Forms** | react-hook-form + **Zod** · Formik | Shared Zod schemas with the backend | **RHF + Zod** |
| **Unit/component tests** | **Vitest** + Testing Library · Jest | Vitest: fast, modern default | **Vitest** |
| **E2E** | **Playwright** (+ mocked wagmi connector) · Synpress (real MetaMask) · Cypress | Mocked connector: fast, stable. Synpress: realistic, flaky | **Playwright + mocked connector**; one Synpress smoke test if time |

**Considerations:** watch the React Server Components / wallet boundary; typed contract hooks via **wagmi CLI** codegen from Foundry; accessibility and clear transaction states (Sent → In transit (CCIP) → Delivered) matter for institutional UX.

---

## R11 — Backend & off-chain services (options, trade-offs, job-market signal)

**Job-market signal:**
- PostgreSQL is the most-used database among professional developers (**58.2%**, Stack Overflow 2025). **V**
- **Express** is still the most-used Node back-end framework; **NestJS** is growing; **Hono** has the highest satisfaction among newer frameworks (State of JS 2025). **V**
- Many crypto-native infra roles also use **Go/Rust**, but for a fullstack TS profile, TS end-to-end is the strongest story. **L**

| Concern | Options (brief) | Pros / cons | Recommendation |
|---|---|---|---|
| **Runtime / language** | **Node.js LTS + TypeScript** · Bun · Go | Node/TS: one language across web/api/indexer/MCP, largest job market. Bun: faster, less mature. Go: strong for infra roles, but splits the codebase | **Node.js + TS** |
| **API framework** | **Express** (most used) · **NestJS** (structured, DI; common in enterprise/fintech) · **Fastify** (fast, plugins) · **Hono** (light, typed, top satisfaction) | Express: universal, but dated DX. NestJS: resume value for enterprise/fintech, heavier to learn. Fastify: solid middle. Hono: fastest to build, modern, runs anywhere | **Hono** for delivery speed (skills transfer to Express/Fastify). **Choose NestJS instead** if you're targeting enterprise/fintech backend roles, at +1–2 days of learning |
| **API style** | **REST + OpenAPI** (via Zod) · tRPC · GraphQL | REST/OpenAPI: universal; external/institutional integrations and the MCP server can consume it. tRPC: great DX but TS-only coupling. GraphQL: overkill | **REST + OpenAPI** |
| **Database** | **PostgreSQL** · MySQL · MongoDB | Postgres: #1, relational fits invoices/proposals/audit logs; Ponder and pg-boss use it | **PostgreSQL** |
| **ORM / query** | **Drizzle** · **Prisma** · Kysely | Prisma: most known in job posts, great DX, heavier runtime. Drizzle: SQL-like, light, rising fast; Ponder's schema API is Drizzle-style **L**. Kysely: query builder only | **Drizzle** (mention Prisma familiarity on your CV; concepts transfer) |
| **Indexer** | **Ponder** (TS, self-hosted, writes to Postgres) · **The Graph** (subgraphs; most recognised in job posts) · Envio (fast, hosted) · custom viem listener | Ponder: same language, same DB, multichain; used by Uniswap's The Compact indexer **V**. The Graph: resume keyword, but separate stack (AssemblyScript) + hosted service. Envio: fast, vendor-hosted. Custom: reorg handling is hard | **Ponder** (optionally also publish a small subgraph later as a CV keyword) |
| **Background jobs** | **pg-boss** (Postgres-backed) · **BullMQ + Redis** (most common) · Trigger.dev / Inngest (SaaS) | pg-boss: no extra infra. BullMQ: very common in job posts, needs Redis. SaaS: nice DX, vendor | **pg-boss** for the MVP; BullMQ if you want the Redis keyword |
| **Auth** | **SIWE (EIP-4361)** + session/JWT · Auth.js/NextAuth with SIWE · Privy | SIWE: wallet = identity, standard | **SIWE** (+ roles read from chain) |
| **Relayer** | Own · OZ Relayer · Gelato | See R6.4 | **Own** (behind `IRelayer`) |
| **MCP server** | `@modelcontextprotocol/sdk` | — | Separate app (R8) |
| **Monorepo** | **pnpm + Turborepo** · Nx · single repo without tooling | Turborepo: simple, popular in Next.js shops. Nx: powerful, heavier | **pnpm + Turborepo** |
| **Local dev** | Docker Compose (Postgres) + **Anvil** (2 chains) + Chainlink Local | Reproducible README setup (a guide requirement) | ✅ |
| **Hosting (demo)** | Vercel (web) + Railway / Fly / Render (api, indexer, mcp, Postgres) | Cheap, quick | Two-way door |
| **Observability** | pino logs + a health endpoint; OpenTelemetry = roadmap | Enough for the demo | ✅ |

### R11 v3 — revised after Pops' direction (2026-09-27): **job demand first; a second language and extra infra are fine**
The table above optimised for "one language, least infra". With those constraints removed, the recommendation changes to a **Go backend**, since Go is heavily used for crypto backend/infra roles, and **The Graph** as the indexer.

**Evidence:**
- web3.career lists **~1,440 Golang jobs** (Sep 2026), including payments, messaging and SRE roles at crypto firms (e.g., LayerZero Labs, Jump Crypto listings), with senior ranges of about $110k–$280k. **V** (job board snapshot; counts for other languages not shown on that page)
- **Redis** is used by 30.7% and **PostgreSQL** by 58.2% of professional developers. **V** (Stack Overflow 2025)
- **The Graph** is the indexing name most associated with web3 roles. **V** (gm.careers)

| Concern | Pick | Why (job demand + fit) | Notes |
|---|---|---|---|
| Language | **Go** | Common in exchanges, custody, payments and infra teams; strong concurrency for relayer/listeners | **Learning cost if Go is new: ~3–5 days to productive** (see risk below) |
| HTTP framework | **Gin** (most widely used Go web framework **L**) · alt: Chi (stdlib-style) | Recognised keyword; simple | REST + **OpenAPI** spec → generate a typed TS client for Next.js (`openapi-typescript`), so types still flow across languages |
| Ethereum client | **go-ethereum** (`ethclient`, `abigen` bindings generated from Foundry ABIs) | "geth" is a strong CV keyword; the standard Go EVM library | Relayer, event listeners, EIP-712 verification in Go |
| Database | **PostgreSQL** + **sqlc** (type-safe SQL codegen) + **pgx** driver; migrations with goose/golang-migrate | Idiomatic modern Go data stack | — |
| Indexing / read model | **The Graph** subgraph (Subgraph Studio; Sepolia and Base supported **L**; free tier 100k queries/month **V**) | Top web3 indexing keyword; the frontend queries history via GraphQL | Subgraph handlers are AssemblyScript (TS-like). The Go service also runs a **light event listener** for actions (reminders, CCIP tracking) so jobs don't depend on subgraph lag |
| Jobs / queue | **Asynq + Redis** (alt: River on Postgres) | Redis is a common requirement; Asynq is a popular Go task queue **L** | Reminders (F9), CCIP status polling, relayer retries, alerts |
| Auth | **SIWE (EIP-4361)** via a Go SIWE library **L** + JWT sessions | Standard wallet login | — |
| Relayer | Own, in Go (`IRelayer`-style interface) | Same design as R6.5 | Nonce manager, gas bumping, simulate-before-send |
| MCP server | **Official Go MCP SDK** (v1 stable; 2026-07-28 spec support in pre-release) **V** · alt: TS SDK | Keeps the backend in one language | If the Go SDK's new-spec support is still pre-release at build time, pin the stable version (older spec) or use the TS SDK |
| Frontend | Unchanged: Next.js + TS (R10) | — | — |
| Repo | Monorepo: `apps/web` (pnpm/Turborepo), `apps/api` (Go module), `subgraph/`, `contracts/` (Foundry); **Taskfile/Makefile** + **Docker Compose** (Postgres, Redis, Anvil×2, graph-node optional) | README "spin up locally" requirement | — |

**Honest risk (push-back):**
- A new language plus The Graph adds learning load **in the same month** as M13–M16, the security test suite (25–30% of time), and cross-chain work.
- **Mitigation:**
  1. Keep the Go backend scope tight: API, relayer, listeners/jobs, MCP.
  2. Use the subgraph only for read-only history.
  3. Timebox Go ramp-up to dev week 1, alongside the spikes.
  4. **Fallback:** if Go ramp-up blocks progress by the end of week 1, switch the API to TS (Hono/NestJS). The OpenAPI contract stays the same, so the frontend isn't affected.
- If Go is already familiar, the learning cost mostly disappears.

---

## R13 — Competitive landscape & build-vs-buy (expanded)

### R13.1 What each existing product does, and how we differ

| Product | What it is | What it does well | Gap relative to our project |
|---|---|---|---|
| **Fireblocks / BitGo / Coinbase Prime** (R1) | MPC custodians with **off-chain** policy engines | Rich private rules, key security, compliance ops | Rules are off-chain (bypassable via recovery paths, BitGo docs **V**); not focused on **incoming-payment clearance**; caps can span assets/chains (Fireblocks `asset:"*"` USD TIMEFRAME **L**, BitGo enterprise $250k/day check **V**) but only for transactions they sign, not on-chain (R1b). **We complement them:** their MPC key is a Safe owner; we add an on-chain backstop |
| **Safe + Zodiac Roles** (R2) | Multisig + role/permission module with token-unit allowances | Mature, audited, expressive permissions | No USD pricing; no receiving side; module path only (unless combined with a guard); no cross-chain rules |
| **Cobo Argus** (R1, R3) | Safe-based role delegation with pluggable authorizers; Chainlink-priced checks | Institutional DeFi operations (farming, delegation) | Focused on **outgoing DeFi operations**, not invoices/incoming payments or cross-chain treasury **L** |
| **Brahma** | Safe-based accounts with **sub-accounts** and access control for teams | Team operations, sub-account isolation | Outgoing-focused; no invoice escrow **L** |
| **Coinbase Spend Permissions** | Per-app periodic allowances on Coinbase smart wallets (fixed periods) **V** | Subscriptions and recurring app payments | Consumer/app allowances, not institutional treasury governance; no receiving side |
| **Request Network / Request Finance** | Invoicing + on-chain payment **detection** via payment references **V** | Invoices, accounting integrations | **Detects** payments but doesn't **enforce** rules on arrival (no escrow/clearance/refunds) or on outgoing spend |
| **CCIP / LayerZero / CCTP** (R5) | Cross-chain transport | Moving tokens/messages | Transport only; no treasury policy (KelpDAO shows why a policy layer on top matters) |

**Our unique combination:**
1. **On-chain outgoing backstop** (USD-priced limits, tiers, time-locked governance) that sits *on top of* existing custody.
2. **Incoming-payment clearance** (invoice-linked escrow, sanctions/KYB checks, refunds, an unmatched-cash queue).
3. **One rulebook across chains** (hub-enforced, bridge messages treated as untrusted).
4. **AI-agent-safe by construction** (propose-only, $0 auto limit, enforced on-chain).

**Slide line:** *"Custodians and Safe tools guard what goes **out**. We also clear what comes **in**, keep one rulebook across chains, and let AI agents propose but never spend."*
Confidence on competitor details: **L** (docs/landing pages, time-boxed). Phrase gaps as "not their focus".

### R13.2 Build vs off-the-shelf (what we write vs what we use)

| Layer | **Off-the-shelf (use)** | **Build (our code)** |
|---|---|---|
| Wallet / custody | **Safe v1.5** (singleton, proxy factory), Safe Transaction Service, Safe Protocol/API Kit | — |
| Policy | OpenZeppelin: `AccessControlDefaultAdminRules`, `EIP712`, `SignatureChecker`, `ReentrancyGuard`, `SafeERC20`, UUPS + upgrade validation | **PolicyEngine** (rules, token buckets, tiers, time-locked governance), **PolicyGuard** (Safe tx + module guard adapter), **PaymentModule** (proposals, K-of-N approvals), `IPolicyHook` + hooks |
| Pricing | **Chainlink Data Feeds**, MockV3Aggregator pattern | **Price oracle adapter** (staleness, floor, rounding, status) |
| Receiving side | Chainalysis sanctions **interface** (mock on testnet), **EAS** (attestations), Chainlink **CRE** (Could), USDC ERC-3009 | **InvoiceRegistry + Escrow** (state machine, refunds, unmatched queue), sanctions/EAS hooks, CRE receiver |
| Cross-chain | **Chainlink CCIP** (router, token pools, RMN), CCTP under CCIP, **Chainlink Local** | **CcipAdapter** (`ICrossChainAdapter`), **SpokeGateway**, message encoding, per-lane caps |
| Gas | (Should) Pimlico/Candide bundler + paymaster, `Safe4337Module` | **Relayer** (`IRelayer`) |
| Agent | **MCP TypeScript SDK** | **MCP server** tools + agent signing |
| Backend | Node, Hono, Postgres, Drizzle, **Ponder**, pg-boss, SIWE libs | API, indexer config/handlers, jobs (reminders, CCIP tracking, alerts), relayer, compliance clearer |
| Frontend | Next.js, wagmi/viem, RainbowKit/AppKit, Tailwind/shadcn, TanStack Query | All screens and flows |
| Security tooling | Foundry, Slither, Aderyn, Echidna, crytic/properties, Playwright | Tests, invariants, harnesses, CI pipeline |

**Why this split matters for the pitch:** we reuse audited infrastructure for everything that's already solved (custody, messaging, pricing, attestations) and write our own code only where the value is (policy, clearance, cross-chain rules, agent safety). That's the "don't reinvent the wheel" guidance from the instructor meeting.

---

## Summary of picks (status)
1. **R6:** Must = own minimal relayer (`IRelayer`), Safe v1.5 kept; Should = 4337 via Pimlico (Candide alt) after the spike proves v1.5 + `Safe4337Module` + Module Guard. **Approved (D18–D21).**
2. **R8:** **Approved (D17).**
3. **R10:** Next.js 16 + viem/wagmi + RainbowKit (or AppKit) + Safe Protocol/API Kit + Tailwind/shadcn + TanStack Query + RHF/Zod + Vitest + Playwright. **Approved (D18–D21).**
4. **R11 (v3):** **Go** (Gin) + REST/OpenAPI (typed TS client for the frontend) + go-ethereum + PostgreSQL/sqlc/pgx + **The Graph** subgraph (read model) + light Go event listener + **Asynq/Redis** jobs + SIWE + own Go relayer + Go MCP SDK; Docker Compose monorepo. Fallback to TS API if Go ramp-up blocks by end of dev week 1. **Approved (D18–D21).**
5. **R13:** differentiation + build-vs-buy as above. **Approved (D18–D21).**

## Sources
- Pimlico — `toSafeSmartAccount`: https://docs.pimlico.io/permissionless/reference/accounts/toSafeSmartAccount
- Safe4337Module source: https://github.com/safe-fndn/safe-modules/blob/main/modules/4337/contracts/Safe4337Module.sol
- Safe + ERC-4337: https://docs.safe.global/advanced/erc-4337/4337-safe
- Safe v1.5.0 (module guards): https://safefoundation.org/blog/introducing-safe-v1-5-0-module-guards-enhanced-smart-account-features
- Candide — Safe account versions: https://docs.candide.dev/wallet/abstractionkit/safe-account/
- Candide — AbstractionKit v0.4.0: https://docs.candide.dev/blog/abstractionkit-v0-4-0-release
- OpenZeppelin — Phasing out Defender (sunset July 1, 2026): https://www.openzeppelin.com/news/doubling-down-on-open-source-and-phasing-out-defender
- OpenZeppelin — Defender migration: https://docs.openzeppelin.com/defender/migration
- Gelato Relay — ERC-2771: https://docs.gelato.cloud/web3-services/relay/erc-2771-recommended
- MCP 2026-07-28 spec RC: https://blog.modelcontextprotocol.io/posts/2026-07-28-release-candidate/
- MCP TypeScript SDK: https://github.com/modelcontextprotocol/typescript-sdk
- Stack Overflow Developer Survey 2025 — Technology: https://survey.stackoverflow.co/2025/technology
- State of JS 2025 — Back-end frameworks: https://2025.stateofjs.com/en-US/libraries/back-end-frameworks/
- gm.careers — Frontend developer in Web3: https://gm.careers/blog/frontend-developer-web3-dapps
- Next.js releases: https://releasebot.io/updates/vercel/next-js
- Reown AppKit examples → wagmi 3: https://github.com/reown-com/appkit-web-examples/pull/228
- Safe Transaction Service — Sepolia: https://docs.safe.global/core-api/transaction-service-reference/sepolia
- Safe Transaction Service — Base Sepolia: https://docs.safe.global/core-api/transaction-service-reference/base-sepolia
- Ponder: https://ethereum.org/developers/tools/ponder/
- Uniswap The Compact indexer (Ponder): https://github.com/Uniswap/the-compact-indexer
- Brahma sub-accounts: https://docs.brahma.fi/features-and-functionalities/access-control/sub-accounts
- Cobo Argus: https://www.cobo.com/developers/v1/overview/smart-contract-wallet/coboargus
- Coinbase Spend Permissions: https://github.com/coinbase/spend-permissions
- Request Network: https://docs.request.network/advanced/protocol-overview/how-payment-networks-work
- BitGo policies (recovery bypass): https://developers.bitgo.com/guides/policy-builder/overview
- web3.career — Golang jobs (Sep 2026): https://web3.career/golang-jobs
- MCP Go SDK releases: https://github.com/modelcontextprotocol/go-sdk/releases
- The Graph — Subgraph Studio pricing: https://thegraph.com/studio-pricing/
- The Graph — Sepolia in Subgraph Studio: https://graphcentral.substack.com/p/support-for-sepolia-testnet-live
- River (Go + Postgres job queue): https://github.com/riverqueue/river