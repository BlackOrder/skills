# AGENTS.md

Context for future agent sessions working in this repository. This file uses the [AGENTS.md](https://agents.md/) convention so it is picked up by any agent — Claude Code, GitHub Copilot, Antigravity, Cursor, Codex, etc. — without per-tool duplication.

## What this repo is

`BlackOrder/skills` — a personal collection of cross-platform agent skills consumed by the [`skills`](https://github.com/vercel-labs/skills) CLI:

```sh
npx skills add BlackOrder/skills --list
npx skills add BlackOrder/skills --skill <name>
```

…which pulls one or more skills from the [skills/](skills/) directory into the consumer's project (or global profile via `-g`).

## Repo layout

```
/
├── README.md          # User-facing: what's in the repo, how to install
├── AGENTS.md          # This file: agent/session context
├── LICENSE            # MIT
└── skills/
    └── <skill-name>/
        └── SKILL.md   # name field MUST equal folder name
```

One skill per folder under `skills/`. Each skill is self-contained — the goal is a single `SKILL.md` with no helper scripts unless absolutely required, so the skill ports unchanged across every agent platform.

## Skill conventions (apply to every new skill in this repo)

- **Folder name == `name:` frontmatter field.** Mismatch = silent discovery failure.
- **Description is the discovery surface.** Pack trigger phrases into the YAML `description`. Use the "Use when… / Do not use for…" pattern. Keep under 1024 chars.
- **Cross-platform first.** No platform-specific tooling. Bash + standard Unix + the tool the skill is actually about (`git`, `npm`, etc.). No Node/Python helper scripts unless the skill is about Node/Python.
- **Hard rules section.** Every skill that has user-approval gates or destructive operations lists them in a numbered "Hard Rules (do not violate)" section near the top.
- **Approval gates over autopilot.** Skills that mutate the repo (commits, file rewrites) MUST require explicit user approval before applying. Display a structured table, accept a reply grammar, re-display after merge/split/loop.
- **Recovery first.** Before any destructive op, save a backup (e.g. `git diff --binary > .git/<skill>/ALL.binary.patch`) and document the recovery one-liner in the SKILL.md.
- **Conventional Commits.** When a skill produces commits, messages follow Conventional Commits 1.0.0. Both `short` and `full` variants offered; `full` is mandatory for breaking changes, refs, or non-trivial diffs.

## Current skills

| Skill                                            | Purpose                                                                                                                                                                                                                                  |
| ------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [intent-commits](skills/intent-commits/SKILL.md) | Split a mixed working tree into atomic, intent-scoped commits using `git apply --cached --recount` on split unified diffs. Mandatory user-approval gate; Conventional Commits enforced; renames committed separately from content edits. |

## Installation paths consumers will use

The `skills` CLI writes to the per-agent directory automatically. A few of the most common targets (full table: [vercel-labs/skills → Supported Agents](https://github.com/vercel-labs/skills#supported-agents)):

| Agent          | Project                  | Global                                 |
| -------------- | ------------------------ | -------------------------------------- |
| Claude Code    | `.claude/skills/<name>/` | `~/.claude/skills/<name>/`             |
| GitHub Copilot | `.agents/skills/<name>/` | `~/.copilot/skills/<name>/`            |
| Codex          | `.agents/skills/<name>/` | `~/.codex/skills/<name>/`              |
| Cursor         | `.agents/skills/<name>/` | `~/.cursor/skills/<name>/`             |
| Antigravity    | `.agents/skills/<name>/` | `~/.gemini/antigravity/skills/<name>/` |
| OpenCode       | `.agents/skills/<name>/` | `~/.config/opencode/skills/<name>/`    |
| Gemini CLI     | `.agents/skills/<name>/` | `~/.gemini/skills/<name>/`             |

Note `.agents/skills/` is the shared project-scope path for many agents — a single project install usually covers Codex, Cursor, GitHub Copilot, Antigravity, OpenCode, Gemini CLI and friends at once. Claude Code is the main exception (its own `.claude/skills/`).

Skills in this repo do **not** pin themselves to any of these paths; the CLI handles placement. SKILL.md files only need to live under `skills/<name>/SKILL.md` in this repo for the CLI to discover them.

## Commit messages (this repo)

Commits to this repo follow [Conventional Commits 1.0.0](https://www.conventionalcommits.org/en/v1.0.0/) — same rule we enforce inside the `intent-commits` skill, applied to ourselves.

- **type**: `feat`, `fix`, `refactor`, `perf`, `docs`, `test`, `build`, `ci`, `chore`, `style`, `revert`.
- **scope**: the skill folder name when the change is skill-scoped (e.g. `feat(intent-commits): …`, `docs(intent-commits): …`); `repo` for cross-cutting changes (e.g. `chore(repo): rename CLAUDE.md to AGENTS.md`).
- **description**: imperative, lowercase, no trailing period, ≤ 72 chars.
- Body explains _why_ and is mandatory for non-trivial changes, breaking changes, or anything with a `Refs:` footer.
- Breaking changes use `!` before the colon AND a `BREAKING CHANGE:` footer.

## PR / change checklist (for agents)

When applying any change to this repo, before declaring done:

1. **Adding a new skill** (`skills/<name>/SKILL.md`):
   - Folder name == `name:` field.
   - Description ≤ 1024 chars and contains explicit trigger phrases.
   - "Hard Rules" section present if the skill mutates anything.
   - Recovery procedure documented if the skill is destructive.
   - Cross-Platform Installation table at the bottom uses the skill's actual folder name (not a placeholder).
   - **Updated the "Skills" / "Current skills" table in BOTH [README.md](README.md) AND this file.**
2. **Renaming a skill**:
   - Renamed folder, `name:` field, and every install-path reference inside its `SKILL.md`.
   - Updated both tables (README + AGENTS).
3. **Editing an existing skill**:
   - Preserved its Hard Rules and approval-gate structure unless the user explicitly asked to change them.
   - Did not add helper scripts unless explicitly asked.
4. **Run the validation snippet below** and confirm no output before finishing.

## Validation

Sanity-check that every skill's `name:` field matches its folder name:

```sh
for d in skills/*/; do
  n=$(awk -F': *' '/^name:/{print $2; exit}' "$d/SKILL.md")
  [ "$n" = "$(basename "$d")" ] || echo "MISMATCH: $d -> name=$n"
done
```

Expected output: nothing. Any line printed is a silent-discovery-failure waiting to happen.

## Working notes for future sessions

- **Don't add scripts to existing skills** unless the user explicitly asks. The script-free rule is a hard design constraint, not an oversight.
- **Don't auto-rename `name:` without renaming the folder** (and vice versa). They must match.
- **When asked to "add a skill"** → create `skills/<name>/SKILL.md`, update the "Current skills" table in this file and in `README.md`.
- **When asked to update an existing skill** → preserve its hard-rules and approval-gate structure; those reflect deliberate decisions made with the repo owner.
- **The skills here are the canonical copies.** Symlinks/copies elsewhere on the system are downstream — edit here, then redistribute.

## Reference: agent-customization primitives

Skills are one of several VS Code / Copilot customization primitives. For deeper authoring guidance, the `agent-customization` skill that ships with VS Code Copilot covers the full taxonomy (instructions, prompts, hooks, custom agents, skills, MCP). Consult it when designing a new primitive — but in this repo, default to **skill** unless there's a clear reason not to.
