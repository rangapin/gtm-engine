---
name: enrich-and-score
description: Enrich prospect data with emails, phones, and company details. Score each prospect against the ICP. Uses Apollo for enrichment, optionally Clay for additional data. Use after /prospect.
---

# /enrich-and-score - Enrich Prospects and Score Against ICP

When the user runs `/enrich-and-score <client-name>`:

## What to do

### Step 1: Load context

Read:
- `clients/<client-name>/prospects.csv` (the prospect list from /prospect)
- `clients/<client-name>/icp.json` (for scoring)
- `clients/<client-name>/brief.md` (for context on the client)

### Step 2: Enrich contacts

For each person in the prospect list, use **Apollo people enrichment** (`apollo_people_match`) to get:
- Email address
- Phone number
- LinkedIn URL (if not already present)
- Current title (verified)
- Company details

**Important:** This consumes Apollo credits per person. Before running, tell the user:

```
I'll enrich <N> contacts via Apollo. This uses 1 credit per person.
Total estimated credits: <N>

Proceed?
```

**Batch wisely:** Process contacts in batches. If the list is large (50+), suggest starting with the top 20-30 and enriching more later.

For each contact, call `apollo_people_match` with:
- `first_name` + `last_name` (from prospects.csv)
- `domain` (company domain from prospects.csv)
- `organization_name` (company name)

If Apollo returns no match, note it in the output. Don't retry endlessly.

### Step 3: Optional Clay enrichment

If the user has Clay connected and wants deeper enrichment (tech stack, funding details, news), offer:

```
Want me to also run Clay enrichment for additional company data (tech stack, recent news, funding)? This is optional and uses Clay credits.
```

If yes, use Clay's `find-and-enrich-company` for each unique company domain, requesting relevant data points like Tech Stack, Recent News, or Latest Funding.

### Step 4: Score against ICP

For each enriched prospect, calculate an ICP fit score (0-100) based on:

**Firmographic match (0-40 points):**
- Industry match: +15 if exact match, +8 if adjacent
- Company size match: +15 if in target range, +5 if close
- Region match: +10 if in target regions

**Title/seniority match (0-30 points):**
- Exact title match: +30
- Related title (same department, similar seniority): +15
- Wrong department or too junior: +0

**Signal match (0-30 points):**
- Each matching signal from the ICP: +10 (cap at 30)
- Examples: recent funding, hiring in relevant roles, tech stack match

**Disqualifier check:**
- If any disqualifier matches, set score to 0 and flag as disqualified. Don't waste time on these.

Write a brief `score_reasoning` for each row explaining the score.

### Step 5: Write enriched output

Write `clients/<client-name>/prospects.enriched.csv` with columns:

```csv
company_name,company_domain,industry,employee_count,hq_location,person_name,person_title,person_email,person_phone,person_linkedin,icp_score,score_reasoning,disqualified
```

Sort by `icp_score` descending. Disqualified rows go to the bottom with `disqualified=true`.

### Step 6: Review gate

Present a summary:

```
Enriched <N> contacts for <client-name>.

Email found: <N>/<total> (<percent>%)
Phone found: <N>/<total> (<percent>%)

Score distribution:
- High fit (70-100): <N> contacts
- Medium fit (40-69): <N> contacts
- Low fit (1-39): <N> contacts
- Disqualified: <N> contacts

Top prospects:
1. <Name> - <Title> at <Company> - Score: <score>
   Email: <email> | <score_reasoning>
2. ...
3. ...

Full list saved to clients/<client-name>/prospects.enriched.csv

Review the scored list. You can:
- Remove low-scoring prospects
- Adjust scores manually
- Move disqualified back in (with reason)
- Approve and move to sequence drafting

What would you like to do?
```

### Step 7: Log

Write to `clients/<client-name>/logs/enrich-score.log.md`:

```markdown
# Enrichment & Scoring Log
- **Date:** <timestamp>
- **Input:** clients/<client-name>/prospects.csv (<N> contacts)
- **Tool:** Apollo people enrichment
- **Credits used:** ~<N>
- **Email hit rate:** <N>/<total> (<percent>%)
- **Phone hit rate:** <N>/<total> (<percent>%)
- **Score distribution:** High: <N>, Medium: <N>, Low: <N>, DQ: <N>
- **Clay enrichment:** <yes/no>
- **Output:** clients/<client-name>/prospects.enriched.csv
- **Status:** User reviewed
```

## Tools used

- **Apollo.io: apollo_people_match** for contact enrichment
- **Clay: find-and-enrich-company** (optional) for company enrichment
- **Clay: add-company-data-points** (optional) for tech stack, news, funding

## Scoring guidelines

The scoring is intentionally simple. It doesn't need to be a machine learning model. It needs to be:
- Transparent (the user can read the reasoning and agree or disagree)
- Adjustable (the user can say "actually, fintech should score higher")
- Fast (no external API calls needed for scoring itself)

If the user disagrees with how scoring works, adjust the weights and re-score. The weights are a conversation, not a formula cast in stone.

## What NOT to do

- Don't skip the credit warning. Enrichment costs real money.
- Don't enrich people who are obviously disqualified. Check disqualifiers FIRST, then enrich only the non-disqualified prospects.
- Don't present a spreadsheet with 30 columns. Show the user the important fields in a readable summary.
- Don't score without reasoning. Every score needs a one-line explanation.
