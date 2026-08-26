# MCP-Sentinel

An automated security-testing agent that pentests **other AI agents and MCP (Model Context Protocol) servers** — instead of web apps. Most AI security tooling checks whether a *model* behaves safely; MCP-Sentinel checks whether the *infrastructure* around it — the tools it's wired to, the permissions it's granted, the manifests it trusts — actually holds up under adversarial pressure.

Built entirely in [n8n](https://n8n.io), using a Claude/GPT-powered reasoning agent for triage and reporting.

> Part of my security research portfolio: [shashankbagule.github.io](https://shashankbagule.github.io) — see the [MCP-Sentinel write-up](https://shashankbagule.github.io/#projects) for the full story of how this was built and validated.

## What it does

MCP-Sentinel enumerates a target's full tool manifest via the MCP `initialize` handshake and `tools/list` call, then runs a set of automated checks against it:

| Check | What it looks for |
|---|---|
| **Scope Audit** | Over-permissioned tools — actively calls a tool and verifies whether it performed an unauthorized action (e.g. a delete), not just flags it from the description |
| **Auth Check** | Whether a session/auth header is actually required, by retrying with a fake or missing session ID |
| **Secrets Scan** | Common secret patterns (API keys, bearer tokens, private key blocks) leaking in tool manifests or responses |
| **Tool Poisoning Check** | Tools whose description makes false safety claims (e.g. "read-only") — with negation-aware text analysis so "does not modify" isn't flagged the same as "will modify" |
| **Schema Drift Check** | Tool definitions that silently change between two `tools/list` calls a few seconds apart |
| **Injection Test** | A **live AI agent**, wired to the target's real tools via an MCP Client Tool node, is fed content containing a hidden instruction — to see if it actually calls a tool as a result, not just whether the payload "looks dangerous" |
| **Tool Chaining Check** | Static analysis for confused-deputy risk — a tool that exposes sensitive data feeding into another tool that accepts it as input |
| **Fuzz Testing** | Sends an unrecognized argument to a tool and checks for crashes, hangs, or leaked stack traces |
| **Behavioral Fingerprinting** | Calls the same tool 3x and checks the response structure stays consistent |

A **Scope Guardrail** node hard-blocks any target not on an explicit allow-list, so the workflow can never accidentally scan something you're not authorized to test.

## Reporting

Findings from every check are merged, deduplicated, and handed to a **Reasoning & Triage** agent that assigns each one a severity (Info → Critical) and a real-vulnerability-vs-expected-behavior verdict. A second **Report Generator** agent turns that into a clean Markdown disclosure-style report — executive summary, findings table, closing notes — as a downloadable `.md` file.

Scans are also saved to a data table, so each new run includes a **regression analysis**: what's new, what's resolved, and what's persistent since the last scan of that target.

## Validation

This isn't just theoretical — I ran it against:
- A self-built "MCP Test Target" to develop and debug each check
- A real external MCP server with a deliberately planted flaw (a tool that claimed to be read-only but silently deleted data) — MCP-Sentinel correctly produced a clean baseline report before the flaw was introduced, and a correctly-flagged high-severity report after

## Setup

This is an n8n workflow export (`mcp-sentinel.json`), not a standalone script. To run it:

1. Import `mcp-sentinel.json` into your own n8n instance (Workflows → Import from File)
2. Edit the **Scope Guardrail** node's `allowedTargets` array to list only the MCP server(s) you're authorized to test
3. Attach your own OpenAI credential to the `OpenAI Chat Model`, `OpenAI Chat Model1`, and `Injection Test Model` nodes (any LangChain-compatible chat model works)
4. If you want scan history/regression analysis, create an n8n Data Table with columns `target_url`, `scanned_at`, `finding_summary`, and point the `Get Previous Scan` / `Save Scan To History` nodes at it
5. Run the workflow manually, or wire it to a trigger of your choice

All credential IDs, instance URLs, and data table IDs in this export are placeholders — you'll need to fill in your own before it runs.

## Why I built this

I wanted to understand AI agent security from the builder's side, not just the attacker's. Writing the checks that catch tool poisoning, scope creep, and prompt injection meant thinking hard about how those attacks actually work — which turned out to be the fastest way to get good at both.

## Author

**Shashank Bagule** — Security Researcher
[Portfolio](https://shashankbagule.github.io) · [LinkedIn](https://linkedin.com/in/shashankbagule) · shashankbagule@gmail.com
