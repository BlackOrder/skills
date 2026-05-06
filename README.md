# skills

A collection of cross-platform agent skills for LLM coding assistants — Claude Code, GitHub Copilot, Antigravity, Cursor, Codex, OpenCode, and the [50+ other agents](https://github.com/vercel-labs/skills#supported-agents) supported by the [`skills`](https://github.com/vercel-labs/skills) CLI.

Every skill in this repo is **single-file, script-free, and platform-agnostic**: drop the folder into your agent's skill directory and it works.

## Install

Use the [`skills`](https://github.com/vercel-labs/skills) CLI:

```sh
# List available skills in this repo
npx skills add BlackOrder/skills --list

# Install one skill (auto-detects which agents you have)
npx skills add BlackOrder/skills --skill git-split

# Install everything, no prompts
npx skills add BlackOrder/skills --all

# Install globally (~/<agent>/skills/) instead of into the current project
npx skills add BlackOrder/skills --skill git-split -g

# Target specific agents
npx skills add BlackOrder/skills --skill git-split -a claude-code -a codex
```

The CLI auto-detects every supported agent installed on your machine and writes the skill to each one's correct directory (project-scoped by default, global with `-g`).

## Manual install

If you'd rather not use the CLI, clone the repo and symlink (or copy) the skill folder into the path your agent reads from. Most modern agents (Codex, Cursor, GitHub Copilot, Antigravity, OpenCode, Gemini CLI, …) share `.agents/skills/` for project scope, so a single symlink usually covers many of them:

```sh
git clone https://github.com/BlackOrder/skills.git ~/src/blackorder-skills
cd your-project

# Project scope — covers most agents at once
mkdir -p .agents/skills
ln -s ~/src/blackorder-skills/skills/git-split .agents/skills/git-split

# Claude Code uses its own path
mkdir -p .claude/skills
ln -s ~/src/blackorder-skills/skills/git-split .claude/skills/git-split
```

For the full per-agent path table, see [vercel-labs/skills → Supported Agents](https://github.com/vercel-labs/skills#supported-agents). The folder name on the destination side **must** equal the skill's `name:` field (in this repo, folder name and `name:` already match — keep them identical wherever you copy them).

## Skills

| Skill                                  | What it does                                                                                                                                                                                                                                                                                                                                           |
| -------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| [git-split](skills/git-split/SKILL.md) | Take a messy working tree (feature + bugfix + refactor mixed in the same files / hunks) and turn it into clean, atomic, intent-scoped commits. Uses `git apply --cached --recount` on split unified diffs — never re-edits source files. Mandatory user-approval gate, Conventional Commits enforced, renames committed separately from content edits. |

## Repo layout

```
/
├── README.md          # this file
├── AGENTS.md          # context for agent sessions working on this repo
├── LICENSE            # MIT
└── skills/
    └── <skill-name>/
        └── SKILL.md
```

One skill per folder. Each `SKILL.md` is self-contained.

## Contributing a skill

1. Add a folder under `skills/` whose name matches the skill's `name:` frontmatter field.
2. Write a single `SKILL.md` — no helper scripts unless the skill is genuinely about Node/Python/etc.
3. Pack trigger phrases into the `description` field; that's how agents discover the skill.
4. If the skill mutates the repo (commits, file rewrites, etc.), include a "Hard Rules" section with an explicit user-approval gate and a recovery procedure.
5. Add a row to the **Skills** table above.

See [AGENTS.md](AGENTS.md) for the full conventions used in this repo.

## License

[MIT](LICENSE).
