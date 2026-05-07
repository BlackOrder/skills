# Recovery and termination paths

Recovery is the single termination handler for **any** abnormal exit: `git apply --check` failure, dirty index after commit, user abort during the approval gate, an unexpected error — anything that is not the clean "`ALL.patch` empty" finish.

## Why recovery never touches disk

The skill never modifies files on disk — `git apply --cached` writes only to the index, and `git commit` only advances `HEAD`. So the working tree at any point during the run is bit-identical to what it was when the skill started. **Recovery requires zero disk operations.**

## The two commands

```sh
git reset <HEAD-recorded-in-preconditions>     # mixed (the default): moves HEAD, clears index, leaves files alone
rm -rf .git/intent-commits                     # wipe the ephemeral workspace
```

After recovery:

- `HEAD` is back at the pre-skill commit.
- The index is empty.
- The working tree is unchanged — exactly as the user left it before invoking the skill.
- `.git/intent-commits/` is gone. The next run starts from a clean slate.

The skill's commits are not lost data; they are recoverable from `git reflog` for the usual `gc.reflogExpire` window if the user wants them back.

## What recovery MUST NOT do

Do **not** use `git reset --hard`, `git checkout`, `git restore`, or `git apply` (without `--cached`) as part of recovery. None of them are needed, and all of them would violate the working-tree-read-only invariant by writing to the working tree.

## Termination paths (all converge on the cleanup above)

| Exit reason                                               | Action                                                                              |
| --------------------------------------------------------- | ----------------------------------------------------------------------------------- |
| Clean finish (`ALL.patch` empty)                          | `rm -rf .git/intent-commits` (no `git reset` — the commits are the desired outcome). |
| HALT during Preconditions (in-progress merge/rebase/…)    | Stop before creating `.git/intent-commits/`. Nothing to clean.                       |
| User aborts at the approval gate before any commit lands  | `rm -rf .git/intent-commits` (no `git reset` — no commits were made).                |
| `git apply --check` fails mid-batch                       | `git reset <recorded-HEAD>` + `rm -rf .git/intent-commits`.                          |
| Index dirty after a commit, or any other unexpected error | `git reset <recorded-HEAD>` + `rm -rf .git/intent-commits`.                          |
