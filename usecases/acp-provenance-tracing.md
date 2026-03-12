# ACP Provenance 追蹤：讓 Agent 知道請求從哪裡來

## 使用場景

當 Codex、Claude Code 或其他 ACP 客戶端透過 `openclaw acp` bridge 向 OpenClaw Agent 發送指令時，Agent 預設無法區分這條 prompt 是來自「真實使用者」還是「另一個自動化程序」。啟用 `--provenance` 後，bridge 會在每條 ingress 訊息附上 ACP 來源資訊，讓 Agent 能正確判斷請求的可信度與上下文。

適用情境：
- Codex / Claude Code 透過 ACP bridge 呼叫 OpenClaw Agent 執行子任務
- 多 Agent 協作鏈中，需要追蹤每條指令的來源 session
- 審計或 debug：區分哪些操作是人發起的、哪些是自動化流程發起的

## 功能依賴

- OpenClaw v2026.3.8+（[#40473](https://github.com/openclaw/openclaw/pull/40473)）
- `openclaw acp --provenance` 旗標
- Gateway 正常運行（`openclaw gateway status`）

---

## Provenance 三種模式

| 模式 | 行為 |
|------|------|
| `off` | 不注入任何來源資訊（預設） |
| `meta` | 注入 ACP origin metadata 至 Gateway ingress（kind、session ID、來源 channel/tool） |
| `meta+receipt` | `meta` 基礎上，額外插入可讀的 `[Source Receipt]` block 至 Agent 的對話上下文 |

---

## 實際原始碼驗證（v2026.3.8）

### `--provenance meta` 注入的 InputProvenance

```typescript
// 來源：dist/acp-cli-BVqlbdKB.js — buildSystemInputProvenance()
{
  kind: "external_user",
  originSessionId: "<ACP session UUID>",
  sourceChannel: "acp",
  sourceTool: "openclaw_acp"
}
```

這個 provenance 物件跟隨 `chat.send` 進入 Gateway，作為 ingress 訊息的 metadata，讓 transcript 層能區分「ACP 客戶端發的」和「真實使用者發的」。

### `--provenance meta+receipt` 額外注入的 Receipt Block

```typescript
// 來源：dist/acp-cli-BVqlbdKB.js — buildSystemProvenanceReceipt()
[Source Receipt]
bridge=openclaw-acp
originHost=<hostname>
originCwd=<工作目錄（~ 已縮寫）>
acpSessionId=<ACP session UUID>
originSessionId=<ACP session UUID>
targetSession=<gateway session key>
[/Source Receipt]
```

這個純文字 block 會插入到 Agent 的對話上下文裡，Agent **可以直接讀到**，並據此調整行為。

### 本機實際執行記錄（Linux VPS, v2026.3.8）

啟動 ACP client：

```bash
openclaw acp --provenance meta+receipt client
```

```
[acp-client] spawning: /usr/bin/node /usr/bin/openclaw acp
[acp-client] initializing
[acp-client] creating session
OpenClaw ACP client
Session: a81ecee9-4734-4044-b053-6d64f84df399
Type a prompt, or "exit" to quit.
>
```

Session UUID（`a81ecee9-...`）即為注入至 receipt block 的 `acpSessionId`。

---

## 核心區別：meta vs meta+receipt

| | `meta` | `meta+receipt` |
|---|---|---|
| Gateway 內部可見 | ✅ | ✅ |
| Agent 上下文可見（可讀） | ❌ | ✅ |
| 用途 | 審計、transcript 區分 | Agent 主動感知並調整行為 |

`meta` 是 gateway 層的 metadata，不進入 Agent 的 LLM 上下文。
`meta+receipt` 才是讓 Agent「知道自己被誰呼叫」的方式。

---

## 讓 Agent 根據 Receipt 調整行為

在 `AGENTS.md` 加入規則，讓 Agent 偵測 `[Source Receipt]` block：

```markdown
## ACP 請求處理規則

若對話上下文開頭包含 [Source Receipt] ... [/Source Receipt] block，
表示此請求來自 ACP bridge（Codex、Claude Code 等自動化客戶端）：
- 直接執行具體指令，不需向使用者二次確認
- 敏感操作（刪除、部署）仍需等待人工審批
- 完成後回報 acpSessionId，方便呼叫方追蹤
- 不需要發送 Telegram 通知（避免重複打擾）
```

---

## 與 acpx 整合（Codex / Claude Code）

若透過 `acpx` 讓 Codex 或 Claude Code 呼叫 OpenClaw，在 `~/.acpx/config.json` 固定開啟 provenance：

```json
{
  "agents": {
    "openclaw": {
      "command": "env OPENCLAW_HIDE_BANNER=1 OPENCLAW_SUPPRESS_NOTES=1 openclaw acp --provenance meta+receipt --session agent:main:main"
    }
  }
}
```

執行：

```bash
acpx openclaw exec "整理 /tmp 目錄並回報結果"
```

Agent 收到的上下文裡會包含完整的 `[Source Receipt]` block，可追蹤到是哪台機器、哪個工作目錄發起的請求。

---

## 什麼時候不需要開

- **純互動使用**（只有人在打字）：不需要，`off` 即可
- **只需要 Gateway 層審計，不需要 Agent 感知**：用 `meta` 即可，開銷更小
- **全自動 Multi-Agent 鏈**：建議 `meta+receipt`，Agent 能做來源判斷

---

## 延伸閱讀

- `openclaw acp --help`
- [ACP Bridge 文件](https://docs.openclaw.ai/cli/acp)
- [Session 概念](https://docs.openclaw.ai/concepts/session)
