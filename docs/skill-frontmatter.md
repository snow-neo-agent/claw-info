# SKILL.md Frontmatter Keys

OpenClaw reads frontmatter from each `SKILL.md` to register skills and expose them as slash commands in Telegram/Discord.

## Keys

| Key | Required | Description |
|-----|----------|-------------|
| `name` | ✅ | Skill name; becomes the slash command (e.g. `name: prcli` → `/prcli`) |
| `description` | ✅ | Shown in slash command menu; used by LLM to auto-select skill (max 100 chars) |
| `command-dispatch` | ❌ | Set to `tool` to bypass LLM and dispatch directly to a tool |
| `command-tool` | ❌ | Tool name to invoke (e.g. `exec`); required when `command-dispatch: tool` |
| `command-arg-mode` | ❌ | How args are forwarded; `raw` = pass as-is (only supported value) |

Source: [`src/agents/skills/workspace.ts`](https://github.com/openclaw/openclaw/blob/v2026.3.2/src/agents/skills/workspace.ts) (v2026.3.2)
- `name` / `description`: [L686](https://github.com/openclaw/openclaw/blob/v2026.3.2/src/agents/skills/workspace.ts#L686), [L699](https://github.com/openclaw/openclaw/blob/v2026.3.2/src/agents/skills/workspace.ts#L699)
- `command-dispatch`: [L706](https://github.com/openclaw/openclaw/blob/v2026.3.2/src/agents/skills/workspace.ts#L706)
- `command-tool`: [L720](https://github.com/openclaw/openclaw/blob/v2026.3.2/src/agents/skills/workspace.ts#L720)
- `command-arg-mode`: [L734](https://github.com/openclaw/openclaw/blob/v2026.3.2/src/agents/skills/workspace.ts#L734)

## Dispatch Modes

### Default (LLM-mediated)

```
User /skillname args → OpenClaw → LLM → tool call → result
```

Flexible but slower. Suitable for skills requiring reasoning.

### Direct dispatch (`command-dispatch: tool`)

```
User /skillname args → OpenClaw → tool (exec) → result
```

Bypasses LLM entirely. Faster, cheaper, deterministic. Suitable for CLI-wrapper skills.

## Example

```yaml
---
name: my-skill
description: Run my-cli tool. Use when asked about my-cli.
command-dispatch: tool
command-tool: exec
command-arg-mode: raw
---
```

With this config, `/my-skill foo bar` directly invokes the `exec` tool with `foo bar` as raw args.

## Native Slash Commands

Slash commands are auto-enabled for Telegram and Discord when `nativeSkills` is `auto` (default) in `openclaw.json`:

```json
"commands": {
  "nativeSkills": "auto"
}
```

| Value | Behavior |
|-------|----------|
| `auto` | Enabled for TG/Discord, disabled for Slack |
| `true` | Always enabled |
| `false` | Always disabled |
