# WF1 — Lead Capture &amp; Validation

[← back to README](../../README.md) · [← workflow index](README.md)

**File:** [`/workflows/wf1-lead-capture.json`](../../workflows/wf1-lead-capture.json) · **Nodes:** 55 · **Status:** Active

The funnel's entry point. Two webhooks on one workflow because the lead
capture and the window-details upload are conceptually one customer
intake split across two emails — keeping them under one definition lets
both share the same lead-lookup helpers and activity logging.

## Triggers

| Webhook | Method | Source |
|---|---|---|
| `/apex-lead-v2` | POST (JSON) | [Form 1](../../frontends/lead-intake/index.html) — name, email, phone, address, num_windows, issue_description |
| `/apex-window-details-v2` | POST (multipart/form-data) | [Form 2](../../frontends/window-details/index.html) — per-window photos + notes |

## Flow A: lead intake (`/apex-lead-v2`)

```
Webhook
  └─► Map webhook to form fields
        └─► Dispatch to outbound voice agent (HTTP POST → /new-lead, fire-and-forget)
              └─► IF required fields valid?
                    ├─► OpenAI lead validation (spam/coherence check on issue_description)
                    │     └─► IF AI says valid?
                    │           └─► Supabase dedup check (existing email?)
                    │                 ├─► duplicate → log + stop
                    │                 └─► new lead → sanitize fields
                    │                       └─► INSERT leads (pipeline_stage='New Lead')
                    │                             └─► INSERT activity_log
                    │                                   └─► INSERT escalations
                    │                                       (type='quote_approval_pending', due_at=now+24h)
                    │                                         └─► Build Form 2 URL
                    │                                               └─► Email SM (new lead alert)
                    │                                                     └─► Email customer (Form 2 link)
                    │                                                           └─► Respond 200
                    │           └─► invalid → stop + log
                    └─► fail → stop + 4xx
```

Key points:

- **Spam guard before insert.** The OpenAI call on `issue_description`
  filters obvious junk submissions before they reach the database. It's a
  single-shot prompt, not an agent — fast, cheap, and the result is a
  binary keep/drop.
- **Field sanitization is centralized.** One Code node lowercases the
  email, normalizes the phone to E.164, and title-cases the name. Every
  downstream node reads from this node's output, not from the raw form
  fields.
- **Two parallel sends after insert.** The outbound voice agent
  (`/new-lead`) is dispatched in parallel with the Form 2 email so the
  customer hears from us within minutes whether or not they open the
  email.

## Flow B: window details &amp; AI estimate (`/apex-window-details-v2`)

```
Webhook (multipart)
  └─► Lookup lead by email
        └─► Validate file sizes (≤5MB per photo)
              └─► Split windows into items (each window processed individually)
                    └─► For each window:
                          ├─► IF has photo
                          │     └─► HTTP upload to Supabase Storage (window-photos bucket)
                          │           └─► Build photo URL
                          └─► ELSE: photo URL null
                                └─► INSERT windows row (lead_id, issue_type, photo_url, notes, window_number)
              └─► UPDATE leads.pipeline_stage = 'Window Details Received'
                    └─► Resolve open quote_approval_pending escalation
                          └─► Fetch all windows for lead
                                └─► Fetch pricing_config table
                                      └─► Build Claude prompt (windows + pricing + issue_description)
                                            └─► HTTP POST → Anthropic Messages API
                                                  └─► Parse AI response
                                                        └─► INSERT estimates (line_items JSONB)
                                                              └─► UPDATE leads.pipeline_stage = 'Estimate Generated'
                                                                    └─► Build PDF HTML template
                                                                          └─► HTTP POST → PDFShift
                                                                                └─► Upload PDF to Supabase Storage
                                                                                      └─► UPDATE estimates.pdf_url
                                                                                            └─► Email SM (estimate review)
                                                                                                  └─► Email customer (acknowledgement)
                                                                                                        └─► HTTP POST → /quote-approval-ready (triggers WF2)
                                                                                                              └─► Respond 200
```

The estimate is generated from `windows.notes` + `windows.photo_url` +
the pricing config. **There are no width / height / glass-type fields on
Form 2 — the form was deliberately kept short to maximize completion
rate, and Claude infers what it needs from the photos and free-text
notes.** Manual SM adjustment is supported via `quotes.adjusted_total`
+ `manually_adjusted` flag (handled in [WF2](wf2-quote-and-payment.md)).

## Where the data lands

| Table | Operation | Fields written |
|---|---|---|
| `leads` | INSERT | name, email, phone, address, issue_description, num_windows, pipeline_stage='New Lead' |
| `leads` | UPDATE | pipeline_stage transitions through 'Window Details Received' → 'Estimate Generated' |
| `windows` | INSERT (one per submitted window) | lead_id, issue_type, photo_url, notes, window_number |
| `estimates` | INSERT | lead_id, line_items, subtotal, tax_amount, total |
| `estimates` | UPDATE | pdf_url after PDFShift upload |
| `escalations` | INSERT | type='quote_approval_pending', due_at=now+24h |
| `escalations` | UPDATE | resolved=true once Form 2 submitted |
| `activity_log` | INSERT | event_type='lead_created' / 'estimate_generated' |

## External calls

| Service | Why | Node |
|---|---|---|
| OpenAI | spam / coherence check on issue_description | `AI - Lead Validation (OpenAI)` |
| Anthropic Claude | structured line-item estimate from photos + notes + pricing | `HTTP Request - OpenAI API` |
| Supabase Storage | upload window photos and the generated PDF | `HTTP Request - Upload Photo`, `HTTP Request - Upload PDF` |
| PDFShift | HTML → PDF for the estimate document | `HTTP Request - PDFShift` |
| Gmail SMTP | SM and customer emails | several `Send Email` nodes |

## Engineering note: the placeholder-output pattern

The window-details branch chains 20+ Supabase nodes. Several of them have
empty-result edge cases that would silently halt the pipeline — for
example, on an inbound-call lead (created by [WF7](wf7-inbound-call.md)),
no `quote_approval_pending` escalation row exists, so the
`Update Escalation` node returns 0 items, which by default makes n8n skip
every downstream node.

The fix: enable `alwaysOutputData: true` on the escalation update so it
emits an empty `{}` even on no-match. Downstream nodes then read the
`lead_id` they need from a fixed upstream node — not from `$json` — so
the empty pass-through doesn't poison the chain. The same fix is applied
in [WF3 Post-Call](wf3-post-call.md) for empty-calendar handling.

---

[← back to README](../../README.md) · [← workflow index](README.md)
