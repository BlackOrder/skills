# Patch splitting — rules for `git apply --cached --recount`

Sub-patches live under `.git/intent-commits/`, named `NN-<type>-<topic>.patch` (e.g. `01-feat-user-lookup.patch`).

These rules are what make `git apply` accept a hand-split patch.

## Rules

1. **Keep the full per-file header block** verbatim from `ALL.patch`:

   ```
   diff --git a/path b/path
   index abc123..def456 100644
   --- a/path
   +++ b/path
   ```

   Do **not** invent or edit `index` lines.

2. **Keep each `@@ -A,B +C,D @@` hunk header line intact**, including any function-context tail. After splitting:
   - The **starting line numbers** `A` and `C` must remain valid (use the originals from `ALL.patch`). `--recount` does **not** fix wrong or missing start lines.
   - The **line counts** `B` and `D` may be wrong; that is what `--recount` repairs.
   - Never omit, zero out (`@@ -0,0 +0,0 @@`), or strip the header. Never collapse the form to `@@ ... @@` without numbers.

   If `git apply --check` rejects the sub-patch with "corrupt patch at line N" or "patch fragment without header", the cause is almost always a mangled `@@` line — fix the start numbers, do not retry with different flags.

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

## Rename handling

A rename is its own intent and MUST be committed separately from any content edit on the renamed file.

`ALL.patch` shows a rename as one of:

```
diff --git a/old.ts b/new.ts
similarity index 100%
rename from old.ts
rename to new.ts
```

(pure rename, no content change — already a single intent)

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

(rename + content edit in one block — must split into two intents)

### Splitting "rename + content edit" into two intents

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

   Apply **after** intent A, so on disk the file already lives at the new path.

Renames therefore appear as **two (or more) rows** in the intents table — one `refactor:rename-…` and one or more content-edit intents on the new path. The user is free to merge them back into one commit during approval, but the default presentation always keeps them separate.

### Undetected renames

If git did not detect the rename (similarity below `diff.renameLimit`), `ALL.patch` shows a delete + add. Treat the same way: one intent for the delete + new-file-skeleton, additional intents for the substantive content changes — and tell the user in the table that this pair is a rename git failed to auto-detect.

**Annotate the table, not the commit body.** Do not write "git did not auto-detect this rename" into the rename commit's message body. Once the rename is split out into its own atomic commit (whose only content is `app.py => mathlib.py` with zero insertions/deletions), git almost always re-detects it as a 100% rename in `git log --stat` — making the warning false in the very history it tries to explain. The "undetected" classification is a property of the *dirty working tree*, not of the *committed history*. Surface it in the approval-gate table where it is true; omit it from the commit message where it would become misleading.
