# QwenPaw Adoption — Part 2: Linux Adoption Plan

> **Date:** 2026-08-23
> **Author:** Valerii (Senior Developer) + Zoo agent (GLM-5.3, QwenPaw-contributor session)
> **Status:** Proposal — draft for review (design layer)
> **Strategy:** server/CI-first — prove the runtime where it is strongest
> (headless), keep Roo-Bob authoritative until Phase-3 exit criteria.

Directiveful register for steps and gates (NORMATIVES-ADMIN §3): infra rules
are MUST/NEVER; KG-adjacent steps stay heuristic.

---

## Phase 0 — Co-existence baseline (status: **partially complete 2026-08-23**)

Everything in this phase was executed and verified in the QwenPaw-contributor
session unless marked TODO.

| Step | Detail | Status |
|---|---|---|
| 0.1 | QwenPaw 2.1.0 built from source, branch `release/v2.1.0`, venv on local disk (`~/.venvs/qwenpaw`) | ✅ done |
| 0.2 | Console + REST + doctor green; providers configured (Token Plan, Z.AI, MiniMax, DeepSeek; Kimi pending key) | ✅ done |
| 0.3 | Fork organized for contribution (`valeriiD/QwenPaw`, upstream remote, PR branch `feat/deepseek-catalog-update`) | ✅ done |
| 0.4 | Register **mcp-knowledgebase** stdio server: command `node /home/valerii/source/mcp-knowledgebase/server-stdio.js`, env `NEO4J_URI=bolt://foundation007.sollers.live:57687`, `NEO4J_USER`, `NEO4J_PASSWORD`, `EMBEDDING_PROVIDER=server`, `SERVER_EMBEDDING_ENDPOINT=http://127.0.0.1:7997` (embedding endpoint must be reachable from Linux — TODO if it binds localhost on a Windows host) | TODO |
| 0.5 | Register **git-mcp-server** (`npx @cyanheads/git-mcp-server@latest`, stdio) | TODO |
| 0.6 | Register read-only externals: sequentialthinking, context7, brave-search, microsoft-learn (HTTP), zread (HTTP) | TODO |
| 0.7 | **Gate 0:** from the running QwenPaw agent, exercise all 37 KB tools used by `knowledgebase-read` + `session-read` groups (query_memories, build_context, cypher_query, get_active_session, …). Record coverage in KB (`store_and_link`). | TODO |

**Exit rule:** Gate 0 passes ⟶ Phase 1. If the embedding endpoint is the only
blocker, run KB with `EMBEDDING_PROVIDER=local` semantics or expose the
endpoint on the LAN; NEVER weaken Neo4j auth to proceed.

## Phase 1 — Scout pilot (read-only, low risk)

Goal: one QwenPaw agent performs **Explorer / knowledge-scout** duties on
Linux while Roo-Bob remains the team runtime.

1. Create pilot agent `explorer-pilot` (workspace `~/.qwenpaw/workspaces/explorer-pilot`) —
   an Explorer-class TEAM-mode pilot, not a scout (Explorer is a premium-reasoning
   team mode with contract semantics: fork-equivalent context, refusal contract,
   mandatory map writes):
   - `AGENTS.md` ← port of `AIs-roo/modes/rules/explorer.md` (self-contained,
     neutral-path wording; direct port is acceptable at pilot scale).
   - Model slot: premium-reasoning provider (Explorer is a premium-reasoning
     team mode; a cheap model would gut the contract compliance — see KB
     58c61a30 cold-probe evidence that rule adherence is model-tier-conditional).
   - Tool surface: KB read tools + filesystem read + git-read only. Enforce via
     compat plugin config when it exists; at pilot scale enforce by
     **omission** (register only those MCP servers on this agent) and File
     Guard allowlist (`/home/valerii/source` read-only).
2. Port 2–3 scout workflows as skills (start with `knowledge-search`,
   `rules-recall`): `SKILL.md` front-matter with trigger keywords + scripts
   lifted from `AIs-roo/Workflows/`.
3. Cron: schedule `knowledge-pulse`-equivalent skill on the agent (QwenPaw
   cron is per-agent and native).
4. **Gate 1 (parity checklist, per run):** for 10 real queries issued to both
   Roo-Bob explorer and `explorer-pilot`: same memories retrieved (by id), same
   citation behaviour, reuse-yield report produced. 8/10 semantic matches =
   pass.
5. **Rollback:** delete agent + skills; zero impact on Roo-Bob (nothing shared
   except the KB, which is append-mostly and leased per workflow).

## Phase 2 — Boomerang compatibility plugin (the decisive build)

Single Python plugin, platform-neutral (ships to Windows in Part 3 unchanged):
`qwenpaw-boomerang/`

```
plugin.json            # type: tool (+ future channel for approvals)
boomerang/
  tools.py             # qp_new_task, qp_return_to_caller, qp_show_user,
                       # qp_execute_command, qp_read_artifact, qp_todo,
                       # qp_switch_profile/qp_restore_profile
  fork_filter.py       # port of context-copy-filter.ts semantics
  gates.py             # intent gate (search/action allowlist), chain-block,
                       # artifact truncation rules (output_truncated contract)
  groups.py            # per-agent tool-group config (ports toolGroupsOverride
                       # + mcpGroupAliases as YAML next to the plugin)
  preambles.py         # forkingPreamble default/perMode/nonForked (DD#57 texts)
```

Hard requirements (MUST):

- `qp_execute_command` MUST persist a canonical artifact per invocation
  (`artifact_id`, `captured_bytes`, `artifact_state`, `output_truncated`) in
  the agent workspace; `qp_read_artifact` MUST support offset/limit/search.
  Test-classification gates depend on the truncation contract.
- `qp_new_task` MUST implement fork:true (copy filtered parent turns into the
  child's first message via `fork_filter`) and fork:false (clean context),
  plus preamble injection per `preambles.py`.
- `qp_show_user` MUST map to Console/web push + channel message in headless
  contexts; in TUI it pauses inline. The content-delivery-gate guarantee
  ("substance reaches the user") is the acceptance test, not the widget.
- `qp_todo` MUST write the checklist to the workspace so checkpoint restores
  can derive the completed/[OPEN] split (same invariant as Roo-Bob).
- Intent gate MUST reproduce modes_config.json `codeSearch` semantics,
  including single-command-per-action chain-block and the search-limit
  delegation message for architect/discusser-equivalents.
- Provider bookends: on skill invocation mapped in `apiProfiles`, switch the
  agent's model slot and restore after (QwenPaw per-agent model slots make
  this a config write + reload, no fork needed).

Deliverable acceptance = **Gate 2**: replay 5 archived Roo-Bob sessions
(from KB session history) against QwenPaw with the plugin; each replay must
produce equivalent delegation trees, artifact evidence, and todo checkpoints.
3/5 clean + 2/5 minor-divergence = pass.

Also in Phase 2 (parallel): create operative dir `AIs-qwenpaw/` in
mcp-knowledgebase per the NORMATIVES-ADMIN layer model — self-contained ports
of rules/modes/workflows — deployed by a new `Sync-QwenPawConfig.sh`
(mirrors `Sync-RooConfig.sh`; target `~/.qwenpaw/` staging dirs). NEVER edit
the deployed runtime directly.

**Porting manifest (binding, first deliverable of `AIs-qwenpaw/`):**
a tracked table mapping EVERY item of the `AIs-roo/` inventory
(01-analysis §1.0 — system.md, 21 modes, mode rules, boris.md, 26 workflows,
skills, modes_config.json, settings.json, mcp_settings.json, condense prompt,
copy-filter, sync script) to its `AIs-qwenpaw/` destination and port status.
⛔ Ports derive exclusively from `AIs-roo/` (complete functional spec);
`AIs-alpha-omega/` MUST NOT be used as a porting source. Phase 2 cannot exit
while any inventory row is unmapped.

## Phase 3 — Team modes + agent-as-gate CI on Linux

1. Stand up 8 team-mode agents from `AIs-qwenpaw/` ports (architect, code,
   expert, debug, discusser, explorer, runner, writer). Optional add-on:
   QwenPaw's built-in QwenPaw_QA_Agent_0.2 as a QA assistant beyond the 8 —
   it supplements, not replaces, writer.
2. Headless deployment: `qwenpaw app` as a systemd service on the Linux
   host (same machine class as foundation007); Console for oversight; TUI for
   terminal operators; ACP for editor users.
3. **CI per the manifesto:** GitHub Actions `workflow_dispatch`-only runners
   execute `curl`/`acp` calls against the QwenPaw REST/ACP surface; the agent
   runs the testing-guide suites inside the kernel sandbox and interprets
   results (agent-as-gate §2). The YAML remains a clean runtime — no triggers,
   no schedules (§4).
4. Cron: knowledge-pulse / rules-drift-summary / hygiene jobs move from
   Roo-Bob slash-invocations to scheduled skills.
5. **Gate 3 (cutover criteria, all MUST hold for 2 weeks):**
   - All 21 modes have QwenPaw equivalents exercised in real work.
   - Reuse-yield per session ≥ Roo-Bob baseline (measured via KB, which is
     runtime-agnostic — this metric ports for free).
   - Zero open G-1 parity defects; artifact/checkpoint restores verified.
   - Team sign-off per manifesto §5 (cultural gate).
6. Cutover: Roo-Bob moves to maintenance-only; `AIs-roo/` archived to the
   template layer alongside `AIs-alpha-omega/`.

## Linux-specific risks & mitigations

| Risk | Mitigation |
|---|---|
| filesystem-MCP is a Windows Go binary | Build `valeriiD/filesys` for linux/amd64 (Go cross-compile, trivial) — or rely on QwenPaw native file tools + File Guard, which Gate-2 replays will adjudicate |
| Embedding endpoint bound to Windows localhost | Expose on LAN behind auth, or run a Linux embedding sidecar; KB stays authoritative either way |
| Network-mount symlink corruption (observed on `//dev-desktop…/source`) | Keep venv + artifacts on local disk (session-verified workaround); document in runbook |
| Plugin API churn | Pin `release/vX` branches on the fork; contrib workflow already established |
| Secrets sprawl | All env secrets move into QwenPaw encrypted credential store (verified: `~/.qwenpaw.secret/`, encrypted at rest) |
