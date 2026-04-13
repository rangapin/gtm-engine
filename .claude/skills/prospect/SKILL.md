---
name: prospect
description: Find target companies and contacts matching the client's ICP. Uses Apollo for prospecting. Outputs a prospect list with companies and people. Use after /icp-define.
---

# /prospect - Find Target Companies and Contacts

When the user runs `/prospect <client-name>`:

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

Start with `per_page: 25` and `page: 1`. You can paginate for more results if the user wants a bigger list.

**Important:** Apollo search consumes credits. Before running, tell the user:

```
I'll search Apollo for companies matching your ICP. This will use Apollo credits.
Searching for: <summarize the filters>
Estimated results: 25 companies (first page)

Proceed?
```

### Step 3: Search for people at those companies

For each company found, use **Apollo people search** (`apollo_mixed_people_api_search`) to find contacts matching the target titles.

Map ICP fields to Apollo filters:
- `target_titles` -> `person_titles`
- Company domains from step 2 -> `q_organization_domains_list`
- Seniority levels -> `person_seniorities` (infer from titles: "VP" = "vp", "Head of" = "director", "CTO" = "c_suite")

**Note:** The people search endpoint does NOT return emails or phone numbers. That happens in the enrichment step.

### Step 4: Compile the prospect list

Write `clients/<client-name>/prospects.csv` with columns:

```csv
company_name,company_domain,industry,employee_count,hq_location,person_name,person_title,person_linkedin,apollo_person_id,apollo_org_id,icp_match_notes
```

Fill `icp_match_notes` with a brief note on why this company/person matches the ICP. For example: "Series B fintech, 120 employees, CTO - matches size + industry + title."

### Step 5: Review gate

Present a summary to the user:

```
Found <N> companies and <M> contacts matching the ICP for <client-name>.

Top companies:
1. <Company A> - <industry>, <size> employees, <location>
   Contacts: <Name> (<Title>), <Name> (<Title>)
2. <Company B> - ...
3. ...

Full list saved to clients/<client-name>/prospects.csv

Review the list. You can:
- Remove companies that don't fit
- Add specific companies by domain
- Adjust filters and re-run
- Approve and move to enrichment

What would you like to do?
```

Wait for the user to review. If they want to remove or add prospects, update the CSV accordingly.

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
- **Status:** User reviewed
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
