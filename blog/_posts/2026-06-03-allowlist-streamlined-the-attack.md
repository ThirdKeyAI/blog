---
layout: post
title: "The Allowlist Streamlined the Attack"
author: ThirdKey Team
categories: [AI Security, Agent Runtime, Governance]
tags: [owasp, agentic security, allowlist, cve, prompt injection, runtime enforcement, sandbox escape, symbiont, cedar, typestate, audit log, ai agents, governance]
---

*What the 2026 OWASP agentic security report's headline finding actually says — and why it's about reachability, not allowlists.*

**Jascha Wanger &mdash; ThirdKey AI Research**

---

OWASP's Agentic Security Initiative published the second edition of *State of Agentic AI Security and Governance* this month. It is written for CISOs, and most of it reads the way you would expect a year-two report to read. The threats that were architectural worries in 2025 now have production incidents, vendor advisories, and CVEs attached to almost every entry.

One finding deserves more attention than it is getting, because it is being read as the opposite of what it says.

## Two CVEs, one shape

The report documents two coding-agent vulnerabilities from the last few months.

CVE-2026-22708, disclosed against Cursor in January 2026, lets an attacker who can influence the agent's instructions poison the execution environment so that commands the user or the allowlist has already approved run attacker code instead. The dangerous payload does not arrive through a command anyone would have blocked. It arrives through `git branch` or `python3 script.py`, commands that were waved through precisely because they looked routine.

CVE-2025-59532, disclosed against OpenAI's Codex CLI, is the same shape one layer down. The agent's own output was able to redefine the writable boundary of its sandbox, so file writes and command execution escaped the workspace they were supposed to be confined to.

The report's line on the Cursor case is the one to sit with. The allowlist, it says, "does not fail to prevent the attack. It streamlines it." The approved set became the delivery mechanism.

It is easy to read that as a verdict on allowlists. It is not.

## The thing both CVEs share

Neither vulnerability is about the contents of an allowlist. Both are about who can reach the allowlist.

In the Cursor case the agent could influence the execution environment that the approved commands ran inside. In the Codex case the agent's output could move the sandbox boundary. In each one the component being governed had a path to the thing doing the governing. The report names the root cause itself: controls calibrated for human operators become exploitable when the executor can influence its own containment.

That sentence is the whole lesson. These controls were designed on the assumption that the operator and the enforcement boundary are separate parties. A human runs a command, a sandbox confines it, and the human cannot quietly rewrite the sandbox from inside the command. Agents break that assumption. The executor and the thing being executed are now the same probabilistic process, and any control that process can touch is a control an injected instruction can touch.

So the familiar framing, allowlist versus denylist, is arguing about the wrong axis. A denylist fails because you cannot enumerate every bad action in advance. An allowlist is stronger on paper because it inverts that burden. But an allowlist enforced by a process the agent can influence is not a weaker allowlist. It is not an allowlist at all. It is a suggestion that happens to be phrased as a constraint.

The axis that decides whether a control holds is not what it permits. It is whether the governed component can reach the governor.

## This is not only a coding-agent problem

It is tempting to file these CVEs under "coding agents are dangerous" and move on. The rest of the report does not let you.

Two of its three headline findings point at the same place. Safety and security, it argues, converge at the deployment layer, because the same permission surface that lets an agent cause harm on its own is the surface an attacker triggers through injection. And governance has to move from static pre-deployment certification to runtime control, because an agent composes its behavior after it is assessed, not before.

Then comes the admission that should interest anyone building in this space. The report says the field is converging on runtime observability as the substitute for certification. Trajectory-level logging, plan-divergence detection, behavioral envelope monitoring. And it adds, plainly, that no consensus architecture has yet taken hold.

Observability is necessary. You cannot govern what you cannot see, and the regulatory clocks now attached to agentic incidents, DORA's four hours, NIS2's twenty-four, make telemetry non-negotiable. But observation is not enforcement, and a detector whose inputs the agent's output can shape inherits the exact problem the CVEs demonstrated, one level up. The report says as much about the deterministic hooks that LangGraph, the OpenAI Agents SDK, Google ADK, and Claude Code have converged on. In practice, it notes, they work better as an early warning layer than as a hard security boundary.

An early warning layer is a denylist with better tooling. It still depends on recognizing the bad thing, and it still sits somewhere the agent can reach.

## What actually has to be true

If the lesson is that enforcement fails wherever the governed can influence the governor, then the requirement falls out of it. Enforcement has to be structurally independent of the thing it governs.

Three properties define that, and none of them is exotic.

First, policy evaluation happens outside the model's address space and outside its influence. The decision about whether an action is allowed is computed by a component the model cannot read, prompt, or rewrite. It is not a system prompt, not a tool the agent calls, not a hook the agent's output can reshape.

Second, the executor exposes a fixed, named set of actions and nothing else. There is no path by which model output can introduce a new action, redefine an existing one, or move the boundary of what an action may touch. If the action is not in the set at build time, it is not expressible at run time. This is the property both CVEs lacked.

Third, the audit record is append-only and the agent cannot edit it. A trajectory log the agent can influence is evidence the way a diary written by the suspect is evidence. For the deterministic reconstruction that regulators are starting to require, the record has to be tamper-evident independent of the actor it describes.

State those three plainly and most of the current guardrail conversation reclassifies itself. Probabilistic prompt-layer filters and advisory hooks are observability. They belong in the telemetry tier. They are not the boundary, and treating them as one is how you end up with an allowlist that streamlines the attack.

## Where we sit, honestly

Symbiont is our attempt to build a runtime where those three properties hold by construction rather than by configuration.

Policy is evaluated by Cedar, outside the model. The reasoning loop is typestate-enforced in Rust, so an invalid phase transition is a compile error rather than a runtime check the agent might talk its way past. The executor is a profile of one: a static handler map with a name-membership check, so an action the model names but the profile does not contain is refused before policy is even consulted. The audit journal is Ed25519 hash-chained and append-only.

We are not claiming this closes the problem. Two of the report's open questions are open for us too. Human oversight does not scale to an agent taking ten thousand actions an hour, and risk-tiered review only helps if you can compute blast radius in real time, which is hard. Bounding the permission inheritance of dynamically spawned sub-agents is unsolved in the general case, ours included. We think structural independence is the right foundation. We do not think it is the finished building.

But the foundation is the part the 2026 evidence is unusually clear about. The allowlist did not fail because it was an allowlist. It failed because the agent could reach it. Build the enforcement somewhere the agent cannot reach, and most of this report's worst incidents stop being expressible.

---

*Jascha Wanger is the founder of ThirdKey AI, building cryptographic trust infrastructure for enterprise AI agents. Symbiont is the zero-trust agent runtime described above; the broader trust stack includes SchemaPin (tool schema verification), AgentPin (agent identity), and ToolClad (declarative tool contracts).*

**Links:**
- Symbiont: [https://github.com/ThirdKeyAI/Symbiont](https://github.com/ThirdKeyAI/Symbiont)
- SchemaPin: [https://github.com/ThirdKeyAI/SchemaPin](https://github.com/ThirdKeyAI/SchemaPin)
- AgentPin: [https://github.com/ThirdKeyAI/AgentPin](https://github.com/ThirdKeyAI/AgentPin)
- ToolClad: [https://github.com/ThirdKeyAI/ToolClad](https://github.com/ThirdKeyAI/ToolClad)

*Based on OWASP GenAI Security Project, "State of Agentic AI Security and Governance" v2.01 (June 2026), CC BY-SA 4.0. CVE references: CVE-2026-22708 (Cursor), CVE-2025-59532 (OpenAI Codex CLI).*
