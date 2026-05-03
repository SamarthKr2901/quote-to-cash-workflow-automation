# WF2 — Quote Approval &amp; Payment

[← back to README](../../README.md) · [← workflow index](README.md)

**File:** [`/workflows/wf2-quote-and-payment.json`](../../workflows/wf2-quote-and-payment.json) · **Nodes:** 85 · **Status:** Active

The most complex piece of the system. Nine webhook entry points,
five conceptual sub-flows (A–E) plus a customer-facing accept/decline
landing pair, all sharing the same set of helper sub-paths for fetching
leads / quotes / estimates. Together they cover the full quote
lifecycle: from "estimate is ready" through "Stripe payment confirmed".

## Triggers — nine webhooks under one workflow

| # | Webhook | Method | Source | Sub-flow |
|---|---|---|---|---|
| 1 | `/quote-approval-ready` | POST | [WF1](wf1-lead-capture.md) after estimate generation | **A** — quote queued |
| 2 | `/manager-approved` | POST | CRM "Approve" button | **B** — quote sent to customer |
| 3 | `/manager-decline` | POST | CRM "Decline" button (rolling out — not yet wired in CRM HTML) | **B′** |
| 4 | `/quote-accept` | GET | customer email link | **C2** — landing page (also fires C internally) |
| 5 | `/quote-decline` | GET | customer email link | **C3** — landing page (direct PATCH, no internal fire) |
| 6 | `/customer-accepted` | POST | C2 internal trigger | **C** — server-side state transition |
| 7 | `/job-completed` | POST | CRM "Mark Complete" button | **D** — invoice + Stripe payment link |
| 8 | `/payment-received` | POST | manual entry (cash / off-Stripe) | **E** |
| 9 | `/stripe-payment` | POST | Stripe `checkout.session.completed` webhook | **E** |

Why these all live on one workflow: see
[architecture.md §4.2](../architecture.md#42-one-workflow-many-webhooks).

## Sub-flow A — quote queued (`/quote-approval-ready`)

WF1 calls this immediately after the estimate PDF is uploaded.

```
Webhook
  └─► Fetch Lead A (by id)
        └─► Fetch Estimate (by lead_id, status=pending_review)
              └─► Generate Token  (crypto.randomBytes(32).toString('hex') — 64 chars)
                    └─► INSERT quotes (status='pending_approval', acceptance_token, total_amount, pdf_url)
                          └─► Build approval URL (CRM link with quote_id)
                                └─► UPDATE leads.pipeline_stage = 'Quote Pending Approval'
                                      └─► INSERT activity_log (event_type='quote_queued')
```

Output: a `quotes` row in `pending_approval` state, ready for the SM to
review in the CRM. Note that the acceptance token is generated *here* —
before the SM has even seen the quote — because it'll be embedded in
the quote email URL later, and we want it stable across SM
adjustments.

## Sub-flow B — manager approved (`/manager-approved`)

The SM clicks Approve in the CRM. CRM POSTs `{quote_id, lead_id,
approved_by, adjusted_price?}`.

```
Webhook
  └─► Fetch Lead B
        └─► Validate Quote (SELECT quotes WHERE id=? AND status='pending_approval')
              └─► IF price was adjusted in CRM
              │     └─► UPDATE estimates.adjusted_total + manually_adjusted=true
              │     └─► Set "final total" = adjusted_total
              │     ELSE: final total = estimates.total
              └─► Build quote email HTML (with accept/decline URLs embedding the 64-char token)
                    └─► Email customer (subject: "Your Apex Windows quote is ready")
                          └─► UPDATE quotes.status = 'sent', sent_at = now()
                                └─► UPDATE leads.pipeline_stage = 'Quote Sent'
                                      └─► INSERT escalations (type='customer_not_responded', due_at=now+72h)
                                            └─► INSERT escalations (type='quote_reminder', due_at=now+24h)
                                                  └─► Email SM ("quote sent to customer" confirmation)
                                                        └─► INSERT activity_log
```

The two escalation rows here are the entry points for two different
reminder loops:
[WF2B](wf2b-reminders.md) handles `quote_reminder` (24h cadence,
auto-decline at day 3); [WF3 Post-Payment](wf3-post-payment.md) handles
`customer_not_responded` (72h alert).

### Magic-link acceptance token — the auth pattern in detail

The quote email contains two links the customer can click without
logging in:

```
{{COMPANY_DOMAIN}}/quote/accept?token=<64-char-hex>
{{COMPANY_DOMAIN}}/quote/decline?token=<64-char-hex>
```

**Token construction:** `crypto.randomBytes(32).toString('hex')` — 32
bytes = 256 bits of entropy = effectively unguessable. Stored on
`quotes.acceptance_token` at quote creation time (Sub-flow A). One
token per quote.

**Click handler (sub-flow C2):** the GET webhook validates with a single
SQL predicate:

```sql
SELECT * FROM quotes
WHERE acceptance_token = $1
  AND status = 'sent'
LIMIT 1;
```

The `status = 'sent'` clause is the revocation mechanism. As soon as
sub-flow C transitions the row to `accepted`, the same link becomes a
no-op — repeated clicks return "this link has expired" without any
explicit token revocation logic. Same trick on the decline link.

**Why this works:**
- The token *is* the customer's identity. We don't know who clicked,
  we only know what quote they intended to act on. That's enough.
- No session, no cookie, no email-link expiry job required — the row's
  `status` column carries the whole state machine.
- Stateless: the customer can forward the email to a partner, the
  partner can click, and either party closes the loop. Acceptable
  semantics for a B2C quote.

**Where it would not work:**
- High-value transactions (anything where the quote acceptance creates a
  binding contract requiring the customer's verified identity).
- Repeated actions (token is single-use by virtue of `status='sent'`;
  a "view" action would need different logic).

## Sub-flow C — customer accepted

The accept flow is split across three webhooks: **C2** is the GET link
(returns an HTML "thanks, we'll be in touch" page), **C** is the POST
that updates state (triggered internally by C2 so the page can return
fast), and **C3** is the decline GET (which shortcuts and updates state
inline because there's no async work to defer).

```
[customer clicks Accept link in email]
  │
  ├─► GET /quote-accept?token=…   (Webhook - Quote Accept)
  │     └─► HTTP POST internally → /customer-accepted (Trigger Flow C)
  │     └─► Respond - Accept Page (HTML "We've got it from here")
  │
  └─► (in parallel) POST /customer-accepted
        └─► Validate Token (SELECT quotes WHERE token=? AND status='sent')
              └─► IF invalid → Respond Invalid Token + stop
              └─► IF valid:
                    └─► Fetch Lead C
                          └─► UPDATE quotes.status = 'accepted', accepted_at = now()
                                └─► UPDATE leads.pipeline_stage = 'Quote Accepted'
                                      └─► Resolve customer_not_responded escalation
                                            └─► Email SM ("customer accepted quote")
                                                  └─► Email customer ("we'll book the install soon")
                                                        └─► INSERT activity_log
```

```
[customer clicks Decline link in email]
  │
  └─► GET /quote-decline?token=…   (Webhook - Quote Decline)
        └─► Respond - Decline Page (HTML "we're sorry to see you go")
        └─► (in parallel) Direct UPDATE quotes.status = 'declined'
              └─► Fetch Lead - Declined
                    └─► Email SM ("customer declined quote")
```

## Sub-flow D — job completed (`/job-completed`)

The SM marks the job complete in the CRM after the install. WF2 then
generates the invoice and the Stripe payment link.

```
Webhook
  └─► Check duplicate job (was this lead already marked complete?)
        └─► IF already completed → silent dead-end (known issue, see below)
        └─► UPDATE leads.pipeline_stage = 'Completed'
              └─► Build invoice data (invoice_number = APX-YYYY-XXXX, due_date = now+14d)
                    └─► Fetch accepted quote (for amount + lead_id)
                          └─► INSERT invoices (amount_due, invoice_number, due_date, status='unpaid')
                                └─► HTTP POST → Stripe Prices API (create one-off price)
                                      └─► HTTP POST → Stripe Payment Links API (create checkout link)
                                            └─► UPDATE invoices.stripe_payment_link
                                                  └─► UPDATE leads.pipeline_stage = 'Payment Pending'
                                                        └─► INSERT escalations (type='payment_reminder')
                                                              └─► Email customer (invoice + Stripe link)
```

The invoice number format is `APX-YYYY-XXXX` (4-digit zero-padded
sequence per year). Stripe Prices + Payment Links are used because
the order is one-off, not subscription — Prices is the right primitive
for a fixed amount, Payment Links wraps it in Stripe's hosted checkout.

## Sub-flow E — payment received (Stripe + manual)

Two webhooks feed into this. The Stripe one is the primary path; the
manual one exists for cash / off-platform payments the SM enters by hand.

```
[Stripe checkout.session.completed]
  └─► Stripe Webhook
        └─► Parse Stripe Event (filter to checkout.session.completed only)
              └─► Normalize Payment (extract amount_paid, transaction_id, payment_method)
                    └─► Validate Invoice (SELECT invoices WHERE id=?)
                          └─► IF amount_paid >= amount_due (full payment)
                          │     └─► UPDATE invoices.status = 'paid', paid_at = now()
                          │           └─► UPDATE leads.pipeline_stage = 'Closed Won', closed_won_at = now()
                          │                 └─► Resolve all open escalations for this lead
                          │                       └─► Fetch Lead E
                          │                             └─► Email customer (payment confirmed)
                          │                                   └─► Email SM (closed won)
                          │                                         └─► HTTP POST → /closed-won (triggers WF3 Post-Payment)
                          │                                               └─► INSERT activity_log
                          └─► ELSE (partial payment)
                                └─► UPDATE invoices.status = 'partial', amount_paid = $partial
                                      └─► Email SM (partial payment alert)
```

Closed-won fires a fire-and-forget POST to WF3 Post-Payment's
`/closed-won` webhook. WF3 responds 200 immediately and waits 30 minutes
before sending the Google review request — both giving the customer a
chance to receive the payment-confirmed email first, and giving the SM
time to step in if a refund is needed.

## Engineering challenge: the `$json.body.lead_id` vs `$json.lead_id` bug

A bug that took half a day to find and surfaces a real n8n quirk worth
documenting.

**Symptom:** Sub-flow B was working when triggered from the CRM in the
browser, but failing — silently, with `lead_id = undefined` propagated
into Supabase queries — when triggered by an automated test that
POST'd to `/manager-approved` directly.

**Root cause:** n8n's webhook node exposes the incoming payload at
`$json.body.<field>` when the request comes from outside the n8n
instance, and at `$json.<field>` when the request comes from another
n8n node's HTTP Request (because the internal call serializes the
payload as the top-level object, not as a nested `body`). Several
nodes downstream had been written reading `$json.lead_id` based on
behavior under one trigger source and never re-tested under the other.

**The fix:** standardize all reads at the head of every webhook flow on
`$json.body.lead_id || $json.lead_id` via a Set node, so every
downstream node reads from one normalized field. The patch touched
14 nodes across sub-flows B, B′, C, D, and E.

**Why MCP didn't catch it:** n8n's MCP tooling has an
`execute_workflow` operation that runs a workflow with a synthetic
payload, but for multi-webhook workflows it consistently routes to the
first webhook node — not the one identified by path. The end-to-end test
for sub-flow B therefore had to be a real HTTP POST to the live webhook
URL, which is how the bug eventually surfaced. Tests via MCP were a
false negative.

**Takeaway:** the implicit payload contract between webhook and node
isn't enforced by n8n. Centralizing the "read from request" step into
one Set node per flow is now the convention across this project — a
small amount of ceremony that prevents an entire class of bug.

## Sub-flow B′ — manager declined

A newer flow added after the others. The CRM's Decline button currently
PATCHes Supabase directly (a known security issue,
[architecture.md §5](../architecture.md#5-known-security-debt)); the
`/manager-decline` webhook exists to receive the proper server-side
flow (UPDATE quotes status=declined → UPDATE leads pipeline_stage =
'Closed Lost' → INSERT activity_log → email SM). The CRM has not been
switched over yet — that's a tracked open task.

## Where the data lands

| Table | Operations |
|---|---|
| `leads` | UPDATE pipeline_stage transitions: 'Quote Pending Approval' → 'Quote Sent' → 'Quote Accepted' / 'Closed Lost' → 'Completed' → 'Payment Pending' → 'Closed Won' |
| `estimates` | UPDATE adjusted_total, manually_adjusted (Sub-flow B price-adjust path) |
| `quotes` | INSERT (Sub-flow A); UPDATE status (B, C, B′) |
| `invoices` | INSERT (Sub-flow D); UPDATE status, paid_at (Sub-flow E) |
| `escalations` | INSERT customer_not_responded + quote_reminder + payment_reminder; UPDATE resolved=true on accept/payment |
| `activity_log` | INSERT on every state transition |

## External calls

| Service | Why | Where |
|---|---|---|
| Stripe | create Price + Payment Link; receive `checkout.session.completed` webhook | Sub-flow D, Sub-flow E |
| Gmail SMTP | every customer + SM email | every flow |
| Supabase REST API | direct PATCH calls on three nodes that need DB writes the n8n Supabase node didn't support at build time | C, E |

---

[← back to README](../../README.md) · [← workflow index](README.md)
