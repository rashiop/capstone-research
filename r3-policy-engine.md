# R3 — Policy Engine Design Patterns

_Step 3 research, Wave 1. Researched 2026-09-25. Inputs: R1 §3.1 (primitives P1–P9), R2 §7–§9 (integration + privacy), R5 §5 (hub enforcement, `ccipSend` decoding). Method: `02a-research-method.md`. Confidence: **V** Verified · **L** Likely · **U** Unverified._

## 1. Decisions & requirements served
R3 is a set of design decisions, not one product pick. Each decision point (DP) lists options → pros/cons → pick. The two one-way doors (DP1 limit algorithm, DP3 engine architecture) get D7 scoring.

Serves: F1, F4, F8 (agent limits), P1–P7, and the grading items (oracles, access control, invariants).

### Terms
- **Fixed window:** "$X per day", where the day resets at a set time.
- **Sliding window:** "$X in any 24-hour span".
- **Token bucket:** a budget that refills continuously (e.g., $100k/day = ~$4.17k per hour), up to a maximum.
- **Fail-closed:** when something is uncertain (e.g., price unavailable), choose the safer outcome.
- **Time-lock:** a change only takes effect after a waiting period, so others can notice and react.

---

## 2. Prior art (what existing systems do)

| System | Limit model | Notes |
|---|---|---|
| **Safe Allowance Module** | Per delegate + token: `amount`, `spent`, `resetTimeMin`, `lastResetMin`, `nonce`, packed into one storage slot. Resets after the interval, measured from the last reset. EIP-712 signed transfers by delegates | Max reset interval ~45 days (uint16 minutes). Token units, not USD. **V** |
| **Coinbase Spend Permissions** | **Fixed periods** anchored to a start time (`start`, `period`, `allowance`, `end`); unused allowance does not carry over | Chosen for predictability (subscriptions, payroll). Audited via a Cantina competition. **V** |
| **Zodiac Roles allowances** | **Refill** model: `refill` amount per `period`, capped at `maxRefill`, with a running `balance` | Close to a token bucket. **V** |
| **CCIP token pools** | **Token bucket** rate limit (capacity + refill rate) per lane | Same model as our choice → one mental model across the system. **V** |
| **Fireblocks TAP** | `amountScope` SINGLE_TX or TIMEFRAME with `periodSec`; amounts in USD/EUR/native; first-match rules → ALLOW / BLOCK / 2-TIER | Institutional reference for rule shape and outcomes. **V** (R1) |

---

## 3. Decision points

### DP1 — Velocity limit algorithm (one-way door)

| | A. Fixed window (Coinbase / Safe style) | B. Token bucket (Zodiac / CCIP style) | C. Rolling 24h via 24 hourly slots |
|---|---|---|---|
| How it works | Spent counter resets at each period boundary | `available = min(cap, available + elapsed × rate)`; spend subtracts | A ring buffer of 24 hourly totals; sum the last 24 |
| Pros | Simplest; easiest to explain ("resets at 00:00 UTC"); O(1) | O(1); **no boundary burst** (at most `cap` can go out in a short burst); smooth; matches CCIP's rate limiter | Closest to "any 24 hours" as institutions phrase it; strict bound ≈ `cap` per 24h |
| Cons | **Boundary burst:** spend the full cap at 23:59 and again at 00:00 → 2× cap in 2 minutes | Over a full 24h, up to ~2× cap can leave (a full bucket + 24h of refill); "available now" is less intuitive than "resets at midnight" | Up to 24 storage reads per check (gas); more code; hour granularity |
| Known issues | The burst is the classic flaw; an attacker with a stolen key would time it | Precision/rounding in refill math → fuzz it | Stale-slot clearing bugs are easy to write |
| **D7 score** | 3.76 | **4.35** | 4.02 |

**Pick: B, token bucket, denominated in USD (18 decimals).**
- **Exact guarantees:** (1) in any short burst, at most `cap` leaves; (2) over any window of length `t`, at most `cap + rate × t` leaves. With `rate = cap / 24h`, that's ≤ 2 × cap per 24h in the worst case (vs. 2× cap *in minutes* for a fixed window).
- If a strict "≤ cap in any 24h" is required, halve the capacity or switch the library to option C. The change stays local to one library.
- Buckets: one **global** bucket per Safe, plus **per-initiator-role** buckets (the operator and the MCP agent get their own smaller buckets). The CCIP fee counts toward the bucket.
- UI shows: "Available now: $62,400 · refills $4,167/h · full in 9h".
- Fuzz/Echidna invariant: *for any sequence of spends and time jumps, spent over any window of length t ≤ cap + rate × t*.

### DP2 — USD pricing with Chainlink (oracle requirement)

Checklist for every price read (**V**, Macro / Chainlink docs):
1. Use `latestRoundData()` (not the deprecated `latestAnswer`).
2. `answer > 0`, `updatedAt != 0`.
3. **Staleness:** `block.timestamp - updatedAt ≤ heartbeat + buffer`, **configured per feed** (heartbeats differ by feed).
4. Normalize decimals (feed decimals + token decimals → 18-decimal USD).
5. Wrap the feed call in `try/catch`.
6. Optional: deviation check against the last good price.
7. **L2 sequencer uptime feed:** only needed if the engine runs on an L2. The hub is **Sepolia (L1)**, so not needed in the MVP. Note for the roadmap if the hub moves to an L2.

**Failure behaviour: fail-closed, but not bricked.**
- Price invalid or stale → the auto-approve tier is **disabled** for that token and the payment falls to the **approval tier** (humans decide). Payments can still go out with approvals; they just can't go out automatically. Same principle as "block risky ops, allow safe ops" (**V**, Macro).
- **Conservative rounding:** round USD value **up** when checking limits.
- **Stablecoins:** price at `max(feedPrice, $1.00)` for limit checks, so a de-peg can't make spending look cheaper.
- **Tokens without a feed** (e.g., CCIP-BnM): an admin-set **fixed price** (a mock feed contract). The UI labels it "fixed price".

### DP3 — Engine architecture (one-way door)

| | A. Modular monolith | B. Pluggable policy registry |
|---|---|---|
| How it works | One `PolicyEngine` contract; rules are internal libraries run in a fixed order (tx-type → allowlist → per-tx cap → bucket → tier); one optional external **hook slot** for compliance | The engine loops over registered `IPolicy` contracts (`check(intent) → ALLOW / DENY / NEEDS_APPROVAL`) |
| Pros | Cheaper gas; no external calls in the hot path; much easier to test and to state invariants for; simple mental model | Add rules without redeploying the engine; very "platform-like"; good portfolio signal |
| Cons | Adding a new rule type = new engine version | External calls on every transaction (gas, reentrancy/DoS surface); a malicious or buggy policy can brick the guard; ordering/precedence complexity; harder Echidna setup |
| Known issues | — | Guard bricking risk multiplied by N policies (R2 §2A) |
| **D7 score** | **4.33** | 3.94 |

**Pick: A, modular monolith.** Pluggable registry goes on the roadmap (it pairs naturally with the ERC-7579 wrapper).

### DP4 — Decision outcomes and approval tiers

The engine returns `ALLOW | NEEDS_APPROVAL(tier) | DENY` (the Fireblocks-style outcome set).

| Initiator path (R2) | ≤ auto limit | ≤ tier-2 limit | > tier-2 limit | Always denied |
|---|---|---|---|---|
| **Module path:** operator / MCP agent (PaymentModule) | ALLOW (agent has a smaller auto limit) | NEEDS_APPROVAL: K-of-N approver **EIP-712 signatures** collected off-chain, submitted with the execution | DENY on this path → must go through the owners (path 1) | Non-allowlisted payee, forbidden call type, bucket exhausted |
| **Owner path:** Safe owners (custodian MPC key(s)) | ALLOW | ALLOW (already a multisig decision) | ALLOW up to the **hard cap** (backstop) | Above the hard cap, `delegatecall` to non-allowlisted targets, self-calls outside the time-locked admin path, bucket exhausted |

- **Approvals:** EIP-712 typed data `{ proposalId, intentHash, chainId, nonce, deadline }`, verified with OZ `SignatureChecker`. That supports both EOA and smart-contract approvers (EIP-1271).
- **Agent (D5):** propose-only means the agent's role has **auto limit = 0**. Every agent proposal needs human approval but must still pass all rules first.
- **Hard cap** = the coarse, public backstop (R2 §8). It sits well above normal operations.

### DP5 — Allowlist (address book) storage and privacy

| | A. Plain mapping + activation delay | B. Salted Merkle root | C. Hashed-key mapping |
|---|---|---|---|
| How | `payee[chainSelector][address] = {active, activatesAt, labelHash}` | Store one root; each payment supplies a proof and the entry's secret salt | `allowed[keccak(salt, chain, addr)]` |
| Privacy | None: the list is public | **Hidden until used**. Each entry is revealed only when first paid | Weak. The salt must be on-chain to check, so anyone can test candidate addresses |
| Pros | Simple UX (add/remove one payee); easy events for the audit trail; easy to index | Real privacy gain; nice portfolio item | Looks private, isn't |
| Cons | Reveals counterparties | Every change = new root + off-chain tree management; proofs in every payment; harder UX | False sense of security |

**Pick: A for the MVP, B as a Should / stretch; reject C.** Honest note for the pitch: on a public chain, real privacy needs Merkle/ZK or off-chain rules. Coarse on-chain caps plus the custodian's private engine is the institutional answer (R1, R2 §8).

**Activation delay** (institutional practice, and a defence against a compromised admin): a **new payee becomes usable only after a cool-down** (e.g., 24h on mainnet; minutes for the demo). Removal is **immediate**.

### DP6 — Rule-change governance ("separation of duties", P7)
- **Asymmetric changes:** making rules **stricter** (lower a limit, remove a payee, pause) is **immediate**. Making rules **looser** (raise a limit, add a payee, add a contract target) goes through a **time-lock** (reuse the M07 Timelock knowledge; OZ `TimelockController` or a built-in delay).
- **Roles** (OZ `AccessControlDefaultAdminRules`, 2-step admin transfer with delay, M05):
  - `POLICY_ADMIN` (the Safe itself, via owner quorum)
  - `GUARDIAN` (can **pause** instantly; can't unpause alone)
  - `OPERATOR` (proposes via the module)
  - `AGENT` (propose-only)
  - `APPROVER` (signs tiers)
- **Emergency:** `pause()` → only owner-path transactions below a minimal cap plus admin recovery are allowed. The **anti-bricking guarantee** (R2): the time-locked removal of the guard is always allowed.

### DP7 — Transaction "shapes" the engine must understand (from R2 §7.2, R5 §5.2)

The adapter (PolicyGuard) normalizes calls to `Intent { initiator, chainId, calls[]: (target, value, data, operation) }`. The engine then classifies each call:

| Shape | Handling | MVP |
|---|---|---|
| ETH transfer (`value > 0`, empty data) | Payee allowlist + USD price of ETH | Must |
| ERC-20 `transfer` / `transferFrom` | Decode recipient and amount; token must be supported (has a price source) | Must |
| ERC-20 `approve` | Only to allowlisted spenders (e.g., the CCIP Router); amount ≤ the payment being made; **no unlimited approvals** | Must |
| CCIP `ccipSend` | Decode destination chain selector, receiver, token amounts and fee; payee allowlisted **for that chain**; the fee counts toward the bucket | Must |
| Self-call (`to == safe`: owners, threshold, guard, modules) | Only via the time-locked admin path | Must |
| `delegatecall` | Deny, except to an allowlisted MultiSendCallOnly (if batching is enabled) | Must |
| MultiSend batch | **v1: blocked.** Should: allow only `MultiSendCallOnly` and check each inner call | Must (block) / Should (decode) |
| Any other contract call | Deny by default; allowlist of `(target, selector)` pairs | Must |

- Cap `calls.length` (e.g., ≤ 10) to bound gas.
- Unknown selector → DENY.

### DP8 — Spend accounting with Safe's execution model
- In Safe, a failing inner call **doesn't necessarily revert** `execTransaction`: it can return `success = false` and emit a failure event. **L** (known Safe behaviour when `safeTxGas`/`gasPrice` are non-zero; confirm in the spike)
- **Pattern:** reserve spend in the pre-check (`checkTransaction` / `checkModuleTransaction`), then in the after-check, **refund the bucket if `success == false`**.
- Only the guard path records spend. The module's pre-checks are read-only (R2 §9). This rules out double-counting.
- Echidna invariant: *bucket usage equals the sum of successful payments' USD value (within the rounding bound)*.

### DP9 — Gas budget
- Every check is O(1) per call, with at most one oracle read per distinct token.
- `calls.length` is capped.
- No loops over payee lists (mapping lookups only).
- Target: < 60k extra gas per simple transfer. **U**, to be measured in dev (Foundry gas snapshots) and shown in the README.

---

## 4. Resulting v1 rule set (maps R1 primitives → engine)

| R1 primitive | Engine rule | Where configured |
|---|---|---|
| P1 Allowlist | Payee `(chain, address)` with activation delay | DP5 |
| P2 Per-tx threshold (USD) | Per-tx cap per initiator role | DP4 |
| P3 Velocity (USD) | Token buckets: global + per role | DP1 |
| P4 Tiered approvals | ALLOW / NEEDS_APPROVAL(K-of-N EIP-712) / DENY | DP4 |
| P5 Initiator roles | OPERATOR, AGENT, APPROVER, GUARDIAN, POLICY_ADMIN | DP6 |
| P6 Tx-type restrictions | Shape classifier; deny-by-default | DP7 |
| P7 Policy-change governance | Asymmetric time-lock, pause, 2-step admin | DP6 |
| P8 External check hook | One hook slot (outgoing: optional; incoming: R4) | DP3 |
| P9 % of balance | Roadmap | — |

## 5. Invariants to hand to R9 (Echidna / Foundry invariant tests)
1. No payment executes to a payee that isn't active in the allowlist for that chain.
2. Bucket balance never exceeds capacity; USD spent over any window of length t ≤ cap + rate × t.
3. Module-path payments above the auto limit never execute without K valid, unused, unexpired approver signatures.
4. No `delegatecall` executes except to allowlisted targets.
5. A loosening rule change never takes effect before its time-lock expires; tightening is immediate.
6. The owner quorum can always remove or replace the guard after the delay (anti-bricking).
7. A stale or invalid price never results in an auto-approved (ALLOW) outcome.
8. Failed inner calls don't consume the bucket (refund path).

## 6. Risks, unknowns, revisit triggers
- **Safe failure semantics (DP8, L):** confirm in the dev-week-1 spike.
- **Token bucket UX and the 2× bound:** needs clear UI copy. If the instructor or users strongly prefer "resets at midnight" or a strict 24h cap, switch to option A or C (local to one library).
- **Oracle availability on Sepolia** for chosen tokens → R7.
- **Gas target (U):** measure early; if too high, drop per-role buckets to global-only.
- **Rule precedence bugs:** a fixed evaluation order plus unit tests per rule and per shape.

## 7. Status
Recommendations for DP1 (token bucket), DP3 (modular monolith), DP4 (tiers), DP5 (mapping + activation delay) and DP6 (asymmetric time-lock) are **awaiting Pops' approval**.

## 8. Deeper notes (Q&A from Pops' review, 2026-09-25)

### 8.1 Oracle terms: heartbeat, deviation, `latestRoundData`
- A Chainlink feed writes a new price on-chain when **either** trigger fires. **V** (Ackee)
  - **Deviation threshold:** the off-chain price moved more than X% since the last update (e.g., 0.5%).
  - **Heartbeat:** a maximum time between updates (e.g., 1 hour), even if the price didn't move.
- So on a calm day, a feed may only update once per heartbeat. **"Stale"** = older than the heartbeat (plus a small buffer). Something is wrong: nodes are down, the network is congested, or the feed is deprecated.
- **Staleness check:** `if (block.timestamp - updatedAt > heartbeat + buffer) → stale`.
  - Set it **per feed**, because heartbeats differ (e.g., 1h for one feed, 24h for a stablecoin feed). **L**, check each feed's page on Sepolia in R7.
  - Too tight (below the heartbeat) → false alarms that block payments. Too loose → accepts old prices. **V** (Ackee example: 1h heartbeat → ~90 min threshold)
- **`latestRoundData()` vs `latestAnswer()`:** `latestAnswer` returns only the price, with no timestamp, so you **can't** check staleness. It's deprecated. `latestRoundData` returns `(roundId, answer, startedAt, updatedAt, answeredInRound)` → check `answer > 0` and `updatedAt` freshness. **V**
- **Deviation check (optional):** compare the new price to the last good price we stored. If it jumped more than, say, 20% in one read, treat it as suspicious → no auto-approval. This protects against a broken or manipulated feed. Chainlink aggregators also have `minAnswer`/`maxAnswer` bounds, and a price stuck at a bound during a crash is a known edge case. **V**

### 8.2 Stablecoin pricing: `max(feedPrice, $1.00)`
- USD value used for limits = `amount × price`.
- **The problem:** if USDC de-pegs to $0.90, sending 111,111 USDC is "worth" $100k. A $100k limit would let 11% more tokens out than intended. Worse, a manipulated or broken feed that reads $0.10 would let 10× the tokens out.
- **The fix:** for limit checks, value stablecoins at **at least $1.00**. If the feed says $1.02, use $1.02 (still conservative); if it says $0.90, use $1.00.
- **Effect:** a de-peg can only make limits **tighter** in real terms, never looser. That's fail-safe. It matches accounting practice: the treasury books stablecoins at face value, and invoices are denominated in USD.

### 8.3 What "bricking" means
- From "turning a device into a brick": a contract ends up in a state where it **can no longer do anything useful and can't be fixed**.
- For us: if PolicyGuard reverts on every transaction (a bug, a dead oracle it requires, a paused dependency), Safe can't execute **any** transaction, **including the one that removes the guard**. Funds are stuck forever.
- Prevention (from R2): the guard **always** allows the time-locked "remove/replace guard" admin call; oracle failure degrades to the approval tier instead of reverting; invariant test: "owners can always remove the guard after the delay".

### 8.4 DP3 — architecture best practice and how to avoid plugin-registry issues
**Best-practice pattern: "fixed core + constrained extensions".**
- **Core rules** (allowlist, caps, buckets, tx-type, tiers) are in the engine: internal libraries, fixed evaluation order, pure/view logic separated from storage. They can't be removed at runtime.
- **Extensions** (compliance/invoice hooks, future custom rules) are external, but **constrained**.

**Known issues of plugin registries and how to avoid each:**

| Issue | What goes wrong | Mitigation |
|---|---|---|
| **DoS / bricking** | One plugin reverts or burns all gas → every transaction fails | Call with a **gas cap**; `try/catch`; a failing plugin → **NEEDS_APPROVAL** (not revert) on the module path; max N plugins |
| **Fail-open bypass** | A malicious or buggy plugin returns ALLOW for everything | Combine results with **deny-overrides** (any DENY wins; ALLOW from a plugin can never override a core DENY); core rules always run first |
| **Reentrancy** | A plugin calls back into the engine or the Safe mid-check | Call plugins with **`staticcall`** (read-only: can't change state or re-enter with effects) |
| **Unauthorized plugin** | A compromised admin adds an evil plugin | Adding a plugin = a **loosening** change → **time-locked** (DP6); optional allowlist of audited implementations |
| **State inconsistency** | A plugin records state in the pre-check, then the transaction fails | Plugins are view-only; only the core records spend, with the refund-on-failure path (DP8) |
| **Precedence confusion** | Unclear which rule wins | A fixed, documented order + deny-overrides + a unit test per combination |

**MVP application:** monolith core + **one** extension slot (the compliance/invoice hook) that follows all the rules above. A multi-plugin registry is roadmap work, and this pattern already makes it safe to add later.

### 8.5 DP4 — tiers explained with numbers (demo values; configurable)

| Term | Meaning | Example |
|---|---|---|
| **Auto limit** | Max per payment that can execute with **no extra approval** (still must pass all rules) | Operator: $1,000 · MCP agent: **$0** |
| **Tier-2 limit** | Max per payment the **module path** can execute **with K-of-N approver signatures** | $50,000 with 2-of-3 approvers |
| **Owner tier** | Above tier-2, only the **Safe owners** (the custodian's MPC key quorum) can execute | $50,001 – $500,000 |
| **Hard cap** | The absolute max **per transaction for anyone**, including owners. The coarse public backstop (R2 §8). Raising it = a loosening change → time-locked | $500,000 |
| **Buckets** | Velocity limits over time (DP1), applied **on top of** all of the above | Global $1M/day; operator $20k/day; agent $5k/day |

**Walk-through**
- **$800 by the operator** to an allowlisted vendor → ALLOW (≤ auto, bucket OK).
- **$800 proposed by the MCP agent** → NEEDS_APPROVAL (agent auto = $0) → K-of-N approvers sign (K is configurable per role; e.g., 1-of-3 for agent payments under $1k) → executes.
- **$30,000 by the operator** → NEEDS_APPROVAL: 2-of-3 approvers sign the **exact intent hash** (EIP-712) → executes.
- **$120,000** → DENY on the module path → must be submitted as an owner-signed Safe transaction → allowed (≤ hard cap), still counted in the global bucket.
- **$700,000** → DENY for everyone. Split into smaller payments? The global bucket still caps the total per day.

**Approval rules (maker-checker / "four-eyes principle" from banking):**
- The approver must hold `APPROVER` and **can't be the proposer**.
- K **distinct** approvers.
- Each signature covers the **exact `intentHash`** (payee, chain, token, amount, nonce, deadline). Changing anything invalidates it, which is the Bybit lesson: sign exactly what executes.
- Nonce single-use; deadline enforced.
- Signatures are collected off-chain by the backend and submitted in one execution.

### 8.6 DP5 — can we hide the allowlist permanently?
**Key point first: hiding the rules isn't enough if the payment itself is public.** The moment you pay a vendor, the transfer (to, amount, token) is visible on-chain. So on a public chain, the best any "private allowlist" can do is **"hidden until used"**. Permanent hiding needs **private payments**, not just private rules.

| Option | Hides | Status / fit |
|---|---|---|
| **Normal rollups** (Base, Arbitrum, Optimism) | **Nothing.** Their data is public (posted to Ethereum) | ❌ Not a privacy tool |
| **Privacy L2: Aztec** | Private state, balances and calls | Alpha mainnet **launched 2026-03-31, with known critical vulnerabilities while audits continue**. **V** Contracts are in **Noir**, not Solidity. Safe/CCIP support not available as far as we know **U** → ❌ capstone, ✅ roadmap |
| **Permissioned chain / validium / own L2** (like bank networks, e.g., JPMorgan Kinexys) | Everything, from the public | Privacy comes from **trusting the operator**. A roadmap pitch point ("institutions can run the hub on a permissioned L2"); CCIP has enterprise private-transaction offerings (ANZ pilot) **V** vendor-reported |
| **ZK proof of membership** (Noir/Circom circuit + Solidity verifier) | Which list entry matched (but the payment still reveals the payee) | Heavy; little extra gain over Merkle for us → roadmap |
| **Stealth addresses (ERC-5564)** | Links the payee's identity to the receiving address | Requires the payee's cooperation → roadmap |
| **Salted Merkle root** | Unused entries (hidden until first used) | ✅ Feasible (below) |
| **Off-chain rules at the custodian** | Everything detailed | ✅ Already the design (R1) |

### 8.7 Is the salted Merkle root much harder? (with a 1-month build)
**Moderately harder, and doable in about 4–5 days** if planned as a Should after the Musts.

| Piece | Work | Estimate |
|---|---|---|
| Contract | `MerkleProof.verify` (OZ); leaf = **double-hashed** `keccak256(bytes.concat(keccak256(abi.encode(chain, payee, salt))))`, OZ's standard to prevent second-preimage attacks **V** | ~1 day |
| Backend | Store entries + **secret salts** in the DB; build the tree with `@openzeppelin/merkle-tree` (`StandardMerkleTree`) **V**; generate proofs; propose new roots | ~1.5 days |
| UI | Address-book screen manages entries; shows "pending root update" | ~1 day |
| Tests | Proof valid/invalid, second-preimage, revoked leaf, root time-lock | ~1 day |

**Three design problems the Merkle version creates (and fixes):**
1. **The contract can't tell if a new root adds or removes entries** (the root is opaque) → breaks asymmetric governance (DP6).
   **Fix:** every root update is treated as loosening (time-locked), plus a separate **revocation list** (`revoked[leafHash] = true`) for **immediate** removals.
2. **Owner-path transactions can't easily carry a proof** (the Safe guard only sees the transaction).
   **Fix: "reveal-then-pay":** a one-time `revealPayee(leaf, proof)` call marks the entry as revealed (it goes public at that moment). After that, payments check a simple mapping.
3. **Activation delay per entry becomes per root update.** Acceptable: new payees are batched into the next time-locked root.

**Recommendation unchanged:** plain mapping for the MVP (Must). Salted Merkle + reveal-then-pay + revocation list as a **Should in week 3–4** if the Musts are done. It's a strong portfolio item, and a pitch line: "hidden until used; full privacy on a privacy L2 is on the roadmap."

### 8.8 DP6 — what "asymmetric changes" actually means
Not "strict first, then gradually looser". It means **the delay depends on the direction of the change**:

| Direction | Examples | When it takes effect |
|---|---|---|
| **Tightening** | Lower a limit, remove a payee, pause, add a blocked target | **Immediately** |
| **Loosening** | Raise a limit or the hard cap, add a payee, add a contract target, add a plugin, new Merkle root | **After a time-lock** (e.g., 48h mainnet; minutes in the demo) |

**Lifecycle of a loosening change**
1. **Propose:** the admin quorum queues "raise the operator auto limit to $5k". An event is emitted, the dashboard and alerts show it, with an ETA.
2. **Wait (veto window):** during the delay, a `GUARDIAN` or owners can **cancel** it. This is the "halt if something goes wrong" mechanism: a hacked admin's change can be stopped before it's live.
3. **Execute:** after the ETA, anyone can execute it → it takes effect.
4. **Revert after it's live:** just apply the opposite (tightening) change, which is **immediate**. Or `pause()`.

**Why it works:** an attacker who compromises an admin key wants to **loosen** rules to steal. Every loosening change is visible and delayed, giving humans time to react. Tightening can never help an attacker steal, so there's no reason to delay it.

**Implementation note:** the contract must know the direction, so use **separate functions per direction** (`lowerLimit` / `raiseLimit`, `addPayee` / `removePayee`) rather than one generic `setLimit`. Reuses the UdraDAO Timelock pattern (M07).

### 8.9 DP3 — what does industry actually do? (monolith vs registry)
There's no single winner; real systems split along the same line.

| System | Style | Relevance |
|---|---|---|
| **Aave v3** | Monolith: `Pool` + internal logic libraries | DeFi's reference for "one core, split into libraries" **L** |
| **Zodiac Roles v2** | **Data-driven monolith:** one contract; permissions are *configuration*, not new code | Rules are flexible without plugins **V** (R2) |
| **OZ Governor** | **Compile-time modularity:** features added by inheriting extensions, deployed as one contract | Modular code, monolithic deployment **L** |
| **Cobo Safe / Argus** (institutional, on Safe) | **Pluggable "Authorizers"** with `preExecCheck` / `postExecCheck` (+ optional process hooks), each declaring via a `flag()` which phases it uses | An institutional product that chose plugins **V** |
| **Safe guards/modules, ERC-7579** | Pluggable at the **account** level | Our layer is itself a plugin **V** |
| **ERC-7484 module registry** (Draft) | Plugins are only used if **N trusted attesters** vouch for them on-chain | The standard answer to "how do you trust plugins?" **V** |
| **Uniswap v4 hooks** | Fixed core + **constrained** plugins (each hook declares its allowed callbacks up front) | "Constrained extensions" in a major protocol **L** |

**Consensus pattern: small trusted core + constrained, vetted extensions.**
- Security-critical invariants (limits, allowlist, anti-bricking) live in a **core that can't be unplugged**.
- Extensibility comes from **data/config first** (Zodiac-style), **compile-time modules second** (OZ-style), and **runtime plugins only where extensibility is the product** (Cobo, 7579), always with constraints (static calls, gas caps, deny-overrides, time-locked install, optional attestation).

**Refined recommendation for DP3: option A+ ("hybrid").**
- **Core:** the modular monolith (unchanged).
- **Extension point:** a Cobo-like `IPolicyHook` interface: `preCheck(intent) → ALLOW | NEEDS_APPROVAL | DENY` + optional `postCheck`, and a `flags()` bitmask declaring which phases it uses. **Max 3 hooks**, installed via time-lock, called with `staticcall` + gas cap, and combined with deny-overrides (§8.4).
- **MVP hook:** the compliance/invoice check (R4). Roadmap: ERC-7484-style attestation check before installing a hook.
- **Extra effort vs. a single slot:** about 1 day (a loop over ≤ 3 hooks + tests).

| | A. Modular monolith | B. Pluggable registry | **A+ Hybrid** |
|---|---|---|---|
| D7 score | 4.33 | 3.94 | **4.72** |
| Why A+ scores higher | — | — | Keeps A's security and testability; adds the institutional pattern Cobo uses (fit + institutional + portfolio) for ~1 day extra |

## 9. Sources
- Safe Allowance Module README: https://github.com/safe-fndn/safe-modules/blob/47e2b486b0b31d97bab7648a3f76de9038c6e67b/allowances/README.md
- Safe Allowance Module contract: https://github.com/safe-fndn/safe-modules/blob/main/modules/allowances/contracts/AllowanceModule.sol
- Coinbase Spend Permissions accounting: https://github.com/coinbase/spend-permissions/blob/main/docs/SpendPermissionAccounting.md
- Coinbase Spend Permissions repo: https://github.com/coinbase/spend-permissions
- Zodiac Roles allowances: https://docs.roles.gnosisguild.org/general/allowances
- CCIP Cross-Chain Token (rate limits): https://docs.chain.link/ccip/concepts/cross-chain-token/overview
- Macro — How to consume Chainlink price feeds safely: https://0xmacro.com/blog/how-to-consume-chainlink-price-feeds-safely/
- Chainlink Data Feeds API reference: https://docs.chain.link/data-feeds/api-reference
- Fireblocks policy structure: https://developers.fireblocks.com/reference/configure-transaction-authorization-policy
- Ackee — Chainlink Data Feeds, security researcher's perspective (heartbeat, deviation, min/max): https://ackee.xyz/blog/chainlink-data-feeds/
- The Defiant — Aztec alpha mainnet (2026-03-31, known critical vulnerabilities): https://thedefiant.io/news/blockchains/aztec-launches-alpha-network-ethereum-s-first-l2-for-private-smart-contracts
- Aztec docs: https://docs.aztec.network/
- OpenZeppelin merkle-tree library: https://github.com/OpenZeppelin/merkle-tree
- RareSkills — Merkle tree second preimage attack: https://rareskills.io/post/merkle-tree-second-preimage-attack
- Cobo Safe — Authorizer overview: https://www.cobo.com/developers/v1/overview/smart-contract-wallet/en/4_1_authorizer
- Cobo Safe repo: https://github.com/CoboGlobal/cobosafe
- ERC-7484 (module registry, Draft): https://eips.ethereum.org/EIPS/eip-7484