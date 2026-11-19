# AI Contract Review Squad: 4 Agents (n8n)

An n8n workflow where four AI agents review a contract in sequence: one extracts the clauses, one assesses the risk, one checks the clauses against the company policy from the database, and one writes the recommendation and redline wording. Rules in code stop a risky or non-compliant contract from being cleared, and anything that is not cleared goes to legal.

▶ **Demo video:** https://drive.google.com/file/d/14loWRFEuFy3AylyUpwhs4mXJDb38Weqo/view

## How it works

1. Mock contracts arrive (SaaS subscription, logistics agreement, mutual NDA) and company policies are read from the Postgres `contract_policies` table
2. Agent 1, Clause Extractor: copies term, auto-renewal, liability, payment terms and other clauses into JSON, writing "not stated" for anything missing
3. Agent 2, Risk Analyst: rates the legal and commercial risk for the customer (low, medium, high)
4. Agent 3, Compliance Checker: compares the clauses with the policy text and lists only real violations
5. Agent 4, Negotiation Advisor: recommends sign, negotiate or do_not_sign, writes a short summary and suggested replacement wording
6. A Code node applies the rules: an invalid answer becomes `negotiate`, and `sign` is downgraded to `negotiate` if the contract is non-compliant or high risk. Only `sign` is `cleared`, everything else is `needs_legal`
7. Contracts that need legal go to Slack and by email, every review is saved to Postgres

```
Manual Trigger -> Mock Contracts -> Get Policies (Postgres) -> Attach Policies
-> Clause Extractor -> Risk Analyst -> Compliance Checker -> Negotiation Advisor
-> Parse Advice -> Needs Legal?
   yes -> Slack Legal Alert + Send Legal Summary (email) + Save Review
   no  -> Save Review
```

## Agents and models

Each agent uses its own model, from three providers (all have free tiers):

| Agent | Provider | Model |
|---|---|---|
| Clause Extractor Agent | Groq | `llama-3.1-8b-instant` |
| Risk Analyst Agent | Groq | `llama-3.3-70b-versatile` |
| Compliance Checker Agent | OpenRouter | `meta-llama/llama-3.3-70b-instruct:free` |
| Negotiation Advisor Agent | Google Gemini | `models/gemini-2.5-flash` |

## Rules in code

- Invalid AI output defaults to `negotiate`, never to `sign`
- `sign` is downgraded to `negotiate` when compliance is `no` or the risk is `high`
- Only `sign` is cleared, every other recommendation goes to legal

## Tech stack

n8n, AI Agents, Groq, OpenRouter, Google Gemini, Slack, Postgres (Supabase), SMTP

## Setup

1. Copy the JSON from `workflow.md` into a file named `contract-review-squad-cloud.json` and import it into n8n
2. Create credentials: Groq (API key), OpenRouter (API key), Google Gemini (API key), Slack (Bot token and signing secret), Postgres (Supabase Session pooler, SSL: Require), SMTP (for the email node, for example Gmail with an app password)
3. Open each model node and pick your own credential for its provider
4. In the Slack node set channel `legal-review`, then run `/invite @your-bot` in that channel
5. In the Send Email node replace `YOUR_EMAIL@gmail.com` with your address
6. Create the database tables (below)
7. In the Postgres save nodes remove the `id` and `created_at` fields, the database fills them
8. Click **Execute workflow**: the mock inputs run through all branches

Free-tier models have rate limits. If you see "Too Many Requests", wait a few minutes and run again.

## Database

```sql
create table if not exists public.contract_policies (
  id bigint generated always as identity primary key,
  topic text,
  rule text
);

create table if not exists public.contract_reviews (
  id bigint generated always as identity primary key,
  contract_id text,
  name text,
  counterparty text,
  risk_level text,
  compliant text,
  recommendation text,
  status text,
  summary text,
  redlines text,
  created_at timestamptz default now()
);

insert into public.contract_policies (topic, rule) values
  ('Auto-renewal', 'Automatic renewal is allowed only with a cancellation notice of 30 days or less.'),
  ('Liability cap', 'Vendor liability must be at least 12 months of fees paid and must never be unlimited on the customer side.'),
  ('Indemnity', 'Customer indemnity must be limited to third-party claims. Indemnity against all claims is not allowed.'),
  ('Payment terms', 'Payment terms must be 60 days or less.'),
  ('Exclusivity', 'Exclusive supplier arrangements must not last longer than 12 months.'),
  ('Termination', 'The customer must be able to terminate for convenience with 30 to 90 days notice.'),
  ('Governing law', 'Delaware, New York, or England and Wales only.'),
  ('Confidentiality', 'Confidentiality obligations must last at least 2 years.');
```

## Demo data

| Mock input | Expected result |
|---|---|
| CT-01 CloudDesk SaaS: auto-renews with 90 days notice, liability capped at 3 months of fees | Policy violations, needs legal |
| CT-02 FastFreight logistics: 5-year exclusive supplier, indemnity against all claims, 120-day payment, termination only for cause | Several violations, needs legal, likely do_not_sign |
| CT-03 Mutual NDA: 2 years confidentiality, 30 days notice, England and Wales | Compliant, likely sign and cleared |

AI answers can vary between runs. The rules in code do not.

## Notes

- The sample `contract_policies` rows are examples, replace them with your real policy
- Demo contracts are invented. This is not legal advice, a lawyer should make the final call
- The legal summary email goes to `YOUR_EMAIL@gmail.com` in the demo, replace it in the Send Email node
- Slack message text must be in **Expression** mode so `{{ }}` values are filled in
- Never commit API keys. Use n8n credentials only

## Author

Zarin Tasnim Shaily, AI Automation Engineer
[LinkedIn](https://linkedin.com/in/5ha1l72) | [GitHub](https://github.com/ztsprofessionalaiid7-alt)
