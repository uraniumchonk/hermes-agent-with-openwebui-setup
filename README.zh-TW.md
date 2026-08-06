# Hermes Agent with Open WebUI Setup

把 **[Hermes Agent](https://github.com/NousResearch/hermes-agent)** 接在
**[Open WebUI](https://github.com/open-webui/open-webui)** 後面用的
**連接鏈路 + 可選 patch**。

客戶端：電腦瀏覽器，或手機 **[Conduit](https://github.com/cogwheel0/conduit)**。

[English](README.md) · 繁體中文

> 本 repo **只放文件與 patch**。  
> 執行本體請裝各上游專案（以及 tool filter adapter）。

---

## 你會拿到什麼

| 內容 | 用途 |
|------|------|
| **拓樸** | 手機 / 瀏覽器怎麼經 OWUI 打到 Hermes |
| **痛點** | 沒膠合層時壞在哪 |
| **Hermes patch** | `api_server` / `prompt_builder` |
| **Open WebUI patch** | 給 Hermes 的 path 注入 + follow-up（0.10.2） |
| **連結** | 真正要裝的上游 |

不含 Hermes / OWUI / filter 完整原始碼。

---

## 連接鏈路

```
手機 Conduit   ─┐
                ├─→  Open WebUI  ─→  hermes_tool_filter :9099/<port>/v1
電腦瀏覽器     ─┘      │ ★ OWUI patch          │
                       │                       ▼
                       │              Hermes Gateway
                       │              （★ Hermes patch）
                       │                       │
                       │                       ▼
                       │              OpenAI-compatible 模型
```

```
Open WebUI Base URL = http://<host>:9099/<gateway_port>/v1
例：http://127.0.0.1:9099/30001/v1 → gateway :30001
```

可選本機後端：`Hermes → metrics_proxy → llama-swap → vLLM`（壓圖／路由，非膠合必要）。

詳見：[docs/topology.md](docs/topology.md)

---

## 客戶端

### 瀏覽器

Open WebUI → 連線 → OpenAI 相容 → URL `http://…:9099/<port>/v1`，Key = Hermes `API_SERVER_KEY`。

### 手機 Conduit

指到**同一個 Open WebUI**（WAN 用 HTTPS）。不要直連 Hermes / filter。  
路徑：Conduit → OWUI → filter → Hermes。

---

## 痛點一覽

| # | 現象 | 層 |
|---|------|-----|
| 1 | 下一輪 tool 失憶 / 學 `<details>` | **filter** |
| 2 | filter 送了 `role=tool` 被 Gateway 丟 | **Hermes** api_server patch |
| 3 | 多圖 vision、MB 級 progress SSE 斷線 | **Hermes** progress redact |
| 4 | api_server 禁 markdown，card 怪 | **Hermes** prompt_builder patch |
| 5 | 上傳變巨大 base64，agent 沒 path | **OWUI** middleware patch |
| 6 | 非圖附件重載後消失 | **OWUI** middleware patch |
| 7 | Follow-up 建議空白 | **OWUI** misc patch |

filter 的下一輪 payload 對照見英文 README / adapter repo。

---

## 上游元件

| 元件 | Repo |
|------|------|
| Hermes | https://github.com/NousResearch/hermes-agent |
| Open WebUI | https://github.com/open-webui/open-webui |
| Conduit | https://github.com/cogwheel0/conduit |
| Tool filter | https://github.com/uraniumchonk/hermes-open-webui-adapter |
| 本 repo | 拓樸 + patch |

---

## 快速串線

### Hermes patch

```bash
cd /path/to/hermes-agent
git apply …/patches/hermes/api_server_chat_completions_all.patch
git apply …/patches/hermes/prompt_builder_api_server_hint.patch
```

→ [docs/patches.md](docs/patches.md)

### Open WebUI patch（0.10.2 黃金副本）

```bash
SP="<venv>/lib/python3.12/site-packages/open_webui/utils"
cp patches/openwebui/middleware.patched.0.10.2.py "$SP/middleware.py"
cp patches/openwebui/misc.patched.0.10.2.py       "$SP/misc.py"
# 啟動務必 export DATA_DIR=…
```

→ [docs/openwebui-patches.md](docs/openwebui-patches.md)

### Filter

https://github.com/uraniumchonk/hermes-open-webui-adapter  
範例：[config-examples/filter-upstreams.example.yaml](config-examples/filter-upstreams.example.yaml)

OWUI 請指 filter，不要裸指 gateway（若要 card + 歷史改寫）。

---

## 必要 vs 可選

| 層 | 基本聊天 | OWUI 工具體驗佳 |
|----|----------|----------------|
| Hermes / OWUI | 要 | 要 |
| Tool filter | 否 | **要** |
| Hermes patch | 否 | **強烈建議** |
| OWUI patch | 否 | **強烈建議**（path / 附件） |
| Conduit | 否 | 僅手機 |

---

## 目錄

```
patches/hermes/
patches/openwebui/          # 0.10.2 patch + official/patched 黃金副本
docs/topology.md
docs/patches.md             # Hermes
docs/openwebui-patches.md
config-examples/
```

---

## License

MIT（本 repo）。上游各自授權。
