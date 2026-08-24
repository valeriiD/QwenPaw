# QwenPaw Adoption — Part 1: Analysis

> **Date:** 2026-08-23
> **Author:** Valerii (Senior Developer) + Zoo agent (GLM-5.3, QwenPaw-contributor session)
> **Status:** Proposal — draft for review (design layer)

---

## 1. The Boomerang v2 Runtime Contract

Boomerang v2 currently runs on **Windows 11 + the in-house Roo-Bob fork** of the
Roo Code VS Code extension (NORMATIVES-ADMIN §1). The operative layer
(`AIs-roo/`) is the **complete functional specification** of what is being
adopted — it defines the runtime capabilities below. Any replacement runtime
must provide equivalents or a bridge. ⛔ Porting work MUST derive exclusively
from `AIs-roo/`; the template layer (`AIs-alpha-omega/`) is not a porting
source (may be outdated/incomplete).

### 1.0 Porting backlog — full `AIs-roo/` inventory

Every item below is in scope for adoption. The porting manifest (Linux Part 2,
Phase 2) MUST map each to a destination in `AIs-qwenpaw/`:

| `AIs-roo/` item | Contents | Porting destination (target) |
|---|---|---|
| `system.md` | Runtime system-prompt template (platform tools, gates, personality) | Agent `AGENTS.md` base + compat-plugin tool docs |
| `modes/manifest.yaml` | 21 mode registrations (8 team + 13 scout) | `AIs-qwenpaw/agents/` (team) + skills (scouts) |
| `modes/rules/*.md` | Per-mode self-contained rules | per-agent rule files / SKILL.md front-matter |
| `Rules/boris.md` | Mode-agnostic parent rules (persona, knowledge protocol, git, filesystem) | agent base rules shared by all agents |
| `Workflows/*.md` (26) | Slash-command workflows (`/knowledge-*`, `/session-*`, `/rules-*`, `/initiate-*`, …) | skills with scripts |
| `skills/` (normatives, vs-mcp) | Universal skill definitions | skills (near-direct port) |
| `modes_config.json` | codeSearch intent gate, forking preambles, apiProfiles | compat plugin (`gates.py`, `preambles.py`, profile map) |
| `settings.json` | toolGroupsOverride, MCP group aliases, disabled built-ins | compat plugin (`groups.py`) + per-agent MCP registration |
| `mcp_settings.json` | 12 MCP servers + env | QwenPaw MCP registry (encrypted creds) |
| `context-condense-prompt.md` | Condensation contract (12 sections, Knowledge Debt) | condensation skill (budget-triggered) |
| `context-copy-filter.ts` | Forked-context filter | compat plugin (`fork_filter.py`) |
| `Workflows/initiate-*.md` (3) | Spawn-time/session-entry boot gates (D22/D22b/D22c): re-anchor mode gates as the freshest context at spawn/session start — anti-attention-decay mechanism | compat plugin `qp_new_task` first-action injection (child boot procedure) + session-entry skill |

### 1.1 Mode system

- **21 custom modes** (`AIs-roo/modes/manifest.yaml`: 8 team + 13 scout), each
  with self-contained rules under `AIs-roo/modes/rules/`.
- **26 slash-command workflows** (`AIs-roo/Workflows/`), including three
  spawn-time/session-entry **boot gates** (`/initiate-architect`,
  `/initiate-explorer`, `/initiate-programmer`, added 2026-08-24) that
  re-anchor mode contracts as the freshest context at spawn — the compat
  plugin must reproduce this injection mechanic.
- **Per-mode tool groups** (`settings.json → roo-bob.toolGroupsOverride`):
  e.g. architect gets `show_user`/`new_task`/`read_command_output` but not
  `execute_command`; team-mode gets delegation + shell; system tools
  (`run_slash_command`, `switch/restore_provider_profile`, `update_todo_list`,
  `get_runtime_state`) are a distinct group.
- **Per-mode MCP enablement** via group aliases (`scout-standard`,
  `team-standard`, `team-root`, `git`, `git-read`, `external-search`, …)
  composing allowlists over 12 configured MCP servers.

### 1.2 Custom built-in tools (fork-level, not plugins)

| Tool | Semantics that must survive |
|---|---|
| `new_task` | Cross-mode delegation; `fork:true` copies parent context filtered through `context-copy-filter.ts`; `fork:false` clean context; injected preambles per mode (modes_config.json `forkingPreamble`, incl. non-forked variant) |
| `return_to_caller` | Terminates a delegated subtask; returns the contract payload to parent |
| `show_user` | Session-root: pause for user input. Forked child: bubbles response up as tool result. Content-delivery gates depend on it (system.md CONTENT DELIVERY GATES) |
| `update_todo_list` | Whole-list-replace checklist; **checkpoints derive completed/[OPEN] split from todo state** — stale state corrupts restores |
| `execute_command` | Per-invocation fresh terminal; **canonical artifact persistence** (`artifact_id`, `artifact_state`, `captured_bytes`, `output_truncated`) + intent gate (`"search"|"action"`) with per-mode allowlist, chain-block on action |
| `read_command_output` | Artifact retrieval (offset/limit/search); mandatory before classifying truncated test output (§Code-5 / §Expert-4 / §Debug-2a gates) |
| `run_slash_command` | Executes workflow files as commands — never inlined |
| `switch/restore_provider_profile` | apiProfiles bookends around mapped slash commands (modes_config.json `apiProfiles`, 21 mapped commands) |

### 1.3 Context machinery

- **Condensation** (`context-condense-prompt.md`): 20–30 % budget, 12 mandatory
  sections incl. Tool-Call Reasoning Consistency and the **Knowledge Debt
  Report**; relies on the fork's guarantee that mode rules + boris.md are
  **re-injected every API call**.
- **Fork context copies** filtered by `context-copy-filter.ts`.

### 1.4 MCP stack (mcp_settings.json)

knowledgebase (Neo4j, 37 tools / 12 domains — **the knowledge authority**),
filesystem (in-house Go binary, `valeriiD/filesys-1.0.0`), git-mcp-server,
sequentialthinking, zai-mcp-server, zread (HTTP), docker-hub, context7,
brave-search, microsoft-learn (HTTP); disabled: sqlserver-pro, vs-mcp.

### 1.5 Governance layers

Conceptual (`docs/`) · Template (`AIs-alpha-omega/`) · Operative (`AIs-roo/`,
sole truth, self-contained files) · Runtime (`%USERPROFILE%\.roo-bob\`, sync
target). Two stylistic registers (heuristic for KG, directiveful for
infrastructure). Agentic CI: agent-as-gate, testing guide is law,
`workflow_dispatch`-only runners (manifesto).

---

## 2. QwenPaw Capability Inventory (verified 2026-08-23, v2.1.0)

Verified by building and exercising the platform end-to-end on Linux
(branch `release/v2.1.0`; fork `valeriiD/QwenPaw`):

1. **Agent OS workspaces** — per-agent dirs with `AGENTS.md`, `SOUL.md`,
   `PROFILE.md`, `MEMORY.md`, `HEARTBEAT.md`, own skills, own model slot;
   multi-agent + runtime **sub-agents** with independent memory.
2. **MCP connector layer** — protocol-neutral MCP client (stdio + remote),
   encrypted credentials (`~/.qwenpaw.secret/`), per-call policy gate; managed
   from Console/REST (`/api/mcp...`).
3. **Skills system** — `SKILL.md` (+ `references/`, `scripts/`), skill pool,
   enable/disable manifests, cron-driven or agent-invoked; marketplace import.
4. **Plugin system** — Python plugins registering **tools**, channels,
   providers, doctor entry-points (in-repo examples: qwen-image, wan27 tool
   plugins). `plugin.json` manifest + `register()` API.
5. **LLM provider management** — 30+ built-in providers, JSON model catalog,
   per-model params, OAuth, local runtimes; verified live against 5 providers.
6. **Security stack** — kernel sandbox (Landlock/bubblewrap on Linux,
   AppContainer on Windows, Seatbelt macOS), Tool Guard (YAML rule engine,
   approval levels), File Guard, Skill Scanner, access policy.
7. **Context** — Scroll Context: every turn persisted verbatim; evicted turns
   indexed and recallable on demand (no lossy summarization in the base loop).
8. **Memory** — ReMe self-evolving personal knowledge base (markdown, local,
   editable) — conversational memory, *not* a graph DB.
9. **Interfaces** — Console (web, port 8088), full-screen **TUI** (same
   agent/memory/sessions), REST API (`/api/...`), **ACP** (Agent Client
   Protocol) endpoint (`qwenpaw acp`), desktop app (Tauri, beta), IM channels
   (Telegram/Discord/DingTalk/…).
10. **Scheduling** — cron jobs, heartbeat check-ins.
11. **Ops** — `qwenpaw doctor` (install/config/agent/API health + fix runners),
    headless `qwenpaw app`, per-agent daemon, update machinery.

---

## 3. Mapping: Boomerang need → QwenPaw capability

| # | Boomerang v2 need | QwenPaw capability | Fit | Notes |
|---|---|---|---|---|
| 1 | MCP stack (§1.4) | MCP connector layer | ✅ direct | Node stdio servers run unchanged on Linux; Neo4j bolt endpoint network-reachable; HTTP servers (zread, microsoft-learn) native |
| 2 | `new_task` delegation | Sub-agents + multi-agent | ✅ conceptual | fork-true/false + copy-filter must be rebuilt in compat plugin (G-1) |
| 3 | 21 modes (8 team incl. Explorer + 13 scouts) | Agents (team incl. Explorer — a premium-reasoning contract-bearing team mode, NOT a scout) + skills/sub-agents (scouts) | ✅ with mapping | 21 heavyweight agents would be wrong; see §5.2 |
| 4 | Slash workflows (26) | Skills (`SKILL.md` + scripts) | ✅ port | mechanical; `/cmd` → skill invocation; provider profiles → per-agent/model slots |
| 5 | Condensation + re-injection | Scroll Context | ✅ stronger base | Knowledge-Debt workflow must be ported as a skill; no re-injection needed — rules live in agent md_files |
| 6 | Cron (knowledge-pulse…) | cron + heartbeat | ✅ native | |
| 7 | Intent gate / tool allowlists | Tool Guard + access policy + per-agent tools | ✅ partial | per-mode allowlist semantics need compat plugin config (G-1) |
| 8 | Agent-as-gate CI runner | **headless Linux server + REST + sandbox** | ✅✅ | better than desktop VS Code ext for `workflow_dispatch` runners (manifesto §2/§4) |
| 9 | Filesystem discipline | File Guard + sandbox | ✅ | allowlist dirs ≈ `MCP_ALLOWED_DIRS` model |
| 10 | Knowledge authority (KB) | via MCP — unchanged | ✅ | ReMe stays subordinate conversational memory; KB remains sole authority (reuse cycle intact) |
| 11 | Editor embedding (VS Code) | ACP + Console + TUI | ⚠️ | different UX; ACP bridges editors; cultural variable (manifesto §5) |
| 12 | Custom built-ins (§1.2) | — | ❌ | **the gap** → Boomerang compatibility plugin (G-1) |

---

## 4. Gap Register

| ID | Gap | Severity | Resolution path |
|---|---|---|---|
| G-1 | Roo-Bob custom built-ins (fork-level tools, tool groups, artifact shell, intent gate, provider bookends, todo-checkpoints) | **blocking** | Build **Boomerang compatibility plugin** (Python, platform-neutral) on QwenPaw plugin API: register `qp_new_task`, `qp_show_user`, `qp_return_to_caller`, `qp_execute_command` (artifact-backed), `qp_read_artifact`, `qp_todo`; per-agent tool-group config; port `context-copy-filter` logic to the plugin's fork handler |
| G-2 | Mode granularity (cheap roles vs heavyweight agents) | high | Mapping doc: 8 team modes → agents; 13 scouts → skills/sub-agents spawned with clean or forked context |
| G-3 | Platform shift (Win→Linux binaries, `R:\`, Git Bash) | medium | Linux plans: filesystem-MCP needs Linux build of `valeriiD/filesys` (or in-tree equivalent); npx servers unchanged; Windows plan keeps everything native |
| G-4 | Condensation behaviour (Knowledge Debt, reasoning-consistency) | medium | Port `context-condense-prompt` semantics as a skill triggered on context budget; Scroll Context covers persistence |
| G-5 | UX / developer habits | medium-cultural | ACP into editors; TUI for terminal users; Console for oversight; parallel-run with Roo-Bob until parity |
| G-6 | Secrets in `mcp_settings.json` env | low | move to QwenPaw encrypted credential store (verified working) |
| G-7 | QwenPaw release cadence dependency | low | own fork (`valeriiD/QwenPaw`) + contributor workflow already established (2026-08-23); pin release branches |

---

## 5. Verdict

### 5.1 Can QwenPaw adopt Boomerang v2 semantics? — Yes, with one decisive investment

The **only blocking gap is G-1** — and it is the same class of work the team
already does maintaining the Roo-Bob fork, except as a *platform-neutral plugin
against a stable public plugin API* instead of source patches against a
fast-moving VS Code extension. Everything else is mapping, porting workflows,
and platform packaging.

### 5.2 The mapping that makes it work

- **Team modes (8: architect, expert, code, debug, discusser, runner,
  writer, explorer)** → QwenPaw **agents** (own workspace, rules in
  `AGENTS.md`/`SOUL.md`, own model slot ≈ provider profile). Explorer is a
  team mode — premium reasoning, `fork: true` ALWAYS, refusal contract,
  unconditional mapping duties (D15–D21); porting MUST preserve these
  behavioural invariants, not just the tool surface.
- **Scout modes (13)** → **skills + sub-agents** (lightweight, spawned per
  delegation with clean context per compat plugin — scouts are never forked).
- **Slash workflows (26)** → **skills** with scripts; apiProfiles → per-agent
  model slots.
- **KB (Neo4j)** → unchanged via MCP; **ReMe explicitly subordinate**.
- **Testing guide as law** → skills + Tool Guard rules enforce the same
  "run the suite" gates (agent-as-gate manifesto §3).

### 5.3 Strategic advantages over Roo-Bob

1. **Headless first-class citizen** — matches the CI manifesto better than a
   desktop extension (no GUI session on runners).
2. **Kernel sandbox + Tool Guard** — enforcement depth Roo-Bob lacks.
3. **Apache-2.0 upstream, active, contributing relationship established** —
   replaces "maintain private fork" liability with "maintain plugin".
4. **Channels** — Boomerang v2 becomes reachable (Telegram/Discord/etc.) for
   oversight, approvals (`show_user` analogues), and alerts — a new capability,
   not a port.
5. **Scroll Context** — verbatim persistence beats lossy condensation for
   auditability; condensation becomes an *opt-in* budget tool.

### 5.4 Risks

- Plugin API churn (mitigate: pin release branch, contrib relationship).
- Cultural adoption (mitigate: parallel-run, Windows plan keeps familiar UX
  via ACP/desktop app).
- Effort estimate for G-1: comparable to date invested in the Roo-Bob fork
  tooling (DD#57-class work), split across the phased plans (Parts 2–3).
