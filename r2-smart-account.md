# R2 — Smart Account Base ("custodian stand-in")

_Step 3 research, Wave 1. Researched 2026-09-23 (v2: per-option detail; v3 2026-09-24: Safe integration notes, ERC-7579 draft status, on-chain rule privacy, inputs to R3). Method: `02a-research-method.md` (gates + D7 weights). Confidence: **V** Verified in primary source · **L** Likely · **U** Unverified._

## 1. Decision & requirements served
- **Decision:** which wallet our policy layer attaches to, and **where** the policy code plugs in. One-way door: hard to change after we build on it.
- **Serves:** F1 (spend rules), F4 (works with a custodian), F7 (gas sponsorship / 4337), F8 (agent proposals), and rule types P1–P7 from R1.

### Terms used below
- **Smart account:** a wallet that is a smart contract, so it can have rules, multiple owners, plug-ins, etc.
- **Guard:** a contract the wallet calls *before* (and after) each transaction. It can say "no" and stop it.
- **Module:** a plug-in contract allowed to make the wallet send transactions without the usual owner signatures.
- **Hook** (ERC-7579 term): the same idea as a guard, but in a standard plug-in format that works across many wallet brands.
- **ERC-7579:** a standard for "modular" smart accounts. Write a plug-in once, install it on any compliant wallet.
- **EntryPoint:** the shared ERC-4337 contract that processes gasless "user operations". It has versions (v0.6, v0.7, v0.8, v0.9), and wallets, bundlers and paymasters must agree on the version.

---

## 2. Options in detail

### Option A — Safe v1.5 + our Guard + Module Guard + our Module

**What it is.** Use the standard Safe multisig. We write:
- a **PolicyGuard** that Safe calls on every transaction (`checkTransaction` / `checkAfterExecution` for owner-signed transactions, `checkModuleTransaction` for module transactions),
- a **PaymentModule** where operators and the MCP agent propose payments and approvers sign.

Both call one **PolicyEngine** holding the rules.

**Pros**
- **Most-used institutional smart wallet.** Cobo Argus (R1) builds its institutional product on Safe. Reviewers will recognise it instantly.
- **Every path is guarded since v1.5.0** (released 2025-07-22): the new *Module Guard* checks module-initiated transactions too. Reviewed by Certora and Ackee. **V**
- **Strong real-world story:** Bybit's $1.5B loss was a Safe without a restrictive guard (R1; see §7.4). We show the fix.
- **Custodian fit:** a custodian's MPC key is just a Safe owner. No integration needed.
- **4337 available** via `Safe4337Module` (Safe v1.4.1+). **V**
- Our code is **ordinary Solidity contracts** (guard, module, engine), so graded items (OZ, access control, reentrancy, tests) all live in our code.

**Cons** (details in §7)
- **Safe's own contracts are large and older-style.** You integrate with them, you don't read all of them. → §7.1
- **Two code paths** (owner-signed vs. module) must both be tested. → §7.2
- ERC-7579 plug-in portability isn't native. → §7.3

**Limitations / considerations**
- **The guard runs on every transaction**, so the rules must be cheap (constant-time lookups, no loops over lists). R3 designs this.
- **Rules are public on-chain** (R1 trade-off). Keep on-chain rules coarse. → §8
- **Safe v1.5 addresses on Base Sepolia** should be confirmed at build time. **L**
- **The Safe4337Module / EntryPoint version** pairing with bundler providers is checked in R6. Older Safe guides use EntryPoint v0.6 + module v0.2.0. **V** Newer versions are expected to support v0.7+. **L**

**Known issues & known fixes**

| Issue | Status / fix |
|---|---|
| **Guard can lock (brick) the Safe.** Safe docs: "a broken Guard can cause a denial of service for a Safe." The guard also checks the transaction that would remove it, so a guard that rejects everything can never be removed. **V** | **Our fix:** the guard always allows the owner-quorum call to `setGuard` / `setModuleGuard` after a **time-lock** delay; a minimal-checks emergency mode; an invariant test that "owners can always remove the guard after the delay". Safe also recommends auditing the guard and planning recovery. **V** |
| **Before v1.5, modules bypassed guards.** Guards only covered owner-signed `execTransaction`. **V** | **Fixed in v1.5.0** by Module Guard. We require v1.5. If it's unavailable on a testnet, PaymentModule calls PolicyEngine itself as a fallback. |
| **Bybit-style attack** (malicious UI tricks signers into a `delegatecall` that replaces the wallet's code). **V** | **Our fix:** the guard blocks `delegatecall` except to allowlisted contracts (e.g., Safe's MultiSend), and blocks changes to the Safe's own settings except via the time-locked path. This is ScopeGuard's approach. **V** |
| `setGuard` requires the guard to declare the right interface (ERC-165). **L** | Implement `supportsInterface` correctly; covered by a unit test. |

---

### Option B — Safe + Safe7579 adapter + our ERC-7579 Hook

**What it is.** Keep Safe, but install Rhinestone's **Safe7579 adapter**, which makes Safe accept ERC-7579 plug-ins. Our PolicyEngine becomes an **ERC-7579 hook module**. It's deployed through the adapter's "Launchpad" factory.

**Pros**
- **Portable:** the same hook works on other 7579 wallets (Kernel/ZeroDev, Biconomy Nexus, OZ accounts). Strong portfolio signal: ERC-7579 is the current direction of smart accounts.
- **Built-in 4337 compliance**, plus access to Rhinestone's audited module library (14 modules: social recovery, dead-man switch, etc.). **V**
- Still Safe underneath, so institutions still recognise it.

**Cons**
- **An extra layer** (adapter + launchpad) between you and Safe: more moving parts to understand, debug and test.
- **Smaller adoption** than plain Safe; fewer examples. The GitHub repo is modest (~50 stars). **V**
- **The hook API differs** from Safe's guard. Learning cost for a solo builder in a short window.
- **ERC-7579 is still a Draft standard** (§7.5).

**Limitations / considerations**
- **Deployment must go through the Launchpad** to get matching 4337 addresses. **V** More complex scripts, harder multichain deploys.
- **Tooling:** Rhinestone ModuleKit (Foundry-based) helps, but it's another toolkit to learn.

**Known issues & known fixes**

| Issue | Status / fix |
|---|---|
| Ackee audit (June–July 2024): **1 critical, 2 high, 5 medium**. The critical: an attacker can **front-run the Safe deployment via the Launchpad and take over the wallet**. The high included front-running `initializeAccount`. Code quality rated "average" (unresolved TODOs, incomplete docs). **V** | Ackee recommended fixing it immediately and protecting init functions. Fix status isn't stated on the README page we could read. **U → would need checking in the repo's audits folder before use.** |
| New tech = fewer battle-tested deployments. **L** | Mitigation: use only audited Rhinestone modules and pin versions. |

---

### Option C — OpenZeppelin custom smart account (`AccountERC7579Hooked`)

**What it is.** Build our own wallet from OpenZeppelin's new account contracts:
- `Account` (4337),
- `AccountERC7579` / `AccountERC7579Hooked` (plug-ins + hooks),
- multisig signer `MultiSignerERC7913`,
- `ERC7821` batching.

Added in **OZ v5.4.0 (2025-07-17)**. The latest is v5.6.1 (2026-02-27). **V**

**Pros**
- **Straight from the guide's wording:** "use audited libraries (OpenZeppelin)". Maximum grading signal.
- **Full control and deep learning:** you'd understand the account end to end. Great interview material.
- **Modern:** 7579 hooks, EIP-7702 signer support (`SignerERC7702`), passkeys (`SignerP256`). **V**
- Clean Foundry workflow, MIT license.

**Cons**
- **Not what institutions run.** "We built our own wallet" is exactly what the instructor warned against (don't reinvent custody). Big hit on your 20% institutional weight.
- **You own the security of the wallet itself**, not just the policy. More to test with Echidna, more risk.
- No existing web UI (the Safe web app wouldn't work with it), so the frontend must do everything.

**Limitations / considerations**
- **EntryPoint version moves fast:** v5.5.0 defaults `Account` to **EntryPoint v0.9**. **V** Bundlers and paymasters on testnet must support that version (R6 risk).
- The ERC-7579 contract files are named **`draft-`**, and OZ warns that "draft-" contracts **may have breaking changes between releases**. **V** Pin the version.

**Known issues & known fixes**

| Issue | Status / fix |
|---|---|
| `draft-AccountERC7579` API may change. **V** | Pin OZ version in `foundry.toml`. Don't upgrade mid-project. |
| v5.5.0 fix: `AccountERC7579` no longer reverts when a module's uninstall hook fails. Earlier behaviour could block removing a broken module. **V** | Use ≥ v5.5.0. |
| Audit status for the account contracts isn't stated on the release notes we read. OZ normally audits main-library releases. **L** | Check OZ's audits page before relying on it. |

---

### Option D — Zodiac Roles Modifier v2 (use existing permission system)

**What it is.** Gnosis Guild's Safe add-on. It sits between Safe modules and the Safe and enforces **role-based permissions**: which address may call which function with which parameters. It also has **allowances**: a spending budget that refills every period, with a max cap. **V**

**Pros**
- **Mature and audited:** G0 Group and Omniscia; all findings resolved as of a stated commit. **V** Used by DAOs (e.g., ENS endowment permissions). **V**
- **Very expressive** parameter conditions (e.g., "may call `transfer` only to these addresses, below this amount").
- LGPL-3.0, TypeScript SDK, subgraph, web UI already exist. **V**

**Cons**
- **Doesn't cover our differentiators:** allowances are in raw token units, **no USD/oracle pricing**, and there's **no receiving-side logic** (invoices, sender checks). **L** (by absence in docs)
- **Weak portfolio signal:** the core logic would be someone else's code. Configuring it isn't the same as building it, which hurts the "demonstrate mastery" goal.
- **Complex to configure correctly:** permission trees are powerful but easy to get wrong.

**Limitations / considerations**
- Only applies to transactions that go **through the Roles modifier** (module path). Owner-signed Safe transactions aren't covered unless you also add a guard.
- The README says "WITHOUT ANY WARRANTY". Standard, but audit scope must still be checked. **V**

**Known issues & known fixes**
- No open critical issues found in the time box. Audit findings were resolved. **V**
- **Best use for us:** as the **reference design** for our velocity limits (refill amount, period, max refill, balance) and as a competitor on the "why different" slide. Not as the base.

---

### Option E — Fully custom vault contract (no smart account)

**What it is.** One contract we write that holds the funds and has its own owners, approvals and rules.

**Pros**
- **Simplest to build and test.** Everything is in one place and fully under your control.
- Easiest Echidna/fuzz setup.

**Cons**
- **Reinvents custody:** it contradicts the instructor's direction and R1's finding that institutions use proven wallets plus policy layers.
- Custodians can't "plug in" naturally. No ecosystem (no Safe app, no 4337 tooling unless we build it).
- You carry the full security burden of a funds-holding wallet.

**Limitations / known issues**
- Every classic vault bug is yours to prevent: reentrancy, signature replay, approval race conditions, stuck funds. No external audit backing.

---

### Dropped at the gates
- **ERC-6900 (Alchemy modular accounts):** a smaller ecosystem than ERC-7579 and weaker tooling. Fails the maintenance/tooling gate for our time box. **L**

---

## 3. Hard gates

| Option | Sepolia + Base Sepolia | No business account | Foundry + viem | Maintained / audited | Permissive license | Solo-buildable (M13–16) | Result |
|---|---|---|---|---|---|---|---|
| A. Safe v1.5 + Guard/Module | ✅ L | ✅ | ✅ | ✅ Certora, Ackee | ✅ LGPL | ✅ | **Pass** |
| B. Safe + Safe7579 Hook | ✅ L | ✅ | ✅ ModuleKit | ⚠️ audited, critical fix unconfirmed | ✅ | ✅ | Pass (flag) |
| C. OZ custom account | ✅ we deploy | ✅ | ✅ | ✅ L | ✅ MIT | ✅ | Pass |
| D. Zodiac Roles as base | ✅ | ✅ | ✅ | ✅ G0, Omniscia | ✅ LGPL | ✅ | Pass (reference only, see D) |
| E. Custom vault | ✅ | ✅ | ✅ | n/a | ✅ | ✅ | Pass |
| ERC-6900 | — | ✅ | ⚠️ | ⚠️ | ✅ | — | Drop |

## 4. Weighted scoring (D7 weights)

| Criterion (weight) | A. Safe + Guard | B. Safe7579 Hook | C. OZ account | D. Zodiac Roles | E. Custom vault |
|---|---|---|---|---|---|
| Requirement fit (25) | 4 | 4 | 4 | 2 — no USD, no receiving side | 3 |
| Institutional alignment (20) | **5** | 4 | 2 | 4 | 1 |
| Grading coverage (15) | 4 | 4 | **5** | 2 — little own code | 4 |
| Portfolio & learning (12) | 4 | **5** | 4 | 2 | 3 |
| Maturity & security (10) | **5** | 3 | 3 | **5** | 2 |
| Testnet & tooling (10) | 4 | 4 | 4 | 4 | **5** |
| Solo effort & risk (8) | 4 | 3 | 3 | 4 | **5** |
| **Weighted total (/5)** | **4.30** | 3.94 | 3.57 | 3.06 | 3.01 |

**Sensitivity check:** swapping Institutional (20→12) and Portfolio (12→20) gives A 4.22 vs B 4.02. A still wins, so the ranking is robust.

---

## 5. Recommendation: Option A (Safe v1.5 + our Guard + Module), with an account-agnostic, ERC-7579-ready PolicyEngine

```
Custodian MPC key(s) / approvers ──► Safe v1.5 (owners, threshold)
                                          │
        ┌─────────────────────────────────┼──────────────────────────────┐
        │ Path 1: owner-signed Safe tx    │ Path 2: proposals via module │
        │  (execTransaction)              │  (operators, MCP agent)      │
        ▼                                 ▼                              │
  PolicyGuard.checkTransaction     PaymentModule: propose → EIP-712      │
        │                          approvals (tiered) → execTransaction- │
        │                          FromModule → PolicyGuard.checkModule- │
        │                          Transaction                            │
        └──────────────► PolicyEngine.check(Intent) ◄───────────────────┘
                          (limits, allowlist, USD via Chainlink, tx-type rules)
```

### Reasoning
1. **It matches the brief.** The instructor asked for a policy layer on top of existing custody, not a new wallet. Safe is the existing custody wallet institutions use, and R1 showed custodians connect to it as signers. A scores 5/5 on your highest non-fit weight (institutional, 20%). C and E lose most of those points.
2. **It covers every path.** Since v1.5, owner-signed *and* module transactions both hit our guard. That's what makes the "backstop that can't be bypassed" pitch true. B also covers this, but with more layers. D only covers the module path.
3. **Lowest security risk for a solo build.** Safe's core is audited and battle-tested. Our risk is limited to our own guard, module and engine, and the biggest one (guard bricking) has a known, testable fix. B has an unconfirmed critical-finding fix. C and E make you responsible for the whole wallet.
4. **It keeps grading and portfolio strong anyway.** All the graded work (OZ libraries, access control, reentrancy, pull payments, oracle pricing, Echidna invariants) sits in *our* contracts, not Safe's. Portfolio story: "I built a guard + module + policy engine that would have stopped the Bybit attack."
5. **We keep B's upside as a roadmap item.** PolicyEngine has no Safe-specific code (§7.3). Later, a thin ERC-7579 hook wrapper makes it portable. Pitch line: *"Built on Safe today; the policy engine is ERC-7579-ready."*
6. **Robust to your weights.** It wins under the sensitivity check, and the runner-up (B) shares most of the design, so switching later is cheap.

### Conditions / what would change the recommendation
- **The instructor explicitly prefers ERC-7579** → switch to B. PolicyEngine and PaymentModule carry over; only PolicyGuard becomes a 7579 hook. Before that, confirm the Launchpad critical fix.
- **Safe v1.5 isn't deployed on Base Sepolia** → deploy it ourselves from the official repo (it's permissionless), or use v1.4.1 with a transaction guard and have PaymentModule call PolicyEngine directly.
- **R6 finds no bundler/paymaster supporting Safe4337Module's EntryPoint version** → gas sponsorship moves to "Should" via a simple relayer (EIP-712 meta-transactions, M12) instead of 4337.
- **Stretch goal (dev phase, only if ahead of schedule):** build the ERC-7579 hook wrapper and demo PolicyEngine on a second account type.

---

## 6. Status
**Recommended, awaiting Pops' approval.** When approved, it replaces D4 in `00-decisions-log.md`.

---

## 7. Integration notes for Option A (explains the "Cons")

### 7.1 "Safe's contracts are large and older-style"
**What it means**
- The Safe singleton combines several managers: owners, modules, guards, fallback handler, and signature checking. Signature checking accepts four kinds: EOA signatures, contract signatures (EIP-1271), pre-approved hashes, and `eth_sign`-style signatures.
- **Proxy pattern:** every Safe is a thin proxy that `delegatecall`s one shared singleton. New Safes come from a proxy factory.
- **Written for wide compatibility, not modern style:** a broad Solidity version range, some inline assembly, owners/modules stored in custom sentinel linked lists, and a repo built around Hardhat rather than Foundry. **L**

**What it means for us**
- We only need a small set of touchpoints:
  - `execTransaction` and its parameters (to, value, data, operation, safeTxGas, baseGas, gasPrice, gasToken, refundReceiver, signatures)
  - `getTransactionHash` (the EIP-712 hash owners sign)
  - `execTransactionFromModule`
  - the Guard / Module Guard interfaces
  - `setGuard`, `setModuleGuard`, `enableModule`
- **Test setup cost:** in Foundry, deploy the Safe singleton + proxy factory in `setUp()`, or fork-test against the testnet deployments.
- **Budget:** about half a day to read these parts and get a Safe running in a Foundry test.

### 7.2 "Two code paths must both be tested"

| | Path 1: owner-signed | Path 2: PaymentModule |
|---|---|---|
| Entry | Owners sign off-chain; anyone submits `execTransaction` | Our module calls `execTransactionFromModule` |
| Guard hooks | `checkTransaction` (before) + `checkAfterExecution` (after) | `checkModuleTransaction` (+ after-execution hook, **L**) |
| Guard sees | Tx params, signatures, submitter (`msg.sender`) | Tx params and which module; no signatures |
| Initiator identity | The owners | Operator or MCP agent, tracked inside PaymentModule |

**Risks**
- **Double-counting or no counting:** the velocity limit must count each payment exactly once, whichever path it took. Rule of thumb: only PolicyEngine (called via the guard) updates spend counters. PaymentModule may *pre-check* read-only but never records spend.
- **Rule enforced on one path only** = a bypass.

**The same payment can take several "shapes"; the guard must understand each one**
- **ETH transfer:** the amount is in `value`.
- **ERC-20 transfer:** a call to the token contract; the recipient and amount are inside `data`. Decode `transfer`, `transferFrom`, `approve` (and block or cap unlimited `approve`).
- **Batches:** Safe's MultiSend runs as a `delegatecall`. The guard sees one call to MultiSend unless it unpacks the batch. **Decision for R3:** decode MultiSend batches and check each inner call, or block batching in v1.
- **Self-calls** (`to == safe`: add owner, change threshold, remove guard, enable module) → only via the time-locked admin path.
- **Unknown contract calls** → deny by default (allowlist of contracts + function selectors).

**Test plan implication:** each rule × each path × each shape, plus deliberate bypass attempts. Echidna invariant, for example: *"total USD spent in the current window ≤ limit, regardless of path or shape."*

### 7.3 "ERC-7579 portability isn't native"
- A Safe guard implements Safe's interfaces. An ERC-7579 hook implements `preCheck` / `postCheck`, install/uninstall functions, and uses 7579's own encoding for single vs. batch execution. They aren't interchangeable.
- **Mitigation (built into the design):** PolicyEngine only accepts a normalized **Intent**; each adapter translates its wallet's format into it:

```
Intent { initiator, chainId, calls[]: (target, value, data) }
        ▲                                   ▲
 PolicyGuard (Safe format)        future ERC-7579 hook (7579 format)
```

- **Cost:** decoding logic per adapter.
- **Benefit:** PolicyEngine is small, pure and easy to fuzz on its own.

### 7.4 Bybit incident (21 Feb 2025) — why guards matter
- About **$1.5B** stolen (roughly 400k ETH) from Bybit's Safe cold wallet. Publicly attributed by the FBI to North Korea's Lazarus Group. **L** (widely reported)
- **How it happened:**
  1. Attackers compromised a Safe developer's machine and got into Safe's AWS.
  2. They injected malicious code into the Safe web app, targeting only Bybit's wallet.
  3. Signers saw a normal transfer but actually signed a `delegatecall` that swapped the wallet's implementation for the attacker's.
  4. The signers blind-signed: they didn't verify the transaction on their hardware wallets.
  5. With the implementation swapped, the attacker drained the wallet. **V**
- **Lesson:** the multisig worked as designed. It checked *who* signed, not *what* was signed. A guard banning `delegatecall` to non-allowlisted targets would have blocked it. **V** (Ackee)
- **Honest framing for the pitch:** the root causes were a supply-chain/UI compromise and blind signing. A guard is one strong layer that would have stopped this specific attack; it's not a claim to prevent all hacks.
- **Pitch line:** *"A multisig checks who signed. Our guard checks what they signed."*

### 7.5 ERC-7579 is still "Draft" — why wallets use it anyway, and our stance
- **Status:** ERC-7579 is a **Draft** (created 2023-12-14); its security section still "needs more discussion". **V** By comparison, ERC-4337 (created 2021) is now **Final**, but was widely used in production long before that. **V** status / **L** timeline
- **Why wallets use it anyway:**
  - ERC status tracks the *document*, not real-world use.
  - The major account vendors (Rhinestone, ZeroDev/Kernel, Biconomy/Nexus, OKX, OpenZeppelin, Safe via adapter) agreed on it for interoperability: write a module once, run it on every compliant account.
  - Security comes from audited implementations, not the ERC's label.
- **Real risks of a draft:** the spec can change (OZ's `draft-` files may break between releases); edge-case behaviour can differ between vendors; docs and tutorials go stale quickly.
- **Our stance:** don't make it the base, but design for it (§7.3), mention it in the pitch, and treat the 7579 hook wrapper as a stretch goal.

---

## 8. Are public on-chain rules dangerous?

**Security logic: not weakened by being public.** A good system stays secure even when the attacker knows the design. A $10k/day cap still stops a $1M theft whether or not the attacker knows about it.

**What *is* exposed:**
1. **Business privacy:** the allowlist reveals counterparties (vendors, exchanges, partners); limits hint at treasury size and operating patterns. Competitors can read it.
2. **Attack planning:**
   - An attacker with a stolen key can drain *just under* the cap every day (a $100k/day cap ≈ $3M/month if nobody notices).
   - A known allowlist tells attackers which addresses to target: compromise a vendor's wallet, or **address poisoning** (look-alike addresses sent from so someone copies the wrong one).
   - Approver addresses are public, making those people phishing targets.
3. **Confidentiality obligations:** some institutions are contractually or legally required not to disclose counterparties.

**Context:** transactions are already public, so past payees are already visible. Public rules add *future intent* and exact thresholds. A real but moderate incremental leak.

**Mitigations (carried into R3):**

| Mitigation | What it does | MVP? |
|---|---|---|
| **Coarse on-chain caps** | On-chain limits set well above normal operations as a disaster backstop; tight, detailed rules stay in the custodian's private off-chain engine | Must |
| **Salted Merkle-root allowlist** | Store only a Merkle root of the allowlist; an address is revealed only when used (with a Merkle proof). The salt stops attackers hashing known addresses (e.g., exchanges) to test membership | Should (plain mapping is simpler; decide in R3) |
| **Velocity alerts (backend)** | Warn when spend approaches a cap; addresses "drain just under the limit" | Should (fits D6 backend) |
| **Address-poisoning defence** | Exact-match allowlist (never "similar" addresses); UI shows full address + label from the allowlist | Must (UI + contract) |
| **Privacy roadmap** | Privacy L2 / ZK proofs of policy compliance | Roadmap only |

---

## 9. Inputs to R3 (policy engine design)
1. PolicyEngine takes a normalized `Intent { initiator, chainId, calls[] }`; adapters (PolicyGuard now, 7579 hook later) do the decoding.
2. Only the guard path records spend; module pre-checks are read-only (no double-counting).
3. Decode ETH value, ERC-20 `transfer` / `transferFrom` / `approve`; deny unknown selectors by default.
4. MultiSend: decode and check each inner call, or block batching in v1 (decide in R3).
5. Self-calls (owner/threshold/guard/module changes) only via the time-locked admin path; the guard must always allow its own removal after the delay (anti-bricking).
6. Block `delegatecall` except to allowlisted targets (Bybit lesson).
7. Privacy: coarse caps; choose plain mapping vs. salted Merkle-root allowlist; velocity alerts in the backend.
8. Gas: O(1) checks on every transaction.

## 10. Sources
- Safe Guards: https://docs.safe.global/advanced/smart-account-guards
- Guard tutorial: https://docs.safe.global/advanced/smart-account-guards/smart-account-guard-tutorial
- Safe v1.5.0 (module guards): https://safefoundation.org/blog/introducing-safe-v1-5-0-module-guards-enhanced-smart-account-features
- Safe + ERC-4337: https://docs.safe.global/advanced/erc-4337/4337-safe
- Safe 4337 permissionless guide (EntryPoint v0.6 / module v0.2.0): https://docs.safe.global/advanced/erc-4337/guides/permissionless-detailed
- Safe + ERC-7579: https://docs.safe.global/advanced/erc-7579/7579-safe
- Safe7579 repo: https://github.com/rhinestonewtf/safe7579
- Ackee audit summary (Safe7579): https://ackee.xyz/blog/rhinestone-erc-7579-safe-adapter-audit-summary/
- OpenZeppelin Smart Accounts: https://docs.openzeppelin.com/contracts/5.x/accounts
- OpenZeppelin changelog (v5.4.0, v5.5.0, draft- warning): https://docs.openzeppelin.com/contracts/5.x/changelog
- ERC-7579 (status: Draft): https://eips.ethereum.org/EIPS/eip-7579
- ERC-4337 (status: Final): https://eips.ethereum.org/EIPS/eip-4337
- Zodiac Roles repo (audits, license): https://github.com/gnosisguild/zodiac-modifier-roles
- Zodiac Roles allowances: https://docs.roles.gnosisguild.org/general/allowances
- Ackee — Safe-native solution to the Bybit hack (ScopeGuard): https://ackee.xyz/blog/a-safe-native-solution-to-the-bybit-hack/
- NCC Group — Bybit technical analysis: https://www.nccgroup.com/research/in-depth-technical-analysis-of-the-bybit-hack/
- Check Point Research — The Bybit incident: https://research.checkpoint.com/2025/the-bybit-incident-when-research-meets-reality/