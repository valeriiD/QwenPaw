# QwenPaw — Comprehensive Local Build, Install & Configuration Guide

Complete playbook for building QwenPaw **from source** on Linux, configuring multiple
model providers via the REST API, and tuning every model with **vendor-doc-verified
parameters**. Based on a fully working setup of branch `release/v2.1.0` (QwenPaw 2.1.0).

> Resulting state: **5 providers / 39 models**, all doc-verified context windows,
> encrypted key storage, Console + TUI + doctor all green. Unwanted built-in
> providers (Kilo Code, OpenCode) and vendor-retired models (deepseek-chat /
> deepseek-reasoner) removed via catalog source edits (§12).

---

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [Build the console frontend](#2-build-the-console-frontend)
3. [Python environment & editable install](#3-python-environment--editable-install)
4. [Initialize the working directory](#4-initialize-the-working-directory)
5. [Run the app](#5-run-the-app)
6. [Configure providers & API keys via REST](#6-configure-providers--api-keys-via-rest)
7. [Add models missing from the catalog](#7-add-models-missing-from-the-catalog)
8. [Per-model parameters (vendor-doc-verified)](#8-per-model-parameters-vendor-doc-verified)
9. [How parameters flow to the vendor API](#9-how-parameters-flow-to-the-vendor-api)
10. [Verification & troubleshooting](#10-verification--troubleshooting)
11. [Maintenance](#11-maintenance)

---

## 1. Prerequisites

| Requirement | Version used | Notes |
|---|---|---|
| Python | **3.12.13** (any `>=3.11,<3.14`) | repo pins 3.11 in `.python-version` |
| Node.js + npm | **22.23.1 / 10.9.8** (`.nvmrc` = 22) | only needed to build the Console UI |
| [uv](https://docs.astral.sh/uv/) | 0.12.5 | optional but recommended (`curl -LsSf https://astral.sh/uv/install.sh | sh`) |
| git branch | `release/v2.1.0` | `git branch --show-current` |

```bash
python3 --version && node --version && npm --version
```

### ⚠️ Network-mount gotcha (important!)

If the repo checkout lives on a **network filesystem** (CIFS/SMB share etc., e.g.
`df .` shows `//host/...`), symlink targets get **corrupted** (`/usr/bin/python3.12`
becomes `/??r/:in/python3.12`). Both `python -m venv` and `uv venv` will silently
produce broken environments **inside the repo**.

**Fix: always create the venv on local disk** (`/home`, `/tmp`), e.g.
`~/.venvs/qwenpaw`. The editable install still points back into the repo, which is
fine — only *symlink creation* is unreliable on the mount.

---

## 2. Build the console frontend

Required for the web UI (served from `src/qwenpaw/console/`).

```bash
cd console
npm ci            # ~2 min; audit warnings are normal
npm run build     # ~2.5 min Vite build + Monaco CSS verification
cd ..
```

Stage the build output into the package directory:

```bash
mkdir -p src/qwenpaw/console
cp -R console/dist/. src/qwenpaw/console/
```

---

## 3. Python environment & editable install

```bash
# venv on LOCAL disk (see network-mount gotcha), Python 3.12
uv venv /home/<user>/.venvs/qwenpaw --python 3.12

# editable install of the repo (identical result with plain pip:
#   /home/<user>/.venvs/qwenpaw/bin/pip install -e . )
uv pip install -e . --python /home/<user>/.venvs/qwenpaw/bin/python

# sanity check
/home/<user>/.venvs/qwenpaw/bin/qwenpaw --version   # -> QwenPaw, version 2.1.0
```

Optional extras (only if you will run the test suite / pre-commit):

```bash
uv pip install -e ".[dev,test,full]" --python /home/<user>/.venvs/qwenpaw/bin/python
```

---

## 4. Initialize the working directory

Creates `~/.qwenpaw/` (config.json, workspaces, skills, agents, heartbeat).

```bash
/home/<user>/.venvs/qwenpaw/bin/qwenpaw init --defaults --accept-security
```

Notes:
- `--defaults` = non-interactive (for scripts). **`--accept-security` is required
  together with it** — without a TTY the security-confirmation prompt aborts
  (`Warning: Input is not a terminal … Aborted!`).
- `--defaults` auto-accepts anonymous telemetry (version/OS/CPU only). Opt out
  afterwards via the prompt or config.
- Output should end with `✓ Initialization complete!` (16 skills enabled by default).

---

## 5. Run the app

```bash
nohup /home/<user>/.venvs/qwenpaw/bin/qwenpaw app > /tmp/qwenpaw-app.log 2>&1 &
```

- Console UI: **http://127.0.0.1:8088/**
- Logs: `/tmp/qwenpaw-app.log` (also `~/.qwenpaw/qwenpaw.log`)
- TUI (same agent/memory as Console): `/home/<user>/.venvs/qwenpaw/bin/qwenpaw`

---

## 6. Configure providers & API keys via REST

Everything the Console UI does is available over the REST API at
`http://127.0.0.1:8088/api/...` — this is the scriptable path used throughout.
**Keys are stored encrypted** under `~/.qwenpaw.secret/providers/`.

### Endpoint reference

| Purpose | Method & path |
|---|---|
| List providers & models | `GET /api/models` |
| Set provider API key / base URL | `PUT /api/models/{provider_id}/config` |
| Provider connection test | `POST /api/models/{provider_id}/test` |
| Add a model to a provider | `POST /api/models/{provider_id}/models` |
| Per-model parameters | `PUT /api/models/{provider_id}/models/{model_id}/config` |
| Per-model live generation test | `POST /api/models/{provider_id}/models/test` |
| Set active (default) model | `PUT /api/models/active` |

> URL-encode model ids containing `/` (use `{model_id:path}` style already handled
> by the router; plain curl works for the ids below).

### Keys used in this setup (redacted — paste your own)

```bash
QP=/home/<user>/.venvs/qwenpaw/bin/python   # helper for pretty output
API=http://127.0.0.1:8088/api

# 1) Aliyun Token Plan (International)  — "DashScope Pro plan" subscription
curl -s -X PUT $API/models/aliyun-tokenplan-intl/config \
  -H "Content-Type: application/json" \
  -d '{"api_key": "<DASHSCOPE_TOKENPLAN_SK-SP-KEY>"}'
#    base_url (built-in): https://token-plan.ap-southeast-1.maas.aliyuncs.com/compatible-mode/v1

# 2) Zhipu Coding Plan (Z.AI) — "z.AI Coding plan (Max)" subscription
curl -s -X PUT $API/models/zhipu-intl-codingplan/config \
  -H "Content-Type: application/json" \
  -d '{"api_key": "<ZAI_CODING_KEY>"}'
#    base_url (built-in): https://api.z.ai/api/coding/paas/v4

# 3) MiniMax (International) — pay-as-you-go
curl -s -X PUT $API/models/minimax/config \
  -H "Content-Type: application/json" \
  -d '{"api_key": "<MINIMAX_SK-API-KEY>"}'
#    base_url (built-in): https://api.minimax.io/anthropic

# 4) DeepSeek — pay-as-you-go
curl -s -X PUT $API/models/deepseek/config \
  -H "Content-Type: application/json" \
  -d '{"api_key": "<DEEPSEEK_SK-KEY>"}'
#    base_url (built-in): https://api.deepseek.com

# 5) Kimi (International) — needs a key from platform.kimi.com
curl -s -X PUT $API/models/kimi-intl/config \
  -H "Content-Type: application/json" \
  -d '{"api_key": "<MOONSHOT_KEY>"}'
#    base_url (built-in): https://api.moonshot.ai/v1
```

### Connection tests

```bash
curl -s -X POST $API/models/aliyun-tokenplan-intl/test   # -> {"success":true,...}
curl -s -X POST $API/models/zhipu-intl-codingplan/test
curl -s -X POST $API/models/minimax/test
curl -s -X POST $API/models/deepseek/test
```

### Set the active default model

```bash
curl -s -X PUT $API/models/active \
  -H "Content-Type: application/json" \
  -d '{"provider_id": "aliyun-tokenplan-intl", "model": "qwen3.7-plus", "scope": "global"}'
# -> {"active_llm":{...},"effective_max_input_length":1000000}
```

### Verify the whole model inventory

```bash
curl -s $API/models | $QP -c "
import json,sys
data=json.load(sys.stdin)
provs = data if isinstance(data,list) else data.get('providers',[])
for p in provs:
    print('==', p['id'], '| key set:', bool(p['api_key']))
    for m in p.get('models',[])+p.get('extra_models',[]):
        print(f\"  {m['id']:22s} ctx={m['max_input_length']:>9,} max_tokens={m['max_tokens']} relay={m['relay_reasoning']}\")
"
```

---

## 7. Add models missing from the catalog

The branch's built-in catalog can lag behind vendor releases. **First probe whether
the upstream actually serves the model**, then add it:

```bash
# Probe upstream directly (Token Plan example):
curl -s https://token-plan.ap-southeast-1.maas.aliyuncs.com/compatible-mode/v1/models \
  -H "Authorization: Bearer <DASHSCOPE_TOKENPLAN_SK-SP-KEY>" | $QP -c \
  "import json,sys; print([m['id'] for m in json.load(sys.stdin)['data']])"

# Tiny chat probe to confirm routing (model_not_found vs quota/access errors):
curl -s https://.../chat/completions -H "Authorization: Bearer <KEY>" \
  -H "Content-Type: application/json" \
  -d '{"model":"<NEW-MODEL-ID>","messages":[{"role":"user","content":"hi"}],"max_tokens":5}'
```

Add + configure a new model (real example — `qwen3.8-max`, live upstream but absent
from the catalog):

```bash
# add (capability flags per vendor docs)
curl -s -X POST $API/models/aliyun-tokenplan-intl/models \
  -H "Content-Type: application/json" \
  -d '{"id":"qwen3.8-max","name":"Qwen3.8 Max","supports_image":false,"supports_video":false}'

# params (context per vendor docs)
curl -s -X PUT $API/models/aliyun-tokenplan-intl/models/qwen3.8-max/config \
  -H "Content-Type: application/json" \
  -d '{"max_input_length":1000000,"max_tokens":8192,"relay_reasoning":true}'

# live test
curl -s -X POST $API/models/aliyun-tokenplan-intl/models/test \
  -H "Content-Type: application/json" -d '{"model_id":"qwen3.8-max"}'
```

Models added this way in this setup: **qwen3.8-max**, **glm-5.3** (Token Plan /
Z.AI catalogs), **kimi-k3** (Kimi catalog — 1M ctx per platform.kimi.ai).

> Real-world catch: the Token Plan relay's live `/models` list (11 ids) has drifted
> from its published catalog — e.g. `MiniMax-M3` / `kimi-k3` are **not** routable
> there ("Model not exist"), and even `MiniMax-M2.5` is plan-denied. That's why M3
> and K3 live on their dedicated MiniMax / Kimi providers with their own keys.

---

## 8. Per-model parameters (vendor-doc-verified)

### How QwenPaw resolves a model's context window

`Provider.get_context_size()` → `resolve_context_window()` with precedence
(see `src/qwenpaw/providers/context_windows.py`):

1. **explicit per-model `max_input_length`** (set via API/UI — wins outright)
2. **static vendor-docs catalog** (`context_windows.py`, pattern-matched)
3. **default 128k**

⚠️ Setting an explicit value **pins** it and overrides future catalog updates —
only set explicit values for models the catalog gets wrong or doesn't know.

### Doc sources used

| Vendor | Source | Facts extracted |
|---|---|---|
| Z.AI | `docs.z.ai/guides/llm/*.md`, `/guides/vlm/*.md` | per-model Context Length / Max Output cards |
| DeepSeek | `api-docs.deepseek.com/quick_start/pricing` | v4: 1M ctx / 384K max out; thinking default |
| Kimi | `platform.kimi.ai/docs/...` | K2.x: 256K; K3: 1M (1,048,576) |
| MiniMax | HF `MiniMaxAI/MiniMax-M2.5` / `-M3` `config.json` | `max_position_embeddings` = 196,608 / 1,048,576 |
| Aliyun | `alibabacloud.com/help/en/model-studio/qwen-flash` | Qwen Flash tier: 1M ctx |

### Final configured matrix (as deployed)

All models: `max_tokens 8192` (agent-safe; doc'd *caps* are higher — GLM-5.x 128K
out, DeepSeek V4 384K out), `relay_reasoning true`, thinking budget 1–81920,
reasoning effort options none→xhigh, sampling params at vendor defaults.

**aliyun-tokenplan-intl — Aliyun Token Plan (International)**

| Model | Context | Source of truth | Capabilities |
|---|---:|---|---|
| qwen3.7-plus | 1,000,000 | catalog | image+video |
| qwen3.7-max | 1,000,000 | catalog | text |
| qwen3.6-plus | 1,000,000 | catalog | image+video |
| qwen3.6-flash | 1,000,000 | explicit (Aliyun doc) | image+video |
| qwen3.8-max | 1,000,000 | explicit (new, upstream-verified) | text |
| deepseek-v4-pro | 1,000,000 | explicit (DeepSeek docs) | text |
| deepseek-v4-flash | 1,000,000 | explicit (DeepSeek docs) | text |
| deepseek-v3.2 | 131,072 | default (V3 line = 128K) | text |
| glm-5.2 | 1,000,000 | catalog (Z.AI doc card) | text |
| glm-5.1 | 200,000 | explicit (Z.AI doc card) | text |
| glm-5 | 200,000 | explicit (Z.AI doc card) | text |
| MiniMax-M2.5 | 196,608 | explicit (HF config.json) | text |
| kimi-k2.6 | 262,144 | catalog (Kimi docs: 256K) | image+video |
| kimi-k2.5 | 262,144 | catalog (Kimi docs: 256K) | image+video |

**zhipu-intl-codingplan — Zhipu Coding Plan (Z.AI)**

| Model | Context | Source | Capabilities |
|---|---:|---|---|
| glm-5.3 | 1,000,000 | explicit (doc card: 1M / 128K out) | text |
| glm-5.2 | 1,000,000 | catalog | text |
| glm-5.1 | 200,000 | explicit (doc card) | text |
| glm-5 | 200,000 | explicit (doc card) | text |
| glm-5-turbo | 200,000 | explicit (doc card) | text |
| glm-5v-turbo | 200,000 | explicit (doc card) | image |
| glm-4.7-flash | 200,000 | explicit (4.7 family doc) | text, free |

**minimax — MiniMax (International)** (pay-as-you-go key set)

| Model | Context | Source |
|---|---:|---|
| MiniMax-M3 | 1,048,576 | explicit (HF config + catalog) |
| MiniMax-M2.7 | 204,800 | explicit (catalog table) |
| MiniMax-M2.5 | 196,608 | explicit (HF config) |
| M2.7/M2.5/M2.1/M2 highspeed + M2 | defaults | conservative |

**deepseek — DeepSeek** (pay-as-you-go key set)

| Model | Context | Source |
|---|---:|---|
| deepseek-v4-pro | 1,000,000 | explicit (pricing table) |
| deepseek-v4-flash | 1,000,000 | explicit (pricing table) |
| deepseek-v4-flash-vision-exp | 1,000,000 | catalog edit (live `/models` + pricing table) |

*`deepseek-chat` / `deepseek-reasoner` (V3.x) were retired by the vendor — confirmed
via live `GET api.deepseek.com/models`, which now serves only the three v4 models —
and removed from the catalog (§12).*

**kimi-intl — Kimi (International)** (key from platform.kimi.com required)

| Model | Context | Source |
|---|---:|---|
| kimi-k3 | 1,048,576 | explicit (pricing page: "1,048,576 tokens") |
| kimi-k2.5 | 262,144 | catalog (256K docs) |

### Generic per-model config command

```bash
curl -s -X PUT $API/models/<provider>/models/<model>/config \
  -H "Content-Type: application/json" \
  -d '{
        "max_tokens": 8192,
        "max_input_length": 1000000,
        "relay_reasoning": true,
        "thinking_enabled": null,
        "thinking_budget": null,
        "reasoning_effort": null,
        "generate_kwargs": {}
      }'
```

Notes:
- `thinking_*`: leave `null` (auto) unless a model needs explicit control.
- `generate_kwargs` **replaces** the model's dict — include existing keys in the
  same payload when updating (deep-merged over provider-level kwargs).
- **top_p / temperature**: deliberately **unset** (vendor-recommended). Unset means
  *omitted from the request* → the API's server default applies. Qwen/DashScope
  docs explicitly advise against pinning `top_p` for chat/agentic use.

---

## 9. How parameters flow to the vendor API

Verified in code (`src/qwenpaw/providers/provider.py`,
`src/qwenpaw/providers/openai_provider.py`) — **nothing is dropped**:

```
provider.generate_kwargs ──deep-merge──> model.generate_kwargs   (model wins)
        │ get_effective_generate_kwargs()
        ▼
┌─ first-class request params ──────────────────────────────┐
│  max_tokens (→ max_completion_tokens for o-style models)  │
│  temperature, top_p                                       │
├─ forwarded VERBATIM (extra_generate_kwargs) ──────────────┤
│  any vendor JSON param: enable_thinking, top_k,           │
│  presence_penalty, response_format, …                     │
├─ behavioral ──────────────────────────────────────────────┤
│  context_size  → compaction trigger + UI usage%           │
│  relay_reasoning → reasoning-stream relay in formatter    │
└───────────────────────────────────────────────────────────┘
```

Serialization check (agentscope `OpenAIChatModel.Parameters`):
`None` top_p/temperature are **omitted** from the payload; a set value (e.g.
`0.95`) lands first-class. Demo:

```
Parameters(top_p=None)      -> {max_tokens, thinking_enable, parallel_tool_calls}
Parameters(top_p=0.95)      -> … + top_p: 0.95
extra_generate_kwargs       -> every remaining key, unchanged
```

---

## 10. Verification & troubleshooting

```bash
# Full install health check (working dir, console statics, providers, API, agents)
/home/<user>/.venvs/qwenpaw/bin/qwenpaw doctor

# Per-model live generation
curl -s -X POST $API/models/<provider>/models/test \
  -H "Content-Type: application/json" -d '{"model_id":"<model>"}'

# App up?
curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:8088/    # 200
curl -s http://127.0.0.1:8088/api/healthz
```

| Symptom | Cause / fix |
|---|---|
| `venv/bin/python: No such file` | repo on network mount corrupts symlinks → venv on local disk (§1) |
| `init` aborts: "Input is not a terminal" | add `--accept-security` (§4) |
| `qwen3.8-max`-style model "not exist" | model not routable on that relay — probe upstream first (§7) |
| Effective ctx wrong in UI | explicit `max_input_length` overrides the catalog — clear/fix via per-model config |
| Provider test fails after key set | key prefix validation (`api_key_prefix`) or wrong provider variant (CN vs Intl) |

---

## 11. Maintenance

```bash
# Update to a newer revision of the branch:
git pull
cd console && npm ci && npm run build && cd ..
cp -R console/dist/. src/qwenpaw/console/
# editable install: no reinstall needed, just restart
kill %1  # or: pkill -f "qwenpaw app"
nohup /home/<user>/.venvs/qwenpaw/bin/qwenpaw app > /tmp/qwenpaw-app.log 2>&1 &
# hard-refresh browser (Ctrl+Shift+R)
```

- Rebuild console without reinstall: `qwenpaw doctor fix -y --only rebuild-console-npm`
- Reset config/data: `qwenpaw clean` (working dir) · `qwenpaw uninstall` (keeps
  config) · `qwenpaw uninstall --purge` (everything)
- Keys & provider configs: `~/.qwenpaw.secret/providers/**.json` (encrypted)
- Working dir: `~/.qwenpaw/` (config.json, workspaces, skills, logs)

---

## 12. Removing built-in providers & vendor-retired models

QwenPaw has **no UI/API to disable a built-in provider or delete a catalog model**
(`DELETE …/models/{id}` only removes *user-added* `extra_models` —
`Provider.delete_model` filters `extra_models` only). Since this is an editable
source install, edit `src/qwenpaw/providers/provider_manager.py` and restart:

```bash
# 1) Remove a built-in provider (e.g. OpenCode, Kilo Code):
#    delete its _add_builtin(...) line in ProviderManager._init_builtins()

# 2) Remove a retired model (e.g. deepseek-chat / deepseek-reasoner)
#    or add a newly released one (e.g. deepseek-v4-flash-vision-exp):
#    edit the provider's <NAME>_MODELS list (e.g. DEEPSEEK_MODELS)

# 3) Restart (port-based kill avoids pkill self-matching the start command):
fuser -k 8088/tcp; sleep 3
nohup /home/<user>/.venvs/qwenpaw/bin/qwenpaw app > /tmp/qwenpaw-app.log 2>&1 &

# 4) Verify: provider gone, models correct, keys intact
curl -s http://127.0.0.1:8088/api/models | /home/<user>/.venvs/qwenpaw/bin/python -c \
  "import json,sys; [print(p['id'], len(p['models'])) for p in json.load(sys.stdin)]"
```

> ⚠️ `pkill -f "qwenpaw app"` kills your own terminal (the compound command or
> wrapper cmdline contains the same string) — kill by port with `fuser -k` instead.

Applied in this setup: **OpenCode + Kilo Code** providers removed; **deepseek-chat /
deepseek-reasoner** removed (retired upstream, verified); **deepseek-v4-flash-vision-exp**
added to the catalog (1M ctx, image input, live-tested). User API keys survive all of
this (stored encrypted, re-merged into the new catalog on load).
