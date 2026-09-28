# R7 — Oracles & Multi-Token Support (final token list)

_Step 3 research, Wave 2. Researched 2026-09-27. Inputs: D3 (multi-token), R3 DP2 (pricing rules), R5 §6 (token matrix). Confidence: **V** Verified · **L** Likely · **U** Unverified (must be checked at build time)._

## 1. Decisions & requirements served
- **Decisions:** (a) the final MVP token list, (b) which price feeds, (c) the price-oracle interface, (d) testnet fallbacks.
- **Serves:** guide requirement "use oracles for reliable price data", D3, R3 DP1/DP2 (USD limits), R5 (bridgeable tokens).
- Mostly a **two-way door** (token config is data, not code), so no weighted matrix. Picks follow the rule **(bridgeable or hub-native) ∩ (priceable)**.

## 2. Key findings

### 2.1 Chainlink Data Feeds on Ethereum Sepolia (hub)
| Feed | Address (Sepolia) | Confidence | Use |
|---|---|---|---|
| ETH / USD | `0x694AA1769357215DE4FAC081bf1f309aDC325306` | **V** (Etherscan + widely used in Chainlink/Cyfrin courses) | Price ETH/WETH payments and CCIP fees |
| BTC / USD | `0x1b44F3514812d835EB1BDB0acB33d3fA3351Ee43` | **V** (Chainlink getting-started example) | Not needed for the MVP |
| USDC / USD | listed in Chainlink's feed directory | **U**: address, heartbeat and liveness must be checked at build time | Price USDC (with the $1 floor) |
| LINK / USD | listed in Chainlink's feed directory | **U** | Optional second volatile asset |

- **Testnet feeds are less reliable than mainnet.** There are public reports of Sepolia feeds failing or going stale (e.g., GitHub issue #8905 on ETH/USD, 2023). **V** (issue exists) → our fail-closed behaviour (R3 DP2) and the mock fallback below matter even more on testnet.
- Heartbeat/deviation values differ per feed. Read them from Chainlink's feed page for each feed and store them in config. **U** until checked.
- **Hub is L1 Sepolia → no L2 sequencer-uptime feed needed** (R3 DP2).

### 2.2 Token availability
| Token | Hub (Sepolia) | Spoke (Base Sepolia) | Cross-chain via CCIP (Sepolia ↔ Base Sepolia) | Free source |
|---|---|---|---|---|
| **USDC** (Circle test USDC) | ✅ | ✅ (Base Sepolia USDC `0x036C…CF7e`) **V** | **U**: CCIP supports USDC via CCTP; lane-specific support not confirmed | Circle faucet, 20 USDC / 2h / chain **V** |
| **WETH / ETH** | ✅ | ✅ | WETH is listed as a CCIP token/fee token on Sepolia **V** (lane transfer **U**) | ETH faucets |
| **LINK** | ✅ (Sepolia LINK `0x7798…4789`) **L** | ✅ | LINK listed on Sepolia CCIP **V** (lane **U**) | Chainlink faucet |
| **CCIP-BnM** | ✅ | ✅ | ✅ **V** (all testnets) | `drip()` |
| GHO | listed on Sepolia CCIP **V** | ? | ? | — (not needed) |

## 3. Decision: MVP token list

| Role | Token | Pricing | Why |
|---|---|---|---|
| **Primary stablecoin** | **USDC** | USDC/USD feed with **floor $1** (R3 DP2). If no live Sepolia feed → fixed $1 mock feed | The realistic treasury asset; free faucet; CCTP-backed on CCIP |
| **Volatile asset** | **WETH** (and native ETH for fees) | **ETH/USD** Chainlink feed (verified address) | Proves USD pricing actually matters: the same token amount uses a different share of the limit as the price moves. Also counts CCIP fees in USD |
| **Bridge demo fallback** | **CCIP-BnM** | **Fixed-price mock** ($1, clearly labelled in the UI) | Guaranteed on every CCIP lane; used if the USDC lane isn't available (R5 condition) |
| Optional | LINK | LINK/USD feed | Only if time allows; adds nothing new beyond WETH |

**D3 resolved:** multi-token = **USDC + WETH/ETH** (+ CCIP-BnM as fallback), not "many tokens". Two genuinely different price behaviours is enough to demonstrate the design; more tokens is only config.

## 4. Price-oracle interface (design, no code)
```
IPriceOracle
  usdValue(token, amount) → (usd18, status)       // status: OK | STALE | INVALID | NO_FEED
  config per token: { feed, heartbeat + buffer, isStable, fixedPriceUsd (optional), maxDeviationBps }
```
- **ChainlinkPriceOracle** (implementation): `latestRoundData` → checks (answer > 0, updatedAt ≠ 0, staleness per feed, optional deviation vs last good price) → normalise decimals → stable floor → round **up**.
- **The engine never reverts on an oracle problem.** Status ≠ OK → the payment drops to the **approval tier** (R3 DP2, anti-bricking R3 §8.3).
- **Mock feeds:** Chainlink's `MockV3Aggregator` pattern for unit tests; a deployed mock for feedless tokens (CCIP-BnM) and as an emergency testnet fallback.
- **Switching a token to a mock/fixed price is a loosening change → time-locked** (R3 DP6); short delay on testnet for the demo.
- The interface is the `IPriceOracle` promised in R5 §9.3 (vendor-swap point for a second oracle later).

## 5. Tests (feeds into R9)
- Unit: stale price → approval tier; answer ≤ 0 → approval tier; decimals 6/8/18 combinations; floor $1 for stables; rounding up.
- Fuzz: `usdValue` monotonic in amount; never below `amount × $1` for stables.
- Fork test against real Sepolia feeds (read-only) to catch config mistakes (wrong address/decimals).

## 6. Risks, unknowns, revisit triggers
- **USDC/USD and LINK/USD Sepolia feed addresses + heartbeats (U):** verify on day 1 of the build; if missing/stale → fixed $1 mock for USDC (documented).
- **Testnet feed staleness during the demo:** rehearse the day before; keep the mock fallback ready.
- **USDC on the CCIP lane (U):** R5 spike; fallback CCIP-BnM.
- **M13 (oracles) arrives in dev week 1:** this design lets M13 fill in details, not change the design.

## 6b. Mainnet demo fallback: rough cost (added 2026-09-27)
Inputs **V** (2026-09-26): Ethereum gas ≈ **0.2 gwei**, ETH ≈ **$2,690** (Etherscan). CCIP network fee: **Ethereum → non-Ethereum $0.50**, **non-Ethereum → Ethereum $1.50**, **non-Ethereum ↔ non-Ethereum $0.25** (paid in non-LINK; LINK ~10% cheaper), plus destination gas. Gas units below are **estimates (U)**.

**Option M1: hub on Ethereum mainnet, spoke on Base mainnet**

| Item | Assumption | @0.2 gwei (today) | @1 gwei | @5 gwei (spike) |
|---|---|---|---|---|
| Deploy + configure hub (engine, guard, module, escrow, oracle adapter, Safe, ~30 config txs) | ~15M gas | ~$8 | ~$40 | ~$200 |
| Demo + rehearsal transactions on hub | ~30 txs × 250k = 7.5M gas | ~$4 | ~$20 | ~$100 |
| ×3 buffer (redeploys, mistakes) | | **~$36** | ~$180 | ~$900 |
| Spoke on Base (gateway + config + demo) | Base gas is a fraction of L1 | ~$1–5 | | |
| CCIP messages | 10 × Eth→Base (~$0.6 each) + 10 × Base→Eth (~$1.7 each) | ~$25 | ~$27 | ~$35 |
| **Total spend** | | **≈ $60–70** | ≈ $210 | ≈ $950 |

**Option M2: hub and spoke both on L2s (e.g., Base hub + Arbitrum spoke)**
- Deployment + demo: a few dollars in gas; CCIP $0.25 + small gas per message → **≈ $10–25 total**.
- Trade-off: the hub runs on an L2, so add the **L2 sequencer-uptime check** (R3 DP2).

**Plus working capital (not spent, but at risk):** real USDC for demo invoices (e.g., $20–50) + a little WETH. Recoverable if nothing goes wrong.

**What mainnet gives you**
- Real, reliable Chainlink feeds (USDC/USD, ETH/USD).
- Confirmed CCIP lanes with native USDC (via CCTP).
- The real Chainalysis sanctions oracle (mainnet-only).
- Safe v1.5 and EAS all live.

**What it costs beyond money**
- **Real funds + unaudited contracts:** keep amounts tiny and set low hard caps in the contracts.
- **Key management:** use a hardware wallet or a separate "deployer" wallet with only the needed funds.
- **Mistakes are permanent and public;** gas can spike (5 gwei ≈ 25× the cost).

**Recommendation:** keep **testnet as primary**. Prepare a **mainnet fallback on L2s (M2, ~$10–25)** only if testnet feeds/lanes fail during rehearsal. Choose M1 (~$60–70 today, budget $200) only if showing an Ethereum-mainnet hub matters for the story. **Decide after the dev-week-1 spike** (USDC lane + feed liveness).

## 7. Status
**Approved 2026-09-27 → D14.**

## 8. Sources
- Chainlink — Consuming Data Feeds (Sepolia BTC/USD example): https://docs.chain.link/data-feeds/getting-started
- Chainlink — Price feed contract addresses: https://docs.chain.link/data-feeds/price-feeds/addresses
- Chainlink — Data Feeds API reference: https://docs.chain.link/data-feeds/api-reference
- Sepolia ETH/USD aggregator on Etherscan: https://sepolia.etherscan.io/address/0x694AA1769357215DE4FAC081bf1f309aDC325306
- GitHub issue #8905 (Sepolia feed not working): https://github.com/smartcontractkit/chainlink/issues/8905
- CCIP Directory — Ethereum Sepolia: https://docs.chain.link/ccip/directory/testnet/chain/ethereum-testnet-sepolia
- Base Sepolia USDC on BaseScan: https://sepolia.basescan.org/address/0x036cbd53842c5426634e7929541ec2318f3dcf7e
- Sepolia LINK token: https://sepolia.ethplorer.io/address/0x779877a7b0d9e8603169ddbd7836e478b4624789
- Circle faucet: https://faucet.circle.com/
- Etherscan gas tracker (2026-09-26): https://etherscan.io/gastracker
- CCIP billing (network fees): https://docs.chain.link/ccip/billing