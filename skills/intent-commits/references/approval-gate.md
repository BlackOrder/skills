# Approval gate — intents table format and reply grammar

Before applying anything, the agent MUST output the intents table to the user, in this exact format.

## Table format

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

## Hard requirements

- The agent MUST emit **both** `short` and `full` for every intent, even when one is flagged "(not recommended)".
- The agent MUST NOT proceed past this step without an explicit user reply naming the intents to apply.
- If the user replies with a **merge** or **split** instruction, the agent re-runs patch splitting for the affected intents and re-emits the **entire** updated table (do not skip the re-display, even if "obvious"). **No commit is made until the user approves the post-merge/post-split table.**
- The user's reply order is the commit order. The agent MUST honor it. If the requested order is impossible because of patch dependencies (the later patch will not apply on top of the earlier one), the agent stops, surfaces the conflict in plain language, and proposes a re-order — it does not silently reorder.
- The user can mix and match in a single reply: e.g. `4 short, 7 full, 2 short` means "commit only those three intents, in that order, with those message lengths; leave the rest for the next round".
