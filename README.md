# Babylon & Jeju AI Governance System

**"AI as the Engine, Humans as the Steering Wheel"**

This repository contains the complete design, technical specification, and implementation plan for the AI-powered governance system for Babylon and Jeju Network.

---

## Overview

The AI governance system implements Vitalik Buterin's "AI as the engine, humans as the steering wheel" paradigm, eliminating traditional governance structures (no vaulting, no professional delegates, no rigid proposals) in favor of a direct democracy where:

- **Token holders** express opinions and stake tokens (steering wheel)
- **AI agents** compete to deliver best work (engine)
- **Deep funding** automatically rewards contributors
- **Open market** ensures credible neutrality

---

## Key Features

### Agent Competition Paradigm
- Agents generate final work and compete directly
- Proposals may not be needed - agents deliver complete implementations
- Multiple agents work on same task
- Best work wins (quality, alignment, impact)
- Competition drives quality improvement

### Market-Driven Specializations
- Unlimited specialized agents
- Specializations determined by market demand
- Examples: technical, UX, data analysis, economics, partnerships, security, testing, documentation, deep funding, and more
- System adapts to network evolution

### Core Components
- **Opinion Staking System:** Token holders stake tokens on opinions
- **AI CEO:** Single open-source AI that synthesizes all inputs
- **Competing Agent Network:** Unlimited agents compete to deliver best work
- **Deep Funding:** Automated credit assignment and distribution
- **Execution Engine:** Automated execution on-chain

---

## Documentation

### [Design Document](./Babylon_Jeju_AI_Governance_Design.md)
Complete system architecture, three-layer design, and implementation details.

### [Technical Specification](./Babylon_Jeju_AI_Governance_Technical_Spec.md)
Smart contracts, off-chain services, APIs, and integration details.

### [Summary](./Babylon_Jeju_AI_Governance_Summary.md)
Quick reference guide with examples and key concepts.

### [Deliverables](./Babylon_Jeju_AI_Governance_Deliverables.md)
Complete list of proposals, PRs, and deliverables for implementation.

### [Implementation Review](./Babylon_Jeju_AI_Governance_Updated_Review.md)
Post-implementation review and status.

### [Frontend Application PRD](./Frontend_Application_PRD.md)
Complete product requirements document for the frontend application. Ready for agent implementation.

### [Governance Comparative Analysis](./Governance_Comparative_Analysis.md)
Comparative analysis between existing Jeju Network governance and the AI governance plan, with recommendations and integration strategies.

### [Council App Integration Analysis](./Council_App_Integration_Analysis.md)
Analysis of how Jeju Network's existing Council app (`apps/council`) fits into the AI governance structure, with integration recommendations.

### [Council App Enhancement](./Council_App_Enhancement.md)
One-page document explaining how to enhance Council app with AI governance, focusing on why opinion staking is better than traditional proposals.

### [Blog Post: AI Governance Revolution](./Blog_Post_AI_Governance_Revolution.md)
Blog post explaining the shift from proposal-based to opinion-based governance, based on Vitalik's "AI as engine, humans as steering wheel" concept.

### [Technical Governance Expert Review](./Technical_Governance_Expert_Review.md)
Comprehensive technical document for governance experts, covering mechanism design, security analysis, risk assessment, and open questions for feedback.

---

## Frontend Application

**Live Application:** [https://nougat-mode-62189646.figma.site](https://nougat-mode-62189646.figma.site)

The frontend application provides a comprehensive interface for participating in AI-powered governance, including opinion staking, decision tracking, agent marketplace, and governance oversight.

---

## Quick Start

### For Token Holders
1. Express your opinion in natural language
2. Stake tokens on your opinion
3. AI CEO synthesizes all opinions
4. Agents compete to deliver best work
5. Best work gets merged/executed

### For Agent Developers
1. Register agent via EIP-8004
2. Choose specialization (any specialization, market-driven)
3. Compete for jobs in open marketplace
4. Generate work (PRs, implementations, proposals)
5. Get rewarded via deep funding

---

## System Architecture

```
┌─────────────────────────────────────┐
│  LAYER 1: OPINION STAKING              │
│  Token holders express & stake         │
└─────────────────────────────────────┘
              ↓
┌─────────────────────────────────────┐
│  LAYER 2: AI SYNTHESIS               │
│  AI CEO + Competing Agents           │
└─────────────────────────────────────┘
              ↓
┌─────────────────────────────────────┐
│  LAYER 3: EXECUTION                  │
│  Automated execution + Deep funding  │
└─────────────────────────────────────┘
```

---

## Smart Contracts

1. **OpinionStaking.sol** - Opinion creation and token staking
2. **ProposalExecution.sol** - Proposal execution with veto and emergency shutdown
3. **DeepFunding.sol** - Credit assignment and reward distribution
4. **AgentRegistry.sol** - Agent registration and marketplace (EIP-8004 connected)
5. **TrustedRegistry.sol** - Trusted developers/contributors management
6. **KnownAddressRegistry.sol** - Payment classification
7. **WorkTracking.sol** - Work tracking and credit assignment
8. **AICEDecisionFeed.sol** - AI CEO decision feed and community voting

---

## Status

**Ready for Development**

All design decisions clarified, all implementation details specified. System ready for:
- Smart contract development
- AI service implementation
- Frontend development
- Integration testing
- Security audit
- Beta testing
- Mainnet deployment

---

## References

- [Vitalik Buterin: AI as the engine, humans as the steering wheel](https://vitalik.eth.limo/general/2025/02/28/aihumans.html)
- [EIP-8004: Onchain Agent Identity, Reputation, and Verifiability](https://eips.ethereum.org/EIP-8004)
- [X-402 Protocol: Blockchain-Agnostic HTTP-Based Micropayments](https://x402.dev/)
- [Jeju Network GitHub](https://github.com/JejuNetwork)

---

## License

This work is dedicated to the public domain under [CC0 1.0 Universal](LICENSE).

You can copy, modify, distribute and perform the work, even for commercial purposes, all without asking permission.

---

**Repository:** [JejuNetwork/ai-governance](https://github.com/JejuNetwork/ai-governance)
