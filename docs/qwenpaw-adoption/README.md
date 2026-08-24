# QwenPaw as Boomerang v2 Runtime — Adoption Study

> **Date:** 2026-08-23
> **Author:** Valerii (Senior Developer) + Zoo agent (GLM-5.3, QwenPaw-contributor session)
> **Status:** Proposal — draft for review (design layer; not operative)
> **Revision:** 2026-08-24 — mode-taxonomy correction pass (Explorer = team mode
> not scout; 8 team modes enumerated incl. writer; Phase-1 pilot renamed
> explorer-pilot with premium-reasoning model slot; workflow count 23→26 with
> boot-gate porting requirement; compat-plugin first-action injection note)

## What this is

A three-part study evaluating **QwenPaw** (Apache-2.0, AgentScope-based personal
agent platform, `agentscope-ai/QwenPaw`) as a replacement runtime for the
Roo-Bob fork, per the template-layer intent of NORMATIVES-ADMIN §2.2
("a future Roo-Bob replacement, a different agent runtime that adopts
Boomerang v2 semantics").

| Document | Contents |
|---|---|
| [`01-analysis.md`](01-analysis.md) | Current-state contract, capability mapping, gap register, verdict |
| [`02-adoption-plan-linux.md`](02-adoption-plan-linux.md) | Phased Linux adoption (server/CI-first) |
| [`03-adoption-plan-windows.md`](03-adoption-plan-windows.md) | Phased Windows adoption (workstation parity) |

## Porting source of truth (binding)

**`AIs-roo/` is the exclusive, complete functional specification** of what is
being adopted into QwenPaw — every capability listed there is in scope. The
template layer (`AIs-alpha-omega/`) is **not a porting source**: it may be
outdated and incomplete, and is referenced only as historical evidence that a
runtime port was anticipated. Any discrepancy between the two resolves in
favour of `AIs-roo/` (per NORMATIVES-ADMIN §2.3: the operative layer is the
sole source of truth, self-contained).

## Ground truth basis

- QwenPaw **2.1.0** built, installed, configured and exercised end-to-end on
  Linux (branch `release/v2.1.0`, fork `valeriiD/QwenPaw`) — providers, MCP
  layer, agent workspaces, skills, cron, REST API, doctor — during the
  2026-08-23 session. Claims below cite either that session's verified
  behaviour or named source files.
- Boomerang v2 contract read from the operative layer: `AIs-roo/system.md`,
  `modes_config.json`, `settings.json`, `mcp_settings.json`,
  `context-condense-prompt.md` and the conceptual docs (`NORMATIVES-ADMIN.md`,
  `agentic-ci-manifesto.md`, `Knowledge-Reuse-Strategy-Executive-Brief.md`).

## Layer placement

These documents live in the **design layer** (`docs/design/`). They do not
modify the operative layer (`AIs-roo/`) and do not create the future
`AIs-qwenpaw/` operative directory — that is a Phase-2 outcome gated on the
decision criteria in the plans.

## One-paragraph verdict

QwenPaw can host Boomerang v2 semantics, but not as a drop-in swap: the
Roo-Bob fork's custom built-ins (delegation/fork context, tool groups, intent
gates, artifact-backed command execution, provider-profile bookends) must be
rebuilt as a platform-neutral **Boomerang compatibility plugin** plus a
**mode→agent/skill mapping**. Everything else — MCP stack, knowledge workflows,
cron, security posture, headless CI execution — maps cleanly, and QwenPaw's
headless Linux server is a strictly better fit for the agent-as-gate CI model
than a desktop VS Code extension. The Neo4j knowledgebase remains the single
knowledge authority in all scenarios; QwenPaw's ReMe layer is subordinate
conversational memory only.
