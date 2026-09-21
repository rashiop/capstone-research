# Research 1 — Institutional Custody Problem

## Institutional Users

- [x] Treasury / Finance
- [x] Administrator
- [x] Initiator
- [x] Approver
- [x] Authorized Signer
- [x] Trader / Operator
- [x] Compliance / Risk
- [x] Auditor
- [x] Developer / Integration

### Key finding
> Institutional custody uses separation of duties. The person initiating a transaction does not necessarily have authority to approve or sign it.

---
## Institutional Workflow
Typical flow:
Transaction Request
→ Policy Evaluation
→ Allow / Approval Required / Deny
→ Approval / Consensus
→ Signing
→ Blockchain

### Key finding
> Transaction initiation, policy evaluation, approval and signing can be separate responsibilities.

---
## Policy Requirements
Common policy dimensions:
- Source
- Destination
- Asset
- Amount
- Spending limit
- Velocity limit
- Initiator
- Whitelist
- Approval requirements
- Role/permission
- Smart-contract/function interaction

### Generic policy model
Scope
→ Trigger
→ Conditions
→ Action

Possible actions:
- ALLOW
- REQUIRE_APPROVAL
- DENY

---

## On-Chain vs Off-Chain

### Good candidates for on-chain enforcement
- Transaction amount
- Destination address
- Asset
- Spending limits
- Velocity limits
- Approval state
- Roles
- Emergency pause
- Contract/function allowlists

### Better suited to off-chain systems
- KYC
- Sanctions screening
- Invoice matching
- ERP/accounting data
- Employee identity systems
- Risk scoring
- Business context
- Compliance data

### Key principle

The blockchain should enforce objective rules based on verifiable on-chain state.

External business/compliance context can be provided by off-chain infrastructure.

---

## Custody Boundary

### Custodian

Responsible for custody/security/execution infrastructure such as:

- Key management
- MPC
- Wallet infrastructure
- Signing
- Secure storage
- Transaction execution

### MPC

Answers:

> How do we securely authorize/sign?

### Policy Engine

Answers:

> Should this transaction be allowed to reach the signing stage?

### Our Project

Should primarily focus on:

> Programmable institutional transaction-control infrastructure.

We should not attempt to become:

- Full custody provider
- MPC provider
- Key-management system
- Compliance provider
- Blockchain bridge

---

## Current Project Positioning

> We are building a programmable smart-contract policy layer that controls institutional asset movement while custody/key management can remain with an external custody or wallet infrastructure.

### Core

- Transaction limits
- Velocity/daily limits
- Address allowlists
- Asset restrictions
- Role-based permissions
- Approval requirements
- Emergency controls
- Auditability

### Future / Advanced

- Cross-chain policy enforcement
- Receiving/payment controls
- Multi-level approvals
- Function-level DeFi controls
- ERC-4337 / EIP-7702
- Gas sponsorship
- MCP / AI integration

# Research 2 — Existing Institutional Custody Architecture

## Goal

Understand how real institutional custody systems separate:

- Transaction intent
- Policy
- Approval
- Custody
- Signing
- Blockchain execution

Then determine which architectural principles should be adopted by our capstone.

---

## Industry Systems Investigated

- Fireblocks
- BitGo
- Coinbase Prime

---

## Common Architecture

Institution
→ Transaction Request
→ Policy / Governance
→ Approval / Consensus
→ Signing / Key Management
→ Blockchain

---

## Major Layers

### Layer 1 — Application / Operations

Responsible for business intent:

- Treasury
- Trading
- Payments
- Settlement
- Operations

Example:

> Pay vendor $50,000 USDC.

The application converts business intent into a transaction request.

---

### Layer 2 — Policy / Governance

Determines whether a transaction is allowed.

Common policy dimensions:

- Source
- Destination
- Asset
- Amount
- Spending limit
- Velocity limit
- Initiator
- User role
- Whitelist
- Approval requirement
- Contract/function interaction

Generic model:

Scope
→ Trigger
→ Conditions
→ Action

Possible actions:

- ALLOW
- REQUIRE_APPROVAL
- DENY

---

### Layer 3 — Custody / Signing

Responsible for secure authorization and transaction signing.

Possible technologies:

- MPC
- TSS
- Multisig
- Smart accounts
- External custodians

Important:

> MPC/security infrastructure does not replace policy governance.

---

### Layer 4 — Blockchain

Final execution and state transition.

---

## Important Industry Patterns

### Pattern 1 — Policy is separate from custody

Policy determines whether an action is allowed.

Custody/signing determines how the transaction is cryptographically authorized.

---

### Pattern 2 — Initiation is separate from approval

The person creating a transaction request does not necessarily have authority to approve it.

---

### Pattern 3 — Approval is separate from signing

Approval/consensus happens before the final signing step.

---

### Pattern 4 — Policy can produce different outcomes

A transaction may:

- Be allowed automatically
- Require additional approval
- Be rejected

---

### Pattern 5 — Policies are composable

Policies can combine:

- Scope
- Trigger
- Conditions
- Actions

Conditions can use AND / OR logic.

---

### Pattern 6 — Policy ordering can matter

Some systems evaluate rules sequentially.

More restrictive/specific rules may need to be evaluated before broader rules.

---

### Pattern 7 — Policy configuration itself needs governance

Changing the policy can be as sensitive as executing a transaction.

Policy modifications should therefore require appropriate authorization.

---

### Pattern 8 — Policy should be independent of signing technology

The same policy should conceptually work with:

- MPC
- Multisig
- Smart accounts
- External custody

---

## Candidate Capstone Architecture
```text
Institution
↓
Application / API / MCP
↓
Transaction Intent
↓
Smart Contract Policy Layer
↓
Allow / Approval Required / Deny
↓
Execution / Custody Layer
↓
Blockchain
```
---

## Our Smart Contract Policy Layer

Potential responsibilities:

- Roles
- Spending limits
- Velocity limits
- Address allowlists
- Asset restrictions
- Approval requirements
- Emergency controls
- Policy governance
- Audit events

---

## External Responsibilities

Should NOT initially implement:

- MPC
- Private-key infrastructure
- Full custody provider
- Enterprise compliance platform
- Blockchain bridge
- Exchange infrastructure

These can be represented as external dependencies/interfaces.

---

## Core Architectural Principle

> The policy layer is an independent authorization layer between transaction intent and execution.

```text
Transaction Intent
    ↓
Authorization
    ↓
Execution
```

Authorization

* Who?
* What?
* From where?
* To where?
* How much?
* Which asset?
* Which chain?
* Under what conditions?
* How many approvals?

Execution

* MPC
* Multisig
* Smart Account
* ERC-4337
* EIP-7702
* External Custodian

⸻

Research 2 — Current Conclusion

Existing institutional custody systems separate transaction policy and governance from the underlying wallet/custody/signing infrastructure.

The policy layer determines whether and under what conditions a transaction can proceed, while the custody/signing layer provides the cryptographic mechanism for execution.

Our capstone should therefore focus on a programmable smart-contract policy layer rather than attempting to implement full institutional custody infrastructure.

⸻

Sources

* Fireblocks — Governance and Policies
* Fireblocks Developer Docs — What Is Fireblocks?
* Fireblocks Developer Docs — Set Policies
* BitGo — Policy Structure
* BitGo — MPC/TSS Wallet Operations
* BitGo — Crypto Policy Engines
* Coinbase Prime — Onchain Policy Engine
* Coinbase Prime — Perform an Onchain Wallet Transaction
* Coinbase Prime — Consensus for Onchain Wallet

# Research 4 - Core Policy
Upgradeable, policy-controlled institutional treasury layer with governed upgrades, namespaced storage, configurable policies, approval requirements, spending controls, and a final on-chain execution gate.

	| Component                   | MVP                               |
	| --------------------------- | --------------------------------- |
	| Policy engine               | Yes                               |
	| Upgradeable                 | Yes                               |
	| UUPS                        | Yes, candidate architecture       |
	| ERC-7201                    | Yes                               |
	| Policy versioning           | Yes                               |
	| Spending limits             | Yes                               |
	| Daily/velocity limits       | Yes                               |
	| Asset allowlist             | Yes                               |
	| Recipient allowlist         | Yes                               |
	| Chain allowlist             | Yes                               |
	| Approval thresholds         | Yes                               |
	| Emergency pause             | Yes                               |
	| Audit events                | Yes                               |
	| Role-based governance       | Yes                               |
	| Upgrade governance          | Yes                               |
	| Arbitrary calldata policies | Limited initially                 |
	| ERC-7579                    | Optional integration              |
	| EIP-7702                    | Optional integration              |
	| ERC-7730                    | Future integration                |
	| MCP                         | Interface, not security authority |

```
Upgradeable Institutional Policy Infrastructure
│
├── Upgradeable Policy Engine
│   ├── Policy registry
│   ├── Policy evaluation
│   ├── Spending limits
│   ├── Velocity accounting
│   ├── Allowlists
│   ├── Approval requirements
│   └── Emergency controls
│
├── Upgradeable Governance / Access
│
├── Upgradeable Account / Execution Integration
│
└── ERC-7201 Namespaced Storage 
```

OZ's UUPS for policy, so we use namespaced layout
```text
Layout:
InstitutionalPolicy
├── Policy namespace
├── Approval namespace
├── Limit namespace
├── Allowlist namespace
└── Governance namespace

Upgrade:
                    Proxy
                      │
                 ERC-1967
                      │
                      ▼
             PolicyEngine V1
                      │
                 upgrade
                      │
                      ▼
             PolicyEngine V2
        
Results:     
        PolicyEngine implementation V2
        │
        ├── Treasury Policy V7
        ├── Payroll Policy V3
        └── Operations Policy V5
```
Guarantee:

Only authorized accounts/requesters can initiate controlled actions.
Only approved assets and chains can be used.
Only approved destinations can receive funds.
Transactions cannot exceed configured per-transaction limits.
Aggregate spending cannot exceed configured velocity/daily limits.
Transactions above configured thresholds can require additional approval.
Emergency pause can prevent policy-controlled execution.
Policy decisions are bound to the exact transaction intent.
Policy changes are authorized and versioned.
Every execution path must pass the policy enforcement boundary.

# Research 5 - Payments control
Clearance status: PENDING, HELD, CLEARED, SETTLED, REJECTED, EXPIRED
Reject 
- unknown paymentId
- expired paymentId
- settled paymentId
- wrong payer, asset, amount
Prevent replay, signature: deadline, payment status, payment nonce


Role separations:
- Smart contract roles: Verify authority & binding of a clearance
- Off chain: reconciliation, KYC, sanctions screening, risk assessment, fraud detection, internal compliance review 

| Requirement         |                   On-chain? | Reason                               |
| ------------------- | --------------------------: | ------------------------------------ |
| Allowed asset       |                     **Yes** | Token address is known               |
| Allowed sender      |                     **Yes** | Address is known                     |
| Amount min/max      |                     **Yes** | Deterministic                        |
| Destination account |                     **Yes** | Contract state                       |
| Payment ID          |                     **Yes** | State binding                        |
| Payment deadline    |                     **Yes** | Timestamp                            |
| Replay prevention   |                     **Yes** | Nonce/status                         |
| Payment status      |                     **Yes** | Settlement state                     |
| Hold/freeze         |                     **Yes** | Asset custody/control                |
| Clearance signature |                     **Yes** | Signature/attestation verification   |
| Invoice existence   |                  **Hybrid** | On-chain reference, off-chain source |
| Invoice matching    |               **Off-chain** | ERP/business data                    |
| KYC                 |               **Off-chain** | Identity/compliance system           |
| Sanctions screening | **Off-chain + attestation** | External data                        |
| Fraud/risk scoring  |               **Off-chain** | Complex external computation         |
| Fiat conversion     |                  **Hybrid** | Requires price oracle                |
| Notifications       |               **Off-chain** | Operational concern                  |


# 6. Research Cross-Chain Architecture

> Include cross-chain capability in the architecture, but do not make a bridge protocol a hard dependency of the core MVP.

Goal: determine whether the custody policy layer can safely operate across multiple chains without building our own bridge. 
The original research scope includes CCIP vs LayerZero, messaging, token transfers, security, trust assumptions, fees, complexity, testnets, initiation, destination execution, failure handling, replay protection, message verification, and finality.

---

## 6.1 Why Cross-Chain Matters

The current policy architecture is:

```text
Institution
     ↓
Transaction Intent
     ↓
Policy Engine
     ↓
┌────┼──────────────┐
ALLOW  APPROVAL_REQUIRED  DENY
     ↓
Execution Gate
     ↓
Custody / Smart Account
     ↓
Blockchain
```

Institutional custody may operate across multiple chains:

```text
                    Institution
                         ↓
                   Policy Engine
                         ↓
             ┌───────────┴───────────┐
             ↓                       ↓
          Ethereum                Polygon
        Treasury A              Treasury B
```

A cross-chain operation introduces another question:

> Where is the institutional policy enforced when an operation moves from one chain to another?

Example:

```text
Ethereum
Treasury A
    │
    │ 100,000 USDC
    ↓
Cross-chain protocol
    │
    ↓
Polygon
Treasury B
```

The project should not implement its own bridge.

Instead:

* the external cross-chain protocol handles message/token transport;
* our policy layer determines whether the operation is permitted;
* the destination policy layer can perform a second enforcement check.

---

# 6.2 Cross-Chain Architecture Options

## Option A — Source-Chain Policy

```text
Ethereum
    ↓
Policy Engine
    ↓
Approved
    ↓
CCIP / LayerZero
    ↓
Polygon
    ↓
Destination Account
```

The source chain makes the authorization decision.

### Advantages

* simpler architecture;
* fewer policy evaluations;
* easier initial implementation.

### Risk

The destination must rely heavily on the validity of the source-side authorization and cross-chain message.

---

## Option B — Policy Enforcement at Both Ends

```text
Ethereum
    ↓
Source Policy
    ↓
Cross-chain Message
    ↓
Polygon
    ↓
Destination Policy
    ↓
Execution Gate
```

The destination independently verifies:

* source chain;
* source account;
* destination account;
* asset;
* amount;
* recipient;
* message validity;
* nonce;
* deadline;
* destination policy.

This provides a stronger defense-in-depth model.

### Decision

Use **destination-side policy enforcement** for the architecture.

The cross-chain protocol is responsible for message transport and verification, while our policy engine remains responsible for institutional authorization.

---

# 6.3 Chainlink CCIP

Chainlink CCIP provides cross-chain messaging and token-transfer capabilities.

Conceptually:

```text
Source Chain
     ↓
    CCIP
     ↓
Destination Chain
```

CCIP supports:

* arbitrary cross-chain messages;
* token transfers;
* programmable token transfers;
* multiple blockchain ecosystems.

CCIP also has a dedicated Risk Management Network as part of its security architecture.

### Architectural implication

CCIP should be treated as:

> External cross-chain infrastructure with its own security and trust assumptions.

Our project should not claim that the policy engine itself guarantees the security of the cross-chain transport layer.

---

# 6.4 LayerZero V2

LayerZero V2 uses an application-oriented model based around **Omnichain Applications (OApps)**.

Conceptually:

```text
Chain A
  ↓
OApp
  ↓
LayerZero Endpoint
  ↓
DVNs
  ↓
Verification
  ↓
Executor
  ↓
LayerZero Endpoint
  ↓
OApp
  ↓
Chain B
```

LayerZero separates:

* message verification;
* message execution.

Its verification model uses **Decentralized Verifier Networks (DVNs)**.

The application can configure its security stack and required DVNs.

LayerZero documentation recommends multiple independent DVNs for production configurations rather than depending on a single verifier.

### Architectural implication

LayerZero provides more explicit application-level control over the cross-chain security configuration, but that also creates more configuration responsibility for the application.

---

# 6.5 Token Transfer Models

Cross-chain messaging and token movement should be considered separately.

## CCIP

CCIP supports cross-chain token-transfer infrastructure.

Conceptually:

```text
Treasury A
    ↓
CCIP
    ↓
Treasury B
```

## LayerZero

LayerZero provides **OFT — Omnichain Fungible Token**.

Conceptually:

```text
Source OFT
    ↓
LayerZero Message
    ↓
Destination OFT
```

LayerZero also provides an **OFT Adapter** for integrating existing ERC-20 tokens.

### Capstone implication

We should not build a custom token bridge.

The policy layer should operate above the token-transfer protocol:

```text
Policy Engine
     ↓
CrossChainRouter
     ↓
Protocol Adapter
     ↓
Cross-chain Infrastructure
```

---

# 6.6 CCIP vs LayerZero

| Area                       | CCIP                                   | LayerZero V2                                     |
| -------------------------- | -------------------------------------- | ------------------------------------------------ |
| Core abstraction           | Cross-chain messaging + token transfer | Omnichain messaging                              |
| Application model          | CCIP sender/receiver                   | OApp                                             |
| Cross-chain messages       | Yes                                    | Yes                                              |
| Token transfers            | Yes                                    | Yes                                              |
| Existing token integration | Supported token mechanisms             | OFT Adapter                                      |
| Omnichain token            | CCT                                    | OFT                                              |
| Verification               | CCIP infrastructure                    | Configurable DVNs                                |
| Execution                  | CCIP infrastructure                    | Executor                                         |
| Security configuration     | More protocol-managed                  | More application-configurable                    |
| Developer abstraction      | Higher-level                           | More configurable                                |
| Complexity                 | Moderate                               | Higher                                           |
| Policy integration         | Strong                                 | Strong                                           |
| Security responsibility    | Significant protocol-level assumptions | Greater application configuration responsibility |

### Important conclusion

This comparison should **not** be reduced to "which bridge is safer."

The architectural difference is primarily:

```text
CCIP
→ more protocol-managed infrastructure

LayerZero
→ more configurable application-level security
```

Both introduce external trust and operational assumptions that must be understood before production use.

---

# 6.7 Cross-Chain Transaction Intent

The existing transaction intent can be extended for cross-chain operations.

### Existing intent

```text
TransactionIntent
├── institution
├── account
├── requester
├── chainId
├── target
├── operation
├── asset
├── amount
├── recipient
├── calldataHash
├── nonce
└── deadline
```

### Cross-chain intent

```text
CrossChainIntent
├── institution
├── sourceChain
├── sourceAccount
├── destinationChain
├── destinationAccount
├── asset
├── amount
├── recipient
├── operation
├── nonce
├── deadline
└── messageHash
```

The policy engine can therefore evaluate:

```text
Source chain allowed?
        ↓
Destination chain allowed?
        ↓
Asset allowed?
        ↓
Recipient allowed?
        ↓
Amount allowed?
        ↓
Spending limit available?
        ↓
Approval required?
        ↓
ALLOW
```

This makes cross-chain support an extension of the existing policy model rather than a separate bridge product.

---

# 6.8 Cross-Chain Spending Limits

Cross-chain support introduces an important policy problem.

Suppose the institution has:

```text
Daily institutional limit = $1,000,000
```

and performs:

```text
Ethereum → Polygon      $400,000
Ethereum → Arbitrum     $300,000
Ethereum → Base         $300,000
```

The policy engine should potentially understand:

```text
Total institutional spending
= $1,000,000
```

rather than treating each chain independently.

Therefore we should distinguish between:

### Chain-local limits

```text
Ethereum: $500k/day
Polygon:  $500k/day
```

and:

### Institution-global limits

```text
All chains combined: $1M/day
```

This is an important reason for keeping the policy engine logically independent from individual chains.

---

# 6.9 Cross-Chain Replay Protection

Cross-chain authorization must be bound to its intended context.

At minimum, the authorization should conceptually bind:

```text
sourceChain
destinationChain
sourceAccount
destinationAccount
asset
amount
nonce
deadline
intentHash
```

For example:

```text
hash(
    institution,
    sourceChain,
    destinationChain,
    sourceAccount,
    destinationAccount,
    asset,
    amount,
    nonce,
    deadline
)
```

The destination must not accept:

```text
same authorization
+
different destination
```

or:

```text
same authorization
+
different chain
```

Cross-chain message state should also prevent a message from being processed more than once.

---

# 6.10 Cross-Chain Failure Handling

Cross-chain execution is asynchronous.

Normal EVM execution:

```text
Contract A
    ↓
Contract B
```

can happen inside one transaction.

Cross-chain execution is closer to:

```text
Source
  ↓
Source transaction
  ↓
Message created
  ↓
Verification
  ↓
Delivery
  ↓
Destination execution
```

Therefore:

> A successful source transaction does not necessarily mean that the destination operation has completed.

The architecture should represent lifecycle states.

```text
CREATED
   ↓
AUTHORIZED
   ↓
SENT
   ↓
VERIFIED
   ↓
DELIVERED
   ↓
EXECUTED
```

Potential failure states:

```text
FAILED
EXPIRED
CANCELLED
```

This state machine should be handled by the cross-chain integration layer rather than being hidden inside the policy engine.

---

# 6.11 Finality

Cross-chain finality should not be modeled as:

```text
Source transaction confirmed
=
Destination transaction final
```

Instead:

```text
Source finality
      ↓
Cross-chain verification
      ↓
Message execution
      ↓
Destination finality
```

Finality and verification assumptions depend on:

* source chain;
* destination chain;
* cross-chain protocol;
* protocol configuration.

Therefore the project should rely on the selected protocol's documented verification/finality model rather than inventing a separate finality mechanism.

---

# 6.12 Cross-Chain Router

The architecture should introduce a protocol-independent abstraction:

```text
                    Policy Engine
                         ↓
                  CrossChainIntent
                         ↓
                  CrossChainRouter
                         ↓
              ┌──────────┴──────────┐
              ↓                     ↓
        CCIP Adapter          LayerZero Adapter
              ↓                     ↓
            CCIP                LayerZero
              └──────────┬──────────┘
                         ↓
                    Destination
                         ↓
                Destination Policy
                         ↓
                  Execution Gate
```

The policy engine should not directly depend on CCIP or LayerZero-specific APIs.

This gives us:

* protocol abstraction;
* easier testing;
* ability to change providers;
* separation of policy and transport;
* clearer security boundaries.

---

# 6.13 Destination Policy Gate

A cross-chain message should not automatically execute merely because the bridge protocol delivered it.

The destination policy layer should verify:

```text
Message verified?
        ↓
Source trusted?
        ↓
Destination correct?
        ↓
Intent not replayed?
        ↓
Policy still valid?
        ↓
Deadline valid?
        ↓
Account not paused?
        ↓
Execute
```

This creates two security layers:

```text
Cross-chain protocol security
            +
Institutional policy security
```

The cross-chain protocol does not replace our policy engine.

---

# 6.14 Hub Chain Architecture

A possible architecture is:

```text
              Hub Chain
             /    |    \
            /     |     \
     Ethereum   Polygon  Arbitrum
```

instead of:

```text
Ethereum ─────────→ Polygon
Ethereum ─────────→ Arbitrum
Polygon  ─────────→ Base
```

A hub can provide a centralized coordination point.

However, it also introduces:

* additional transactions;
* additional latency;
* additional fees;
* another operational dependency;
* additional failure modes;
* another chain that must remain available.

Therefore:

> **Do not make the hub chain the default architecture.**

The policy layer can be logically centralized without requiring every cross-chain operation to physically route through one blockchain.

Hub-chain architecture will be evaluated separately in Step 7.

---

# 6.15 MVP Boundary

## Include in architecture

* Cross-chain transaction intent
* Source-chain policy evaluation
* Destination-chain policy enforcement
* Cross-chain spending-limit model
* Replay protection model
* Message lifecycle
* CrossChainRouter abstraction
* Protocol adapter architecture

## Do not build

* Our own bridge
* Our own validator network
* Our own message verification network
* Our own token bridge
* Our own cross-chain consensus mechanism

## Protocol integration

Possible implementations:

```text
CrossChainRouter
├── CCIPAdapter
└── LayerZeroAdapter
```

After further research and implementation assessment, select **one protocol for the demonstrable integration**.

---

# 6.16 Decision

| Requirement                             | Decision               |
| --------------------------------------- | ---------------------- |
| Cross-chain research                    | **Include**            |
| Cross-chain architecture                | **Include**            |
| Build our own bridge                    | **Exclude**            |
| Protocol abstraction                    | **Include**            |
| CCIP                                    | **Candidate**          |
| LayerZero                               | **Candidate**          |
| Cross-chain transaction intent          | **Include**            |
| Destination policy verification         | **Include**            |
| Replay protection                       | **Include**            |
| Cross-chain global spending limits      | **Design requirement** |
| Message lifecycle                       | **Include**            |
| Hub chain                               | **Do not assume**      |
| Multiple chains in architecture         | **Include**            |
| Production-grade multi-chain deployment | **Out of MVP scope**   |

---

# 6.17 Final Research Conclusion

### Decision

> **Include cross-chain capability in the architecture, but do not make CCIP or LayerZero a hard dependency of the core MVP.**

The core policy engine should remain protocol-independent:

```text
Policy Engine
      ↓
CrossChainRouter
      ↓
Protocol Adapter
      ↓
CCIP / LayerZero
      ↓
Destination Policy
      ↓
Execution Gate
```

The project should **not become a bridge**.

Instead, cross-chain infrastructure should be treated as an external transport layer underneath the institutional policy system.

The key architectural principle is:

> **The cross-chain protocol transports and verifies the message; our policy layer determines whether the institutional operation is allowed to execute.**

This also preserves the project's central boundary:

```text
Our System
────────────────────────────────
Policy
Authorization
Limits
Approvals
AllowLists
Execution Gates
Audit
Cross-chain policy
────────────────────────────────
External Infrastructure
Custody
MPC
Bridge / interoperability
Compliance
ERP / invoices
AI / MCP
```

This keeps cross-chain capability relevant to institutional custody without turning the capstone into a bridge implementation.

---

## Sources

* Chainlink CCIP documentation — cross-chain messaging, token transfers, and CCIP architecture.
* LayerZero V2 documentation — OApps, DVNs, Executors, and cross-chain configuration.
* LayerZero OFT documentation — omnichain fungible-token architecture.
* Capstone Todo — Cross-Chain Architecture research requirements.
* Capstone Todo — Hub Chain architecture validation.

# 7. Validate the “Hub Chain” Architecture
> **Decision:** Use direct cross-chain routing with a logical routing layer. Do not introduce a dedicated hub blockchain for the MVP.

Investigating whether payments should be routed through a hub chain. The research goal is to determine whether the hub actually improves the institutional payment workflow or introduces unnecessary complexity, trust assumptions, fees, and failure modes. The original research questions cover direct routing, policy location, hub state, fees, gas, failure handling, and whether the hub could instead be a logical routing layer.

---

## 7.1 Proposed Hub Model

The initial concept was:

```text
Source Chain
     ↓
Cross-chain messaging
     ↓
Hub / Settlement Layer
     ↓
Cross-chain messaging
     ↓
Destination Chain
```

Example:

```text
Ethereum
    ↓
CCIP / LayerZero
    ↓
Hub
    ↓
CCIP / LayerZero
    ↓
Polygon
```

At first glance, this appears attractive because the hub could become the central location for:

* institutional policy;
* routing;
* settlement;
* accounting;
* cross-chain state.

However, this introduces another blockchain into the trust and execution path.

---

# 7.2 Why Would We Need a Hub?

A hub could provide several functions.

### 1. Central policy coordination

```text
                  Hub
                   │
        ┌──────────┼──────────┐
        ↓          ↓          ↓
    Ethereum     Polygon    Arbitrum
```

Policies could theoretically be maintained in one location.

### 2. Cross-chain accounting

The hub could track:

```text
Institution
├── Ethereum spending
├── Polygon spending
├── Arbitrum spending
└── Global spending
```

### 3. Routing

The hub could determine:

```text
Source → Hub → Destination
```

### 4. Settlement coordination

The hub could maintain some representation of:

```text
pending
settled
failed
```

cross-chain operations.

These are legitimate benefits.

But we need to determine whether they require a **blockchain hub**.

---

# 7.3 Direct Source → Destination Routing

The alternative is:

```text
Ethereum
    │
    │ cross-chain message
    ▼
Polygon
```

or:

```text
Ethereum ─────→ Arbitrum
Ethereum ─────→ Base
Polygon  ─────→ Ethereum
```

The policy layer remains logically consistent without requiring every transaction to pass through a third chain.

The architecture becomes:

```text
                  Policy Engine
                       │
                CrossChainRouter
                       │
             ┌─────────┴─────────┐
             ↓                   ↓
        CCIP Adapter       LayerZero Adapter
             │                   │
             └─────────┬─────────┘
                       ↓
                Destination Chain
                       ↓
               Destination Policy
                       ↓
                Execution Gate
```

This is consistent with the architecture established in Step 6.

---

# 7.4 Does the Policy Need to Live on a Hub?

Not necessarily.

The policy engine can be deployed on each supported chain:

```text
Ethereum
┌────────────────────┐
│ Policy Engine      │
│ Execution Gate     │
└────────────────────┘

Polygon
┌────────────────────┐
│ Policy Engine      │
│ Execution Gate     │
└────────────────────┘

Arbitrum
┌────────────────────┐
│ Policy Engine      │
│ Execution Gate     │
└────────────────────┘
```

The logical policy can remain institution-wide while enforcement happens locally.

For example:

```text
Institution Policy

Daily Limit: $1M

Ethereum: $400k
Polygon:  $300k
Base:     $300k

Total:    $1M
```

The important architectural problem is therefore **policy state synchronization**, not necessarily the existence of a hub chain.

---

# 7.5 Logical Hub vs Blockchain Hub

This distinction is important.

## Blockchain Hub

A real blockchain becomes part of the transaction path:

```text
Source
  ↓
Hub Blockchain
  ↓
Destination
```

## Logical Hub

An application/service provides routing and coordination:

```text
                 CrossChainRouter
                  /           \
                 /             \
            Source             Destination
```

The logical router can decide:

```text
source chain
destination chain
protocol
message format
policy context
```

without becoming another settlement blockchain.

### Decision

Use the **logical hub concept**, not a dedicated hub chain.

The `CrossChainRouter` effectively becomes the logical routing layer.

---

# 7.6 What Should the Logical Router Store?

The router should not become a second policy database.

It only needs enough information to coordinate cross-chain execution.

Conceptually:

```text
CrossChainRequest
├── requestId
├── sourceChain
├── destinationChain
├── sourceAccount
├── destinationAccount
├── protocol
├── messageId
├── status
└── timestamps
```

Possible states:

```text
CREATED
AUTHORIZED
SENT
VERIFIED
DELIVERED
EXECUTED
FAILED
EXPIRED
```

The actual institutional policy remains owned by the policy layer.

---

# 7.7 Where Is Policy Enforced?

Policy should be enforced at two points.

### Source

```text
Source Chain
     ↓
Policy Engine
     ↓
ALLOW / APPROVAL / DENY
     ↓
CrossChainRouter
```

The source policy determines whether the institution is allowed to initiate the operation.

### Destination

```text
Destination Chain
     ↓
Message Verification
     ↓
Destination Policy
     ↓
Execution Gate
```

The destination performs an independent final check.

This creates:

```text
Source authorization
        +
Cross-chain message verification
        +
Destination policy enforcement
```

rather than relying on a hub as a single enforcement point.

---

# 7.8 What Would the Hub Actually Store?

If we introduced a blockchain hub, we would need to decide what belongs there.

Potential state:

```text
Institution
Policy
Account
Cross-chain request
Source chain
Destination chain
Asset
Amount
Settlement status
Nonce
Message ID
```

But this creates an important problem.

The hub would become a **new source of truth**.

Now we would have:

```text
Ethereum state
      +
Hub state
      +
Polygon state
```

and cross-chain synchronization between them.

That is additional complexity.

For example:

```text
Hub says:
"Payment settled"

Destination says:
"Execution failed"
```

We now need another reconciliation mechanism.

A hub therefore does not eliminate distributed state problems. It can actually introduce another state boundary.

---

# 7.9 Fees

A hub introduces another execution step.

### Direct

```text
Source
  ↓
Cross-chain protocol
  ↓
Destination
```

Costs include:

```text
source gas
+
cross-chain protocol fee
+
destination execution
```

### Hub

```text
Source
  ↓
Cross-chain protocol
  ↓
Hub
  ↓
Cross-chain protocol
  ↓
Destination
```

Potentially:

```text
source gas
+
first cross-chain message
+
hub execution
+
second cross-chain message
+
destination execution
```

The exact economics depend on the selected protocol and implementation, but architecturally the hub creates an additional execution path and therefore potentially additional cost.

LayerZero, for example, explicitly quotes messaging fees based on configured workers/DVNs, destination execution options, and other parameters.

Therefore, a hub should only exist if the additional step provides a strong functional benefit.

---

# 7.10 Who Pays Gas?

A hub does not automatically solve gas sponsorship.

With direct routing:

```text
Source
  ↓
Sender pays source gas
  ↓
Cross-chain protocol
  ↓
Destination execution
```

The cross-chain protocol can handle destination execution depending on its model.

For example, LayerZero's model includes Executor fees as part of the cross-chain messaging fee.

With a hub:

```text
Source
  ↓
Hub transaction
  ↓
Destination transaction
```

we now have another execution environment whose gas must be funded.

Therefore:

> **A hub should not be justified on the assumption that it makes gas management easier.**

Gas abstraction should instead be handled separately through the account-abstraction/gas-sponsorship architecture researched later.

---

# 7.11 What Happens If Destination Execution Fails?

This is one of the strongest arguments against making the hub a settlement authority.

Consider:

```text
Source
  ↓
Hub
  ↓
Destination
  X
Execution fails
```

What is the hub supposed to do?

Possible states:

```text
Hub:
AUTHORIZED

Destination:
FAILED
```

The system now needs a recovery mechanism.

Possible recovery operations:

```text
retry
cancel
refund
manual intervention
compensation
```

Each adds additional state and security considerations.

With direct routing, the cross-chain protocol's message lifecycle can be treated as an external asynchronous process, while our destination policy layer determines whether the delivered message can execute.

The project does not need to invent another settlement state machine unless the product actually requires it.

---

# 7.12 Does the Hub Reduce Trust?

Not automatically.

Instead, it introduces another trust/security boundary:

```text
Source
   ↓
Bridge / Messaging
   ↓
Hub
   ↓
Bridge / Messaging
   ↓
Destination
```

Compared with:

```text
Source
   ↓
Bridge / Messaging
   ↓
Destination
```

the hub introduces:

* another chain;
* another deployment;
* another set of contracts;
* another state transition;
* another failure point;
* potentially another governance authority.

Therefore:

> **A hub is justified only if its additional functionality outweighs the additional trust and operational complexity.**

For the current capstone scope, that has not been demonstrated.

---

# 7.13 Cross-Chain Policy Synchronization Without a Hub

We can still maintain institution-wide policies.

For example:

```text
Institution Policy
──────────────────────────────
Daily Limit: $1,000,000
Allowed Chains:
  Ethereum
  Polygon
  Arbitrum
Allowed Asset:
  USDC
```

The policy configuration can be synchronized to destination policy contracts through the cross-chain messaging layer.

Conceptually:

```text
Policy Governance
       ↓
Policy Update
       ↓
Cross-chain message
       ↓
Destination Policy Engine
       ↓
New policy version
```

This is more appropriate than introducing a permanent settlement hub.

However, policy synchronization introduces its own security requirements:

* policy version;
* authorized updater;
* destination chain;
* nonce;
* deadline;
* replay protection;
* message verification.

These should be researched when designing the final policy governance architecture.

---

# 7.14 Policy Versioning

Our previous decision to use upgradeable contracts makes this especially relevant.

There are two independent versions:

### Contract implementation version

```text
PolicyEngine V1
      ↓
PolicyEngine V2
```

This is controlled through the upgrade mechanism.

### Institutional policy version

```text
Policy v1
      ↓
Policy v2
      ↓
Policy v3
```

This represents business-rule changes.

Cross-chain synchronization should reference the **policy version**.

For example:

```text
Ethereum
Policy Version 12
      ↓
Cross-chain update
      ↓
Polygon
Policy Version 12
```

The destination can reject an operation if it is using an outdated policy version.

This is a more useful mechanism than putting the policy state on a hub chain.

---

# 7.15 Hub Chain vs Logical Router

| Requirement                    | Blockchain Hub | Logical Router |
| ------------------------------ | -------------: | -------------: |
| Central routing                |            Yes |            Yes |
| Central policy coordination    |            Yes |            Yes |
| Additional blockchain          |            Yes |             No |
| Additional gas                 |         Likely |     No hub gas |
| Additional cross-chain hop     |            Yes |             No |
| Additional trust boundary      |            Yes |        Minimal |
| Central settlement state       |            Yes |       Optional |
| Failure complexity             |         Higher |          Lower |
| Policy synchronization         |       Possible |       Possible |
| Protocol abstraction           |       Possible |            Yes |
| MVP complexity                 |           High |       Moderate |
| Fits current capstone boundary |         Weakly |       Strongly |

---

# 7.16 Direct Routing Architecture

The validated architecture is:

```text
                         Institution
                              │
                              ▼
                       Policy Engine
                              │
                       Cross-chain Intent
                              │
                              ▼
                     CrossChainRouter
                              │
                ┌─────────────┴─────────────┐
                │                           │
           Source Chain                Protocol Adapter
                                            │
                                  ┌─────────┴─────────┐
                                  ↓                   ↓
                                CCIP              LayerZero
                                  │                   │
                                  └─────────┬─────────┘
                                            ↓
                                     Destination Chain
                                            │
                                     Destination Policy
                                            │
                                      Execution Gate
                                            │
                                      Custody Account
```

The router is the **logical hub**.

There is no additional blockchain.

---

# 7.17 When Would a Real Hub Become Justified?

A future version could justify a hub if we introduce requirements such as:

### Centralized cross-chain settlement

```text
Multiple chains
      ↓
Central settlement ledger
```

### Complex netting

Instead of:

```text
A → B
A → C
B → C
```

the system could net obligations centrally.

### Institutional accounting

The hub could maintain a canonical accounting/settlement state across chains.

### Non-EVM coordination

A hub could potentially simplify interactions between chains with very different execution models.

### Complex cross-chain governance

A hub could become useful if institutional governance itself requires a canonical coordination layer.

These are legitimate future directions, but they are beyond the current MVP.

---

# 7.18 Final Decision

| Question                               | Decision                                                   |
| -------------------------------------- | ---------------------------------------------------------- |
| Why do we need a hub?                  | No mandatory requirement identified                        |
| Can source → destination be direct?    | **Yes**                                                    |
| What does a hub store?                 | Would introduce another cross-chain state source           |
| Where is policy enforced?              | **Source + destination**                                   |
| Where are fees calculated?             | By selected cross-chain protocol                           |
| Who pays gas?                          | Source/user/account or protocol execution mechanism        |
| What happens if destination fails?     | Track asynchronous message status and handle retry/failure |
| Does a hub introduce additional trust? | **Yes**                                                    |
| Could the hub be logical instead?      | **Yes**                                                    |
| Blockchain hub for MVP?                | **No**                                                     |
| Logical routing layer?                 | **Yes**                                                    |
| Direct cross-chain routing?            | **Yes**                                                    |
| Defer dedicated hub?                   | **Yes**                                                    |

---

# 7.19 Final Research Conclusion

> **Use direct source-to-destination cross-chain routing with a logical `CrossChainRouter`. Do not introduce a dedicated hub blockchain into the MVP.**

The main reason is that the functions we want from a hub can largely be separated:

```text
Routing
→ CrossChainRouter

Policy
→ Policy Engine

Authorization
→ Approval Manager

Cross-chain transport
→ CCIP / LayerZero

Destination enforcement
→ Destination Policy + Execution Gate

Custody
→ External custody / smart account

Compliance
→ External service
```

This preserves clean system boundaries.

The architecture becomes:

```text
              ┌──────────────────────────┐
              │     Institutional Layer  │
              │                          │
              │ Policy + Approval        │
              │ Limits + Allowlists      │
              │ Governance + Audit       │
              └────────────┬─────────────┘
                           │
                    CrossChainRouter
                           │
               ┌───────────┴───────────┐
               │                       │
             Chain A                Chain B
               │                       │
        Policy + Gate            Policy + Gate
               │                       │
             Custody                 Custody
```

### Architectural principle

> **Centralize policy logically, not necessarily on a blockchain.**

This gives us the institutional-wide policy model we want without introducing an unnecessary settlement chain.

The dedicated hub concept can remain a **future extension** if later requirements demonstrate a need for canonical cross-chain accounting, netting, settlement coordination, or complex governance.

---

## Sources

* Chainlink Developer Documentation — CCIP and multi-chain tooling.
* Chainlink CCIP release notes — current support/deprecation changes demonstrate that supported networks should be treated as a time-sensitive integration concern.
* LayerZero V2 Endpoint documentation — message channels, nonces, payload hashes, exactly-once processing, configurable messaging libraries, DVNs, Executors, and finality configuration.
* LayerZero V2 protocol documentation — message fees and DVN/Executor fee model.
* Capstone Todo — Hub Chain Architecture research requirements.
