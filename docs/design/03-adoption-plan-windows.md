# QwenPaw Adoption — Part 3: Windows Adoption Plan

> **Date:** 2026-08-23
> **Author:** Valerii (Senior Developer) + Zoo agent (GLM-5.3, QwenPaw-contributor session)
> **Status:** Proposal — draft for review (design layer)
> **Strategy:** workstation parity — replace the Roo-Bob extension in its home
> environment, preserving developer UX (editor presence, Git Bash, `R:\` paths)
> while consuming the same platform-neutral compatibility plugin built in
> Linux Part 2 Phase 2.

Directiveful register for infra steps; heuristic for KG-adjacent steps
(NORMATIVES-ADMIN §3).

---

## W-Phase 0 — Parallel install (no behaviour change)

| Step | Detail |
|---|---|
| 0.1 | Install QwenPaw on the Win11 workstation. Options, in preference order: (a) script installer `irm https://qwenpaw.agentscope.io/install.ps1 \| iex` (manages uv + venv under `%USERPROFILE%\.qwenpaw\`); (b) `pip install qwenpaw` into an existing Python 3.11–3.13; (c) from the fork: `pip install -e R:\source\QwenPaw` after building the console (`npm ci && npm run build` in `console\`). NEVER install into the same venv as other tooling. |
| 0.2 | Run `qwenpaw init --defaults --accept-security`, then `qwenpaw app` → verify Console at `http://127.0.0.1:8088/` and `qwenpaw doctor` all-OK. |
| 0.3 | Register the existing MCP stack into QwenPaw — the entries port **verbatim** from `mcp_settings.json` because they are already Windows-native: knowledgebase (`C:\Program Files\nodejs\node.exe R:\source\mcp-knowledgebase\server-stdio.js` + Neo4j env), filesystem (`R:\dstrs\filesystem-mcp-windows-amd64-1.3.1.exe`, `MCP_ALLOWED_DIRS=C:\users\valer,R:\source`), git-mcp-server, sequentialthinking, zai, zread, docker-hub, context7, brave-search, microsoft-learn. Keep sqlserver-pro / vs-mcp disabled, as today. Move env secrets into QwenPaw's encrypted store instead of plaintext env (G-6). |
| 0.4 | **Gate W0:** same KB-tool coverage check as Linux Gate 0 (37 read-domain tools from the Windows install). |

Note: Roo-Bob and QwenPaw coexist harmlessly (different config trees
`%USERPROFILE%\.roo-bob\` vs `%USERPROFILE%\.qwenpaw\`); both talk to the same
Neo4j KB. Do not change `AIs-roo/` in this phase.

## W-Phase 1 — Developer UX parity study

Goal: answer "can a developer live in QwenPaw instead of the Roo pane?" with
evidence, before any mode porting.

1. **Shell equivalence:** QwenPaw command execution must use Git Bash for
   parity with `roo-bob.execaShellPath` (`C:\Program Files\Git\bin\bash.exe`).
   Configure via the compat plugin's `qp_execute_command` (Part 2) or the
   agent's environment. Verify `&&` chaining, forward-slash path handling, and
   the `cwd` semantics developers rely on.
2. **Editor presence:** evaluate, in order:
   - **ACP** (`qwenpaw acp`) into ACP-capable editors — closest analogue to
     an embedded agent pane;
   - **TUI** (`qwenpaw`) in Windows Terminal for terminal-first developers;
   - **Console** in a browser pane / VS Code simple-browser for oversight;
   - **Tauri desktop app** (beta) for non-terminal users — note the README
     LTSC warning: Constrained Language Mode may require manual PATH setup.
3. **Path discipline:** all workflows/skills ported to `AIs-qwenpaw/` MUST use
   workspace-relative paths (already a Boomerang rule — system.md PLATFORM
   RULES) so the same files work on both OSes.
4. **Gate W1:** two developers complete one real task each (one bug fix via
   architect→code→debug flow on QwenPaw-Console/ACP, one on TUI) with KB
   pre-flight + harvest; reuse-yield reported and ≥ Roo-Bob baseline for the
   same workspaces.

## W-Phase 2 — Compat plugin lands on Windows

The `qwenpaw-boomerang` plugin from Linux Part 2 Phase 2 is platform-neutral
Python — deployment is `plugins/tool/qwenpaw-boomerang/` in the QwenPaw
working dir (or via the fork). Windows-specific verification:

- `qp_execute_command` artifacts under `%USERPROFILE%\.qwenpaw\…` on NTFS
  (long-path enablement if artifacts grow past 260 chars — prefer
  short artifact names + offset reads);
- AppContainer sandbox interplay: QwenPaw's Windows sandbox is AppContainer —
  verify it does not block `R:\` network-share access that Boomerang workflows
  assume; if it does, scope the sandbox to deny-by-exception for team agents
  and keep File Guard allowlists (`R:\source`, `C:\users\valer`) as the
  primary control, matching today's `MCP_ALLOWED_DIRS` model;
- git-mcp-server + `Session-Commit:` / `Knowledge-Harvested KB:` trailers —
  unchanged, the server is the same npx package;
- **Gate W2:** replay the same 5 archived sessions as Linux Gate 2 on the
  Windows install; identical pass bar.

## W-Phase 3 — Team-mode cutover on Windows

1. Deploy `AIs-qwenpaw/` (created in Linux Part 2) via
   `Sync-QwenPawConfig.sh` to `%USERPROFILE%\.qwenpaw\` staging.
2. Port the 8 team modes as agents + 13 scouts as skills (same mapping both
   OSes — single source of truth in `AIs-qwenpaw/`, derived exclusively from
   `AIs-roo/`; the template layer is not a porting source).
3. Interim co-running per developer preference; Roo-Bob stays installable.
4. **Gate W3 (mirror of Linux Gate 3):** 2 weeks of all-modes real work,
   reuse-yield ≥ baseline, zero G-1 parity defects, team sign-off
   (manifesto §5). Then Roo-Bob → maintenance-only on Windows too.

## Windows-specific risks & mitigations

| Risk | Mitigation |
|---|---|
| Desktop app is Beta (README: incomplete compat testing, startup 10–60 s) | Prefer ACP/TUI/Console for parity study; desktop app optional track |
| LTSC / Constrained Language Mode breaks install.ps1 | Documented fallback exists (manual uv install + PATH, README §Windows LTSC); include in runbook |
| AppContainer sandbox vs `R:\` share access | W-Phase 2 explicit verification; fallback to File Guard as primary gate |
| Long paths / artifact growth | short artifact ids + offset/limit reads (already in the tool contract) |
| Two runtimes drifting during co-running | `AIs-roo/` frozen (edits prohibited) once `AIs-qwenpaw/` reaches Phase 3; KB records which runtime produced each session (runtime-agnostic metric) |
| Python version drift on workstation | installer-managed uv venv (pin 3.11–3.13 per `requires-python`) |

## Sequencing against the Linux plan

- Linux Phase 2 (compat plugin) is the prerequisite for Windows Phase 2 —
  build once, verify on both.
- Windows Phase 0–1 can start immediately and independently.
- Recommended order: **W0/W1 ∥ L0/L1 → L2 (plugin) → W2 → L3/W3 together**,
  with cutover gates evaluated per-OS.
