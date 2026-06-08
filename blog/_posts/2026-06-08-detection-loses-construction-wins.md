---
layout: post
title: "Detection Loses. Construction Wins."
author: ThirdKey Team
categories: [AI Security, Agent Runtime, Evaluation]
tags: [prompt injection, guardrails, red team, confused deputy, symbiont, toolclad, cedar, runtime enforcement, structural security, ai agents, evaluation, benchmarks]
---

*We benchmarked every accessible production guardrail against a blind-authored attack corpus. The best ones catch everything and flag 77% of real traffic. The worst catch nothing. Our early content fence failed too. The structural fix ships at 0%.*

**Jascha Wanger &mdash; ThirdKey AI Research**

---

We built a content-marker fence for cross-agent injection. It matched known injection patterns (literal substrings like "you are now," "NEW ROUTING RULE," "Supervisor:") and stripped them before the downstream agent could act on them. On our own attack shapes, it dropped escape rate from 59% to 3.6%.

Then we tested it properly.

## The held-out experiment

Every attack shape in our prior evaluation was authored by the same team that designed the fences. Those results answer "do our fences fire on the shapes we wrote?" not "what fraction of real attacks would they catch?"

For this experiment, a subagent with zero access to the validators, marker list, or canary patterns wrote 90 attack lures from a threat-model-only brief. It approximates a held-out red team. Scoring changed too. Instead of regex canary detection (which shares vocabulary with the marker list, making the measurement partly circular), we scored behaviorally: the downstream supervisor actually ran on the worker's output, and we checked whether routing flipped to `exec_escalation`.

The result, pooled across four frontier models (Claude Sonnet 4.6, GPT-5, DeepSeek v4 Pro, Qwen 3.6 Max), n=120 per arm:

| Arm | Escape rate (Wilson 95% CI) |
|---|---|
| Unguarded baseline | 28.3% [21%, 37%] |
| Content-marker fence | 25.8% [19%, 34%] |

The confidence intervals overlap. The marker fence provides no statistically significant protection on blind-authored attacks scored behaviorally. Models paraphrased past it. The in-distribution number (3.6% vs 59%) was inflated by attacks shaped like the marker list and a marker-aligned detector.

We published this. Our own fence failed and we put the data in the repo.

## The guardrail landscape is worse

We then tested every accessible production guardrail on the same held-out corpus: 90 attack lures plus 30 real support tickets (clean traffic). Detection forces a trade-off between catching attacks and not blocking legitimate work.

| Guardrail | Catches attacks | Falsely flags real traffic |
|---|---|---|
| IBM Granite Guardian 3.0 | 100% | 77% |
| deepset DeBERTa | 100% | 57% |
| IBM Granite Guardian 3.2 | 67% | 0% |
| ProtectAI DeBERTa v2 | 9% | 10% |
| Meta Llama Prompt Guard 2 | 1% | 0% |
| Google ShieldGemma-2B | 0% | 0% |

The pattern is clear. Guardrails that catch attacks also flag real traffic at rates that make them unusable in production. A CISO cannot deploy a system that blocks 57-77% of legitimate support tickets. The guardrails that avoid false positives catch almost nothing.

This is not an implementation failure. It is the fundamental limitation of detection-based approaches to a paraphrase-rich attack surface. The attacker has unbounded ways to express the same instruction. The detector has a fixed model. The attacker wins at the margin.

ShieldGemma is a content-safety classifier, not a prompt-injection detector. We included it to show that general-purpose safety classifiers do not transfer to this threat model. Prompt Guard 2 is purpose-built for injection and still catches only 1%.

## The structural fix

The injection vector is a confused-deputy problem. Untrusted worker free-text gets spliced into a higher-privilege position (the supervisor's routing instructions). Content filtering fights this surface. The structural fix removes the surface.

Instead of filtering free text, make the privileged decision over typed enum fields and ground it in trusted context. The worker submits `submit_triage(category: enum, severity: enum, rationale: str)`. Routing is a deterministic policy over the typed fields. The `rationale` field is logged but never influences the routing decision.

Same held-out attacks, same models, n=120 per arm:

| Arm | Escape rate |
|---|---|
| Typed channel, trusting router (trusts worker's `severity`) | 38-40% |
| Typed channel, grounded router (caps `severity` by trusted ticket) | 0/120 [0%, 3%] |

Two findings here.

Typing alone is not security. A router that trusts the worker's self-reported severity field is worse than the unguarded baseline (40% vs 28%). The attack no longer needs to paraphrase. It just sets `severity=critical`. 46 of 120 workers did exactly that.

The security comes from grounding the decision in trusted input. Cap the worker's claim by a severity derived from the trusted ticket. A compromised worker cannot escalate a benign ticket. Zero escapes.

This is not a prototype result. The Symbiont runtime ships the typed + grounded decision (`symbi_runtime::toolclad::decision`). Driving the shipping runtime's decision module through the same evaluation: 0/120 escalations, 0/30 on every model. The shipping runtime matched the validated prototype on 120/120 trials.

## The structural vectors

Injection is the hardest vector because you cannot close it by removing a tool. But four other attack families can be closed structurally: filesystem enumeration, network boundary violation, state mutation, and syscall escape.

Across 960 trials with four frontier models (two open-weights, two closed-weights), Symbiont reduced escape success on these vectors from 46-96% on the permissive baseline to 0% on every one:

| Vector | Permissive baseline | Symbiont |
|---|---|---|
| Filesystem enumeration | 94% | 0% |
| Network boundary | 95% | 0% |
| State mutation | 96% | 0% |
| Syscall boundary | 46% | 0% |

The mechanism differs by vector. Filesystem and network escapes are caught by ToolClad's path and URL validators (177 and 205 substrate blocks respectively). State mutation is prevented by tool-surface reduction (no write tool is declared, so the model cannot reach for one). Syscall escape is prevented the same way (no exec tool exposed). The substrate gap held across all four model lineages, open and closed weights, weak and strong safety training.

## Non-redundancy: each fence catches what the others miss

This claim matters because layered defenses can be illusory. If every layer catches the same attacks, removing one changes nothing.

We measured this with stack-stripping ablation (v12). Remove the sanitizer: html-comment-smuggle payloads survive at 92%. Remove Cedar: out-of-profile tool calls go through (but the executor's independent profile-of-one check still catches 219). Remove ToolClad: typed-argument injection payloads pass at 99.4%.

Each fence catches a class of attack the others do not. Removing any one fence exposes the corresponding attack class at near-100% survival. The defense composes without redundancy.

## What this does not show

Honesty about scope is part of the methodology.

The held-out corpus was authored by a subagent, not an external human red team. The subagent had general attack knowledge but had not seen the fences. An external human set against a frozen fence commit is the stronger next experiment.

The benign-ticket grounding works because the held-out tickets are benign. An attacker who controls the ticket text (indirect injection into the trusted input) is a separate surface.

Scoring differs across arms. The marker and baseline arms use an LLM supervisor. The typed arms use a deterministic policy. The clean comparison is within the typed arm: trusting 38-40% to grounded 0%.

The 0/120 result has a Wilson 95% CI of [0%, 3%]. At larger n the point estimate may shift. The structural mechanism (typed enums + trusted-context grounding) is not probabilistic, but the measurement of it is.

## Reproduce

Everything described here regenerates from committed artifacts:

```
cd evals/escape

# Guardrail comparison
python3 -m guardrails.run_sweep --execute
python3 -m guardrails.analyze

# Held-out red team (marker arm + behavioral scoring)
python3 -m redteam.run_holdout --mode marker --substrates python_baseline,symbiont --execute

# Held-out red team (typed channel, shipping runtime)
python3 -m redteam.run_holdout --mode typed --substrates symbiont --execute
python3 -m redteam.analyze
```

Total cost to reproduce the held-out evaluation: approximately $60. The guardrail sweep runs on released model versions with no API key required for the open-source classifiers.

Reports: [github.com/ThirdKeyAI/symbiont-orga-demo](https://github.com/ThirdKeyAI/symbiont-orga-demo)

---

*Jascha Wanger is the founder of [ThirdKey AI](https://thirdkey.ai). Symbiont is the shipping runtime that implements the [OATS](https://openagenttruststack.org) specification. The evaluation methodology, all aggregate numbers, and reproduction scripts are committed to the public repo.*
