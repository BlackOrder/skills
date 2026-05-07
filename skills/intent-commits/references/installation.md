# Cross-platform installation

Install via the [`skills`](https://github.com/vercel-labs/skills) CLI — it auto-detects every supported agent on your machine and writes to each one's correct directory:

```sh
npx skills add BlackOrder/skills --skill intent-commits        # project scope
npx skills add BlackOrder/skills --skill intent-commits -g     # global scope
npx skills add BlackOrder/skills --skill intent-commits -a claude-code -a codex
```

For manual installs, the most common destinations are:

| Agent          | Project                          | Global                                         |
| -------------- | -------------------------------- | ---------------------------------------------- |
| Claude Code    | `.claude/skills/intent-commits/` | `~/.claude/skills/intent-commits/`             |
| GitHub Copilot | `.agents/skills/intent-commits/` | `~/.copilot/skills/intent-commits/`            |
| Codex          | `.agents/skills/intent-commits/` | `~/.codex/skills/intent-commits/`              |
| Cursor         | `.agents/skills/intent-commits/` | `~/.cursor/skills/intent-commits/`             |
| Antigravity    | `.agents/skills/intent-commits/` | `~/.gemini/antigravity/skills/intent-commits/` |
| OpenCode       | `.agents/skills/intent-commits/` | `~/.config/opencode/skills/intent-commits/`    |

Full per-agent path table: [vercel-labs/skills → Supported Agents](https://github.com/vercel-labs/skills#supported-agents). The destination folder name **must** equal the skill's `name:` field (`intent-commits`).
