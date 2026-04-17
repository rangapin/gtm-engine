# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A Claude Code project — no build, no tests, no runtime. The "code" is skill definitions (`SKILL.md` files) that orchestrate MCP tool calls to turn a client domain into a launch-ready outbound campaign. Output is files under `clients/<name>/`.

## Skills

Live at `.claude/skills/<skill-name>/SKILL.md`. Twelve skills:

- `campaign-run` — full-chain playbook; invokes the seven pipeline skills with review gates between each, then `/critique`
- `campaign-status` — read-only inspector; reports which steps each client has completed and the next recommended step
- `critique` — post-run per-step quality rating; output (`clients/<client>/critique.md`) is the eval signal for iterating on the skills
- Pipeline: `campaign-init` → `research-client` → `icp-define` → `prospect` → `gather-signals` → `enrich-and-score` → `draft-sequences` → `activate` → `capture-results`

## The skill contract

Every skill follows the same shape. When editing a skill or writing a new one, preserve it:

1. **Load context** — read the input files from `clients/<client>/` (previous skill's output).
2. **Read prior-run signal** — glob `clients/*/critique.md` (all of them, not just the current client). For this skill's step, extract the 3 most frequently-mentioned weaknesses across critiques. State them to the user at the start ("Prior runs flagged: X, Y, Z — I'll bias against these.") and actually adjust behavior, not just acknowledge. This is how critiques become a compounding asset instead of a journal.
3. **Call tools** — Apollo, Clay, Exa, Attio MCP. Before any credit-consuming Apollo call, warn the user with estimated credit cost and wait for "proceed".
4. **Write output file** — the next skill's input. Filenames are fixed: `brief.md`, `icp.json`, `prospects.csv`, `prospects.enriched.csv`, `sequences/`.
5. **Record assumptions** — append one sentence to `clients/<client>/assumptions.md`: `<step>: I assumed X because Y.` One line per skill-run. This is the 30-second skim-surface for review gates — the user reads assumptions, not full outputs.
6. **Review gate** — summarize for the user (never dump raw tool JSON), accept edits, wait for explicit approval. See "Gate tiering" below — not every step needs a full gate.
7. **Log** — append to `clients/<client>/logs/<step>.log.md`: what ran, which tools, counts, errors, which prior-critique flags were applied.
8. **Confirm** — tell the user the next step.

The file system IS the state. No globals, no hidden context between skills.

## Gate tiering

Not every step deserves a full review gate. The 7-gate sprawl the chain grew into is partly cargo-cult safety — review-everything theater that slows iteration without adding protection. Tier gates by stakes:

- **Silent (no cost gate):** `campaign-init`, `research-client`, `icp-define`. No API spend, no external writes. `campaign-init` is fully automatic. `research-client` and `icp-define` present their output and may ask brief confirmation questions — but there is no credit-spend gate. Output lands in files the user can edit before the budget gate fires.
- **Budget gate, two halves (preview + spend):**
  - **Half 1 — preview (end of `/prospect`):** User sees the actual top-10 rows from `prospects.csv`. No credits spent yet. User can edit `prospects.csv` to trim rows — this *is* the shortlist, no separate skill needed.
  - **Half 2 — spend (start of `/enrich-and-score`):** Gate reads the live row count (post-edit) and confirms the Apollo spend. Accepts `subset <N>` for partial enrichment.
  - `/gather-signals` sits between the two halves and has its own soft gate (free-API but latency-visible). Don't collapse it into either half — different cost shape.
  - Rationale: the original single pre-prospect gate asked the user to approve a credit spend without seeing the data they'd be spending on. Splitting lets real rows inform the spend decision.
- **Quality gate (after `draft-sequences`):** the irreversible-copy-to-real-prospects gate. Full recap view: brief summary, ICP one-pager, prospect count + top-10 rows, 3 sample sequences inline. This is where the user actually reviews.
- **Irreversible gate (before `activate`):** paused-first reminder, final go. No bulk activity before this.

Silent doesn't mean blind — assumptions.md gives the user a 30-sec skim at any point, and if `research-client` or `icp-define` is clearly wrong the user stops the run before the budget gate. "No cost gate" also doesn't mean "no user interaction" — research and ICP definition benefit from a quick confirmation pass before continuing.

## Resuming a partial run

Before starting, check which files exist in `clients/<client>/`. The last file written tells you the last completed step. Offer to resume from the next one rather than restarting.

## MCP availability is conditional

Skills branch on whether an MCP is connected. Do not assume:

- **Apollo** — assumed connected; prospecting and enrichment depend on it.
- **Exa** — assumed connected; used in research and in `/gather-signals` for slug discovery on G2, ProductHunt, and LinkedIn company pages. If Exa is absent, those three sources are skipped (logged; not an error).
- **Clay** — optional; `enrich-and-score` offers it as an add-on.
- **Attio** — may or may not be connected; `activate` falls back to CSV export.
- **HeyReach** — LinkedIn channel for `activate`. Good fit when email coverage is patchy (free-tier enrichment) or when the audience is more responsive on LinkedIn than email. Paused-first discipline applies.
- **Smartlead** — email channel for `activate` via the LeadMagic MCP (`npx smartlead-mcp-by-leadmagic install`, needs `SMARTLEAD_API_KEY`). Paid product — skip if not worth the cost yet.
- **Instantly** — email channel for `activate` via the Instantly v2 API MCP. Current preferred email sender for prod runs. Paused-first discipline applies.
- **Apollo Sequences** — email fallback when Smartlead isn't available, but requires a paid Apollo plan (free tier blocks sequence APIs).

When all email paths are missing, HeyReach alone is a legitimate single-channel activation. When everything is missing, `activate` falls back to CSV export. The skill presents options rather than failing.

## Paused-first discipline

The quality gate (after `draft-sequences`) and the irreversible gate (before `activate`) are the two where mistakes are expensive — bad copy goes to real prospects; records land in real CRM/sequencer. Double-confirm at both.

**Always create sequencer campaigns paused.** HeyReach, Smartlead, Apollo enrollments, and Gmail drafts all start paused/unsent. The user resumes manually in the respective UI after visual review. No exceptions.

## Client isolation

Each client is a folder under `clients/`. Skills read and write only within the folder for the client name they were invoked with. Never cross-reference data between clients.

**One exception:** `/activate` reads and writes `clients/_contacted.csv` — a global ledger that prevents re-contacting the same person across clients or re-runs. This is the only cross-client file and `/activate` is the only skill that touches it. The underscore prefix sorts it above client folders for visibility.

## File schemas (single source of truth)

Every skill reads from and writes to the files below. Column lists live here, not in individual skill files — update here when adding a column and the skills stay in sync.

### `icp.json` (output of `icp-define`)

```json
{
  "client_domain": "acme.com",
  "client_name": "acme",
  "target_industries": ["SaaS", "fintech"],
  "target_company_sizes": ["51-200", "201-500"],
  "target_regions": ["United States", "United Kingdom"],
  "target_titles": ["VP of Engineering", "CTO"],
  "target_signals": ["Recently raised Series A or B", "Hiring backend engineers"],
  "disqualifiers": ["Under 20 employees", "Agencies"],
  "offer_summary": "One sentence about what they sell",
  "value_proposition": "One sentence about why someone buys",
  "messaging_angles": [
    {
      "name": "cost_reduction",
      "hypothesis": "CFO-adjacent buyers under 2026 margin pressure",
      "pain_hooks": [
        "AWS bill grew faster than headcount",
        "finance pushing back on next renewal"
      ],
      "proof_shape": "dollar-denominated outcome from a comparable customer"
    },
    {
      "name": "speed_to_ship",
      "hypothesis": "Eng leaders where shipping velocity is the board-level KPI",
      "pain_hooks": [
        "platform team spending >30% on internal tooling",
        "PR merge-to-prod latency creeping past a day"
      ],
      "proof_shape": "deploy-frequency or lead-time delta"
    },
    {
      "name": "risk_mitigation",
      "hypothesis": "Security/compliance-sensitive buyers (regulated, post-incident, or pre-audit)",
      "pain_hooks": [
        "SOC2 renewal looming",
        "recent public incident in their sector"
      ],
      "proof_shape": "named compliance framework or incident-avoidance outcome"
    }
  ]
}
```

- `messaging_angles`: exactly 3 angles minimum, 5 maximum. Each angle has:
  - `name`: snake_case identifier (used as the `angle:` key in sequence file headers and in logs).
  - `hypothesis`: one sentence describing *which slice of the ICP* this angle lands with. The whole point is that different buyers within the same ICP respond to different framings — don't write angles that would resonate with the entire ICP equally, that defeats the purpose.
  - `pain_hooks`: 2-3 concrete symptoms the prospect might actually be feeling. These are prompts for the drafter, not copy to lift verbatim.
  - `proof_shape`: what kind of proof point this angle expects (not the proof itself — that's per-prospect). Disciplines the drafter away from generic case studies.
- Angles are per-ICP, not per-prospect. `/draft-sequences` picks *which* angle fits each prospect (based on title, signals, score_reasoning) and fills in the specifics.
- Default set when the user has no strong opinion: `cost_reduction`, `speed_to_ship`, `risk_mitigation`. Replace or extend with client-specific angles when the brief makes them obvious (e.g., competitive_displacement, compliance_deadline, hiring_freeze_efficiency, category_creation).

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

### `signals.csv` (output of `gather-signals`)

One row per detected signal. Multi-row per company. Keyed to `company_domain`.

```
company_domain, signal_type, signal_text, source_name, source_url,
detected_at, confidence, notes
```

- `signal_type`: closed vocabulary — `product_launch`, `funding`, `hiring_surge`, `leadership_change`, `press_mention`, `community_signal`, `partnership`, `event_sponsorship`, `content_velocity`. Expand vocab when 2+ campaigns surface something that doesn't fit; don't invent types on the fly.
- `source_name`: one of `hiring_page`, `press_release`, `g2_review`, `github_activity`, `product_hunt`, `reddit_thread`, `company_blog`, `linkedin_company_page`.
- `confidence`: `high` / `med` / `low`. Source-specific rules in the `gather-signals` SKILL.md source table. Any signal with `detected_at` > 180 days old is forced to `low` regardless of per-source rule.
- `detected_at`: ISO date (the signal event's date, not the scrape date — scrape date is in the log).
- `source_url`: direct link to evidence. User-clickable.
- Sort order: within a `company_domain`, rows ordered by `confidence` desc, then `detected_at` desc. Row 0 per company wins for `/draft-sequences` opener fuel.
- Dedup rule: within a company, same `signal_type` + >80% text overlap within 7 days are merged (highest confidence wins; other `source_url` appended to `notes` as `"also seen on: <url>"`).

**Consumers:**
- `/enrich-and-score` reads this file for the "Signal match (0-30)" scoring tier: +10 per high, +5 per med, +2 per low, cap at 30. When absent, signal match = 0 (partial score, not error).
- `/draft-sequences` will read this file as opener fuel (v2 — not yet implemented). Picks row 0 per company to populate `{{HOOK_SHORT}}` and `{{HOOK_EXPANDED}}`.
- `/critique` reads this file alongside `results.csv` and computes per-signal-type reply rates (Finding 3), joining on `company_domain` with top-confidence-per-company attribution. Cross-client aggregates produce `Cross-client weak signal type:` flags that `/gather-signals` reads on its next run to downweight matching signals by one confidence tier.

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

### `results.csv` (output of `capture-results`, user-filled)

One row per non-disqualified prospect from `prospects.enriched.csv`. Pre-filled columns come from the skill; outcome columns are blank and the user fills them as outcomes land.

```
apollo_person_id, person_name, person_email, company_name, company_domain,
icp_score, tier, channel,
sent_at, replied, replied_on_step, bounced, meeting_booked, outcome_notes
```

- `tier`: derived from `icp_score` at skeleton-creation time. `High` (≥70), `Med` (40-69), `Low` (1-39). Snapshot — does not update if the prospect is re-scored later.
- `channel`: `linkedin` / `email` / `gmail` — derived from the `activate.log.md` body via keyword scan (`heyreach`/`linkedin` → linkedin, `smartlead`/`instantly` → email, `gmail` → gmail). If ambiguous, the skill prompts the user.
- `replied`: `true` / `false` / blank. Blank means no outcome recorded yet; `false` means "sent, no reply". Both count as "outcome_filled" for fill-rate calculations.
- `replied_on_step`: integer step number (1..N) if `replied=true`, else blank.
- `apollo_person_id`: may be blank for clients whose `prospects.enriched.csv` predates that column. The skill uses composite key `company_domain + person_name` for merge-on-rerun.
- Join key with `prospects.enriched.csv`: `company_domain + person_name` (robust to `apollo_person_id` absence).

`/critique` is **outcome-first**: it computes `fill_rate` (rows with `replied` non-null / total non-DQ rows) and `campaign_age_days` before asking for ratings. The gate:

- Campaign ≥7 days old AND no `results.csv` → `/critique` **blocks** and sends the user to `/capture-results`.
- Campaign ≥7 days old AND `fill_rate < 50%` → `/critique` **blocks** with the same redirect.
- Campaign <7 days old → proceeds as "preliminary" (ratings flagged outcome-pending).
- Campaign not activated → proceeds as "execution-only" (skill execution only, no outcome signal).
- Fill-rate ≥50% on a >7d campaign → full outcome signal.

When outcome data exists, per-step ratings are **evidence-derived**: `/critique` proposes a cap for each step based on the data (e.g., inverted tier reply rate caps `enrich-and-score` at ≤2; any step with 0 replies across ≥10 sends caps `draft-sequences` at ≤3), and the user confirms or overrides. Overrides upward are marked `*` in the ratings table so they stay visible.

Rationale: outcome-blind ratings pollute the compounding critique signal with vibes. The gate forces the loop to close with real data.

### `clients/_contacted.csv` (global ledger, appended by `/activate`)

Global contacted-prospect ledger. Written by `/activate` after every successful push. Read by the *next* `/activate` run (any client) to dedupe before pre-flight.

```
apollo_person_id, person_email, person_linkedin, company_domain, person_name,
client_name, activated_at, channel, campaign_id
```

- **Match priority** used by `/activate` Step 1.5: `apollo_person_id` → `person_email` → `person_linkedin` → `company_domain|person_name`. First non-empty key is tried against the corresponding ledger column. The composite fallback is for legacy rows whose enrichment predates `apollo_person_id`.
- `channel`: `linkedin` / `email` / `gmail`. Matches the `/capture-results` channel vocab. One row per prospect per channel — a prospect activated on both LinkedIn and email generates two rows.
- `campaign_id`: HeyReach / Smartlead campaign ID or name; blank for Gmail drafts.
- `activated_at`: ISO 8601 timestamp at push time.
- **Append-only.** Never dedupe or edit historical rows — repeated contact attempts to the same person (even from the same client) are signal we want to preserve.
- **Override semantics:** if the user explicitly overrides a dedup warning in Step 1.5 and re-contacts someone already in the ledger, that override IS recorded as a new row. The ledger reflects intent-to-contact, not just first-touch.

### Canonical sequence tokens

Use UPPER_SNAKE_CASE in all sequence files. `/activate` maps these to each platform's variable syntax (see the mapping table in `activate/SKILL.md` Step 5).

Per-prospect (sequencer fills): `{{FIRST_NAME}}`, `{{LAST_NAME}}`, `{{COMPANY}}`, `{{SENDER_NAME}}`
Per-prospect hooks (you fill at drafting): `{{HOOK_SHORT}}`, `{{HOOK_EXPANDED}}`, `{{EMAIL_SUBJECT}}`

## Conventions worth keeping

- Strip TLD + `www.` from domain to get client name (`www.stripe.com` → `stripe`). Confirm with user.
- Score with transparent reasoning, not ML — every score has a one-line `score_reasoning` the user can read and argue with.
- Apollo people *search* does not return emails; emails come from `apollo_people_match` in the enrichment step. Don't call `apollo_people_match` during prospecting.
- Check disqualifiers *before* enrichment to avoid burning credits on dead prospects.
