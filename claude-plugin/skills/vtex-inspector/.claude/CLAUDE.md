# Agent Instructions

## Skills

Load `SKILL.md` for any question about the current state of a live VTEX
account: what is configured, what is active, what architecture has been
implemented. The skill defines the reasoning protocol, the domain-to-tools
map, interpretation rules, and output formats — always consult it first.

Do NOT use this skill for:
- Questions about how the VTEX platform works → load vtex-expert
- Architecture design decisions → load vtex-architect

## Required MCP

This skill depends on the **vtex-account MCP** (`vtex-account` in `.mcp.json`),
run via `npx -y @miguel-carrera/vtex-account-mcp-server`.

It authenticates via the VTEX CLI session — the user must run
`vtex login <account-name>` before calling any tool for that account. The token
is read from `~/.vtex/session/tokens.json` keyed by account name.

Every tool requires an `account` parameter (the VTEX account name, e.g. `mystore`).

Before attempting to answer any account question, verify the MCP is connected.
If it is not available, tell the user to ensure it is registered in `.mcp.json`.

Use the tool names defined in the Domain → Tools Map in `SKILL.md` (all tools
are prefixed `vtex_` e.g. `vtex_get_warehouses`, `vtex_get_promotions`).

## Conventions

- Always follow the Reasoning Protocol in SKILL.md before answering.
- Never answer from memory or assumption when account data is available
  via the MCP. Call the tools first.
- Distinguish clearly between "data returned empty" (nothing is configured)
  and "tool call failed" (data could not be retrieved).
- This skill is read-only. Never suggest or attempt to write, update, or
  delete account configuration. Redirect change requests to the VTEX Admin,
  the VTEX API directly, or the appropriate skill.
- When findings reveal incomplete or potentially problematic configuration,
  surface them proactively in a gap report.
- For a full account overview, follow the Architecture Summary template
  in SKILL.md and base every section on live MCP data.
