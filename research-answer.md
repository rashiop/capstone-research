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