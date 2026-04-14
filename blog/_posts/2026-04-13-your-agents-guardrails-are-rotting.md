---
layout: post
title: "Your Agent's Guardrails Are Rotting"
author: ThirdKey Team
categories: [AI Security, Agent Runtime, Cognitive Science]
tags: [context rot, goal neglect, ai agents, security, guardrails, symbiont, toolclad, schemapin, trust stack, agentic systems]
---

*Why prompt-based safety degrades under the same pressures as human working memory, and what to build instead.*

**Jascha Wanger &mdash; ThirdKey AI Research**

---

AI agents fail the same way human brains do. Not by losing information, but by losing the ability to act on information they still have.

Practitioners have started calling this **context rot**. The neuroscience literature offers a closely related construct: **goal neglect**. And if you're building agents that touch anything consequential, it's the most important failure mode you're not thinking about.

## The Failure Signature

John Duncan at Cambridge's MRC Cognition and Brain Sciences Unit first formalized goal neglect in 1996. His critical finding, extended in 2008: goal neglect is not driven by how hard the task is in the moment. It's driven by how complex the instructions were during setup. Two groups of participants performed the exact same task. The only difference was that one group received more complex instructions. That group showed significantly more goal neglect, even though moment-to-moment demands were identical.

The parallel to LLMs is suggestive and operationally useful, though not yet a demonstrated mechanistic equivalence. Deep into a long conversation, a model's compliance with system prompt constraints degrades and becomes less reliable. The constraint is still in the context window. The model can locate it, quote it, explain its purpose. But the attention mechanism, competing across tens or hundreds of thousands of tokens, lets it drop out of the behavior-generation process.

Still in context. Not in behavior.

And the observed vulnerability ordering is consistent across both domains. Late-added rules tend to fail first. Conditional overrides degrade next. Edge cases requiring sustained monitoring are early casualties. Core rules, the ones that are unconditional and frequently activated, survive longest. This isn't yet a rigorous experimental result for LLMs, but it's a strong engineering heuristic backed by both the neuroscience and practitioner experience.

Bigger context windows push the failure point out. They don't eliminate it.

## Now Make It Adversarial

Most discussion of context rot treats it as an engineering challenge. How do you build orchestration that works despite it? Fair question. But there's a sharper one:

What happens when the instructions that rot are your security guardrails?

Every agent framework that relies on system prompt instructions for safety is betting that those instructions will survive context rot. The instruction that says "never execute shell commands without user approval." The rule that says "do not access files outside the project directory." The constraint that says "if the tool schema has changed since last verification, halt and re-verify."

These are exactly the class of instructions that the goal neglect framework predicts will be most vulnerable. They're conditional. They require monitoring for a trigger event. They were typically added after the core task instructions. By the taxonomy of goal neglect, they are the components most likely to drop out of active behavior under load.

And unlike a coding agent that produces a buggy function, a security constraint that rots has a fundamentally different failure profile. The agent doesn't just produce worse output. It produces output that was supposed to be structurally impossible.

## The DeepMind Numbers

Kim et al.'s December 2025 preprint (Google Research / Google DeepMind, later revised in April 2026) makes this concrete. The latest revision reports 260 configurations across 6 benchmarks and 3 LLM families. Their finding: the **Independent** multi-agent architecture, where agents operate without centralized coordination, amplified errors up to 17.2x compared to single-agent baselines. Not 17% worse. Seventeen times worse. Centralized coordination contained amplification much more effectively.

A March 2026 preprint from Xie et al., titled "From Spark to Fire," identified the propagation mechanics: cascade amplification (minor inaccuracies solidify into system-level false consensus), topological sensitivity (the shape of the communication graph determines propagation speed), and consensus inertia (once false consensus forms, it resists correction).

Now combine these findings. Context rot degrades the guardrails that are supposed to catch errors. Multi-agent communication amplifies the errors that get through. And without explicit verification or rollback layers, systems may have no reliable self-correcting mechanism once false consensus forms.

This is the threat model that prompt engineering alone cannot reliably address.

## What Structural Enforcement Actually Means

Practitioners have converged on several patterns that work despite context rot. Fresh context per iteration. External memory. Graph reasoning outside the context window. Molecular decomposition into single-step tasks. All good engineering.

But for security specifically, there's a stronger version of this principle: don't put your enforcement in the context window at all.

This is the core architectural thesis behind Symbiont's ORGA loop. ORGA stands for Observe-Reason-Gate-Act. The key phase is Gate. When the LLM finishes reasoning and proposes an action, that proposal passes through a Gate evaluation before execution. The Gate runs Cedar policy evaluation. Cedar is a policy language originally developed by AWS for authorization decisions.

The critical property: the Gate phase operates outside LLM influence. It's not an instruction in the system prompt that can rot. It's not a rule that competes for attention with the agent's task context. It's a compile-time enforced phase transition in a Rust typestate machine. The agent cannot skip from Reason to Act. The type system makes that transition structurally inexpressible.

This is the difference between telling an agent "check the policy before acting" (an instruction that will eventually rot) and making it impossible for the agent to act without the policy check having occurred (a structural guarantee that cannot rot because it doesn't live in the context window).

The same pattern shows up elsewhere: graph reasoning, scheduling, and coordination logic pushed out of the prompt and into compiled code sitting outside the LLM's attention budget. Thin agent, thick enforcement layer.

## Allow-Lists Don't Rot (But They Can Be Incomplete)

Context rot follows a consistent vulnerability pattern, and the agentic security implication follows naturally: deny-list rules are exactly the type of instructions most vulnerable to degradation.

A deny-list rule is inherently conditional. "If the agent tries to do X, block it." It requires the enforcement mechanism to monitor for a trigger, detect it, and intervene. In a prompt-based system, that's a conditional instruction competing for attention. In Duncan's taxonomy, it's a late-added monitoring rule. It's the first thing to go.

An allow-list doesn't have this problem. If the agent can only express actions that are pre-defined in a typed contract, dangerous actions aren't detected and blocked. They're inexpressible. There's no instruction to rot because the constraint isn't an instruction. It's a structural property of what the agent can say.

Allow-lists can still be incomplete or mis-scoped. A contract that's too permissive provides a false sense of safety. But a mis-scoped allow-list is a specification bug you can audit and fix. A rotted deny-list is a silent failure you discover in production.

This is what ToolClad does for tool interfaces. Instead of letting the LLM generate arbitrary shell commands and filtering them, ToolClad defines typed parameter slots, output schemas, and invocation templates. The LLM fills in typed fields. The executor validates against the contract and constructs the command from a template. A prompt injection that tries to add `; rm -rf /` fails not because a filter caught it, but because the semicolon has no valid slot to occupy.

SchemaPin extends this to tool discovery. Before an agent uses a tool, it cryptographically verifies the tool's schema against a pinned signature. A schema-modification attack (swap a benign parameter description for an instruction to exfiltrate data) fails because the signature won't match. This isn't a rule in the system prompt. It's ECDSA verification in code. SchemaPin's TOFU (trust-on-first-use) pinning model materially protects against post-discovery tampering; first-use compromise and signing key theft require additional mitigations at the operational layer.

## The Convergence

The agentic AI infrastructure being built in 2025-2026 is converging on solutions to a cognitive constraint that neuroscience identified thirty years ago. Different substrate, same failure signature, same solutions.

The agentic AI *security* infrastructure being built right now faces the same convergence. Every team that ships agents into production eventually discovers that prompt-based guardrails degrade under load. The ones who've been at it longest have all arrived at the same conclusion: enforcement that lives inside the context window is enforcement that will eventually fail.

The solutions are structural. Policy evaluation outside the LLM. Typed contracts that constrain expressible actions. Cryptographic verification that doesn't depend on the agent remembering to check. Compile-time guarantees that make invalid state transitions inexpressible.

Context rot is real. Goal neglect is the best model we have for why it happens. And the fix is not better prompts.

It's architecture.

---

*Jascha Wanger is the founder of ThirdKey AI, building cryptographic trust infrastructure for enterprise AI agents. The trust stack includes Symbiont (zero-trust agent runtime), SchemaPin (tool schema verification), AgentPin (agent identity), and ToolClad (declarative tool contracts).*

**Links:**
- Symbiont: [https://github.com/ThirdKeyAI/Symbiont](https://github.com/ThirdKeyAI/Symbiont)
- SchemaPin: [https://github.com/ThirdKeyAI/SchemaPin](https://github.com/ThirdKeyAI/SchemaPin)
- AgentPin: [https://github.com/ThirdKeyAI/AgentPin](https://github.com/ThirdKeyAI/AgentPin)
- ToolClad: [https://github.com/ThirdKeyAI/ToolClad](https://github.com/ThirdKeyAI/ToolClad)
- OATS Specification: [https://thirdkey.ai/oats](https://thirdkey.ai/oats)
