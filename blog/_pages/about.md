---
layout: page
title: About
permalink: /about/
---

# About ThirdKey Research

ThirdKey Research is dedicated to advancing AI security through our "Zero Trust for AI" approach. We believe that **every AI interaction should be verified, every agent should be authenticated, and every decision should be auditable**.

## Our Mission

As AI agents become increasingly autonomous — managing infrastructure, handling sensitive data, negotiating with other agents — the need for robust security frameworks has never been more urgent. Traditional security models that rely on perimeter defense are insufficient for the dynamic, distributed nature of agentic AI systems.

We focus on extending Zero Trust principles to artificial intelligence, applying the philosophy of "never trust, always verify" across the entire agent lifecycle: from tool integrity to agent identity to runtime policy enforcement.

## The ThirdKey Trust Stack

Our core work is a three-layer cryptographic trust architecture for AI agents:

### SchemaPin — Tool Integrity
**Cryptographic Security for AI Tool Schemas**

SchemaPin prevents "MCP Rug Pull" attacks by enabling developers to cryptographically sign their tool schemas and allowing clients to verify that schemas have not been altered since publication. It answers: *are the tools this agent uses legitimate and untampered?*

- **Website**: [schemapin.org](https://schemapin.org)
- **Features**: ECDSA P-256 signatures, Trust-On-First-Use key pinning, `.well-known` discovery
- **Languages**: Rust, Python, JavaScript, Go
- **License**: MIT

### AgentPin — Agent Identity
**Domain-Anchored Cryptographic Identity for AI Agents**

AgentPin enables organizations to issue verifiable cryptographic credentials to their AI agents, anchored to domain ownership. Verifiers can confirm an agent's identity, capabilities, and authorization through a 12-step verification protocol. It answers: *is this agent who it claims to be?*

- **Repository**: [github.com/thirdkeyai/agentpin](https://github.com/thirdkeyai/agentpin)
- **Features**: ES256 JWTs, domain-anchored discovery, delegation chains, trust bundles, mutual authentication
- **Languages**: Rust, JavaScript, Python
- **License**: MIT

### Symbiont — Runtime Policy Enforcement
**AI-Native Agent Framework**

Symbiont is a framework for building autonomous, policy-aware agents that can safely collaborate with humans, other agents, and large language models. It enforces runtime policy based on verified identity and tool integrity. It answers: *does policy allow this verified agent to perform this action?*

- **Website**: [symbiont.dev](https://symbiont.dev)
- **Features**: DSL for agent definitions, HTTP API, cron scheduling, channel adapters, MCP integration
- **Languages**: Rust
- **License**: MIT

## Security Research

### VectorSmuggle
**Vector-Based Data Exfiltration Research**

A proof-of-concept demonstrating vector-based data exfiltration techniques in AI/ML environments, illustrating potential risks in RAG systems and providing tools for defensive analysis.

- **Repository**: [github.com/jaschadub/VectorSmuggle](https://github.com/jaschadub/VectorSmuggle)
- **License**: MIT

### AgentNull
**AI System Security Threat Catalog + Proof-of-Concepts**

A security research project focused on cataloging and demonstrating threats specific to AI systems, providing both theoretical frameworks and practical proof-of-concepts for AI security vulnerabilities.

- **Repository**: [github.com/jaschadub/AgentNull](https://github.com/jaschadub/AgentNull)

## Research Areas

- **Agent-Tool Interface Security** — Cryptographic verification of tool schemas, secure communication protocols, trust establishment and key management
- **Agent Identity & Authentication** — Domain-anchored credentials, delegation chains, capability scoping, mutual verification
- **Runtime Policy Enforcement** — Automated governance, policy-as-code, real-time constraint enforcement
- **Threat Intelligence** — AI-specific attack vectors, multi-agent system risks, supply chain security

## Contact

- **Email**: [research@thirdkey.ai](mailto:research@thirdkey.ai)
- **GitHub**: [ThirdKeyAI](https://github.com/ThirdKeyAI)
- **Twitter**: [@ThirdKeyAI](https://twitter.com/ThirdKeyAI)
- **Hugging Face**: [thirdkey](https://huggingface.co/thirdkey)

---

*ThirdKey Research is committed to advancing AI security through practical, open-source solutions. Follow our work and join the conversation about building a more secure AI future.*
