
Pitch fact-check: cross-chain caps, bypass, post-hack changes

_Addendum to R1. Confidence tags: V verified in primary source, L likely._


**1 Do custodians have one combined cap across chains?** Partly yes — our "every chain is an island" claim is too strong.
- **Fireblocks (V field docs / L behaviour):** TAP rules take `asset: "*"`, `amountCurrency: USD`, `amountScope: TIMEFRAME`, `periodSec`; the docs' example blocks "when a $15,000 accumulation is reached over a 12-hour period" with `asset: "*"`. Assets are per network (e.g., USDC on Ethereum vs Base are different asset IDs), so this is effectively a workspace-wide USD cap across chains. The docs don't state the cross-asset summing explicitly → L.
- **BitGo (V):** enterprise-level BitGo-enforced policies across all wallets and coins, e.g., live video ID verification when withdrawals exceed **$250k per day (cumulative)**. It's a verification trigger, not a hard block, and customisable.
- **Coinbase Prime Onchain Wallet (V, absence):** policy engine docs describe source/destination/initiator/approver rules; no amount or velocity limits mentioned.
- **Where the gap really is:** these caps only see transactions the custodian signs. On-chain wallets (Safe guards, Zodiac Roles) keep state per chain, so there's no shared on-chain cap across chains; bridges (CCIP token-pool rate limits) cap per lane per token, not per treasury.

**2 Off-chain and bypassable?**
- All three are off-chain (provider-side). **Fireblocks (V):** rules "signed by admin quorum, encrypted in the enclave, and verified across multiple MPC servers before signing"; enforced on every operational route including API. Hard to bypass in normal operation.
- **Bypass paths exist by design for disaster recovery (V):** BitGo — "Recovery transactions that use the user key and the backup key bypass any policies you may have in place with BitGo"; Fireblocks recovery tool — "Recover your workspace private keys … send transactions from recovered wallets"; Coinbase Prime key export — extract the root private key and "transact without Coinbase". Also any signing outside the provider isn't seen by its policy.
- Fireblocks' own comparison notes Safe modules "can bypass signature verification" — our design covers this with the Safe v1.5 module guard.

**3 Improvements after recent hacks?**
- **Safe (V, Feb 2025):** stricter validation, monitoring warnings, extra checks; paused native Ledger integration.
- **Fireblocks (V):** Bybit response blog lists existing features (enclave policy engine, native transaction decoding), no new ones; May 2025 "Next-Generation Policy Engine" with multi-asset rules, dApp access policy.
- **BitGo (V, 30 Apr 2026):** five-layer model — API attestations binding tx to user intent, BitGo Verify app with device attestation, behavioural threat detection, policy engine additions (recommendations, webhooks). No cross-chain or on-chain enforcement.
- **LayerZero (V):** Labs DVN now refuses to sign for 1/1 configs (after KelpDAO).
- Pattern: vendors hardened intent verification, devices and detection, all still off-chain. Nobody we found added an on-chain backstop or a treasury-wide on-chain cross-chain cap.

**Pitch wording change suggested:** replace "Every chain is an island" with "Limits only see what the custodian signs" (recovery paths, other signers and on-chain wallets sit outside it; on-chain wallets have no shared cap across chains).

Sources: Fireblocks TAP docs (developers.fireblocks.com/reference/configure-transaction-authorization-policy); Fireblocks policy comparison (fireblocks.com/report/compare-transaction-policy-engine); Fireblocks defense-in-depth report; Fireblocks Bybit blog; Fireblocks Pulse release (prnewswire, 21 May 2025); github.com/fireblocks/recovery; BitGo Policies overview; BitGo-enforced policy rules (support.bitgo.com); BitGo 5-layer release (businesswire, 30 Apr 2026); Coinbase Prime policy engine + key export help pages; crypto.news on Safe post-Bybit changes.

**4 Fireblocks treasury, payments and Revolut (added 2026-10-01)**
- **Revolut case study (V):** internal crypto treasury on Fireblocks; staff request transfers and "if they go over certain thresholds or hit certain triggers, other people in the organization will be pinged to approve it"; automated rebalancing across liquidity providers; MPC-CMP. Close to our **outgoing** side; nothing on invoices or on-chain enforcement.
- **Treasury Management (V):** "150+ blockchains and thousands of assets"; transaction limits, approval workflows, permissions; Fireblocks Automation ("Build and manage automation rules"); DeFi access.
- **Payments (V):** routing across chains; KYT, Travel Rule and address screening "before funds move"; automate deposits/withdrawals, batch disbursements; MT940 reconciliation exports.
- **Incoming AML (V):** screens incoming transactions; "Auto Freeze allows you to set rules to automatically freeze an incoming transaction's assets… for further review"; frozen balance stays in the workspace, unspendable, manual unfreeze. No invoice matching.
- **Agentic Payments Suite (V, 20 May 2026):** Agent Wallets give agents "scoped, revocable spending authority, bound by the Fireblocks Policy Engine before anything signs" — agents can spend autonomously within off-chain limits.
- **Consequence for our pitch:** don't claim custodians lack multichain, automation, incoming screening or agent support. Our differences: (1) rules enforced on-chain, so they hold whoever signs (recovery keys, other signers, modules) and work without Fireblocks; (2) incoming money linked to an invoice, held in escrow, refunded/queued by contract rules; (3) agents propose-only, enforced on-chain; (4) publicly verifiable rules. Framing: "Fireblocks is the mature off-chain version of much of this; we build the on-chain, open version that also backs it up."
- Sources: fireblocks.com/customers/revolut; fireblocks.com/products/treasury-management; fireblocks.com/products/payments; developers.fireblocks.com/docs/define-aml-policies; fireblocks.com/blog/agentic-payments-suite-psp-fintech.

**5 Slides 2–5 fact-check (2026-10-02)**
- **Bybit (V):** 21 Feb 2025; FBI: ~$1.5B, attributed to TraderTraitor (Lazarus); NCC Group: "more than $1.4 billion… including 401,347 ETH"; malicious JS injected via a compromised Safe{Wallet} developer machine (Sygnia/Mandiant: Safe's AWS); `delegatecall` changed the wallet's implementation; signers blind-signed on hardware wallets.
- **KelpDAO (V):** 18 Apr 2026; 116,500 rsETH (~$292M); 1-of-1 DVN (LayerZero Labs'); two RPC nodes compromised, others DDoS'd; ~80-minute window; "No smart contract was broken" (OpenZeppelin). Lazarus/TraderTraitor attribution comes from **LayerZero**, not the FBI.
- **Wording tightened on slide 2:** "Recovery paths and hacked screens can go around them" → "Recovery paths can skip them; hacked screens can trick approvers" (Fireblocks enforces rules off-device, so a hacked screen tricks approvers rather than skipping rules). "On-chain wallet rules keep no shared cap across chains" → "On-chain wallet rules, like Safe guards, work one chain at a time" (verified for Safe guards / Zodiac Roles; not every wallet checked).
- Sources: nccgroup.com Bybit analysis; securityaffairs.com (FBI/TraderTraitor); sygnia.co Bybit investigation; openzeppelin.com/news/lessons-from-kelpdao-hack; coindesk.com 2026-04-20 (LayerZero attribution).