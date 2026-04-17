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
- Past flag "too many low-confidence Reddit hits" → raise Reddit's confidence floor from 10 upvotes to 25 for this run; log the adjustment.
- Past flag "LinkedIn scrapes always fail" → skip LinkedIn source unconditionally this run.
- Past flag "g2 reviews too stale to matter" → tighten G2 recency window from 90 to 60 days.

**Cross-client weak signal types.** In addition to the generic weakness extract, scan `clients/*/critique.md` for lines matching the exact prefix `Cross-client weak signal type:`. Extract the `<signal_type>` that follows from each match. Build `downweight_types = [<signal_type>, ...]`.

Apply at collection time: when a signal whose `signal_type` is in `downweight_types` is detected, record it in `signals.csv` as usual (the audit trail stays complete), but drop its `confidence` by one tier: `high → med`, `med → low`, `low → low`. Append to its `notes` column: `downweighted: flagged in <k> prior critiques as weak signal type`.

State at skill start when the list is non-empty: `Prior critiques flagged these signal types as underperforming: <list>. I'll downweight matching signals by one confidence tier this run.` Skip the announcement when the list is empty.

Rationale: downweighting is reversible — a signal type might recover with better framing or against a different ICP. Retirement isn't. The skill collects and discounts; the human decides whether to retire the source rule from the source table.

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
| `press_release` | Try in order: `https://{domain}/press`, `/news`, `/blog/press`, `/company/news` | WebFetch each until one returns 200 | Find ISO or localized dates within last 90 days. Extract the H1/H2/article title nearest each date. Emit one signal per recent press item. **Event date vs. publication date:** after extracting the article's publication date, scan the article body for the actual event date (e.g. "announced in December", "since Q4 2025", "closed in January"). If an event date is found and it differs from the publication date by >21 days, use the event date as `detected_at` and append `"article pub: [pub_date]; event date from body: [event_date]"` to `notes`. The event date, not the article date, governs recency scoring. | `press_mention`, `funding` (if text contains "raised"/"series"/"round"/"$M"), `partnership` (if text contains "partners"/"integration"/"joint") | **high** if ≤30 days; **med** if 31-60 days; **low** if 61-90 days; discard if >90 days (see global recency gate below) |
| `g2_review` | `https://www.g2.com/products/{slug}/reviews` — slug via `mcp__claude_ai_Exa__web_search_exa` query `"{company_name} g2 reviews"` | WebFetch primary; fallback `mcp__claude_ai_Exa__web_fetch_exa` on anti-bot | Parse review dates + star trend. Count reviews in last 90 days. | `community_signal` | **high** if ≥10 recent; **med** if 3-9; **low** if 1-2 |
| `github_activity` | `https://api.github.com/orgs/{org}` — org inferred from domain (e.g., `molchanovs.com` → try `molchanovs`). If fails, try Exa search `"{company_name} github organization"`. | WebFetch with `Accept: application/json` | Check `/repos` most recent push. Check for tag releases in last 90 days via `/orgs/{org}/repos?sort=pushed&direction=desc`. | `product_launch` (if tagged release), `content_velocity` (if active commits but no release) | **high** if release ≤90 days; **med** if active commits ≤30 days with no release; **low** otherwise |
| `product_hunt` | `https://www.producthunt.com/products/{slug}` — slug via Exa search `"{company_name} product hunt"` | WebFetch; fallback Exa | Parse launch date + rank/upvote count. | `product_launch`, `community_signal` | **high** if launch ≤90 days AND upvotes ≥500; **med** if launch ≤90 days; **low** if older |
| `reddit_thread` | `https://www.reddit.com/search.json?q={company_name}&sort=new&t=month` | WebFetch (JSON) | Filter to posts in last 90 days with ≥10 upvotes. Extract top thread title + url. | `community_signal` | **high** if ≥50 upvotes; **med** if 10-49; **low** if 1-9 (still emit for paper trail) |
| `company_blog` | Try in order: `https://{domain}/blog`, `/posts`, `/insights`, `/resources` | WebFetch each until one returns 200 | Parse dates of latest 5 posts. Compare cadence (posts/month) against prior 5 posts. Emit `content_velocity` signal with signal_text describing trend. | `content_velocity`, `product_launch` (if post title matches launch pattern), `partnership` (if title mentions partner/integration) | **high** if cadence jumped ≥2× in recent 90 days; **med** if consistent cadence; **low** if stale (>90 days since last post) |
| `linkedin_company_page` | *(no URL fetched — Exa search returns company card directly)* | `mcp__claude_ai_Exa__web_search_exa` query `"{company_name} linkedin company"` | Parse the structured company card Exa returns: employee count, YoY employee growth %, follower count, YoY follower growth %, founded year, HQ. No need to fetch linkedin.com directly — Exa's indexing includes LinkedIn profile enrichment. Fall back to WebFetch only if Exa returns no card. | `hiring_surge`, `community_signal` (for follower growth >30%) | **high** if ≥20% headcount growth YoY; **med** if 5-19%; **low** if <5% or negative. Emit `community_signal` separately for strong follower growth when headcount is stable. |

**Anti-bot fallback (applies to every row above with `WebFetch` as primary fetch_method):** If a WebFetch call returns HTTP 403, retry the same URL once via `mcp__claude_ai_Exa__web_fetch_exa`. Many e-commerce and SaaS sites 403 direct fetches but are accessible through Exa's fetch proxy. This is a one-level fallback — if Exa also fails, log and move on. Counts as the single permitted retry under the failure-tolerance rule.

**Failure tolerance:** Any source can fail (timeout, 404, rate-limit, CAPTCHA, parse error). Log the failure (source, company, reason), move on. Never retry more than once (the Exa-fallback above is the one permitted retry). A company with 0 successful sources is valid — writes 0 rows for that company.

**Reddit environment constraint:** `reddit_thread` may be non-functional in some Claude Code environments (reddit.com fetches are blocked by policy). The skill attempts the fetch; on failure, logs "Reddit blocked by env" and moves on. Not a skill bug. Alternative fetch paths (Reddit MCP, proxy) may unblock this source in other environments.

**Slug discovery via Exa:** If `mcp__claude_ai_Exa__web_search_exa` is unavailable (Exa MCP not connected), skip `g2_review`, `product_hunt`, `linkedin_company_page` for that run and note in the log: "Exa unavailable; skipped 3 sources."

**Global recency gate:** Applied after extracting, before deduplication. Uses the event date (`detected_at`) not the article/scrape date:
- `detected_at` > 90 days → **discard entirely**, unless `signal_text` contains ongoing language ("through [year]", "multi-year", "continues", "ongoing", "partnership through"). These are forward-looking signals that remain actionable regardless of age.
- `detected_at` > 60 days (but ≤ 90 days) → force `confidence=low` regardless of source rule.
- `detected_at` ≤ 60 days → source's own confidence rule applies.

The previous 180-day window emitted stale signals that made hooks sound like reporting last year's news. Hooks need recency to land.

### Step 5: Deduplicate

Within each `company_domain`, merge duplicate signals:
- Same `signal_type` + near-identical `signal_text` (case-insensitive, >70% text overlap) → keep the row with highest confidence (ties broken by earliest `detected_at`). Append the other source's `source_url` to `notes` as `"also seen on: <url>"`. **No time gap restriction** — a December event re-reported in April is still the same event and should collapse into one row.
- Rationale: GitHub + company blog both surfacing "released v2.0" = one signal with cross-source corroboration. Stronger, not double-counted. The old 7-day window let journalist re-reports of old news look like fresh signals — removing the window closes that gap.

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
