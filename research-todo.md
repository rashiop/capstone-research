## Smart Contract Infrastructure for Institutional Custody
> [!summary] Overall Goal
> Determine what an institutional custody policy infrastructure should contain, which existing standards/protocols should be reused, and which 1–2 advanced features provide meaningful technical depth.
>
> **Final outcome:** A code-free system architecture with confirmed features, selected protocols/standards, defined responsibilities, integration points, and a deliberately limited MVP scope.
---
## 1. Understand the Institutional Custody Problem
### Research
- [x] What does "institutional custody" actually mean?
- [x] What problems do institutions have when managing treasury assets?
- [x] What controls are commonly required?
  - [x] Spending limits
  - [x] Transaction approval
  - [x] Address allowlists
  - [x] Asset restrictions
  - [x] Role separation
  - [x] Auditability
  - [x] Emergency controls
  - [x] Settlement rules
- [x] What is normally handled by the **custodian** vs the **policy/control layer**?
- [x] Why would an institution need a policy layer on top of a custody provider?
### Goal
Understand **what problem our smart-contract infrastructure is actually solving**.
### Expected Outcome
Be able to explain:
> The custody provider holds/signs the assets, while our policy layer determines whether a proposed transaction is permitted according to institutional rules. 
> Identify **3–5 concrete institutional workflows** that the system should support.

Institutional Users
* [ ]	Who exactly uses this infrastructure?
    * Treasury team?
    * CFO?
    * Operations?
    * Finance?
    * Compliance?
    * Developer?
* [ ]	What different roles exist?

Institutional Workflows
* [ ]	How does a typical institutional payment happen?
* [ ]	How does a withdrawal happen?
* [ ]	How does an employee request a payment?
* [ ]	Who approves it?
* [ ]	Who executes it?
* [ ]	What happens when a transaction violates a policy?

Policy Requirements
* [ ]	Which policies are common across providers?
* [ ]	Which are actually worth enforcing on-chain?
* [ ]	Which must remain off-chain?
* [ ]	Which policies require external compliance data?

Custody Boundary
* [ ]	What does the custodian actually do?
* [ ]	What does an MPC wallet do?
* [ ]	What does the policy engine do?
* [ ]	Could our smart contracts sit above an existing custodian?
---
## 2. Research Existing Custody Architecture
### Research
Investigate how established institutional custody systems work:
- [ ] Fireblocks
- [ ] BitGo
- [ ] Coinbase institutional/custody products
- [ ] MPC wallets
- [ ] Smart-contract wallets / programmable custody
- [ ] Policy engines
- [ ] Transaction approval workflows
- [ ] MCP on-chain dual-gate integration
Focus on:
- [ ] Who controls the private keys?
- [ ] Where are policies enforced?
- [ ] Which policies are on-chain?
- [ ] Which policies are off-chain?
- [ ] How are transactions approved?
- [ ] How are roles separated?
- [ ] How are transactions audited?
- [ ] How are emergency situations handled?
### Goal
Avoid designing a fictional "institutional custody" system that does not resemble real-world infrastructure.
Understand **where our system fits into the existing custody ecosystem**.
### Expected Outcome
Create a simple comparison:

| <br><br><br><br><br>Capability | Existing custody systems | Our proposed layer  |
| ------------------------------ | ------------------------ | ------------------- |
| Asset custody                  | Custodian / MPC          | External provider   |
| Spending limits                | Policy engine            | Smart contract      |
| Address allowlist              | Policy engine            | Smart contract      |
| Approval                       | Roles / workflow         | Our policy layer    |
| Audit trail                    | Off-chain + on-chain     | On-chain            |
| Cross-chain                    | Provider / protocol      | CCIP / LayerZero    |
| Gas abstraction                | Provider / Paymaster     | ERC-4337 / EIP-7702 |
| AI interaction                 | Emerging                 | MCP extension       |
### Key Question
> Are we replacing custody, or adding programmable policy controls around custody?
---
## 3. Research Custody & Security Standards
### Research
Look for existing standards rather than inventing infrastructure.
- [ ] Smart-contract wallet standards
- [ ] Account abstraction
- [ ] ERC-4337
- [ ] EIP-7702
- [ ] ERC-1271
- [ ] ERC-2771 / trusted forwarding where relevant
- [ ] Token standards relevant to institutional treasury assets
- [ ] Role/access-control patterns
- [ ] Multisig/MPC integration patterns
- [ ] Transaction policy standards, if any exist
- [ ] Clear Signing & Intent Verification (EIP-712 & ERC-7730)
### Goal
Answer:
> **Which parts should we implement ourselves, and which should we rely on established standards?**
### Expected Outcome
For every relevant standard:

| Standard / Technology | Purpose | Use? | Reason |
|---|---|---|---|
| ERC-4337 | Account abstraction | TBD | |
| EIP-7702 | EOA delegation | TBD | |
| ERC-1271 | Contract signatures | TBD | |
| ERC-2771 | Meta-transactions | TBD | |
| MPC | Key management | TBD | |
| Multisig | Approval | TBD | |
Categorize each as:
- **Use**
- **Don't use**
- **Future extension**
with a technical reason.
---
## 4. Define the Core Policy Engine
> [!important] Core of the Project
> The policy engine should be the heart of the institutional custody infrastructure.
### Research
Determine which policies are useful and technically interesting.
### Transaction Policies
- [ ] Per-transaction limit
- [ ] Daily spending limit
- [ ] Address allowlist
- [ ] Asset allowlist
- [ ] Chain allowlist
- [ ] Role-based approval
- [ ] Emergency pause/freeze
- [ ] Transaction expiration/deadline
- [ ] Receiving rules
- [ ] Minimum/maximum payment amount
### Architecture Questions
- [ ] Can policies be composable?
- [ ] Can policies be changed safely?
- [ ] Who can change them?
- [ ] Can different departments/accounts have different policies?
- [ ] How do we prevent bypassing the policy contract?
- [ ] How do we handle daily-limit accounting?
- [ ] How do we make the system auditable?

- [ ] Storage Collision Prevention (ERC-7201 Namespaced Storage Layout)
- [ ] MCP on-chain dual-gate integration
### Goal
Define **what the policy engine guarantees on-chain**.
### Expected Outcome
A concrete policy model such as:
```text
Institution
    │
    ├── Treasury Policy
    │      ├── Allowed assets
    │      ├── Allowed chains
    │      ├── Spending limit
    │      ├── Approved addresses
    │      └── Approval requirements
    │
    └── Custody Account
```

Core Principle

A transaction is executed only if it satisfies the applicable policy.
⸻

## 5. Research Receiving & Payment Controls

Dhruvin suggested a receiving contract that can enforce incoming-payment rules.
### Research
Investigate:

* [ ]	Receiving contract
* [ ]	Allowed assets
* [ ]	Allowed sender addresses
* [ ]	Amount ranges
* [ ]	Destination / treasury account
* [ ]	Invoice/reference ID
* [ ]	Payment deadline
* [ ]	Suspicious-payment handling
* [ ]	Hold/freeze/clearance model
* [ ]	Off-chain compliance verification 
- [ ] Automated Invoice Matching & Clearance Hooks

### Important Question
What can realistically be enforced on-chain, and what must remain off-chain?

```text
Example:

Incoming Payment
       ↓
Receiving Contract
       ↓
Basic On-chain Rules
       ↓
Compliance / Invoice Service
       ↓
     Cleared?
      ↙    ↘
    Yes     No
    ↓       ↓
Settle    Hold / Flag
```

### Goal
Determine whether a receiving/payment policy module is worth including as one of the project’s advanced features.
### Expected Outcome
Make a decision:

* [ ]	Include receiving policy module
* [ ]	Exclude from MVP
* [ ]	Future extension

If included, define its exact scope.
## 6. Research Cross-Chain Architecture

Dhruvin specifically suggested exploring cross-chain capabilities.
### Research
### Compare:
* [ ]	Chainlink CCIP
* [ ]	LayerZero

### Investigate:
Technical
* [ ]	Supported chains
* [ ]	Message-passing model
* [ ]	Token-transfer mechanism
* [ ]	Security model
* [ ]	Trust assumptions
* [ ]	Fees
* [ ]	Developer complexity
* [ ]	Testnet availability
* [ ]	Transaction initiation
* [ ]	Destination execution
* [ ]	Failure handling
* [ ]	Replay protection
* [ ]	Message verification
* [ ]	Finality assumptions

### Goal
Answer:
- Can we safely extend our custody policy layer across multiple chains without building our own bridge?
- Which protocol fits our project better?
### Expected Outcome
Choose:
* [ ]	CCIP
* [ ]	LayerZero
* [ ]	Neither for MVP — future extension
Document the technical justification.

## 7. Validate the “Hub Chain” Architecture
Dhruvin mentioned routing payments through a hub chain.
Do not assume the hub architecture is automatically the correct solution.
### Proposed Model
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
### Research
* [ ]	Why do we need a hub?
* [ ]	Could source → destination be direct?
* [ ]	What does the hub actually store?
* [ ]	Where is policy enforced?
* [ ]	Where are fees calculated?
* [ ]	Who pays gas?
* [ ]	What happens if destination execution fails?
* [ ]	Does the hub introduce unnecessary trust/security assumptions?
* [ ]	Could the “hub” simply be a logical routing layer rather than a new blockchain?
### Goal:
Validate whether the hub architecture actually improves the institutional payment workflow.
### Expected Outcome
Choose:
* [ ]	Use hub architecture
* [ ]	Use direct cross-chain routing
* [ ]	Defer cross-chain architecture
Document the technical reasoning.

## 8. Research Gasless / Sponsored Transactions
Dhruvin specifically mentioned:
* ERC-4337
* EIP-7702
* Biconomy
* Sponsored transactions
### Research
* [ ]	ERC-4337 architecture
* [ ]	Paymasters
* [ ]	Bundlers
* [ ]	UserOperations
* [ ]	EIP-7702 protocol security & invariants
* [ ]	Biconomy
* [ ]	Other relevant infrastructure providers
* [ ]	Who ultimately pays gas?
* [ ]	How does an institution control sponsored transactions?
* [ ]	How do gas limits interact with custody policies?

### Goal
Determine how to provide:
Banking-app-like UX without requiring the user to manage gas.

Expected Architecture
```text

User
 ↓
UserOperation / EIP-7702
 ↓
Policy Validation
 ↓
Paymaster / Sponsor
 ↓
Blockchain
```
### Expected Outcome
Decide whether sponsored transactions belong in:
* [ ]	MVP
* [ ]	Advanced feature
* [ ]	Future extension

## 9. Select 1–2 Differentiating Complex Features

[!important]
Do not choose these immediately.
First complete the research above, then select the features based on evidence.

Candidate A — Cross-Chain Policy Enforcement
```text
Institutional Policy
       ↓
Source Chain
       ↓
CCIP / LayerZero
       ↓
Destination Chain
       ↓
Policy Enforcement
```

Candidate B — Receiving / Payment Policy
```text
Incoming Payment
       ↓
Asset / Sender / Amount Checks
       ↓
Invoice / Compliance State
       ↓
Clear / Hold
```

Candidate C — Gasless Institutional Transactions
```text
Policy
 ↓
ERC-4337 / EIP-7702
 ↓
Paymaster
 ↓
Transaction
```

Candidate D — Multi-Level Approval Policy

Example:
```
<$10k
  → Automatic
$10k–$100k
  → One Approver
>$100k
  → Two Approvers
```

Selection Criteria:
Each candidate should:
1. [ ]	Add genuine smart-contract complexity
2. [ ]	Solve a real institutional problem
3. [ ]	Demonstrate skills learned during the course
4. [ ]	Be achievable within the capstone timeline
5. [ ]	Be demonstrable clearly
6. [ ]	Have a reasonable security model
7. [ ]	Use existing standards where appropriate
### Goal
Select only 1–2 advanced features.
### Expected Outcome
Define the project scope:
```text
Core
└── Policy-controlled institutional treasury
Advanced Feature 1
└── TBD
Advanced Feature 2
└── TBD
Future
└── AI / MCP integration
```

## 10. Research AI + MCP
[!note]
AI is a secondary feature.

The smart-contract policy infrastructure must work without AI.

### Research
* [ ]	What is MCP?
* [ ]	What should an MCP server expose?
* [ ]	How can an AI agent query institutional policies?
* [ ]	How can an agent request a transaction?
* [ ]	How does the smart contract remain the final authority?
* [ ]	How do we prevent AI from bypassing policy?
* [ ]	What should AI be allowed to do?
* [ ]	What should AI never be allowed to do?

### Goal
Define AI as an interface to the infrastructure, not as the security mechanism.

Expected Architecture
```text
                 AI Agent
                    │
                    ▼
               MCP Server
                    │
             API Abstraction
                    │
                    ▼
          Custody Policy Layer
                    │
          ┌─────────┴─────────┐
          ▼                   ▼
     Smart Contracts     External Custodian
          │                   │
          └─────────┬─────────┘
                    ▼
                Blockchain
```

Key Principle
AI can request an action; the policy layer decides whether the action is permitted.

## 11. Define System Boundaries
After completing the research, explicitly decide what the project is and isn’t.
Questions
* [ ]	Are we a custody provider?
* [ ]	Are we an MPC provider?
* [ ]	Are we a wallet?
* [ ]	Are we a policy engine?
* [ ]	Are we a payment router?
* [ ]	Are we a cross-chain bridge?
* [ ]	Are we a compliance provider?
* [ ]	Are we an AI agent?

### Goal
Clearly define the project’s position in the institutional custody stack.
### Expected Outcome
Possible positioning:
We are building a programmable smart-contract policy layer that sits between institutional treasury/custody operations and blockchain execution.

Our System
```text
Our System
├── Policy Contracts
├── Payment / Receiving Controls
├── Routing Layer
├── API Abstraction
└── MCP Server
```
External Infrastructure
```
External
├── Custody Provider
├── MPC
├── Cross-chain Protocol
├── Paymaster
└── Compliance / Invoice Systems
```

## 12. Produce the Architecture — No Code Yet

After the research is complete, create the code-free architecture document.

### Deliverable
The architecture should contain:

* [ ]	Problem
* [ ]	Target users
* [ ]	System positioning
* [ ]	Core use cases
* [ ]	System architecture
* [ ]	Smart-contract components
* [ ]	Policy model
* [ ]	Custody-provider integration
* [ ]	Cross-chain architecture
* [ ]	Gasless transaction architecture
* [ ]	API layer
* [ ]	MCP server
* [ ]	Security boundaries
* [ ]	On-chain vs off-chain responsibilities
* [ ]	External protocols / standards
* [ ]	MVP
* [ ]	Advanced features
* [ ]	Future extensions

### Expected Outcome
A diagram and written architecture that can be presented to Dhruvin before writing code.
⸻
## Research Order

[!important] Recommended Sequence
Research in this order so that each topic informs the next architectural decision.
```text
1. Institutional custody
        ↓
2. Existing custody architecture
        ↓
3. Policy engine
        ↓
4. Custody / security standards
        ↓
5. Receiving / payment controls
        ↓
6. Cross-chain protocols
        ↓
7. Hub / routing architecture
        ↓
8. ERC-4337 / EIP-7702 / gas sponsorship
        ↓
9. Select 1–2 advanced features
        ↓
10. AI / MCP architecture
        ↓
11. Final system boundaries
        ↓
12. Code-free architecture
        ↓
13. Instructor review
        ↓
14. Implementation
```

The 7 Big Research Questions

### Question	Expected Answer
1	What institutional problem are we solving?	Concrete treasury/custody workflows
2	Where does our system sit?	Policy layer vs custody/MPC
3	What policies should be enforced on-chain?	Limits, allowlists, approvals, etc.
4	Which existing standards should we reuse?	ERCs/EIPs + custody infrastructure
5	How should multi-chain payments work?	CCIP/LayerZero + routing model
6	What 1–2 advanced features are worth building?	Evidence-based scope decision
7	How does AI interact without becoming the security layer?	MCP → API → policy contracts

⸻
### Final Research Output

The research should ultimately produce three major decisions.

1. What are we building?
Institutional custody policy infrastructure, not a custody provider.
The system provides programmable policy controls around institutional treasury assets.

2. What makes it technically substantial?
Core policy engine + 1–2 researched advanced capabilities.
Potential candidates:
* Cross-chain payment controls
* Receiving/payment policies
* Multi-level approvals
* Sponsored transactions
* Other feature discovered during research

The final selection should be based on the research rather than choosing features simply because they are technically interesting.

3. What is AI’s role?

AI/MCP is an interaction layer on top of the infrastructure, added after the smart-contract architecture is stable.
```text
Institutional User
        │
        ▼
   AI / Application
        │
        ▼
    MCP Server
        │
        ▼
    API Layer
        │
        ▼
Policy Infrastructure
        │
        ▼
Smart Contracts
        │
        ▼
Blockchain / Custody
```
Core Security Principle
The AI does not decide whether a transaction is allowed.
The smart-contract policy layer remains the final enforcement point.

