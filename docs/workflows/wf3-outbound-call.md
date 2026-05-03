# WF3 — Outbound Call

[← back to README](../../README.md) · [← workflow index](README.md)

**File:** [`/workflows/wf3-outbound-call.json`](../../workflows/wf3-outbound-call.json) · **Nodes:** 11 · **Status:** Active

Smallest workflow in the system. Dispatches an ElevenLabs voice agent
to call the customer minutes after they submit Form 1 — or sends an
SMS with the Form 2 link if it's outside calling hours. Also runs a
daily retry cron for missed calls.

## Triggers

| Trigger | Source |
|---|---|
| `/new-lead` POST | [WF1](wf1-lead-capture.md) immediately after lead creation |
| daily 9 AM CST cron | retry loop for missed calls |

## Flow

```
Webhook /new-lead
  └─► Validate &amp; normalize payload (E.164 phone)
        └─► IF valid?
              └─► Time check (current hour in America/Chicago)
                    ├─► IF 8 AM – 8 PM CST:
                    │     └─► HTTP POST → ElevenLabs API (start outbound conversation)
                    │           └─► Respond 200
                    │           └─► UPDATE leads.conversation_id (when call ends)
                    └─► ELSE (out of hours):
                          └─► Twilio SMS with Form 2 link
                                └─► Respond 200
```

```
Daily cron 9 AM CST
  └─► Fetch leads with call_attempts < 7 AND no conversation outcome
        └─► UPDATE leads.call_attempts++
              └─► IF attempts < 7:  retry SMS
              └─► ELSE:  switch to weekly re-engagement SMS
```

## ElevenLabs call setup

When the call fires, the workflow passes a structured set of dynamic
variables into the ElevenLabs agent's system prompt:

```json
{
  "customer_first_name": "...",
  "customer_address": "...",
  "customer_issue_description": "...",
  "num_windows": 0,
  "second_form_completed": false
}
```

The agent (named "Eric" in the system prompt) is configured with shared
tool webhooks pointing at [WF3 Post-Call](wf3-post-call.md):

| Tool | Webhook |
|---|---|
| `BookAppointment` | `/BookAppointment` (WF3 Post-Call) |
| `GetAvailability` | `/GetAvailability` (WF3 Post-Call) |
| `update_form_status` | `/update_form_status` (WF3 Post-Call) |
| `transfer_to_number` | system tool — Conference type, target = SM phone |
| `transfer_unavailable_handler` | shared webhook |

Same toolset is reused by the inbound agent in
[WF7](wf7-inbound-call.md), which is why the booking / availability
endpoints aren't owned by this workflow — they live in WF3 Post-Call as
shared infrastructure.

## Time-of-day gating

The hour check uses `America/Chicago` because the customer base is in
DFW, not because the n8n instance is. n8n's default timezone is the
server's, so hour math is done explicitly:

```javascript
const cstHour = new Date().toLocaleString('en-US', {
  timeZone: 'America/Chicago',
  hour: 'numeric',
  hour12: false
});
```

If the lead arrives at 11 PM CST, the SMS path fires instead and the
customer gets the Form 2 link by text. The voice agent will pick it
up the next morning via the retry cron — though if the customer has
already submitted Form 2 in the meantime, the retry skips them.

## Retry semantics

The daily 9 AM cron is the watchdog for "we tried, they didn't answer,
try again tomorrow." Retries cap at 7 attempts (~one calendar week of
business days), after which the cadence drops to weekly until the lead
is otherwise resolved (Form 2 submitted, lead closed, or manually
removed).

## Where the data lands

| Table | Operation |
|---|---|
| `leads` | UPDATE conversation_id, call_attempts |

## External calls

| Service | Why |
|---|---|
| ElevenLabs | start an outbound conversational AI call (agent dispatch) |
| Twilio SMS | OOH fallback + retry loop nudges |

---

[← back to README](../../README.md) · [← workflow index](README.md)
