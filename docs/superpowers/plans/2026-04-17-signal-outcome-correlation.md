# Signal → Outcome Correlation in /critique — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Close the only open loop in the pipeline. Today `signals.csv` (from `/gather-signals`) and `results.csv` (from `/capture-results`) exist as parallel files with no code path between them — which means the "Signal match (0-30)" scoring tier in `/enrich-and-score` and the signal-collection itself in `/gather-signals` have never been proven to influence reply rates. This plan makes `/critique` compute per-signal-type reply rates so the eval loop finally feeds back into signal weighting.

**Architecture:** All changes live in skill-definition markdown. No new files, no new tools, no schema changes to existing CSVs. `/critique` gains a third outcome finding ("Finding 3 — reply rate by signal type") that joins `signals.csv` and `results.csv` on `company_domain`. A new data-derived cap rule is added for `/gather-signals` in the critique's rating step, and the cross-client pattern scan surfaces weak signal types. `/gather-signals` is updated to read those flags from prior critiques and downweight confidence on signal types with proven zero-reply history — making the loop compounding rather than just informational.

**Tech Stack:** Markdown (skill definitions), CSV joins described in prose, JSON (`icp.json`). No code, no tests, no runtime.

**Why no TDD:** Skills are markdown read by Claude at runtime. The "test" of each task is a consistency read — the edited file and its dependents must agree on schema, terminology, and flow.

**Attribution model — design decision:** A company can have multiple rows in `signals.csv`. When a prospect at that company replies, which signal gets credit? Three options considered:

1. **Top-confidence-per-company (row 0).** Matches the existing schema sort rule ("Row 0 per company wins for /draft-sequences opener fuel"). The signal the opener was most likely built from *is* the one we measure. Simple, consistent, available today. **Chosen for v1.**
2. **Credit all signals equally.** Inflates numerators for signal-rich companies; biases toward "more signals = higher rate" artefact. Rejected.
3. **Explicit attribution at draft time.** `/draft-sequences` would record the specific signal row used in each sequence file header; `/critique` reads that back. Cleanest, but requires upstream change in `draft-sequences` and retrofit of existing sequence files. **Deferred to v2.**

**Out of scope:**
- Decay-curve calibration (article suggestion #1 — needs years of data, not weeks).
- Signal combinations as new scoring entities (article suggestion #2 — needs v3 attribution first).
- Per-signal-type recency × reply-rate analysis (interesting, not MVP).
- Auto-retirement of weak signal types. The skill surfaces patterns; humans decide what to cut from `/gather-signals`' source table.
- Sequencer-level per-step sends (still approximated from `results.csv` non-null rows; Instantly auto-pull v2 will sharpen this separately).
- Touching `results.csv` schema. Join works on existing `company_domain` column.

---

## File Structure

**Modified files (no new files created):**
- `.claude/skills/critique/SKILL.md` — add Finding 3 (signal → reply rate join) to Step 2; add one cap rule + one cross-client pattern in Step 3 / Step 6.
- `.claude/skills/gather-signals/SKILL.md` — add a "prior-run signal" block that extracts the weak-signal-type flag from `clients/*/critique.md` and downweights matching signals at collection time. Pre-existing Step 2 (read critiques) gets a specific extraction rule, not a new step.
- `CLAUDE.md` — append one paragraph to the `signals.csv` **Consumers** block noting `/critique` is now a consumer (for traceability; existing consumers list already covers `/enrich-and-score` and `/draft-sequences`).

Files intentionally untouched: `results.csv` schema, `/capture-results` SKILL.md, `/enrich-and-score` SKILL.md (scoring weights stay where they are until outcome evidence justifies moving them — this plan produces that evidence, doesn't consume it).

---

## Task 1: Add Finding 3 to `/critique` Step 2

**Files:**
- Modify: `.claude/skills/critique/SKILL.md`

**Why first:** The outcome finding is the foundation — everything else in this plan consumes it. Get the computation and output format locked before wiring it into rating caps or cross-client scans.

- [ ] **Step 1: Add "Finding 3 — reply rate by signal type" subsection to Step 2**

Insert after the existing "Finding 2 — replies by step." block (currently ends at the line `Step <K> got 0 replies across <sends> sends — candidate for rewrite or retirement.`). Before the "Print both findings to the user" sentence.

Content:

```markdown
**Finding 3 — reply rate by signal type.**

Only runs if `clients/<client-name>/signals.csv` exists. If absent, skip silently and log `signals.csv absent — signal-correlation finding skipped.` in the critique log.

Build the join:
1. For each company in `signals.csv`, pick the **top-confidence row** (the file is already sorted by `confidence` desc, `detected_at` desc per CLAUDE.md's `signals.csv` schema, so this is row 0 per `company_domain`). Build `company_signal = {company_domain: signal_type}`.
2. For each row in `results.csv` where `replied` is non-null, look up `company_signal[company_domain]`. If absent, tag the prospect as `signal_type = "none"` (control group — companies with no detected signal).
3. Group by `signal_type`. For each group, count `n_exposed` (rows with `replied` non-null) and `n_replied` (rows with `replied = true`). Compute `reply_rate = n_replied / n_exposed`.

Print:

```
Reply rate by signal type (fill-rate: <pct>%):
  funding            <r>/<n>  (<pct>%)
  hiring_surge       <r>/<n>  (<pct>%)
  leadership_change  <r>/<n>  (<pct>%)
  product_launch     <r>/<n>  (<pct>%)
  ...
  (none / no signal) <r>/<n>  (<pct>%)   ← control group
```

Rules:
- Sort rows by `reply_rate` desc.
- If any `signal_type` has `n_exposed < 5`, append `(small sample)` to that row instead of a percentage.
- Always include the `(none / no signal)` control row if present. This is the baseline any signal type must beat to be worth collecting.

After the table, add exactly one annotation:

- If every signal type beats the control: `All signal types out-replied no-signal baseline — signal collection is paying off.`
- If at least one signal type lags the control by ≥5pp (with `n_exposed ≥ 10`): `<signal_type> underperforms no-signal baseline — candidate for retirement from /gather-signals source table.`
- If all signal types are within ±5pp of control: `No signal type meaningfully beats the no-signal baseline. Signal collection is noise at current sample.`
- If total exposed rows < 15 across all signal types: `Small sample overall (n=<N>) — directional only, re-critique after more campaigns.`
```

**Verification:**
- [ ] Re-read the full Step 2. Findings 1, 2, 3 read in order without redundancy.
- [ ] Confirm the fallback language ("skip silently") is consistent with Step 2's existing "campaign not activated → execution-only" handling — Finding 3 is independent of campaign age, only on file presence.

---

## Task 2: Add gather-signals cap rule to Step 3

**Files:**
- Modify: `.claude/skills/critique/SKILL.md`

**Why second:** Ratings are where findings become enforcement. Without a cap rule, Finding 3 is an FYI chart the user can ignore.

- [ ] **Step 1: Add the rule to the "data-derived rating rules" table in Step 3**

Insert a new row in the table (currently has 6 rows, one per skill × rule). Add after the `capture-results` row:

```markdown
| `gather-signals` | Finding 3 shows any signal type with `n_exposed ≥ 15` lagging no-signal baseline by ≥5pp | ≤3 |
| `gather-signals` | Finding 3 shows **all** signal types lagging or tied with no-signal baseline (overall n ≥ 20) | ≤2 |
```

Also add `gather-signals` as a rated step in the per-step ratings loop. Currently Step 3 rates: research-client, icp-define, prospect, enrich-and-score, draft-sequences, activate, capture-results. Add `gather-signals` after `prospect` (matching pipeline order). If `gather-signals` was skipped in the run (signals.csv absent), skip its rating with the note `gather-signals: not run for this campaign.`

**Verification:**
- [ ] Confirm the step list in Step 3 now reads: research-client → icp-define → prospect → gather-signals → enrich-and-score → draft-sequences → activate → capture-results. Matches CLAUDE.md's pipeline order.
- [ ] Verify the cap rule's threshold (`n ≥ 15`) is higher than Finding 3's display threshold (`n ≥ 5`) — you don't want to cap a rating on a small-sample row.

---

## Task 3: Update the per-step ratings table template in Step 5

**Files:**
- Modify: `.claude/skills/critique/SKILL.md`

- [ ] **Step 1: Add `gather-signals` row to the critique.md template**

The template currently has rows for research-client, icp-define, prospect, enrich-and-score, draft-sequences, activate, capture-results. Insert a `gather-signals` row between `prospect` and `enrich-and-score`:

```markdown
| gather-signals | | ≤2 / ≤3 / — | |
```

- [ ] **Step 2: Add the Finding 3 output to the "Outcome signal" block**

The template currently has one placeholder line: `<either the two findings from Step 2 verbatim, OR the line "Campaign not activated — execution-only critique.", OR the line "Preliminary — campaign <N> days old, outcomes still landing.">`.

Replace with: `<the two (or three, if signals.csv exists) findings from Step 2 verbatim, OR the line "Campaign not activated — execution-only critique.", OR the line "Preliminary — campaign <N> days old, outcomes still landing.">`

**Verification:**
- [ ] Re-read Step 5's full critique.md template. Confirm the new row appears once and only once.
- [ ] Cross-check Step 2 → Step 3 → Step 5: the list of rated steps matches in all three places.

---

## Task 4: Add cross-client pattern scan for weak signal types

**Files:**
- Modify: `.claude/skills/critique/SKILL.md`

**Why fourth:** Single-client findings are suggestive; cross-client patterns are what justify changing `/gather-signals`' source table. This is the bridge from local finding to global signal about signals.

- [ ] **Step 1: Add a new pattern scan to Step 6**

Step 6 currently has two pattern scans: step-rating ≤2 across ≥2 clients, and step-level 0-reply across ≥2 clients. Add a third after the "Cross-client template weakness" block:

```markdown
Additionally, aggregate Finding 3 across all `clients/*/critique.md` that have outcome signals. If a `signal_type` appears in ≥2 clients with aggregate `n_exposed ≥ 20` AND its aggregate reply rate is ≤ the aggregate no-signal baseline, flag:

```
Cross-client weak signal type: <signal_type> has <replies>/<n> (<pct>%) across <k> clients — at or below the no-signal baseline (<base_pct>%).
Candidate for retirement from /gather-signals source table, or for confidence downweighting.
```

Parse prior critiques by reading the "Finding 3" block verbatim — no structured extract needed, since the table format in Finding 3 is deterministic. If a prior critique predates this feature and has no Finding 3 block, skip it silently. Don't backfill.
```

**Verification:**
- [ ] Confirm the scan only runs when ≥2 critiques with Finding 3 blocks exist. Single-client signal-type flags stay local.
- [ ] Confirm the aggregate-baseline calculation uses the aggregate no-signal row, not a per-client baseline — cross-client comparability matters more than per-client purity.

---

## Task 5: Wire the flag into `/gather-signals`

**Files:**
- Modify: `.claude/skills/gather-signals/SKILL.md`

**Why fifth:** Without this, Tasks 1-4 produce reports the user can act on but the system doesn't. The compounding asset only compounds if downstream skills read the signal.

- [ ] **Step 1: Locate Step 2 of gather-signals SKILL.md**

Per CLAUDE.md's skill contract, every skill already "reads prior-run signal" by globbing `clients/*/critique.md`. Check gather-signals' existing Step 2 text for how it currently does this extraction. (If the skill already has the general 3-most-frequent-weaknesses extract, we're adding a *specific* extract alongside it, not replacing.)

- [ ] **Step 2: Add weak-signal-type extraction to Step 2**

Append to gather-signals' Step 2:

```markdown
**Cross-client weak signal types.** In addition to the generic weakness extract, scan `clients/*/critique.md` for lines matching the pattern `Cross-client weak signal type: <signal_type>`. Build a list of `downweight_types = [<signal_type>, ...]`.

Apply at collection time: when a signal of a type in `downweight_types` is detected, record it in `signals.csv` as usual, but drop its `confidence` by one tier (`high → med`, `med → low`, `low → low`). Note in the `notes` column: `downweighted: flagged in <k> prior critiques`.

State at skill start: `Prior critiques flagged these signal types as underperforming: <list>. I'll downweight matching signals by one confidence tier.` If the list is empty, skip the announcement.

Don't suppress detection entirely — the signal type might recover with better framing or on a different ICP. Downweighting is reversible; retirement isn't.
```

**Verification:**
- [ ] Confirm the downweight only changes `confidence`, not `signal_type` or `source_url`. The audit trail (signal was found, was downweighted, here's why) stays complete.
- [ ] Confirm that if `critique.md` files don't yet use the Cross-client-weak-signal-type line format, the pattern scan just returns empty and the skill proceeds normally — no error path needed.

---

## Task 6: Add `/critique` as a consumer to the signals.csv block in CLAUDE.md

**Files:**
- Modify: `CLAUDE.md`

- [ ] **Step 1: Extend the Consumers list**

The `signals.csv` schema block in CLAUDE.md currently has a `**Consumers:**` subsection listing `/enrich-and-score`, `/draft-sequences`, and a `/critique may cite signal-to-reply correlations once…` forward-reference. Replace the forward-reference bullet with the present-tense version:

```markdown
- `/critique` reads this file alongside `results.csv` and computes per-signal-type reply rates (Finding 3). The cross-client aggregate feeds `/gather-signals`' confidence-downweight loop.
```

**Verification:**
- [ ] Re-read the full signals.csv block in CLAUDE.md. Consumers list now accurately reflects three live consumers with no forward-references.
- [ ] Grep for any other stale "not yet implemented" language about signal-outcome correlation in the repo — remove or update.

---

## Risks and mitigations

- **Attribution noise.** The top-confidence-per-company rule credits the strongest signal with every reply at that company, even when the sequence opener actually leaned on a different signal. This biases reply rates *up* for consistently-strong signal types and *down* for weaker ones — which is the wrong direction for spotting hidden-useful signals. Mitigation: the small-n guards and the no-signal baseline catch the most egregious cases. v2 (explicit attribution in sequence headers) fixes this properly.
- **Survivorship bias in disqualified prospects.** `results.csv` excludes disqualified rows by design. If a signal type correlates with disqualification (e.g., `hiring_surge` at too-small companies → disqualified on size), we'll undercount its true exposure. Mitigation: the existing disqualifier check in `/enrich-and-score` runs *before* the signal scoring tier, so this is mostly fine; revisit if a signal type consistently disappears from results.csv vs enriched.csv.
- **Premature retirement.** A signal type can look weak in 2 campaigns and strong in the 10th because of ICP-specific interaction effects. Mitigation: the plan does not auto-retire anything. The cross-client flag is a *candidate*; the human decides.
- **Baseline inflation.** If no-signal companies are systematically lower-tier (lower ICP score), their lower reply rate will make any signal type look good by comparison. Mitigation: v2 could stratify by `icp_score` tier; for now, the small-sample annotation and the "within ±5pp of control" band protect against over-reading tiny deltas.
- **Pattern-string fragility.** Task 4's cross-client scan parses `critique.md` text. If a future edit to the template drifts the wording, the scan silently stops finding flags. Mitigation: the exact string (`Cross-client weak signal type:`) is the contract — documented in Task 4. Don't change it without grepping consumers.

---

## Verification (end-to-end)

After all tasks complete, simulate a `/critique` run mentally on `theline` or `claybootcamp` (whichever has `signals.csv` + `results.csv`):

- [ ] Finding 3 appears in the user-facing output and in the saved critique.md.
- [ ] If a weak signal type exists for that client, `gather-signals` cap fires and rating is ≤3.
- [ ] If ≥2 clients show the same weak signal type, Step 6's pattern scan surfaces it.
- [ ] A subsequent run of `/gather-signals` (on any client) announces the downweight list at start and applies it to the `confidence` column in the new `signals.csv`.
- [ ] Re-run `/critique` on that new campaign. The downweighted signals still appear in the join; if they stay weak, the cross-client flag gets louder, not quieter.

That's the loop. The asset compounds when every leg runs.
