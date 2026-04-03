---
layout: post
title: "Stop Letting Your Agent Write Shell Commands"
author: ThirdKey Team
categories: [AI Security, Agent Runtime, Tool Safety]
tags: [toolclad, ai agents, security, tool contracts, mcp, cedar, symbiont, trust stack, agentic systems]
---

*Introducing ToolClad: Declarative Tool Interface Contracts for Agentic Runtimes*

**Jascha Wanger | ThirdKey AI Research**

---

Every team building agentic systems hits the same wall eventually. You have a CLI tool. You want your agent to use it. The question is always the same: "How do I safely let an agent run `nmap`?"

The current answer, across the entire ecosystem, is: write custom glue code. For each tool, you build a wrapper script with argument sanitization, timeout enforcement, output parsing, evidence capture, policy mappings, capability registration, and authorization rules. That is seven steps per tool. It does not scale.

So we built ToolClad.

## The Problem Is Structural

The popular approach to agent tool safety is the sandbox. Let the LLM generate a shell command, then intercept it on the way out. Pattern-match against known-dangerous operations. Block the bad ones, allow the rest.

This is a deny-list. And deny-lists have gaps by definition.

```
LLM generates shell command -> sandbox intercepts -> allow/deny
```

The agent has already formulated an arbitrary command string. You are hoping your filter catches everything dangerous. You are playing whack-a-mole with an adversary that has the entire shell language at its disposal.

For interactive tools the gap is even wider. An agent with an open `psql` connection can `DROP TABLE` as easily as it can `SELECT *`, because both are just text sent to a PTY. A sandbox watching syscalls sees identical operations. The semantic difference is invisible to the enforcement layer.

Browser agents face the same problem. Today's browser-capable agents (Claude in Chrome, OpenAI Operator, Playwright-based systems) rely on the LLM's instruction-following to stay on allowed domains, avoid submitting sensitive forms, and not execute arbitrary JavaScript. That is prompt-based security. It is exactly as robust as the model's compliance on any given inference.

## ToolClad Inverts the Model

ToolClad takes the opposite approach. Instead of generating commands and filtering them, the agent fills typed parameters in a declarative manifest. The runtime validates those parameters, constructs the command from a template, and executes it. The agent never sees or generates a shell command.

```
LLM fills typed parameters -> policy gate -> executor validates ->
  constructs command from template -> executes -> structured JSON
```

This is an allow-list. The dangerous action cannot be expressed because the interface does not permit it.

A ToolClad manifest (`.clad.toml`) is the complete behavioral contract for a tool. It answers four questions:

1. **What can this tool accept?** Typed parameters with validation constraints: enums, ranges, regex patterns, scope checks, injection sanitization.
2. **How do you invoke it?** A command template, HTTP request, MCP server call, PTY session, or browser action. The LLM never generates raw invocation details.
3. **What does it produce?** Output format, parsing rules, and a mandatory output schema that normalizes raw output into structured JSON. The LLM knows the shape of results before proposing a call.
4. **What is the interaction model?** Oneshot execution, interactive PTY session, or governed headless browser. All three share the same governance layer.

Here is what a manifest looks like in practice:

```toml
# tools/nmap_scan.clad.toml
[tool]
name = "nmap_scan"
version = "1.0.0"
binary = "nmap"
description = "Network port scanning and service detection"
timeout_seconds = 600
risk_tier = "low"

[tool.cedar]
resource = "PenTest::ScanTarget"
action = "execute_tool"

[args.target]
position = 1
required = true
type = "scope_target"
description = "Target CIDR, IP, or hostname"

[args.scan_type]
position = 2
required = true
type = "enum"
allowed = ["ping", "service", "version", "syn", "os_detect"]
description = "Type of scan to perform"

[command]
template = "nmap {scan_type_flags} {target}"

[output]
format = "xml"
parser = "builtin:xml"
envelope = true

[output.schema]
type = "object"

[output.schema.properties.hosts]
type = "array"
description = "Discovered hosts with open ports and services"
```

The agent cannot inject shell metacharacters into `target` because the `scope_target` type rejects them. It cannot request a scan type outside the declared enum. It cannot alter the command template. The parameter space is bounded and enumerable.

## The Type System Does the Heavy Lifting

ToolClad ships 14 built-in types (10 core, 4 extended) that cover the patterns repeated across every tool wrapper we have ever written. Every type includes injection sanitization by default. The design principle: "valid according to the type" means "safe to interpolate into a command."

The core types are `string`, `integer`, `port`, `boolean`, `enum`, `scope_target`, `url`, `path`, `ip_address`, and `cidr`. Each one blocks shell metacharacters (`;|&$\`(){}[]<>!\n\r`) at the validation layer, before the command is ever constructed.

The extended types handle domain-specific patterns: `credential_file` (path that must exist and be read-only), `duration` (integer with time suffix), `regex_match` (matches a declared pattern), and `msf_options` (semicolon-delimited key-value pairs for Metasploit).

Projects can also define custom reusable types:

```toml
# toolclad.toml (project root)
[types.service_protocol]
base = "enum"
allowed = ["ssh", "ftp", "http", "https", "smb", "rdp", "mysql", "postgres"]

[types.severity_level]
base = "enum"
allowed = ["info", "low", "medium", "high", "critical"]
```

Then reference them across manifests: `type = "service_protocol"`. Define once, validate everywhere.

## Three Execution Modes, One Governance Layer

The real power of ToolClad is that it extends the same allow-list model to interactive and browser-based tools, not just one-shot CLI commands.

**Oneshot mode** is the default. Execute a command, return the result. Three backends: shell command, HTTP request, or MCP server proxy. This covers the 80% case of wrapping existing CLI tools.

**Session mode** maintains a running PTY process (think `psql`, `redis-cli`, `msfconsole`) where each interaction is independently validated and policy-gated. The manifest declares which commands are allowed, validates each one against a regex pattern, applies Cedar policy per interaction, and tracks session state. The interactive tool becomes a typed, state-aware, policy-gated API surface.

**Browser mode** maintains a governed headless browser session via CDP or Playwright. Navigation URLs are scope-checked against an allow-list of domains. Form submission requires explicit approval. JavaScript execution is a separately gated high-risk command. The governance layer is identical to session mode; only the transport differs.

All three modes produce the same structured evidence envelope: scan ID, timestamps, SHA-256 hash of output, exit code, stderr. Every tool invocation is auditable by default.

## What This Enables

When tool contracts are declarative, interesting properties follow.

**Static analysis.** You can determine what any tool can possibly do before it ever runs, by inspecting the manifest. Cedar policies can reference manifest-declared properties. A compliance team can audit the entire tool surface without reading wrapper code.

**Formal verification.** The parameter space is finite and enumerable for enum types, bounded for numeric types, and regex-constrained for string types. You can prove properties about valid invocations.

**Automatic policy generation.** A tool with a `target` parameter of type `scope_target` inherently requires scope authorization. Cedar policies can be derived from manifests. The manifest is the policy's source of truth.

**MCP schema generation.** Every manifest auto-generates `inputSchema` and `outputSchema` for MCP tool registration. Write one TOML file, get type-safe LLM tool use for free.

**Cryptographic integrity via SchemaPin.** The manifest's SHA-256 hash can be signed and published using SchemaPin's existing TOFU pinning infrastructure. If someone tampers with a command template, a validation rule, or a scope constraint, the hash changes and verification fails. This is strictly stronger than signing only the MCP JSON Schema, because the JSON Schema does not capture execution behavior.

## The 80/20 Split

ToolClad's command template system is deliberately not Turing-complete. It handles conditionals for the common case ("include this flag when this parameter is set") but it does not try to express every possible tool invocation.

Our experience with the [symbi-redteam](https://github.com/ThirdKeyAI/symbi-redteam) toolkit confirms the split: roughly 14 of 19 tools are pure template tools, 3 use simple conditionals, and 2 need escape hatches (a `command.executor` wrapper script). When you find yourself building elaborate conditional chains in TOML, that is the signal to use the escape hatch. ToolClad still provides parameter validation, scope enforcement, timeout, evidence capture, and the output envelope. The wrapper only handles command construction.

If more than 20-30% of your tools need escape hatches, the tools themselves are complex enough to justify custom wrappers. ToolClad's value shifts from "eliminate wrappers" to "standardize the contract around them."

## Available Now, Four Languages

ToolClad v0.5.1 ships reference implementations in Rust, Python, JavaScript, and Go. Each provides manifest parsing, 14 type validators, command construction, execution with process group isolation, output parsing, MCP schema generation, and evidence envelopes.

```bash
cargo install toolclad        # Rust / crates.io
pip install toolclad           # Python / PyPI
npm install toolclad           # JavaScript / npm
```

The CLI gives you four commands: `validate` (parse and check a manifest), `run` (execute with validated arguments), `test` (dry run), and `schema` (output MCP JSON Schema).

The specification is MIT licensed. The Symbiont runtime integration is Apache 2.0. Use it however you want.

## What Comes Next

v1 is local-only by design. Manifests live in your `tools/` directory and get checked into version control. Remote manifest fetching introduces supply chain risk that we are not willing to accept without mandatory SchemaPin signature verification, which is planned for v2.

In the Symbiont runtime, ToolClad manifests are auto-discovered from the `tools/` directory at startup. The runtime registers them as MCP tools, evaluates Cedar policy using manifest-declared resource/action pairs, and wraps every invocation in the ORGA Gate phase. No Rust code required to add a new tool.

The broader goal is a standard contract format for agent tool use across runtimes. If your agent platform can read a `.clad.toml` file, it can safely dispatch any tool described by one. The manifest is the API. The manifest is the policy. The manifest is the documentation.

Stop writing wrapper scripts. Stop letting your agent write shell commands. Declare the contract instead.

---

*ToolClad is open source at [github.com/ThirdKeyAI/ToolClad](https://github.com/ThirdKeyAI/ToolClad). The full design specification is in [TOOLCLAD_DESIGN_SPEC.md](https://github.com/ThirdKeyAI/ToolClad/blob/main/TOOLCLAD_DESIGN_SPEC.md). Questions, feedback, and contributions welcome.*

*ToolClad is part of the [ThirdKey Trust Stack](https://thirdkey.ai): SchemaPin (tool integrity), AgentPin (agent identity), ToolClad (behavioral contracts), and Symbiont (zero-trust runtime).*
