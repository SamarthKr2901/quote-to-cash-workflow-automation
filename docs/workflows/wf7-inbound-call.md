# WF7 — Inbound Call Lead Capture

[← back to README](../../README.md) · [← workflow index](README.md)

**File:** [`/workflows/wf7-inbound-call.json`](../../workflows/wf7-inbound-call.json) · **Nodes:** 8 · **Status:** Active

The voice equivalent of [Form 1](../frontends.md#1-lead-intake--form-1).
When a customer dials the company's Twilio number instead of submitting
the web form, an ElevenLabs agent ("Alex") answers, collects the same
intake fields, and posts them here. WF7 creates the lead, emails the
Form 2 link, and returns `lead_id` so the agent can chain
`BookAppointment` mid-call.

The smallest workflow in the system but a meaningful one — it makes the
phone a first-class entry point alongside the web form.

## Trigger

| Webhook | Method | Source |
|---|---|---|
| `/inbound-lead` | POST | ElevenLabs Alex agent's `create_lead` tool |

## Flow

```
Webhook /inbound-lead
  └─► Validate &amp; normalize (trim, lowercase email, E.164 phone, 1000-char issue cap)
        └─► IF valid?
              ├─► Supabase: INSERT leads (pipeline_stage='Inbound Call - Form 2 Pending')
              │     └─► Code: Extract lead_id from Supabase response
              │           └─► Send Form 2 link email (customer)
              │                 └─► Respond 200 with { success, lead_id, email, customer_name }
              └─► Respond 400 with { success: false, errors: [...] }
```

## Expected payload from the agent

```json
{
  "customer_name": "John Smith",
  "phone_number": "+12145551234",
  "email": "john@example.com",
  "customer_address": "123 Main St, Dallas, TX",
  "issue_description": "Foggy windows in living room",
  "window_count": 3
}
```

`phone_number` comes from the ElevenLabs `system__caller_id` dynamic
variable — pulled from Twilio's caller-ID and never asked of the caller.
The agent collects everything else by voice.

## Response back to the agent

```json
{
  "success": true,
  "lead_id": "uuid-string",
  "email": "john@example.com",
  "customer_name": "John Smith",
  "message": "Lead created. Form 2 link sent to email."
}
```

The `lead_id` is the critical return value. The agent captures it via
ElevenLabs' Dynamic Variable Assignment (JSON Path: `lead_id`) and
passes it as a dynamic variable to subsequent tool calls in the same
conversation:

| Tool | Webhook (lives in [WF3 Post-Call](wf3-post-call.md)) |
|---|---|
| `book_appointment` | `/BookAppointment` — uses captured `lead_id` |
| `update_form_status` | `/update_form_status` — uses captured `lead_id` |
| `transfer_to_number` | system tool — Conference type to SM |
| `transfer_unavailable` | shared webhook |

## Pipeline-stage convention

WF7 inserts with `pipeline_stage = 'Inbound Call - Form 2 Pending'` —
distinct from the web entry's `'New Lead'`. This makes inbound vs. web
provenance trivially queryable without an extra column. Once the
customer fills out Form 2, the standard
[WF1](wf1-lead-capture.md#flow-b-window-details--ai-estimate-apex-window-details-v2)
window-details branch transitions the stage to
`'Window Details Received'`, and from that point on inbound and web
leads are indistinguishable.

## Why this is a separate workflow rather than a path inside WF1

Three reasons it lives on its own:

1. **Different validator.** The voice agent's structured output is
   already cleaned by the LLM (`+12145551234` not `(214) 555-1234`),
   so the validation is a thin re-check, not the full Form 1
   sanitization that includes ground-truth string parsing of free-form
   address fields.
2. **Different response contract.** WF7 returns a JSON object the agent
   programmatically reads (`lead_id` capture); WF1's lead webhook
   returns 200/400 only.
3. **Different failure semantics.** A failed WF7 call needs to be
   recoverable mid-conversation — the agent can apologize and ask the
   customer to wait, or transfer to a human. A failed Form 1 just
   shows a red error in the browser.

## What WF7 deliberately doesn't do

- Doesn't insert a `quote_approval_pending` escalation row. That gets
  created by WF1 when Form 2 is submitted, which keeps the escalation
  scheduler aligned with the actual estimate-ready moment.
- Doesn't trigger the outbound voice agent. The customer is already on
  the phone — calling them again is the wrong gesture.
- Doesn't write `num_windows` or `lead_source`. Both are filled in
  later: `num_windows` from Form 2, and any source tracking is
  inferred from `pipeline_stage`'s "Inbound Call - …" prefix.

## Where the data lands

| Table | Operation |
|---|---|
| `leads` | INSERT name, email, phone, address, issue_description, pipeline_stage='Inbound Call - Form 2 Pending' |

That's it. Single-table insert by design.

## External calls

| Service | Why |
|---|---|
| Supabase | INSERT lead row |
| Gmail SMTP | send Form 2 link email to the caller |

---

[← back to README](../../README.md) · [← workflow index](README.md)
