---
name: critique
description: Capture a post-run critique of each pipeline step's output for a client campaign. Writes clients/<client>/critique.md. Use after /activate, or any time to revisit a past campaign. The accumulated critiques across clients are the signal for skill improvements.
---

# /critique — Per-run quality signal

When the user runs `/critique <client-name>`:

This skill is the **eval loop primitive** for gtm-engine. Each run of the pipeline produces outputs of variable quality. Without a structured critique, "the copy was so-so" stays a vibe — not actionable signal. This skill turns that vibe into a one-page artifact the user (or Claude) can diff across clients to spot recurring weak spots.

## What to do

### Step 1: Load context

Read the artifacts that exist in `clients/<client-name>/`:
- `brief.md` (from research-client)
- `icp.json` (from icp-define)
- `prospects.csv` (from prospect)
- `prospects.enriched.csv` (from enrich-and-score)
- `sequences/` (from draft-sequences)
- `logs/activate.log.md` (from activate)

If the run is partial, that's fine — rate only the steps that produced output.

### Step 2: Interactive rating

For each step that produced output, ask the user two questions:

```
<step-name>: how was the output?
  1 = useless, 2 = weak, 3 = ok, 4 = good, 5 = excellent
What was the single weakest thing about it? (one line, skip if nothing)
```

Don't batch — ask step by step so the user can actually think. If the user wants to skip a step ("idk, fine I guess"), record rating=3 with a note that it wasn't closely reviewed. Vague ratings are less useful but better than made-up specifics.

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
**Channel:** <HeyReach / Smartlead / Gmail / CSV>
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
