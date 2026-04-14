---
name: prospect
description: Find target companies and contacts matching the client's ICP. Uses Apollo for prospecting. Outputs a prospect list with companies and people. Use after /icp-define.
---

# /prospect - Find Target Companies and Contacts

When the user runs `/prospect <client-name> [count=N]`:

**`count=N` (optional):** target roughly N contacts. Default is 25. The skill paginates through Apollo until the contact count reaches N (or Apollo exhausts matches — log whichever). Read the arg from the invocation string; if absent, use 25. Accept common variants: `count=50`, `count:50`, `n=50`, or plain trailing `50` if it's clearly a number.

## What to do

### Step 1: Load context

Read:
- `clients/<client-name>/icp.json`
- `clients/<client-name>/brief.md` (for additional context)

### Step 2: Search for companies

Use **Apollo organization search** (`apollo_mixed_companies_search`) to find companies matching the ICP.

Map ICP fields to Apollo filters:
- `target_industries` -> `q_organization_keyword_tags`
- `target_company_sizes` -> `organization_num_employees_ranges` (convert: "51-200" becomes "51,200")
- `target_regions` -> `organization_locations`
- `target_signals` like "recently raised Series B" -> `latest_funding_date_range` (last 12 months) + relevant funding filters

Start with `per_page: 25` and `page: 1`. Paginate until the contact count in Step 3 reaches the `count=N` target (or Apollo runs out of matches). If the user asks for more after seeing the preview, continue paginating.

**Cost note:** Apollo company + people search uses a small number of search-credits per page (not the high-cost `apollo_people_match` credits — those fire in enrichment). Proceed without a gate; the real budget decision comes after prospecting when the user sees the list.

**Target contact count:** read `count=N` from the invocation. Default 25. The goal applies to the contact count (output of Step 3), not the company count (Step 2) — companies-per-contact varies by title density. If Apollo exhausts before hitting the target, stop paginating and log the shortfall (`requested N, surfaced M`).

### Step 3: Search for people at those companies

For each company found, use **Apollo people search** (`apollo_mixed_people_api_search`) to find contacts matching the target titles.

Map ICP fields to Apollo filters:
- `target_titles` -> `person_titles`
- Company domains from step 2 -> `q_organization_domains_list`
- Seniority levels -> `person_seniorities` (infer from titles: "VP" = "vp", "Head of" = "director", "CTO" = "c_suite")

**Note:** The people search endpoint does NOT return emails or phone numbers. That happens in the enrichment step.

### Step 4: Compile the prospect list

Write `clients/<client-name>/prospects.csv`. Schema is defined in `CLAUDE.md` ("File schemas" section) — use those columns exactly.

Fill `icp_match_notes` with a brief note on why this company/person matches the ICP. For example: "Series B fintech, 120 employees, CTO - matches size + industry + title."

### Step 5: Preview + handoff (Gate 1 of 2 for the budget decision)

This is the **preview half** of the budget gate. The user sees real data here; the credit-spend decision is deferred to `/enrich-and-score`'s Step 2 ("Gate 2 of 2").

Present inline — not a summary, not counts, the actual top rows:

```
Prospects found for <client-name>: <N> companies, <M> contacts.
(target: <count=N>; <"hit target" | "Apollo exhausted at M">)
Saved to clients/<client-name>/prospects.csv.

Top 10 rows (sorted by ICP-match strength of icp_match_notes):

  1. <Company A> — <industry>, <size> emp, <location>
     <Person Name>, <Title>  · icp_match_notes: <note>
  2. <Company B> — ...
  ...
  10. ...

Next: /gather-signals then /enrich-and-score.
Enrichment will cost ~<M> Apollo credits (1 per contact).

Before continuing, you can:
- Edit prospects.csv directly (delete rows you don't want to pay to enrich — this IS the shortlist)
- Add specific companies by domain (re-run /prospect with new filters, or hand-add rows)
- Adjust the ICP and re-run /prospect
- Proceed to /gather-signals

What would you like to do?
```

**User-editing is the shortlist.** No separate `/shortlist` skill. The user deletes rows from `prospects.csv` that they don't want enriched. The row count at enrichment time is what they actually pay for. This is cheaper and more ergonomic than a structured shortlist step.

Wait for explicit direction. Common responses:
- "Looks good, continue" → confirm, suggest `/gather-signals <client>`.
- "Drop rows 3, 7, 12" → open prospects.csv, delete those rows, confirm, re-print the top 10.
- "Add <domain>" → call Apollo company search for that domain, append to prospects.csv.
- "Re-run with narrower filter" → ask what to narrow on, re-enter Step 2.

### Step 6: Log

Write to `clients/<client-name>/logs/prospect.log.md`:

```markdown
# Prospecting Log
- **Date:** <timestamp>
- **ICP used:** clients/<client-name>/icp.json
- **Tool:** Apollo organization search + people search
- **Filters applied:** <list the actual Apollo filters used>
- **Companies found:** <N>
- **Contacts found:** <M>
- **Output:** clients/<client-name>/prospects.csv
- **Target count:** <count=N> (or "default 25")
- **Surfaced:** <M> contacts across <K> companies (<"hit target" | "Apollo exhausted at M, short of N">)
- **Status:** Preview shown (Gate 1 of 2) — user <approved | edited | re-ran>
- **Rows removed at preview:** <N> (if user trimmed prospects.csv)
```

## Tools used

- **Apollo.io: apollo_mixed_companies_search** for company search
- **Apollo.io: apollo_mixed_people_api_search** for people search

## Handling edge cases

- **Too few results (<10 companies):** Suggest broadening the ICP. Maybe relax company size, add industries, or expand geography.
- **Too many results (>100 companies):** Suggest narrowing. Add signal filters, tighten geography, or be more specific on industry.
- **No results:** Check that the ICP filters translate correctly to Apollo's parameter format. Common issue: industry tags in Apollo don't always match plain English industry names.
- **User wants a specific company added:** Use `apollo_mixed_companies_search` with `q_organization_domains_list` to look up that specific domain.

## What NOT to do

- Don't enrich contacts yet. That's the next step. Prospecting is about building the list, not filling in every data point.
- Don't call `apollo_people_match` (enrichment endpoint) during prospecting. Save the credits.
- Don't present raw Apollo JSON to the user. Summarize it in a readable format.
