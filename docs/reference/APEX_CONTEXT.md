> **Note:** This is the project's internal context document, kept here as a reference artifact. Infrastructure secrets, credential IDs, workflow IDs, and personal contact details have been replaced with placeholders.

> [← back to README](../../README.md)

---

# APEX CONTEXT — 2026-04-29

---

## 1. PROJECT SNAPSHOT

**Client:** Apex Windows TX — foggy glass / window replacement and repair, 50+ cities, Dallas-Fort Worth, TX. **apexwindowstx.com**
**Built by:** OyeLabs (Samarth Kumar). **Sales Managers (human gate):** Alvin (`sales-manager@example.com`), Dan.
**Purpose:** Replace 100% manual sales cycle (form → call → quote → invoice) with zero-touch automation. Only deliberate human touchpoint: SM quote approval.

| Layer | Technology | Detail |
|---|---|---|
| Automation Engine | n8n self-hosted | Contabo VPS `{{N8N_BASE_URL}}`, Caddy reverse proxy |
| Database + Storage | Supabase PostgreSQL | Project `{{SUPABASE_PROJECT_ID}}`, region `ap-southeast-2` (Sydney) |
| AI Estimation | Anthropic Claude API | `claude-sonnet-4-20250514` — line-item estimate from photos+notes |
| Transcript AI | OpenAI GPT-4 | Post-call analysis + delay message generation |
| Voice Agent | ElevenLabs Conversational AI | Outbound calls (agent: Eric/Maya) AND inbound calls (agent: Alex) — both via Twilio native integration |
| Voice/SMS Carrier | Twilio | All SMS: reminders, fallbacks, retry sequences, error alerts |
| Email | Gmail (SMTP + Gmail API) | All customer + internal emails |
| Payments | Stripe | Payment links, `checkout.session.completed` webhook |
| PDF | PDFShift API | HTML → PDF conversion, stored in Supabase Storage |
| Calendar | Google Calendar API | Appointment availability + booking |
| CRM | HubSpot | Backfilled mirror of leads; not live-synced from n8n |
| Call Logging | Google Sheets | Post-call outcome log for team |

**Deployment URLs:** n8n: `https://{{N8N_BASE_URL}}` · CRM: `https://apexwindowscrm.vercel.app` · Form 2: `https://windowdetails.vercel.app` · Form 1: `https://apexwindowstx.com` (or LeadFlow deploy)

---

## 2. SYSTEM ARCHITECTURE

```
Customer
  │
  ├─[Form 1: /apex-lead-v2]──────────────────────────────────────────────────┐
  │                                                                            │
  │                                                              WF1 — Lead Capture ({{WF1_ID}})
  │                                                             ┌──────────────────────────────────────┐
  ├─[Form 2: /apex-window-details-v2]──────────────────────────►│ Validate → Dedup → Create Lead       │
  │                                                             │ OpenAI spam check → Supabase INSERT  │
  │                                                             │ Upload photos → Supabase Storage     │
  │                                                             │ Claude AI estimate → PDFShift PDF    │
  │                                                             │ Email SM → Trigger WF2               │
  │                                                             └──────────────────┬───────────────────┘
  │                                                                                │ POST /quote-approval-ready
  │                                                                                ▼
  │                                               WF2 — Quote Approval ({{WF2_ID}})
  │                                              ┌─────────────────────────────────────────────────────┐
  │                                              │ Flow A: Register quote, generate accept token        │
  │                                              │ Flow B: [SM approves in CRM] → email quote + token  │
  │                                              │ Flow C: Customer clicks Accept → pipeline advances   │
  │                                              │ Flow D: SM marks job done → Stripe invoice link      │
  │                                              │ Flow E: Stripe webhook → Closed Won → trigger WF3   │
  │   [quote-decline/accept GET pages] ──────────►│ Decline: direct Supabase PATCH (bypasses n8n!)     │
  │                                              └───────────────────────────┬─────────────────────────┘
  │                                                                           │ POST /closed-won
  │                                                                           ▼
  │            WF2B — Reminder System ({{WF2B_ID}}) ◄── hourly schedule ─┐
  │            [quote + payment reminder emails, auto-decline on day 3]        │
  │                                                                             │
  │            WF3 Post-Payment ({{WF3_POSTPAY_ID}}) ◄── /closed-won + hourly ──┘
  │            [review requests, newsletter, escalation engine, error alerts]
  │
  ├─[new lead trigger]──► WF3 Outbound Call ({{WF3_OUTBOUND_ID}})
  │                        [time check → ElevenLabs call OR Twilio SMS]
  │                        [daily 9am retry cron for missed calls]
  │
  ├─[Inbound call to Twilio number]──► ElevenLabs (Alex agent)
  │  │                                  ├─[/webhook/inbound-lead]──► WF7 — Inbound Call Lead Capture ({{WF7_ID}})
  │  │                                  │                            [Validate → Supabase INSERT lead → Email Form 2 link]
  │  │                                  │                            [Returns lead_id to agent]
  │  │                                  ├─[/webhook/GetAvailability] (shared with outbound)
  │  │                                  ├─[/webhook/BookAppointment] (shared with outbound)
  │  │                                  ├─[/webhook/update_form_status] (shared with outbound)
  │  │                                  └─[/webhook/transfer_unavailable_handler] (shared with outbound)
  │  │                                       ↓
  │  │                                  Customer fills Form 2 → joins existing pipeline at WF1 Flow B
  │  
  └─[/post_call_webhook]──► WF3 Post-Call ({{WF3_POSTCALL_ID}})
  │ [/update_form_status]    [GPT-4 analysis → Supabase update → Google Sheets]
  │ [/BookAppointment]       [Google Calendar → appointments table → confirm email]
  └─[/GetAvailability]       [Google Calendar read → available slots response]
```

**Lead data flow (E2E):** Form 1 → WF1 validates/creates `leads` row with UUID → Form 2 → WF1 uploads photos to `window-photos` bucket, inserts `windows` rows, calls Claude → inserts `estimates`, generates PDF to `service-pdfs`, emails SM → triggers WF2 → SM approves in CRM → customer receives quote email with 64-char token → customer accepts → SM marks job done → Stripe payment link generated → customer pays → `Closed Won` → WF3 sends review email after 30-min wait → hourly escalation engine monitors all active leads.

**Inbound call data flow (alternative entry):** Customer dials Twilio number → routed to ElevenLabs Alex agent → agent collects name/email/address/window count/issue → calls `/webhook/inbound-lead` → WF7 creates `leads` row (`pipeline_stage='Inbound Call - Form 2 Pending'`) + sends Form 2 email → returns `lead_id` to agent → agent uses `lead_id` for `BookAppointment` + `update_form_status`. Once customer submits Form 2, flow merges with the standard WF1 Flow B path (window details → estimate → WF2 → ...).

---

## 3. DATABASE SCHEMA (Supabase — `{{SUPABASE_PROJECT_ID}}`)

**RLS status:** Configured on all tables. Anon key used for CRM reads — verify policies. Service-role JWT hardcoded in several n8n nodes (security debt — see §8).

### `leads` — master record, one row per customer
| Column | Type | Notes |
|---|---|---|
| id | uuid PK | `gen_random_uuid()` — master FK for all tables |
| name | text | Full name (title case) |
| email | text | Lowercase |
| phone | text | E.164 e.g. `+12145551234` |
| address | text | Assembled by Form 1 JS: `streetAddress + ', ' + city + ', TX'` |
| issue_description | text | Customer freetext |
| pipeline_stage | text | State machine — see canonical values below |
| num_windows | int | From Form 1 |
| service_request_pdf_url | text | Set by WF1 after PDFShift upload |
| invoice_number | text | Set by WF2 Flow D |
| invoice_sent_at | timestamptz | Set by WF2 Flow D |
| closed_won_at | timestamptz | Set by WF2 Flow E |
| google_review_requested_at | timestamptz | Set by WF3 |
| last_newsletter_sent_at | timestamptz | Set by WF3 |
| Lead_score | int | Set by post-call AI agent |
| lead_temp | text | `hot`/`warm`/`cold` — set by post-call processing |
| qualification_stage | text | Set by post-call AI |
| customer_intent | text | GPT-4 extracted from call transcript |
| recomonded_action | text | GPT-4 next action (typo in column name — do not fix) |
| Voice Agent | text | `call`/`sms` — contact method used |
| conversation_id | text | ElevenLabs conversation ID |
| call_attempts | int | Incremented by retry scheduler |
| form_status | text | Set by voice agent via `/update_form_status` |
| created_at | timestamptz | `now()` |

**Canonical `pipeline_stage` values (exact strings — any variation breaks WF2B/WF3):**
`New Lead` (web form entry) | `Inbound Call - Form 2 Pending` (inbound call entry, WF7) → `Window Details Received` → `Estimate Generated` → `Quote Pending Approval` → `Quote Sent` → `Quote Accepted` → `Completed` → `Payment Pending` → `Closed Won` | `Closed Lost`

### `windows` — one row per window submitted via Form 2
| Column | Type | Notes |
|---|---|---|
| id | uuid PK | |
| lead_id | uuid FK→leads | |
| issue_type | text | `Foggy/Broken` \| `Full Window Replacement` \| `Other` |
| other_description | text | Only when issue_type=Other |
| photo_url | text | Public Supabase Storage URL or null |
| notes | text | Customer freetext per window |
| window_number | int | Index from Form 2 (1–10) |
| created_at | timestamptz | |

**NOTE:** Form 2 captures NO width/height/glass_type. AI estimation is based entirely on `notes` + photos.

### `estimates` — AI-generated cost breakdown
| Column | Type | Notes |
|---|---|---|
| id | uuid PK | |
| lead_id | uuid FK→leads | |
| line_items | jsonb | `[{window_number, description, base_price, multiplier, final_price}]` |
| subtotal / tax_rate / tax_amount / total | numeric | tax_rate default 0.0825 |
| customer_note / sales_manager_note | text | AI-generated per estimate |
| status | text | `pending_review` → `approved` |
| adjusted_total | numeric | Set by WF2 Flow B if SM adjusts price |
| manually_adjusted | bool | |
| created_at | timestamptz | |

### `quotes` — formal quote lifecycle
| Column | Type | Notes |
|---|---|---|
| id | uuid PK | |
| lead_id / estimate_id | uuid FK | |
| status | text | `pending_approval` → `sent` → `accepted` \| `declined` |
| acceptance_token | text | 64-char hex, `crypto.randomBytes(32)` |
| pdf_url / total_amount / approved_by | text/numeric | |
| approved_at / sent_at / accepted_at / created_at | timestamptz | |

### `invoices` — payment records
| Column | Type | Notes |
|---|---|---|
| id | uuid PK | |
| lead_id / quote_id | uuid FK | |
| invoice_number | text | Format `APX-YYYY-XXXX` |
| amount_due / amount_paid | numeric | |
| status | text | `unpaid` \| `partial` \| `paid` |
| stripe_payment_link | text | Set by WF2 d013→new-save-stripe-url |
| invoice_date / due_date | date | due = invoice + 14 days |
| paid_at / payment_method / transaction_id | timestamptz/text | |

### `appointments` — service bookings
| Column | Type | Notes |
|---|---|---|
| id | uuid PK | lead_id FK |
| appointment_date / appointment_time | date/time | |
| service_type | text | CHECK: `estimating` \| `installation` \| `quote_review_call` |
| status | text | `confirmed` default |
| booked_by / notes | text | |
| reminder_24hr_sent / reminder_2hr_sent | bool | Updated by WF3 reminder crons |

### `escalations` — scheduler timers
| Column | Type | Notes |
|---|---|---|
| id | uuid PK | lead_id FK |
| type | text | `quote_approval_pending` \| `customer_not_responded` \| `payment_reminder` \| `review_request` |
| due_at | timestamptz | When scheduler should act — **NULL on `payment_reminder` (bug)** |
| resolved | bool | false = active |
| last_notified_at | timestamptz | 12hr cooldown (SM alerts) / 24hr (customer) |
| notification_count | int | ≥3 → Closed Lost candidate alert |

### `activity_log` — append-only audit trail
`id, lead_id FK, event_type text, details jsonb, created_at`. Every workflow writes here.

### `pricing_config` — editable base prices
`id, service_type (foggy_glass_igu / full_window_replacement / other_service), base_price numeric, updated_at`. Loaded into Claude prompt for estimates.

### `post_job_comms` — deduplication for post-sale emails
`id, lead_id FK, comm_type (review_request / newsletter), sent_at, created_at`. Prevents double review emails when both `/closed-won` webhook and hourly cron fire simultaneously.

---

## 4. WORKFLOWS — COMPLETE TECHNICAL MAP

### WF1 — Lead Capture & Validation | `{{WF1_ID}}` | 51 nodes | ACTIVE

**Triggers:** `POST /webhook/apex-lead-v2` (Form 1) · `POST /webhook/apex-window-details-v2` (Form 2)

**Flow A — Lead Intake:**

| Node | Type | Purpose | Key Fields / Logic |
|---|---|---|---|
| Webhook - Lead Submitted | webhook | Entry | path: `apex-lead-v2`, POST |
| Set - Map Webhook to Form Fields | set | Normalize payload keys | Maps raw body → standard field names |
| HTTP Request1 | httpRequest | **[BUG]** Sends `lead_id: email` to `/webhook/new-lead` | Should send UUID, sends email string instead |
| IF - Validate Required Fields | if | Block if name/email/phone/address missing | Fails → Stop - Missing Fields |
| AI - Lead Validation (OpenAI) | httpRequest | Spam/coherence check on `issue_description` | Calls OpenAI API |
| IF - Valid Lead | if | Route on AI result | FALSE → Error - AI Validation Failed |
| HTTP Request - Dedup Check | httpRequest | Check for existing email in Supabase | Returns existing record if found |
| IF - Is Duplicate | if | Skip creation if email exists | TRUE → Supabase - Log Duplicate → Stop |
| Code - Sanitise Fields | code | lowercase email, E.164 phone, title-case name | |
| Supabase - Create Lead | supabase | INSERT `leads` | pipeline_stage='New Lead' |
| Supabase - Log Lead Created | supabase | INSERT `activity_log` | event_type='lead_created' |
| Supabase - Create Escalation | supabase | INSERT `escalations` | type='quote_approval_pending', due_at=now+24h |
| Set - Build Window Form URL | set | Build personalized Form 2 URL | Embeds lead_id |
| Send Email - Sales Manager New Lead | emailSend | Email SM with lead info | **[BUG]** Uses `$json.issue_clean` and `$json.submitted_at` — both undefined → blank fields |
| Send Email - Customer form 2 | emailSend | Email customer with Form 2 link | |

**Flow B — Window Details & Estimation:**

| Node | Type | Purpose | Key Fields / Logic |
|---|---|---|---|
| Webhook - Window Details | webhook | Entry | path: `apex-window-details-v2`, multipart POST |
| Lookup Lead by Email | supabase | Verify email exists in `leads` | Links Form 2 to existing lead |
| Code - Validate File Sizes2 | code | Check photos ≤5MB | Rejects oversized files |
| Code - Split Windows Into Items | code | Split windows array into individual items | Each window processed separately |
| If - Has Photo | if | Route per window | TRUE → upload photo; FALSE → Set - Photo URL Null |
| HTTP Request - Upload Photo | httpRequest | Upload binary to Supabase Storage | **[BUG]** `photo_mime` never set → Content-Type header undefined → upload may fail |
| Set - Build Photo URL1 | set | Construct storage URL | path: `lead_id/window_N_image_M` |
| Set - Map Window Form1 | set | **[DEAD CODE]** | Not connected to any output — remove |
| Supabase - Save Window Record | supabase | INSERT `windows` | window_number, notes, photo_url, lead_id |
| Supabase - Pipeline Window Details Received | supabase | UPDATE `leads` | pipeline_stage='Window Details Received' |
| Supabase - Update Escalation1 | supabase | Resolve open escalation | resolved=true (customer responded) |
| Supabase - Fetch All Windows2 | httpRequest | SELECT `windows` WHERE lead_id | All windows for this lead |
| Supabase - Fetch Pricing Config1 | supabase | SELECT `pricing_config` | Loaded into Claude prompt |
| Code - Build AI Prompt | code | Assemble Claude prompt | windows + pricing_config + issue_description |
| HTTP Request - OpenAI API | httpRequest | Call Anthropic Claude | Returns structured JSON estimate |
| Code - Parse AI Response | code | Parse estimate JSON | Validates structure |
| Supabase - Save Estimate | supabase | INSERT `estimates` | line_items (JSONB), total, subtotal, tax |
| Supabase - Pipeline Estimate Generated | supabase | UPDATE `leads` | pipeline_stage='Estimate Generated' |
| Code - Build PDF HTML | code | HTML template with estimate data | Branded Apex Windows layout |
| HTTP Request - PDFShift | httpRequest | POST HTML to PDFShift | Returns PDF binary |
| HTTP Request - Upload PDF | httpRequest | Upload PDF to Supabase Storage | bucket: `service-pdfs` |
| Set - Build PDF URL | set | Construct public PDF URL | |
| Supabase - Save PDF URL | supabase | UPDATE `estimates` | pdf_url set |
| Send Email - Sales Manager Review Estimate | emailSend | Email SM with estimate + PDF link | |
| Send Email - Customer Acknowledgement | emailSend | Tell customer quote is being prepared | |
| Supabase - Log Workflow 1 Complete | supabase | INSERT `activity_log` | event_type='estimate_generated' |
| HTTP POST - Trigger Workflow 2 | httpRequest | POST to `/webhook/quote-approval-ready` | Passes lead_id, estimate_id, pdf_url, total |
| Respond to Webhook | respondToWebhook | 200 to Form 2 caller | |

**Supabase tables:** `leads` (SELECT/INSERT/UPDATE), `windows` (INSERT), `estimates` (INSERT/UPDATE), `escalations` (INSERT/UPDATE), `activity_log` (INSERT), `pricing_config` (SELECT)

---

### WF2 — Quote Approval & Payment | `{{WF2_ID}}` | 71 nodes | ACTIVE

**Webhooks:** 7 entry points (see Quick Reference §10)

**Flow A — Quote Queued (`/quote-approval-ready`):**
Fetch Lead A → Fetch Estimate → Generate Token (`crypto.randomBytes(32)`) → Create Quote Record (status=pending_approval) → Build Approval URLs → Update Pipeline='Quote Pending Approval' → Log activity

**Flow B — Manager Approved (`/manager-approved`):**
Fetch Lead B → Validate Quote (status=pending_approval guard) → IF Price Adjusted → [Update Estimate Adjusted] → Set Final Total → Build Quote Email HTML → Email Quote to Customer → Update Quote='sent' → Update Pipeline='Quote Sent' → Create Escalation 72hr (customer_not_responded) → Create Quote Reminder Escalation → Email SM Quote Sent → Log

**⚠️ BUGS in Flow B:**
- b008 `Email Quote to Customer`: `toEmail` hardcoded; quote email subject has encoding artifact (`â€"` instead of em-dash); Accept/Decline URLs may use raw Contabo IP

**Flow C — Customer Accepted (`/customer-accepted`):**
Validate Token (SELECT quotes WHERE acceptance_token=token AND status=sent) → Fetch Lead C → Update Quote='accepted' → Update Pipeline='Quote Accepted' → Resolve Escalation C (direct HTTP PATCH with hardcoded service-role JWT — c005) → Email SM Quote Accepted → Email Customer Booking Confirmed → Log

**Flow C2 — Quote Accept Page (`/quote-accept` GET):**
`Webhook - Quote Accept` → `Trigger Flow C` (internal POST to `/customer-accepted`) → `Respond - Accept Page` (HTML)

**Flow C3 — Quote Decline (`/quote-decline` GET):**
`Webhook - Quote Decline` → `Respond - Decline Page` (HTML) + `Update Quote - Declined` (direct Supabase PATCH — no n8n trigger) → `Fetch Lead - Declined` → `Email SM - Quote Declined`
**⚠️ CRM decline** also patches Supabase directly via anon key — bypasses n8n entirely, no SM email, no activity_log.

**Flow D — Job Completed (`/job-completed`):**
Check Duplicate Job → IF Already Completed (TRUE branch → **[BUG]** silent dead-end, no log) → Update Pipeline='Completed' → Build Invoice Data (invoice_number=APX-YYYY-XXXX, due=+14d) → Fetch Accepted Quote → Create Invoice → Create Stripe Price (d012 — **hardcoded Stripe SK**) → Create Stripe Payment Link (d013 — **hardcoded Stripe SK**) → Set Stripe Payment URL → Save Stripe URL to Invoice (new-save-stripe-url — **hardcoded service-role JWT**) → Update Pipeline='Payment Pending' → Create Payment Reminder Escalation (**[BUG]** due_at not set → NULL) → Email Invoice (d010 reads email from Check Duplicate Job node — fragile)

**Flow E — Payment Received (Stripe + manual):**
`Stripe Webhook` (`/stripe-payment`) → Parse Stripe Event (filters `checkout.session.completed`) → Normalize Payment → Validate Invoice → Payment Check Data → IF Full Payment:
- **Full:** Update Invoice Paid → Update Pipeline='Closed Won' → Resolve All Escalations (e010 — **hardcoded service-role JWT**) → Fetch Lead E → Email Customer Payment Confirmed → Email SM Closed Won → Trigger WF3 (**[BUG]** e014 reads lead_email/lead_name from UPDATE response → undefined) → Log Closed Won
- **Partial:** Handle Partial Payment → Update Invoice Partial → Email SM Partial (**[BUG]** e004: no customer email sent)

**Supabase tables:** `leads`, `quotes`, `estimates`, `invoices`, `escalations`, `activity_log`
**External:** Stripe API (d012/d013), Gmail SMTP (all email nodes)

---

### WF2B — Reminder System | `{{WF2B_ID}}` | 20 nodes | ACTIVE

**Trigger:** Schedule (hourly — exact interval [UNKNOWN - verify])
**Purpose:** Sends quote reminders and payment reminders for overdue escalations; auto-declines quote on day 3.

| Node | Purpose |
|---|---|
| Schedule Trigger | Hourly cron |
| Build Date Filter | Code: calculates current date window |
| Fetch Quote Reminder Escalations | HTTP GET: escalations type=quote_reminder, resolved=false, overdue |
| Fetch Payment Reminder Escalations | HTTP GET: escalations type=payment_reminder, resolved=false, overdue |
| Split Quote Escs / Split Payment Escs | Code: flatten to individual items |
| Fetch Lead Q/P + Fetch Quote Data/Invoice | HTTP GET: populate email context |
| Merge Quote/Payment Data | Code: assemble email data |
| Send Quote/Payment Reminder Email | emailSend: **[BUG]** both subjects have `[TEST]` prefix — remove before production |
| Update Quote/Payment Escalation | HTTP PATCH: increment notification_count, update last_notified_at |
| Is Day 3? | IF: notification_count >= 3 |
| Auto-Decline Quote | HTTP PATCH: set quote status=declined |
| Update Pipeline - Quote Expired | HTTP PATCH: update leads.pipeline_stage |
| Resolve Quote Escalation | HTTP PATCH: resolved=true |

**⚠️ Bug:** `payment_reminder` escalation has NULL `due_at` (set by WF2 d-flow) — may crash filter query.
**Supabase tables:** `escalations`, `leads`, `quotes`, `invoices` (all via direct HTTP requests)

---

### WF3 — Post-Payment Followups | `{{WF3_POSTPAY_ID}}` | 82 nodes | ACTIVE

**Triggers:** `POST /webhook/closed-won` + `Schedule Trigger — Every 1 Hour` (gated: 6am–10pm CST)

**Closed-Won webhook path:**
Respond 200 Immediately → Wait 30 Minutes → Get a row (fetch lead by email) → Check Review Already Sent → IF not sent → Send Email Google Review Request → INSERT `post_job_comms` → UPDATE `leads.google_review_requested_at` → UPDATE `leads.last_newsletter_sent_at` → Log Activity Post-Sale Sequence Complete → Email SM summary

**Hourly scheduler (4 parallel branches after operating hours gate):**

| Branch | Query | Cooldown | Action |
|---|---|---|---|
| Check 1: SM Overdue Escalations | `escalations` type=`quote_approval_pending`, due_at<now | 12hr on `last_notified_at` | Loop: Calc hours overdue → Fetch Lead → Send SM+Supervisor OVERDUE ALERT email → IF type=quote_approval_pending → AI Delay Message (OpenAI) → Parse → Email customer → Twilio SMS to SM → Update escalation notified |
| Check 2: Customer No-Response | `escalations` type=`customer_not_responded`, due_at<now | 24hr cooldown | Loop: Fetch lead + quote accept link → Email customer follow-up → Email SM alert → IF notification_count≥3: Email Closed Lost Candidate + Twilio SMS → UPDATE escalation |
| Check 3: Review Request Fallback | `escalations` type=`review_request`, overdue | None | Loop: Check `post_job_comms` dedup → Verify lead stage = complete → Send Google Review email → INSERT `post_job_comms` → Resolve escalation |
| Check 4: Quarterly Newsletter | `leads` WHERE stage IN ('Completed','Payment Pending','Closed Won') AND `last_newsletter_sent_at` < now-90d | 90 days | Limit 50/run → 2-sec wait (rate limit) → Send newsletter email → UPDATE `leads.last_newsletter_sent_at` → Log activity |

**Error handler:** `errorTrigger` → Gmail alert email + Twilio SMS to 2 team members on any scheduler failure.
**Supabase tables:** `leads`, `escalations`, `post_job_comms`, `activity_log`

---

### WF3 — Outbound Call | `{{WF3_OUTBOUND_ID}}` | 11 nodes | ACTIVE

**Trigger:** `POST /webhook/new-lead` (called by WF1 after lead created)
**Daily retry cron:** 9am CST — fetches leads with missed calls, increments attempts.

| Node | Purpose | Logic |
|---|---|---|
| Webhook: /new-lead | Entry | |
| Code: Validate & Normalize | Validate required fields, E.164 phone | |
| IF: Payload Valid? | Gate | FALSE → Respond 400 |
| Code: Time Check 8AM-8PM | Calculate current CST hour | `America/Chicago` timezone |
| IF: Within Call Hours? | Route | TRUE → ElevenLabs call; FALSE → Twilio SMS |
| elevan lab agent | httpRequest | POST to ElevenLabs API with dynamic vars: customer_first_name, customer_address, customer_issue_description, num_windows, second_form_completed |
| Send an SMS/MMS/WhatsApp message | twilio | Out-of-hours fallback SMS |
| Respond: 200 Success | respondToWebhook | |
| Code in JavaScript | Check ElevenLabs conversation_id | |
| Update a row | supabase | UPDATE `leads.conversation_id` |
| Schedule Trigger (daily 9am) + Get many rows2 → update lead attempt reach → IF: Keep Retrying? | Retry loop | <7 attempts: Wait 24hr → retry SMS; ≥7: Wait 7d → weekly re-engagement SMS |

**⚠️ Voice Agent Bugs:**
- System prompt names agent "Eric"; objection-handling script says "I'm Maya" — naming inconsistency
- Dynamic variables inject `email = {{lead_id}}` (UUID); Step 1 script uses `{{customer_email}}` → undefined

---

### WF3 — Post-Call Processes | `{{WF3_POSTCALL_ID}}` | 79 nodes | ACTIVE

**Webhooks:** `POST /webhook/post_call_webhook` · `POST /webhook/update_form_status` · `POST /webhook/BookAppointment` · `POST /webhook/GetAvailability`

**Post-call path:**
`post call webhook` → `IF` (connected/missed?) → **Connected:** Code in JavaScript (format transcript) → AI Agent (GPT-4 backed by OpenAI Chat Model — extracts intent, interest level, appointment request) → Code in JavaScript1 (parse output) → Get a row2 (fetch lead) → Update a row (UPDATE `leads`) + Append or update row in sheet (Google Sheets) → IF1 (appointment requested?) → send mail to expert + thank you (Twilio SMS)
**Missed call:** Get Leads → send sms (Twilio missed-call SMS) → update lead

**update_form_status path:**
`update_form_status` webhook → Message a model (OpenAI) → Code in JavaScript2 → IF2 → Get a row3 → Update a row1 (UPDATE leads) → [conditional] Get a row1 → Send form link email (disabled) + send link of form Twilio SMS (disabled) → Update a row2

**BookAppointment path:**
`Book Appointment Webhook` → Parse & Validate Booking → Booking Valid? → data (Set) → Book appointment (Google Calendar) → Merge Booking Data → Get Lead from Supabase → Save Appointment to Supabase (INSERT `appointments`) → Send Confirmation Email (Gmail) → Build Booking Response → Respond - Booking Success
**service_type mapping:** "Quote Review Call"→`quote_review_call`, "Installation"→`installation`, else→`estimating`

**GetAvailability path:**
`Get Availability Webhook` → Parse Availability Request *(auto-corrects past dates to today)* → Availability Valid? → Get Calendar Events *(alwaysOutputData: true)* → Code: Ensure Calendar Items *(placeholder if calendar empty)* → Build Available Slots (code) → Respond - Availability
**Fixed 2026-04-30:** Empty calendar no longer skips slot generation. LLM year confusion (2024 dates) auto-corrected to today in both Parse Availability Request and Parse & Validate Booking.

**Appointment reminder crons (in this workflow):**
- Daily 8am CT: `Get many rows` (appointments WHERE date=tomorrow) → IF any → Split → Format Time → Twilio 24hr SMS → Update a row3
- Every 30 min: `Get many rows1` → Filter 2hr window → IF any → Split → Build 2hr message → Twilio SMS → Update a row4

**Supabase tables:** `leads` (UPDATE), `appointments` (INSERT, UPDATE), `windows` (read for photos)

---

### WF7 — Inbound Call Lead Capture | `{{WF7_ID}}` | 8 nodes | ACTIVE (created 2026-04-29)

**Trigger:** `POST /webhook/inbound-lead` (called by ElevenLabs Alex agent's `create_lead` tool)
**Purpose:** When a customer calls in (rather than submitting Form 1), the inbound voice agent collects the same fields Form 1 would and posts them here. WF7 creates the lead in Supabase, sends the Form 2 email, and returns the `lead_id` so the agent can chain `BookAppointment` and `update_form_status` calls.

| Node | Type | Purpose | Key Fields / Logic |
|---|---|---|---|
| Webhook: /inbound-lead | webhook | Entry | path: `inbound-lead`, POST, responseMode: responseNode |
| Code: Validate & Normalize | code | Trim/lowercase/E.164 normalization | Required: customer_name, phone_number, email, customer_address; Optional: issue_description (1000 char cap), window_count (parseInt fallback to 0) |
| IF: Valid? | if | Gate | TRUE → Supabase create; FALSE → Respond 400 |
| Respond: 400 Invalid | respondToWebhook | Validation failure | Returns `{ success: false, errors: [...] }` |
| Supabase: Create Lead | supabase | INSERT `leads` row | Fields: name, phone, email, address, issue_description, pipeline_stage='Inbound Call - Form 2 Pending'. Credential: `{{CRED_SUPABASE}}`. NOTE: does NOT insert `num_windows`, `lead_source`, or `activity_log` row — possible future enhancement |
| Code: Extract Lead ID | code | Pull UUID from Supabase response and merge with validated data | Returns lead_id, customer_name, email, phone_number, address, issue_description, window_count |
| Email: Send Form 2 Link | emailSend | Gmail SMTP — sends branded Form 2 email | From: `sales-manager@example.com`. To: customer email. Subject: "Action Required: Hi [first name], complete your window details". Body links to `https://windowdetails.vercel.app/`. Credential: `{{CRED_GMAIL_SMTP}}` |
| Respond: 200 Success | respondToWebhook | Final response to ElevenLabs agent | Returns `{ success, lead_id, email, customer_name, message }` |

**Expected payload from agent:**
```json
{
  "customer_name": "John Smith",
  "phone_number": "+12145551234",  // ElevenLabs system__caller_id
  "email": "john@example.com",
  "customer_address": "123 Main St, Dallas, TX",
  "issue_description": "Foggy windows in living room",
  "window_count": 3
}
```

**Response to agent:**
```json
{
  "success": true,
  "lead_id": "uuid-string",
  "email": "john@example.com",
  "customer_name": "John Smith",
  "message": "Lead created. Form 2 link sent to email."
}
```

**Inbound voice agent (ElevenLabs):** Name `Alex`. First message: "Thank you for calling Apex Windows TX! My name is Alex — how can I help you today?" Tools attached: `create_lead` (this WF7 webhook), `get_availability`, `book_appointment`, `update_form_status` (all shared with outbound agent), `transfer_to_number` (system tool, Conference type), `transfer_unavailable` (shared webhook).

**Dynamic variable chain in agent:** `lead_id` is captured from `create_lead`'s response via Dynamic Variable Assignment (JSON Path: `lead_id`), then passed as Dynamic Variable to all subsequent tool calls. `phone_number` / `customer_phone` use Dynamic Variable → `system__caller_id`.

**Inbound-specific `form_status` values used by Alex agent:**
- `Contacted - Inbound Call - Appointment Booked`
- `Contacted - Inbound Call - On-Site Estimate Booked`
- `Contacted - Inbound Call - Callback Scheduled`
- `Contacted - Transfer Failed - Callback Pending`
- `Inbound Call - No Info Collected`

**Convergence with main pipeline:** After WF7 sends the Form 2 email, the customer submits at `windowdetails.vercel.app` → triggers WF1's `/apex-window-details-v2` webhook → standard estimate generation → WF2 → ... (identical to web Form 1 path).

**Supabase tables:** `leads` (INSERT only)

**Twilio integration:** Twilio number imported into ElevenLabs via native integration (Account SID + Auth Token). ElevenLabs auto-configures Twilio's voice webhooks. Inbound agent assignment: `Inbound Call - Apex Windows TX`. Outbound continues to use existing agent_id passed by WF3-outbound.

---

## 5. FRONTEND PAGES

### Form 1 — Lead Intake (`ApexWindows_LeadFlow/index.html`)
**Deploy:** `apexwindowstx.com` (or LeadFlow Vercel deploy) | **Webhook:** `POST https://{{N8N_BASE_URL}}/webhook/apex-lead-v2`

| HTML Field | ID | Payload Key | Validation |
|---|---|---|---|
| First Name | firstName | `name` (joined w/ lastName) | Required, non-empty |
| Last Name | lastName | `name` (joined) | Required |
| Email | email | `email` | `.includes('@')` only — **weak: 'a@' passes** |
| Country Code | countryCode | prepended to phone | Select dropdown, default +1 |
| Phone | phone | `phone` | Non-empty digits only; assembled as `countryCode + digits.replace(/\D/g,'')` |
| Street Address | streetAddress | `address` (part) | Required |
| City | city | `address` (part) | Required; assembled: `streetAddress + ', ' + city + ', TX'` — **state hardcoded** |
| Number of Windows | numWindows | `num_windows` | parseInt ≥1 AND ≤10 (HTML max=10 + JS check) |
| Describe Your Issue | issueDescription | `issue_description` | Required textarea |

**Payload sent:** `{ name, phone, email, address, issue_description, num_windows }` as JSON.

### Form 2 — Window Details (`ApexWindowDetails/index.html`)
**Deploy:** `windowdetails.vercel.app` | **Webhook:** `POST https://{{N8N_BASE_URL}}/webhook/apex-window-details-v2` (multipart/form-data)

| Field | Key | Logic |
|---|---|---|
| Email | `email` | Required; `.includes('@')` validation |
| Issue Type | `issue_type` | Required select: `Foggy/Broken` / `Full Window Replacement` / `Other` |
| Other Description | `other_description` | Appended only when issueType==='Other' |
| Windows 1–10 | `photo_w{N}` (binary) + `windows_meta` (JSON) | Window included if photo OR notes non-empty; `windows_meta = [{window_number, notes}]` — **no width/height/glass_type** |

Accept filter: `.jpg,.jpeg,.png` (advisory only — server must validate MIME). Max 5MB per file (client enforced by `handleFile()`).

### CRM — Manager Portal (`ApexWindows_CRM/index.html`)
**Deploy:** `apexwindowscrm.vercel.app` | **Auth: NONE** — publicly accessible with URL

| Action | Method | Target |
|---|---|---|
| Load leads / quotes | GET | Supabase REST API with anon key (hardcoded in HTML source — **security issue**) |
| Approve quote | POST | `/webhook/manager-approved` with `{quote_id, lead_id, approved_by, adjusted_price?}` |
| Decline quote | PATCH | Supabase REST directly `quotes?id=eq.{id}` — **bypasses n8n, no SM notification, no activity_log** |
| Mark job complete | POST | `/webhook/job-completed` with `{lead_id, marked_by: 'Alvin'}` — marked_by hardcoded |

**Bug:** Timestamp display hardcoded to IST (`Asia/Kolkata`) — SM in Texas sees wrong timezone.

### Voice Intake — Inbound Call (no UI; equivalent of Form 1)
**Surface:** Twilio number imported into ElevenLabs · **Agent:** Alex (Inbound Call - Apex Windows TX) · **Webhook:** `POST /webhook/inbound-lead` (WF7)

The inbound voice agent collects exactly the same fields Form 1 captures, in this order, one question at a time:

| Voice Question | Maps to | Source |
|---|---|---|
| "Can I start with your full name?" | `customer_name` | LLM extraction |
| (caller ID — never asked) | `phone_number` | ElevenLabs `system__caller_id` dynamic variable |
| "What's the best email for your quote documents?" | `email` | LLM extraction (agent confirms back) |
| "What's the address of the property?" / "And that's in [city], Texas?" | `customer_address` | LLM extraction (always assembled with `, [city], TX`) |
| "Roughly how many windows?" | `window_count` | LLM extraction |
| "What's going on — foggy film, full replacement, or something else?" | `issue_description` | LLM extraction (customer's own words) |

After collection, agent calls `create_lead` (WF7) → receives `lead_id` → uses it for downstream `book_appointment` and `update_form_status` calls. The Form 2 email is sent automatically by WF7; the agent only has to inform the customer.

---

## 6. INTEGRATION POINTS & EXTERNAL SERVICES

| Service | Role | Used In | Credentials |
|---|---|---|---|
| Anthropic Claude | AI line-item estimate from windows+notes+pricing | WF1 (HTTP Request - OpenAI API node) | OpenAI - ApexWindows: `{{CRED_OPENAI}}` |
| OpenAI GPT-4 | Spam validation, post-call transcript analysis, delay message generation, update_form_status analysis | WF1, WF3 Post-Call, WF3 Post-Payment | Same credential |
| ElevenLabs | Outbound voice calls; agent named Eric (system) / Maya (objection script — inconsistency) | WF3 Outbound | ElevenLabs API key in HTTP node |
| Twilio | SMS: OOH fallback, reminders (24hr/2hr), retry sequences, closed-lost SMS, error alerts; voice carrier for ElevenLabs | WF3 Outbound, WF3 Post-Call, WF3 Post-Payment | Twilio credential in n8n |
| Stripe | Payment links, `checkout.session.completed` webhook | WF2 d012/d013/e_stripe | SK hardcoded in d012/d013 — **must move to credential**; Product: `{{STRIPE_PRODUCT_ID}}`; Webhook: `{{STRIPE_WEBHOOK_ID}}` |
| PDFShift | HTML→PDF for estimates | WF1 | Auth: `Basic {{PDFSHIFT_BASIC_AUTH}}` hardcoded |
| Google Calendar | Appointment availability + booking | WF3 Post-Call | Google Calendar n8n credential |
| Google Sheets | Post-call outcome log | WF3 Post-Call | Google Sheets n8n credential |
| Gmail / SMTP | All customer + internal emails | WF1, WF2, WF3 | Gmail SMTP - ApexWindows: `{{CRED_GMAIL_SMTP}}`; Gmail account 2 (booking): `{{CRED_GMAIL_BOOKING}}` |
| HubSpot | Lead/deal mirroring (backfilled; not live-synced) | WF1 (HTTP to HubSpot API) | HubSpot API credential |
| Supabase | DB + Storage | All WFs | Supabase - ApexWindows: `{{CRED_SUPABASE}}`; service-role JWT hardcoded in c005/e010/new-save-stripe-url/storage uploads |

---

## 7. CURRENT SYSTEM STATE

### All 7 Workflows — ACTIVE in n8n

| WF | Name | Status | Last Verified |
|---|---|---|---|
| WF1 `{{WF1_ID}}` | Lead Capture & Validation | Active — partially buggy | 2026-04-23 |
| WF2 `{{WF2_ID}}` | Quote Approval & Payment | Active — Flow E tested E2E ✓ | 2026-04-13 |
| WF2B `{{WF2B_ID}}` | Reminder System | Active — `[TEST]` in subjects | 2026-04-23 |
| WF3 `{{WF3_POSTPAY_ID}}` | Post-Payment Followups | Active | 2026-04-23 |
| WF3 `{{WF3_OUTBOUND_ID}}` | Outbound Call | Active | 2026-04-23 |
| WF3 `{{WF3_POSTCALL_ID}}` | Post-Call Processes | Active | 2026-04-23 |
| WF7 `{{WF7_ID}}` | Inbound Call Lead Capture | Active — created 2026-04-29 | 2026-04-29 |

### Known Bugs (all unresolved as of 2026-04-25)

| # | WF | Node | Bug | Severity |
|---|---|---|---|---|
| WF1-1 | WF1 | HTTP Request1 | Sends `lead_id: email` (string) instead of UUID to `/webhook/new-lead` | Critical |
| WF1-2 | WF1 | Send Email - Sales Manager New Lead | Uses `$json.issue_clean` / `$json.submitted_at` — both undefined at this node → blank fields in SM email | Critical |
| WF1-3 | WF1 | HTTP Request - Upload Photo | `photo_mime` never set → Content-Type header undefined → Supabase upload may fail silently | Critical |
| WF1-4 | WF1 | Set - Map Webhook (address parsing) | Address without comma → city=undefined → validation fails (only affects direct API bypass; Form 1 JS auto-adds comma) | High |
| WF1-5 | WF1 | Set - Map Window Form1 | Dead code — outputs to `Supabase - Save Window Record` but has no input connection | Low |
| WF1-6 | WF1 | Supabase - Log Duplicate | Reads `$json.id` but Supabase returns array → may fail silently | Medium |
| WF2-1 | WF2 | e014 Trigger WF3 | Reads `lead_email`/`lead_name` from UPDATE response (undefined) → WF3 receives no customer identity | Critical |
| WF2-2 | WF2 | new-pay-reminder-esc | `payment_reminder` escalation inserted without `due_at` → NULL → may violate NOT NULL constraint | Critical |
| WF2-3 | WF2 | b008 Build Quote Email HTML | Subject has `â€"` encoding artifact instead of em-dash | High |
| WF2-4 | WF2 | d010 Email Invoice | Reads customer email from `Check Duplicate Job` node output (fragile — may be undefined) | Critical |
| WF2-5 | WF2 | b007 Build Quote Email HTML | Accept/Decline URLs may use raw Contabo IP instead of domain | High |
| WF2-6 | WF2 | e004 Handle Partial Payment | No customer email sent on partial payment | Critical |
| WF2-7 | WF2 | d003 IF Already Completed | TRUE branch (already completed) is a silent dead-end — no log, no response | High |
| WF2B-1 | WF2B | email-q / email-p | `[TEST]` prefix in both email subjects (quote reminder + payment reminder) | High |
| WF3-1 | WF3 PostCall | Build Available Slots | ~~0 calendar events → node skipped → empty GetAvailability response~~ **FIXED 2026-04-30**: Added `Code: Ensure Calendar Items` node + `alwaysOutputData: true` on `Get Calendar Events` | ~~Critical~~ Fixed |
| WF3-2 | WF3 PostCall | Parse Availability Request + Parse & Validate Booking | ~~LLM (GPT-4o) has no current date context; sends year 2024 for all dates → n8n rejects as "in the past"~~ **FIXED 2026-04-30**: Both nodes auto-correct past-year dates to current year | ~~Critical~~ Fixed |
| WF1-7 | WF1 | Supabase - Update Escalation1 | ~~Inbound call leads (WF7) have no escalation record → 0 items returned → all estimation nodes skipped silently~~ **FIXED 2026-04-30**: `alwaysOutputData: true` on escalation update node | ~~Critical~~ Fixed |
| WF1-8 | WF1 | Supabase - Fetch All Windows2 | ~~Reads `$json.lead_id` from upstream node; when escalation update returns `{}` (empty), lead_id is undefined → Supabase 400 error~~ **FIXED 2026-04-30**: URL now reads `$('Code - Validate File Sizes2').first().json.lead_id` | ~~Critical~~ Fixed |
| VA-1 | Voice | ElevenLabs prompt | Agent named "Eric" in system prompt, "Maya" in objection script — naming inconsistency | High |
| VA-2 | Voice | ElevenLabs dynamic vars | `email: {{lead_id}}` in dynamic vars; script uses `{{customer_email}}` → undefined in Step 1 | Critical |
| SEC-1 | CRM | index.html | Supabase anon key exposed in HTML source | Critical |
| SEC-2 | CRM | Decline action | Direct Supabase PATCH bypasses n8n — no SM notification, no activity_log | Critical |
| SEC-3 | WF2 | d012/d013 | Stripe SK hardcoded in node config | Critical |
| SEC-4 | WF2 | c005/e010/new-save-stripe-url | Supabase service-role JWT hardcoded in node config | Critical |

---

## 8. CRITICAL TECHNICAL NOTES

**Architecture invariants:**
- `lead_id` UUID is generated ONLY by `Supabase - Create Lead` in WF1. Never generate a new one. All workflows receive it via webhook payload.
- `pipeline_stage` strings are exact — typos silently break WF2B and WF3 filter queries. Copy from §3.
- Token security: 64-char hex = 256-bit entropy. Stateless — no session. Token IS the customer's identity for quote acceptance.
- WF3 `/closed-won` webhook must respond 200 immediately before the 30-min wait, otherwise WF2 times out. Architecture: `Respond 200 — Immediately` fires before `Wait — 30 Minutes`.
- Duplicate review prevention: both `/closed-won` webhook and hourly scheduler can fire review emails. `post_job_comms` NOT IN check prevents duplicates.

**n8n-specific patterns:**
- `patchNodeField` only works for string fields — use `updateNode` with dot-path for object fields.
- No `URLSearchParams` in n8n code sandbox — use `encodeURIComponent` + string concatenation.
- WF2 webhooks use `responseMode: onReceived` (immediate 200) — Respond to Webhook nodes removed after n8n multi-webhook validation bug.
- Supabase direct HTTP PATCH (not Supabase node) used in c005, e010 because the Supabase node lacked PATCH support at build time.
- n8n credential IDs: Supabase `{{CRED_SUPABASE}}`, Gmail SMTP `{{CRED_GMAIL_SMTP}}`, OpenAI `{{CRED_OPENAI}}`, Gmail account 2 (booking) `{{CRED_GMAIL_BOOKING}}`.

**Security debt (must fix before production):**
1. Move Stripe SK from d012/d013 → n8n credential store
2. Move service-role JWT from c005/e010/new-save-stripe-url/storage nodes → n8n env var
3. Add authentication to CRM page (currently fully public)
4. Remove Supabase anon key from CRM HTML — proxy reads through n8n or backend
5. CRM Decline must fire n8n webhook, not patch Supabase directly

**Form layer:**
- Form 1 assembles address as `streetAddress + ', ' + city + ', TX'` — state is hardcoded. Non-Texas customers will have wrong state in DB.
- Form 1 email validation: `!email.includes('@')` — accepts 'a@'. Server must validate.
- Form 2 `windows_meta` structure is `[{window_number, notes}]` — no dimensions or glass_type. Claude estimation is 100% from `notes` text + photos.
- File accept filter `.jpg,.jpeg,.png` is advisory in browsers — server-side MIME validation required.

**WF2B escalation types:**
- WF2 creates two escalation types on quote sent: `customer_not_responded` (72hr, monitored by WF3 post-payment hourly) AND a second `quote_reminder` type (monitored by WF2B). Ensure no double-firing.

---

## 9. NEXT STEPS & OPEN TASKS

### Immediate (unblocked)
1. Fix WF2 Bug #1: `e014 Trigger WF3` — fetch lead before trigger, pass real `lead_email`/`lead_name`
2. Fix WF2 Bug #2: `new-pay-reminder-esc` — set `due_at` on payment_reminder escalation
3. Fix WF2 Bug #3: Quote email subject encoding — use proper UTF-8 or replace em-dash with `-`
4. Fix WF2 Bug #6: `e004` — add customer email on partial payment
5. Remove `[TEST]` from WF2B email subjects (email-q, email-p nodes)
6. Remove `[TEST EMAIL]` footer from WF2 quote email HTML
7. Move Stripe SK to n8n credential (nodes d012, d013)
8. Move Supabase service-role JWT to n8n env var (c005, e010, new-save-stripe-url, storage upload nodes)
9. Fix GetAvailability empty-calendar edge case — return full day slots when no events exist

### Blocked / Dependent
- **Fix WF2 Bug #4** (d010 invoice email reads wrong email): needs WF2-1 fixed first (lead data available)
- **Go-live emails**: replace `developer@example.com`/`developer@example.com` with real Apex Windows addresses — needs SM sign-off on addresses
- **CRM auth**: needs hosting decision (Vercel env vars or backend proxy)
- **Voice agent naming**: confirm Eric or Maya as canonical name with SM, update prompt

### Backlog
- Verify WF3 reminder crons fire at correct times (daily 8am CST + every 30min) in production
- Confirm pricing_config table values with Alvin/Dan before go-live (foggy_glass_igu: $250, full_window: $650, other: $200, tax: 8.25%)
- Implement server-side MIME validation for photo uploads
- Add authentication to CRM
- HubSpot live sync from n8n (currently only historical backfill)
- Add Stripe webhook signature validation in WF2 `Parse Stripe Event` node

---

## 10. QUICK REFERENCE

### Workflow IDs
| WF | n8n ID | Node Count | Status |
|---|---|---|---|
| WF1 Lead Capture | `{{WF1_ID}}` | 51 | Active |
| WF2 Quote & Payment | `{{WF2_ID}}` | 71 | Active |
| WF2B Reminder System | `{{WF2B_ID}}` | 20 | Active |
| WF3 Post-Payment | `{{WF3_POSTPAY_ID}}` | 82 | Active |
| WF3 Outbound Call | `{{WF3_OUTBOUND_ID}}` | 11 | Active |
| WF3 Post-Call | `{{WF3_POSTCALL_ID}}` | 79 | Active |
| WF7 Inbound Call Lead Capture | `{{WF7_ID}}` | 8 | Active |

### Webhook URL Reference
| Path | Method | WF | Triggered By |
|---|---|---|---|
| `/webhook/apex-lead-v2` | POST | WF1 | Form 1 submission |
| `/webhook/apex-window-details-v2` | POST | WF1 | Form 2 multipart submission |
| `/webhook/new-lead` | POST | WF3 Outbound | WF1 after lead creation |
| `/webhook/quote-approval-ready` | POST | WF2 | WF1 after estimate ready |
| `/webhook/manager-approved` | POST | WF2 | CRM Approve button |
| `/webhook/quote-accept` | GET | WF2 | Customer Accept link in email |
| `/webhook/quote-decline` | GET | WF2 | Customer Decline link in email |
| `/webhook/customer-accepted` | POST | WF2 | Internal (quote-accept fires this) |
| `/webhook/job-completed` | POST | WF2 | CRM Mark Complete button |
| `/webhook/payment-received` | POST | WF2 | Manual payment entry |
| `/webhook/stripe-payment` | POST | WF2 | Stripe `checkout.session.completed` |
| `/webhook/closed-won` | POST | WF3 Post-Payment | WF2 after full payment |
| `/webhook/BookAppointment` | POST | WF3 Post-Call | Voice agent (inbound + outbound) / internal |
| `/webhook/GetAvailability` | POST | WF3 Post-Call | Voice agent (inbound + outbound) during call |
| `/webhook/post_call_webhook` | POST | WF3 Post-Call | ElevenLabs after call ends |
| `/webhook/update_form_status` | POST | WF3 Post-Call | Voice agent (inbound + outbound) after call |
| `/webhook/transfer_unavailable_handler` | POST | (Custom handler) | Voice agent when transfer_to_number fails (no answer / busy / failed) |
| `/webhook/inbound-lead` | POST | WF7 Inbound Call Lead Capture | Inbound voice agent (Alex) `create_lead` tool |

All base URL: `https://{{N8N_BASE_URL}}`

### Supabase Tables
`leads` · `windows` · `estimates` · `quotes` · `invoices` · `appointments` · `escalations` · `post_job_comms` · `activity_log` · `pricing_config`

### n8n Credential IDs
| Name | ID |
|---|---|
| Supabase - ApexWindows | `{{CRED_SUPABASE}}` |
| Gmail SMTP - ApexWindows | `{{CRED_GMAIL_SMTP}}` |
| OpenAI - ApexWindows | `{{CRED_OPENAI}}` |
| Gmail account 2 (booking confirmations) | `{{CRED_GMAIL_BOOKING}}` |

### Key Config Values
| Key | Value |
|---|---|
| n8n instance | `{{N8N_BASE_URL}}` |
| Supabase project | `{{SUPABASE_PROJECT_URL}}` |
| Supabase region | `ap-southeast-2` (Sydney) |
| Stripe product ID | `{{STRIPE_PRODUCT_ID}}` |
| Stripe webhook ID | `{{STRIPE_WEBHOOK_ID}}` |
| PDFShift auth | `Basic {{PDFSHIFT_BASIC_AUTH}}` (hardcoded) |
| SM email | `sales-manager@example.com` (prod) |
| Dev email | `developer@example.com` |
| CRM URL | `apexwindowscrm.vercel.app` |
| Form 2 URL | `windowdetails.vercel.app` |

### Test Data
| Item | Value |
|---|---|
| Test lead UUID | `aaaaaaaa-0000-0000-0000-000000000001` |
| Test email | `testcustomer@apexwindowstx.com` |
| Test estimate | `bbbbbbbb-0000-0000-0000-000000000002` |
| Test quote | `cccccccc-0000-0000-0000-000000000003` |
| Test token | `aabbccddeeff00112233445566778899aabbccddeeff00112233445566778899` |
| Test invoice | `APX-2026-TEST` (amount: $270.63) |

**Cleanup SQL:** `DELETE FROM appointments WHERE client_email = 'testcustomer@apexwindowstx.com'; DELETE FROM invoices WHERE lead_id = 'aaaaaaaa-...'; DELETE FROM escalations WHERE lead_id = 'aaaaaaaa-...'; DELETE FROM quotes WHERE lead_id = 'aaaaaaaa-...'; DELETE FROM estimates WHERE lead_id = 'aaaaaaaa-...'; DELETE FROM activity_log WHERE lead_id = 'aaaaaaaa-...'; DELETE FROM windows WHERE lead_id = 'aaaaaaaa-...'; DELETE FROM leads WHERE id = 'aaaaaaaa-...';`
