# Agent Instructions

## Skills

Load `SKILL.md` when the user wants to **write, draft, or document** a VTEX
integration/architecture, or to **fact-check / verify** an existing VTEX doc.
The scribe authors documentation and then proves it correct with an
independent adversarial fact-check.

Do **not** use this skill to make architecture decisions (that's
`vtex-architect`) or to report live account state (that's `vtex-inspector`).

## Required MCPs

The scribe needs the plugin's MCPs across its two passes:

- **vtex-architect-mcp** — `retrieve_context`, `get_architecture`. Grounds the draft
  in client architecture and prior decisions. In the fact-check pass this is
  the *synthesized* layer, used last and never to self-confirm a claim.
- **vtex-developer** — `search_documentation`, `fetch_document`,
  `search_endpoints`, `get_endpoint_details`. Ground-truth API contracts —
  the fact-checker's primary source.
- **vtex-account** — live, read-only account inspection. Lets the fact-checker
  *execute*: check account-dependent claims against the real configuration.

If a target account is not reachable via vtex-account, do not fail — verify
what you can and flag account-dependent claims as `unverified`.

## Conventions

- Two passes, **separate contexts**: draft, then an **independent subagent**
  fact-checks. Never fact-check in the writer's own context.
- Ask for source context before drafting; never hardcode a `docs/` layout.
- Output to chat by default; write to disk only when the user names a path.
- The fact-checker returns a **verdict list** (confirmed/refuted/unverifiable
  with evidence); the scribe reconciles it into the doc.
- Read-only against live accounts — never mutate to confirm a claim.
