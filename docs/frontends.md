# Frontends

[← back to README](../README.md)

Three single-page HTML apps make up the human surface of the system.
No build step, no framework — vanilla HTML, CSS, and a small amount of
JavaScript per page. Hosted as static sites on Vercel; each page talks
directly to the n8n webhook layer (or, in the CRM's case, to Supabase).

| Page | Purpose | Source |
|---|---|---|
| Lead intake (Form 1) | Customer-facing first touch | [`/frontends/lead-intake/index.html`](../frontends/lead-intake/index.html) |
| Window details (Form 2) | Per-window photo + notes upload | [`/frontends/window-details/index.html`](../frontends/window-details/index.html) |
| CRM (Manager portal) | SM dashboard for leads, quotes, invoices | [`/frontends/crm/index.html`](../frontends/crm/index.html) |

The decision to keep these as plain HTML rather than reach for React /
Next is deliberate: each page is small enough that a build pipeline
would add more friction than it removes, and the SM team can read /
edit the source if needed.

---

## 1. Lead intake — Form 1

[`/frontends/lead-intake/index.html`](../frontends/lead-intake/index.html)

The customer's first touch. Eight fields, one submission, posts to
`POST /webhook/apex-lead-v2` (the entry webhook for
[WF1](workflows/wf1-lead-capture.md)).

| Field | Validation |
|---|---|
| First name + last name | required, joined as `name` |
| Email | required (basic `@` check; server validates more strictly) |
| Country code + phone | E.164 assembled client-side from the two inputs |
| Street address + city | joined as `streetAddress + ', ' + city + ', TX'` |
| Number of windows | 1 ≤ N ≤ 10 |
| Issue description | required textarea — what's wrong, in the customer's own words |

The free-text `issue_description` is what feeds the
[OpenAI spam check](workflows/wf1-lead-capture.md#flow-a-lead-intake-apex-lead-v2)
on the WF1 side; nothing else on this form goes through AI.

**Design choices worth calling out:**
- A "📞 Prefer to talk?" CTA is shown above the form with a `tel:`
  link to the company's inbound number — the same number that routes
  to the inbound voice agent in [WF7](workflows/wf7-inbound-call.md).
  Lead never has to read past the fold to find a way to talk.
- Country-code dropdown rather than a single phone field: it's three
  extra DOM nodes but it gives ground-truth E.164 formatting for
  international numbers. The Twilio integration later assumes E.164.
- The state is hardcoded to `TX` because the customer base is a
  single metro. If/when this expands beyond DFW the state would need
  to be a real input — flagged in the project's known-issues list.

**Payload shape posted to n8n:**

```json
{
  "name": "Jane Doe",
  "email": "jane@example.com",
  "phone": "+12145551234",
  "address": "123 Oak St, Plano, TX",
  "num_windows": 4,
  "issue_description": "Two foggy windows in the kitchen…"
}
```

---

## 2. Window details — Form 2

[`/frontends/window-details/index.html`](../frontends/window-details/index.html)

The customer receives a link to this page in an email after submitting
Form 1 (or after the inbound voice agent collects their intake fields).
Posts to `POST /webhook/apex-window-details-v2` as `multipart/form-data`
with up to 10 photo files + a JSON metadata blob.

**Form structure:**
- Email field at the top (matches the lead created in WF1 / WF7).
- Issue type dropdown: `Foggy/Broken`, `Full Window Replacement`, `Other`.
  If `Other`, a description field appears.
- 1–10 window cards, each with:
  - File input (`.jpg`, `.jpeg`, `.png`, ≤5MB)
  - Notes textarea (free text)

A window is included in the submission if either the photo or the
notes are non-empty — empty cards are dropped. The notes field is
critical: it's the primary signal Claude uses to estimate a per-window
price. There is no width / height / glass-type input — see
[WF1's window-details branch](workflows/wf1-lead-capture.md#flow-b-window-details--ai-estimate-apex-window-details-v2)
for why.

**Payload shape:**
- File parts: `photo_w1`, `photo_w2`, … `photo_wN` (binary)
- Field: `email` (string)
- Field: `issue_type` (string)
- Field: `other_description` (string, only if `issue_type === 'Other'`)
- Field: `windows_meta` (JSON string)
  ```json
  [
    { "window_number": 1, "notes": "Cracked top pane" },
    { "window_number": 3, "notes": "Foggy, condensation between panes" }
  ]
  ```

The `windows_meta` array only includes windows that were submitted —
the indices are not necessarily contiguous, which is intentional.

---

## 3. CRM — Manager portal

[`/frontends/crm/index.html`](../frontends/crm/index.html)

A dashboard the sales managers use to review leads, approve quotes, and
mark jobs complete. Single ~62 KB HTML file with embedded CSS and
JavaScript; reads directly from Supabase via the REST API and writes
through the n8n webhook layer.

| Action | How |
|---|---|
| Load leads / quotes / invoices | GET to Supabase REST API with anon key |
| Live refresh | poll every 60 seconds |
| Approve quote | POST to `/webhook/manager-approved` (fires [WF2 sub-flow B](workflows/wf2-quote-and-payment.md#sub-flow-b--manager-approved-manager-approved)) |
| Decline quote | direct PATCH to Supabase `quotes` row (legacy — `/webhook/manager-decline` exists in WF2 but the CRM hasn't been wired over yet) |
| Mark job complete | POST to `/webhook/job-completed` (fires [WF2 sub-flow D](workflows/wf2-quote-and-payment.md#sub-flow-d--job-completed-job-completed)) |

**UI patterns:**
- Dark theme; GitHub-inspired surface palette. Heat-signature row
  colors based on `lead_temp` (`hot` / `warm` / `cold`).
- Slide-out drawer per lead — surfaces window photos, AI insights,
  appointment history, outreach log, invoice details.
- Sortable lead table with a "Lead Intel" column that combines AI
  score + qualification stage + recommended action.
- Live indicator (pulsing dot) when the 60-second refresh poll is
  active.

**Known security debt** (see
[architecture.md §5](architecture.md#5-known-security-debt)):
- The Supabase anon key is shipped to the browser. Row-Level Security
  is configured server-side, but exposing the key is a net negative.
- The CRM has no authentication — anyone with the URL can use it. A
  proper Vercel-env-var or backend-proxy fix is on the prep list.
- Decline button still PATCHes Supabase directly; should fire the
  n8n webhook so the SM email and activity_log entries happen.

These items are tracked openly in the project context document — they
don't block the workflow logic but need to be cleared before a real
production hand-off.

---

[← back to README](../README.md)
