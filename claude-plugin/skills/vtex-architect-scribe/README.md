# vtex-architect-scribe-skill

The authoring-and-verification companion to `vtex-architect`. The architect
decides; the scribe **writes the documentation down and proves it correct**
before it ships.

## What this skill does

Two passes, in separate contexts:

1. **Draft** — authors the doc (integration spec, functional doc, design doc)
   grounded in client architecture, prior decisions, and API contracts, using
   progressive disclosure (Why → How → Implementation).
2. **Adversarial fact-check** — an **independent subagent** tries to *refute*
   the draft against ground truth (VTEX API contracts + the live account),
   returns a verdict list (confirmed / refuted / unverifiable), and the scribe
   reconciles it in.

Two entry points: "write X" runs both passes; "fact-check this doc" runs only
the second.

## Why the fact-check is independent

A checker that shares the writer's context re-confirms the writer's own
sources. The value comes from **role asymmetry**, not different tools: opposite
objective (refute), ground-truth routing (contracts + live account over the
synthesized KB), execution against the real account, and honest flagging of
what can't be verified. See `references/fact-check.md`.

## Repo structure

```
vtex-architect-scribe/
├── SKILL.md                  # The skill — two-pass author + verify workflow
├── README.md                 # This file
├── .claude/
│   └── CLAUDE.md             # Auto-loaded instructions for Claude Code users
└── references/
    ├── README.md
    └── fact-check.md         # Adversarial fact-check playbook (subagent mandate)
```

## Required MCPs

| MCP             | Purpose                                                        |
| --------------- | ------------------------------------------------------------- |
| vtex-architect-mcp     | Client architecture + prior decisions (drafting); synthesized layer (checked last) |
| vtex-developer  | Ground-truth API contracts — the fact-checker's primary source |
| vtex-account    | Live, read-only account inspection — lets the fact-checker execute |

## Relationship to the other skills

- `vtex-architect` decides; it **hands off** to this skill when the deliverable
  is a document that must be verified. Architects invoke `vtex-architect`; the
  scribe engages transparently.
- `vtex-inspector` reports live account state; the scribe's fact-checker *uses*
  account data but does not produce an inspection report.
- `vtex-expert` explains how the platform works.

## Known limitation

With the current MCPs, claims whose only source is the synthesized knowledge
base cannot be independently verified — they are flagged, not confirmed.
Strengthening this may require additional sources (web search, a second KB).
Tracked as an open item.
