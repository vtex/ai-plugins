# Agent Instructions

## Skills

Load `SKILL.md` for any question involving VTEX architecture, platform
decisions, storefront, payments, OMS, search, data, personalization,
or multi-tenancy. The skill contains decision frameworks, known constraints,
and architecture patterns — always consult it before answering.

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

The **VTEX Architect MCP** is the source of truth for all accepted architecture
decisions and client solution architecture files. Query it before proposing
any direction that could conflict with an existing decision. When the user
accepts a new ADR, build a document and deliver it to the user.

MCP tools available:

- `retrieve_context` — semantic search over the VTEX Atlas Knowledge Base; returns raw passages (ADRs, VTEX cases, indexed docs)
- `get_architecture` — fetch a client's solution architecture document by VTEX account name

See the Client Architecture Files section in SKILL.md for the full retrieval protocol.

## Conventions

- Always follow the Reasoning Protocol in SKILL.md before answering.
- Before recommending an approach, query the VTEX Architect MCP (`retrieve_context`)
  for existing accepted decisions in that domain.
- Produce an ADR for any decision that is hard to reverse (account architecture,
  storefront choice, search engine selection, payment provider model). Once
  accepted, deliver it to the user.
- Cite the MCP source when referencing live documentation, API specs, or ADRs.
- Flag constraints from the constraints table proactively — do not wait
  for the user to encounter them.
- Never recommend a third-party tool without first confirming native VTEX
  capability is genuinely insufficient.
