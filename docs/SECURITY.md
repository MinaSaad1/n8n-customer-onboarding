# Security & Hardening

## Threat model

What we assume:
- The n8n instance itself is reasonably hardened (auth on the UI, HTTPS, credentials stored encrypted at rest)
- The Gmail / SMTP credential is held in n8n's credential store, not in the workflow JSON
- The webhook URL is not casually shared

What we don't protect against:
- A compromised n8n instance, once an attacker has admin on n8n they own every workflow and credential
- Insider exfiltration via the customer database (someone with read on the customer table can read every customer)
- Spam / abuse of your signup endpoint upstream of the webhook, that's a different system's job

## Layered defenses (ordered by impact)

### Layer 1, webhook signature verification

**Problem**: The webhook URL is the only thing protecting the workflow from spoofed customer events. Anyone who finds the URL (logs, screenshots, terminated employees) can POST a fake customer and trigger a 30-day sequence to any email address. With Stripe, the spoofed event could imply a paid subscription that doesn't exist.

**Fix for Stripe**: Verify the `stripe-signature` header in a Code node before extracting customer data. The verification compares an HMAC-SHA256 of the raw request body against the signature header using your Stripe webhook signing secret.

```javascript
// In a Code node placed immediately after the webhook trigger
const crypto = require('crypto');
const sig = $input.first().json.headers['stripe-signature'];
const body = $input.first().json.body;
const secret = $env.STRIPE_WEBHOOK_SECRET;

const [tPart, v1Part] = sig.split(',');
const timestamp = tPart.split('=')[1];
const expectedSig = v1Part.split('=')[1];
const signedPayload = `${timestamp}.${JSON.stringify(body)}`;
const computedSig = crypto.createHmac('sha256', secret).update(signedPayload).digest('hex');

if (computedSig !== expectedSig) {
  throw new Error('Invalid Stripe signature');
}
return $input.all();
```

**Fix for HubSpot / generic CRM**: Add a shared-secret header check. Configure your CRM to send `X-Webhook-Secret: <random-32-byte-token>` and reject the request if it doesn't match.

**Caveat**: Don't store the signing secret in the workflow JSON. Use n8n environment variables or a credential.

### Layer 2, email deliverability and reputation

**Problem**: Sending welcome / lifecycle emails from an unverified domain (or worse, a free Gmail account) will land you in spam fast. Once your sender reputation is in the gutter, even legitimate emails stop being delivered. For a customer onboarding flow, that's catastrophic, the customer never sees the welcome and assumes the product doesn't work.

**Fix**: Use a transactional email provider with a verified domain. Resend, SendGrid, Postmark, Mailgun all give you a few minutes of DNS work (SPF, DKIM, DMARC) and dramatically better inbox placement than personal Gmail.

```
SPF:   v=spf1 include:_spf.resend.com ~all
DKIM:  resend._domainkey.yourdomain.com CNAME ...
DMARC: v=DMARC1; p=quarantine; rua=mailto:dmarc@yourdomain.com
```

**Caveat**: If you must use Gmail, send only from a workspace account (not gmail.com), keep volume low (under ~50/day), avoid image-heavy templates, and never send to addresses you don't have explicit permission to email.

### Layer 3, Wait-node reliability for multi-week timers

**Problem**: n8n's Wait node holds the execution in a queue until the timer fires. On self-hosted instances, container restarts, deployments, or crashes can lose in-flight timers. For a 30-day sequence, that means a customer who started Day 0 on Monday might never get Day 14 if the container restarted in week two.

This isn't a security issue in the classic sense, but it's a customer-experience failure that compounds: customers silently drop out of the sequence and the operator has no visibility into which sequences fired and which got eaten.

**Fix for production**: Replace the Wait nodes with absolute-time scheduling. Pattern:

1. The webhook workflow writes one row per customer to a `customer_onboarding_state` table:

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
  unsubscribed boolean NOT NULL DEFAULT false
);
```

2. A separate workflow runs hourly, queries for due-but-not-sent rows, sends the email, and updates the `*_sent_at` column.

```sql
SELECT * FROM customer_onboarding_state
WHERE unsubscribed = false
  AND day_3_due_at <= NOW()
  AND day_3_sent_at IS NULL;
```

This pattern survives any number of restarts. The cost is one extra workflow and one table.

**Caveat**: The two workflows must coordinate: the cron workflow must mark `*_sent_at` atomically with the send, otherwise duplicate sends are possible on cron overlaps.

### Layer 4, PII in customer data

**Problem**: The workflow handles customer name, email, company, and plan. If the executions panel logs are world-readable inside n8n, anyone with n8n access can see every customer who signed up in the last 30 days.

**Fix**: 
- n8n auth: every n8n user with executions-panel access can see this. Limit n8n logins to operators who need it.
- Don't log PII to external systems unless required. If you add an `Add Customer to CRM` node downstream, that's the right place for the data, not a debug log.
- Set n8n's execution data retention to something short (7 to 14 days) for this workflow specifically. Use the `Save execution progress` setting on the workflow to limit per-execution retention.

**Caveat**: If you wire Claude / OpenRouter for personalized copy, the LLM provider sees customer name, company, and plan. Make sure your data-processing agreement covers that or use a model with a no-training-data guarantee.

### Layer 5, NPS and survey response privacy

**Problem**: Free-text replies to the Day 7 friction survey and the Day 30 NPS often contain frustrated customer quotes, complaints about specific staff, or contractually-confidential context. Treating them like generic email is wrong.

**Fix**:
- Route the operator-facing inbox for these replies to a shared inbox with limited access, not the founder's primary inbox if the team is larger
- Don't post survey replies to general Slack channels. The "Sarah from Acme said she's about to churn" leak ends careers.
- If you summarize NPS responses with Claude / OpenRouter for digesting, redact obviously identifying details (customer name, company) before sending.

**Caveat**: Some replies will arrive structured (via Typeform / Tally), some unstructured (direct email reply). The unstructured ones are the higher-risk category, they bypass any tooling-level redaction.

## Priority if implementing only some

If you can only do a few:

1. ✅ **Webhook signature verification**, especially for Stripe. Non-negotiable in production.
2. ✅ **Email deliverability setup**, verified domain plus SPF/DKIM/DMARC. Do this before sending the first welcome.
3. ⬜ **Absolute-time scheduling**, replace Wait nodes once you cross ~50 in-flight sequences.
4. ⬜ **PII access control on n8n**, audit who can see executions and what's retained.
5. ⬜ **Survey reply handling**, formalize once you have more than one operator reading replies.

## What about adding AI personalization?

If you wire a Claude / OpenRouter call (see Configuration in the README), the LLM sees customer name, company, plan, and any context you pass it. That's fine for a self-hosted internal use case with a vendor under a DPA. It's not fine for healthcare, finance, or any vertical with data-residency rules. Audit what you're sending before activating.

## What about embedding the workflow URL in a public form?

The webhook URL itself is a secret if signature verification isn't implemented. Even with verification, treat the URL like an API endpoint: not user-visible, not in client-side JavaScript, not in screenshots.

## Reporting security issues

If you find a vulnerability in this template (not a misuse, an actual flaw), please open a [GitHub security advisory](https://github.com/MinaSaad1/n8n-customer-onboarding/security/advisories/new). Don't open a public issue.
