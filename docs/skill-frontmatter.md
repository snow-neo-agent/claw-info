# SKILL.md Frontmatter 欄位說明

OpenClaw 讀取每個 `SKILL.md` 的 frontmatter，將 skill 註冊為 Telegram/Discord slash command。

## 欄位一覽

| 欄位 | 必填 | 說明 |
|------|------|------|
| `name` | ✅ | Skill 名稱；自動成為 slash command（如 `name: prcli` → `/prcli`） |
| `description` | ✅ | 顯示於 slash command 選單；LLM 依此自動選用 skill（上限 100 字元） |
| `command-dispatch` | ❌ | 設為 `tool` 可繞過 LLM，直接 dispatch 至指定工具 |
| `command-tool` | ❌ | 要呼叫的工具名稱（如 `exec`）；`command-dispatch: tool` 時必填 |
| `command-arg-mode` | ❌ | 參數傳遞方式；`raw` = 原樣傳入（目前唯一支援值） |

來源：[`src/agents/skills/workspace.ts`](https://github.com/openclaw/openclaw/blob/v2026.3.2/src/agents/skills/workspace.ts)（v2026.3.2）
- `name` / `description`：[L686](https://github.com/openclaw/openclaw/blob/v2026.3.2/src/agents/skills/workspace.ts#L686)、[L699](https://github.com/openclaw/openclaw/blob/v2026.3.2/src/agents/skills/workspace.ts#L699)
- `command-dispatch`：[L706](https://github.com/openclaw/openclaw/blob/v2026.3.2/src/agents/skills/workspace.ts#L706)
- `command-tool`：[L720](https://github.com/openclaw/openclaw/blob/v2026.3.2/src/agents/skills/workspace.ts#L720)
- `command-arg-mode`：[L734](https://github.com/openclaw/openclaw/blob/v2026.3.2/src/agents/skills/workspace.ts#L734)

## Dispatch 模式比較

### 預設（經 LLM 判斷）

```
用戶 /skillname args → OpenClaw → LLM 推理 → 呼叫工具 → 回傳結果
```

彈性高但較慢，適合需要 LLM 理解意圖的 skill。

### 直接 dispatch（`command-dispatch: tool`）

```
用戶 /skillname args → OpenClaw → 直接呼叫工具 → 回傳結果
```

跳過 LLM，速度快、成本低、行為確定。適合 CLI 封裝型 skill。

## 範例

```yaml
---
name: my-skill
description: 執行 my-cli 工具。當詢問 my-cli 相關問題時使用。
command-dispatch: tool
command-tool: exec
command-arg-mode: raw
---
```

設定後，用戶輸入 `/my-skill foo bar`，OpenClaw 直接以 `foo bar` 為參數呼叫 `exec` 工具，不經 LLM。

## Native Slash Command 設定

Slash command 預設在 Telegram/Discord 自動啟用，由 `openclaw.json` 的 `nativeSkills` 控制：

```json
"commands": {
  "nativeSkills": "auto"
}
```

| 值 | 行為 |
|----|------|
| `auto` | TG/Discord 啟用，Slack 停用 |
| `true` | 全部啟用 |
| `false` | 全部停用 |
