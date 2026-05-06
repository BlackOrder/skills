---
name: git-split
description: 'Stage and commit a mixed working tree as multiple intent-scoped commits (feature, bugfix, refactor, docs, chore) using `git apply --cached` on split unified diffs, with a mandatory user-approval step before any commit is made. Use when the working tree contains unrelated changes — even multiple intents in the same file or same hunk — and the user asks to "split this into commits", "separate these changes", "commit by intent", "stage by feature/bugfix/refactor", "make atomic commits", "untangle this diff", or "commit only the bugfix part". Avoids the brittle `git add -p` TTY workflow and the destructive "revert files and reapply" routine. Works on every agent platform (Claude, Copilot, Antigravity, Cursor, Codex, etc.) — relies only on `git`, `git apply`, and a text patch file. No helper scripts.'
---

# git-split

Split a mixed working tree into atomic, intent-scoped commits using `git apply --cached --recount` on split unified diffs.

## When to Use

Trigger this skill whenever **all** are true:

1. The working tree has uncommitted changes (`git status` is non-empty).
2. Those changes mix more than one logical intent (e.g. a new feature + an unrelated bugfix + a refactor + a stray formatting change).
3. The user wants separate, atomic commits — possibly across changes that touch the **same file** or even the **same hunk**.

Do **not** use this skill for:

- A single-intent change → just `git add` + `git commit`.
- Cleaning up commit history that is already committed → use `git rebase -i` instead.

## Why This Approach

`git add -p` is interactive and fragile to script — it requires a TTY and `expect`-style automation that breaks across platforms. The reliable primitive for agents is:

```
git apply --cached --recount <patch-file>
```

It stages exactly the hunks present in the patch, with no prompts. `--recount` lets `git` recompute the line counts in `@@` headers, so the agent does not have to keep them perfectly accurate when splitting.

The flow is therefore: **capture one big patch → split into intent-scoped sub-patches → present the intents table for user approval → apply only what the user approved → recompute the remaining diff → present the table again.** The agent never re-edits source files, so it cannot accidentally lose work.

## Hard Rules (do not violate)

These rules are non-negotiable. Violating any of them is a bug in the skill execution.

1. **No commit without explicit user approval of the intents table.** The agent MUST display the table defined in [Step 3](#step-3--present-the-intents-table-mandatory-approval-gate) and wait for the user to pick which intents to commit, in what order, with which message variant. Auto-committing all intents in one shot is forbidden, even if the agent is "sure" about the grouping.
2. **Conventional Commits is mandatory.** Every commit message — both `short` and `full` variants — MUST follow the [Commit Message Convention](#commit-message-convention) below. No free-form messages.
3. **After every batch of approved commits, recompute the diff and re-present the table.** The agent does not assume the previously-shown intents are still accurate after a commit lands.
4. **If the user merges or splits intents, the agent re-presents the table and waits for approval again.** Never apply a merged/split grouping without re-confirmation.
5. **Renames are their own intent.** A rename is committed separately from any content change to the renamed file. See [Rename Handling](#rename-handling).
6. **Never edit `+` / `-` / context lines** while splitting. Only delete whole lines, or convert `-` to a leading-space context line per [Step 2 rule 4](#step-2--split-the-patch).
7. **Never run `git add -p`** from inside the agent. Never use `git checkout -- <file>` to "revert and reapply" the user's work.

## Commit Message Convention

All commit messages MUST follow [Conventional Commits 1.0.0](https://www.conventionalcommits.org/en/v1.0.0/):

```
<type>(<scope>)<!>: <description>

[optional body]

[optional footer(s)]
```

- **type** — one of: `feat`, `fix`, `refactor`, `perf`, `docs`, `test`, `build`, `ci`, `chore`, `style`, `revert`.
- **scope** — optional, lowercase, kebab-case, derived from the touched module/path (e.g. `api`, `auth`, `users-page`).
- **`!`** — append `!` before the colon for a breaking change, AND include a `BREAKING CHANGE: <reason>` footer.
- **description** — imperative mood, lowercase, no trailing period, ≤ 72 chars.
- **body** — wrapped at 72 chars, explains _why_.
- **footers** — `Refs: #123`, `Co-authored-by: …`, `BREAKING CHANGE: …`.

For every intent the agent MUST prepare two variants:

| Variant | Contents                                                        | Used for                                                                                                              |
| ------- | --------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------- |
| `short` | Subject line only (`<type>(<scope>): <description>`).           | Trivial, self-evident commits.                                                                                        |
| `full`  | Subject + blank line + body (the _why_) + any required footers. | Anything non-obvious; mandatory when `!` breaking, when there is a `Refs:`, or when the body adds meaningful context. |

If a `full` variant is mandatory (breaking change, ref, or non-trivial diff), the `short` variant is still computed and offered, but flagged: `short (not recommended)`.

## Preconditions

Before doing anything else:

1. Confirm the repo state and record the starting commit:

   ```sh
   git status --porcelain=v1
   git stash list
   git rev-parse HEAD          # record this — it is the recovery point
   ```

2. Make sure nothing is already staged that would contaminate the new commits:

   ```sh
   git diff --cached --quiet || echo "INDEX NOT EMPTY"
   ```

   If non-empty, ask the user whether to commit the existing index first or `git reset` to fold those changes back into the working tree.

3. Save a full backup of the current diff so the operation is recoverable:

   ```sh
   mkdir -p .git/intent-split
   git diff > .git/intent-split/ALL.patch
   git diff --binary > .git/intent-split/ALL.binary.patch
   ```

   `ALL.patch` is the source of truth for splitting. Recovery: `git reset --hard <recorded-HEAD> && git apply .git/intent-split/ALL.binary.patch`.

4. Handle untracked files explicitly. `git diff` does **not** include them. List them:
   ```sh
   git ls-files --others --exclude-standard
   ```
   For each untracked text file, run `git add -N <file>` so its content shows up in `git diff` and can be split with the rest. Untracked **binary** new files cannot be split — surface them in the intents table as their own atomic intent (apply with plain `git add` at commit time).

## Procedure

### Step 1 — Inventory the change

Read `ALL.patch` end to end. Build an internal hunk inventory of this shape (kept in agent state, not necessarily shown to the user yet):

| #   | file                         | hunk header          | summary                        | intent label                 |
| --- | ---------------------------- | -------------------- | ------------------------------ | ---------------------------- |
| 1   | src/api/users.ts             | @@ -10,6 +10,18 @@   | adds `getUserById` endpoint    | feat:user-lookup             |
| 2   | src/api/users.ts             | @@ -55,3 +67,3 @@    | fixes off-by-one in pagination | fix:pagination               |
| 3   | src/api/users.ts             | @@ -90,10 +102,8 @@  | extracts `validateRole` helper | refactor:validate-role       |
| 4   | README.md                    | @@ -1,3 +1,5 @@      | documents new endpoint         | docs:user-lookup             |
| 5   | src/api/{users.ts → user.ts} | (rename, no content) | renames module                 | refactor:rename-users-module |
| 6   | src/api/user.ts              | @@ -20,4 +20,7 @@    | adds field to renamed module   | feat:user-lookup             |

Intent labels are `<type>:<scope-or-topic>` and group hunks 1:1 into commits. Renames always get their own label (rule 5). Hunks that span multiple intents are duplicated in this inventory, one row per intent (see Step 2 rule 4).

### Step 2 — Split the patch

Create one sub-patch file per intent under `.git/intent-split/`, named `NN-<type>-<topic>.patch` (e.g. `01-feat-user-lookup.patch`).

Rules — these are what make `git apply` succeed:

1. **Keep the full per-file header block** verbatim from `ALL.patch`:

   ```
   diff --git a/path b/path
   index abc123..def456 100644
   --- a/path
   +++ b/path
   ```

   Do **not** invent or edit `index` lines.

2. **Keep each `@@ ... @@` hunk header line intact**, including any function-context tail. Line counts may be wrong after splitting — that is what `--recount` is for. Never delete the header.

3. **Never edit `+`, `-`, or context (` `) lines.** Copy them byte-for-byte.

4. **Splitting a single hunk that mixes intents.** Duplicate the hunk into both sub-patches and in each copy:
   - Lines that belong to _this_ intent: keep as `+` / `-` / ` `.
   - `+` lines that belong to _another_ intent: drop them entirely.
   - `-` lines that belong to _another_ intent: convert to ` ` (leading space), so context still matches.

   Apply sub-patches in dependency order so each one sees the on-disk state it expects.

5. **Preserve all special headers** when present in the original file block:
   - `new file mode <mode>` / `deleted file mode <mode>`
   - `old mode` / `new mode`
   - `similarity index N%`
   - `Binary files ... differ` (binary changes are atomic — the whole file is one intent).

6. **End every patch file with a trailing newline.** `git apply` rejects patches missing the final newline.

### Rename Handling

A rename is an intent of its own and MUST be committed separately from any content edit on the renamed file.

When `ALL.patch` shows a rename, it looks like one of:

```
diff --git a/old.ts b/new.ts
similarity index 100%
rename from old.ts
rename to new.ts
```

(pure rename, no content change — already a single intent, easy)

or:

```
diff --git a/old.ts b/new.ts
similarity index 87%
rename from old.ts
rename to new.ts
index abc..def 100644
--- a/old.ts
+++ b/new.ts
@@ -10,3 +10,5 @@
 ...
-old line
+new line
```

(rename + content edit in one diff block — must be split into two intents)

To split a "rename + content edit" into two intents:

1. **Intent A — pure rename.** A sub-patch containing only:

   ```
   diff --git a/old.ts b/new.ts
   similarity index 100%
   rename from old.ts
   rename to new.ts
   ```

   Force `similarity index 100%` and attach no content hunks. This commits the rename alone.

2. **Intent B — content edit on the renamed file.** A sub-patch where both sides of the header use the **new** path, and only the content hunks remain:
   ```
   diff --git a/new.ts b/new.ts
   --- a/new.ts
   +++ b/new.ts
   @@ -10,3 +10,5 @@
    ...
   -old line
   +new line
   ```
   This is applied **after** intent A, so on disk the file already lives at the new path.

Renames therefore appear as **two (or more) rows** in the intents table — one `refactor:rename-…` and one or more content-edit intents on the new path. The user is free to merge them back into one commit during approval (Step 3), but the default presentation always keeps them separate.

If the rename involves a content edit large enough that `git` did not detect it as a rename (similarity below `diff.renameLimit`), `ALL.patch` will show a delete + add. Treat it the same way: one intent for the delete + new-file-skeleton, additional intents for the substantive content changes — and tell the user in the table that this pair is a rename git failed to auto-detect.

### Step 3 — Present the intents table (mandatory approval gate)

Before applying anything, the agent MUST output the intents table to the user, in this exact format:

```
Intents discovered (N total). Pick what to commit, in what order, and which message variant.

#  | type:scope             | files / hunks                          | message variants
---+------------------------+----------------------------------------+--------------------------------
 1 | refactor:rename-users  | src/api/users.ts → src/api/user.ts     | short | full
 2 | refactor:validate-role | src/api/user.ts (1 hunk)               | short | full
 3 | fix:pagination         | src/api/user.ts (1 hunk)               | short | full (recommended)
 4 | feat:user-lookup       | src/api/user.ts (1 hunk),              | short | full (recommended)
   |                        | README.md (1 hunk)                     |
 5 | docs:user-lookup       | README.md (1 hunk)                     | short | full
...

Proposed messages:

#1 short: refactor(api): rename users module to user
#1 full : refactor(api): rename users module to user

          Aligns with the new singular-resource naming convention used
          across the v2 API surface.

#2 short: refactor(user-api): extract validateRole helper
#2 full : refactor(user-api): extract validateRole helper

          No behaviour change. Prepares the codebase for the role-based
          access work tracked in #482.
          Refs: #482
...

How to approve:
  • Reply with a comma-separated list like:  4 short, 7 full, 2 short
    (use `short` or `long`/`full` per intent; order in your reply = commit order)
  • Or say:  all in order, defaults
    (uses the recommended variant for each, in table order)
  • Or merge intents:   merge 3+4 as fix:pagination
  • Or split an intent: split 4 — and describe how
  • Or defer intents:   skip 5
```

Hard requirements for this step:

- The agent MUST emit **both** `short` and `full` for every intent, even when one is flagged "(not recommended)".
- The agent MUST NOT proceed past this step without an explicit user reply naming the intents to apply.
- If the user replies with a **merge** or **split** instruction, the agent re-runs Step 2 for the affected intents and re-emits the **entire** updated table (do not skip the re-display, even if "obvious"). **No commit is made until the user approves the post-merge/post-split table.**
- The user's reply order is the commit order. The agent MUST honor it. If the requested order is impossible because of patch dependencies (the later patch will not apply on top of the earlier one), the agent stops, surfaces the conflict in plain language, and proposes a re-order — it does not silently reorder.
- The user can mix and match in a single reply: e.g. `4 short, 7 full, 2 short` means "commit only those three intents, in that order, with those message lengths; leave the rest for the next round".

### Step 4 — Validate, apply, commit (one intent at a time, in user order)

For each intent the user approved, in the user-specified order:

```sh
git apply --check --cached --recount .git/intent-split/NN-<type>-<topic>.patch
```

If `--check` fails, **stop immediately**, report the failure to the user, and do not touch the index. Common causes:

- A context line was edited → re-copy from `ALL.patch`.
- The intent depends on an earlier intent that the user did not include in this batch → ask the user how to proceed (include the dependency, or reorder).

On success:

```sh
git apply --cached --recount .git/intent-split/NN-<type>-<topic>.patch
git diff --cached --stat                         # sanity check
git commit -m "<subject>" [-m "<body>"] [-m "<footer>"]
```

Use the message variant the user picked for that intent (`-m` once for `short`, multiple `-m`s for `full` to keep the blank-line separators). After the commit:

```sh
git diff --cached --quiet && echo "index clean"
```

If the index is not clean, the patch staged more than expected — `git reset` and stop; do not proceed to the next intent.

### Step 5 — Recompute and re-present (loop)

After the **whole batch** the user approved is committed:

1. Regenerate the source of truth:
   ```sh
   git diff > .git/intent-split/ALL.patch
   git diff --binary > .git/intent-split/ALL.binary.patch
   rm -f .git/intent-split/[0-9]*.patch          # old sub-patches are stale
   ```
2. Run Steps 1–3 again on the new `ALL.patch`. The intents table is re-emitted with the **remaining** intents (numbering restarts from 1). Even if only one intent remains, the table is still shown and approval is still required.
3. Loop until the user says "done" or `ALL.patch` is empty.

When `ALL.patch` is empty:

```sh
git diff --stat              # should be empty
git status --porcelain=v1    # should show only files the user explicitly deferred / untracked-by-design
git log --oneline -n 20      # confirm the intent commits in expected order
```

Then clean up (only after the user confirms):

```sh
rm -rf .git/intent-split
```

## Recovery

If anything goes wrong mid-flow and the working tree is in an unknown state:

```sh
git reset --hard <HEAD-recorded-in-preconditions>
git apply .git/intent-split/ALL.binary.patch
```

This restores the exact starting state. Then restart from Step 1 with a better split.

## Anti-patterns to Avoid

- **Do not** auto-commit without showing the intents table and waiting for approval.
- **Do not** invent commit messages outside Conventional Commits.
- **Do not** apply a merge or split of intents without re-displaying the updated table.
- **Do not** silently reorder the user's requested commit order — surface the dependency conflict instead.
- **Do not** combine a rename with a content edit in one commit by default.
- **Do not** `git checkout -- <file>` then re-edit it to "reapply only the feature part".
- **Do not** use `git add -p` from inside an agent — it requires a TTY.
- **Do not** edit `+`/`-`/context lines while splitting.
- **Do not** commit a sub-patch without running `git apply --check` first.
- **Do not** skip the `ALL.patch` backup. Without it, recovery is impossible.
- **Do not** add helper scripts to this skill — the procedure is intentionally script-free so it ports to every agent platform unchanged.

## Cross-Platform Installation

Install via the [`skills`](https://github.com/vercel-labs/skills) CLI — it auto-detects every supported agent on your machine and writes to each one's correct directory:

```sh
npx skills add BlackOrder/skills --skill git-split        # project scope
npx skills add BlackOrder/skills --skill git-split -g     # global scope
npx skills add BlackOrder/skills --skill git-split -a claude-code -a codex
```

For manual installs, the most common destinations are:

| Agent          | Project                     | Global                                    |
| -------------- | --------------------------- | ----------------------------------------- |
| Claude Code    | `.claude/skills/git-split/` | `~/.claude/skills/git-split/`             |
| GitHub Copilot | `.agents/skills/git-split/` | `~/.copilot/skills/git-split/`            |
| Codex          | `.agents/skills/git-split/` | `~/.codex/skills/git-split/`              |
| Cursor         | `.agents/skills/git-split/` | `~/.cursor/skills/git-split/`             |
| Antigravity    | `.agents/skills/git-split/` | `~/.gemini/antigravity/skills/git-split/` |
| OpenCode       | `.agents/skills/git-split/` | `~/.config/opencode/skills/git-split/`    |

Full per-agent path table: [vercel-labs/skills → Supported Agents](https://github.com/vercel-labs/skills#supported-agents). The destination folder name **must** equal the skill's `name:` field (`git-split`).
