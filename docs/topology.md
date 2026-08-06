# Topology

## Logical path

```
┌─────────────────┐     ┌─────────────────┐
│ Conduit (phone) │     │ Browser (PC)    │
└────────┬────────┘     └────────┬────────┘
         │  Open WebUI API       │
         └───────────┬───────────┘
                     ▼
              ┌─────────────┐
              │ Open WebUI  │
              └──────┬──────┘
                     │ OpenAI-compatible
                     │ base = http://filter:9099/<port>/v1
                     ▼
         ┌───────────────────────┐
         │ hermes_tool_filter    │
         │ enhance-v2 +          │
         │ structured history    │
         └───────────┬───────────┘
                     │ /<port>/v1/* → 127.0.0.1:<port>
                     ▼
         ┌───────────────────────┐
         │ Hermes Gateway        │
         │ api_server (profile)  │
         │ optional patches ★    │
         └───────────┬───────────┘
                     │ model.base_url
                     ▼
         ┌───────────────────────┐
         │ Model endpoint        │
         │ (vLLM / cloud / …)    │
         └───────────────────────┘
```

★ Patches in this repo apply on the Hermes git tree, not on Open WebUI.

## Multi-profile example

One filter process, many gateway ports:

| Open WebUI connection URL | Filter rewrite | Gateway |
|---------------------------|----------------|---------|
| `…:9099/30001/v1` | → `http://127.0.0.1:30001` | coder |
| `…:9099/30002/v1` | → `http://127.0.0.1:30002` | coder-master |
| `…:9099/30005/v1` | → `http://127.0.0.1:30005` | family |

Each gateway profile has its own `API_SERVER_PORT` + `API_SERVER_KEY`.  
Open WebUI stores one OpenAI connection per URL+key pair.

## Request / response roles

### Chat turn (user message)

```
Client → Open WebUI → filter (sanitize history) → Gateway → model
```

Filter inbound job: turn stored `<details type="tool_calls">` HTML into
`assistant.tool_calls` + `role=tool` before Gateway sees the body.

### Tool execution (same HTTP stream)

```
Gateway runs tool
  → SSE hermes.tool.progress {running|completed}
  → filter injects <details> tool card into delta.content
  → Open WebUI renders card + persists assistant HTML
```

Progress payloads must stay small (no multi-MB base64).  
Model-facing multimodal tool results are separate (agent loop).

## Where Conduit sits

Conduit is **only** an Open WebUI client:

```
Conduit ──HTTPS──▶ Open WebUI ──▶ filter ──▶ Hermes
```

It does not speak Hermes native protocols.  
If tool cards look wrong on phone but OK in browser, debug Open WebUI rendering /
Conduit first; if **both** break next-turn memory, debug filter + patches.

## Optional lab backend chain

Not part of the Open WebUI glue; shown for completeness:

```
Hermes model.base_url
    → metrics_proxy (:18080)   # metrics + optional image resize
        → llama-swap (:8090)   # model router
            → vLLM
```

Image resize here protects **backend VRAM**.  
Progress SSE redact (Hermes patch) protects **filter/Open WebUI streams**.

## Trust boundary notes

- Prefer filter and gateways on `127.0.0.1`; expose only Open WebUI (or a reverse proxy) to LAN/WAN.
- Phone over the internet: terminate TLS in front of Open WebUI; keep `API_SERVER_KEY` out of client apps that are not Open WebUI.
- Filter `bind_host: 0.0.0.0` is convenient on a homelab host; firewall accordingly.
