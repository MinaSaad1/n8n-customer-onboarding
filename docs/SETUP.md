# Setup notes

Specifics that don't fit cleanly in the README's Quickstart. Read this before activating in production.

## The 30-day sequence at a glance

| Day | Touchpoint | Channel | Edit point |
|---|---|---|---|
| 0 | Welcome email with setup steps | Customer email | `Day 0 Welcome Email` node, subject and message |
| 3 | First check-in plus tips | Customer email | `Day 3 Check-in Email` node |
| 7 | Friction survey, "scale of 1-10" | Customer email | `Day 7 Friction Survey` node, plus survey tool integration |
| 14 | Internal alert to operator (no customer email) | Operator inbox or Notion task | `Day 14 Flag for Human Touch` node, edit the recipient |
| 30 | NPS plus referral ask | Customer email | `Day 30 NPS plus Referral` node |

The Wait nodes between them total exactly 30 days: 3 + 4 + 7 + 16 = 30.

## Webhook source examples

### Stripe (most common for SaaS)

In Stripe Dashboard, **Developers** → **Webhooks** → **Add endpoint**:

- **URL**: `https://your-n8n.example.com/webhook/customer-onboarding`
- **Events**: `customer.subscription.created`

Stripe POSTs an envelope where the payload is at `data.object`. The default Set node expects `body.name`, `body.email`, etc., so you'll need to either:

- (Easier) Adapt the Set node to read from `body.data.object.customer_email`, etc.
- (Cleaner) Insert a Code node between the webhook and Set that reshapes the Stripe payload into `{ name, email, company, plan }`.

Stripe webhook test payload:

```json
{
  "id": "evt_test",
  "type": "customer.subscription.created",
  "data": {
    "object": {
      "customer_email": "test@example.com",
      "customer_name": "Test Customer",
      "items": { "data": [{ "price": { "nickname": "Pro" }}]}
    }
  }
}
```

### HubSpot

In a HubSpot workflow, add the action **Send a webhook**. Configure:

- **Method**: POST
- **URL**: `https://your-n8n.example.com/webhook/customer-onboarding`
- **Body**: JSON with mapped contact properties

```json
{
  "name": "{{contact.firstname}} {{contact.lastname}}",
  "email": "{{contact.email}}",
  "company": "{{contact.company}}",
  "plan": "{{deal.deal_stage}}"
}
```

Trigger the workflow on lifecycle stage = "Customer".

### Salesforce

Use a Process Builder or Flow with an Outbound Message action pointed at the webhook URL. Salesforce sends XML by default; you'll likely want a small middleware (a Cloud Function) that converts to JSON, or use a third-party connector.

### Airtable

Airtable Automations → **When a record is created in Customers** → **Run a script** with `fetch()` POSTing to the webhook URL.

### Manual / curl (for testing)

```bash
curl -X POST https://your-n8n.example.com/webhook/customer-onboarding \
  -H 'Content-Type: application/json' \
  -d '{
    "name": "Test Customer",
    "email": "test@yourdomain.com",
    "company": "Acme Co",
    "plan": "Pro"
  }'
```

You should get `{"received": true}` back within a second.

## Webhook signature verification

For Stripe, this is non-optional in production. See [`SECURITY.md`](SECURITY.md) for the full Code-node verification snippet. The short version:

1. Configure the webhook in Stripe and copy the signing secret (`whsec_...`)
2. Set it as an n8n environment variable: `STRIPE_WEBHOOK_SECRET`
3. Add a Code node between **New Customer Webhook** and **Extract Customer Data** that verifies the `stripe-signature` header against an HMAC-SHA256 of the raw body. Throw on mismatch.

For HubSpot and other CRMs, use a shared-secret header check: have the CRM send `X-Webhook-Secret: <token>` and reject in a Code node if it doesn't match `$env.WEBHOOK_SHARED_SECRET`.

## Email template customization

Each `Day N` Gmail node has a `subject` and `message` field. The available expression variables are:

| Expression | Source | Example |
|---|---|---|
| `{{ $json.name }}` | Set node | "Sarah" |
| `{{ $json.email }}` | Set node | "sarah@acme.com" |
| `{{ $json.company }}` | Set node, defaults to '' | "Acme Co" |
| `{{ $json.plan }}` | Set node, defaults to 'Standard' | "Pro" |
| `{{ $json.sender_name }}` | Set node, hardcoded | "Mina" |
| `{{ $now.format('YYYY-MM-DD') }}` | n8n built-in | "2026-05-04" |

Edit the `sender_name` field once in the **Extract Customer Data** Set node and it propagates to all five emails.

### Things to replace

The default templates contain placeholders you must replace:

- `[STEP 1]`, `[STEP 2]`, `[STEP 3]` in Day 0
- `[TIP 1]`, `[TIP 2]`, `[TIP 3]` in Day 3
- `[PRODUCT]` in Day 7 and Day 30
- `YOUR_NAME@YOUR_DOMAIN.com` in Day 14 (the internal alert recipient)
- `YOUR_NAME` in the `sender_name` field of the Set node

### Personalized copy with Claude

For real per-customer personalization (not just `{{ name }}` substitution), insert an HTTP Request node before each Gmail node:

- **Method**: POST
- **URL**: `https://openrouter.ai/api/v1/chat/completions`
- **Auth**: Bearer token (your OpenRouter key, set as a credential)
- **Body** (JSON):

```json
{
  "model": "anthropic/claude-sonnet-4.6",
  "messages": [
    {
      "role": "user",
      "content": "Write a Day 3 check-in email for {{ $json.name }} at {{ $json.company }} on the {{ $json.plan }} plan. Tone: warm, direct, never pushy. Under 100 words. Subject line on first line. One specific question, one clear CTA."
    }
  ]
}
```

Then in the Gmail node, set `subject` and `message` to read from the HTTP node's response. Cost: roughly 1 USD per 1000 customers on Claude Sonnet via OpenRouter. Worth it for high-ACV products, overkill for free-tier signups.

## Notion task creation for Day 14

The default Day 14 step emails an internal address. To replace with a Notion task:

1. Delete the **Day 14 Flag for Human Touch** Gmail node
2. Add a **Notion** node, operation **Create a Page**
3. Configure:
   - **Database**: your team's tasks DB
   - **Title**: `=Day 14 personal touchpoint, {{ $json.name }} at {{ $json.company }}`
   - **Properties**: Status = "To do", Owner = your team-member ID, Due date = `={{ $now }}`, Notes = the body of the alert email

Wire the input from **Wait 7 More Days** and the output to **Wait 16 More Days**.

## Day 7 friction survey, structured response capture

The default template asks "scale of 1-10" in the email body. Replies are unstructured email, which is fine for a starter but bad for branching logic.

For structured capture:

### Option A, Typeform

1. Build a one-question Typeform: "On a scale of 1-10, how likely are you to keep using [PRODUCT]?"
2. Set up a Typeform webhook pointing at a sibling n8n workflow
3. The sibling workflow updates a `customer_state` row with `friction_score` and `friction_reason`
4. Modify the Day 7 Gmail node to send the Typeform link instead of asking inline
5. On Day 14, the workflow can read `friction_score` and decide whether to fire the human-touchpoint alert or skip it

### Option B, Tally

Same pattern, but Tally is free and has a similar webhook interface. Recommended for low-volume / bootstrapped use.

### Option C, Google Forms

Less clean (no native webhook on free tier), but works via a Google Apps Script that POSTs to n8n on submit.

## NPS scoring branch on Day 30

The Day 30 email asks two questions: NPS score and a referral introduction. To branch on the score:

1. Use a structured survey (Typeform / Tally) instead of inline email
2. The survey webhook updates a `customer_state.nps_score` field
3. A sibling workflow reads the score and routes:
   - **Score 9 to 10 (promoter)**: send a referral incentive email, request a case study, add to the testimonial-candidates list
   - **Score 7 to 8 (passive)**: nurture sequence with content
   - **Score 0 to 6 (detractor)**: alert CSM for a save call, add to the at-risk list

Branching belongs in the sibling workflow that handles the survey response, not in the main onboarding flow. Keep this template linear.

## Wait-node reliability, the absolute-time scheduling fallback

The Wait nodes hold execution state in n8n's queue. On crash / restart cycles, in-flight Waits can be lost. For multi-week sequences in production, this is the single biggest reliability risk.

### The pattern

Replace the linear "Webhook → Set → Day 0 → Wait → Day 3 → Wait → ..." flow with two cooperating workflows:

**Workflow 1, signup ingestion**:
- Webhook trigger
- Set node extracts customer fields
- Postgres "Insert" node writes to `customer_onboarding_state`
- Gmail node sends Day 0 immediately
- Postgres update sets `day_0_sent_at = NOW()`

**Workflow 2, daily scheduler (cron at 09:00)**:
- Schedule trigger
- Postgres "Execute Query" finds rows where `day_3_due_at <= NOW() AND day_3_sent_at IS NULL`
- Loop: send Day 3 email, mark `day_3_sent_at = NOW()`
- Repeat the same query / send / mark pattern for Day 7, Day 14, Day 30

### The schema

```sql
CREATE TABLE customer_onboarding_state (
  email text PRIMARY KEY,
  name text NOT NULL,
  company text,
  plan text,
  signup_at timestamptz NOT NULL,
  day_0_sent_at timestamptz,
  day_3_due_at timestamptz GENERATED ALWAYS AS (signup_at + interval '3 days') STORED,
  day_3_sent_at timestamptz,
  day_7_due_at timestamptz GENERATED ALWAYS AS (signup_at + interval '7 days') STORED,
  day_7_sent_at timestamptz,
  day_14_due_at timestamptz GENERATED ALWAYS AS (signup_at + interval '14 days') STORED,
  day_14_sent_at timestamptz,
  day_30_due_at timestamptz GENERATED ALWAYS AS (signup_at + interval '30 days') STORED,
  day_30_sent_at timestamptz,
  unsubscribed boolean NOT NULL DEFAULT false,
  friction_score int,
  nps_score int
);

CREATE INDEX idx_due_unsent ON customer_onboarding_state (day_3_due_at) WHERE day_3_sent_at IS NULL;
```

This survives any number of n8n restarts. The cost is one extra workflow and one table.

### When to switch

- Under ~50 active sequences at once: Wait nodes are fine
- 50 to 500 active sequences, on stable n8n Cloud: still fine, but watch for missed sends
- 500+ sequences or self-hosted with any restart cycle: switch to absolute-time scheduling

## Adding an unsubscribe check

Add a Postgres / Airtable lookup at the top of each Day node that exits if `unsubscribed = true`. Concrete:

1. Before each Gmail node, insert an **IF** node
2. Condition: `{{ $json.unsubscribed }}` is not equal to `true`
3. Wire the "true" output to the Gmail node, leave "false" disconnected (which terminates that branch)

Combined with a one-link unsubscribe footer in every email (`https://yourdomain.com/unsubscribe?email={{ $json.email }}`), you stop sending the moment a customer opts out.

## Testing the full flow

Don't wait 30 real days to verify the sequence works. Two options:

### Option A, shorten the Waits temporarily

Edit each Wait node to seconds instead of days while testing. Run the full flow in 2 minutes. Restore before activating.

### Option B, manual trigger per touchpoint

Disconnect the Wait nodes and use n8n's "Execute node" feature to fire each Gmail node individually with the same input data. Faster than rewiring waits.

After testing, restore the Wait nodes to their production values: 3, 4, 7, 16 days.
