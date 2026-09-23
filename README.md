# VTEX AI Plugins

Marketplace for **VTEX AI plugins** — Claude and ChatGPT (Codex) packages that bring VTEX's architecture knowledge base, live platform documentation, and read-only account inspection into your AI assistant.

Each plugin is authored once and packaged for both platforms, so System Integrators get the same skills whether they work in Claude or ChatGPT.

## Quick Start

### Claude (Cowork / Claude Code)

```text
/plugin marketplace add vtex/ai-plugins
/plugin install vtex-solution-architect-agent@vtex-ai-plugins
```

Or download the packaged plugin from [`plugins/vtex-solution-architect-agent/claude/dist/vtex-solution-architect-agent.plugin`](./plugins/vtex-solution-architect-agent/claude/dist/vtex-solution-architect-agent.plugin) and install it from **Settings → Plugins → Install from file** in the Claude desktop app.

### ChatGPT (Codex)

Download the packaged plugin from [`plugins/vtex-solution-architect-agent/codex/dist/vtex-solution-architect-agent.zip`](./plugins/vtex-solution-architect-agent/codex/dist/vtex-solution-architect-agent.zip) and install it through Codex plugin support (desktop, CLI, or IDE extension).

Full installation, authentication, and getting-started steps: [Onboarding Guide](./docs/vtex-solution-architect-agent/onboarding-guide.md).

## Available Plugins

| Plugin | Platforms | Description | Docs |
| ------ | --------- | ------------ | ---- |
| **VTEX Solution Architect Agent** | Claude, ChatGPT (Codex) | Architecture guidance, platform expertise, and read-only account inspection for System Integrators | [Onboarding guide](./docs/vtex-solution-architect-agent/onboarding-guide.md) |

### Skills in VTEX Solution Architect Agent

| Skill | What it does |
| ----- | ------------- |
| **VTEX Architect** | Answers architecture questions — account topology, storefront decisions, integration patterns, trade-offs — backed by the Atlas knowledge base of real VTEX implementation cases. |
| **VTEX Expert** | Explains how specific VTEX platform features work — OMS lifecycle, catalog structure, checkout behavior, logistics configuration, promotions, payments. |
| **VTEX Inspector** | Inspects the live, read-only configuration of a specific VTEX account — carriers, warehouses, trade policies, payment affiliations, promotions, installed apps. |
| **VTEX Architect Scribe** | Drafts technical documentation and verifies it against ground truth with an independent adversarial fact-check before it ships. |

## Platforms

- **Claude** (Cowork / Claude Code) — `plugins/<plugin>/claude/`
- **ChatGPT** (Codex) — `plugins/<plugin>/codex/`

## Feedback and Support

The VTEX Solution Architect Agent has a built-in feedback mechanism — rate any response and flag issues directly through the skill. Feedback is reviewed by the VTEX Atlas team to improve the knowledge base. See [Feedback and support](./docs/vtex-solution-architect-agent/onboarding-guide.md#feedback-and-support) for details.
