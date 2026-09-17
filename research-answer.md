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