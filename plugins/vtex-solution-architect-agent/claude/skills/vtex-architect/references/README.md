# References

VTEX-sourced deep-dive material that supplements `SKILL.md`. These files
provide depth on platform constraints, schema rules, endpoint contracts,
and anti-patterns. The frameworks and decision rules live in `SKILL.md` —
references provide the supporting detail.

## When to load

The Reasoning Protocol in `SKILL.md` directs the agent to load the relevant
reference: **step 4 (Triage & grill)** loads `grilling.md`; **step 6 (Load
the relevant reference)** loads the domain file for the question. The full
mapping lives in the References Index in `SKILL.md`. Summary:

| Domain                                                  | File                               |
| ------------------------------------------------------- | ---------------------------------- | --- |
| Cross-cutting architecture, Well-Architected pillars    | `architecture-well-architected.md` |
| Headless storefronts (BFF, IS, checkout proxy, caching) | `headless.md`                      |
| MasterData v2 strategy and schema design                | `masterdata-strategy.md`           |     |
| Payments (PPP, PPF, idempotency, PCI)                   | `payments.md`                      |
| How to grill (interaction behavior, not platform facts) | `grilling.md` — **locally authored** |

## Source and attribution

Most files here are imported from the official **vtex/skills** repository
(`exports/claude/` directory), which is the source of truth for VTEX
platform documentation packaged for AI agents.

- Source repo: https://github.com/vtex/skills
- Import path: `exports/claude/{architecture,masterdata,payment,headless}.md`
- Files were renamed locally for clarity (e.g. `payment.md` → `payments.md`,
  `masterdata.md` → `masterdata-strategy.md`).

**Exception — locally-authored behavioral references.** `grilling.md` is **not**
imported from `vtex/skills`. It governs how the skill *interacts* (when to
interrogate the user, how deep, how to state its context basis), not VTEX
platform facts. It is maintained in this repo directly and must **not** be
overwritten by an upstream reference refresh.

## Updating

These files are a point-in-time snapshot. To refresh:

1. Pull the latest from `vtex/skills`.
2. Re-copy the matching files from `exports/claude/` into this folder.
3. Note the source commit or release tag in `CHANGELOG.md`.

Do not edit the content of reference files directly — local edits will
be lost on the next refresh. If a reference is wrong or out of date,
contribute the fix upstream to `vtex/skills`.

## What does _not_ belong here

- Opinionated decision frameworks → those live in `SKILL.md`.
- ADRs → managed by the VTEX Architect MCP, not stored in the repo.
