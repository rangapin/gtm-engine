# Signal layer — design spec

**Date:** 2026-04-14
**Status:** Approved, ready for implementation plan
**Author:** brainstormed with Claude (Opus 4.6)

## Problem

`/enrich-and-score` today detects campaign-relevant signals (product launches, hiring surges, funding, partnerships, event sponsorships) implicitly, inside its scoring pass. The signals end up embedded in `score_reasoning` prose — e.g., *"3 signals confirmed — sponsors CMAS+AIDA events, launched product Feb 2026, +41% YoY headcount growth (+30)"*. This has four downsides:

1. **Not inspectable.** User can't argue with a specific signal row; they're arguing with 6 words of prose.
2. **Not reusable.** `/draft-sequences` can't cleanly consume signals as opener fuel — it would have to parse prose.
3. **Not auditable.** No source URL, no detected-at date, no confidence tier.
4. **Not eval-able.** `/critique` can't tell whether signal quality correlates with reply rate.

This spec promotes signals from an implicit, prose-embedded concept to a first-class structured artifact: `signals.csv`. A new skill `/gather-signals` detects them; `/enrich-and-score` switches to deterministic lookup against the file; `/draft-sequences` will consume it as opener fuel (spec only, not built here).

## Design constraints (non-negotiable per brief)

- **No new paid tools.** Signals come from free / already-available sources: hiring pages, press releases, G2 reviews, GitHub activity, Product Hunt, Reddit, company blogs, LinkedIn company pages.
- **Fits between `/prospect` and `/enrich-and-score`.** New pipeline position. Input: `prospects.csv`. Output: sibling `signals.csv`.
- **Output is human-readable.** Each signal has a one-line `signal_text` the user can argue with — same discipline as `score_reasoning`.
- **Feeds `/draft-sequences` as opener fuel.** Contract stated here even though integration isn't built in this spec cycle.
- **Follows the 8-step skill contract** from CLAUDE.md, including `assumptions.md`, `logs/gather-signals.log.md`, and prior-critique-signal read.

## Scope boundaries

**In scope (v1):**
- New `/gather-signals` skill — single SKILL.md with internal source table (8 sources).
- New `signals.csv` schema, per-company keying, one row per detected signal.
- `/enrich-and-score` refactor: Step 4 "Signal match (0-30)" replaced by deterministic lookup against `signals.csv`.
- CLAUDE.md updates: pipeline order, `signals.csv` schema, budget-gate scope, Exa usage expansion.
- Backward compat: `/enrich-and-score` without `signals.csv` still produces valid scores (signal match = 0, partial).

**Out of scope (v2 or later):**
- Dispatcher / per-source sub-skill fragmentation. Source table is factored to enable this mechanically later.
- Per-person signals (LinkedIn profile scraping, individual bylines). Company-level only.
- Tiered scraping by `icp_score` or firmographic pre-score. No tiering until prospect lists routinely exceed 20 unique companies.
- Per-signal-type weights in scoring. Generic confidence-tier weights only.
- `/draft-sequences` build — consumption rules specified but not implemented.
- Signal rotation across email 1/2/3. Top signal → email 1 opener only.
- Auto-pull from any paid API.

## Architecture

**Pipeline position:**
```
campaign-init → research-client → icp-define → prospect → gather-signals → enrich-and-score → draft-sequences → activate → capture-results
```

**Files touched:**
1. **Create:** `.claude/skills/gather-signals/SKILL.md`
2. **Modify:** `.claude/skills/enrich-and-score/SKILL.md` (Step 1 + Step 4 rewrite)
3. **Modify:** `CLAUDE.md` (skill count, pipeline, schema, gate tiering, MCP section)

**Gate tier: soft budget gate** (same phrasing pattern as `/enrich-and-score`):
> "About to scrape signals across N unique companies × up to 8 sources. Estimated ~M minutes. No paid APIs. Proceed? (yes / no / subset <list>)"

Free APIs but latency-visible. Silent-tier would be dishonest about wait time.

## Data model — `signals.csv`

One row per detected signal. Multi-row per company. Keyed to `company_domain`.

| Column | Type | Notes |
|---|---|---|
| `company_domain` | string | FK to `prospects.csv` / `prospects.enriched.csv` |
| `signal_type` | enum | Closed vocabulary, see below |
| `signal_text` | string | Human-readable one-liner. User can argue with it. |
| `source_name` | enum | Which source surfaced it (see source table) |
| `source_url` | string | Direct link to evidence |
| `detected_at` | ISO date | When the signal event occurred (not when scraped) |
| `confidence` | enum | `high` / `med` / `low` |
| `notes` | string | Optional. Cross-source corroboration, free context. |

**`signal_type` closed vocabulary (v1):**

| `signal_type` | Definition |
|---|---|
| `product_launch` | New product, release, version bump |
| `funding` | Announced round, YC batch, accelerator |
| `hiring_surge` | Multiple open roles in target function, or LinkedIn headcount jump |
| `leadership_change` | Founder/exec move; new hire in target title |
| `press_mention` | External press, podcast, major media |
| `community_signal` | Reddit/HN thread, G2 review burst, PH ranking |
| `partnership` | Named integration, partnership announcement |
| `event_sponsorship` | Conference/event sponsor or speaker |
| `content_velocity` | Publishing cadence jump (blog, YouTube) |

Expand vocab only when 2+ campaigns surface something that doesn't fit; don't invent types on the fly.

**Sort order:** within `company_domain`, rows ordered by `confidence` desc, then `detected_at` desc. `/draft-sequences` picks row 0 per company for opener fuel.

**Decay rule:** any signal with `detected_at` > 180 days old is forced `confidence=low` regardless of per-source confidence rule.

**Dedup rule:** within a company, signals with same `signal_type` + near-identical `signal_text` within 7 days are merged (highest confidence wins; other `source_url` appended to `notes` as `"also seen on: <url>"`). Cross-source corroboration strengthens a signal, not double-counts it.

## `/gather-signals` — behavior

### Step 1: Load context
Read `prospects.csv`, `icp.json`, `brief.md`. Build the set of unique `company_domain` values. If `prospects.csv` is missing, stop: "Run `/prospect <client>` first."

### Step 2: Prior-critique read
Glob `clients/*/critique.md`, grep for `gather-signals` mentions. Extract top 3 repeated weaknesses. State plainly if empty. Bias behavior (e.g., past critiques flagged "too many low-confidence Reddit hits" → raise Reddit confidence floor or skip).

### Step 3: Pre-flight budget gate
```
About to gather signals for <client>.

Companies (unique): N  (from M prospects)
Sources enabled: 8
Estimated fetches: N × up to 8 = ~Q
Estimated time: ~T minutes

No paid APIs. All public sources.

Proceed? (yes / no / subset <source_name list>)
```

`subset` accepts comma-separated source names. Useful when iterating or when one source is broken.

### Step 4: Gather loop
For each unique `company_domain`, for each enabled source in the source table:
- Attempt fetch using the source's `fetch_method` + `url_template`.
- On success: apply `parse_rule` to extract candidate signals. For each, compute `signal_type`, `signal_text`, `source_url`, `detected_at`, `confidence` per the source's rules.
- On failure (timeout, 404, rate-limit, unparseable): log, move on. Never retry more than once.

### Step 5: Deduplicate
Per the dedup rule above. Preserve cross-source URLs in `notes`.

### Step 6: Write output
Write `clients/<client>/signals.csv`. Sort as specified.

**If `signals.csv` already exists:**
- Show a diff preview ("3 new, 1 dropped, 2 unchanged").
- Ask: overwrite / append-only / abort.
- Never silently overwrite. User may have deleted weak-signal rows to DQ them — preserve their curation.

### Step 7: Assumptions + log + review gate
- Append per-notable-company to `assumptions.md`: `gather-signals: For <company>, I tagged <signal> as <confidence> because <reason>.`
- Append run entry to `logs/gather-signals.log.md`: per-source success/fail counts, total signals, dedup hits, prior-critique flags applied.
- Review gate summary table:
  ```
  <company>   <total> signals (H: <n> M: <n> L: <n>)  [top: <signal_text>]
  ```
  User edits `signals.csv` directly if needed. No re-prompt.

### Step 8: Confirm next
"Next step: `/enrich-and-score <client>` — signals will feed the scoring rubric."

## Source table (lives inside the SKILL.md)

| source_name | url_template | fetch_method | parse_rule | signal_types | confidence_rule |
|---|---|---|---|---|---|
| `hiring_page` | `https://{domain}/careers`, fallback `/jobs` | HTTP GET, follow 1 redirect | Count open roles; if ≥3 in target function (from `icp.json` titles), emit `hiring_surge`. List top 3 titles in `signal_text`. | `hiring_surge`, `leadership_change` | **high** if ≥5 roles OR exec title present; **med** if 3-4; **low** if 1-2 |
| `press_release` | `https://{domain}/press`, `/news`, `/blog/press` | HTTP GET, try each | Grep ISO dates in last 180 days, extract H1/H2 near each | `press_mention`, `funding`, `partnership` | **high** if dated ≤90d; **med** if 91-180d; **low** older |
| `g2_review` | `https://www.g2.com/products/{slug}/reviews` (slug via Exa) | HTTP GET | Count recent reviews + star trend | `community_signal` | **high** if ≥10 in 90d; **med** if 3-9; **low** if 1-2 |
| `github_activity` | `https://api.github.com/orgs/{org}` (org via domain lookup) | API GET (unauth, 60/hr) | Most recent repo push + tagged release in 90d | `product_launch`, `content_velocity` | **high** if release ≤90d; **med** if active commits ≤30d no release; **low** otherwise |
| `product_hunt` | `https://www.producthunt.com/products/{slug}` (slug via Exa) | HTTP GET | Launch date + rank/upvotes | `product_launch`, `community_signal` | **high** if launch ≤90d AND upvotes ≥500; **med** if launch ≤90d; **low** older |
| `reddit_thread` | `https://www.reddit.com/search.json?q={company}` | API GET (JSON, unauth) | Posts in last 90d with ≥10 upvotes; top thread | `community_signal` | **high** if ≥50 upvotes; **med** if 10-49; **low** if 1-9 |
| `company_blog` | `https://{domain}/blog`, `/posts`, `/insights` | HTTP GET | Dates of latest 5 posts; cadence delta vs prior 5 | `content_velocity`, `product_launch`, `partnership` | **high** if cadence ≥2× in 90d; **med** if consistent; **low** if stale (>90d silent) |
| `linkedin_company_page` | `https://www.linkedin.com/company/{slug}` (slug via Exa) | HTTP GET | Employee count + growth chart | `hiring_surge`, `leadership_change` | **high** if ≥20% headcount growth ≤90d; **med** if 5-19%; **low** otherwise. Heavy rate-limit; expect failures. |

**Failure tolerance.** Any source can fail. Skill logs and moves on. A company with 0 successful sources is valid — 0 signals written.

**Exa fallback for slug discovery.** G2, ProductHunt, LinkedIn company page require a slug. Use `web_search_exa` with the company name to find canonical URL; extract slug. If Exa fails, skip that source.

**No personal LinkedIn scraping.** Company pages only (Q2=A). Profile scraping is person-level and has heavier TOS concerns.

## `/enrich-and-score` changes

### Step 1 (add)
Read `signals.csv` if present.

### Step 4 (rewrite "Signal match")

**Before:**
> Each matching signal from the ICP: +10 (cap at 30). Examples: recent funding, hiring in relevant roles, tech stack match.

**After:**
> Load `signals.csv`; filter rows where `company_domain` equals this prospect's company. For each signal row:
> - `high` → +10
> - `med` → +5
> - `low` → +2
>
> Sum, cap at 30. If `signals.csv` is absent, signal match = 0 (partial score, not error).

### `score_reasoning` format change

**Before:**
```
…3 signals confirmed — sponsors CMAS+AIDA events, launched product Feb 2026, +41% YoY headcount growth (+30)
```

**After:**
```
…+30 from signals.csv (molchanovs.com): 3 signals (2 high, 1 med) — product_launch, event_sponsorship, hiring_surge
```

User clicks through to `signals.csv` for `signal_text` + `source_url` details.

### Backward compat invariant

`/enrich-and-score` without `signals.csv` still produces valid scored file. Critical for legacy clients (`theline`, `claybootcamp`) whose `prospects.enriched.csv` pre-dates `/gather-signals`.

### Migration path

To retrofit a legacy client:
```
/gather-signals <client>
/enrich-and-score <client>   # overwrites prospects.enriched.csv with signal-informed scores
```

## `/draft-sequences` consumption contract (spec only)

**Not built in this spec cycle.** Stated so `/gather-signals`'s output shape is load-bearing.

**What `/draft-sequences` will do (future):**
1. Load `signals.csv`; join each prospect on `company_domain`.
2. Pick the top signal per prospect using file sort order (row 0 per company wins).
3. Populate `{{HOOK_SHORT}}` → condensed `signal_text` reference; `{{HOOK_EXPANDED}}` → 1-2 sentences referencing the signal with source context.
4. If `signals.csv` absent OR company has 0 signals → fall back to current behavior (enrichment-data hooks, firmographic framings, ICP-grounded openers). Not an error.

**Properties that fall out for free:**
- Contacts at same company share the same top signal — consistent, not a problem at MVP.
- `/critique` can correlate hook-having-prospects vs hook-less-prospects with reply rate.

**Explicitly v2, not MVP:**
- Signal rotation across email 1/2/3. MVP = top signal in email 1 only.
- Per-prospect signal preference. Only relevant if per-person signals are ever added (Q2=A forecloses for now).

**Non-contract constraint:** `/draft-sequences` must not mutate `signals.csv`. Read-only consumer. Weak signals are a `/critique` concern.

## CLAUDE.md updates

1. **Skill count + pipeline order.** `Eleven skills:` → `Twelve skills:`. Pipeline adds `gather-signals` between `prospect` and `enrich-and-score`.
2. **Gate tiering.** Extend Budget gate entry: now covers `prospect + enrich-and-score + gather-signals`.
3. **File schemas.** New subsection for `signals.csv` (schema + sort + dedup + decay + consumer contracts).
4. **MCP availability.** Note Exa is now also used by `/gather-signals` for slug discovery on G2/PH/LinkedIn; absent → those three sources skipped.

## Success criteria

The feature is correct when:

1. Running `/gather-signals theline` produces a `signals.csv` with ≥1 row for most non-DQ companies, each with a valid `source_url` and a `confidence` tier.
2. Re-running `/gather-signals theline` after the user deletes a row does NOT re-create it silently — the diff-preview + overwrite/append/abort gate fires.
3. Running `/enrich-and-score theline` with `signals.csv` present produces scores that include signal contribution (+0 to +30), and `score_reasoning` cites signal count + tier distribution.
4. Running `/enrich-and-score theline` with `signals.csv` absent still produces valid scored output (signal match = 0). Backward compat holds.
5. Running `/gather-signals` on a client whose `prospects.csv` has 5+ unique companies completes in under 5 minutes (latency budget) with ≥60% source success rate.
6. `/critique` can read `signals.csv` (read-only) and surface quality observations in future critique runs — no mutation.

## Open questions (deferred, not blocking)

- **Per-signal-type weights.** MVP uses generic per-confidence weights. Promote to per-type (via `icp.json.relevant_signals` field) if 2+ critiques flag "irrelevant signal type inflated score."
- **Dispatcher migration (Q1=B).** Single-skill source table is factored to migrate to per-source sub-skills if isolation benefits materialize (e.g., one source breaking frequently). Trigger: 2+ consecutive runs where isolating a failing source matters.
- **Tiered scraping.** When a single campaign exceeds 20 unique companies, add optional `--top N` flag with firmographic pre-score. Not urgent until that volume.
- **Signal confidence model auto-tuning.** Later, confidence rules could be tuned by outcomes (did signals flagged `high` correlate with replies?). Requires mature `results.csv` + `signals.csv` co-existence across many campaigns. 500+ outcomes minimum, per the outcome-capture spec's rationale.
- **Legacy LinkedIn data freshness.** LinkedIn company pages gate heavily; extracted employee-count growth may be stale. Accept for v1. Revisit if LinkedIn becomes the best source for hiring signals.
- **Reddit source blocked in some environments (surfaced in theline smoke test, 2026-04-14):** Claude Code's default WebFetch blocks `reddit.com` by policy. `reddit_thread` returns 100% failure until the environment has an alternative (Reddit MCP, proxy, or policy change). Skill handles gracefully — logs and skips. Not a spec bug. Track as environmental limitation to revisit when tooling catches up.
- **Effective source count for B2C niches (surfaced in theline smoke test):** 8-source design assumes SaaS-style digital footprint. Hardware/consumer brands reliably have ~3 working sources: `company_blog`, `linkedin_company_page` (via Exa), and third-party `press_mention` findings. Skill's graceful-skip handles this, but worth documenting: at MVP volumes, signals.csv per B2C company averages 2-4 rows, not 6-8. Tune expectations.
- **`detected_at` ambiguity for trend signals (surfaced in theline smoke test):** LinkedIn's "+36% YoY headcount" and Reddit's "+53% YoY followers" are aggregate trends with no point-in-time event date. Current spec says `detected_at` = "when the signal event occurred" but trend signals have no such date. Smoke test used scrape date as pragmatic fallback. v2 schema: add `detected_at_type` column with values `event` (exact) / `approximate` / `trend_scraped`. Non-breaking add; `/enrich-and-score` doesn't need this; `/critique` could use it to discount trend-type confidence when correlating with outcomes.
