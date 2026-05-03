# Workflows

[← back to project README](../../README.md) · [← architecture](../architecture.md)

Seven n8n workflows make up the system. They live as JSON exports in
[`/workflows`](../../workflows/); each has a companion design doc here.

| # | Workflow | Triggers | Nodes | What it does |
|---|---|---|---|---|
| WF1 | [Lead Capture &amp; Validation](wf1-lead-capture.md) | 2 webhooks | 55 | Form 1 + Form 2 entry, dedup, AI estimate, PDF generation |
| WF2 | [Quote Approval &amp; Payment](wf2-quote-and-payment.md) | 9 webhooks | 85 | Quote lifecycle: SM approve → customer accept → invoice → Stripe → closed-won. The deepest piece of the system. |
| WF2B | [Reminder System](wf2b-reminders.md) | hourly cron | 20 | Quote reminders, payment reminders, day-3 auto-decline |
| WF3 Outbound | [Outbound Call](wf3-outbound-call.md) | 1 webhook + daily cron | 11 | ElevenLabs voice agent dispatch, OOH SMS fallback, retry loop |
| WF3 Post-Call | [Post-Call Processes](wf3-post-call.md) | 4 webhooks + 3 schedules | 86 | Transcript analysis, appointment booking + availability, reminder crons |
| WF3 Post-Payment | [Post-Payment Followups](wf3-post-payment.md) | 1 webhook + hourly cron + error trigger | 82 | Review requests, newsletter, escalation engine, error alerts |
| WF7 | [Inbound Call Lead Capture](wf7-inbound-call.md) | 1 webhook | 8 | Voice-to-lead intake from inbound calls; Form 1 equivalent over phone |

**Total:** ~347 nodes across the seven workflows.

## Reading order

If you only have time for one, read **[WF2](wf2-quote-and-payment.md)** —
nine webhook entry points, the magic-link acceptance pattern, the Stripe
integration, and a documented cross-node payload bug that took half a day
to find.

For a quick mental model of the full pipeline, start with
[WF1](wf1-lead-capture.md) → [WF2](wf2-quote-and-payment.md) →
[WF3 Post-Payment](wf3-post-payment.md). The voice agent path
([WF3 Outbound](wf3-outbound-call.md), [WF7](wf7-inbound-call.md),
[WF3 Post-Call](wf3-post-call.md)) is a parallel branch off the main
funnel.

---

[← back to project README](../../README.md)
