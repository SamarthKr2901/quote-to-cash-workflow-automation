# WF3 — Post-Call Processes

[← back to README](../../README.md) · [← workflow index](README.md)

**File:** [`/workflows/wf3-post-call.json`](../../workflows/wf3-post-call.json) · **Nodes:** 86 · **Status:** Active

The voice agents' shared back-end. Four webhooks called *during* and
*after* a conversation, plus three schedule triggers for appointment
reminders. Both the inbound ([WF7](wf7-inbound-call.md)) and outbound
([WF3 Outbound](wf3-outbound-call.md)) agents call into this workflow —
which is why the booking / availability endpoints don't live in either
of those.

## Triggers

| Webhook | Method | Called by |
|---|---|---|
| `/post_call_webhook` | POST | ElevenLabs webhook fired after a conversation ends |
| `/update_form_status` | POST | voice agent tool — runs as the call ends to write a status into `leads.form_status` |
| `/BookAppointment` | POST | voice agent tool — book an appointment mid-call |
| `/GetAvailability` | POST | voice agent tool — check open slots mid-call |

| Schedule | What it does |
|---|---|
| Daily 8 AM CT | 24-hour appointment reminder SMS |
| Every 30 min | 2-hour appointment reminder SMS |
| Schedule Trigger (legacy) | unused / placeholder |

## Path 1 — post-call analysis (`/post_call_webhook`)

ElevenLabs hits this with the full conversation transcript when the call
hangs up.

```
Webhook
  └─► IF call connected vs missed?
  │     ├─► CONNECTED:
  │     │     └─► Format transcript
  │     │           └─► AI Agent (GPT-4 LangChain agent — extracts intent, interest level, requested appointment)
  │     │                 └─► Parse output JSON
  │     │                       └─► Fetch lead
  │     │                             └─► UPDATE leads.lead_temp / qualification_stage / customer_intent / recommended_action
  │     │                             └─► Append row to Google Sheet (call outcome log)
  │     │                                   └─► IF appointment requested?
  │     │                                         └─► Send SM email with details + Twilio "thanks" SMS to customer
  │     └─► MISSED:
  │           └─► Twilio missed-call SMS to customer with Form 2 link
  │                 └─► UPDATE leads.call_attempts
```

The post-call AI is structured as a tool-calling agent rather than a
single prompt because the output schema is rich — qualification_stage,
lead temperature, customer intent, recommended next action, and
optionally an appointment_request — and a Code node parses the JSON the
agent returns into typed columns.

## Path 2 — `/update_form_status`

A small webhook the voice agent calls before hanging up to write a
status string like `Contacted - Inbound Call - Appointment Booked` or
`Contacted - Transfer Failed - Callback Pending` into `leads.form_status`.
Used by the SM-facing CRM to filter active conversations.

## Path 3 — `/BookAppointment`

Runs while the customer is still on the call. The agent has just told
the customer "OK, let me book that for you" — this webhook does it.

```
Webhook
  └─► Parse &amp; validate booking (date / time / service_type)
        └─► IF valid?
              └─► Build event data
                    └─► Google Calendar — create event
                          └─► Merge booking data with calendar response
                                └─► Fetch lead from Supabase
                                      └─► INSERT appointments (lead_id, date, time, service_type)
                                            └─► Send confirmation email (customer)
                                                  └─► Respond 200 with booking confirmation
              └─► invalid → Respond 400
```

`service_type` mapping is handled in the validator: `"Quote Review
Call"` → `quote_review_call`, `"Installation"` → `installation`, anything
else → `estimating`. The CHECK constraint on `appointments.service_type`
enforces these three values.

## Path 4 — `/GetAvailability`

Runs while the customer is on the call. The agent asks "what works for
you, Tuesday or Wednesday?" and this webhook returns the open slots.

```
Webhook
  └─► Parse availability request (date)
        └─► IF valid?
              └─► Google Calendar — list events on that date
                    └─► Code: ensure calendar items (placeholder pass-through)
                          └─► Build available slots (from full-day grid minus busy events)
                                └─► Respond with slots
```

The "ensure calendar items" node is doing real work — see the
engineering note below.

## Appointment reminder crons

Two parallel reminder cadences:

| Cron | Window | Action |
|---|---|---|
| Daily 8 AM CT | appointments tomorrow | Send 24-hour reminder SMS, set `reminder_24hr_sent = true` |
| Every 30 min | appointments in next 2 hours, where 2hr-reminder hasn't fired | Send 2-hour reminder SMS, set `reminder_2hr_sent = true` |

Both reminders are gated by their respective boolean column to prevent
double-sends if a cron run overlaps the appointment window.

## Engineering note: the empty-calendar / past-date double bug

A real bug pair worth calling out — both surfaced from the same root
cause (the LLM's lack of "now"), and both required n8n-level fixes.

**Bug 1 — empty calendar skipped slot generation.** When a day had no
existing events, `Get Calendar Events` returned 0 items, which made n8n
skip every downstream node (the default behavior is "no input → don't
run"). The customer would hear "let me check… one moment…" and the agent
would receive a null response, then improvise.

The fix has two parts:
1. Set `alwaysOutputData: true` on `Get Calendar Events`, which makes
   it output an empty `{}` when no events exist.
2. Add a `Code: Ensure Calendar Items` node downstream that converts
   that empty `{}` into a single placeholder item, so the slot-builder
   has something to iterate.

**Bug 2 — LLM year confusion.** GPT-4o has no current-date context, so
when a customer said "next Tuesday" the agent confidently sent
`appointment_date: "2024-05-17"` to the BookAppointment webhook. n8n
rejected it: "Appointment date cannot be in the past."

The fix is in the validator code node — it auto-corrects past-year dates
to the current year, falling back to the next year only if same-month-
day in the current year would also be in the past:

```javascript
const today = new Date().toISOString().split('T')[0];
if (appointment_date && appointment_date < today) {
  const [_, mm, dd] = appointment_date.split('-');
  const yr = new Date().getFullYear();
  const sameYearDate = `${yr}-${mm}-${dd}`;
  appointment_date = sameYearDate >= today
    ? sameYearDate
    : `${yr + 1}-${mm}-${dd}`;
}
```

The same correction is applied in `/GetAvailability`'s parser.

The deeper fix (out of scope for this workflow) is to inject a current
date variable into the agent's prompt at conversation start via
ElevenLabs' `conversation_initiation_client_data_webhook`. That cuts
the bug at the source.

## Where the data lands

| Table | Operation |
|---|---|
| `leads` | UPDATE lead_temp, qualification_stage, customer_intent, recommended_action, form_status, call_attempts, conversation_id |
| `appointments` | INSERT (booking); UPDATE reminder_24hr_sent / reminder_2hr_sent |
| `windows` | SELECT (read for transcript context only) |

## External calls

| Service | Why |
|---|---|
| OpenAI GPT-4 | post-call transcript analysis (LangChain agent) + update_form_status parsing |
| Google Calendar | create / read events for booking and availability |
| Google Sheets | append call outcome row to the team's call log |
| Twilio SMS | missed-call SMS, confirmation thanks, 24-hour and 2-hour appointment reminders |
| Gmail SMTP | booking confirmation email to customer |

---

[← back to README](../../README.md) · [← workflow index](README.md)
