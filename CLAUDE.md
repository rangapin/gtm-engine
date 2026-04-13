# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A Claude Code project — no build, no tests, no runtime. The "code" is skill definitions (`SKILL.md` files) that orchestrate MCP tool calls to turn a client domain into a launch-ready outbound campaign. Output is files under `clients/<name>/`.

## Skills

Live at `.claude/skills/<skill-name>/SKILL.md`. Nine skills:

- `campaign-run` — full-chain playbook; invokes the seven pipeline skills with review gates between each
- `campaign-status` — read-only inspector; reports which steps each client has completed and the next recommended step
- Pipeline: `campaign-init` → `research-client` → `icp-define` → `prospect` → `enrich-and-score` → `draft-sequences` → `activate`

## The skill contract

Every skill follows the same shape. When editing a skill or writing a new one, preserve it:

1. **Load context** — read the input files from `clients/<client>/` (previous skill's output).
2. **Call tools** — Apollo, Clay, Exa, Attio MCP. Before any credit-consuming Apollo call, warn the user with estimated credit cost and wait for "proceed".
3. **Write output file** — the next skill's input. Filenames are fixed: `brief.md`, `icp.json`, `prospects.csv`, `prospects.enriched.csv`, `sequences/`.
4. **Review gate** — summarize for the user (never dump raw tool JSON), accept edits, wait for explicit approval.
5. **Log** — append to `clients/<client>/logs/<step>.log.md`: what ran, which tools, counts, errors.
6. **Confirm** — tell the user the next step.

The file system IS the state. No globals, no hidden context between skills.

## Resuming a partial run

Before starting, check which files exist in `clients/<client>/`. The last file written tells you the last completed step. Offer to resume from the next one rather than restarting.

## MCP availability is conditional

Skills branch on whether an MCP is connected. Do not assume:

- **Apollo** — assumed connected; prospecting and enrichment depend on it.
- **Exa** — assumed connected; used in research.
- **Clay** — optional; `enrich-and-score` offers it as an add-on.
- **Attio** — may or may not be connected; `activate` falls back to CSV export.
- **HeyReach** — LinkedIn channel for `activate`. Good fit when email coverage is patchy (free-tier enrichment) or when the audience is more responsive on LinkedIn than email. Paused-first discipline applies.
- **Smartlead** — email channel for `activate` via the LeadMagic MCP (`npx smartlead-mcp-by-leadmagic install`, needs `SMARTLEAD_API_KEY`). Paid product — skip if not worth the cost yet.
- **Apollo Sequences** — email fallback when Smartlead isn't available, but requires a paid Apollo plan (free tier blocks sequence APIs).

When all email paths are missing, HeyReach alone is a legitimate single-channel activation. When everything is missing, `activate` falls back to CSV export. The skill presents options rather than failing.

## Two irreversible gates

`draft-sequences` review and `activate` are the two gates where mistakes are expensive (bad copy goes to real prospects; records land in real CRM/sequencer). Double-confirm at both. **Always create sequencer campaigns paused** — HeyReach, Smartlead, and Apollo enrollments all start paused. The user resumes manually in the respective UI after visual review. No exceptions.

## Client isolation

Each client is a folder under `clients/`. Skills read and write only within the folder for the client name they were invoked with. Never cross-reference data between clients.

## File schemas (single source of truth)

Every skill reads from and writes to the files below. Column lists live here, not in individual skill files — update here when adding a column and the skills stay in sync.

### `prospects.csv` (output of `prospect`)

```
company_name, company_domain, industry, employee_count, hq_location,
person_name, person_title, person_linkedin,
apollo_person_id, apollo_org_id, icp_match_notes
```

- `icp_match_notes`: one-line plain English reason this row passed the ICP filter.
- No emails/phones at this stage — those come from `enrich-and-score`.

### `prospects.enriched.csv` (output of `enrich-and-score`)

All columns from `prospects.csv`, plus:

```
person_email, person_phone,
icp_score, score_reasoning, disqualified
```

- `icp_score`: integer 0-100. Sorted descending in the output file.
- `score_reasoning`: one-line plain English. Must be human-readable — the user needs to argue with it.
- `disqualified`: boolean (`true` / `false`). Disqualified rows sort to the bottom with `icp_score=0`.

### `sequences/` (output of `draft-sequences`)

Two patterns are supported:

**Individual** — one file per prospect with full sequence inline:
```
clients/<client>/sequences/<company-domain>_<person-name>.md
```

**Grouped** — segment templates + per-prospect hook files:
```
clients/<client>/sequences/
  _SEGMENT_<A|B|C>_template.md     # reusable body with {{...}} tokens
  <company-domain>_<person-name>.md  # per-prospect hook values only
```

Underscore prefix on segment templates sorts them to the top of the directory listing.

### Canonical sequence tokens

Use UPPER_SNAKE_CASE in all sequence files. `/activate` maps these to each platform's variable syntax (see the mapping table in `activate/SKILL.md` Step 5).

Per-prospect (sequencer fills): `{{FIRST_NAME}}`, `{{LAST_NAME}}`, `{{COMPANY}}`, `{{SENDER_NAME}}`
Per-prospect hooks (you fill at drafting): `{{HOOK_SHORT}}`, `{{HOOK_EXPANDED}}`, `{{EMAIL_SUBJECT}}`

## Conventions worth keeping

- Strip TLD + `www.` from domain to get client name (`www.stripe.com` → `stripe`). Confirm with user.
- Score with transparent reasoning, not ML — every score has a one-line `score_reasoning` the user can read and argue with.
- Apollo people *search* does not return emails; emails come from `apollo_people_match` in the enrichment step. Don't call `apollo_people_match` during prospecting.
- Check disqualifiers *before* enrichment to avoid burning credits on dead prospects.
