# WF3 — Post-Payment Followups

[← back to README](../../README.md) · [← workflow index](README.md)

**File:** [`/workflows/wf3-post-payment.json`](../../workflows/wf3-post-payment.json) · **Nodes:** 82 · **Status:** Active

The post-sale workflow and the system's escalation engine. Sends a
review request 30 minutes after a customer pays, runs a quarterly
newsletter, and acts as the watchdog for any escalation row that
the rest of the system created — overdue SM approvals, customer
no-response after a quote, missed review requests.

## Triggers

| Trigger | Source |
|---|---|
| `/closed-won` POST | [WF2 sub-flow E](wf2-quote-and-payment.md#sub-flow-e--payment-received-stripe--manual) immediately after a Stripe payment |
| Schedule (hourly, 6 AM – 10 PM CST) | escalation watchdog + newsletter sweep |
| Error trigger | catches scheduler failures and sends an alert |

## Path 1 — closed-won webhook

Fires once per closed-won transition. The deliberate 30-minute wait
gives the SM a buffer to step in if the payment turns out to need a
refund.

```
Webhook /closed-won
  └─► Respond 200 immediately   ← critical: WF2 times out if this is delayed
        └─► Wait 30 minutes
              └─► Fetch lead by email
                    └─► Check post_job_comms for prior review_request
                          └─► IF not yet sent:
                                └─► Send Google review request email
                                      └─► INSERT post_job_comms (comm_type='review_request')
                                            └─► UPDATE leads.google_review_requested_at
                                                  └─► UPDATE leads.last_newsletter_sent_at
                                                        └─► INSERT activity_log
                                                              └─► Email SM (post-sale sequence complete)
```

The duplicate guard via `post_job_comms` is what makes it safe for both
this webhook *and* the hourly cron's review-request fallback (Path 2c
below) to fire for the same lead — see
[architecture.md §4.6](../architecture.md#46-idempotent-post-sale-comms).

## Path 2 — hourly escalation watchdog

A four-branch fan-out off the operating-hours gate. Each branch handles
a different escalation type with its own cooldown.

```
Schedule Trigger (hourly)
  └─► Operating hours check (6 AM – 10 PM CST)
        ├─► Branch 1: SM overdue escalations
        ├─► Branch 2: customer no-response escalations
        ├─► Branch 3: review request fallback
        └─► Branch 4: quarterly newsletter
```

### Branch 1 — SM overdue (`type='quote_approval_pending'`)

```
Fetch escalations where due_at < now AND resolved=false
  └─► (cooldown: skip if last_notified_at < 12 hours ago)
        └─► Loop:
              └─► Calculate hours overdue
                    └─► Fetch lead
                          └─► Send SM + supervisor "OVERDUE ALERT" email
                                └─► IF type=quote_approval_pending (not other types):
                                      └─► Generate AI delay message (GPT-4) for customer
                                            └─► Email customer with delay apology
                                                  └─► Twilio SMS to SM
                                                        └─► UPDATE escalations.last_notified_at + notification_count++
```

### Branch 2 — customer no-response (`type='customer_not_responded'`)

The 72-hour escalation that WF2 sub-flow B inserts when a quote is sent.

```
Fetch escalations type='customer_not_responded', due_at < now, resolved=false
  └─► (cooldown: skip if last_notified_at < 24 hours ago)
        └─► Loop:
              └─► Fetch lead + acceptance link
                    └─► Email customer follow-up
                          └─► Email SM alert
                                └─► IF notification_count >= 3:
                                      └─► Email "Closed Lost candidate" alert
                                            └─► Twilio SMS to SM
                                └─► UPDATE escalations
```

### Branch 3 — review request fallback (`type='review_request'`)

Backup path for when the `/closed-won` webhook didn't fire (rare, but
possible if Stripe was offline).

```
Fetch escalations type='review_request', overdue
  └─► (no cooldown — only fires once)
        └─► Loop:
              └─► Check post_job_comms for prior review_request (dedup)
                    └─► Verify lead.pipeline_stage in ('Completed', 'Closed Won')
                          └─► IF safe to send:
                                └─► Send Google review email
                                      └─► INSERT post_job_comms
                                            └─► UPDATE escalations.resolved=true
```

### Branch 4 — quarterly newsletter

Doesn't read from `escalations`; reads directly from `leads`.

```
SELECT leads
  WHERE pipeline_stage IN ('Completed', 'Payment Pending', 'Closed Won')
    AND last_newsletter_sent_at < now - 90 days
  LIMIT 50
  └─► 2-second wait between sends (rate limiting)
        └─► Send newsletter email
              └─► UPDATE leads.last_newsletter_sent_at = now()
                    └─► INSERT activity_log
```

The 50-per-run cap + hourly cadence gives a soft 1,200/day ceiling on
newsletter sends, which is well below Gmail SMTP's daily limit but well
above the actual customer base size — the cap is there for safety, not
throughput.

## Path 3 — error trigger

n8n's `errorTrigger` node is wired to catch any unhandled failure
*inside this workflow*. When it fires:

```
Error Trigger
  └─► Email alert to two team members
        └─► Twilio SMS to two team members
```

This is a deliberate "alert the humans, don't try to recover" pattern.
The escalation engine touches customer-facing comms (review emails,
delay apologies, closed-lost alerts) where a silent failure could erode
trust, so the trade-off is to be loud about failure.

## The escalation table as a unified scheduler

Worth re-stating because three of the four branches above hang off the
same table. See
[architecture.md §4.3](../architecture.md#43-escalation-table-as-a-unified-scheduler)
for the design rationale.

## Where the data lands

| Table | Operation |
|---|---|
| `leads` | UPDATE google_review_requested_at, last_newsletter_sent_at |
| `escalations` | UPDATE notification_count, last_notified_at, resolved |
| `post_job_comms` | INSERT for review_request (dedup target) and newsletter |
| `activity_log` | INSERT on every send |

## External calls

| Service | Why |
|---|---|
| Gmail SMTP | review email, newsletter, delay apologies, SM alerts, error alerts |
| OpenAI GPT-4 | generate per-customer delay-apology message text |
| Twilio SMS | SM nudges, closed-lost alerts, error alerts |

---

[← back to README](../../README.md) · [← workflow index](README.md)
