---
layout: post
title: "Detection Loses. Construction Wins."
author: ThirdKey Team
categories: [AI Security, Agent Runtime, Evaluation]
tags: [prompt injection, guardrails, red team, confused deputy, symbiont, toolclad, cedar, runtime enforcement, structural security, ai agents, evaluation, benchmarks]
---

*We benchmarked every accessible production guardrail against a blind-authored attack corpus. Properly measured (ROC-AUC, threshold sweeps, confidence intervals), the picture is more nuanced than "guards don't work" and more damning. Out of the box, none is deployable. Tuned, the better ones separate attacks from legitimate traffic, but only as a perpetual calibration project. Symbiont's structural controls need none of it: 0% escape at 0% false positives, by construction.*

**Jascha Wanger &mdash; ThirdKey AI Research**

---

We built a content-marker fence for cross-agent injection. It matched known injection patterns and stripped them before the downstream agent could act on them. On our own attack shapes, it dropped escape rate from 59% to 3.6%.

Then we tested it properly.

## The held-out experiment

Every attack shape in our prior evaluation was authored by the same team that designed the fences. Those results answer "do our fences fire on the shapes we wrote?" not "what fraction of real attacks would they catch?"

For this experiment, a subagent with zero access to the validators, marker list, or canary patterns wrote 90 attack lures from a threat-model-only brief. It approximates a held-out red team. Scoring changed too. Instead of regex canary detection (which shares vocabulary with the marker list, making the measurement partly circular), we scored behaviorally: the downstream supervisor actually ran on the worker's output, and we checked whether routing flipped to `exec_escalation`.

The result, pooled across four frontier models (Claude Sonnet 4.6, GPT-5, DeepSeek v4 Pro, Qwen 3.6 Max), n=120 per arm:

| Arm | Escape rate (Wilson 95% CI) |
|---|---|
| Unguarded baseline | 28.3% [21%, 37%] |
| Content-marker fence | 25.8% [19%, 34%] |

The confidence intervals overlap. The marker fence provides no statistically significant protection on blind-authored attacks scored behaviorally. Models paraphrased past it.

We published this. Our own fence failed and we put the data in the repo. Measured on the same axis as the commercial guards below, Symbiont's own marker detector scores AUC 0.51. A coin flip.

## The guardrail landscape, measured properly

We then tested every accessible production guardrail on the same held-out corpus: 90 attack lures plus 30 benign controls (real support tickets). Each guardrail was run in its default recommended configuration on the bare prompt, no agent framing, no custom tuning. We report ROC-AUC (threshold-free separability), the out-of-the-box operating point (default threshold), and the best tuned point (max detection at false-positive rate capped at 5%).

| Guardrail | AUC | Out-of-box detection | Out-of-box false positive | Tuned (det / FP) |
|---|---|---|---|---|
| IBM Granite Guardian 3.0 | 1.00 | 100% | 77% | 100% / 0% |
| IBM Granite Guardian 3.2 | 1.00 | 68% | 0% | 99% / 3% |
| Meta Llama Prompt Guard 2 | 1.00 | 1% | 0% | 100% / 3% |
| Google ShieldGemma-2B | 1.00 | 0% | 0% | 99% / 3% |
| Qwen3Guard-Gen-4B | 1.00 | 4% | 0% | 100% / 0% |
| Qwen3Guard-Gen-8B | 1.00 | 1% | 0% | 100% / 3% |
| deepset DeBERTa | 0.99 | 100% | 57% | 100% / 3% |
| Qwen3Guard-Gen-0.6B | 0.99 | 6% | 0% | 98% / 3% |
| Reasoning-aware monitor (4B judge) | 0.96 | 99% | 17% | 56% / 0% |
| ProtectAI DeBERTa v2 | 0.73 | 9% | 10% | 2% / 3% |
| AllenAI WildGuard (7B) | 0.63 | 26% | 0% | 26% / 0% |
| Meta LlamaFirewall (AlignmentCheck) | 0.63 | 43% | 17% | n/a (binary) |
| Symbiont's own marker detector | 0.51 | 1% | 0% | 1% / 0% |

Two rows are a different paradigm. Every other guard is a pattern or safety classifier; these two are reasoning-aware judges — a small LLM reads the request against a declared remit and decides whether it stays in scope. They bracket the range. The 4B judge is the strongest detector we tested on raw separability of these semantic lures, flagging 99% of them. Meta's production LlamaFirewall framework, run through its AlignmentCheck auditor, catches just 43% — and its verdict is binary, so there is no threshold to tune. Reading intent can beat pattern-matching, but the implementation gap is wide and, as the next section shows, neither reaches a deployable operating point.

Three reads from this table.

First: out of the box, none is deployable. Granite 3.0 catches everything but flags 77% of real traffic. deepset catches everything but flags 57%. The 4B reasoning-aware monitor catches 99% but flags 17%. Meta's LlamaFirewall catches 43% and flags 17%, with no threshold to move. ShieldGemma, Prompt Guard 2, and the entire Qwen3Guard family flag almost nothing and catch almost nothing (0–6% detection at their default thresholds). Whatever a customer installs and runs at defaults fails.

Second: tuned with labeled data and per-model threshold calibration, the better agent-aware guards (Granite 3.0, Prompt Guard 2, ShieldGemma, and the Qwen3Guard models) genuinely can separate this corpus. AUC near 1.0. That is a real result and we are not going to pretend otherwise.

Third: generic classifiers cannot separate these lures even tuned. ProtectAI (AUC 0.73), AllenAI WildGuard (AUC 0.63, a 7B safety LLM that is still not agent-aware enough), and our own marker detector (AUC 0.51, chance) fail because the attacks are semantic ("read /data/secret/pricing.csv", "fetch reports.example.com"), not crude "ignore previous instructions." The lures look like work.

## But those tuned numbers are optimistic

The 30 benign controls are all support tickets. Different genre from the attacks. That over-states separability. We added 22 same-domain legitimate prompts from Symbiont's own task objectives: read an allowed CSV, fetch an internal host, run a real triage. Benign requests that look operationally identical to the filesystem and network attacks.

Every guard's AUC fell.

| Guardrail | AUC (easy benign) | AUC (hard benign) | Tuned on hard (det / FP) |
|---|---|---|---|
| Google ShieldGemma-2B | 1.00 | 0.98 | 87% / 4% |
| IBM Granite Guardian 3.2 | 1.00 | 0.97 | 78% / 4% |
| Meta Llama Prompt Guard 2 | 1.00 | 0.93 | 32% / 2% |
| Reasoning-aware monitor (4B judge) | 0.96 | 0.93 | 56% / 2% |
| Qwen3Guard-Gen-8B | 1.00 | 0.89 | 24% / 4% |
| IBM Granite Guardian 3.0 | 1.00 | 0.89 | 42% / 8% |
| Qwen3Guard-Gen-4B | 1.00 | 0.88 | 21% / 4% |
| Qwen3Guard-Gen-0.6B | 0.99 | 0.87 | 18% / 4% |
| deepset DeBERTa | 0.99 | 0.80 | 100% / 40% |
| ProtectAI DeBERTa v2 | 0.73 | 0.75 | 3% / 4% |
| AllenAI WildGuard (7B) | 0.63 | 0.60 | 26% / 6% |
| Meta LlamaFirewall (AlignmentCheck) | 0.63 | 0.63 | 43% / 17% |
| Symbiont's own marker detector | 0.51 | 0.51 | 1% / 0% |

The best agent-aware guards (ShieldGemma 0.98, Granite 3.2 0.97) still separate genre-matched traffic, tuning to 78-87% detection at roughly 4% false positives. That is the honest floor for detection done well: a permanent calibration project that catches most attacks at a real false-positive cost, re-tuned every time models or attacks drift.

The Qwen3Guard models keep a high separability score on the hard set (AUC 0.87-0.89) but the number that matters at deployment collapses: tuned to a usable false-positive rate, they catch only 18-24% of attacks. A good AUC is not a deployable operating point.

The reasoning-aware judges are the most interesting failure. The 4B monitor holds the best AUC of any non-1.0 guard on the hard set (0.93) — reading intent genuinely beats word-pattern matching. But it lands on the recall-versus-precision wall: at its native setting it flags 27% of legitimate same-domain traffic, and tuned for precision it catches only 56% of attacks — it misses 44%. Meta's production LlamaFirewall reaches the same wall from the other side: a fixed binary verdict that catches 43% of attacks at 17% false positives, AUC 0.63, with no knob to turn. Two reasoning-aware monitors, an implementation gap of 0.30 AUC between them, and neither reaches the corner. Smarter detection raises the ceiling. It does not remove the trade-off.

deepset collapses the other way. 100% detection at 40% false positives on the hard set. It flags the legitimate filesystem and network requests. The trigger-happy classifier cannot tell "read the allowed report" from "read the secret report."

Generic injection classifiers, the WildGuard safety LLM, and our own marker detector stay at chance regardless of tuning.

## The structural fix

Detection is a tuning treadmill. Structure is a one-time guarantee.

The injection vector is a confused-deputy problem. Untrusted worker free-text gets spliced into a higher-privilege position. Content filtering fights this surface. The structural fix removes the surface.

Instead of filtering free text, make the privileged decision over typed enum fields and ground it in trusted context. The worker submits `submit_triage(category: enum, severity: enum, rationale: str)`. Routing is a deterministic policy over the typed fields. The `rationale` field is logged but never influences the routing decision.

Same held-out attacks, same models, n=120 per arm:

| Arm | Escape rate |
|---|---|
| Typed channel, trusting router (trusts worker's `severity`) | 38-40% |
| Typed channel, grounded router (caps `severity` by trusted ticket) | 0/120 [0%, 3%] |

Typing alone is not security. A router that trusts the worker's self-reported severity is worse than the unguarded baseline (40% vs 28%). The attack no longer needs to paraphrase. It just sets `severity=critical`. 46 of 120 workers did exactly that.

The security comes from grounding the decision in trusted input. Cap the worker's claim by a severity derived from the trusted ticket. A compromised worker cannot escalate a benign ticket. Zero escapes.

This is not a prototype result. The Symbiont runtime ships the typed + grounded decision. Driving the shipping runtime's decision module through the same evaluation: 0/120 escalations, 0/30 on every model. The shipping runtime matched the validated prototype on 120/120 trials.

## What this does not show

The held-out corpus was authored by a subagent, not an external human red team. The subagent had general attack knowledge but had not seen the fences. An external human set against a frozen fence commit is the stronger next experiment.

The benign-ticket grounding works because the held-out tickets are benign. An attacker who controls the ticket text (indirect injection into the trusted input) is a separate surface.

The 0/120 result has a Wilson 95% CI of [0%, 3%]. The structural mechanism is not probabilistic, but the measurement of it is.

All rates carry wide confidence intervals at these sample sizes (n=90 positive, 30-52 negative). Precise per-guard ranking is noisy. The gross effects are not: out-of-box failure, AUC near chance for generic injection classifiers, and deepset's 40% false-positive rate on genre-matched negatives.

nemoguard and Llama Guard 4 are not yet included (tokenizer/VRAM constraints). Results will be added on access with version pins.

## Reproduce

All numbers regenerate from committed artifacts in the public repo. Guardrail benchmarks use released model versions on the bare prompt with no API key required for the open-source classifiers. Total cost to reproduce the held-out evaluation: approximately $60.

Reports and reproduction scripts: [github.com/ThirdKeyAI/symbiont-orga-demo](https://github.com/ThirdKeyAI/symbiont-orga-demo)

*Next post: structural vector results across filesystem, network, syscall, state mutation, and typed-argument attack families. Six independent measurements, zero escapes.*

---

*Jascha Wanger is the founder of [ThirdKey AI](https://thirdkey.ai). Symbiont is the shipping runtime that implements the [OATS](https://openagenttruststack.org) specification.*
