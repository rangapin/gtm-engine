# Signal Layer Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add `/gather-signals` skill that scrapes 8 free public sources per company into a structured `signals.csv`, and refactor `/enrich-and-score` to use it for deterministic signal-tier scoring.

**Architecture:** New skill slots between `/prospect` and `/enrich-and-score`. Single SKILL.md with an internal source table (8 rows — hiring_page, press_release, g2_review, github_activity, product_hunt, reddit_thread, company_blog, linkedin_company_page). Per-company keying. Output is `signals.csv` (one row per signal, multi-row per company). Soft budget gate (free APIs but latency-visible). `/enrich-and-score` loses implicit signal detection and gains deterministic lookup against `signals.csv`.

**Tech Stack:** Markdown SKILL.md files (no code, no tests, no runtime). Fetches done at runtime via WebFetch (primary, cheap) or mcp__claude_ai_Exa__web_fetch_exa (fallback for anti-bot sites). Slug discovery uses mcp__claude_ai_Exa__web_search_exa. "Testing" = running skills in Claude Code against `clients/theline/` and inspecting output files.

**Reference spec:** `docs/superpowers/specs/2026-04-14-signal-layer-design.md`

---

## Context for the implementing engineer

Skills in this repo are `SKILL.md` files in `.claude/skills/<name>/`. Claude reads them at invocation time and follows the prompt. "Implementation" = writing/editing Markdown. "Testing" = invoking the skill in Claude Code against existing client data.

Read first:
- `CLAUDE.md` (project root) — skill contract (8 steps), gate tiering, file schemas.
- `docs/superpowers/specs/2026-04-14-signal-layer-design.md` — the design this plan implements.
- `.claude/skills/enrich-and-score/SKILL.md` — existing skill we're refactoring.
- `.claude/skills/capture-results/SKILL.md` — most recent example of a skill that follows the full 8-step contract. Use as pattern.
- `clients/theline/prospects.csv` — actual data shape for smoke test (note: this file may not exist; may be `prospects.seed.csv` or `prospects.enriched.csv`. Check before smoke testing).

Four implementation notes:

1. **Fetch tool selection.** At runtime the skill uses whichever fetch tool is available. Prefer `WebFetch` (native, free) for simple pages. Fall back to `mcp__claude_ai_Exa__web_fetch_exa` when the source is anti-bot (LinkedIn, G2). For slug discovery (G2/PH/LinkedIn need a `{slug}` in their URL template), use `mcp__claude_ai_Exa__web_search_exa` to find the canonical URL for the company. The SKILL.md doesn't force a specific tool — it says "fetch the URL, prefer WebFetch, fall back to Exa on anti-bot failure."

2. **theline has sparse prospects.csv.** The smoke test should use whichever prospect file exists (`prospects.csv` or `prospects.seed.csv`). Companies in theline: Molchanovs, Alchemy Hellas, C4 Carbon, Leaderfins, Elios Sub — several are small/obscure enough that most sources will return nothing. That's fine — zero-signal companies are a valid output.

3. **GitHub API rate limit.** Unauth is 60/hr. At theline's 5 unique companies it's trivial. Mentioned in the skill file as a note; no auth required for MVP.

4. **Exa fallback behavior.** If Exa MCP is not connected at runtime, `/gather-signals` skips the three sources that need Exa (G2, ProductHunt, LinkedIn company page) and logs it. Not an error.

---

## File Structure

- **Create:** `.claude/skills/gather-signals/SKILL.md` — the new skill (Task 1).
- **Modify:** `.claude/skills/enrich-and-score/SKILL.md` — Step 1 load-context extension + Step 4 Signal match rewrite + score_reasoning format (Task 2).
- **Modify:** `CLAUDE.md` — skill count, pipeline order, `signals.csv` schema, gate tiering scope, MCP availability note (Task 3).
- **No changes to:** `.claude/skills/prospect/SKILL.md` (feeds `prospects.csv` to us unchanged), `.claude/skills/draft-sequences/SKILL.md` (consumption is v2 per spec), `.claude/skills/campaign-run/SKILL.md` (may need a one-line mention update but not tracked as a separate task — include in Task 3 if relevant).

---

## Task 1: Create `/gather-signals` skill

**Files:**
- Create: `.claude/skills/gather-signals/SKILL.md`

- [ ] **Step 1: Write the skill file**

Write `.claude/skills/gather-signals/SKILL.md` with the exact content below:

````markdown
---
name: gather-signals
description: Scrape free public sources (hiring pages, press releases, G2, GitHub, Product Hunt, Reddit, company blogs, LinkedIn company pages) to detect campaign-relevant company signals like product launches, hiring surges, funding, partnerships. Writes per-company signals to clients/<client>/signals.csv. Feeds /enrich-and-score (signal tier scoring) and /draft-sequences (opener fuel). Runs between /prospect and /enrich-and-score.
---

# /gather-signals — Detect campaign-relevant company signals

When the user runs `/gather-signals <client-name>`:

This skill promotes "signal detection" from implicit prose-inside-score_reasoning to a first-class `signals.csv` artifact. Signals are detected once, stored structured, and reused by `/enrich-and-score` (for scoring) and `/draft-sequences` (for opener fuel). Per-company keying — all contacts at the same company share the same signal set.

**Gate tier:** soft budget gate. Free APIs but latency-visible (up to ~5 min at theline scale). Same phrasing pattern as `/enrich-and-score`.

## What to do

### Step 1: Load context

Read from `clients/<client-name>/`:
- `prospects.csv` (required — source of companies). If missing, also try `prospects.seed.csv` (some campaigns use a seed file before full prospecting).
- `icp.json` (for target titles — used by hiring_page source rule)
- `brief.md` (for context, especially when picking between same-company-name variants)

If no prospect file is found, stop: "No prospects for `<client>`. Run `/prospect <client>` first."

### Step 2: Read prior-run signal

Glob `clients/*/critique.md` across all clients. Extract the 3 most-repeated weaknesses flagged for `gather-signals` specifically (if any).

On early runs there will be none — state that plainly: "No prior critiques mention gather-signals yet."

When prior flags exist, state them: "Prior runs flagged: X, Y, Z — I'll bias against these." Example adjustments:
- Past flag "channel keyword scan false-positive on pivot mentions" → irrelevant here (different skill).
- Past flag "too many low-confidence Reddit hits" → raise Reddit's confidence floor from 10 upvotes to 25 for this run; log the adjustment.
- Past flag "LinkedIn scrapes always fail" → skip LinkedIn source unconditionally this run.

### Step 3: Pre-flight budget gate

Compute:
- N = count of unique `company_domain` values in prospects file (after DQ filter if column present)
- Estimated fetches = N × 8 (one per source per company)
- Estimated time = N × ~30s avg per source attempt

Print:

```
About to gather signals for <client>.

Companies (unique): <N>  (from <M> prospects)
Sources enabled: 8 (hiring_page, press_release, g2_review, github_activity,
                    product_hunt, reddit_thread, company_blog, linkedin_company_page)
Estimated fetches: <N> × up to 8 = ~<Q>
Estimated time: ~<T> minutes

No paid APIs. All public sources.

Proceed? (yes / no / subset <comma-separated source_names>)
```

Do NOT proceed without explicit "yes" or "subset …". If "subset", only run the named sources.

### Step 4: Gather loop

Use this source table as reference. For each unique `company_domain`, for each enabled source, attempt the fetch:

| source_name | url_template | fetch_method | parse_rule | signal_types_detected | confidence_rule |
|---|---|---|---|---|---|
| `hiring_page` | `https://{domain}/careers`, fallback `https://{domain}/jobs` | WebFetch; follow 1 redirect | Count open roles. If ≥3 in target function (match against `icp.json` titles, case-insensitive), emit a `hiring_surge` signal. List top 3 role titles in `signal_text`. If an exec title (CEO/VP/Director/Head of) is open, also emit `leadership_change`. | `hiring_surge`, `leadership_change` | **high** if ≥5 roles OR exec title; **med** if 3-4 roles; **low** if 1-2 roles |
| `press_release` | Try in order: `https://{domain}/press`, `/news`, `/blog/press`, `/company/news` | WebFetch each until one returns 200 | Find ISO or localized dates within last 180 days. Extract the H1/H2/article title nearest each date. Emit one signal per recent press item. | `press_mention`, `funding` (if text contains "raised"/"series"/"round"/"$M"), `partnership` (if text contains "partners"/"integration"/"joint") | **high** if ≤90 days; **med** if 91-180 days; **low** if older |
| `g2_review` | `https://www.g2.com/products/{slug}/reviews` — slug via `mcp__claude_ai_Exa__web_search_exa` query `"{company_name} g2 reviews"` | WebFetch primary; fallback `mcp__claude_ai_Exa__web_fetch_exa` on anti-bot | Parse review dates + star trend. Count reviews in last 90 days. | `community_signal` | **high** if ≥10 recent; **med** if 3-9; **low** if 1-2 |
| `github_activity` | `https://api.github.com/orgs/{org}` — org inferred from domain (e.g., `molchanovs.com` → try `molchanovs`). If fails, try Exa search `"{company_name} github organization"`. | WebFetch with `Accept: application/json` | Check `/repos` most recent push. Check for tag releases in last 90 days via `/orgs/{org}/repos?sort=pushed&direction=desc`. | `product_launch` (if tagged release), `content_velocity` (if active commits but no release) | **high** if release ≤90 days; **med** if active commits ≤30 days with no release; **low** otherwise |
| `product_hunt` | `https://www.producthunt.com/products/{slug}` — slug via Exa search `"{company_name} product hunt"` | WebFetch; fallback Exa | Parse launch date + rank/upvote count. | `product_launch`, `community_signal` | **high** if launch ≤90 days AND upvotes ≥500; **med** if launch ≤90 days; **low** if older |
| `reddit_thread` | `https://www.reddit.com/search.json?q={company_name}&sort=new&t=month` | WebFetch (JSON) | Filter to posts in last 90 days with ≥10 upvotes. Extract top thread title + url. | `community_signal` | **high** if ≥50 upvotes; **med** if 10-49; **low** if 1-9 (still emit for paper trail) |
| `company_blog` | Try in order: `https://{domain}/blog`, `/posts`, `/insights`, `/resources` | WebFetch each until one returns 200 | Parse dates of latest 5 posts. Compare cadence (posts/month) against prior 5 posts. Emit `content_velocity` signal with signal_text describing trend. | `content_velocity`, `product_launch` (if post title matches launch pattern), `partnership` (if title mentions partner/integration) | **high** if cadence jumped ≥2× in recent 90 days; **med** if consistent cadence; **low** if stale (>90 days since last post) |
| `linkedin_company_page` | `https://www.linkedin.com/company/{slug}` — slug via Exa search `"{company_name} linkedin company"` | WebFetch primary; fallback Exa web_fetch | Parse employee count + growth delta (LinkedIn shows "X employees" and sometimes trend arrows). | `hiring_surge`, `leadership_change` (if new "joined" employee in exec title) | **high** if ≥20% headcount growth in last 90 days; **med** if 5-19%; **low** otherwise. Note: LinkedIn heavily rate-limits; expect frequent failures. |

**Failure tolerance:** Any source can fail (timeout, 404, rate-limit, CAPTCHA, parse error). Log the failure (source, company, reason), move on. Never retry more than once. A company with 0 successful sources is valid — writes 0 rows for that company.

**Slug discovery via Exa:** If `mcp__claude_ai_Exa__web_search_exa` is unavailable (Exa MCP not connected), skip `g2_review`, `product_hunt`, `linkedin_company_page` for that run and note in the log: "Exa unavailable; skipped 3 sources."

**Global decay rule:** After extracting, any candidate signal with `detected_at` > 180 days old is forced to `confidence=low` regardless of the source's confidence rule.

### Step 5: Deduplicate

Within each `company_domain`, merge duplicate signals:
- Same `signal_type` + near-identical `signal_text` (case-insensitive, >80% text overlap) within 7 days of each other → keep the row with highest confidence (ties broken by earliest `detected_at`). Append the other source's `source_url` to `notes` as `"also seen on: <url>"`.
- Rationale: GitHub + company blog both surfacing "released v2.0" = one signal with cross-source corroboration. Stronger, not double-counted.

### Step 6: Write output

Write `clients/<client-name>/signals.csv` with this exact header and column order:

```
company_domain,signal_type,signal_text,source_name,source_url,detected_at,confidence,notes
```

Sort rows by:
1. `company_domain` (alphabetical, for grouping)
2. Within company: `confidence` desc (high > med > low)
3. Within confidence: `detected_at` desc (most recent first)

**If `signals.csv` already exists (re-run):**
- Read existing file.
- Compute diff: which signals are new (not in existing), which are unchanged, which existed but weren't re-detected.
- Show the user:
  ```
  signals.csv diff for <client>:
    New: <N> signals
    Unchanged: <M> signals
    Missing in this run: <K> signals (existed in current file, not re-detected)

  How to proceed?
    (1) overwrite — replace with this run's output (drops the K "missing" rows)
    (2) append-only — keep all existing rows, add only the N new ones
    (3) abort — no changes written
  ```
- Default to append-only. User may have deleted weak-signal rows deliberately to DQ them — preserve their curation.

### Step 7: Assumptions + log + review gate

Append to `clients/<client>/assumptions.md` (create if missing):
- One line per notable decision. Example format:
  - `gather-signals: For molchanovs.com, I tagged product_launch as high because launch date was within 60 days and the company blog had a dedicated announcement post.`
  - `gather-signals: For c4carbon.com, no signals detected from any source; company has minimal public footprint.`

Append to `clients/<client>/logs/gather-signals.log.md`:
- Date
- Mode (first-run / re-run + diff decision)
- Companies scanned, total fetches attempted
- Per-source success/fail/skip counts
- Signals written by confidence: H: <n>, M: <n>, L: <n>
- Dedup merges: <n>
- Prior-critique flags applied (if any)

Review gate — print summary:

```
signals.csv written: clients/<client>/signals.csv
<N> total signals across <M> companies.

Per-company summary:
  molchanovs.com   5 signals (H: 2 M: 2 L: 1)  [top: Launched carbon freediving fins Feb 2026]
  alchemy.gr       2 signals (H: 1 M: 1 L: 0)  [top: New 'Phantom Pro' fin Q1 2026]
  c4carbon.com     0 signals  [low public footprint]
  …

Edit clients/<client>/signals.csv directly to curate (delete weak rows, tighten signal_text). Then:

Next step: /enrich-and-score <client> — signals will feed the scoring rubric.
```

### Step 8: Confirm next step

Print: `"Next step: /enrich-and-score <client> — scoring will now incorporate signals.csv."`

## Output files

- `clients/<client>/signals.csv` — the signal table (created or updated per the re-run flow).
- `clients/<client>/assumptions.md` — one line per notable per-company decision.
- `clients/<client>/logs/gather-signals.log.md` — run log with per-source success/fail counts.

## What NOT to do

- **Don't call paid APIs.** No Apollo, Clay, Hunter, FullEnrich, etc. v1 is strictly free sources.
- **Don't scrape LinkedIn personal profiles.** Company pages only. Profile scraping has heavier TOS concerns and is out of scope (per-person signals are v2).
- **Don't retry failed sources more than once.** If hiring_page returned 404 once, don't loop — log and skip.
- **Don't invent signal types outside the closed vocabulary.** Use the closest existing type and describe nuance in `signal_text` / `notes`.
- **Don't silently overwrite existing signals.csv on re-run.** Always show diff + ask.
- **Don't score signals or rank prospects.** That's `/enrich-and-score`'s job. Just detect and write.
- **Don't attempt personal-level scraping** (LinkedIn posts, individual bylines). Per-company only.

## When to run

- **Right after /prospect** — to populate signals before scoring (the pipeline expects this order).
- **Periodically as a campaign ages** — re-run to catch new signals (user picks append-only in the re-run gate).
- **Retrofitting a legacy client** — run once on existing prospects.csv, then re-run /enrich-and-score to fold in signal-based scoring.

## Gate tier note

Soft budget gate (same as `/enrich-and-score`'s credit warning). Latency-visible, free-API. Not silent, not quality, not irreversible.
````

- [ ] **Step 2: Verify file reads as a coherent prompt**

Re-read the file start to finish. Check:
- Frontmatter `name` and `description` populated correctly.
- All 8 contract steps present (Step 1 load context → Step 8 confirm next).
- Source table has all 8 rows with all 6 columns populated (no blanks, no "TBD").
- Explicit instructions to fall back to Exa when WebFetch fails on anti-bot sites.
- Explicit "slug discovery via Exa" mechanism for G2/PH/LinkedIn.
- Re-run flow (Step 6) has all 3 user options: overwrite / append-only / abort, with append-only as default.
- Dedup rule (Step 5) is concrete: same type + >80% text overlap + within 7 days.
- Global 180-day decay rule stated in Step 4.

- [ ] **Step 3: Commit**

```bash
git add .claude/skills/gather-signals/SKILL.md
git commit -m "feat: add /gather-signals skill — 8-source signals.csv detection"
```

---

## Task 2: Refactor `/enrich-and-score` to consume `signals.csv`

**Files:**
- Modify: `.claude/skills/enrich-and-score/SKILL.md`

- [ ] **Step 1: Update Step 1 (Load context) to also read signals.csv**

In `.claude/skills/enrich-and-score/SKILL.md`, find:

```markdown
### Step 1: Load context

Read:
- `clients/<client-name>/prospects.csv` (the prospect list from /prospect)
- `clients/<client-name>/icp.json` (for scoring)
- `clients/<client-name>/brief.md` (for context on the client)
```

Replace with:

```markdown
### Step 1: Load context

Read:
- `clients/<client-name>/prospects.csv` (the prospect list from /prospect)
- `clients/<client-name>/icp.json` (for scoring)
- `clients/<client-name>/brief.md` (for context on the client)
- `clients/<client-name>/signals.csv` (optional — from /gather-signals; feeds the signal tier of scoring)

If `signals.csv` is absent, scoring still works — signal match just contributes 0 points per prospect. Not an error. The pipeline gracefully degrades for legacy clients and for campaigns where signal gathering was skipped.
```

- [ ] **Step 2: Rewrite Step 4's "Signal match" subsection**

In the same file, find the section under `### Step 4: Score against ICP` that contains:

```markdown
**Signal match (0-30 points):**
- Each matching signal from the ICP: +10 (cap at 30)
- Examples: recent funding, hiring in relevant roles, tech stack match
```

Replace with:

```markdown
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
```

- [ ] **Step 3: Update the score_reasoning format example**

In the same file, find:

```markdown
Write a brief `score_reasoning` for each row explaining the score.
```

Replace with:

```markdown
Write a brief `score_reasoning` for each row explaining the score. For the signal-match portion, cite `signals.csv` rather than narrating signals inline.

**Format example (new):**

```
Industry match (+15); company size 73 in target range (+15); HQ Singapore not in target regions but global brand serving them (+5); Founder title exact (+30); +30 from signals.csv (molchanovs.com): 3 signals (2 high, 1 med) — product_launch, event_sponsorship, hiring_surge
```

**Do NOT inline signal_text** (no more `"sponsors CMAS+AIDA events, launched carbon freediving fins Feb 2026, +41% YoY headcount growth"` in score_reasoning). The full signal details live in `signals.csv`; score_reasoning cites count + tier distribution + types. User clicks through to `signals.csv` for evidence.
```

- [ ] **Step 4: Add a brief note about the contract change**

In the same file, find the section `## Scoring guidelines` near the bottom. Prepend a new paragraph:

```markdown
**Signal detection is delegated to `/gather-signals`.** This skill does not attempt to discover signals — it reads them from `signals.csv`. This keeps scoring deterministic (same input → same score), keeps signal evidence auditable (each signal has a source_url), and enables downstream reuse (draft-sequences pulls the same signals as opener fuel).

For legacy clients whose prospects.enriched.csv was created before /gather-signals existed: run `/gather-signals <client>` once, then re-run `/enrich-and-score <client>` to refresh scores with signal data. Existing prospects.enriched.csv will be overwritten.

```

- [ ] **Step 5: Commit**

```bash
git add .claude/skills/enrich-and-score/SKILL.md
git commit -m "refactor(enrich-and-score): delegate signal detection to /gather-signals

Step 4 Signal match becomes deterministic lookup against signals.csv.
Weights: high=+10, med=+5, low=+2, cap 30. Backward compat preserved
(signals.csv absent → signal match = 0, not error).
score_reasoning cites signal count + tiers, no longer inlines signal prose."
```

---

## Task 3: Update CLAUDE.md

**Files:**
- Modify: `CLAUDE.md`

- [ ] **Step 1: Update skill count**

Find:

```markdown
Live at `.claude/skills/<skill-name>/SKILL.md`. Eleven skills:
```

Replace with:

```markdown
Live at `.claude/skills/<skill-name>/SKILL.md`. Twelve skills:
```

- [ ] **Step 2: Update pipeline order**

Find:

```markdown
- Pipeline: `campaign-init` → `research-client` → `icp-define` → `prospect` → `enrich-and-score` → `draft-sequences` → `activate` → `capture-results`
```

Replace with:

```markdown
- Pipeline: `campaign-init` → `research-client` → `icp-define` → `prospect` → `gather-signals` → `enrich-and-score` → `draft-sequences` → `activate` → `capture-results`
```

- [ ] **Step 3: Extend the Budget gate entry under Gate tiering**

Find:

```markdown
- **Budget gate (before credit-spending):** one gate covers `prospect` + `enrich-and-score`. Phrased as an estimate: "About to spend ~N Apollo credits on M prospects matching this ICP. Proceed?" Don't gate these independently — the decision is joint.
```

Replace with:

```markdown
- **Budget gate (before credit/bandwidth-spending):** one gate covers `prospect` + `gather-signals` + `enrich-and-score`. Phrased as an estimate: "About to spend ~N Apollo credits on M prospects matching this ICP. Proceed?" for Apollo-backed steps, or "About to do N fetches, ~M minutes" for `gather-signals`. Don't gate these independently when they run back-to-back — the decision is joint. `gather-signals` is free-API but latency-visible, so same discipline applies.
```

- [ ] **Step 4: Add `signals.csv` schema section**

Find the section:

```markdown
### `sequences/` (output of `draft-sequences`)
```

Immediately before this, insert a new subsection:

````markdown
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
- `/critique` may cite signal-to-reply correlations once `results.csv` + `signals.csv` co-exist for enough campaigns.

````

- [ ] **Step 5: Extend the MCP availability note for Exa**

Find:

```markdown
- **Exa** — assumed connected; used in research.
```

Replace with:

```markdown
- **Exa** — assumed connected; used in research and in `/gather-signals` for slug discovery on G2, ProductHunt, and LinkedIn company pages. If Exa is absent, those three sources are skipped (logged; not an error).
```

- [ ] **Step 6: Commit**

```bash
git add CLAUDE.md
git commit -m "docs: signal layer — pipeline, signals.csv schema, gate tiering, Exa note"
```

---

## Task 4: Smoke test on `clients/theline/`

**Files:**
- Read-only inputs: `clients/theline/prospects.csv` (or `prospects.seed.csv`), `clients/theline/icp.json`, `clients/theline/brief.md`
- Will be created: `clients/theline/signals.csv`, `clients/theline/logs/gather-signals.log.md`, and appended to `clients/theline/assumptions.md`
- Will be regenerated (if user chooses): `clients/theline/prospects.enriched.csv`

This task is manual — the implementing engineer (or user) runs skills in Claude Code and inspects output.

- [ ] **Step 1: Verify prospect file present**

Before invoking `/gather-signals`, confirm one of these exists:
- `clients/theline/prospects.csv`
- `clients/theline/prospects.seed.csv`

If neither exists, the skill will fail its load-context check and halt correctly. If only `prospects.enriched.csv` exists (theline's current state), copy the minimal schema to `prospects.csv` first or skip smoke test for this client.

Run:

```bash
ls C:/Users/Richard/dev/gtm-engine/clients/theline/prospects*.csv
```

Expected: at least one file listed.

- [ ] **Step 2: Baseline — verify backward compat of /enrich-and-score**

Before `/gather-signals` runs for the first time, confirm `/enrich-and-score theline` still produces a valid scored output when `signals.csv` is absent. If theline's `prospects.enriched.csv` already exists, the quick check is: the score_reasoning column should still parse coherently (even if stale). Inspect 2-3 rows manually.

If you want to verify the new code path, temporarily delete `prospects.enriched.csv`, run `/enrich-and-score theline`, then inspect the output. You'll see signal match = 0 for every prospect, with score_reasoning showing only firmographic + title contributions. That confirms backward compat.

- [ ] **Step 3: First-run /gather-signals**

Run `/gather-signals theline` in Claude Code. Expected behavior:

- Loads prospects file. Identifies unique company domains (expect ~5 for theline: molchanovs.com, alchemy.gr, c4carbon.com, leaderfins.com, eliossub.com).
- Prints the budget gate asking for yes/no/subset. Say yes.
- Attempts up to 40 fetches (5 × 8). Most LinkedIn/G2 fetches will fail (rate-limit) — that's expected.
- Writes `signals.csv`.

Verify after completion:

- `clients/theline/signals.csv` exists with the 8-column header.
- At least one company has at least one signal (Molchanovs almost certainly — active brand with press + blog + product launches).
- Small/obscure companies (Elios Sub, Leaderfins) may have 0 signals — that's valid.
- `clients/theline/assumptions.md` gained one or more `gather-signals:` lines.
- `clients/theline/logs/gather-signals.log.md` was created with per-source success/fail counts.
- Summary table in the review gate matches the file contents.

- [ ] **Step 4: Re-run /gather-signals (re-run flow)**

Run `/gather-signals theline` again. Expected behavior:

- Skill detects existing `signals.csv`.
- Computes diff (likely most signals are unchanged, some new or missing due to fetch variance).
- Prints the overwrite/append-only/abort gate.
- Default to append-only by saying "append-only" or just pressing yes.

Verify:
- No rows you might have curated are removed.
- New signals (if any fetch newly succeeded this run) appended.
- Log file has a second run entry.

- [ ] **Step 5: Manually curate signals.csv**

Open `clients/theline/signals.csv` in your editor. Delete 1-2 rows that look weak (low-quality text, low confidence, stale dates). Save.

Re-run `/gather-signals theline` and choose append-only. Verify:
- Deleted rows do NOT come back.
- This preserves user curation — required behavior.

- [ ] **Step 6: Re-run /enrich-and-score to pick up signals**

Run `/enrich-and-score theline`. Expected behavior:

- Loads `signals.csv`.
- For each prospect, filters signals by `company_domain`.
- Applies new weights: high=+10, med=+5, low=+2, cap at 30.
- `score_reasoning` column now cites `signals.csv` in the new format: `"…+N from signals.csv (<domain>): M signals (H high, K med) — <types>"`.
- Does NOT inline specific signal_text strings.

Verify by inspecting 2-3 rows in the new `prospects.enriched.csv`:

- Alexey Molchanov — if Molchanovs has signals, his score should now include +X from signals.
- Someone at a low-signal company — score should be lower than before (no more inline signal detection compensating).

- [ ] **Step 7: Report findings**

Document any issues surfaced during smoke test. Append a `## Smoke-test notes` section to this plan file (bottom) if anything material came up. Candidates to note:
- Which sources had consistently high/low success rates at theline scale.
- Latency observed (was "~5 minutes" estimate accurate?).
- Any parse errors worth tightening in v2.
- Whether the user curation flow felt natural.

Commit notes if added:

```bash
git add docs/superpowers/plans/2026-04-14-signal-layer.md
git commit -m "plan: signal-layer smoke-test notes from theline"
```

---

## Self-Review

After writing the plan, cross-checked against `docs/superpowers/specs/2026-04-14-signal-layer-design.md`:

**Spec coverage:**
- [x] New `/gather-signals` skill with 8-step contract → Task 1
- [x] Source table with 8 sources (url_template, fetch_method, parse_rule, signal_types, confidence_rule) → Task 1 Step 1
- [x] Soft budget gate with yes/no/subset → Task 1 Step 1 (Step 3 of skill)
- [x] Failure tolerance, Exa fallback, 180-day decay, 7-day dedup → Task 1 Step 1
- [x] Re-run flow (overwrite/append-only/abort) with append-only default → Task 1 Step 1 (Step 6 of skill)
- [x] `/enrich-and-score` Step 1 update → Task 2 Step 1
- [x] `/enrich-and-score` Step 4 Signal match rewrite with high/med/low weights → Task 2 Step 2
- [x] `score_reasoning` format change with explicit "don't inline signal_text" rule → Task 2 Step 3
- [x] Backward compat invariant (signals.csv absent → valid partial score) → Task 2 Step 1 (language) + Task 4 Step 2 (verification)
- [x] Migration path for legacy clients → Task 2 Step 4
- [x] CLAUDE.md: skill count, pipeline, gate tiering, schema, MCP note → Task 3 Steps 1-6

**Placeholder scan:** No TBDs, no "TODO", no "similar to Task N". Every concrete value (url templates, confidence thresholds, weights, dedup rules) is specified exactly. Task 4 Step 7 optional notes are explicitly optional ("if anything material came up"), not a placeholder for missing content.

**Type / name consistency:**
- Column order in `signals.csv` header matches between Task 1 (skill file) and Task 3 (CLAUDE.md): `company_domain, signal_type, signal_text, source_name, source_url, detected_at, confidence, notes`. ✓
- `signal_type` closed vocabulary matches in both files (9 entries). ✓
- `source_name` enum matches in both files (8 entries). ✓
- Confidence weights (10/5/2) match between Task 1 source-table notes, Task 2 Step 2, and Task 3 Step 4. ✓
- 180-day global decay, 7-day dedup — stated identically in both Task 1 and Task 3. ✓

**Deviation / ambiguity check:** None found beyond the spec's own flagged ambiguity (`leadership_change` detection in `hiring_page` means "exec title listed as open," not true change detection — acknowledged in spec's open questions, fine for v1).
