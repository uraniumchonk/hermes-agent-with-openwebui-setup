# Hermes Agent with Open WebUI Setup

Connection topology + optional patches for using
**[Hermes Agent](https://github.com/NousResearch/hermes-agent)** behind
**[Open WebUI](https://github.com/open-webui/open-webui)**
(browser and/or **[Conduit](https://github.com/cogwheel0/conduit)** mobile app).

English · [繁體中文](README.zh-TW.md)

> This repo is **docs + patches only**.  
> Runtime code lives in the upstream projects (and the optional filter adapter below).

---

## What you get

| Piece | Role |
|-------|------|
| **Topology** | How phone / browser reach Hermes through Open WebUI |
| **Pain points** | What breaks without the glue layer |
| **Hermes patches** | `api_server` / `prompt_builder` diffs |
| **Open WebUI patches** | path injection for Hermes tools + follow-up fix (0.10.2) |
| **Links** | Upstream repos you actually install |

Not included: full Hermes / Open WebUI / filter source trees.

---

## Connection chain

Minimal production path:

```
Phone (Conduit)  ─┐
                  ├─→  Open WebUI  ─→  hermes_tool_filter :9099/<port>/v1
Desktop browser  ─┘         │                  │
                            │ ★ OWUI patches   ▼
                            │         Hermes Gateway api_server
                            │         (★ Hermes patches)
                            │                  │
                            │                  ▼
                            │         OpenAI-compatible model
```

Path-prefix routing on the filter:

```
Open WebUI base URL = http://<host>:9099/<gateway_port>/v1

Examples:
  http://127.0.0.1:9099/30001/v1   → Hermes coder gateway :30001
  http://127.0.0.1:9099/30005/v1   → Hermes family gateway :30005
```

Optional lab backend (not required for the glue):

```
Hermes model.base_url → metrics_proxy :18080 → llama-swap → vLLM
```

Details: [docs/topology.md](docs/topology.md)

---

## Clients

### Desktop browser

1. Open WebUI in the browser.
2. Admin → Connections → OpenAI-compatible:
   - **URL**: `http://127.0.0.1:9099/<port>/v1`
   - **Key**: Hermes `API_SERVER_KEY`
3. Chat with the listed model/profile.

### Phone — Conduit

[Conduit](https://github.com/cogwheel0/conduit) is an Open WebUI mobile client.

1. Point Conduit at the **same Open WebUI** URL as the browser (HTTPS on WAN).
2. Do **not** point Conduit at Hermes or the filter directly.
3. Path stays: Conduit → Open WebUI → filter → Hermes.

---

## Pain points

### A. Open WebUI ↔ Hermes tools

| # | Symptom | Layer |
|---|---------|--------|
| 1 | Next turn forgets tools / mimics `<details>` | **filter** ([adapter](https://github.com/uraniumchonk/hermes-open-webui-adapter)) |
| 2 | Filter sends `role=tool` but Gateway drops it | **Hermes** `api_server` patch |
| 3 | Parallel vision / multi-MB progress SSE drops OWUI | **Hermes** progress redact in same patch |
| 4 | api_server forces plain text; cards look wrong | **Hermes** `prompt_builder` patch |
| 5 | Uploads become huge base64; agent has no file path | **Open WebUI** middleware patch |
| 6 | Non-image attachments disappear on reload | **Open WebUI** middleware patch |
| 7 | Follow-up suggestions empty | **Open WebUI** misc patch |

### B. Real next-turn payload (filter)

Open WebUI stores HTML tool cards inside assistant text. Without the filter the
model sees that string again. With the filter (`structured`):

```json
[
  {
    "role": "assistant",
    "content": "Let me check.",
    "tool_calls": [{
      "id": "call_htf_a1b2",
      "type": "function",
      "function": { "name": "web_search", "arguments": "{\"query\": \"BTC price\"}" }
    }]
  },
  {
    "role": "tool",
    "tool_call_id": "call_htf_a1b2",
    "name": "web_search",
    "content": "{\"price\": 64000}"
  },
  { "role": "assistant", "content": "About 64000." }
]
```

---

## Components (install upstream yourself)

| Component | Repo |
|-----------|------|
| Hermes Agent | https://github.com/NousResearch/hermes-agent |
| Open WebUI | https://github.com/open-webui/open-webui |
| Conduit | https://github.com/cogwheel0/conduit |
| Tool filter | https://github.com/uraniumchonk/hermes-open-webui-adapter |
| This repo | topology + patches only |

---

## Quick wiring

### 1. Hermes gateway

```bash
API_SERVER_ENABLED=true
API_SERVER_HOST=127.0.0.1
API_SERVER_PORT=30001
API_SERVER_KEY=your-secret
```

### 2. Optional Hermes patches

```bash
cd /path/to/hermes-agent
git apply /path/to/this-repo/patches/hermes/api_server_chat_completions_all.patch
git apply /path/to/this-repo/patches/hermes/prompt_builder_api_server_hint.patch
# restart gateway
```

Docs: [docs/patches.md](docs/patches.md) (= Hermes)

### 3. Optional Open WebUI patches (0.10.2 goldens)

```bash
SP="<venv>/lib/python3.12/site-packages/open_webui/utils"
cp patches/openwebui/middleware.patched.0.10.2.py "$SP/middleware.py"
cp patches/openwebui/misc.patched.0.10.2.py       "$SP/misc.py"
# restart Open WebUI; DATA_DIR must be set on start
```

Docs: [docs/openwebui-patches.md](docs/openwebui-patches.md)

### 4. Tool filter

https://github.com/uraniumchonk/hermes-open-webui-adapter  

Example: [config-examples/filter-upstreams.example.yaml](config-examples/filter-upstreams.example.yaml)

Open WebUI base URL: `http://<filter-host>:9099/<port>/v1` — **not** raw gateway if you want cards + history rewrite.

### 5. Conduit

Same Open WebUI origin as the browser.

---

## Required vs optional

| Layer | Basic chat | Good Hermes tools in OWUI |
|-------|------------|---------------------------|
| Hermes api_server | yes | yes |
| Open WebUI | yes | yes |
| Tool filter | no | **yes** |
| Hermes patches | no | **strongly recommended** |
| Open WebUI patches | no | **strongly recommended** (paths / attachments) |
| Conduit | no | phone only |

---

## Repo layout

```
patches/hermes/          # git apply onto hermes-agent
patches/openwebui/       # copy onto open-webui site-packages (0.10.2)
docs/topology.md
docs/patches.md          # Hermes
docs/openwebui-patches.md
config-examples/
README.md
README.zh-TW.md
LICENSE
```

---

## License

MIT (docs and patches in this repository).  
Upstream projects keep their own licenses.
