# Business Workflows

Concrete step-by-step procedures for every recurring operation. Each workflow shows what the user says, what the AI does, and the exact tool calls in order.

All Zoho entity IDs in tool outputs are Z:-prefixed (e.g., `Z:8809863000000097002`). All monetary values in data fields use raw numbers (not formatted strings) unless otherwise noted.

---

## Currency & FX Context

This org uses Zoho Invoice free plan (single-currency, AED). All amounts on invoices, estimates, credit notes, and payments are in AED — Zoho does not support multi-currency on this plan.

**For international clients**, communicate FX context using the invoice's Customer Notes and Terms & Conditions free-text fields:

- On `start-project` and `accept-estimate`, use `invoiceNotes` and `invoiceTerms` parameters to stamp FX context on every milestone invoice.
- On `issue-credit`, use `creditNoteNotes` and `creditNoteTerms`.
- For post-creation updates, use `manage-invoice { action: 'update', notes, terms }`.

**Example:** `invoiceNotes: "Equivalent to USD 5,000 at AED/USD 3.6725 as of 2026-04-20."`

**Never** ask the user for an `exchangeRate` or `currencyId` — these inputs have been removed from all tools because Zoho Invoice free plan ignores them.

---

## 1. New Client Onboarding

**Trigger**: User says "new client", "I have a new client", "onboard", "add client", or provides client details unprompted.

| Step | Action                                                                                                    | Tool Call                                                                                                                                                                  |
| ---- | --------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1    | Collect: name, email, companyName (optional), phone (optional), paymentTerms (optional), notes (optional) | -- (ask user)                                                                                                                                                              |
| 2    | Create Zoho contact and log onboarding                                                                    | `onboard-client(name, email, companyName?, phone?, paymentTerms?, notes?)`                                                                                                 |
| 3    | Confirmation preview returned -- present to user, wait for approval                                       | -- (show preview: name, email, company, paymentTerms)                                                                                                                      |
| 4    | Re-call with confirmation token to execute                                                                | `onboard-client(name, email, companyName?, paymentTerms?, confirmationToken)`                                                                                              |
| 5    | Log onboarding to comms memory                                                                            | `manage-memory(action:upsert-entry, namespace:comms, entry:{ts:"[ISO]", channel:"other", direction:"out", summary:"Onboarded [client]. Terms: [terms]. Email: [email]."})` |
| 6    | Suggest next step                                                                                         | "Ready to start a project? Provide scope and value."                                                                                                                       |

**Confirmation**: `onboard-client` uses the two-call confirmation pattern. First call returns a `ConfirmationRequest` with preview + single-use token (5-minute TTL, session-scoped). Second call with the `confirmationToken` executes.

**Field notes**:

- `name` is the client's full name (person, not company). Maps to Zoho `contact_name`.
- `companyName` (not `company`) is the company or organization name.
- `paymentTerms` accepts a string label (e.g., "Net 30", "Due on Receipt"). The tool converts it to the Zoho integer format internally. When omitted, Zoho applies the org default.
- Duplicate name or email triggers a `duplicateWarning` in the preview -- not a hard error. Present the warning and let user decide.

---

## 2. New Project Setup

**Trigger**: User says "new project", "start a project for [client]", or provides scope + value.

| Step | Action                                                                                                        | Tool Call                                                                                                                                                                                                                          |
| ---- | ------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1    | Collect: project name, client, total value, milestone split strategy or custom milestones, startDate, endDate | -- (ask user)                                                                                                                                                                                                                      |
| 2    | Propose milestone split based on value (see defaults below)                                                   | -- (present table, get user approval)                                                                                                                                                                                              |
| 3    | Create GitHub milestones + draft invoices                                                                     | `start-project(project, client, totalValue, splitStrategy?, milestones?:[...], sendFirst?, repo?, invoiceNotes?, invoiceTerms?)`                                                                                                   |
| 4    | Confirmation preview returned -- present sum check, milestone breakdown, sendFirst flag                       | -- (wait for approval)                                                                                                                                                                                                             |
| 5    | Re-call with confirmation token                                                                               | `start-project(project, client, totalValue, ..., confirmationToken)`                                                                                                                                                               |
| 6    | Log new project to projects memory                                                                            | `manage-memory(action:upsert-entry, namespace:projects, entry:{projectName:"[name]", contactId:"[clientId]", status:"active", billingMode:"fixed", fixedAmount:[totalValue], scopeNotes:"Split: [breakdown]", startedAt:"[ISO]"})` |
| 7    | First milestone invoice sent automatically                                                                    | Unless `sendFirst: false` was specified                                                                                                                                                                                            |

### Milestone Split Defaults

| Project Value          | Default Split Strategy     |
| ---------------------- | -------------------------- |
| <= AED 2,000           | 50/50 (2 milestones)       |
| AED 2,001 - AED 10,000 | 30/30/40 (3 milestones)    |
| > AED 10,000           | 25/25/25/25 (4 milestones) |

Named strategies: `"50/50"`, `"30/30/40"`, `"25/25/25/25"`, `"equal"`, `"custom"`. If `"custom"`, provide `milestones` array where amounts MUST sum to `totalValue`.

**Rollback**: If creation fails mid-flow, `start-project` automatically rolls back: invoices are voided, milestones are deleted. Do not attempt manual cleanup.

**Invoice references**: Each draft invoice gets `reference_number = "{project} — {milestone.name}"`. This em-dash convention is used by `complete-milestone` and `close-project` to match invoices to milestones. Legacy colon format (`"{project}:{milestone.name}"`) is also recognized during parsing for backward compatibility.

---

## 3. Milestone Completion

**Trigger**: User says "milestone done", "finished the backend", "completed phase 2", "ready to invoice for X".

| Step | Action                                                                                                                               | Tool Call                                                                                                                                                                                              |
| ---- | ------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 1    | Resolve milestone by fuzzy match against project prefix                                                                              | -- (handled internally)                                                                                                                                                                                |
| 2    | Close GitHub issues + milestone, find linked draft invoice by reference_number, auto-attach unbilled billable expenses, send invoice | `complete-milestone(project, milestone, force?, repo?)`                                                                                                                                                |
| 3    | Confirmation preview returned -- shows milestone name, open issues count, invoice amount, recipient                                  | -- (wait for approval)                                                                                                                                                                                 |
| 4    | Re-call with confirmation token                                                                                                      | `complete-milestone(project, milestone, confirmationToken)`                                                                                                                                            |
| 5    | Log to comms memory                                                                                                                  | `manage-memory(action:upsert-entry, namespace:comms, entry:{ts:"[ISO]", channel:"other", direction:"out", summary:"Milestone [name] completed for [project]. Invoice #[number] sent: AED X,XXX.XX."})` |
| 6    | Show summary                                                                                                                         | Invoice amount sent, remaining milestones with their invoice statuses, outstanding balance                                                                                                             |

**If open issues exist**: Tool returns the list for review. User can re-call with `force: true` to close all issues and proceed, or resolve issues first.

**Expenses**: Unbilled billable expenses for the project are auto-detected and auto-attached as invoice line items at execution time. No expense IDs need to be passed as input. Before completing, if the user has mentioned billable expenses, verify they are recorded by calling `manage-expense(action:list, projectId, isBillable:true)` first.

**Note**: `complete-milestone` does NOT accept billing data as input -- expense attachment is fully automatic. All unbilled billable expenses for the project are attached (no date filtering), regardless of which milestone they relate to.

**Invoice edge cases**:

- Invoice already sent (not draft): milestone closes normally, no invoice action taken.
- Invoice voided: milestone closes, warning issued. Suggest creating a new invoice manually.
- Multiple drafts for same reference: data integrity error -- resolve manually.

---

## 4. Payment Recording

**Trigger**: User says "got paid", "received payment", "[client] paid", "[client] paid AED 450", or provides payment details.

| Step | Action                                                                                               | Tool Call                                                                                                                                                                                                                          |
| ---- | ---------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1    | Record payment against resolved invoice                                                              | `record-payment(amount, date, invoiceId?, invoiceRef?, client?, method?, notes?)`                                                                                                                                                  |
| 2    | Confirmation preview returned -- shows amount, client, invoice details, resulting balance and status | -- (wait for approval)                                                                                                                                                                                                             |
| 3    | Re-call with confirmation token                                                                      | `record-payment(amount, date, ..., confirmationToken)`                                                                                                                                                                             |
| 4    | Log to comms memory                                                                                  | `manage-memory(action:upsert-entry, namespace:comms, entry:{ts:"[ISO]", channel:"other", direction:"in", summary:"Payment received from [client]: AED X,XXX.XX. Applied to invoice #[number]. Remaining balance: AED X,XXX.XX."})` |
| 5    | Show updated balance + next milestone due                                                            | -- (from tool response)                                                                                                                                                                                                            |
| 6    | If all invoices for a project are paid                                                               | Suggest: "All milestones invoiced and paid. Run `close-project`?"                                                                                                                                                                  |

**Invoice resolution priority** (highest first):

1. `invoiceId` -- exact Zoho ID (direct lookup)
2. `invoiceRef` -- reference number like "WebRedesign:Backend" (exact match)
3. `client` -- finds oldest unpaid sent invoice for this client (fuzzy match)

At least one of `invoiceId`, `invoiceRef`, or `client` is required.

**Guards**:

- Payment amount must not exceed invoice balance. Overpayments are rejected with AMOUNT_EXCEEDED.
- Payments against draft invoices are rejected -- send the invoice first.
- Payments against void or paid invoices are rejected.
- Amount is always in AED (the org's only currency on Zoho Invoice free plan). If the client paid in a foreign currency, convert to AED at the payment-date rate and record the AED amount; note the FX context in the payment `notes` or the invoice's `notes`/`terms`.

**Partial payments**: Invoice auto-transitions to `partially_paid`. Follow up with client for the remainder.

---

## 5. Payment Reminder Escalation

**Trigger**: Overdue invoice detected in standup, or user says "send a reminder to [client]", "follow up on invoice #X", "any reminders I should send?".

### Escalation Tier Table

| Days Overdue | Level      | Subject Pattern                                                 | Follow-up                           |
| ------------ | ---------- | --------------------------------------------------------------- | ----------------------------------- |
| 0            | EXCLUDED   | Invoice due today -- not yet overdue. Must NOT be included.     | --                                  |
| 1-7 days     | Friendly   | "Friendly reminder: Invoice {number}"                           | Follow up in 7 days                 |
| 8-21 days    | Firm       | "Follow-up: Invoice {number} -- {days} days past due"           | Follow up in 14 days                |
| 22+ days     | Escalation | "Overdue notice: Invoice {number} requires immediate attention" | No automated follow-up -- see below |

The escalation level is determined automatically by days overdue. The AI does not choose the tone.

### Procedure

| Step | Action                                                                    | Tool Call                                                                                                                                                                                                                    |
| ---- | ------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1    | Preview reminder(s) with dry run                                          | `send-reminder(client?, invoiceId?, dryRun:true)`                                                                                                                                                                            |
| 2    | Present preview to user (messages composed, not sent)                     | -- (show email subject + body per invoice)                                                                                                                                                                                   |
| 3    | User approves                                                             | -- (wait for "yes", "send it", etc.)                                                                                                                                                                                         |
| 4    | Call send-reminder without dryRun -- returns confirmation preview + token | `send-reminder(client?, invoiceId?)`                                                                                                                                                                                         |
| 5    | Re-call with confirmation token to send                                   | `send-reminder(client?, invoiceId?, confirmationToken)`                                                                                                                                                                      |
| 6    | Log to comms memory                                                       | `manage-memory(action:upsert-entry, namespace:comms, entry:{ts:"[ISO]", channel:"email", direction:"out", subject:"Reminder", summary:"Reminder sent to [client] for invoice #[number]. Tier: [tier]. Days overdue: [N]."})` |

**Dry run flow**: `dryRun:true` returns immediately with `sent: false` on all items. No confirmation token is issued. The total flow from dry run to execution is 3 calls: (1) dryRun preview, (2) confirmation preview (returns token), (3) confirmed execution (with token).

**If neither client nor invoiceId specified**: Finds ALL overdue invoices across all clients.

**Escalation tier (22+ days) continuation paths** -- the AI must present all three and wait for user choice:

1. **Direct client call**: No automated tool. Offer to log the outcome via `manage-memory(action:upsert-entry, namespace:comms, entry:{ts, channel:"call", direction:"out", summary})`.
2. **Write-off**: `manage-invoice(action:write-off, invoiceId)`. After write-off, check if all project invoices are at AED 0 balance; if so, suggest `close-project`.
3. **External collections**: No automated tool. Log the referral via `manage-memory(action:upsert-entry, namespace:comms, entry:{ts, channel:"other", direction:"out", summary})`.

If user declines all three, log the decision. The invoice remains as-is. Future `send-reminder` calls will re-present the escalation menu (stateless across invocations).

---

## 6. Estimate Lifecycle

**Trigger**: User says "create an estimate", "send a quote to [client]", or "the estimate was accepted/declined".

### Creating and Sending

| Step | Action                           | Tool Call                                                              |
| ---- | -------------------------------- | ---------------------------------------------------------------------- |
| 1    | Activate zoho-extended if needed | `activate-zoho-extended`                                               |
| 2    | Create estimate                  | `manage-estimate(action:create, clientName, items:[...], expiryDate?)` |
| 3    | Send to client                   | `manage-estimate(action:send, estimateId)`                             |

### On Acceptance

| Step | Action                                                                                  | Tool Call                                                                                                                                                                                                    |
| ---- | --------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 1    | Accept estimate and set up project in one call                                          | `accept-estimate(estimateId?, estimateRef?, client?, splitStrategy?, milestones?, sendFirst?, invoiceNotes?, invoiceTerms?)`                                                                                 |
| 2    | Confirmation preview returned -- shows estimate details, milestone breakdown, sum check | -- (wait for approval)                                                                                                                                                                                       |
| 3    | Re-call with confirmation token                                                         | `accept-estimate(..., confirmationToken)`                                                                                                                                                                    |
| 4    | Log to comms memory                                                                     | `manage-memory(action:upsert-entry, namespace:comms, entry:{ts:"[ISO]", channel:"other", direction:"in", summary:"Estimate #[number] accepted by [client]. Value: AED X,XXX.XX. Project setup initiated."})` |

**Estimate resolution**: Three-way resolution -- `estimateId` (direct), `estimateRef` (reference), or `client` (most recent pending). At least one required.

**Milestone source precedence**: (1) explicit `milestones` array overrides everything, (2) `splitStrategy` overrides estimate line items, (3) otherwise estimate line items become milestones directly.

**Currency**: All estimates and resulting milestone invoices are in AED (Zoho Invoice free plan is single-currency). For international clients, stamp FX context on each milestone invoice via `invoiceNotes` / `invoiceTerms` — e.g., `invoiceNotes: "Equivalent to USD 5,000 at AED/USD 3.6725 as of 2026-04-20."` Do not pass `currencyId` or `exchangeRate` (removed from tool surface).

**Partial failure**: If estimate acceptance succeeds but project setup fails, the estimate status is "accepted" -- do NOT re-call `accept-estimate`. Call `start-project` directly to rebuild infrastructure.

### On Decline

| Step | Action                                          | Tool Call                                                                                                                                                                        |
| ---- | ----------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1    | Decline estimate with reason, optionally revise | `decline-estimate(estimateId?, estimateRef?, client?, reason, revise?)`                                                                                                          |
| 2    | Confirmation preview returned                   | -- (wait for approval)                                                                                                                                                           |
| 3    | Re-call with confirmation token                 | `decline-estimate(..., confirmationToken)`                                                                                                                                       |
| 4    | Log to comms memory                             | `manage-memory(action:upsert-entry, namespace:comms, entry:{ts:"[ISO]", channel:"other", direction:"in", summary:"Estimate #[number] declined by [client]. Reason: [reason]."})` |
| 5    | Offer revision                                  | "Would you like to create a revised estimate with adjusted scope or pricing?"                                                                                                    |

**Revision in one call**: `decline-estimate` supports a `revise` parameter with `adjustments` (modify existing line items), `addItems` (add new line items), `notes`, and `sendImmediately`. This declines the original and creates a revised estimate in one operation.

---

## 7. Estimate Revision

**Trigger**: Estimate was declined and user wants to revise, or user says "revise the estimate", "lower the quote".

| Step | Action                                                                                  | Tool Call                                                                                                                                                                                                                                                                            |
| ---- | --------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 1    | Review original estimate                                                                | `manage-estimate(action:get, estimateId)`                                                                                                                                                                                                                                            |
| 2    | Check rate card floor                                                                   | `manage-memory(action:read, namespace:rates)`                                                                                                                                                                                                                                        |
| 3    | Discuss scope changes with user                                                         | -- (present options: reduce scope, adjust rate, phase delivery)                                                                                                                                                                                                                      |
| 4    | Decline and revise in one call                                                          | `decline-estimate(estimateId, reason, revise:{adjustments:[...], addItems:[...], notes?, sendImmediately?})`                                                                                                                                                                         |
| 5    | Or create revised estimate separately                                                   | `manage-estimate(action:create, clientName, items:[revised], referenceEstimate?)`                                                                                                                                                                                                    |
| 6    | Send revised estimate                                                                   | `manage-estimate(action:send, estimateId)`                                                                                                                                                                                                                                           |
| 7    | Log revised estimate to comms memory                                                    | `manage-memory(action:upsert-entry, namespace:comms, entry:{ts:"[ISO]", channel:"email", direction:"out", subject:"Revised estimate #[number]", summary:"Revised estimate #[number] sent to [client]. Original: AED X,XXX.XX -> Revised: AED X,XXX.XX. Scope changes: [summary]."})` |
| 8    | Log rate impact (rates is single-document; use upsert-entry on the rate-card singleton) | `manage-memory(action:upsert-entry, namespace:rates, entry:{defaultHourlyRate:[N], currency:"AED", lastReviewedAt:"[ISO]"})`                                                                                                                                                         |

**Guard**: If the revised rate falls below the rate card floor, WARN the user before proceeding. The floor is a hard minimum -- never accept below it without explicit user override and logged exception.

---

## 8. Credit / Adjustment

**Trigger**: User says "issue a credit", "we overcharged [client]", "apply a credit note", "billing correction".

| Step | Action                                                                                                                      | Tool Call                                                                                                                                                                                                                   |
| ---- | --------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1    | Identify the overcharge or correction                                                                                       | -- (ask user for amount, reason, target invoice if any)                                                                                                                                                                     |
| 2    | Issue credit note                                                                                                           | `issue-credit(client, amount, reason, invoiceId?, invoiceRef?, lineItems?, creditNoteNotes?, creditNoteTerms?)`                                                                                                             |
| 3    | Confirmation preview returned -- shows client, amount, reason, line items, and if applying to invoice: before/after balance | -- (wait for approval)                                                                                                                                                                                                      |
| 4    | Re-call with confirmation token                                                                                             | `issue-credit(client, amount, reason, ..., confirmationToken)`                                                                                                                                                              |
| 5    | Log to comms memory                                                                                                         | `manage-memory(action:upsert-entry, namespace:comms, entry:{ts:"[ISO]", channel:"other", direction:"out", summary:"Credit note issued for [client]: AED X,XXX.XX. Reason: [reason]. Applied to: [invoice or unapplied]."})` |
| 6    | Show updated balance                                                                                                        | -- (from tool response)                                                                                                                                                                                                     |

**Key notes**:

- Do NOT include `creditnote_number` -- Zoho auto-generates it. Orgs with auto-numbering reject custom numbers.
- `lineItems` uses `{ description, amount }` format (NOT `{ description, rate, quantity }`). The tool converts internally.
- Line item amounts must be positive and must sum exactly to the top-level credit `amount`.
- If `invoiceId` or `invoiceRef` provided, the credit is applied to that invoice. Amount must not exceed invoice balance.
- If no invoice specified, the credit is created as unapplied -- it remains on the client's account for future use. No upper limit on amount, but a warning is shown if it exceeds the client's total billing history.

**Applied vs. Unapplied**: `issue-credit` always applies the FULL credit amount to the specified invoice. For partial application across multiple invoices, use `manage-credit-note(action:apply)` directly.

---

## 9. Monthly Close

**Trigger**: End of month, or user says "monthly review", "how did this month go", "month-end summary".

| Step | Action                              | Tool Call                                                                                                                                                                                                                                                                      |
| ---- | ----------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 1    | Generate monthly summary            | `monthly-summary(month?, client?)`                                                                                                                                                                                                                                             |
| 2    | Present key metrics                 | Invoiced vs collected, collection rate, outstanding AR, expenses, net revenue, profit margin, 30-day forecast, recurring revenue status                                                                                                                                        |
| 3    | Review action items                 | Overdue invoices to chase, milestones approaching deadline, rate reviews needed                                                                                                                                                                                                |
| 4    | Send reminders for overdue invoices | Follow Reminder Escalation workflow for each                                                                                                                                                                                                                                   |
| 5    | Log to comms memory                 | `manage-memory(action:upsert-entry, namespace:comms, entry:{ts:"[ISO]", channel:"other", direction:"out", summary:"Monthly close completed. Invoiced: AED X,XXX.XX. Collected: AED X,XXX.XX. Collection rate: XX.X%. Outstanding: AED X,XXX.XX. Net revenue: AED X,XXX.XX."})` |
| 6    | End with one prioritized action     | The single most impactful thing to do next month                                                                                                                                                                                                                               |

**`monthly-summary` is read-only** -- no confirmation required. It is the only orchestration tool that does not use the confirmation pattern.

**Financial details**:

- `collectionRate` is a ratio 0.0-1.0 in data (e.g., 0.685). Display it as a percentage (e.g., "68.5%") using the pre-computed `collectionRatePercent` field.
- `collectionRate > 1.0` is valid -- indicates cross-period payments (payments for prior-month invoices received this month). Display as "100%+".
- `collectionRate = null` means no invoiced revenue in scope. Omit the line rather than showing an error.
- `totalPaid` in close-project uses the write-off-aware formula: `sum(Math.max(0, invoice.total - invoice.balance - invoice.write_off_amount))`. Write-offs are excluded from collected revenue.
- `profitMargin = netRevenue / invoiced.total`. Can be negative (loss-making period). Null when no invoiced revenue.
- Void and draft invoices are excluded from all financial calculations.

**Prior period comparison**: `comparedToPrior` shows absolute deltas for invoiced, collected, and collection rate vs. the prior month.

---

## 10. Project Close

**Trigger**: User says "close the project", "project is done", "wrap up [project name]".

| Step | Action                                                                               | Tool Call                                                                                                                                                                                                                                                                                           |
| ---- | ------------------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1    | Close project -- verifies all milestones, invoices, calculates financial summary     | `close-project(project, client?, lessons?, totalHours?, repo?)`                                                                                                                                                                                                                                     |
| 2    | Confirmation preview returned -- shows financial summary, open items, draft invoices | -- (wait for approval)                                                                                                                                                                                                                                                                              |
| 3    | Re-call with confirmation token                                                      | `close-project(project, client?, lessons?, totalHours?, confirmationToken)`                                                                                                                                                                                                                         |
| 4    | Log to comms memory                                                                  | `manage-memory(action:upsert-entry, namespace:comms, entry:{ts:"[ISO]", channel:"other", direction:"out", summary:"Project [name] closed for [client]. Total value: AED X,XXX.XX. Effective rate: AED XXX.XX/hr. Lessons: [summary]."})`                                                            |
| 5    | Mark project closed in projects memory                                               | `manage-memory(action:upsert-entry, namespace:projects, entry:{projectName:"[project]", contactId:"[clientId]", status:"closed", billingMode:"fixed", fixedAmount:[finalValue], scopeNotes:"Closed. Effective rate: AED XXX.XX/hr vs quoted AED XXX.XX/hr.", startedAt:"[ISO]", closedAt:"[ISO]"})` |
| 6    | Lessons learned                                                                      | Ask user: "Any lessons learned or notes for future projects with this client?" Append response to rates memory.                                                                                                                                                                                     |

**`client` parameter**: Always include the `client` parameter if you know the client name. This is REQUIRED when invoices don't use the standard `{project}:milestone` reference_number format. If omitted AND no invoices match by reference_number prefix, the tool returns MISSING_REQUIRED with a candidates list of clients that have invoices. Re-call with the `client` parameter:

```
close-project(project: "ProjectName", client: "ClientName")
```

**Completeness rules**: A project is `complete: true` ONLY when ALL three conditions are met:

1. All milestones closed
2. All non-void, non-draft invoices paid (balance = AED 0.00)
3. No draft invoices exist for the project

**Draft invoice awareness**:

- `draftValue` reports the sum of draft invoice totals. `draftInvoiceCount` reports how many.
- Draft invoices are EXCLUDED from `totalValue`, `totalPaid`, and `outstandingAmount` -- they represent uncommitted billing.
- Draft invoices BLOCK project closure. They must be sent, voided, or deleted before closing.
- If `draftInvoiceCount > 0`: STOP. Do not proceed with closure. Present each draft invoice number and amount. Ask: "Should I send these draft invoices to the client, or should they be deleted?"

**Financial formulas**:

- `totalValue` = sum of invoice.total for sent/overdue/partially_paid/paid invoices (not void, not draft).
- `totalPaid` = `sum(Math.max(0, invoice.total - invoice.balance - invoice.write_off_amount))` -- write-offs excluded from collected revenue, per-invoice clamping prevents negative contributions.
- `totalExpenses` = sum of all project-linked expense totals (post-tax, both billable and non-billable, excluding void/deleted).
- `netRevenue` = `totalPaid - totalExpenses` (can be negative).
- `effectiveRate` = `(totalPaid + outstandingAmount) / totalHours` when `totalHours` provided; null when absent. Projects as-if-collected. Also null when zero projected revenue.
- If `totalHours` is omitted, `effectiveRate` is null — provide hours from external tracker.
- `effectiveRateIsProjected` = true when any non-void/non-draft invoice has balance > 0.
- `profitMargin` = `netRevenue / totalValue`. Null when totalValue = 0. Can be negative.

**Write-off treatment**: An invoice with balance = AED 0.00 via write-off counts as financially closed. The write-off amount is visible in the output but excluded from `totalPaid`.

**Unbilled expenses**: If billable expenses have no matching invoice line item, the tool flags them. Create a final invoice or write them off before closing.

---

## 11. Recurring Invoicing

**Trigger**: User says "set up recurring billing", "monthly retainer invoice", "automate invoicing for [client]".

| Step | Action                           | Tool Call                                                                                                                                                                                                                                                       |
| ---- | -------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1    | Activate zoho-extended           | `activate-zoho-extended`                                                                                                                                                                                                                                        |
| 2    | Create recurring invoice profile | `manage-recurring(action:create, clientName, items:[...], frequency, startDate, endDate?)`                                                                                                                                                                      |
| 3    | Verify profile is active         | `manage-recurring(action:get, recurringId)`                                                                                                                                                                                                                     |
| 4    | Log to comms memory              | `manage-memory(action:upsert-entry, namespace:comms, entry:{ts:"[ISO]", channel:"other", direction:"out", summary:"Recurring invoice set up for [client]. Amount: AED X,XXX.XX/[frequency]. Start: [date]."})`                                                  |
| 5    | Record retainer as project entry | `manage-memory(action:upsert-entry, namespace:projects, entry:{projectName:"[client]-retainer", contactId:"[clientId]", status:"active", billingMode:"retainer", scopeNotes:"Retainer AED X,XXX.XX/[frequency]. Covers: [scope summary].", startedAt:"[ISO]"})` |

### Monitoring Recurring Invoices

| Action                             | Tool Call                                                     |
| ---------------------------------- | ------------------------------------------------------------- |
| List all active recurring profiles | `manage-recurring(action:list)`                               |
| Pause a recurring profile          | `manage-recurring(action:update, recurringId, status:paused)` |
| Resume a paused profile            | `manage-recurring(action:update, recurringId, status:active)` |
| Stop recurring billing             | `manage-recurring(action:delete, recurringId)`                |

**Monthly summary integration**: `monthly-summary` reports on recurring revenue status (paid/overdue/upcoming per profile) and detects recurring gaps (missing or late invoices vs. expected schedule).

---

## 12. Year-End Summary

**Trigger**: End of year, or user says "year in review", "annual summary", "how was this year".

| Step | Action                      | Tool Call                                                                                                                                                                                                                                                  |
| ---- | --------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1    | Activate reporting group    | `activate-reporting`                                                                                                                                                                                                                                       |
| 2    | Generate annual report      | `generate-report(period:annual, year)`                                                                                                                                                                                                                     |
| 3    | Present summary metrics     | Total revenue, expenses by category, net income, top clients by revenue                                                                                                                                                                                    |
| 4    | Review expenses by category | `manage-expense(action:list, fromDate:[year]-01-01, toDate:[year]-12-31)`                                                                                                                                                                                  |
| 5    | Identify top clients        | Rank by total invoiced amount, show payment reliability                                                                                                                                                                                                    |
| 6    | Log to comms memory         | `manage-memory(action:upsert-entry, namespace:comms, entry:{ts:"[ISO]", channel:"other", direction:"out", summary:"Year-end review for [year]. Revenue: AED XX,XXX.XX. Expenses: AED X,XXX.XX. Net: AED XX,XXX.XX. Top client: [name] (AED XX,XXX.XX)."})` |
| 7    | Refresh rate-card singleton | `manage-memory(action:upsert-entry, namespace:rates, entry:{defaultHourlyRate:[N], currency:"AED", lastReviewedAt:"[ISO]"})`                                                                                                                               |
| 8    | Recommend rate adjustments  | Compare effective rates to rate card; suggest increases where effective rate consistently exceeds quoted rate                                                                                                                                              |

**Requires**: Both `zoho-extended` (for expenses) and `reporting` (for `generate-report`) groups to be activated.

---

## 13. Daily Session Opener (Standup)

**Trigger**: First interaction of the day, "standup", "what happened?", "catch me up", "morning briefing", "what's new since last time?"

| Step | Action                                                                                | Tool Call                                                |
| ---- | ------------------------------------------------------------------------------------- | -------------------------------------------------------- |
| 1    | Run standup                                                                           | `standup(since?)`                                        |
| 2    | Lead with `sessionOpener` as the opening sentence (present verbatim, do not rephrase) | -- (from tool response)                                  |
| 3    | If blockers exist, present them immediately after the opener                          | --                                                       |
| 4    | Summarize "what happened" since last session                                          | -- (payments received, invoices sent/paid, comms logged) |
| 5    | Present "what's next" as prioritized action items                                     | -- (sorted high -> medium -> low, with suggested tools)  |
| 6    | If overdue invoices appear, suggest send-reminder                                     | --                                                       |

**Notes**:

- `standup` is the recommended session opener. It reads the last-session timestamp from memory and reports changes since then.
- `since` parameter is optional -- defaults to time since last session, or 72 hours if first session.
- The tool writes only a session timestamp (bookkeeping, not business mutation).
- If `happened` is empty, omit the "what happened" section -- do NOT say "nothing happened".

---

## 14. Mark Invoice as Sent (Historical / Manual Delivery)

**Trigger**: User says "mark as sent", "I already sent this invoice", "this invoice was delivered manually", or is backfilling historical invoices.

| Step | Action                            | Tool Call                                                                                                                                                                                                                 |
| ---- | --------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1    | Activate zoho-core if needed      | `activate-zoho-core`                                                                                                                                                                                                      |
| 2    | Identify the invoice              | `manage-invoice(action:get, invoiceId)` or `manage-invoice(action:list, referenceNumber)`                                                                                                                                 |
| 3    | Verify invoice is in draft status | -- (check status field)                                                                                                                                                                                                   |
| 4    | Mark as sent                      | `manage-invoice(action:mark-as-sent, invoiceId)`                                                                                                                                                                          |
| 5    | Log to comms memory               | `manage-memory(action:upsert-entry, namespace:comms, entry:{ts:"[ISO]", channel:"other", direction:"out", summary:"Invoice #[number] marked as sent (delivered outside Zoho). Client: [client]. Amount: AED X,XXX.XX."})` |

**Key distinction**: `mark-as-sent` changes status to "sent" WITHOUT emailing. `send` emails the PDF to the client. If the user says "send", clarify: "Do you want to email this invoice to the client, or just mark it as sent because it was already delivered?"

**Pre-condition**: Invoice must be in `draft` status. If already sent/overdue/paid, no action needed.

---

## 15. Fix Payment Details

**Trigger**: User says "fix the payment method", "change payment to bank transfer", "wrong payment date", "relink this payment to a different invoice".

| Step | Action                       | Tool Call                                                                                                                                                                               |
| ---- | ---------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1    | Activate zoho-core if needed | `activate-zoho-core`                                                                                                                                                                    |
| 2    | Identify the payment         | `manage-payment(action:get, paymentId)` or `manage-payment(action:list, clientId)`                                                                                                      |
| 3    | Update the payment           | `manage-payment(action:update, paymentId, method?, date?, invoiceId?, amount?, reference?, notes?)`                                                                                     |
| 4    | Verify the update            | -- (check response fields match expected values)                                                                                                                                        |
| 5    | Log if significant           | `manage-memory(action:upsert-entry, namespace:comms, entry:{ts:"[ISO]", channel:"other", direction:"out", summary:"Payment #[number] updated for [client]. Changed: [what changed]."})` |

**Prefer update over delete+recreate**: Update preserves the payment ID and audit trail. Only delete+recreate if the Zoho API rejects the update.

**Field mapping**: Input `method` maps to Zoho's `payment_mode` field. Common values: "bank_transfer", "cash", "check", "credit_card".

---

## 16. Revert Invoice to Draft

**Trigger**: User says "revert to draft", "need to fix this invoice", "undo the send", "make it editable again".

| Step | Action                                               | Tool Call                                                                                    |
| ---- | ---------------------------------------------------- | -------------------------------------------------------------------------------------------- |
| 1    | Activate zoho-core if needed                         | `activate-zoho-core`                                                                         |
| 2    | Identify the invoice                                 | `manage-invoice(action:get, invoiceId)`                                                      |
| 3    | Verify invoice is sent or overdue (not paid or void) | -- (check status)                                                                            |
| 4    | Revert to draft                                      | `manage-invoice(action:mark-as-draft, invoiceId)`                                            |
| 5    | Make corrections                                     | `manage-invoice(action:update, invoiceId, lineItems?, date?, dueOn?, notes?)`                |
| 6    | Re-send or mark as sent                              | `manage-invoice(action:send, invoiceId)` or `manage-invoice(action:mark-as-sent, invoiceId)` |

**Cannot revert**: Paid invoices and void invoices cannot be reverted to draft. For paid invoices, use `issue-credit` instead. For void invoices, create a new invoice.
