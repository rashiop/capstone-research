# R9 — Security, Threat Model & Testing Toolchain

_Step 3 research, Wave 2. Researched 2026-09-27. Inputs: R2 §7 (Safe paths, bricking), R3 §5 + §8 (invariants, hooks), R4 §4 (escrow invariants), R5 §9.6 (KelpDAO), R7 (oracles). Guide requirements: access control, reentrancy protection, pull-over-push, oracles, Slither; unit / E2E / fork / fuzz / property-based (Echidna) tests. Confidence: **V** Verified · **L** Likely · **U** Unverified._

## 1. Decisions & requirements served
- **Decisions:** (DP1) upgradeability strategy (one-way door, scored); (DP2) toolchain + CI; (DP3) test plan structure.
- **Outputs:** threat model, a consolidated invariant list, a test plan skeleton.
- **Timing:** M14 (Slither, Echidna, Advanced Foundry) arrives mid-build, so this plan is written **now** and set up in **dev week 1** (R1 risk table).

---

## 2. Threat model

**Assets:** treasury funds in the Safe (hub); escrowed incoming payments; policy configuration; approver/admin keys; bridge lanes.
**Trust boundary:** the Safe owner quorum (the custodian's MPC keys) is ultimately trusted. Everything else is **not**: operators, the MCP agent, approvers (individually), the UI, oracles, bridges, payers.

| # | Threat | Real-world precedent | Mitigation in our design | Where |
|---|---|---|---|---|
| T1 | Stolen **operator** key drains funds | — | Per-role auto limit, per-tx cap, per-role + global token buckets, allowlist with activation delay | R3 DP1/DP4/DP5 |
| T2 | **MCP agent** tricked (prompt injection) into paying an attacker | — | Agent is propose-only, **auto limit $0**, payee must already be allowlisted, human approval required | D5, R3 DP4 |
| T3 | One or more **approvers** compromised | — | K-of-N distinct approvers, maker-checker, signatures bound to the exact intent hash, buckets + hard cap still apply | R3 DP4 |
| T4 | **Malicious UI / blind signing** makes owners sign something else | **Bybit, Feb 2025, ~$1.5B** | Guard blocks `delegatecall` to non-allowlisted targets and self-calls outside the time-locked path; the UI shows the decoded intent + hash; approvers check on hardware wallets | R2 §7.4 |
| T5 | **Admin key compromise** loosens the rules | — | Loosening = time-locked + events + alerts; `GUARDIAN` veto; tightening is instant | R3 DP6 |
| T6 | **Forged / malicious cross-chain message** | **KelpDAO, Apr 2026, ~$292M** | Router + source chain + sender checks; CCIP RMN + token-pool rate limits; our own per-lane caps; incoming goes to escrow (no auto-release); pause per lane | R5 §5, §9.6 |
| T7 | **Oracle** stale / manipulated / de-peg | — | Per-feed staleness, answer > 0, deviation check, stablecoin floor $1, round up; failure → approval tier (never auto) | R3 DP2, R7 |
| T8 | **Reentrancy** on `settle` / `claimRefund` / module execution | Classic | Checks-effects-interactions + `ReentrancyGuard` + `SafeERC20`; hooks via `staticcall` | R3 §8.4, R4 DP3 |
| T9 | **Signature replay** (other chain, other contract, same payload twice) | Classic | EIP-712 domain with `chainId` + `verifyingContract`; per-proposal nonce; deadline; mark used | R3 DP4 |
| T10 | **Guard DoS / bricking** | Safe docs warning | Anti-bricking path (time-locked guard removal always allowed); oracle/hook failure never reverts the guard | R2 §2A, R3 §8.3 |
| T11 | **Front-running signed payments** | ERC-3009 spec | `receiveWithAuthorization` (payee-only) | R4 §9.3 |
| T12 | **Spam / dust / address-poisoning** deposits | Common | UNMATCHED queue; never auto-sweep unsupported tokens; exact-match allowlist; UI shows full address + label | R4 DP6 |
| T13 | **Weird tokens** (fee-on-transfer, rebasing, blocklists) | Common DeFi bug class | Only allowlisted tokens (USDC, WETH, CCIP-BnM); record received amount as a **balance delta**; handle USDC blocklist reverts gracefully | R7 |
| T14 | **Stuck CCIP messages** (receiver revert, gas) | CCIP manual execution | Receivers never revert on business rules; `gasLimit` tested; mutable `extraArgs`; manual-execution runbook + monitoring | R5 §2A |
| T15 | **Malicious upgrade** | Many proxy incidents | Upgrades = loosening (time-lock + veto); storage-layout checks in CI; escrow not upgradeable | DP1 below |
| T16 | Precision / rounding in buckets and USD math | Common | Round against the spender; fuzz the math; 18-decimal internal USD | R3 DP1 |

**CCIP's own best-practice checklist is covered:** verify destination chain, source chain, sender and router; set a tested `gasLimit`; mutable `extraArgs`; decouple message receipt from business logic; soak-test rate limits; monitoring; multisig-owned admin roles. **V**

---

## 3. DP1 — Upgradeability (one-way door)

| | A. Immutable + "upgrade by replacement" | B. UUPS proxies everywhere | **C. Hybrid** |
|---|---|---|---|
| How | Contracts never change; a new version is deployed and swapped in via the time-locked admin path (e.g., `setGuard`, `setEngine`) | Every contract is a UUPS proxy; the Safe (via time-lock) upgrades | **Escrow + Guard immutable** (hold/protect funds); **PolicyEngine = UUPS proxy** (rules code evolves, keeps state: buckets, allowlist) behind the time-lock; EIP-7201 namespaced storage (M12) |
| Pros | Simplest trust story ("code can't change under you"); auditors like it | Everything fixable in place | Keeps engine state across rule upgrades; funds-holding code stays immutable; shows M08 upgrade skills |
| Cons | Migrating engine state (buckets, allowlist) on replacement is manual | Largest attack surface (upgrade key, storage collisions, init bugs) | Upgrade logic + storage-layout discipline for one contract |
| D7 score | 4.40 | 3.77 | **4.44** |

**Pick: C (hybrid)**, a near-tie with A.
- The engine upgrade is a **loosening change** → time-lock + guardian veto (R3 DP6).
- CI runs OpenZeppelin's upgrade-safety validation (reuse the UdraForge setup).
- If time gets short, fall back to A. The difference is contained to one contract.

---

## 4. DP2 — Toolchain & CI

| Tool | Purpose | When | Notes |
|---|---|---|---|
| **Foundry** (`forge test`, `forge coverage`, gas snapshots) | Unit, fuzz, invariant, fork tests | Every push | Advanced Foundry is taught in M14 |
| **Slither** (Trail of Bits) | Static analysis (guide requirement) | Every push (GitHub Action) | Triage findings into a `SECURITY.md` table (fixed / accepted + reason) |
| **Aderyn** (Cyfrin) | Second static analyser (Rust; different detectors) | Every push (`aderyn-ci` action exists **V**) | Cheap extra coverage |
| **Echidna** | Property-based fuzzing (guide requirement) | Nightly / on-demand (`echidna-action` **V**) | Modes: **property** (functions returning `true` must stay true), **assertion** (`assert` must hold), and a Foundry-style mode **V**. Use the `crytic/properties` library for common ERC-20 properties **V** |
| **Chainlink Local** | CCIP simulated in Foundry (incl. forked) | Every push | R5 |
| Fork tests | Real Sepolia Safe, feeds, CCIP router (read/simulate) | Nightly | Catches config mistakes (addresses, decimals) |
| **Playwright** (+ a mocked wallet connector) | End-to-end dApp tests | Before demo | R10 picks the exact wallet-mocking approach |
| OZ upgrades validation | Storage layout / initializer checks | Every push | For the PolicyEngine proxy |

**CI pipeline (GitHub Actions):** `fmt → build → unit/fuzz/invariant tests + coverage → Slither → Aderyn → gas snapshot diff` on each push; `Echidna + fork tests` nightly. The badge set goes in the README (the guide requires a README).

---

## 5. DP3 — Test plan skeleton (maps to the guide's list)

| Guide item | What we test | Examples |
|---|---|---|
| **Unit** | Each rule, each outcome, each role; each escrow transition | Allowlist activation delay; tier boundaries ($999 / $1,000 / $1,001); stale price → approval tier; each escrow state transition and forbidden transition |
| **Fuzz** (Foundry) | Math and input handling | Bucket refill/consume never exceeds bounds; USD conversion across decimals; random `invoiceId` / amounts to the receiver never revert |
| **Invariant** (Foundry) + **property-based** (Echidna) | System-wide rules across random call sequences | §6 list |
| **Fork** | Real Sepolia contracts | Guard installed on a real Safe v1.5 deployment; real ETH/USD feed; CCIP router fee quote |
| **Cross-chain** | Chainlink Local simulator | PAYOUT and INVOICE_PAYMENT round trips; forged sender/source rejected; receiver never reverts |
| **E2E** | UI → contracts on Anvil/testnet | Propose → approve → execute; pay invoice → clear → settle; reject → refund |
| **Security regressions** | Attack scenarios as tests | Bybit-style `delegatecall` blocked; KelpDAO-style forged message quarantined; replayed approval signature rejected; reentrant token callback blocked |

**Targets:** ≥ 90% line coverage on core contracts; all §6 invariants green in Foundry and Echidna; zero unresolved Slither/Aderyn High findings.

---

## 6. Consolidated invariant list (for Foundry invariant tests + Echidna)

**Policy engine / guard (R3)**
1. No payment executes to a payee not active in the allowlist for that chain.
2. Bucket balance ≤ capacity; USD spent over any window of length t ≤ cap + rate × t.
3. Module-path payments above the auto limit never execute without K valid, distinct, unused, unexpired approver signatures (approver ≠ proposer).
4. No `delegatecall` executes except to allowlisted targets.
5. A loosening change never takes effect before its time-lock expires; tightening is immediate.
6. The owner quorum can always remove or replace the guard after the delay (anti-bricking).
7. A stale or invalid price never produces ALLOW.
8. Failed inner calls don't consume the bucket (refund path).
9. The MCP agent role can never produce an ALLOW without human approval.

**Escrow / invoices (R4)**
10. Conservation: per token, escrow balance ≥ Σ open entries; received = settled + refunded + open.
11. Funds leave the escrow only via `settle` (CLEARED → Safe) or refund (REJECTED / overpay → original payer and chain).
12. No entry is settled or refunded twice.
13. An invoice's `paid` never exceeds its `amount`.
14. Payments for unknown / expired / cancelled invoices never reach CLEARED without an approver action.
15. The receiver never reverts on business-rule failures.
16. Verdicts are accepted only from the configured forwarder/workflow (or clearer key), once per entry.

**Cross-chain (R5)**
17. A spoke releases funds only for messages from the hub router + hub chain + hub sender.
18. Per-lane outbound/inbound totals never exceed the configured lane caps.

**Upgrades (DP1)**
19. The PolicyEngine implementation can only change via the time-locked path; storage layout is preserved (CI check, not fuzz).

---

## 7. Risks, unknowns, revisit triggers
- **Echidna + Safe + CCIP simulator setup complexity:** start with Echidna on **PolicyEngine and Escrow in isolation** (pure harnesses), then integrate. Use Foundry invariant tests for the full system.
- **Slither noise on Safe/Chainlink dependencies:** exclude `lib/` and document exclusions.
- **Time:** security tests are graded. Protect about 25–30% of dev time for them; set them up in week 1.
- **Unknown:** exact Echidna configuration for a Foundry project → M14 fills in details; the properties are already defined here.

## 8. Status
DP1 hybrid upgradeability; DP2 toolchain; DP3 test plan

## 9. Sources
- CCIP best practices (EVM): https://docs.chain.link/ccip/concepts/best-practices/evm
- CCIP defensive example: https://docs.chain.link/ccip/tutorials/evm/programmable-token-transfers-defensive
- Echidna repo: https://github.com/crytic/echidna
- Echidna testing modes: https://github.com/crytic/echidna/wiki/Testing-modes
- Echidna GitHub Action: https://github.com/crytic/echidna-action
- crytic/properties (pre-built properties): https://github.com/crytic/properties
- Aderyn: https://github.com/Cyfrin/aderyn
- Aderyn CI: https://github.com/Cyfrin/aderyn-ci
- Safe guards (DoS warning): https://docs.safe.global/advanced/smart-account-guards
- ERC-3009: https://eips.ethereum.org/EIPS/eip-3009
- Bybit analysis (Ackee): https://ackee.xyz/blog/a-safe-native-solution-to-the-bybit-hack/
- KelpDAO lessons (OpenZeppelin): https://www.openzeppelin.com/news/lessons-from-kelpdao-hack