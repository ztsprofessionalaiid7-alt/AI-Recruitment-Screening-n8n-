# AI Recruitment Screening Squad: 4 Agents (n8n)

An n8n workflow where four AI agents screen a CV: one parses it, two review it in parallel (job fit and red flags), and a coordinator combines the reviews into a shortlist, hold or reject decision with interview questions. Rules in code stop a major red flag or a low match score from being shortlisted.

▶ **Demo video:** https://drive.google.com/file/d/1p1EMZWhT40mYYCXKhkieJI8EU_drSDEx/view

## How it works

1. Mock candidates arrive with a job description (senior automation engineer, remote)
2. Agent 1, Resume Parser: extracts name, years of experience, skills and current role
3. Agent 2, Skills Matcher: scores the job fit from 0 to 100 with strengths and gaps. Agent 3, Red Flag Checker: looks for overlapping dates, gaps and unrealistic claims. These two run in parallel
4. Combine Reviews merges both results, then Agent 4, the Coordinator, proposes shortlist, hold or reject, with a reason, three interview questions and an email text
5. A Code node applies the rules: match score below 50 is `reject`, and `shortlist` is downgraded to `hold` if the red flag is major or the score is below 70. An invalid answer defaults to `hold`
6. A Switch routes by decision: shortlist sends the interview email and a Slack alert, hold sends a Slack alert, reject is only saved. Everything is saved to Postgres

```
Manual Trigger -> Mock Candidates -> Resume Parser
   -> Skills Matcher   (parallel)
   -> Red Flag Checker (parallel)
-> Combine Reviews -> Coordinator -> Parse Decision -> Decision?
   shortlist -> Send Interview Email + Slack HR Alert + Save Screening
   hold      -> Slack HR Alert + Save Screening
   reject    -> Save Screening
```

## Agents and models

Each agent uses its own model, from three providers (all have free tiers):

| Agent | Provider | Model |
|---|---|---|
| Resume Parser Agent | Groq | `openai/gpt-oss-20b` |
| Skills Matcher Agent | Google Gemini | `models/gemini-3.5-flash-lite` |
| Red Flag Checker Agent | OpenRouter | `nvidia/nemotron-3.5-lightning:free` |
| Coordinator Agent | Groq | `openai/gpt-oss-120b` |

## Rules in code

- A match score below 50 is always `reject`
- `shortlist` is downgraded to `hold` when the red flag severity is `major` or the score is below 70
- Invalid AI output defaults to `hold`, never to `shortlist`

## Tech stack

n8n, AI Agents, Groq, OpenRouter, Google Gemini, Slack, Postgres (Supabase), SMTP

## Setup

1. Copy the JSON from `workflow.md` into a file named `recruitment-squad-cloud.json` and import it into n8n
2. Create credentials: Groq (API key), OpenRouter (API key), Google Gemini (API key), Slack (Bot token and signing secret), Postgres (Supabase Session pooler, SSL: Require), SMTP (for the email node, for example Gmail with an app password)
3. Open each model node and pick your own credential for its provider
4. In the Slack node set channel `hr-screening`, then run `/invite @your-bot` in that channel
5. In the Send Email node replace `YOUR_EMAIL@gmail.com` with your address
6. Create the database tables (below)
7. In the Postgres save nodes remove the `id` and `created_at` fields, the database fills them
8. Click **Execute workflow**: the mock inputs run through all branches

Free-tier models have rate limits. If you see "Too Many Requests", wait a few minutes and run again.

## Database

```sql
create table if not exists public.candidate_screenings (
  id bigint generated always as identity primary key,
  candidate_id text,
  name text,
  years_experience numeric,
  match_score numeric,
  red_flag_severity text,
  decision text,
  reason text,
  interview_questions text,
  created_at timestamptz default now()
);
```

## Demo data

| Mock input | Expected result |
|---|---|
| C-01 Nadia Karim: 5 years, 40+ n8n workflows, Python and APIs | Strong match, likely shortlist |
| C-02 Tom Becker: 1 year marketing assistant, interested in learning automation | Low match, likely reject |
| C-03 Rafiq Sheikh: three roles all dated 2019-2022, a 50-person team at a 10-person startup | Major red flag, can never be shortlisted, likely hold |

AI answers can vary between runs. The rules in code do not.

## Notes

- Candidates and CVs are invented. Do not run real candidate data through a demo like this
- AI screening can reflect bias in its training data. A human must review every decision, and a real deployment needs a bias audit and legal review first
- The interview email goes to `YOUR_EMAIL@gmail.com` in the demo, replace it in the Send Email node
- Slack message text must be in **Expression** mode so `{{ }}` values are filled in
- Never commit API keys. Use n8n credentials only

## Author

Zarin Tasnim Shaily, AI Automation Engineer
[LinkedIn](https://linkedin.com/in/5ha1l72) | [GitHub](https://github.com/ztsprofessionalaiid7-alt)
