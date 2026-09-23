# VTEX Solution Architect Agent — System Integrator Onboarding

> **Audience:** VTEX System Integrators  
> **Plugin:** VTEX Solution Architect Agent (first plugin in the [ai-plugins](../../README.md) marketplace)

VTEX SA Agent is an AI assistant built for VTEX implementation teams. It helps Solution Architects and technical leads at System Integrators make better platform decisions faster — with access to VTEX's curated architecture knowledge base, live platform documentation, and account inspection tools, through a conversational interface.

It is distributed as plugins for **Claude** (Cowork / Claude Code) and **ChatGPT** (Codex).

---

## What it can do

| Skill | What it does |
|-------|-------------|
| **VTEX Architect** | Answers architecture questions — account topology, storefront decisions, integration patterns, trade-offs. Queries the Atlas knowledge base of real VTEX implementation cases. |
| **VTEX Expert** | Explains how specific VTEX platform features work — OMS lifecycle, catalog structure, checkout behavior, logistics configuration, promotions, payments. |
| **VTEX Inspector** | Inspects the live configuration of a specific VTEX account — carriers, warehouses, trade policies, payment affiliations, promotions, apps installed. Read-only. |
| **VTEX Architect Scribe** | Drafts technical documentation with an independent adversarial fact-check before it ships. |

---

## What it cannot do

- **Modify any VTEX account configuration.** All account access is strictly read-only.
- **Access accounts you are not authorized for.** Each session is scoped to accounts associated with your credentials — cross-account access is blocked at the authentication layer.
- **Replace VTEX SA review for complex decisions.** The agent is a decision-support tool. For high-stakes or non-standard architectures, always validate with your VTEX Solution Architect.
- **Access non-public VTEX internal data.** The knowledge base contains curated, anonymized cases and platform behavior notes — not confidential client data.

---

## Requirements

- **Claude desktop app (Cowork)** and/or **ChatGPT with Codex plugin support**, depending on which client you use.
- **Node.js ≥ 18** — required to run the plugin's MCP servers.
- A VTEX System Integrator account registered with VTEX — used for authentication and account scoping.
- A VTEX platform login for the account(s) you want to inspect. The **VTEX Inspector** skill requires you to run `vtex login <account>` from a terminal before use — see [Authentication](#authentication).

---

## Installation

### Claude (Cowork)

1. Download the plugin file from [`claude-plugin/dist/vtex-solution-architect-agent.plugin`](../../claude-plugin/dist/vtex-solution-architect-agent.plugin).
2. Open the Claude desktop app (Cowork).
3. Go to **Settings → Plugins → Install from file** (or **Customize → Plugins**).
4. Select the `.plugin` file and follow the prompts.
5. Once installed, the VTEX SA Agent will appear in your plugin list.

**Claude Code (marketplace):**

```text
/plugin marketplace add vtex/ai-plugins
/plugin install vtex-solution-architect-agent@vtex-ai-plugins
```

### ChatGPT (Codex)

1. Download the Codex plugin package from [`codex-plugin/dist/vtex-solution-architect-agent.zip`](../../codex-plugin/dist/vtex-solution-architect-agent.zip).
2. Install it through Codex plugin support (desktop, CLI, or IDE extension).
3. Start a **new** Codex task after install so skills and MCP tools load.
4. Authenticate Atlas when prompted (see below), or run `codex mcp login vtex-architect-mcp`.

---

## Authentication

The agent uses two separate authentication flows, depending on which skill you're using.

### VTEX Architect and VTEX Expert

These skills query the Atlas knowledge base over OAuth. The first time you use either skill, you will be prompted to authenticate:

1. Click **Connect** when the agent asks for authorization.
2. You will be redirected to the VTEX SA Agent login page.
3. Enter your registered email address and the access token provided by your VTEX contact.
4. Once authenticated, your session is valid for **1 year**. You will only need to re-authenticate after expiry or if you revoke access.

> Your credentials are scoped to your organization's accounts. You can only inspect VTEX accounts that have been associated with your SI profile.

### VTEX Inspector

This skill reads live account configuration through a local VTEX CLI session — it does **not** use the OAuth flow above. Before using it:

1. Install the VTEX CLI if you don't already have it.
2. Run `vtex login <account>` in a terminal for each account you need to inspect.
3. Keep that terminal session's login active — the skill reads your local VTEX session token to authenticate.

If you skip this step, Inspector queries will fail even if you've already completed the OAuth login above.

---

## Getting started

Once installed and authenticated, try these starter queries:

- *"/vtex-architect What's the best account strategy for a client with 3 brands across 5 countries?"*
- *"/vtex-expert How does the VTEX OMS handle partial fulfillment?"*
- *"/vtex-inspector Inspect the shipping configuration for account `myaccount`"*
- *"/vtex-architect What's the recommended architecture for a D2C operating in multiple countries with different currencies and languages?"*

---

## Feedback and support

The agent has a built-in feedback mechanism. After any response, you can submit feedback directly through the skill — rating the answer quality and flagging any issues. Feedback is reviewed by the VTEX Atlas team to improve the knowledge base.

---

## Disclaimer

VTEX SA Agent is an AI-assisted tool. Like any AI system, it can produce incomplete, outdated, or incorrect information — always validate its output before relying on it for a client deliverable.

- Responses are decision support, not decisions. VTEX makes no guarantee as to the accuracy, completeness, or fitness of any answer, architecture recommendation, or generated document.
- The System Integrator is solely responsible for all implementation decisions, designs, and configurations made using the agent's output, including validating recommendations against the client's actual requirements and current VTEX documentation.
- Use of the agent does not substitute for review by a VTEX Solution Architect on high-stakes or non-standard architectures.
