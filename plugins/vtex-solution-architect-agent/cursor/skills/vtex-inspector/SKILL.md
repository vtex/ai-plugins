---
name: vtex-inspector
description: debug, validate, and diagnose vtex implementations. use for errors, bugs, unexpected behavior, failed integrations, checkout, payment, orderform issues, oms inconsistencies, catalog, pricing, promotions, logistics problems, webhook failures, app validation, account setup validation, and root cause analysis. do not use for high-level architecture decisions or normal implementation tutorials.
---

# VTEX Inspector

Apply this skill when the user asks why something is broken, how to validate a setup, or how to diagnose unexpected VTEX behavior.

## Mandatory execution flow

1. Classify the request as debugging, validation, RCA, or unexpected behavior.
2. Build an error-focused retrieval query with symptoms, error messages, APIs, entities, account context, or failing steps.
3. Always call the VTEX Atlas MCP tool `retrieve_context(query)` before answering.
4. Use `vtex-developer` to verify current documentation and endpoint contracts
   when the diagnosis depends on platform or API behavior.
5. If the user mentions a client, account, merchant, workspace, production implementation, or real case, call the VTEX Atlas MCP tool `get_architecture(account)`.
6. Apply the debugging reasoning rules below.
7. Return a root cause analysis, checklist, or fix plan.

If the VTEX Atlas MCP tools are unavailable, say that the required retrieval source is unavailable and provide only a clearly marked best-effort answer from general context.

## Retrieval strategy

Use symptom-based and error-focused queries.

Examples:

- `VTEX checkout payment not updating orderForm error webhook retry failure`
- `VTEX OMS order status inconsistent payment approved order not invoiced`
- `VTEX catalog sku not indexed search missing product troubleshooting`
- `VTEX IO app route resolver error workspace production validation`

## Reasoning rules

- Start with the most likely causes.
- Separate confirmed facts, hypotheses, and unknowns.
- Validate against known VTEX constraints before proposing fixes.
- Propose the minimal safe fix first.
- Ask for logs, request IDs, order IDs, account, workspace, payloads, or timestamps only when needed to resolve ambiguity.
- Avoid recommending broad redesign before checking simpler causes.
- Include reproduction and validation steps whenever possible.

## Output formats

Choose the most useful format for the request:

- Root cause analysis
- Debugging checklist
- Validation checklist
- Fix plan
- Reproduction plan

## Default RCA format

Use this structure when the user reports a bug or failure:

1. Symptom summary
2. Most likely causes
3. Evidence to collect
4. Checks to run
5. Minimal fix first
6. Validation steps
7. Escalation path
