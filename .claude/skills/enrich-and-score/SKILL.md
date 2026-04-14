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
- `clients/<client-name>/signals.csv` (optional — from /gather-signals; feeds the signal tier of scoring)

If `signals.csv` is absent, scoring still works — signal match just contributes 0 points per prospect. Not an error. The pipeline gracefully degrades for legacy clients and for campaigns where signal gathering was skipped.

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

**Signal match (0-30 points) — deterministic lookup from signals.csv:**

If `signals.csv` is present, for each prospect:
1. Filter rows where `company_domain` equals this prospect's company domain.
2. For each matching signal row, add points by `confidence`:
   - `high` → +10
   - `med` → +5
   - `low` → +2
3. Sum and cap at 30.

If `signals.csv` is absent or the prospect's company has 0 matching rows, signal match = 0. Not an error — the rest of the score (firmographic, title, disqualifier) stands on its own.

Do NOT attempt implicit signal detection. That job belongs to `/gather-signals`. If you think a signal is missing, the user should re-run `/gather-signals <client>` to refresh the file, not work around it here.

**Disqualifier check:**
- If any disqualifier matches, set score to 0 and flag as disqualified. Don't waste time on these.

Write a brief `score_reasoning` for each row explaining the score. For the signal-match portion, cite `signals.csv` rather than narrating signals inline.

**Format example (new):**

```
Industry match (+15); company size 73 in target range (+15); HQ Singapore not in target regions but global brand serving them (+5); Founder title exact (+30); +30 from signals.csv (molchanovs.com): 3 signals (2 high, 1 med) — product_launch, event_sponsorship, hiring_surge
```

**Do NOT inline signal_text** (no more `"sponsors CMAS+AIDA events, launched carbon freediving fins Feb 2026, +41% YoY headcount growth"` in score_reasoning). The full signal details live in `signals.csv`; score_reasoning cites count + tier distribution + types. User clicks through to `signals.csv` for evidence.

### Step 5: Write enriched output

Write `clients/<client-name>/prospects.enriched.csv`. Schema is defined in `CLAUDE.md` ("File schemas" section) — all columns from `prospects.csv` plus `person_email`, `person_phone`, `icp_score`, `score_reasoning`, `disqualified`.

Sort by `icp_score` descending. Disqualified rows go to the bottom with `disqualified=true` and `icp_score=0`.

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

**Signal detection is delegated to `/gather-signals`.** This skill does not attempt to discover signals — it reads them from `signals.csv`. This keeps scoring deterministic (same input → same score), keeps signal evidence auditable (each signal has a source_url), and enables downstream reuse (draft-sequences pulls the same signals as opener fuel).

For legacy clients whose prospects.enriched.csv was created before /gather-signals existed: run `/gather-signals <client>` once, then re-run `/enrich-and-score <client>` to refresh scores with signal data. Existing prospects.enriched.csv will be overwritten.

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
