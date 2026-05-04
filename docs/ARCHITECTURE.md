# Architecture

## High-level

```
Webhook trigger (CRM or Stripe POST)
        |
        v
Set node, extracts {name, email, company, plan, sender_name}
        |
        +--> Respond to Webhook (immediate 200 OK)
        |
        v
Gmail send, Day 0 welcome
        |
        v
Wait, 3 days
        |
        v
Gmail send, Day 3 check-in
        |
        v
Wait, 4 days
        |
        v
Gmail send, Day 7 friction survey
        |
        v
Wait, 7 days
        |
        v
Gmail send, Day 14 internal alert (to operator, not customer)
        |
        v
Wait, 16 days
        |
        v
Gmail send, Day 30 NPS plus referral
```

The shape is intentionally linear. A 30-day onboarding sequence is a state machine, not a graph: each day fires after the previous one, with no parallelism that would benefit from a more complex topology. The IF / branching logic lives in optional sibling workflows that the survey responses route into, not inline.

## Components

### Webhook trigger (`New Customer Webhook`)

POST endpoint at `/webhook/customer-onboarding`. Accepts JSON with at minimum `name` and `email`, optionally `company` and `plan`. Response mode is `responseNode`, meaning the trigger doesn't auto-respond, it waits for the explicit Respond to Webhook node to fire downstream. This lets us send the 200 OK immediately while the long-running flow continues in the background.

The webhook URL is the entry point for every supported source: Stripe `customer.subscription.created`, HubSpot lifecycle stage workflows, Airtable automations, custom signup endpoints.

### Extract Customer Data (`Set` node)

Pulls the customer fields out of `$json.body` and exposes them as top-level fields for the rest of the workflow. The shape is:

```
{
  name: string,
  email: string,
  company: string,    // defaults to '' if missing
  plan: string,       // defaults to 'Standard'
  sender_name: string // hardcoded YOUR_NAME, edit once
}
```

Centralizing this means the email nodes use clean expressions like `{{ $json.name }}` rather than `{{ $json.body.name }}`, and any future field addition (e.g. `signup_source`) only edits one node.

### Webhook Response (`Respond to Webhook`)

Fires the moment Extract finishes and returns `{ received: true }` to the caller. This matters because some webhook sources (Stripe in particular) retry aggressively if they don't get a 200 within a few seconds. Returning fast lets the long Wait nodes downstream run for weeks without timing the caller out.

### Day 0 Welcome Email (`Gmail send`)

Plain-text email with three numbered setup steps. Subject and body use `{{ $json.name }}` and `{{ $json.sender_name }}` for personalization. No HTML, no images, no tracking pixels by default. Plain text wins on deliverability and renders fine in every client.

The five Gmail nodes (Day 0 / 3 / 7 / 14 / 30) are structurally identical. Each is a copy of the previous with different copy. Editing them is the main customization step.

### Wait nodes (`Wait 3 / 4 / 7 / 16 Days`)

Cumulative timing: 0, 3, 7, 14, 30 days. Each Wait node holds the execution in n8n's queue until the timer fires.

**Critical caveat**: n8n's Wait nodes work in-memory plus persistent queue, but they do not survive every kind of restart cleanly. On self-hosted instances with crash-restart cycles, in-flight Wait timers can be lost. For production reliability with multi-week sequences, replace the Wait nodes with absolute-time scheduling (write the next-action timestamp to a database, run a daily cron that queries due actions). See `SETUP.md` for the pattern.

For low-volume use on n8n Cloud or a stable self-hosted instance, the Wait nodes are fine.

### Day 7 Friction Survey

The simplest version: a plain-text email asking "scale of 1-10" and inviting a free-text reply. The reply lands in the operator's inbox unstructured.

For structured responses, the right pattern is a separate Typeform / Tally form linked from the email, which has its own webhook into a sibling workflow that updates a customer record. Then Day 14's logic can read that record. See `SETUP.md`.

### Day 14 Flag for Human Touch (`Gmail send`, internal)

The sendTo field points at YOUR_NAME@YOUR_DOMAIN.com, not the customer. The point is to make sure the personal touchpoint actually happens by surfacing a task to the operator, not to send another automated message to the customer. This is the most-edited node in the template.

Common variations:
- Replace with a Notion "Create page" in your team's tasks DB
- Replace with a Slack message to a private "customer touchpoints" channel
- Add a Set node before it that reads the Day 7 survey score and skips the alert if the customer is happy

### Day 30 NPS plus Referral (`Gmail send`)

Asks two questions: NPS score (0-10) and an introduction request. Branchable: in production you'd want to capture the score via a structured survey and route detractors (0-6) into a save-offer flow, passives (7-8) into a content nurture, and promoters (9-10) into a referral / case study flow.

The template sends one email to all three. Splitting is a downstream sibling-workflow job.

## Design decisions worth calling out

### Why a Set node holds the customer fields instead of using `$json.body.*` everywhere

Centralizing field extraction means every email expression is short and readable, and any future field rename is a one-node edit. It also lets us inject defaults (`plan || 'Standard'`) in one place. This is the same pattern used across the catalog.

### Why no IF nodes inline

A 30-day sequence with five touchpoints and zero branching renders as twelve nodes. Adding NPS branching, plan-based suppression, and unsubscribe checks inline doubles that. The honest call is: the inline version is fine for a starter, the branched version belongs in sibling workflows that share a customer record (Airtable, Postgres, Notion DB).

### Why the Day 14 alert is a separate Gmail send, not a noop

Two reasons. One, it forces the operator to confront the moment. Two, having an actual artifact (the email) means the touchpoint is auditable: did I do it or didn't I? A silent "TODO" inside n8n is too easy to skip.

### Why Wait nodes instead of a daily cron

For a single-customer demo and low-volume real usage, Wait is simpler: import, wire credentials, done. The cron-plus-database pattern is more reliable but requires a customer-state table and a separate scheduled workflow. The README and SETUP documents both call this out so users can choose.

### Why plain-text emails

Deliverability. Gmail and Outlook score plain-text email higher than HTML for transactional / lifecycle send. Unless the visual design is load-bearing (it isn't, for an onboarding sequence), plain text wins on inbox placement and accessibility.

## Performance notes

| Step | Latency expectation |
|---|---|
| Webhook trigger fire | <100 ms |
| Extract Customer Data | <50 ms |
| Webhook Response | <50 ms |
| Each Gmail send | 500 ms to 2 sec depending on Gmail API latency |
| Wait nodes | exact, by design |

The flow's wall-clock time is dominated by the Wait nodes (30 days). Compute time per execution is under 5 seconds.

If you onboard more than ~500 customers per day, consider the absolute-time scheduling pattern documented in SETUP. Holding 500 concurrent in-flight Wait timers is not what n8n is optimized for.

## Observability

- **n8n Executions panel** shows every webhook hit and every email send. Failed sends surface as red.
- **Sticky note inside the workflow** carries the live README. Edit it as you customize.
- For long-running sequences, the executions panel default retention (30 days on cloud) doesn't cover a 30-day flow end-to-end. Either bump the retention or log each touchpoint to a sheet / DB.

## See also

- [SECURITY.md](SECURITY.md), threat model and what to lock down before activating
- [SETUP.md](SETUP.md), webhook source examples, email customization, the absolute-time scheduling fallback for production
- [Catalog architecture principles](https://github.com/MinaSaad1/n8n-ai-agents/blob/main/docs/architecture-principles.md), patterns shared across every template in the collection
