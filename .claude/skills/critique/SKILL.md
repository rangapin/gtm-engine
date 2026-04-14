---
name: critique
description: Capture a post-run critique of each pipeline step's output for a client campaign. Writes clients/<client>/critique.md. Use after /activate, or any time to revisit a past campaign. The accumulated critiques across clients are the signal for skill improvements.
---

# /critique — Per-run quality signal

When the user runs `/critique <client-name>`:

This skill is the **eval loop primitive** for gtm-engine. Each run produces two kinds of signal: (1) how well Claude executed each step (the per-step 1-5 rating), and (2) whether the campaign actually worked (reply rate by tier, template performance). Without a structured critique, both stay vibes — not actionable. This skill turns both into a one-page artifact the user (or Claude) can diff across clients to spot recurring weak spots. The outcome signal (2) is only available when `/capture-results` has produced a `results.csv`; when absent, the skill still functions and captures (1) normally.

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

### Step 2: Interactive rating

For each step that produced output, ask the user two questions:

```
<step-name>: how was the output?
  1 = useless, 2 = weak, 3 = ok, 4 = good, 5 = excellent
What was the single weakest thing about it? (one line, skip if nothing)
```

Don't batch — ask step by step so the user can actually think. If the user wants to skip a step ("idk, fine I guess"), record rating=3 with a note that it wasn't closely reviewed. Vague ratings are less useful but better than made-up specifics.

### Step 2.5: Outcome signal (if results.csv exists)

Check for `clients/<client-name>/results.csv`. If it doesn't exist, skip this step entirely — silent, not an error. If it exists but has fewer than 3 rows where `replied` is non-null (true or false), also skip — not enough data to say anything directional.

Otherwise compute two findings from the file.

**Finding 1 — reply rate by tier.**

Group rows by `tier` column. For each group, count total rows and rows where `replied=true`. Format:

```
Outcome signal for this run:
  High (70+):  <replies>/<total>   replies (<pct>%)
  Med  (40-69): <replies>/<total>   replies (<pct>%)
  Low  (1-39):  <replies>/<total>   replies (<pct>%)
```

Only show tiers that have at least one row. After the table, add exactly one annotation:

- If reply-rate is strictly decreasing by tier (High > Med > Low): `Scoring directionally correct — higher-tier prospects reply more.`
- If Low's reply rate exceeds High's: `Scoring inverted — Low tier out-replied High. Score rubric likely wrong; worth a pass.`
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

For each step, also compute sends = total rows where that step was reached. With manual-fill v1, "reached step K" is hard to detect precisely — approximate as: all rows with `replied=true OR replied=false` are assumed to have been sent through the full sequence (i.e., `sends = total_with_replied_non_null` for every step). This is a simplification; the Instantly auto-pull in v2 will give exact per-step sends.

If any step has 0 replies AND `sends >= 10`, add an annotation:
`Step <K> got 0 replies across <sends> sends — candidate for rewrite or retirement.`

Output both findings to the user in the terminal (so they see it before filling out Step 3 skill-edit suggestions). Also save them for the written critique file in Step 4.

### Step 3: Extract skill-edit suggestions

After all per-step ratings, ask:

```
Anything concrete that would have made this run better?
- A rule you wish the drafting skill had
- A question you wish /icp-define had asked
- An enrichment field that was missing
- A disqualifier we should have caught earlier
```

Capture up to 3. These are the deliverable: specific, testable, small.

### Step 4: Write the critique file

Write `clients/<client-name>/critique.md`:

```markdown
# Critique — <client>

**Run date:** <ISO date>
**Channel:** <HeyReach / Smartlead / Instantly / Gmail / CSV>
**Prospects activated:** <N>
**Outcome (fill in later):** <replies / meetings / silence>

## Per-step ratings

| Step | Rating (1-5) | Weakest thing |
|---|---|---|
| research-client | | |
| icp-define | | |
| prospect | | |
| enrich-and-score | | |
| draft-sequences | | |
| activate | | |
| capture-results | | |

## Outcome signal

<either the two findings from Step 2.5 verbatim, OR the line "No results.csv yet — run /capture-results when outcomes start landing.">

## Skill-edit candidates
1. <concrete, testable suggestion>
2. …
3. …

## One-line verdict
<one sentence summarising this run>
```

If `critique.md` already exists (re-running this skill), show the user the current contents and ask whether to overwrite, amend, or abort.

### Step 5: Surface patterns (if multiple clients exist)

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

Additionally, if ≥2 clients have `results.csv` files where the same step number shows 0 replies across ≥10 sends (per Finding 2 logic in Step 2.5), flag it as a cross-client template weakness:

```
Cross-client template weakness: step 3 got 0 replies across 2 clients (24 total sends).
This step's template is a candidate for rewrite — the weakness is not client-specific.
```

Don't propose the rewrite here — surface only.

### Step 6: Log

Append to `clients/<client-name>/logs/critique.log.md`:
- Date of critique
- Ratings given
- Skill-edit candidates captured
- Whether this triggered a cross-client pattern flag

## Keep it honest

Don't help the user rationalize high ratings. If they say "5 / excellent" on a step that had a clearly weak output (e.g., sequences with template-shaped copy, ICP without discriminators), gently push back once:

```
You rated draft-sequences a 5, but the theline run flagged 3 of 4 drafts as
still in v1 AI-voice. Are we rating the output or the effort?
```

Accept whatever the user says after the pushback — this is their call, not yours. But better to ask once than to accumulate dishonest signal.

## What NOT to do

- Don't propose skill edits in this file. That's what the patterns scan in Step 5 is for, and even then the fix goes in a separate conversation.
- Don't rate the client or the prospect list ("bad domain, few signals"). Rate the *skill's output* given the inputs it had.
- Don't aggregate scores into a single "run grade." Per-step ratings are the usable signal; a composite number just hides where the weakness is.

## When to run

- **End of campaign-run** — final step after activate (campaign-run calls this automatically).
- **Any time** — to revisit a past client and add a critique if one wasn't captured at the time.
- **Before a skill-improvement pass** — run across all past clients to surface which step actually needs work.
