# Decision Trees

Conditional logic for pricing negotiations, scope management, collections, and business analysis. Each tree defines the trigger, the evaluation path, and the recommended action at each branch.

---

## 1. Client Says "Can You Lower the Price?"

**Trigger**: Client asks for a discount, lower rate, or cheaper option during estimate or project negotiation.

```
Client requests lower price
|
+-- Step 1: Load rate card
|  \-- manage-memory(action:read, namespace:rates)
|
+-- Step 2: Check rate card floor
|  +-- Requested rate >= floor rate
|  |  \-- Continue to Step 3
|  \-- Requested rate < floor rate
|     \-- HARD STOP: "This is below your minimum of AED X/hr.
|        I cannot approve this rate."
|        -> Offer alternatives: reduce scope, phase delivery, or decline
|
+-- Step 3: Check client history
|  \-- client-360(client)
|     +-- Existing client with good payment history (paymentReliability: "good")
|     |  \-- Options:
|     |     - Loyalty discount: 5-10% off standard rate
|     |     - Volume commitment: lower rate for guaranteed hours
|     |     - Scope trade: maintain rate, remove deliverables
|     +-- Existing client with poor payment history (paymentReliability: "poor")
|     |  \-- "Given outstanding balance / late payment history,
|     |     I recommend standard rates with upfront payment terms."
|     \-- New client (no history, paymentReliability: "unknown")
|        \-- "Standard rate applies for first engagement.
|           We can revisit after the first project."
|
\-- Step 4: Log outcome (rate-card singleton refresh)
   \-- manage-memory(action:upsert-entry, namespace:rates,
      entry:{defaultHourlyRate:[agreedRate], currency:"AED",
      perClientOverrides:{"[clientId]":[agreedRate]},
      lastReviewedAt:"[ISO]"})
      // Companion comms entry to preserve context:
      manage-memory(action:upsert-entry, namespace:comms,
      entry:{ts:"[ISO]", channel:"other", direction:"out",
      summary:"Rate negotiation with [client]. Requested: AED X/hr. Agreed: AED X/hr. Context: [reason]."})
```

**Never**: Accept a rate below the floor. If the user insists, require explicit override and log the exception.

---

## 2. Client Says "Add This Feature"

**Trigger**: Client requests additional work, a new deliverable, or expanded scope mid-project.

```
Client requests additional work
|
+-- Step 1: Load original scope
|  \-- manage-memory(action:read, namespace:rates)
|     -> Find the project entry with original scope, value, and milestones
|
+-- Step 2: Assess scope deviation
|  +-- Additional work <= 10% of original scope
|  |  \-- ABSORB: "This fits within the current project scope.
|  |     No adjustment needed."
|  |     -> Log to rates memory anyway as a scope note
|  |
|  \-- Additional work > 10% of original scope
|     \-- FLAG: "Scope has expanded by approximately X%.
|        Original: [description]. Addition: [new work]."
|        |
|        +-- Option A: Change order
|        |  \-- Create new estimate for the additional work
|        |     -> manage-estimate(action:create) for the delta
|        |     -> New milestone added to existing project
|        |
|        +-- Option B: Revised estimate
|        |  \-- Revise total project value to include new scope
|        |     -> Follow Estimate Revision workflow (Workflow 7)
|        |
|        \-- Option C: Separate project
|           \-- If the addition is distinct enough
|              -> Follow New Project Setup workflow (Workflow 2)
|
\-- Step 3: Log scope change (always, regardless of size)
   \-- manage-memory(action:upsert-entry, namespace:projects,
      entry:{projectName:"[project]", contactId:"[clientId]", status:"active",
      billingMode:"fixed", fixedAmount:[updatedValue],
      scopeNotes:"Scope change: [what was added]. Deviation: X%. Resolution: [absorbed/change order/revised].",
      startedAt:"[ISO]"})
      // Companion comms entry for the audit trail:
      manage-memory(action:upsert-entry, namespace:comms,
      entry:{ts:"[ISO]", channel:"other", direction:"in",
      summary:"Scope change for [project] ([client]). Original: [summary]. Change: [what was added]. Deviation: X%. Resolution: [absorbed/change order/revised]."})
```

**Key rule**: Always log scope changes to rates memory, even if absorbed. This builds a record for future pricing accuracy and is used by `rate-review` to calculate `scopeCreepHours` and `scopeCreepCost`.

---

## 3. Invoice 30+ Days Overdue

**Trigger**: Invoice detected as overdue during standup, dashboard-snapshot, or explicit check.

```
Overdue invoice detected
|
+-- Step 1: Check reminder history
|  \-- manage-memory(action:read, namespace:comms)
|     -> Search for previous reminders sent for this invoice
|
+-- Step 2: Check client payment profile
|  \-- client-360(client)
|     -> Payment history, avgPaymentDays, healthScore
|
+-- Step 3: Determine escalation tier by days overdue
|  |
|  +-- 0 days (due today)
|  |  \-- EXCLUDED: Invoice is due today -- not yet overdue.
|  |     Must NOT be included in reminder processing.
|  |     (Applies to all statuses including partially_paid)
|  |
|  +-- 1-7 days overdue
|  |  \-- Tier: FRIENDLY
|  |     Action: send-reminder(client, dryRun:true)
|  |     "Gentle nudge. First offense is usually an oversight."
|  |
|  +-- 8-21 days overdue
|  |  \-- Tier: FIRM
|  |     Action: send-reminder(client, dryRun:true)
|  |     "Direct request. Mention specific invoice and amount."
|  |
|  +-- 22-45 days overdue
|  |  \-- Tier: ESCALATION
|  |     Action: send-reminder(client, dryRun:true)
|  |     "Formal demand. Reference payment terms from contract."
|  |     Present 3 paths: direct call, write-off, external collections.
|  |
|  +-- 45-60 days overdue
|  |  \-- Tier: ESCALATION (same level -- 22+ days)
|  |     "Written notice with deadline. Pause new work for this client."
|  |
|  +-- 60-90 days overdue
|  |  \-- Tier: ESCALATION (same level -- 22+ days)
|  |     "Final warning. Recommend direct phone/video call."
|  |     Suggest: pause all active work for this client
|  |
|  \-- 90+ days overdue
|     \-- Tier: ESCALATION (same level -- 22+ days)
|        Recommend write-off:
|        -> manage-invoice(action:write-off, invoiceId)
|        -> Log as bad debt in comms memory
|        "This invoice is unlikely to be collected.
|        Recommend write-off and lessons learned."
|
+-- Step 4: Cross-reference with client behavior
|  +-- First-time offender + < 21 days
|  |  \-- Stay at current tier (benefit of doubt)
|  +-- Repeat offender (2+ late payments)
|  |  \-- Escalate one tier above calendar-based tier
|  \-- Client has 2+ overdue invoices simultaneously
|     \-- Surface healthScore, recommend freezing new work
|
\-- Step 5: Log all actions
   \-- manage-memory(action:upsert-entry, namespace:comms,
      entry:{ts:"[ISO]", channel:"other", direction:"out",
      summary:"Overdue assessment for [client], invoice #[number]. Days overdue: [N]. Tier: [tier]. Action: [sent/recommended/write-off]. Client history: [first offense / repeat / multiple overdue]."})
```

**Note on escalation levels**: The spec defines 3 escalation levels (friendly, firm, escalation). The business judgment tiers above (45-60, 60-90, 90+) are presentation guidance for the AI -- all map to the "escalation" level in `send-reminder`.

**Note on expense matching**: When matching expenses to projects, use case-insensitive exact match (`.toLowerCase()` + `===`), never substring (`.includes()`).

---

## 4. User Asks "How Am I Doing?"

**Trigger**: User asks for a general status overview, business health check, or performance question.

```
User asks for status
|
+-- Step 1: Determine scope from context
|  +-- General / no qualifier
|  |  \-- dashboard-snapshot
|  |     -> Overdue invoices, due this week, active projects, cash position, timer, pipeline
|  |
|  +-- "This month" / "monthly"
|  |  \-- monthly-summary
|  |     -> Invoiced vs collected, collectionRate, forecast, expenses, profit margin
|  |
|  +-- "Pricing" / "rates" / "am I charging enough?"
|  |  \-- rate-review
|  |     -> Effective rates by project, comparison to rate card, scope creep
|  |
|  +-- "What should I do?" / "action items" / "priorities"
|  |  \-- get-action-items
|  |     -> Prioritized list: overdue follow-ups, invoices to send, milestones due
|  |
|  +-- "Standup" / "catch me up" / "what happened?"
|  |  \-- standup
|  |     -> Changes since last session, what's next, blockers
|  |
|  \-- Specific client name mentioned
|     \-- client-360(client)
|        -> Full client profile: projects, invoices, payments, healthScore
|
+-- Step 2: Identify the most important number
|  +-- Outstanding overdue > 0
|  |  \-- Lead with: "AED X,XXX.XX overdue across N invoices"
|  +-- Cash flow gap within 30 days
|  |  \-- Lead with: "AED X,XXX.XX gap projected in [month]"
|  +-- collectionRate < 0.7 (70%)
|  |  \-- Lead with: "Collection rate at XX.X% -- below 70% target"
|  +-- profitMargin < 0 (loss-making period)
|  |  \-- Lead with: "Loss-making period: margin is -XX.X%"
|  \-- All healthy
|     \-- Lead with: "AED X,XXX.XX collected this month, XX.X% collection rate"
|
+-- Step 3: Generate action items
|  +-- Overdue invoices -> recommend reminders by tier
|  +-- Milestones due within 7 days -> surface completion status
|  +-- Effective rate below quoted rate -> flag projects
|  +-- Clients with declining healthScore -> recommend review
|  \-- Draft invoices blocking project closure -> surface them
|
\-- Step 4: Close with one prioritized action
   \-- Pick the single highest-impact item:
      Priority order:
      1. Overdue invoice > AED 1,000 (revenue at risk)
      2. Milestone due in < 3 days (delivery risk)
      3. Cash flow gap (liquidity risk)
      4. Rate below floor on active project (margin risk)
      5. Client healthScore declining (relationship risk)
      6. Draft invoices unsent (uncommitted billing)
```

**collectionRate display**: Always a ratio 0.0-1.0 in data. Display as percentage: `(collectionRate * 100).toFixed(1) + "%"`. Use the pre-computed `collectionRatePercent` field when available.

---

## 5. Volume Discount Decision

**Trigger**: Client proposes multiple projects, a retainer, or asks for a package deal.

```
Client proposes volume work
|
+-- Step 1: Assess volume
|  \-- manage-memory(action:read, namespace:rates)
|     -> Check client history + current engagement
|  |
|  +-- 3+ projects (completed or proposed)
|  |  \-- Qualifies for volume discount
|  +-- Retainer proposal (ongoing monthly work)
|  |  \-- Qualifies for retainer rate
|  \-- < 3 projects, no retainer
|     \-- "Standard rates apply. Volume pricing kicks in at 3+ projects."
|
+-- Step 2: Calculate discount
|  +-- Volume discount (3+ projects)
|  |  +-- 3-5 projects -> 10% off standard rate
|  |  \-- 6+ projects -> 15% off standard rate
|  |
|  \-- Retainer rate
|     +-- Check guaranteed monthly hours
|     +-- 20+ hrs/month -> 10% off standard rate
|     \-- 40+ hrs/month -> 15% off standard rate
|
+-- Step 3: Validate against floor
|  +-- Discounted rate >= floor rate
|  |  \-- Proceed with discount
|  \-- Discounted rate < floor rate
|     \-- HARD STOP: "Even with the volume discount,
|        the rate cannot go below $X/hr."
|        -> Offer smaller discount that stays above floor
|
\-- Step 4: Log agreement (rate-card singleton + comms audit)
   \-- manage-memory(action:upsert-entry, namespace:rates,
      entry:{defaultHourlyRate:[standardRate], currency:"AED",
      perClientOverrides:{"[clientId]":[discountedRate]},
      lastReviewedAt:"[ISO]"})
      // Companion comms entry for the audit trail:
      manage-memory(action:upsert-entry, namespace:comms,
      entry:{ts:"[ISO]", channel:"other", direction:"out",
      summary:"Volume agreement with [client]. Standard: AED X/hr. Discounted: AED X/hr (X% off). Basis: [N projects / retainer at N hrs/month]. Floor: AED X/hr — margin preserved."})
```

---

## 6. Rush Work Pricing

**Trigger**: Client needs work done urgently, deadline is less than 1 week from request date.

```
Client requests rush delivery
|
+-- Step 1: Assess timeline
|  +-- Deadline < 48 hours
|  |  \-- CRITICAL RUSH: 2.0x multiplier
|  |     "48-hour turnaround requires double the standard rate."
|  +-- Deadline < 1 week
|  |  \-- RUSH: 1.5x multiplier
|  |     "Less than one week requires a 50% rush premium."
|  \-- Deadline >= 1 week
|     \-- STANDARD: No multiplier
|        "Timeline allows for standard pricing."
|
+-- Step 2: Check current workload
|  \-- dashboard-snapshot
|     +-- Active milestones due this week > 2
|     |  \-- Add capacity surcharge consideration
|     |     "Current workload is heavy. Rush work may require
|     |     rescheduling other deliverables."
|     \-- Capacity available
|        \-- Standard rush multiplier applies
|
+-- Step 3: Present rush quote
|  \-- Standard rate x rush multiplier x estimated hours
|     "Rush rate: $X/hr x X.Xx = $X/hr. Estimated hours: N.
|     Total: AED X,XXX.XX"
|
+-- Step 4: Validate against client expectations
|  \-- manage-memory(action:read, namespace:rates)
|     -> Check if client has accepted rush rates before
|
\-- Step 5: Log rush agreement (comms entry; rates singleton stays untouched)
   \-- manage-memory(action:upsert-entry, namespace:comms,
      entry:{ts:"[ISO]", channel:"other", direction:"out",
      summary:"Rush work for [client]. Standard: $X/hr. Rush: $X/hr (X.Xx multiplier). Deadline: [date]. Reason: [client urgency details]."})
```

---

## 7. Retainer vs. Project

**Trigger**: User or client discusses ongoing work that does not fit a fixed-scope project model.

```
Ongoing work discussion
|
+-- Step 1: Assess work pattern
|  \-- manage-memory(action:read, namespace:rates)
|     -> Review client history for recurring work patterns
|  |
|  +-- Client has had 3+ projects in 6 months
|  |  \-- Strong retainer candidate
|  +-- Work is continuous (support, maintenance, ongoing development)
|  |  \-- Retainer recommended
|  +-- Work is periodic but predictable (monthly reports, quarterly updates)
|  |  \-- Light retainer or recurring invoice
|  \-- One-off or unpredictable
|     \-- Standard per-project billing
|        "This looks like a defined project. Let's scope it with milestones."
|
+-- Step 2: Propose retainer structure
|  +-- Hours-based retainer
|  |  \-- "N guaranteed hours per month at $X/hr.
|  |     Unused hours: [roll over 1 month / expire].
|  |     Overage: standard rate applies."
|  |
|  \-- Value-based retainer
|     \-- "Fixed monthly fee of AED X,XXX.XX covering [scope].
|        Out-of-scope work billed separately."
|
+-- Step 3: Calculate effective value
|  \-- Compare retainer rate vs. project rate
|     +-- Retainer provides >= 10% discount to client
|     |  \-- AND retainer guarantees minimum monthly revenue
|     |     -> "Win-win: client saves X%, you get predictable income."
|     \-- Retainer does not benefit both parties
|        \-- "Per-project billing is better for this arrangement."
|
+-- Step 4: Set up if agreed
|  +-- Create recurring invoice -> follow Recurring Invoicing workflow (Workflow 11)
|  \-- Log retainer as a project entry (with companion comms note)
|     \-- manage-memory(action:upsert-entry, namespace:projects,
|        entry:{projectName:"[client]-retainer", contactId:"[clientId]",
|        status:"active", billingMode:"retainer", hourlyRate:[X],
|        scopeNotes:"Type: [hours/value]. Rate: $X/hr or AED X,XXX.XX/month. Hours: N/month. Overage: $X/hr. Term: [duration].",
|        startedAt:"[ISO]"})
|        manage-memory(action:upsert-entry, namespace:comms,
|        entry:{ts:"[ISO]", channel:"other", direction:"out",
|        summary:"Retainer agreement with [client]. Type: [hours/value]. Rate: $X/hr or AED X,XXX.XX/month. Hours: N/month. Overage: $X/hr. Term: [duration]."})
|
\-- Step 5: Ongoing monitoring
   \-- Monthly: compare actual hours vs. retainer hours
      (use monthly-summary recurring revenue status)
      +-- Utilization < 60% for 2+ months
      |  \-- Flag: "Client is under-utilizing retainer.
      |     Consider reducing hours or adding scope."
      \-- Utilization > 120% for 2+ months
         \-- Flag: "Consistently exceeding retainer.
            Consider increasing retainer hours or raising rate."
```
