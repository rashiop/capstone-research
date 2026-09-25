# Research Method

## 1. How research goals are chosen (decision-driven)
We only research questions whose answer **changes a decision**. Chain of traceability:

`Requirement (F1–F10, guide item) → Decision needed → Research question → Evidence → Decision (logged)`

Each track starts by stating:
- **Decision it unblocks** (e.g., "which smart account base")
- **Requirements it serves** (e.g., F1, F4, F7)
- **Done when** (e.g., "recommendation + where policy code plugs in")
- **Timebox** (P0: ~1 day, P1: ~half day, P2: ~2 hours)

A question that doesn't feed a decision goes to a "later / curiosity" list.

## 2. Decision types (how much rigor)
| Type | Examples | Rigor |
|---|---|---|
| **One-way door** (expensive to reverse) | Smart account base, cross-chain protocol, where policy is enforced, upgradeability | Full matrix + spike (small prototype) if close |
| **Two-way door** (cheap to reverse) | UI library, ORM, job runner | Quick pick: default to the mature, popular option unless a hard constraint says otherwise |

## 3. Comparison method
**Step 1 — Hard gates (pass/fail).** An option is dropped if it fails any:
- Works on Ethereum Sepolia + Base Sepolia today (verified in live docs)
- Usable without a business account / sales call
- Works with Foundry (tests, scripts) and viem
- Actively maintained (commits/releases in last ~6 months); audited if it touches funds
- Permissive license for a public portfolio repo
- Buildable solo within the development phase (Modules 13–16) — added because effort weight is low (D7)

**Step 2 — Weighted scoring (1–5)** of the survivors. Weights set by Pops (D7), fixed *before* scoring to avoid bias; a track may adjust them but must say why:

| Criterion | Weight | Traces to |
|---|---|---|
| Requirement fit (covers Must features) | 25% | F1–F10 |
| Institutional credibility / standard alignment | 20% | Instructor direction |
| Grading coverage (shows security, tests, oracles, multichain) | 15% | Capstone guide |
| Portfolio & learning signal (job market, Solidity depth, FE/BE refresh) | 12% | Career, D6 |
| Maturity & security (audits, usage, known incidents) | 10% | Security |
| Testnet & tooling support (local sim, fork tests, docs quality) | 10% | Dev velocity |
| Solo build effort & delivery risk | 8% | D1 |

Each score has a one-line reason + source link.

**Step 3 — Spike (only when needed).** 2–4 hour throwaway prototype to test the riskiest assumption (e.g., "can a Safe guard read Chainlink and block a tx within gas limits").

## 4. Evidence rules
- Primary sources first: official docs, EIPs/ERCs, GitHub repos, audit reports. Blogs only as pointers.
- Anything time-sensitive (testnet support, versions, free tiers) is verified live and dated.
- Confidence tag per finding: **Verified** (read in primary source or tested) / **Likely** (secondary source) / **Unverified** (from memory — must be checked before it drives a decision).

## 5. Dependency order
Decisions cascade, so upstream first:
- R2 (smart account) → shapes R3 (policy hook shape), R6 (4337/7702), R8 (proposals)
- R5 (cross-chain) → shapes R7 (token list), R4 (cross-chain receiving)
- R8 (MCP) → shapes R11 (backend)
Wave 1 order: R1 → R2 + R5 → R3 + R4.

## 6. Track output template (≤ 2 pages each, feeds Step 4)
1. Decision & requirements served
2. Key findings (with confidence tags + links)
3. Options → hard gates → scoring matrix
4. Recommendation + why
5. Risks, unknowns, what would make us revisit
6. Sources

## 7. How a decision is made and recorded
- Logged in `00-decisions-log.md` as: decision, options considered, why, revisit trigger.
- Close calls or anything the instructor cares about (cross-chain, custody alignment) go on a "confirm with Dhruvin" list.

## Caveat
Scores look precise but are judgment. Their value is making trade-offs visible and arguable in the pitch, not proving a winner.