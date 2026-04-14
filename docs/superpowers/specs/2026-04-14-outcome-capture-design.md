# Outcome capture — design spec

**Date:** 2026-04-14
**Status:** Approved, ready for implementation plan
**Author:** brainstormed with Claude (Opus 4.6)

## Problem

`/critique` today rates how well Claude executed each pipeline step. It does not know whether the campaign *worked* — no reply rates, no meeting counts, no template performance. ChatGPT flagged this as the critique loop's biggest weakness ("a journal, not a feedback dataset"). The prior session's triage agreed: outcome capture is the missing primitive; everything else in ChatGPT's critique (scoring evolution, messaging angles, A/B testing) depends on it.

This spec adds the minimal primitive — a `results.csv` per client, manually filled — and teaches `/critique` to read it.

## Scope boundaries

**In scope (v1):**

- New `/capture-results` skill — writes `results.csv` skeleton, never overwrites user edits.
- `/critique` extension — reads `results.csv` when present, computes reply rate by tier and replies by step, writes findings into the critique file.
- `/campaign-status` extension — surfaces results.csv presence/staleness as a flag, adds two stage rows.
- CLAUDE.md updates — new schema, pipeline order, Instantly alongside Smartlead.

**Out of scope (v2 or later):**

- Auto-pull from Instantly / Smartlead / HeyReach MCPs. Manual-fill first.
- Per-step open/click tracking. Gmail drafts has none; `/critique` MVP doesn't use it.
- ICP-score-vs-reply correlation statistics. At 12 prospects/client the number is noise.
- ML scoring, reinforcement loops, auto-template-generation. Premature until 500+ outcomes.

## Architecture

Three files touched:

1. **`.claude/skills/capture-results/SKILL.md`** — new skill.
2. **`.claude/skills/critique/SKILL.md`** — extended with outcome-signal step.
3. **`.claude/skills/campaign-status/SKILL.md`** — extended with stage row, quick stat, flag.
4. **`CLAUDE.md`** — schema + pipeline + MCP updates.

Gate tier: `/capture-results` is **silent** per CLAUDE.md tiering — no credits spent, no external writes, output is local files only. `assumptions.md` gives the 30-second skim surface.

## Data model — `results.csv`

One row per non-disqualified prospect. Columns:

| Column | Source | Notes |
|---|---|---|
| `apollo_person_id` | pre-filled from `prospects.enriched.csv` | may be blank for legacy clients; not the primary join key |
| `person_name` | pre-filled | part of composite join key |
| `person_email` | pre-filled | readability |
| `company_name` | pre-filled | readability |
| `company_domain` | pre-filled | part of composite join key |
| `icp_score` | pre-filled | needed for tier analysis |
| `tier` | pre-filled, derived from `icp_score` | High (70+) / Med (40-69) / Low (1-39) — matches `/campaign-status` bands |
| `channel` | pre-filled from `logs/activate.log.md` | `linkedin` / `email` / `gmail` |
| `sent_at` | blank | user fills (ISO date; Gmail drafts: when they hit send) |
| `replied` | blank | `true` / `false` / blank |
| `replied_on_step` | blank | integer 1-N, or blank |
| `bounced` | blank | `true` / `false` |
| `meeting_booked` | blank | `true` / `false` |
| `outcome_notes` | blank | free text |

**Snapshot semantics:** `tier` is denormalized from `icp_score` at the time of skeleton creation. If scoring is later re-run, `tier` stays on the old value (results.csv is a point-in-time record of what was launched).

**DQ filter:** rows with `disqualified=true` in `prospects.enriched.csv` are excluded from the skeleton.

**Join key (merge-on-rerun):** composite `company_domain + person_name`. Chosen over `apollo_person_id` because legacy `prospects.enriched.csv` files (e.g., `clients/theline/`) predate the `apollo_person_id` column requirement in CLAUDE.md. Composite key works across both legacy and new clients.

## `/capture-results` — behavior

Signature: `/capture-results <client-name>`

### First run (no existing `results.csv`)

1. Load `prospects.enriched.csv`. Filter out `disqualified=true` rows.
2. Read prior critiques (glob `clients/*/critique.md`) — extract top 3 `/capture-results`-relevant weaknesses if any exist. On early runs there will be none; state that plainly.
3. Derive `channel` from the most recent `logs/activate.log.md` (parse the "Channel:" line). If ambiguous or missing, prompt user once: "Channel for this run? linkedin / email / gmail". Remember the answer for the whole skeleton.
4. Write `clients/<client>/results.csv` with pre-filled columns filled in, outcome columns blank.
5. Append to `assumptions.md`: `capture-results: I assumed channel=<X> because <Y>.`
6. Append to `logs/capture-results.log.md`: what ran, prospect count, channel, which prior-critique flags applied (if any).
7. Print: "Skeleton written for N prospects at `<path>`. Fill in outcomes as they land — edit the file directly. Run `/critique <client>` when ready to analyze."

### Subsequent runs (`results.csv` exists)

1. Read it. Compute `N/M` prospects with `replied` non-null, `K` replies, `D` days since last edit.
2. If `prospects.enriched.csv` has rows whose `apollo_person_id` is missing from `results.csv`, offer to append those rows. Never overwrite or reorder existing rows.
3. Print summary. No interactive Q&A. Suggest `/critique` next.
4. Log the run.

### Contract compliance

- **Step 1 (load context):** prospects.enriched.csv, activate log.
- **Step 2 (prior-critique read):** glob + top-3 extraction as above.
- **Step 3 (call tools):** none — fully local.
- **Step 4 (write output):** `results.csv`.
- **Step 5 (assumptions):** one line appended.
- **Step 6 (review gate):** none — silent tier per CLAUDE.md.
- **Step 7 (log):** `logs/capture-results.log.md`.
- **Step 8 (confirm next step):** print `/critique <client>` suggestion.

## `/critique` — extension

Inserts new **Step 2.5** between existing per-step ratings (Step 2) and skill-edit suggestions (Step 3).

### Step 2.5: Outcome signal

Skip if `results.csv` missing OR has <3 rows with `replied` non-null. Skip silently — not an error.

Otherwise compute:

**Finding 1 — reply rate by tier:**

```
Outcome signal for this run:
  High (70+):  2/4   replies (50%)
  Med  (40-69): 1/6   replies (17%)
  Low  (1-39):  0/2   replies (0%)
```

- Annotate "Scoring directionally correct" if reply-rate monotonically decreases by tier.
- Annotate "Scoring inverted — score rubric likely wrong; worth a pass" if Low out-replies High.
- Annotate "Small sample (n=N) — directional only" if total `replied` non-null < 10.

**Finding 2 — replies by step:**

```
Replies by step:
  Step 1: 2
  Step 2: 1
  Step 3: 0
  Step 4: 0
```

- Flag any step with 0 replies across ≥10 sends: "Step 3 got 0 replies across 12 sends — candidate for rewrite or retirement."

### Written critique file

New `## Outcome signal` section inserted after the per-step ratings table. Contains both findings verbatim. Section absent when `results.csv` is absent.

### Step 5 update (cross-client patterns)

Existing pattern-surfacing gets one addition: when ≥2 clients show the same step number with 0 replies across ≥10 sends, flag as cross-client template weakness.

### No breakage

When `results.csv` is absent, `/critique` behaves exactly as today.

## `/campaign-status` — extension

### New stage row

| Files present | Stage |
|---|---|
| + `logs/activate.log.md` (non-placeholder) | activated |
| + `results.csv` | activated + tracking outcomes |

### New quick stats

If `results.csv` exists:
- `outcomes_filled`: rows where `replied` is non-null
- `reply_count`: rows where `replied=true`
- `last_edit`: mtime of `results.csv`

### New flags (inline in single-client report)

- If `outcomes_filled / total_rows < 0.5` AND `days_since_activate >= 7`:
  `results.csv` N% filled (X/Y), D days since activate — outcomes may be getting missed.
- If `last_edit` >14 days ago AND fill < 100%:
  `results.csv` stale (D days since last edit), run `/capture-results` to refresh or mark remaining prospects.

### Next-step mapping update

| Stage | Next skill |
|---|---|
| activated (no `results.csv`) | `/capture-results <client>` |
| activated + tracking outcomes | `/critique <client>` |
| critique done after results | no next step |

"Critique done after results" = `critique.md` exists with `mtime > results.csv mtime` (i.e., critique was run after outcomes landed).

### Contract preserved

Read-only. No writes to `clients/`. No MCP calls.

## CLAUDE.md updates

1. **File schemas section:** add `results.csv` schema (13 columns as above).
2. **Skills list (top-of-file):** update pipeline to `campaign-init → research-client → icp-define → prospect → enrich-and-score → draft-sequences → activate → capture-results`.
3. **MCP availability section:** add Instantly alongside Smartlead ("Smartlead or Instantly — email channel via their respective MCPs"). Keep Smartlead reference — the MCP is still configured.
4. **Gate tiering:** no change. `/capture-results` is silent tier.

## Success criteria

The feature is correct when:

1. Running `/capture-results theline` on an existing client produces a valid `results.csv` skeleton with all pre-filled columns populated from `prospects.enriched.csv` and DQ rows excluded.
2. Re-running `/capture-results theline` after the user manually fills some rows preserves their edits and appends any new prospects.
3. Running `/critique theline` with no `results.csv` produces the same output as today (backward compat).
4. Running `/critique theline` after `results.csv` has ≥3 filled rows produces the new `## Outcome signal` section with both findings.
5. Running `/campaign-status theline` shows the correct stage and any applicable flags.
6. The pipeline order in CLAUDE.md and `campaign-run` reflects the new skill.

## Open questions (deferred, not blocking)

- **Per-channel slicing in `/critique`:** when both LinkedIn and email ran for the same client, should findings be split by `channel`? Deferred — at current volume, combined is fine. Revisit when any single client has ≥20 outcomes per channel.
- **Multi-channel storage:** if a campaign activates the same prospect on both LinkedIn and email, does `results.csv` get one row or two? Current spec = one row (channel column records the primary). Revisit when a real dual-channel campaign runs; likely becomes "one row per prospect × channel" with a composite key.
- **Auto-pull from Instantly MCP:** when this lands, does `/capture-results` gain an optional `--pull` flag, or does it become fully automatic? Deferred to v2 spec.
- **`/campaign-run` integration:** should the orchestrator automatically call `/capture-results` after `/activate`, or keep it manual? Recommendation: manual for now — outcomes need time to arrive, and an immediate call produces a useless empty skeleton. But `/campaign-run` should *print the suggestion* at the end ("when replies start landing, run /capture-results").
