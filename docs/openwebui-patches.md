# Optional Open WebUI patches (for Hermes)

Open WebUI is usually installed via **pip/venv**, not as a git checkout.
Upgrading `open-webui` overwrites `site-packages` — re-apply after every upgrade.

These patches make Open WebUI play nicer with **Hermes Agent** + path-based tools
(`vision_analyze`, `read_file`) instead of stuffing multi-MB base64 into chat.

**Target version in this tree:** Open WebUI **0.10.2** (Python 3.12).

---

## Files

| File | Role |
|------|------|
| `middleware_custom_0.10.2.patch` | unified diff vs official 0.10.2 |
| `middleware.official.0.10.2.py` | stock golden |
| `middleware.patched.0.10.2.py` | patched golden (preferred reapply) |
| `misc_output_fallback_0.10.2.patch` | unified diff |
| `misc.official.0.10.2.py` | stock golden |
| `misc.patched.0.10.2.py` | patched golden |

Live paths (typical venv):

```text
<venv>/lib/python3.12/site-packages/open_webui/utils/middleware.py
<venv>/lib/python3.12/site-packages/open_webui/utils/misc.py
```

---

## Pain points → fixes

### 1. Images as huge base64 in the prompt

Stock OWUI often embeds upload images as data URLs. Hermes then either
chokes on size or never gets a stable filesystem path for `vision_analyze`.

**Patch:** inject **canonical upload paths** under OWUI `data/uploads/`, and
tell the agent to call `vision_analyze(path=...)`.  
Ephemeral `data:image` blobs may still land in a small `image_cache/` as
`data_<md5>.*` only.

### 2. Non-image attachments vanish

On reload, OWUI may keep only images in content and `pop('files')`, dropping
documents before the model sees them.

**Patch:** harvest non-image files and inject paths + `read_file(path=...)`
**before** RAG, after `metadata.files` is finalized.

### 3. Follow-up suggestions empty

Assistant text sometimes lives in `output` / `output_text` while
`get_content_from_message()` only reads `content`.

**Patch (`misc`):** fallback extract from `output` when `content` is empty.

### 4. Follow-up restamp breaks prompt cache

Re-timestamping “last message with image” on every follow-up busts prefix cache.

**Patch:** fixed inject timestamps per message+path (`_get_or_create_inject_ts`).

---

## Apply (0.10.2) — recommended: copy goldens

```bash
SP="<venv>/lib/python3.12/site-packages/open_webui/utils"
PATCH_DIR="/path/to/this-repo/patches/openwebui"

cp "$PATCH_DIR/middleware.patched.0.10.2.py" "$SP/middleware.py"
cp "$PATCH_DIR/misc.patched.0.10.2.py"       "$SP/misc.py"

# ensure data dirs exist
mkdir -p "$DATA_DIR/uploads" "$DATA_DIR/image_cache"

# restart Open WebUI
```

### Alternate: `patch` against clean official files

```bash
cd "$SP"
cp middleware.py middleware.py.bak
patch -p0 < "$PATCH_DIR/middleware_custom_0.10.2.patch"   # may need path massage
```

If hunks fail, use golden copy method.

---

## Verify

```bash
MW="$SP/middleware.py"
MISC="$SP/misc.py"

grep -c 'async def inject_image_file_paths' "$MW"     # 1
grep -c 'async def inject_document_file_paths' "$MW"  # 1
grep -c '_collect_request_file_items' "$MW"           # >=1
grep -c 'BEFORE native RAG' "$MW"                     # 1
grep -c 'Fallback: extract from output field' "$MISC" # 1

md5sum "$MW" "$PATCH_DIR/middleware.patched.0.10.2.py"
md5sum "$MISC" "$PATCH_DIR/misc.patched.0.10.2.py"
```

---

## Dangerous pit: `DATA_DIR`

Never `import open_webui` without `DATA_DIR` set in a throwaway env — it can
create an empty `webui.db` next to site-packages and later migrations can
**wipe the real DB**.

Start script must export something like:

```bash
export DATA_DIR=/path/to/openwebui/data
```

---

## Version skew

If you are not on 0.10.2:

1. Backup live `middleware.py` / `misc.py`
2. Upgrade Open WebUI
3. Port functions/hooks from the patch by hand into new `process_chat_payload` /
   `get_content_from_message`
4. Regenerate goldens + diffs

---

## Out of scope here

| Item | Where |
|------|--------|
| Hermes gateway patches | [`../hermes/`](../hermes/) |
| SSE tool-card proxy | https://github.com/uraniumchonk/hermes-open-webui-adapter |
| Homelab rsync/GC scripts | operator-specific; not required to understand the patch |
