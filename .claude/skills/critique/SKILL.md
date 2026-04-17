---
name: critique
description: Capture a post-run critique of each pipeline step's output for a client campaign. Writes clients/<client>/critique.md. Use after /activate, or any time to revisit a past campaign. The accumulated critiques across clients are the signal for skill improvements.
---

# /critique — Per-run quality signal

When the user runs `/critique <client-name>`:

This skill is the **eval loop primitive** for gtm-engine. Each run produces two kinds of signal: (1) whether the campaign actually worked (reply rate by tier, template performance — read from `results.csv`), and (2) how well Claude executed each step (the per-step 1-5 rating). **The outcome signal comes first and anchors the ratings** — without it, per-step ratings are vibes, and the eval loop doesn't compound. This skill hard-gates on outcome data when the campaign is old enough to have produced any.

**Closed loop:** per CLAUDE.md's skill contract (Step 2), every pipeline skill reads `clients/*/critique.md` on load and extracts the 3 most-repeated weaknesses for its step, biasing its behavior accordingly. Writing the critique is only half the loop — the reads are what make it a compounding asset.

## What to do

### Step 1: Load context

Read the artifacts that exist in `clients/<client-name>/`:
- `brief.md` (from research-client)
- `icp.json` (from icp-define)
- `prospects.csv` (from prospect)
- `prospects.enriched.csv` (from enrich-and-score)
- `sequences/` (from draft-sequences)
- `logs/activate.log.md` (from activate)
- `results.csv` (from capture-results, optional)

If the run is partial, that's fine — rate only the steps that produced output.

### Step 2: Outcome gate

Before any rating, determine whether outcome data exists and whether it's rich enough to ground the critique.

**Compute:**
- `campaign_age_days` = days since `logs/activate.log.md` mtime. If that file doesn't exist, fall back to `prospects.enriched.csv` mtime. If neither exists, treat campaign as not-yet-activated.
- `results_exists` = whether `results.csv` is present.
- `fill_rate` = (rows with `replied` non-null) / (total non-DQ rows). Only computed if `results_exists`.

**Gate decision:**

| Situation | Action |
|---|---|
| Campaign not activated (no activate log, no enriched prospects) | Proceed as "execution-only critique" — skip outcome findings. Warn: "Campaign hasn't run. Ratings will cover skill execution only; no outcome signal." |
| Campaign <7 days old | Proceed as "preliminary critique." Print: "Campaign is only <N> days old — outcomes still landing. Ratings are preliminary; re-run /critique once /capture-results shows ≥50% fill." Do not block. |
| Campaign ≥7 days old AND no `results.csv` | **Block.** Print: "Campaign is <N> days old but no results.csv exists. Outcome-blind ratings are worse than none. Run /capture-results <client>, fill replies, then re-run /critique." Exit the skill. |
| Campaign ≥7 days old AND `fill_rate < 50%` | **Block.** Print: "results.csv is only <X>% filled after <N> days. The ratings won't be grounded. Fill more outcome cells and re-run /critique. (If the campaign really got no replies, record `replied=false` on the sends so the skill can distinguish 'unreplied' from 'unrecorded'.)" Exit the skill. |
| Campaign ≥7 days old AND `fill_rate ≥ 50%` | Proceed with full outcome signal. |

**If proceeding past the gate**, compute two findings from `results.csv` (skip when the campaign isn't activated yet).

**Finding 1 — reply rate by tier.**

Group rows by `tier` column. For each group, count total rows and rows where `replied=true`. Format:

```
Outcome signal for this run (fill-rate: <pct>%, campaign age: <N> days):
  High (70+):   <replies>/<total>   replies (<pct>%)
  Med  (40-69): <replies>/<total>   replies (<pct>%)
  Low  (1-39):  <replies>/<total>   replies (<pct>%)
```

Only show tiers that have at least one row. After the table, add exactly one annotation:

- If reply-rate is strictly decreasing by tier (High > Med > Low): `Scoring directionally correct — higher-tier prospects reply more.`
- If Low's reply rate exceeds High's: `Scoring inverted — Low tier out-replied High. Score rubric likely wrong.`
- Otherwise (e.g., Med highest, or ties): `Scoring mixed — no clear tier signal yet.`
- If total rows with `replied` non-null < 10 across all tiers, append: `Small sample (n=<N>) — directional only.`

**Finding 2 — replies by step.**

Group rows by `replied_on_step` (ignore rows where `replied=false` or `replied_on_step` is blank). Count replies per step 1..N where N is the max step seen. Format:

```
Replies by step:
  Step 1: <count>
  Step 2: <count>
  Step 3: <count>
  Step 4: <count>
```

For each step, approximate `sends` = total rows where `replied` is non-null (simplification — assumes full-sequence exposure; Instantly auto-pull in v2 will give exact per-step sends).

If any step has 0 replies AND `sends ≥ 10`, add an annotation:
`Step <K> got 0 replies across <sends> sends — candidate for rewrite or retirement.`

**Finding 3 — reply rate by signal type.**

Only runs if `clients/<client-name>/signals.csv` exists. If absent, skip silently and log `signals.csv absent — signal-correlation finding skipped.` in the critique log.

Build the join:
1. For each company in `signals.csv`, pick the **top-confidence row** (the file is already sorted by `confidence` desc, `detected_at` desc per CLAUDE.md's `signals.csv` schema, so this is row 0 per `company_domain`). Build a map `company_signal = {company_domain: signal_type}`.
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

Print all findings to the user before Step 3. Also save them for the written critique file in Step 5.

### Step 3: Per-step ratings (evidence-derived when possible)

For each pipeline step that produced output, ask the user rating + weakest-thing. **When outcome data exists**, propose a rating cap derived from the data *first*, then let the user confirm or override. **When outcome data is absent** (fresh campaign or execution-only mode), fall back to pure manual rating.

The data-derived rating rules (apply each rule that fires; use the most pessimistic cap):

| Skill | Rule | Suggested cap |
|---|---|---|
| `enrich-and-score` | Finding 1 shows scoring inverted (Low > High reply rate) | ≤2 |
| `enrich-and-score` | Finding 1 shows "mixed" with n ≥ 20 | ≤3 |
| `draft-sequences` | Any step got 0 replies across ≥10 sends (Finding 2) | ≤3 |
| `draft-sequences` | No step got any reply across ≥20 sends (overall reply rate 0%) | ≤2 |
| `prospect` | High-tier rows are <20% of the filled set (ICP too loose, too many Low/Med) | ≤3 |
| `capture-results` | Campaign ≥14 days old and fill-rate < 80% | ≤3 |
| `gather-signals` | Finding 3 shows any signal type with `n_exposed ≥ 15` lagging no-signal baseline by ≥5pp | ≤3 |
| `gather-signals` | Finding 3 shows **all** signal types lagging or tied with no-signal baseline (overall n ≥ 20) | ≤2 |

When a data-derived cap fires, present it like:

```
draft-sequences — data suggests ≤3 (step 3 got 0 replies across 18 sends).
  Rate (1-5, default 3): _
  Weakest thing (one line, skip if nothing): _
```

The user can override upward with justification ("I saw real engagement on step 3 replies that bounced to phone, not email"), but the default anchors to the data. If no data-derived cap fires for a step, show the open-ended question without a suggested number:

```
<step-name>: how was the output?
  1 = useless, 2 = weak, 3 = ok, 4 = good, 5 = excellent
What was the single weakest thing about it? (one line, skip if nothing)
```

Don't batch — ask step by step so the user can actually think. If the user skips ("idk, fine I guess"), record rating=3 with a note that it wasn't closely reviewed.

### Step 4: Extract skill-edit suggestions

After all per-step ratings, ask:

```
Anything concrete that would have made this run better?
- A rule you wish the drafting skill had
- A question you wish /icp-define had asked
- An enrichment field that was missing
- A disqualifier we should have caught earlier
```

Capture up to 3. These are the deliverable: specific, testable, small.

### Step 5: Write the critique file

Write `clients/<client-name>/critique.md`:

```markdown
# Critique — <client>

**Run date:** <ISO date>
**Channel:** <HeyReach / Smartlead / Instantly / Gmail / CSV>
**Prospects activated:** <N>
**Campaign age at critique:** <N> days
**Fill-rate at critique:** <pct>% (<filled>/<total>)
**Gate outcome:** <full / preliminary / execution-only>

## Per-step ratings

| Step | Rating (1-5) | Data-derived cap | Weakest thing |
|---|---|---|---|
| research-client | | — | |
| icp-define | | — | |
| prospect | | ≤3 or — | |
| gather-signals | | ≤2 / ≤3 / — | |
| enrich-and-score | | ≤2 / ≤3 / — | |
| draft-sequences | | ≤2 / ≤3 / — | |
| activate | | — | |
| capture-results | | ≤3 or — | |

Record the cap column as `—` when no rule fired, or the cap value when one did. If the user overrode a cap upward, mark rating with `*` and note the override reason in the weakest-thing column.

## Outcome signal

<the two (or three, if signals.csv exists) findings from Step 2 verbatim, OR the line "Campaign not activated — execution-only critique.", OR the line "Preliminary — campaign <N> days old, outcomes still landing.">

## Skill-edit candidates
1. <concrete, testable suggestion>
2. …
3. …

## One-line verdict
<one sentence summarising this run>
```

If `critique.md` already exists (re-running this skill), show the user the current contents and ask whether to overwrite, amend, or abort.

### Step 6: Surface patterns (if multiple clients exist)

After writing the file, scan all `clients/*/critique.md` files. If 2+ clients rate the same step ≤ 2, flag it:

```
Pattern across clients: draft-sequences rated ≤2 in 3 of 4 runs.
Top weaknesses called out:
- "too AI-shaped" (theline)
- "forgot to reference the specific signal" (acme)
- "signature format wrong" (stripe)

This step is the top candidate for a skill-improvement pass.
```

Don't propose the fix here — just surface the pattern. Fixing the skill is a separate conversation with the user.

Additionally, if ≥2 clients have `results.csv` files where the same step number shows 0 replies across ≥10 sends (per Finding 2 logic in Step 2), flag it as a cross-client template weakness:

```
Cross-client template weakness: step 3 got 0 replies across 2 clients (24 total sends).
This step's template is a candidate for rewrite — the weakness is not client-specific.
```

Don't propose the rewrite here — surface only.

Additionally, aggregate Finding 3 across all `clients/*/critique.md` that have outcome signals. If a `signal_type` appears in ≥2 clients with aggregate `n_exposed ≥ 20` AND its aggregate reply rate is ≤ the aggregate no-signal baseline, flag:

```
Cross-client weak signal type: <signal_type> has <replies>/<n> (<pct>%) across <k> clients — at or below the no-signal baseline (<base_pct>%).
Candidate for retirement from /gather-signals source table, or for confidence downweighting.
```

Parse prior critiques by reading the "Finding 3" block verbatim — no structured extract needed, since the table format in Finding 3 is deterministic. If a prior critique predates this feature and has no Finding 3 block, skip it silently. Don't backfill.

The exact flag string — `Cross-client weak signal type:` — is the contract read by `/gather-signals` on its next run. Do not reword.

### Step 7: Log

Append to `clients/<client-name>/logs/critique.log.md`:
- Date of critique
- Campaign age at critique, fill-rate at critique
- Gate outcome (blocked / preliminary / execution-only / full)
- Ratings given — mark which were data-derived vs. user-overridden
- Skill-edit candidates captured
- Whether this triggered a cross-client pattern flag

## Keep it honest

The evidence-derived cap in Step 3 is the main honesty mechanism — it anchors ratings to outcomes rather than feelings. When the user overrides a cap upward, push back once:

```
You rated draft-sequences a 4, but step 3 got 0 replies across 18 sends.
The data says ≤3. Are we rating the intent or the result?
```

Accept whatever the user says after the pushback — this is their call. But the whole point of the anchor is to make overrides visible rather than letting vibes leak in silently.

When outcome data is absent (preliminary or execution-only mode), the old honesty rule still applies: if the user says "5 / excellent" on a step that had a clearly weak output, gently push back once citing the specific weakness.

## What NOT to do

- Don't propose skill edits in this file. That's what the patterns scan in Step 6 is for, and even then the fix goes in a separate conversation.
- Don't rate the client or the prospect list ("bad domain, few signals"). Rate the *skill's output* given the inputs it had.
- Don't aggregate scores into a single "run grade." Per-step ratings are the usable signal; a composite number just hides where the weakness is.
- **Don't run the critique on old campaigns with no outcome data.** The Step 2 gate blocks this on purpose. An outcome-blind critique on a 30-day-old campaign pollutes the compounding critique signal with made-up ratings. Fill the data or don't run.
- Don't bypass the gate by editing the results.csv to add fake `replied=false` rows. The gate exists because bad data is worse than no data.

## When to run

- **End of campaign-run** — final step after activate (campaign-run calls this automatically, in "preliminary" mode — outcomes haven't landed yet). The preliminary critique captures execution-only ratings.
- **~10-14 days after activation** — once outcomes have landed and `/capture-results` fill-rate is ≥50%. This is the real critique, with full outcome grounding. Overwrites or amends the preliminary one.
- **Before a skill-improvement pass** — run across all past clients to surface which step actually needs work.
