---
name: intent-commits
description: Split a dirty working tree into atomic, intent-scoped commits via `git apply --cached`. Use when uncommitted changes mix multiple intents (feat, fix, refactor, docs, chore) — even in the same file or same hunk — and the user asks to "split this into commits", "separate these changes", "commit by intent", "make atomic commits", or "untangle this diff".
---

# intent-commits

Split a mixed working tree into atomic, intent-scoped commits using `git apply --cached --recount` on split unified diffs. The working tree on disk is read-only to this skill — every operation writes to the index or to history, never to files.

## When to Use

Trigger this skill whenever **all** are true:

1. The working tree has uncommitted changes (`git status` is non-empty).
2. Those changes mix more than one logical intent (feature + bugfix + refactor + stray formatting, etc.).
3. The user wants separate, atomic commits — possibly across the **same file** or even the **same hunk**.

Do **not** use this skill for:

- A single-intent change → just `git add` + `git commit`.
- Cleaning up history that is already committed → use `git rebase -i`.

## Why this approach

`git add -p` is interactive and fragile to script. The reliable agent primitive is:

```
git apply --cached --recount <patch-file>
```

It stages exactly the hunks in the patch, with no prompts. `--recount` lets git recompute `@@` line counts so the agent does not have to keep them perfectly accurate when splitting.

Flow: **capture one big patch → split into intent-scoped sub-patches → present the intents table for user approval → apply only what the user approved → recompute the remaining diff → present the table again.** The agent never re-edits source files, so it cannot accidentally lose work.

## Hard Rules (do not violate)

1. **Working tree is read-only.** Every op writes to the **index** (`git apply --cached`) or **history** (`git commit`). MUST NOT run `git checkout`, `git restore`, `git reset --hard`, `git apply` (without `--cached`), `git stash pop`, or anything else that modifies a file on disk. This is the entire reason the skill exists — LLMs natively try to "remove the changes and re-apply them step by step", and that is exactly what this skill replaces.
2. **Scope is splitting only.** MUST NOT push, fetch, pull, rebase, merge, cherry-pick, tag, or modify remotes. Local commit creation only.
3. **HALT on in-progress git operations.** Mid-merge / mid-rebase / mid-cherry-pick / mid-revert / mid-bisect → emit a warning naming the state and stop. Do not abort, continue, or skip — that is the user's call. See [Preconditions](#preconditions) step 1.
4. **No commit without explicit user approval of the intents table.** See [`references/approval-gate.md`](references/approval-gate.md). Auto-committing all intents in one shot is forbidden.
5. **Conventional Commits is mandatory.** Every message — both `short` and `full` variants — follows [`references/conventional-commits.md`](references/conventional-commits.md).
6. **After every batch of approved commits, recompute the diff and re-present the table.** Never assume the previously-shown intents are still accurate after a commit lands.
7. **If the user merges or splits intents, re-present the table and wait for approval again.** Never apply a merged/split grouping without re-confirmation.
8. **Renames are their own intent.** Committed separately from any content change to the renamed file. See [`references/patch-splitting.md`](references/patch-splitting.md#rename-handling).
9. **Never edit `+` / `-` / context lines** while splitting. Only delete whole lines, or convert `-` to a leading-space context line per [`references/patch-splitting.md`](references/patch-splitting.md) rule 4.
10. **Never run `git add -p`** from inside the agent.

## Preconditions

1. **HALT-check.** If any in-progress git operation is detected, warn and stop:

   ```sh
   GD=$(git rev-parse --git-dir)
   for f in MERGE_HEAD REBASE_HEAD CHERRY_PICK_HEAD REVERT_HEAD BISECT_LOG \
            rebase-merge rebase-apply; do
     [ -e "$GD/$f" ] && echo "HALT: in-progress git operation detected ($f)." && exit 1
   done
   ```

2. Record the starting commit and inspect state:

   ```sh
   git status --porcelain=v1
   git stash list
   git rev-parse HEAD          # record this — the recovery point
   ```

3. Refuse to contaminate the new commits with anything already staged:

   ```sh
   git diff --cached --quiet || echo "INDEX NOT EMPTY"
   ```

   If non-empty, ask the user whether to commit the existing index first or `git reset` (mixed — disk is not touched) to fold those changes back into the working tree.

4. Capture the source-of-truth patch. **Wipe any leftover workspace from a previous (aborted) run first.**

   ```sh
   rm -rf .git/intent-commits
   mkdir -p .git/intent-commits
   git diff > .git/intent-commits/ALL.patch
   ```

   `ALL.patch` is a working document, not a recovery artifact. The `.git/intent-commits/` directory is ephemeral and is removed on every termination path — see [`references/recovery.md`](references/recovery.md).

5. Handle untracked files. `git diff` does not include them:

   ```sh
   git ls-files --others --exclude-standard
   ```

   For each untracked text file, run `git add -N <file>` so its content shows up in `git diff`. Untracked **binary** files cannot be split — surface them in the intents table as their own atomic intent (apply with plain `git add` at commit time).

## Procedure

### Step 1 — Inventory the change

Read `.git/intent-commits/ALL.patch` end to end. Build an internal hunk inventory: one row per hunk, columns `# | file | hunk header | summary | intent label`. Intent labels are `<type>:<scope-or-topic>` and group hunks 1:1 into commits. Renames always get their own label (Hard Rule 8). Hunks that span multiple intents are duplicated — one row per intent (see split rule 4 in [`references/patch-splitting.md`](references/patch-splitting.md)).

### Step 2 — Split the patch

Create one sub-patch per intent under `.git/intent-commits/`, named `NN-<type>-<topic>.patch`. The split rules — what the splitter MUST follow byte-for-byte to keep `git apply` happy — are in [`references/patch-splitting.md`](references/patch-splitting.md). Rename handling lives in the same file.

### Step 3 — Present the intents table (mandatory approval gate)

Emit the intents table and message variants in the exact format defined in [`references/approval-gate.md`](references/approval-gate.md), then **wait** for the user reply. The reply grammar (`4 short, 7 full, 2 short` / `all in order, defaults` / `merge X+Y` / `split X` / `skip N`) is in the same file. Do not skip or condense the re-display after a merge/split instruction.

### Step 4 — Validate, apply, commit (one intent at a time, in user order)

For each approved intent, in user-specified order:

```sh
git apply --check --cached --recount .git/intent-commits/NN-<type>-<topic>.patch
```

If `--check` fails: stop, do not touch the index, follow [`references/recovery.md`](references/recovery.md).

On success:

```sh
git apply --cached --recount .git/intent-commits/NN-<type>-<topic>.patch
git diff --cached --stat
git commit -m "<subject>" [-m "<body>"] [-m "<footer>"]   # see references/conventional-commits.md
git diff --cached --quiet && echo "index clean"
```

If the index is not clean after the commit, the patch staged more than expected — invoke recovery and stop.

### Step 5 — Recompute and re-present (loop)

After the **whole batch** is committed:

```sh
git diff > .git/intent-commits/ALL.patch
rm -f .git/intent-commits/[0-9]*.patch          # old sub-patches are stale
```

Run Steps 1–3 again on the new `ALL.patch` (numbering restarts from 1). Even if only one intent remains, the table is still shown and approval is still required. Loop until `ALL.patch` is empty or the user says "done".

When `ALL.patch` is empty:

```sh
git diff --stat              # should be empty
git status --porcelain=v1    # only deferred / untracked-by-design files
git log --oneline -n 20      # confirm the intent commits in expected order
rm -rf .git/intent-commits   # cleanup is unconditional on every termination path
```

## Recovery

Single termination handler for any abnormal exit. Two unconditional commands, never touch disk:

```sh
git reset <recorded-HEAD>     # mixed: HEAD back, index cleared, files untouched
rm -rf .git/intent-commits
```

Full rationale, the prohibited-recovery list, and the per-exit termination table are in [`references/recovery.md`](references/recovery.md).

## Installation

See [`references/installation.md`](references/installation.md) for `npx skills add` invocations and the per-agent path table.
