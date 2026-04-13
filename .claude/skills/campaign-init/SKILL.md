---
name: campaign-init
description: Scaffold a new client project folder. Use when starting work for a new client. Takes a domain, creates the folder structure, and prepares for research.
---

# /campaign-init - Start a New Client Campaign

When the user runs `/campaign-init <domain>`:

## What to do

1. **Extract the client name from the domain.** Strip TLD and common prefixes. `acme.com` becomes `acme`. `www.stripe.com` becomes `stripe`. Ask the user to confirm or rename.

2. **Create the client folder structure:**

```
clients/<client-name>/
  brief.md          # will be filled by /research-client
  icp.json          # will be filled by /icp-define
  prospects.csv     # will be filled by /prospect
  sequences/        # will be filled by /draft-sequences
  logs/             # per-step logs
```

3. **Write a starter brief.md:**

```markdown
# Client Brief: <Client Name>

**Domain:** <domain>
**Created:** <date>
**Status:** Research pending

## Company Overview
<!-- Filled by /research-client -->

## What They Sell
<!-- Filled by /research-client -->

## Who They Sell To
<!-- Filled by /research-client -->

## Positioning & Messaging
<!-- Filled by /research-client -->

## Notes
<!-- Your notes here -->
```

4. **Write a starter icp.json:**

```json
{
  "client_domain": "<domain>",
  "client_name": "<client-name>",
  "target_industries": [],
  "target_company_sizes": [],
  "target_regions": [],
  "target_titles": [],
  "target_signals": [],
  "disqualifiers": [],
  "offer_summary": "",
  "value_proposition": ""
}
```

5. **Log the action** to `clients/<client-name>/logs/init.log.md`:

```markdown
# Init Log
- **Date:** <timestamp>
- **Domain:** <domain>
- **Client folder:** clients/<client-name>/
- **Status:** Scaffolded, ready for /research-client
```

6. **Confirm to the user:**

```
Created project for <client-name> at clients/<client-name>/
Next step: run /research-client <client-name> to research the company.
```

## Error handling

- If `clients/<client-name>/` already exists, ask the user: "Client folder already exists. Overwrite, rename, or cancel?"
- If the domain looks malformed (no TLD, obviously not a company), ask for clarification.

## No external tools needed

This skill only creates files. No MCP calls, no API calls.
