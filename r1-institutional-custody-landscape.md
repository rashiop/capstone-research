# R1 — Institutional Custody Landscape

_Step 3 research, Wave 1. Researched 2026-09-23. Method: `02a-research-method.md`. Confidence tags: **V** = Verified in primary source, **L** = Likely (secondary source), **U** = Unverified._

## 1. Decision & requirements served
- **Decisions it unblocks:** (a) which policy primitives v1 must support, (b) what our on-chain layer does vs. what stays with the custodian off-chain, (c) the pitch's "why on-chain?" answer.
- **Serves:** F1, F2, F4, non-functional "institutional fit".
- R1 is a landscape track, not an option pick, so there's no scoring matrix. Its outputs feed R2 and R3.

## 2. Key findings

### 2.1 How the big custodians model policies

| | Fireblocks | BitGo | Coinbase Prime (Onchain Wallet) |
|---|---|---|---|
| Key tech | MPC | Multi-key (user/backup/BitGo); MPC options | MPC, key "born split 2-of-2" between Coinbase and user devices |
| Where the policy runs | Off-chain policy engine (TAP) | Off-chain, BitGo-side | Off-chain policy engine |
| Rule shape | Ordered rules, **first match wins** | Typed rules: Destination, Initiator, % of wallet balance, Threshold, Velocity limit, Webhook | Rules on source, destination, initiator → block or require approval |
| Outcomes | `ALLOW`, `BLOCK`, `2-TIER` (needs approval) | Approve / deny / require extra approval | Block / require approval |
| Amount limits | `amount` + `amountCurrency` (USD/EUR/native) + `amountScope` (`SINGLE_TX` or `TIMEFRAME`) + `periodSec` | Threshold + Velocity limit | Not mentioned in docs |
| Allowlist | `src`/`dst` account filters; whitelisted addresses | Wallet whitelists | "Onchain Trusted Address Book", **on by default** |
| Who can initiate | `operators` (users / groups) | Initiator rules | Initiator rules |
| Approvers | `authorizationGroups` with threshold `th`; `designatedSigners` | Second approval | Custom approval controls |
| Tx types | TRANSFER, CONTRACT_CALL, APPROVE, MINT, BURN, STAKE, RAW, TYPED_MESSAGE | Withdrawals | Onchain tx + message signing |
| Confidence | V | V | V |

### 2.2 Common policy primitives (what "institutional-grade" means)
Common to all three, plus Fireblocks' own design guide (V):
1. **Allowlist / address book** of approved destinations (default-deny for unknown addresses)
2. **Per-transaction threshold** (amount per transaction)
3. **Velocity limit** (total over a time window), per asset, **priced in USD**
4. **Initiator control:** who may start which transaction type
5. **Tiered approvals by amount:** e.g., above X needs a group approval with quorum N
6. **Transaction-type rules:** transfer vs. contract call vs. token approve vs. signing a message
7. **Separation of duties:** a separate admin group, with its own quorum, approves changes to the policy itself
8. **Webhook / external check** (BitGo): an outside system can approve or deny (→ maps to our invoice/compliance hook)
9. **% of balance** limit (BitGo)

### 2.3 On-chain vs. off-chain: the honest trade-off
- **Fireblocks' position (V):** keep security policy **off-chain**. Their reasons: on-chain rules broadcast your internal security logic publicly, and institutions run "dozens and even hundreds of rules" versus "simplistic on-chain spending limits". They pair MPC with EIP-7702 and put UX features on-chain (batching, gas sponsorship, session keys).
- **Weakness of off-chain-only policy (V):**
  - BitGo docs: *"Recovery transactions that use the user key and the backup key bypass any policies you may have in place with BitGo."* An off-chain policy only binds transactions that go through the provider.
  - **Bybit, Feb 2025, ~$1.5B stolen:** attackers compromised the Safe web frontend. Signers blind-signed a transaction that `delegatecall`ed a malicious implementation and took over the vault. The multisig (n-of-m) worked as designed. The missing layer was an **on-chain guard** that would have rejected the `delegatecall` no matter what the signers signed.
- **Our position (for the pitch):** use **both layers, each doing what it's good at**.
  - The custodian's off-chain engine stays the first line: rich rules, private logic, MPC key security.
  - Our on-chain layer is the **backstop**: a small set of hard rules that no key, UI or provider compromise can bypass. It also covers what off-chain engines can't see: **receiving-side** checks, **cross-chain** consistency, and **public verifiability** for auditors and counterparties.
  - Keep on-chain rules few and coarse (hard caps, allowlists, forbidden call types) so we don't leak detailed internal logic.

### 2.4 Existing on-chain precedent
- **Cobo Argus / Cobo Safe (V/L):** an institutional product built **on Safe**, with on-chain role-based access control and delegation for operators, and a tiered authorization procedure. It has integrated Chainlink price feeds (L). → Evidence that "Safe + on-chain policy module + oracle-priced rules" is a real institutional pattern. It's also a competitor to reference in R13.
- **Safe Guards / Zodiac ScopeGuard (V):** can restrict calls "on top of the n-out-of-m scheme", e.g., ban `delegatecall` to arbitrary addresses. → R2.

### 2.5 Custodian adapter (F4)
- Fireblocks, BitGo and Coinbase all present as a **signer**. To our contracts, a custodian-controlled key is just an address that signs.
- So the adapter is simple: **the custodian's MPC key is an owner/signer of the smart account**, and our policy layer sits between "signed" and "executed".
- Nobody needs an API integration for the MVP. On testnet, an EOA or a hardware-wallet key plays the custodian's MPC signer. The pitch shows "Fireblocks/BitGo/Coinbase MPC key → owner of our smart account" as the production path.
- Confidence **L**: Fireblocks can sign for Safes (a forum thread about MPC as a Safe signer, and Fireblocks' 7702 blog). Confirm in R2 whether there's a documented integration.

## 3. What this means for our design (proposed, pending R2/R3)

### 3.1 v1 on-chain policy primitives

| # | Primitive | Source of pattern | MVP? |
|---|---|---|---|
| P1 | Destination allowlist (address book), default-deny | All three | Must |
| P2 | Per-tx threshold in USD (Chainlink-priced) | Fireblocks `SINGLE_TX`, BitGo Threshold | Must |
| P3 | Velocity limit in USD over a window | Fireblocks `TIMEFRAME`, BitGo Velocity | Must |
| P4 | Tiered approvals: under X auto (policy-checked), above X needs N-of-M approvers (EIP-712 signatures) | Fireblocks `2-TIER`/`authorizationGroups` | Must |
| P5 | Initiator roles (operator, agent/MCP proposer, approver) | Fireblocks `operators`, Initiator rules | Must |
| P6 | Tx-type restrictions: block `delegatecall`, unknown contract calls, unlimited `approve` | Fireblocks `type`; Bybit lesson | Must |
| P7 | Policy-change governance: separate admin quorum + time-lock on rule changes | Fireblocks separation of duties | Must (reuses M07 Timelock) |
| P8 | External check hook (invoice / compliance) | BitGo Webhook rule | Must on the receiving side (F2/F3); Should on the sending side |
| P9 | % of balance limit | BitGo | Could |

### 3.2 Responsibility split

| Concern | Custodian (off-chain) | Our layer (on-chain) | Our backend (off-chain) |
|---|---|---|---|
| Key generation and storage (MPC) | ✅ | — | — |
| Rich / private rules | ✅ | — | — |
| Hard caps, allowlist, forbidden call types | optional | ✅ backstop | — |
| Approval quorum above threshold | ✅ | ✅ (EIP-712 approvals) | collects signatures |
| Receiving-side clearance (invoice, sender allowlist) | ❌ not covered by custodians | ✅ | invoice data, Functions source |
| Cross-chain policy consistency | ❌ | ✅ | message tracking |
| Audit trail | internal logs | ✅ events (public) | indexer |
| Agent (MCP) proposals | — | ✅ enforced like any initiator | ✅ queue |

## 4. Risks, unknowns, revisit triggers
- **Privacy of rules:** allowlists and limits are public on-chain. Mitigation: coarse on-chain caps. Revisit if the instructor pushes on privacy (possible roadmap item: hashed allowlists / ZK).
- **Gas cost of many rules:** keep the on-chain rule set small. R3 decides the data structures.
- **Custodian-as-signer (U → verify in R2):** confirm there's a documented Fireblocks/BitGo → Safe owner flow for the pitch slide.
- **"Receiving side isn't covered by custodians":** inferred from the absence of inbound rules in all three docs. Treat as **L**. The pitch should say "not their focus", not "impossible".

## 5. Pitch one-liner
> Custodians protect the **keys**. We protect the **transactions**: an on-chain backstop that enforces the rules even when a key, a UI or a provider is compromised. It extends to incoming payments and to other chains, which custodians' policy engines don't cover.

## 6. Sources
- Fireblocks — Configure Policies (TAP rule fields): https://developers.fireblocks.com/reference/configure-transaction-authorization-policy
- Fireblocks — How to design a transaction policy: https://www.fireblocks.com/blog/designing-a-digital-asset-or-crypto-transaction-policy
- Fireblocks — Mutualism, MPC and EIP-7702: https://www.fireblocks.com/blog/mutualism-mpc-and-eip-7702
- BitGo — Policies Overview: https://developers.bitgo.com/guides/policy-builder/overview
- BitGo — Whitelists: https://developers.bitgo.com/docs/wallets-whitelists-update
- Coinbase — How to secure your Onchain Wallet (Prime): https://help.coinbase.com/en/prime/onchain-wallet/how-to-secure-your-onchain-wallet
- Cobo Argus — Developer intro: https://www.cobo.com/developers/v1/overview/smart-contract-wallet/coboargus
- Cobo Argus × Chainlink: https://www.cobo.com/post/cobo-chainlink-integration
- Ackee — A Safe-native solution to the Bybit hack: https://ackee.xyz/blog/a-safe-native-solution-to-the-bybit-hack/
- NCC Group — Bybit hack technical analysis: https://www.nccgroup.com/research/in-depth-technical-analysis-of-the-bybit-hack/
- Safe forum — MPC as a Safe signer: https://forum.safe.global/t/self-custody-with-multi-party-computation-as-a-safe-signer/2267/5