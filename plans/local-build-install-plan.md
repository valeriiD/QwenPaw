# QwenPaw — Local Build, Install & Configure (branch: release/v2.1.0)

Fully local, no Docker. Editable source install of the current checkout.

## Context

- Repo: /home/valerii/source/QwenPaw, branch `release/v2.1.0`
- Requires: Python >=3.11,<3.14 (repo pins 3.11), Node 22 (`.nvmrc`) for console build
- Entry points: `qwenpaw init --defaults` → `qwenpaw app` (Console at http://127.0.0.1:8088/)
- Working dir: `~/.qwenpaw/` (config.json, workspaces); secrets stored encrypted separately

## Model providers (both built-in — no custom providers needed)

| Plan | Built-in provider name | Base URL | Key |
|------|----------------------|----------|-----|
| DashScope Pro (token plan) | Aliyun Token Plan (International) | https://token-plan.ap-southeast-1.maas.aliyuncs.com/compatible-mode/v1 | sk-sp-… |
| z.AI Coding plan (Max) | Zhipu Coding Plan (Z.AI) | https://api.z.ai/api/coding/paas/v4/ | 1c83663e… |

### All models to enable (from provider_manager.py built-in catalogs)

**Aliyun Token Plan (International)** — 13 models:
- Qwen3.7 Plus *(multimodal: image+video)* ← recommended default
- Qwen3.7 Max, Qwen3.6 Plus *(image+video)*, Qwen3.6 Flash *(image+video)*
- DeepSeek-V4 Pro, DeepSeek-V4 Flash, DeepSeek-V3.2
- GLM-5.2, GLM-5.1, GLM-5
- MiniMax-M2.5
- Kimi K2.6 *(image+video)*, Kimi K2.5 *(image+video)*

**Zhipu Coding Plan (Z.AI)** — 6 models:
- GLM-5.2 ← recommended default for this provider
- GLM-5.1, GLM-5, GLM-5-Turbo, GLM-5V-Turbo *(vision)*, GLM-4.7-Flash *(free)*

### Recommended parameters approach

- Capability flags (image/video support, free tier) ship in the built-in
  catalogs — enabling a provider exposes its full model list with correct metadata
- Keep provider `generate_kwargs` defaults (QwenPaw ships tuned defaults);
  only override per-model `max_input_length` if the Console flags a mismatch
- Set active default model = `qwen3.7-plus`; confirm every model appears in
  the Console model picker

## Steps

1. **Prerequisites check**
   - `python3 --version` (need 3.11–3.13), `node --version` (need 22.x), `npm --version`
   - If Python 3.11+ missing → install via uv (`uv python install 3.11`) or system package manager
   - If Node 22 missing → nvm or system package manager

2. **Build console frontend** (required for web UI)
   ```bash
   cd console && npm ci && npm run build && cd ..
   ```

3. **Stage console into package dir**
   ```bash
   mkdir -p src/qwenpaw/console
   cp -R console/dist/. src/qwenpaw/console/
   ```

4. **Python venv + editable install**
   ```bash
   python3 -m venv .venv
   source .venv/bin/activate
   pip install -e .          # add ".[dev,test,full]" only if running the test suite
   ```

5. **Initialize working dir**
   ```bash
   qwenpaw init --defaults    # creates ~/.qwenpaw, config.json, default workspace
   ```
   Note: `--defaults` auto-accepts anonymous telemetry; opt out later via prompt/config if desired.

6. **Configure models via Console**
   ```bash
   qwenpaw app                # serves on 127.0.0.1:8088
   ```
   Open http://127.0.0.1:8088/ → Settings → Models:
   - Enable **Aliyun Token Plan (International)** → paste `sk-sp-…` key → run connection check
   - Enable **Zhipu Coding Plan (Z.AI)** → paste key → run connection check
   - Pick a default chat model (e.g. qwen-max / glm-4.x coding models per plan)

7. **Verify**
   - Send a test message in Console chat
   - `qwenpaw doctor` for install health check

## Notes

- API keys stay local: QwenPaw encrypts provider keys under `~/.qwenpaw.secret/`
- TUI alternative: `qwenpaw` command after config
- Updating later: `git pull` → rebuild console → re-copy dist → `pip install -e .` → restart
