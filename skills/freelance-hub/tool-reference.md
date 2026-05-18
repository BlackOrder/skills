# Tool Reference

## Tool Architecture

38 tools across 5 layers: 25 visible at startup, 13 hidden behind activators.

### Always-On Layers (27 tools at startup)

| Layer             | Count  | Tools                                                                                                                                                             |
| ----------------- | ------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Orchestration     | 10     | onboard-client, start-project, complete-milestone, record-payment, send-reminder, monthly-summary, close-project, accept-estimate, issue-credit, decline-estimate |
| Analytical        | 5      | dashboard-snapshot, get-action-items, standup, client-360, rate-review                                                                                            |
| System            | 4      | manage-memory, zoho-reauth, configure, poll-changes                                                                                                               |
| Activators        | 5      | activate-zoho-core, activate-zoho-extended, activate-github, activate-reporting, activate-settings                                                                |
| Meta              | 1      | list-tool-groups (shows available groups and activation status; isActivator: false)                                                                               |
| **Startup total** | **25** | 19 functional + 5 activators + 1 meta                                                                                                                             |

### Hidden Layer (13 tools)

| Layer      | Count | Visibility                                   |
| ---------- | ----- | -------------------------------------------- |
| Primitives | 13    | Hidden until their group activator is called |

**Total: 10 + 5 + 4 + 13 + 5 + 1 = 38 tools**

### Confirmation Pattern

9 of 10 orchestration tools use a **two-call confirmation pattern**: the first call returns a preview + token, the second call with the token executes. `monthly-summary` is the read-only exception (no confirmation needed).

### Extension Bypass

When `clientInfo.name === "freelance-hub-vscode"`, the client sees **33 tools** (all 38 minus the 5 `activate-*` activator tools). `list-tool-groups` is still included.

### Zoho ID Prefix

All Zoho entity IDs in tool outputs are `Z:`-prefixed (e.g., `Z:1234567890`).

---

## Activator Groups

| #   | Activator              | Group         | Tools Unlocked                                                                        |
| --- | ---------------------- | ------------- | ------------------------------------------------------------------------------------- |
| 1   | activate-zoho-core     | zoho-core     | manage-invoice, manage-client, manage-payment                                         |
| 2   | activate-zoho-extended | zoho-extended | manage-estimate, manage-credit-note, manage-recurring, manage-expense, manage-contact |
| 3   | activate-github        | github        | manage-milestone, manage-issue                                                        |
| 4   | activate-reporting     | reporting     | generate-report                                                                       |
| 5   | activate-settings      | settings      | lookup, manage-product                                                                |

**Activation rules:**

- **Monotonic** -- once a group is active, it stays active for the entire session. There is no deactivation.
- **Internal activation** -- orchestration tools activate groups silently when they need primitives internally. No user action required.
- **Idempotent** -- calling an activator for an already-active group succeeds with `alreadyActive: true`.

Use `list-tool-groups` to show the user what groups are available and their current activation status.

---

## Tool Selection Guide

Use orchestration tools first. They chain primitives automatically:

| User Intent        | Use This           | Not This                            |
| ------------------ | ------------------ | ----------------------------------- |
| "New client"       | onboard-client     | manage-client                       |
| "Start a project"  | start-project      | manage-milestone + manage-invoice   |
| "Milestone done"   | complete-milestone | manage-milestone + manage-invoice   |
| "Got paid"         | record-payment     | manage-payment                      |
| "Send reminder"    | send-reminder      | manage-invoice remind               |
| "Monthly report"   | monthly-summary    | manage-invoice list + manual math   |
| "Project finished" | close-project      | manage-milestone + manage-issue     |
| "Accept estimate"  | accept-estimate    | manage-estimate + manage-invoice    |
| "Issue credit"     | issue-credit       | manage-credit-note + manage-invoice |
| "Decline estimate" | decline-estimate   | manage-estimate                     |

Only activate primitive groups when the user needs direct entity manipulation that orchestration tools do not cover -- e.g., editing an invoice line item, manually closing a milestone, managing products, looking up Zoho settings.

---

## Autonomy Rules

### Autonomous (no confirmation needed)

All read/list/get operations, plus:

- standup, client-360, dashboard-snapshot, rate-review, monthly-summary
- manage-memory(action:append)
- manage-invoice(action:remind) via send-reminder
- poll-changes
- zoho-reauth
- list-tool-groups
- All activator tools

### Requires User Confirmation

| Tool                                             | Action                                         | Why                                                  |
| ------------------------------------------------ | ---------------------------------------------- | ---------------------------------------------------- |
| manage-invoice                                   | delete, write-off, mark-as-sent, mark-as-draft | Permanent or status-changing financial record change |
| manage-client                                    | create, deactivate                             | Creates or hides business records                    |
| manage-payment                                   | refund, update                                 | Financial reversal or modification                   |
| manage-memory                                    | write                                          | Full replace of memory file (append is safe)         |
| manage-milestone                                 | close                                          | Marks delivery complete                              |
| All orchestration tools (except monthly-summary) | any mutation                                   | Two-call confirmation pattern enforced by the tool   |

### Reactive Rule

If any mutation tool returns `requiresConfirmation: true`, present the preview to the user and wait for explicit approval before re-calling with the confirmation token. Never auto-confirm.

---

## Primitive Tool Notes

### manage-invoice (14 actions)

Actions: list, get, create, update, delete, void, send, remind, apply-credit, write-off, cancel-write-off, batch-create, **mark-as-sent**, **mark-as-draft**

| Action                        | When to Use                                                                                                                                                      |
| ----------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `mark-as-sent`                | Invoice delivered outside Zoho (manual email, PDF handoff, historical backfill). Changes draft→sent without emailing. Do NOT confuse with `send` (which emails). |
| `mark-as-draft`               | Revert sent/overdue invoice to draft for corrections. Does not work on paid or void invoices.                                                                    |
| `create` with `date`          | Set invoice date for backdating (default: today). `dueOn` calculated from `date` + `paymentTerms`.                                                               |
| `create` with `invoiceNumber` | Custom invoice number for migration. Requires `ignoreAutoNumberGeneration: true`.                                                                                |

### manage-payment (6 actions)

Actions: list, get, create, **update**, delete, refund

| Action   | When to Use                                                                                                  |
| -------- | ------------------------------------------------------------------------------------------------------------ |
| `update` | Fix payment method, date, invoice linkage, or reference. Preserves payment ID — prefer over delete+recreate. |

Field mapping: input `method` → Zoho API `payment_mode`. Accepts `description` (free-text) on create/update for payment context. No `exchangeRate` input — Zoho Invoice free plan is single-currency (AED).

---

## Known Pitfalls

Constraints to design your call around. Where a pitfall lists a workaround, prefer it over retrying the failing call.

### Schema constraints

- **`manage-client(action:update)` is replace-shaped, not patch-shaped.** Always include `clientId` + `clientName` + `email` even when only changing one field (e.g. `phone`). If you only have the changed field from the user, first `get` the client to read the current `clientName` / `email`, then send the full triple plus your delta.
- **`manage-recurring(action:update)` does not accept `amount`.** To change the amount, send `lineItems: [{ description, quantity, rate }]` instead. The `amount` key is only valid on `create`.
- **`manage-credit-note(action:remove)` requires `creditNoteInvoiceId`, not `invoiceId`.** Workflow: `apply` → `get` the credit note → read `appliedInvoices[].creditNoteInvoiceId` → `remove` with that ID. Sending `invoiceId` is rejected with `Unknown keys: [invoiceId]`.
- **`lookup` enum is narrow.** Valid `action` values: `currencies`, `taxes`, `organizations`. `payment-terms` and `chart-of-accounts` are not exposed — derive payment terms from invoice/estimate fields instead.
- **`manage-memory` archetype rules:**
  - `key` is only valid for `action:read`. For other actions (e.g. `upsert-entry`), use `entry.id` or namespace-specific identifiers — do not pass `key`.
  - `action:list` is rejected on append-log namespaces (`comms`). Use `action:list-entries` for those; `list` is for object/single-doc namespaces.

### Prefer the working path

- **Sending an invoice: prefer `mark-as-sent` over `send` for non-email delivery.** `manage-invoice(action:send)` can fail with HTTP 400 / Zoho code 1000 on invoices where `mark-as-sent` succeeds. Use `send` only when the user explicitly asks to email; for "just mark it sent" / historical / manual-delivery cases use `mark-as-sent`.
- **Recording a payment: prefer `record-payment` (orchestration) over `manage-payment(action:create)`.** The direct primitive can return Zoho code 24016 ("amount more than balance due") on invoices the orchestration path accepts. `record-payment` also runs the confirmation flow and links the payment to project context — make it the default.

### Verify writes that may silently drop fields

- **`manage-expense(action:update)` may not persist `notes`.** A successful response can return `notes: ""` even when the request set a value. After updating notes, `get` the expense and verify; if blank, surface the discrepancy to the user rather than claiming the note was saved.

---

## Currency & FX Context

Zoho Invoice free plan is **single-currency** (AED for this org). All invoices, estimates, credit notes, and payments are AED-only. `currencyId` and `exchangeRate` are NOT accepted by any tool — Zoho ignores or rejects them on the free plan.

**For international clients**, stamp FX context on free-text fields that Zoho preserves verbatim:

| Tool                                          | Fields                               |
| --------------------------------------------- | ------------------------------------ |
| `start-project`                               | `invoiceNotes`, `invoiceTerms`       |
| `accept-estimate`                             | `invoiceNotes`, `invoiceTerms`       |
| `issue-credit`                                | `creditNoteNotes`, `creditNoteTerms` |
| `manage-invoice(create/update/batch-create)`  | `notes`, `terms`, `customFields`     |
| `manage-estimate(create/update/batch-create)` | `notes`, `terms`, `customFields`     |
| `manage-credit-note(create/update)`           | `notes`, `terms`, `customFields`     |
| `manage-payment(create/update)`               | `description`                        |

**Example:** `invoiceNotes: "Equivalent to USD 5,000 at AED/USD 3.6725 as of 2026-04-20."`

`customFields` is an object of `{ customFieldKey: value }` pairs for org-defined custom fields on Zoho entities. Use for structured metadata beyond notes/terms.

**Never** ask the user for `exchangeRate` or `currencyId` — these inputs have been removed from all tools.
