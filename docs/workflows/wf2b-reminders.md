# WF2B — Reminder System

[← back to README](../../README.md) · [← workflow index](README.md)

**File:** [`/workflows/wf2b-reminders.json`](../../workflows/wf2b-reminders.json) · **Nodes:** 20 · **Status:** Active

The hourly cron that nudges customers about quotes they haven't acted
on, and about invoices they haven't paid. Auto-declines a quote on day
3 of no response.

## Trigger

| Trigger | Schedule |
|---|---|
| `Schedule Trigger` | hourly |

## Flow

```text
Schedule Trigger (hourly)
  └─► Build date filter (current hour window)
        ├─► Fetch quote_reminder escalations (resolved=false, due_at < now)
        │     └─► Split escs (one item per row)
        │           └─► Fetch lead by lead_id
        │                 └─► Fetch quote by lead_id (for accept/decline URLs and amount)
        │                       └─► Merge data into email context
        │                             └─► Send quote-reminder email (customer)
        │                                   └─► UPDATE escalations.notification_count++,
        │                                       last_notified_at = now()
        │                                         └─► IF notification_count >= 3 (day 3):
        │                                               ├─► UPDATE quotes.status = 'declined'
        │                                               ├─► UPDATE leads.pipeline_stage = 'Closed Lost'
        │                                               └─► UPDATE escalations.resolved = true
        │
        └─► Fetch payment_reminder escalations
              └─► Split + fetch lead + fetch invoice
                    └─► Send payment-reminder email
                          └─► UPDATE escalations.notification_count++
```

## Why this is its own workflow

Two reasons it isn't merged into [WF3 Post-Payment](wf3-post-payment.md),
which also runs hourly:

1. **Different audience.** WF3 Post-Payment also handles SM-overdue
   alerts, customer no-response watchdogs, review requests, and the
   quarterly newsletter. Splitting the customer-facing nudges off keeps
   each workflow's responsibility readable in the n8n editor.
2. **Independent retry semantics.** If the email service flakes during a
   newsletter run, the SM-overdue alerts still need to fire — and vice
   versa. Separate workflows mean separate execution histories and
   separate retry policies.

## Auto-decline logic

The day-3 transition is the only place outside the customer-facing
accept/decline links where a quote can move to `declined`. The cooldown
filter (notification_count >= 3) is what makes this safe — it doesn't
depend on absolute time, so a delayed cron run on day 4 or day 5 still
declines correctly rather than skipping.

The unresolved corner case: the `payment_reminder` escalation is
inserted by [WF2 sub-flow D](wf2-quote-and-payment.md#sub-flow-d--job-completed-job-completed)
without a `due_at` value (a known bug, tracked in
[architecture.md §5](../architecture.md#5-known-security-debt)). The
filter query handles NULL gracefully — those rows just don't match
`due_at < now` — but they also never fire a payment reminder. Fix is
straightforward (set `due_at` at insert time) but is on the production
prep list, not in this workflow.

## Where the data lands

| Table | Operation |
|---|---|
| `escalations` | UPDATE notification_count, last_notified_at; UPDATE resolved=true on auto-decline |
| `quotes` | UPDATE status='declined' on day 3 |
| `leads` | UPDATE pipeline_stage='Closed Lost' on day 3 |

## External calls

Gmail SMTP for both reminder emails.

---

[← back to README](../../README.md) · [← workflow index](README.md)
