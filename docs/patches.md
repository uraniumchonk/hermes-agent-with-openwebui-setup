# Optional Hermes patches

Target tree: a git checkout of [Hermes Agent](https://github.com/NousResearch/hermes-agent)
(commonly `~/.hermes/hermes-agent`).

These patches are **optional personal improvements** for Open WebUI + tool-filter
setups. The filter can run without them; tool history and large vision streams
work much better with them.

Patch files: [`../patches/hermes/`](../patches/hermes/)

Open WebUI-side patches (separate): [`openwebui-patches.md`](openwebui-patches.md)

---

## 1. `api_server_chat_completions_all.patch`

**File:** `gateway/platforms/api_server.py`

### Why

| Issue | Without patch | With patch |
|-------|---------------|------------|
| Structured tool history from filter | `role=tool` / `tool_calls` dropped | passed through |
| Primary user message | often `messages[-1]` (can be tool/assistant) | last **user** message |
| `hermes.tool.progress` completed | may omit args/result or dump full multimodal base64 | args + redacted result |
| Parallel `vision_analyze` | multi-MB SSE → proxy/OWUI drop | progress keeps `text_summary` only |

### What it does (summary)

1. Accept Chat Completions history with `role=tool` and `assistant.tool_calls`.
2. Emit richer `hermes.tool.progress` for the filter (`arguments` on running/completed, `result` on completed).
3. **Redact** multimodal / `data:image…base64` from progress SSE via
   `_redact_progress_result` / `_redact_progress_arguments`.  
   Full tool result still goes to the agent loop for the model.

### Apply

```bash
cd /path/to/hermes-agent
git apply /path/to/this-repo/patches/hermes/api_server_chat_completions_all.patch
```

### Verify

```bash
grep -c '_redact_progress_result' gateway/platforms/api_server.py      # >0
grep -c '_redact_progress_arguments' gateway/platforms/api_server.py   # >0
grep -c 'OpenAI-native tool result' gateway/platforms/api_server.py    # >0
```

---

## 2. `prompt_builder_api_server_hint.patch`

**File:** `agent/prompt_builder.py`

### Why

Stock `PLATFORM_HINTS["api_server"]` tells the model to assume plain text and
avoid markdown. Open WebUI + tool cards want markdown.

### What it does

Replace plain-text restriction with: **Markdown is allowed** (keeps MEDIA path notes).

### Apply

```bash
cd /path/to/hermes-agent
git apply /path/to/this-repo/patches/hermes/prompt_builder_api_server_hint.patch
```

### Verify

```bash
grep -c 'Markdown is allowed' agent/prompt_builder.py   # >0
grep -c 'assume plain text' agent/prompt_builder.py     # 0
```

---

## After `hermes update`

Hermes updates often rewrite the tree. Re-apply patches; if `git apply` fails on
hunks, open the `.patch` and port by hand — treat line numbers as hints.

If your working tree already has the intended edits:

```bash
cd /path/to/hermes-agent
git diff HEAD -- gateway/platforms/api_server.py \
  > /path/to/this-repo/patches/hermes/api_server_chat_completions_all.patch
git diff HEAD -- agent/prompt_builder.py \
  > /path/to/this-repo/patches/hermes/prompt_builder_api_server_hint.patch
```

---

## What we deliberately do not patch in the Hermes tree

| Area | Why |
|------|-----|
| Open WebUI core | see [`openwebui-patches.md`](openwebui-patches.md) |
| Tool filter runtime | own repo: [hermes-open-webui-adapter](https://github.com/uraniumchonk/hermes-open-webui-adapter) |
| vLLM / llama-swap | backend ops, not OWUI glue |
