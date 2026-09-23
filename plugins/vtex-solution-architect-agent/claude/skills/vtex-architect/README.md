# vtex-architect-skill

A Claude agent skill for VTEX solution architecture. Provides opinionated
decision frameworks, known platform constraints, and architecture patterns
across all major VTEX domains.

## What this skill covers

- Storefront: FastStore vs VTEX IO vs Full Headless
- Backend: VTEX IO Services vs External Middleware
- Payments & Checkout: native vs PPP vs external orchestration (with PCI-DSS guidance)
- Search: VTEX Intelligent Search vs external engines
- Data storage: MasterData v2 vs external DB
- Multi-tenancy: bindings vs Franchise Accounts vs Seller Portal vs separate accounts
- Personalization & Analytics: native vs external CDP (with LGPD/GDPR guidance)
- OMS & Fulfillment: omnichannel and ship-from-store patterns

## Repo structure

```
vtex-architect-skill/
├── SKILL.md                  # The skill — decision frameworks, constraints, patterns
├── README.md                 # This file
├── .claude/
│   └── CLAUDE.md             # Auto-loaded instructions for Claude Code users
└── references/               # VTEX-sourced deep-dive material (see references/README.md)
```

ADRs are **not** stored in this repo. They are managed by the VTEX Architect MCP.

## Required MCPs

| MCP                | Purpose                                                   |
| ------------------ | --------------------------------------------------------- |
| vtex-developer MCP | Live documentation lookup, endpoint search, API reference |
| VTEX Architect MCP    | Knowledge base search and client solution architecture    |

Both must be connected for the agent to function correctly.

## Setup

### Claude.ai Project (recommended)

1. Connect this GitHub repo to your Claude.ai Project via the GitHub connector.
2. Connect the **vtex-developer MCP** and the **VTEX Architect MCP** to the Project.
3. Copy the contents of `.claude/CLAUDE.md` into the Project Instructions field.

### Claude Code (CLI)

1. Clone this repo into your project directory.
2. Claude Code will auto-load `.claude/CLAUDE.md` at session start.
3. Ensure both the vtex-developer MCP and the VTEX Architect MCP are configured in
   your Claude Code settings.

### API / system prompt

1. Paste the full contents of `SKILL.md` into your system prompt.
2. Add both MCPs to your API call's `mcp_servers` parameter.

## Managing ADRs

ADRs are read via the VTEX Architect MCP — not stored in this repo.
When the agent produces an accepted ADR, it will be provided to the user. Use the ADR template in `SKILL.md` as the format.

## Contributing

Update `SKILL.md` when:

- A new VTEX platform constraint is discovered
- A decision framework needs revision based on real project outcomes
- A new architecture pattern emerges

Note the source commit or a brief description of the change in your PR description so you can trace shifts in agent behavior back to specific edits.
