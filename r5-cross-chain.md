# R5 — Cross-Chain: CCIP vs LayerZero V2 (vs CCTP)

_Step 3 research, Wave 1. Researched 2026-09-25 (v2 same day: adoption evidence, LayerZero institutional score corrected 3→4, deeper notes §10, testnet faucets §10). Method: `02a-research-method.md` (gates + D7 weights). Confidence: **V** Verified in primary source · **L** Likely · **U** Unverified. **Decision: approved → D11.**_

## 1. Decision & requirements served
- **Decisions:** (a) which cross-chain protocol, (b) the hub-and-spoke topology, (c) **where policy is enforced** on cross-chain payments, (d) which tokens can move on testnet (feeds D3/R7).
- **Serves:** F5 (cross-chain payments), F2/F3 (receiving cross-chain payments), F6 (fees), F7 (who pays).
- **Known from M10:** sending a LayerZero message between two chains. This track focuses on the gaps: moving tokens, hub routing, failure handling, CCIP.

### Terms used below
- **Lane / pathway:** a supported source → destination chain pair.
- **Programmable Token Transfer (CCIP):** tokens + a data message in **one** cross-chain message; the receiver contract gets both together.
- **OFT (LayerZero):** Omnichain Fungible Token, LayerZero's token standard (burn/mint or lock/release across chains).
- **Composer (LayerZero):** a contract that runs follow-up logic *after* OFT tokens arrive, in a **separate** second step (`lzCompose`).
- **DVN (LayerZero):** Decentralized Verifier Network, the parties that attest a message is real. You choose and configure them.
- **RMN (CCIP):** Risk Management Network, Chainlink's independent second layer that watches for bad messages and can pause.
- **CCTP:** Circle's Cross-Chain Transfer Protocol: burns native USDC on one chain and mints it on another.
- **Canonical bridge:** the bridge a token's *issuer* officially uses. Moving the token any other way creates a "wrapped" copy.

---

## 2. Options in detail

### Option A — Chainlink CCIP

**What it is.** Chainlink's cross-chain protocol. Send tokens, data, or both (Programmable Token Transfers) via a `Router` on each chain. Fees are paid in LINK or native gas (ETH/WETH). **V**

**Testnet facts (Base Sepolia ↔ Ethereum Sepolia)** **V**
- Active lane Base Sepolia → Ethereum Sepolia (OnRamp v1.6.0). Base Sepolia router `0xD3b06cEbF099CE7DA4AcCf578aaebFDBd6e88a93`, chain selector `10344971235874465080`.
- Fee tokens on Base Sepolia: LINK, WETH, native ETH.
- Test tokens: **CCIP-BnM** (burn & mint, on all testnets, `drip()` = 1 token per call) and **CCIP-LnM** (native on Sepolia, wrapped `clCCIP-LnM` elsewhere).
- **USDC:** CCIP supports USDC; it uses **Circle CCTP under the hood** where both chains support it (burn → attest → mint). **V** Whether USDC is enabled on *this specific lane* on testnet: **L**. **Confirm in the CCIP Directory at build time.**

**Pros**
- **Tokens + data arrive atomically** (§9.1 explains why this matters).
- **Same vendor as our oracle (Data Feeds) and possible compliance hook (Functions, M13)** (§9.3 covers the trade-off).
- **Institutional positioning:** the strongest bank / financial-market-infrastructure track record of the options (§9.2).
- **Defence in depth built in:** RMN plus per-token **rate limits** (token-bucket capacity + refill rate) on token pools. **V**
- **Own token later:** the Cross-Chain Token (CCT) standard lets us register our own token self-serve with Burn&Mint or Lock&Release pools. **V**
- **Testing:** **Chainlink Local** (`CCIPLocalSimulator`) runs CCIP inside Foundry tests, including forked environments. **V**
- **New skill for the portfolio** (not covered in the bootcamp).

**Cons**
- **New to you:** no prior hands-on experience (LayerZero was M10).
- **Less DeFi-native than LayerZero** (§9.4 explains what that costs us).
- **Can't move LayerZero-native tokens** such as USDT0, PYUSD or USDe without a wrapped copy (§9.5).
- **Fees in LINK or native:** must quote (`getFee`) and fund the sender. Adds fee-handling logic (F6).

**Limitations / considerations**
- **Default destination gas:** token-pool operations get ~90,000 gas on the destination. If the receiver needs more, set `gasLimit` in `extraArgs`. **V**
- **Latency:** waits for source-chain finality. Sepolia finality ≈ minutes. Fine for treasury payments, not instant.
- **Testnet lanes and token support can change.** Verify before the demo.

**Known issues & known fixes**

| Issue | Status / fix |
|---|---|
| **Receiver revert → message stuck in "manual execution".** Happens on unhandled errors, too little gas, or token pools needing > 90k gas. After the **Smart Execution window (currently 8 hours)**, later messages from the same sender wait behind it (ordering preserved). **V** | **Our fix:** the receiver **never reverts on business logic**. It accepts tokens into escrow with a status (Pending / Flagged) and handles invoice mismatch or unknown sender *inside* our state machine (clear / refund). Only revert on "impossible" conditions (wrong router, wrong source). |
| **Spoofed messages** (anyone can call `ccipReceive` if unchecked). | Only accept calls from the Router (`CCIPReceiver` base does this); allowlist `(sourceChainSelector, sender)` pairs. |
| CCIP had a public audit contest (Code4rena, Nov 2024, OffRamp code). **V** | Use official releases; pin versions. |

---

### Option B — LayerZero V2 (OFT + Composer)

**What it is.** A messaging protocol with configurable security. Tokens move as **OFTs**. Follow-up logic runs in a **Composer** via `lzCompose` after the tokens land. **V**

**Testnet facts** **V**
- Ethereum Sepolia endpoint ID **40161**; EndpointV2, ULN libraries, Executor and DVNs are deployed on testnet. Base Sepolia pathway supported (verified by Pops, D2).
- **Tokens:** no native test USDC path. We'd **deploy our own OFT** (e.g., a mock stablecoin), or an OFT Adapter around an existing token.

**Pros**
- **Familiar:** you already sent messages in M10 → lower learning cost.
- **The stablecoin issuers' choice:** Tether (USDT0), PayPal (PYUSD) and Ethena (USDe) are native OFTs; Fireblocks embedded LayerZero in its Tokenization Engine (§9.2). **V** (LayerZero blog)
- **Very widely used in DeFi**; lots of examples; **TestHelperOz5** for Foundry unit tests and an address book for fork tests. **V**
- **Configurable security:** choose DVNs (institutions can even run their own, e.g., Fidelity, Worldpay per LayerZero). **V** (vendor claim)

**Cons**
- **Two-step, non-atomic token + logic:** OFT credits tokens to the Composer first, then `lzCompose` runs separately. If compose fails, **tokens stay in the Composer** and the compose must be retried manually. **V**
- **Security configuration is on you:** defaults are **"unsafe for production"**, controlled by LayerZero Labs and can rotate without notice. Production should pin configs on both sides with **≥ 2 required DVNs** from independent operators. **V**
- **No native test USDC:** a mock OFT is less realistic for the demo.
- **Weaker portfolio novelty:** repeats M10.

**Limitations / considerations**
- **`lzReceive` can be called by anyone for a verified message.** Access control must live in your `_lzReceive` / `lzCompose` (check `msg.sender == endpoint` and `from == OFT`). **V**
- **Gas must be set in options.** Too little and `lzReceive` won't execute. **V**

**Known issues & known fixes**

| Issue | Status / fix |
|---|---|
| Default DVN config can drift / is single-DVN in examples. **V** | Explicit `setConfig` on both sides, `requiredDVNCount ≥ 2`, a CI check on the config. |
| Compose failure leaves tokens in the Composer. **V** | Composer holds tokens in escrow state; retry path + admin rescue; invariant test "no tokens are ever unaccounted for in the Composer". |

---

### Option C — Circle CCTP V2 only (USDC-specific)

**What it is.** Circle's native USDC burn-and-mint. V2 adds **Fast Transfer** (seconds, before finality) and **Hooks** (run a function on the destination atomically with the mint). Live on testnet for Ethereum Sepolia and Base Sepolia. **V**

**Pros**
- **Real native USDC,** the most realistic stablecoin for an institutional treasury.
- **Hooks** can call our receiver with invoice data, atomically. **V**
- Circle is an institution-friendly issuer.

**Cons**
- **USDC only.** Conflicts with D3 (multi-token), unless combined with something else.
- **Not a general messaging layer:** no arbitrary cross-chain policy messages.
- Attestation flow (fetch attestation from Circle's API, then mint) adds an off-chain step. **L**

**Best use for us:** it already sits **under CCIP for USDC**. Choosing CCIP gets CCTP's native USDC behaviour without integrating it separately. **V**

---

## 3. Hard gates

| Option | Sepolia + Base Sepolia | No business account | Foundry + viem | Maintained / audited | License | Solo-buildable | Result |
|---|---|---|---|---|---|---|---|
| A. CCIP | ✅ V | ✅ (faucets) | ✅ Chainlink Local | ✅ | ✅ | ✅ | **Pass** |
| B. LayerZero V2 | ✅ V | ✅ | ✅ TestHelperOz5 | ✅ | ✅ | ✅ | Pass |
| C. CCTP V2 only | ✅ V | ✅ (Circle faucet) | ✅ | ✅ | ✅ | ✅ | Pass (USDC only) |

## 4. Weighted scoring (D7 weights)

| Criterion (weight) | A. CCIP | B. LayerZero V2 | C. CCTP only |
|---|---|---|---|
| Requirement fit (25) | **5** — atomic token+data, USDC via CCTP, multi-token, rate limits | 4 — works, but two-step compose; mock tokens | 3 — USDC only |
| Institutional alignment (20) | **5** — banks, FMIs, central-bank pilots (§9.2) | **4** (v2, was 3) — stablecoin issuers, PayPal, Fireblocks; banks less so | 4 — Circle |
| Grading coverage (15) | **5** — pairs with oracle requirement | 4 | 3 |
| Portfolio & learning (12) | **5** — new protocol + Chainlink stack | 3 — repeats M10 | 3 |
| Maturity & security (10) | 4 — RMN, rate limits | **3** (v3, was 4) — KelpDAO $292M exploit via 1-of-1 DVN config; ~47% of apps ran 1-of-1 (§9.6) | **5** |
| Testnet & tooling (10) | **5** — Chainlink Local + fork sim | 4 | 3 |
| Solo effort & risk (8) | 3 — new to you | **4** — familiar | **4** |
| **Weighted total (/5)** | **4.74** | **3.78** (v3; v2 3.88; v1 3.68) | 3.48 |

**Sensitivity check:** doubling effort (8→16) and cutting institutional (20→12) → CCIP still clearly ahead.
**Honest caveat:** CCIP's lead is large partly because it lines up with *your* goals (institutional story + Chainlink oracle requirement + new skill). On pure engineering merit, the two are closer. LayerZero is a valid choice.

---

## 5. Topology and where policy is enforced

### 5.1 Hub-and-spoke for the MVP
- **Hub = Ethereum Sepolia (D2):** the treasury Safe, PolicyEngine, PolicyGuard, PaymentModule and the **InvoiceReceiver / escrow** live here.
- **Spoke = Base Sepolia:** a light **SpokeGateway** contract (receives outgoing payments for payees on Base; sends incoming payer payments to the hub).
- Both MVP flows are **one hop** (hub ↔ spoke). A true two-hop "spoke → hub → spoke" route needs a third chain → roadmap.

```
OUTGOING (treasury pays a vendor on Base)
Safe (Sepolia) ──PolicyGuard/PolicyEngine checks: payee allowlisted for (chain, address),
                 USD limit, velocity ──► CCIP Router.ccipSend(token + data)
                                      ──► SpokeGateway (Base Sepolia)
                                          checks source = hub, sender = our Safe/module
                                          → transfers to payee

INCOMING (customer on Base pays an invoice to the treasury)
Payer (Base Sepolia) ──► SpokeGateway.payInvoice(invoiceId, token, amount)
                      ──► CCIP (token + invoiceId) ──► InvoiceReceiver (Sepolia hub)
                          never reverts on business logic → escrow: Pending
                          → clearance (invoice match, sender allowlist, compliance hook)
                          → Cleared → Safe pulls funds  |  Rejected → refund path
```

### 5.2 Where to enforce policy

| Location | What's checked there | Why |
|---|---|---|
| **Source (hub) — outgoing** | Full policy via PolicyGuard: destination chain + payee allowlisted, USD limits, velocity, approvals. The guard must **decode `ccipSend`** | The decision is made *before* money leaves |
| **Spoke — outgoing arrival** | Only authenticity: came from the hub's Router, from our Safe/module | Keep spokes dumb: one policy source of truth |
| **Hub — incoming** | Sender allowlist per (source chain, address), amount range, invoice match, compliance hook → escrow state machine (R4) | The receiving-side differentiator lives on the hub |
| **Spoke — incoming send** | Optional pre-check (invoice exists, amount in range) to fail fast before paying the fee | UX only; the hub stays authoritative |

**Principle:** policy is decided on the hub, authenticity is checked everywhere, and receivers never revert on business rules.

### 5.3 Fees (F6, F7)
- The CCIP fee is quoted on-chain with `getFee` and paid in native ETH or LINK. **V**
- **MVP choice: pay CCIP fees in native Sepolia/Base Sepolia ETH.** No LINK needed at all (§10).
- Fee modes to design in R6: (a) the treasury pays (sponsored), (b) the payer pays on the spoke, (c) a protocol/UI fee is deducted and shown in the UI.
- The guard counts the CCIP fee toward the velocity limit.

### 5.4 Keep the bridge swappable (added v2)
Define a small **`ICrossChainAdapter`** interface: `quote()`, `send()`, and a normalized receive callback. **CcipAdapter** implements it for the MVP. That makes the "multi-bridge" roadmap concrete: a **LayerZeroAdapter** for USDT0 / PYUSD (§9.5), or a CCTP adapter. It's also the answer to vendor-concentration risk (§9.3).

### 5.5 Hub design in detail — multi-chain example (added v3)

**Example setup (roadmap scale; MVP = hub + Base Sepolia only):**

```
                         ┌──────────────────────────────────────────┐
                         │  HUB: Ethereum Sepolia                    │
                         │  Safe treasury (most funds)               │
                         │  PolicyEngine (ONE set of rules + buckets)│
                         │  PolicyGuard · PaymentModule              │
                         │  InvoiceReceiver / escrow                 │
                         │  HubRouter (CcipAdapter)                  │
                         └───────┬──────────────┬──────────────┬────┘
                          CCIP lane       CCIP lane       CCIP lane
                                 │              │              │
                ┌────────────────▼──┐  ┌────────▼─────────┐  ┌─▼────────────────┐
                │ SPOKE: Base Sep.  │  │ SPOKE: Arbitrum  │  │ SPOKE: Optimism  │
                │ SpokeGateway      │  │ Sepolia          │  │ Sepolia          │
                │ (+ optional float)│  │ SpokeGateway     │  │ SpokeGateway     │
                └───────────────────┘  └──────────────────┘  └──────────────────┘
        Spokes never talk to each other directly. Every cross-chain movement touches the hub.
```

**Why a hub instead of connecting every chain to every other chain ("mesh")?**

| | Hub-and-spoke | Mesh |
|---|---|---|
| Lanes to manage for N chains | N − 1 (4 chains → 3 lanes) | N × (N − 1) directed lanes (4 chains → 12) |
| Where rules live | **Once**, on the hub | Copied to every chain, must stay in sync |
| **Velocity limits** | **One global bucket.** $100k/day means $100k/day total | Each chain enforces its own $100k/day → up to **N × $100k/day** unless the chains constantly sync state (slow, expensive, fragile) |
| Audit trail | One place to read | Scattered across chains |
| Latency / fees | Spoke → spoke needs 2 hops (2× time, 2× fee) | 1 hop anywhere |
| Single point of failure | Hub down or paused → cross-chain paused | More resilient |

**The key reason for a treasury is the global limit.** Policy only means something if it's enforced in one place. That's the same reason banks run one core ledger even when they operate in many countries.

**Message types (all CCIP Programmable Token Transfers with an encoded header):**

| Type | Direction | Carries | Checked by |
|---|---|---|---|
| `PAYOUT` | hub → spoke | token + payee + paymentId | Hub: full policy before sending. Spoke: authenticity only |
| `INVOICE_PAYMENT` | spoke → hub | token + invoiceId + payer | Hub: sender allowlist, amount range, invoice match → escrow |
| `REFUND` | hub → spoke | token + original payer + invoiceId | Hub: only from a Rejected escrow entry |
| `FLOAT_TOPUP` (roadmap) | hub → spoke | token + new spoke allowance | Hub: policy (counts as treasury movement) |

**Worked flows**

1. **Pay a vendor on Base (1 hop, MVP).**
   Operator proposes "pay Vendor V 5,000 USDC on Base" → PolicyEngine checks: V allowlisted for *(Base, address)*, the $5k fits the per-tx cap, the global bucket has $5k + CCIP fee available, approvals if needed → Safe calls `ccipSend` with `PAYOUT` → about minutes later, Base SpokeGateway checks "came from hub router, from our Safe" → transfers 5,000 USDC to V.

2. **Customer on Arbitrum pays invoice #123 (1 hop, roadmap chain).**
   Customer calls `payInvoice(123)` on the Arbitrum SpokeGateway (optional pre-check: invoice exists, amount in range; the payer pays the CCIP fee) → `INVOICE_PAYMENT` → hub InvoiceReceiver **never reverts** → escrow entry *Pending* → clearance (invoice match, sender allowed for Arbitrum, compliance hook) → *Cleared* → the Safe pulls the funds (pull-over-push).

3. **Rejected payment refund (1 hop back).**
   Same as flow 2, but the payer isn't allowlisted → escrow *Flagged/Rejected* → an approver triggers a refund → hub sends `REFUND` to the Arbitrum spoke → back to the payer. The refund's CCIP fee is paid by the treasury or deducted from the refund (a policy choice, R4).

4. **Customer pays on Arbitrum, supplier must be paid on Base (spoke → hub → spoke, 2 hops).**
   Flow 2 lands the money on the hub; after clearance, flow 1 pays out to Base. There is deliberately **no direct Arbitrum → Base shortcut**: the hub is where clearance and limits happen. Cost: two fees and two waits (minutes each). Acceptable for treasury operations, and this is the "routing through a hub" the instructor described.

5. **Spoke float for speed (roadmap).**
   Like a bank keeping cash in a foreign branch (a "nostro account"), the hub can pre-fund a small **float** on a busy spoke with `FLOAT_TOPUP`. Small local payouts on that spoke then skip the cross-chain wait, but the spoke can only spend inside the allowance the hub gave it, and every float top-up counts against the global bucket. This keeps the global limit honest while cutting latency.

6. **Rule changes.**
   Rules change **only on the hub** (with the time-lock from R3 DP6). Spokes hold almost no rules (only "which hub router/sender do I trust"), so nothing needs to be synced across chains. Adding a new spoke = deploy a SpokeGateway + add the lane + allowlist it on the hub (a loosening change → time-locked).

**Where the money lives: two models (clarified after Pops' assumption check)**

| | Model A — funds on hub, tokens travel with the approved message (**MVP**) | Model B — funds on spokes, hub sends command-only messages |
|---|---|---|
| How | Hub validates → sends tokens + instruction together | Each spoke has a vault; hub validates → sends "pay X to Y" (no tokens); spoke vault executes |
| Pros | Spokes hold no money, so a forged message can't drain anything there | No bridging per payment (cheaper and faster once funded) |
| Cons | Every payout bridges tokens | Each spoke vault holds money and **trusts hub messages**. Messaging security becomes critical, so spoke-side rate limits are needed |
| Our use | MVP | Partly via the "float" (flow 5) as a roadmap hybrid |

**Validation principles (summary)**
1. **Policy** decisions (limits, allowlists, approvals, clearance): **hub only**.
2. **Authenticity** checks (right router, right source chain, right hub sender): **every chain**. Otherwise anyone could trigger a spoke payout.
3. **Outgoing** payments are **pre-approved** on the hub, then executed. **Incoming** payments are initiated by third parties on a spoke and can't be pre-approved, only **quarantined on arrival** (escrow) and cleared or refunded afterwards.
4. Policy upgrades happen in one place (the hub). Spoke gateways stay minimal and are only redeployed for message-format or bug-fix changes, so keep a versioned message header.

**Identity is (chain, address), never just address.**
The same address can belong to different people on different chains, especially for smart-contract wallets. Real example: in 2022 Wintermute lost 20M OP tokens that were sent to its Safe address on Optimism, where that Safe hadn't been deployed yet; an attacker deployed a contract at that address and took the tokens. **L** (widely reported) → every allowlist entry and every spoke trust setting is keyed by **(chainSelector, address)**.

**Choosing the hub on mainnet (roadmap).**
Sepolia as the hub is fine for the testnet MVP. On mainnet, Ethereum L1 gas is expensive, so an institution might pick an L2 (e.g., Base or Arbitrum) or its own appchain/L2 as the hub. The instructor mentioned teams building their own L2s. If the hub moves to an L2, the engine must add the **L2 sequencer uptime check** for price feeds (R3 DP2). The contracts don't change; only config does.

---

## 6. Supported token matrix (input to R7 / D3)

| Token | Cross-chain on Sepolia ↔ Base Sepolia (CCIP) | Chainlink price feed | Free testnet source | Status |
|---|---|---|---|---|
| USDC | ✅ L (CCTP-backed, confirm lane) | USDC/USD **L** (check in R7) | **Circle faucet: 20 USDC / 2h / address / chain**, Sepolia + Base Sepolia **V** | Primary stablecoin if the lane is confirmed |
| CCIP-BnM | ✅ V (all testnets) | ❌ → fixed-price mock feed ($1) | `drip()` on the token contract, 1 token per call **V** | Fallback demo token |
| LINK | ✅ L | LINK/USD **L** | Chainlink faucet (faucets.chain.link) **V** exists; amount/eligibility **U** | Optional second priced asset |
| ETH | Native (fees) | ETH/USD **L** | Sepolia / Base Sepolia faucets | Gas + fees |

→ Final list set in R7: (bridgeable) ∩ (has a price feed).

---

## 7. Recommendation: **Chainlink CCIP**, hub-and-spoke with policy decided on the hub — **approved (D11)**

### Reasoning
1. **Best fit for the receiving-side differentiator:** tokens + invoice data arrive **atomically** in one message (§9.1).
2. **Real USDC** via CCIP's CCTP integration (lane to be confirmed), and free testnet USDC from Circle's faucet.
3. **One coherent Chainlink stack** for the bootcamp's oracle requirement, with a swappable adapter to limit lock-in (§5.4, §9.3).
4. **Security posture fits institutions:** RMN and per-token rate limits out of the box; LayerZero puts DVN configuration on us.
5. **Your weights favour it robustly** (4.74 vs 3.88).
6. **Learning risk is manageable:** Chainlink Local + M13.

### Conditions / what would change the recommendation
- **The instructor prefers LayerZero** → switch (design carries over; add DVN config + Composer escrow).
- **USDC not enabled on the Sepolia ↔ Base Sepolia CCIP lane** → CCIP-BnM with a mock $1 feed; USDC via CCTP V2 Hooks as a Should.
- **The CCIP spike fails in dev week 1** → single-chain MVP (D10 fallback).
- **The treasury must hold USDT / PYUSD** → add a LayerZero adapter (roadmap, §5.4).

### Spike plan (dev week 1, ~half day)
1. Foundry + Chainlink Local: Programmable Token Transfer (CCIP-BnM + invoiceId) to a receiver that never reverts and records escrow.
2. Real testnet: the same from Sepolia → Base Sepolia; measure latency and fee.
3. Check the USDC lane; try a USDC transfer with faucet USDC.
4. Exit: works end-to-end → keep; otherwise → fallback above.

---

## 8. Risks, unknowns, revisit triggers
- **USDC lane availability (L)** → verify in the CCIP Directory.
- **Stuck messages:** mitigated by the never-revert receiver + escrow + manual-execution runbook in the backend (R11: monitor and alert).
- **Guard decoding `ccipSend`:** an extra payment "shape" (R3 DP7).
- **Fee accounting:** CCIP fees count toward limits (R3/R6).
- **Latency:** minutes, not seconds. UI status: Sent → In transit → Delivered, with a CCIP Explorer link (R10).
- **Vendor concentration** (oracle + bridge from one vendor): mitigated by adapter interfaces (§5.4, §9.3).

---

## 9. Deeper notes (Q&A from Pops' review)

### 9.1 Why "tokens + data arrive atomically" matters
- **The finance problem it solves: "unapplied cash".** When a bank wire arrives without a usable reference, finance teams can't match it to an invoice. The money sits in a suspense account until someone reconciles it by hand. It's one of the most disliked problems in treasury operations, because money is received but can't be booked.
- **Atomic (CCIP Programmable Token Transfer, CCTP V2 Hooks):** the tokens and the invoice ID arrive in the **same** transaction. Either our InvoiceReceiver gets both and records "Invoice #123: 10,000 USDC received, Pending clearance", or it gets neither. There's never money on the hub without its reference.
- **Non-atomic (LayerZero OFT + Composer):** step 1 credits tokens to the Composer; step 2 (`lzCompose`) runs the invoice logic. If step 2 fails (out of gas, bug, paused contract), tokens sit in the Composer with **no invoice record**. That's the on-chain version of unapplied cash. It's solvable (retry + "unmatched funds" state + admin tooling), but it's extra states, extra tests, extra UI.
- **Nuance:** our "never revert" rule (§2A known issues) means we rarely *use* the revert path. What we gain is the guarantee that **token and reference travel together**, which removes a whole class of reconciliation states from the design.

### 9.2 Institutional adoption: CCIP vs LayerZero (evidence)

**Chainlink / CCIP** (source: Chainlink's own announcements and press releases; "production" labels are Chainlink's, and some items are trials or pilots):
- **Coinbase** chose CCIP as the **exclusive** bridge for Coinbase Wrapped Assets (cbBTC, cbETH, cbDOGE, cbLTC, cbADA, cbXRP; ~$7B market cap), announced **2025-12-11**. **V**
- **Swift:** interoperability trials with 12+ institutions (Citi, BNY Mellon, BNP Paribas, Lloyds) using CCIP; a pilot with UBS Asset Management under MAS Project Guardian. **V** (vendor-reported)
- **Hong Kong Monetary Authority e-HKD Phase 2:** ANZ, China AMC and Fidelity International did cross-chain settlement of tokenized assets with CCIP. **V** (vendor-reported)
- **ANZ:** cross-chain delivery-vs-payment pilots with AUD/NZD stablecoins via CCIP. **SBI Digital Markets:** CCIP as its exclusive interoperability solution. **V** (vendor-reported)
- Plus non-CCIP Chainlink usage by DTCC, UBS, Euroclear and JPMorgan Kinexys (data, runtime environment), which strengthens the "one stack" story.

**LayerZero** (source: LayerZero's own blog and news coverage):
- **Tether USDT0** (omnichain USDT, 24+ chains) and **XAUT0** (tokenized gold). **V** (vendor-reported)
- **PayPal PYUSD** adopted the OFT standard (Nov 2024) and expanded to 7 more chains via LayerZero (Sep 2025). **V** (The Block, Decrypt)
- **Ethena** (USDe, sUSDe, USDtb), **Ondo Finance** (tokenized T-bills and equities). **V** (vendor-reported)
- **Fireblocks** embedded LayerZero in its Tokenization Engine (May 2025). **Fidelity** and **Worldpay/Global Payments** run DVNs (early 2026). **V** (vendor claim, not independently verified)
- **Wyoming** state stable token (FRNT) uses LayerZero. **L** (news coverage)

**Takeaway (correcting v1):**
- "LayerZero = DeFi only" was too strong. **LayerZero dominates stablecoin issuance**: USDT and PYUSD move natively via LayerZero.
- **Chainlink dominates bank / financial-market-infrastructure pilots**: Swift, central banks, tokenized funds.
- **Neither is "the financial standard"** (see §9.5). The LayerZero institutional score was raised 3 → 4; CCIP still leads for *our* use case (a bank-style treasury policy layer).

### 9.3 Does "one vendor for oracle + bridge" matter in real life?
**Yes, in both directions.**
- **Benefits (real):**
  - **Vendor risk management:** institutions run due diligence on every third party (security reviews, contracts, SLAs). One vendor is one assessment instead of two.
  - **One operational model:** shared concepts (DONs, fee handling, monitoring), one support channel, one set of status pages and incident playbooks.
  - **Less integration code:** fewer libraries and interfaces to audit.
  - **For the capstone specifically:** it shortens learning (M13 covers Chainlink), and the pitch is simpler.
- **Costs (also real):**
  - **Concentration risk:** if Chainlink has an outage or incident, both pricing and bridging are affected at once.
  - **Lock-in:** switching later costs more.
  - Many sophisticated institutions **deliberately diversify**: two oracle sources, or multiple bridges with limits per bridge.
- **Our balance:** use one vendor for the MVP, but put **interfaces** in front of both: `IPriceOracle` (R3 DP2, with the fail-closed fallback) and `ICrossChainAdapter` (§5.4). Diversification then becomes a roadmap item, not a rewrite.

### 9.4 "Less DeFi-native than LayerZero": is that bad?
- **What it means:** fewer DeFi protocols build their cross-chain logic on CCIP than on LayerZero. So there are fewer community examples of CCIP composed with DeFi actions (swap-on-arrival, cross-chain lending, etc.), and fewer DeFi tokens move via CCIP natively.
- **Does it hurt us?** Mostly **no**:
  - We don't need DeFi composability; we move stablecoins between a treasury and payees.
  - CCIP's official docs, tutorials, starter kits and Chainlink Local are thorough.
  - M13 teaches Chainlink.
- **Where it can hurt:**
  1. **Debugging help:** fewer community answers when stuck; you'll rely more on official docs and Chainlink's Discord.
  2. **Token coverage:** LayerZero-native stablecoins (USDT0, PYUSD, USDe) don't move natively via CCIP (§9.5).
- **Net:** a small cost for this project, which is why it only affected the Solo-effort score.

### 9.5 What is "the standard" for cross-chain payments?
**There isn't one.** It depends on what you're moving and who you are:

| World | De-facto standard | Notes |
|---|---|---|
| Traditional (Web2) cross-border | **Swift** messaging + **ISO 20022** message format; card networks (Visa, Mastercard); local instant-payment rails | Swift is *messaging*; money settles through correspondent banks |
| USDC | **Circle CCTP** (issuer's canonical bridge) | CCIP uses CCTP under the hood for USDC → consistent with our choice |
| USDT | **USDT0 on LayerZero** (Tether's omnichain USDT) | → why a LayerZero adapter is on the roadmap |
| PYUSD | **LayerZero OFT** | Same |
| Banks / tokenized funds / FMIs | Mostly **Chainlink CCIP** pilots (Swift, HKMA, ANZ, SBI) | Many are pilots, not full production |
| Crypto-native user transfers | **Intent-based bridges** (e.g., Across; ERC-7683 cross-chain intents) | **L**: aimed at speed and UX for users, not treasury controls |

**Practical rule used in industry:** *follow the issuer's canonical bridge* for each stablecoin, and put your own controls on top. That's exactly our architecture: the policy layer is bridge-agnostic, with CCIP (USDC via CCTP) first and LayerZero (USDT0 / PYUSD) as the next adapter.

---

### 9.6 KelpDAO vs LayerZero ($292M rsETH exploit + lawsuit) — added v3, 2026-09-26

**What happened (18 Apr 2026)** **V** (OpenZeppelin, LayerZero, Chainalysis)
- KelpDAO's rsETH bridge (an OFT Adapter holding escrowed rsETH, on the Unichain route) verified messages with a **1-of-1 DVN**: **LayerZero Labs' own DVN was the only verifier**.
- Attackers (attributed to **North Korea's Lazarus / TraderTraitor**) **poisoned RPC nodes** that the DVN relied on (swapped binaries) and **DDoS'd** the healthy ones, so the DVN only saw attacker-controlled chain data.
- The DVN then attested a **fake** message ("116,500 rsETH locked on the source chain") → the OFT Adapter released **116,500 rsETH (~$292M)** from escrow within ~80 minutes.
- Knock-on: ~89.5k rsETH was posted as collateral on **Aave** to borrow ~$190M WETH. Part of the funds was frozen by the Arbitrum Security Council. The largest DeFi exploit of 2026 so far.
- **No smart-contract bug:** every contract worked as designed. The failure was configuration + off-chain infrastructure ("$292M lost, zero bugs found", OpenZeppelin).

**The lawsuit (filed 2026-09-25)** **V** (The Block, CoinDesk)
- Evercrest Technologies (KelpDAO's developer) sued LayerZero Labs Ltd., LayerZero Labs Canada and co-founder/CEO Bryan Pellegrino in the **Supreme Court of British Columbia**.
- Claims: **negligent misrepresentation, negligence, defamation**, plus aggravated and punitive damages.
- **Kelp's case:**
  - LayerZero reviewed and **endorsed** the 1-of-1 setup over ~2.5 years and 8 integration discussions (Telegram screenshots, e.g., "No problem on using defaults either"; CoinDesk couldn't authenticate them).
  - LayerZero's own **quickstart docs and GitHub example configs** showed LayerZero Labs as the sole DVN.
  - LayerZero warned another team (USDT0) about this risk but not Kelp.
  - It was **LayerZero's** DVN infrastructure that got compromised.
  - LayerZero then publicly blamed Kelp (the defamation claim).
- **LayerZero's case:**
  - The protocol "functioned exactly as intended"; the app chose the configuration.
  - LayerZero and others had communicated **DVN-diversification best practices** to Kelp.
  - Pellegrino calls the suit "meritless".
- **Context:**
  - ~**47% of ~2,665 active LayerZero apps** used 1-of-1 DVN configs (~$4.5B exposed) in the 90 days to ~22 Apr 2026 (CoinGecko/Dune, via CoinDesk). **L**
  - A security researcher (a former LayerZero auditor) says he had reported this attack pattern and it was rejected as "not a vuln, requires all DVNs". **L**
- **Changes since:** the LayerZero Labs DVN **now refuses to sign for apps using 1/1 configurations**; the compromised RPCs were replaced. **V**

**Who is at fault?**
Undecided. It's a fresh civil case, and only a court can assign legal liability (not legal advice). On the engineering facts, responsibility looks **shared**:
- **KelpDAO:** under LayerZero's model the **app owns its security config**, and Kelp ran a single-verifier bridge holding ~$300M.
- **LayerZero:** the single verifier that failed was **LayerZero's own**. Its docs and examples made 1-of-1 the easy default, about half the ecosystem ran it, and it only banned 1/1 **after** the loss.
- **The contested question** is whether LayerZero *endorsed* the risky setup (Kelp) or *warned against* it (LayerZero). That's what the court will weigh.

**Why it matters**
1. **Defaults are security decisions.** "Configurable security" shifts risk to integrators, and most integrators keep defaults.
2. **Audits don't catch it:** contract audits usually don't review third-party configs or RPC/infra dependencies.
3. **Same attacker pattern as Bybit:** compromise off-chain infrastructure, make on-chain logic act on false inputs.
4. **Bridges are the highest-value target** in multi-chain systems.
5. **Vendor accountability** in crypto infra may now be tested in court, which matters to institutions weighing vendors.

**Effect on our design**
- **Choice unchanged: CCIP (D11).** LayerZero's score drops 3.88 → **3.78** (maturity & security 4 → 3). The gap widens.
- **Don't be complacent about CCIP.** CCIP isn't configurable per app, but it's still an external verifier network (Chainlink DON + RMN). A similar infrastructure compromise can't be ruled out. So:
  1. **Treat every bridge message as untrusted input:** hub-side authenticity checks + **policy on arrival** (incoming escrow, R4).
  2. **Cap bridge exposure:** our own per-lane inbound/outbound **rate limits** on top of CCIP token-pool limits; spokes hold little or no money (Model A, §5.5).
  3. **Monitoring + circuit breaker:** the backend watches for anomalies (volume spikes, unknown senders) and a `GUARDIAN` can pause lanes instantly (tightening = immediate, R3 DP6).
- **If a LayerZero adapter is ever added (roadmap, for USDT0/PYUSD):**
  - **≥ 2 required DVNs from independent operators** (+ optional threshold), never including the only party you also depend on elsewhere.
  - **Pin all configs** on both sides; a CI check fails on any 1/1 or default config.
  - Independent RPC providers for any self-run DVN.
  - Per-lane caps as above.
- **Pitch value:** alongside Bybit, this is a second 2026-era example that "the contracts were fine, the configuration and infrastructure failed". It's exactly the gap an on-chain policy backstop with caps, quarantine and pause is designed to limit.

## 10. Free testnet tokens

| Token | Where | Limit | Confidence |
|---|---|---|---|
| **USDC** | Circle faucet (faucet.circle.com): Ethereum Sepolia, Base Sepolia and more | **20 USDC every 2 hours, per address, per chain**; ask on Circle's Discord for more | **V** |
| **LINK** | Chainlink faucet (faucets.chain.link/sepolia, /base-sepolia) | Amount and eligibility (e.g., wallet checks) not confirmed | faucet exists **V** / limits **U** |
| **CCIP-BnM / CCIP-LnM** | Call `drip(yourAddress)` on the token contract (address in the CCIP Directory) | 1 token per call | **V** |
| **Sepolia / Base Sepolia ETH** | Chainlink faucet, Coinbase CDP faucet, others | Varies | **L** |

**Tips**
- Pay CCIP fees in **native ETH** so LINK isn't needed.
- 20 USDC per 2h is plenty for a demo if amounts are small (e.g., invoices of 1–5 USDC, with limits scaled down to match).
- For bigger demo numbers, use CCIP-BnM or a mock stablecoin on the hub, and USDC only for the cross-chain showcase.

---

## 11. Sources
- CCIP Directory — Base Sepolia: https://docs.chain.link/ccip/directory/testnet/chain/ethereum-testnet-sepolia-base-1
- CCIP Directory — Ethereum Sepolia: https://docs.chain.link/ccip/directory/testnet/chain/ethereum-testnet-sepolia
- CCIP Programmable Token Transfers: https://docs.chain.link/ccip/tutorials/programmable-token-transfers
- CCIP Manual Execution: https://docs.chain.link/ccip/concepts/manual-execution
- CCIP USDC tutorial: https://docs.chain.link/ccip/tutorials/evm/usdc
- CCIP test tokens: https://docs.chain.link/ccip/test-tokens
- Cross-Chain Token standard: https://docs.chain.link/ccip/concepts/cross-chain-token/overview
- Chainlink Local (Foundry): https://docs.chain.link/chainlink-local/build/ccip/foundry
- Code4rena CCIP contest repo: https://github.com/code-423n4/2024-11-chainlink
- Chainlink — banking & capital markets announcements: https://chain.link/blog/chainlink-banking-capital-markets-announcements
- Coinbase selects CCIP (Nasdaq press release, 2025-12-11): https://www.nasdaq.com/press-release/coinbase-selects-chainlink-ccip-exclusive-bridge-infrastructure-supercharge-coinbase
- LayerZero — The Builder Spectrum: https://layerzero.network/blog/the-builder-spectrum
- The Block — PYUSD expands via LayerZero: https://www.theblock.co/post/371321/paypal-pyusd-stablecoin-expands-blockchains-layerzero
- Decrypt — PYUSD tops $1.3B: https://decrypt.co/340290/paypals-stablecoin-1-3-billion-pyusd-expands-tron-avalanche
- Messari — LayerZero OFT and stablecoin issuers: https://messari.io/report/layerzero-scaling-stablecoin-issuers-with-the-oft-standard
- LayerZero Sepolia deployment: https://docs.layerzero.network/v2/deployments/chains/sepolia
- LayerZero Composer: https://docs.layerzero.network/v2/developers/evm/composer/overview
- LayerZero security stack / default config: https://docs.layerzero.network/v2/developers/evm/protocol-gas-settings/default-config
- LayerZero TestHelper: https://docs.layerzero.network/v2/developers/evm/tooling/test-helper
- Circle CCTP V2: https://www.circle.com/blog/cctp-v2-the-future-of-cross-chain
- Circle testnet faucet: https://faucet.circle.com/
- OpenZeppelin — Lessons from the KelpDAO rsETH exploit: https://www.openzeppelin.com/news/lessons-from-kelpdao-hack
- LayerZero — KelpDAO incident statement: https://layerzero.network/blog/kelpdao-incident-statement
- Chainalysis — Inside the KelpDAO bridge exploit: https://www.chainalysis.com/blog/kelpdao-bridge-exploit-april-2026/
- The Block — KelpDAO sues LayerZero (2026-09-25): https://www.theblock.co/news/regulation/2026-09-25-kelpdao-sues-layerzero-claims-it-endorsed-setup-used-in-292-million-rseth-exploit-416361
- CoinDesk — KelpDAO sues LayerZero and CEO: https://www.coindesk.com/business/2026/09/25/kelpdao-sues-layerzero-for-the-largest-exploit-2026-has-seen-so-far
- CoinDesk — Kelp says LayerZero approved the setup (2026-05-05): https://www.coindesk.com/web3/2026/05/05/kelp-claims-that-layerzero-approved-the-setup-it-blamed-for-usd292-million-bridge-hack
- Chainlink faucets: https://faucets.chain.link/sepolia · https://faucets.chain.link/base-sepolia