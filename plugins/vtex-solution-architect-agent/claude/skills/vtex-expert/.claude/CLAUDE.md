# Agent Instructions

## Skills

Load `SKILL.md` for any question about how the VTEX platform works:
feature mechanics, admin configuration, logistics, OMS, catalog, promotions,
pricing, or checkout behavior. The skill defines the reasoning protocol,
references index, and output format rules — always consult it before answering.

Do NOT load this skill for solution architecture decisions or trade-off
questions. Those belong to `vtex-architect`.

## Required MCPs

The **vtex-developer MCP** must be connected before answering any question
that involves VTEX APIs, endpoints, or live documentation. Verify it is
available at the start of each session. If it is not connected, tell the
user to add it before proceeding with API-level questions.

MCP tools available:

- `search_documentation` — semantic search over VTEX docs
- `fetch_document` — retrieve full content of a VTEX documentation page
- `search_endpoints` — find VTEX API endpoints by query
- `get_endpoint_details` — get full details of a specific endpoint

The **VTEX Architect MCP** is the source of truth for accepted decisions that
may constrain platform configuration recommendations. Query it when the
question touches an area where the team may have made prior choices.

MCP tools available:

- `retrieve_context` — semantic search over the VTEX Atlas Knowledge Base

## Conventions

- Always follow the Reasoning Protocol in SKILL.md before answering.
- Load the matching reference file from `references/` for the relevant domain.
- Use `vtex-developer` MCP for live API details and doc content.
- Use the VTEX Architect MCP (`retrieve_context`) to check for existing accepted
  decisions before recommending a configuration approach.
- Cite the source (reference file, doc URL, or MCP result) when stating
  platform behaviors.
- Flag known quirks, rate limits, and edge cases proactively.
- Never recommend architecture trade-offs here — redirect to `vtex-architect`.
