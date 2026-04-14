---
name: campaign-run
description: Full campaign workflow from domain to launch. Chains all skills in order with review gates between each step. This is the main entry point for running a complete outbound campaign for a client.
---

# /campaign-run - Full Campaign Workflow

When the user runs `/campaign-run <domain>`:

## Overview

This playbook chains all skills in sequence with **3 tiered gates** — but the Budget tier fires in two halves (preview, then spend). See CLAUDE.md "Gate tiering".

```
/campaign-init <domain>
/research-client <client>    } silent auto-run — no gates
/icp-define <client>         } (cheap, reversible, no API spend)
    |
/prospect <client>           — Apollo search is cheap; runs without a pre-gate
    |
    v  ━━━ GATE 1A: Budget preview ━━━
    |     Top-10 rows from prospects.csv inline. User edits the CSV to shortlist.
    |     No credits spent yet.
    |
/gather-signals <client>     — soft gate of its own (free-API, latency-visible)
    |
    v  ━━━ GATE 1B: Budget spend ━━━
    |     "~N Apollo credits on M rows in prospects.csv (after edits). Proceed?"
    |     Accepts: yes / no / subset <top-N>
    |
/enrich-and-score <client>
    |
/draft-sequences <client>
    |
    v  ━━━ GATE 2: Quality ━━━
    |     Full recap: brief summary + ICP one-pager + top-10 prospects + 3 sample sequences.
    |     This is the critical content gate. Take your time here.
    |
    v  ━━━ GATE 3: Irreversible ━━━
    |     Paused-first reminder. Final go to push to CRM + sequencer.
    |
/activate <client>
    |
/critique <client>           — capture per-step ratings for the eval loop
Done.
```

Three tiers (Budget / Quality / Irreversible), but the Budget tier has a preview half at the end of `/prospect` and a spend half at the start of `/enrich-and-score`. The preview half is what lets the user see real rows *before* approving credit spend.

Silent doesn't mean blind: `clients/<client>/assumptions.md` accumulates one-line assumption statements from each silent step, which the user can skim at any time. If research or ICP is clearly wrong, the user stops the run before Gate 1A.

## How to run

### Option A: Full auto-chain

Run the entire chain. Pause at the 3 tiered gates — not every step.

At the start, tell the user:

```
Starting full campaign for <domain>.

9 steps, 3 tiered gates (Budget tier has a preview half + spend half):
- Silent (auto-run): init, research-client, icp-define
- /prospect — runs silent (Apollo search is cheap)
- Gate 1A — Budget preview: real top-10 rows; edit prospects.csv to shortlist
- /gather-signals (free, soft gate)
- Gate 1B — Budget spend: Apollo credits on the post-edit row count
- /enrich-and-score
- /draft-sequences
- Gate 2 — Quality: full recap (the critical gate)
- Gate 3 — Irreversible: paused-first reminder
- /activate, then /critique

Silent steps write to clients/<client>/assumptions.md — skim any time.
Stop me before Gate 1A if the ICP is off.

Ready?
```

### Option B: Step-by-step

The user can run each skill individually. This is useful when:
- They want to take a break between steps
- They need to do manual work between steps (e.g., client review of the ICP)
- They want to re-run a specific step

Guide them:
```
You can also run each step separately:

/campaign-init <domain>     # start here
/research-client <client>   # then this
/icp-define <client>        # then this
/prospect <client>          # then this
/enrich-and-score <client>  # then this
/draft-sequences <client>   # then this
/activate <client>          # then this
/critique <client>          # finally this — capture signal for skill improvements
```

## Gate behavior

Three gates, each with a specific shape:

**Gate 1A — Budget preview** (at the end of `/prospect`)
- Real top-10 rows from `prospects.csv` shown inline. Not a summary, not counts — actual companies + contacts.
- No credits spent yet. The user can edit `prospects.csv` (delete rows to shortlist) before continuing.
- Proceed on explicit "yes" or an edit instruction (e.g., "drop rows 3, 7, 12 and continue").
- If the prospect list is clearly wrong (ICP drift), stop and re-run `/prospect` with tighter filters.

**Gate 1B — Budget spend** (at the start of `/enrich-and-score`)
- State the credit estimate reading the *live* `prospects.csv` row count (post-edit).
- Accepts: `yes` / `no` / `subset <top-N>` for partial enrichment.
- Between 1A and 1B, `/gather-signals` runs with its own soft gate (free-API, latency-visible). Don't collapse it into either half.

**Gate 2 — Quality** (after draft-sequences, before activate)
- Full recap in one view: brief summary (3 bullets), ICP one-pager, prospect count + top-10 scored rows, 3 sample sequences inline.
- Don't paginate across turns — one scroll, everything visible.
- Accept edits to any piece. This is the irreversible-copy gate; take the time.
- Only proceed on explicit "yes."

**Gate 3 — Irreversible** (before activate)
- Paused-first reminder: "Campaigns will be created paused/unsent. You resume in the sequencer UI."
- Final go/no-go. Proceed only on explicit "yes."

Never skip Gate 2 or Gate 3 even if the user says "just do everything." Gate 1B (spend) can be bypassed only if the user has explicitly pre-authorized credit spend for this run. Gate 1A (preview) is cheap — don't skip it; it's the single best defense against wasting credits on a bad list.

## Resuming a partial run

If the user ran some steps earlier and wants to continue:

1. Check which files already exist in `clients/<client-name>/`
2. Identify the last completed step
3. Offer to resume from the next step:

```
I see you already have:
- brief.md (research done)
- icp.json (ICP defined)
- prospects.csv (prospecting done)

Next step would be /enrich-and-score. Want to continue from there?
```

## Multi-client management

If the user is managing multiple clients:

```
Active clients:
1. acme (last step: /draft-sequences, 3 days ago)
2. stripe (last step: /icp-define, today)
3. notion (last step: /prospect, 1 week ago)

Which client do you want to work on?
```

Check the `clients/` directory and the logs in each client folder to determine status.

## Timing expectations

Rough time per step (including review):
- Init: 1 minute
- Research: 5-10 minutes
- ICP: 5-10 minutes
- Prospect: 5-15 minutes (depends on list size)
- Enrich + Score: 10-20 minutes (depends on list size and API speed)
- Draft sequences: 15-30 minutes (depends on how many and how custom)
- Activate: 5-10 minutes

Total: roughly 45 minutes to 1.5 hours for a full campaign, depending on list size and how thorough the reviews are.

## What makes a good campaign

A campaign built with this workflow should have:
- A well-researched brief that shows you understand the client's business
- An ICP that's specific enough to produce a targeted list
- A prospect list of 20-50 high-fit contacts (not 500 generic ones)
- Enrichment with 70%+ email coverage
- Scores with clear reasoning
- Sequences that reference specific signals and feel hand-written
- Clean CRM records with attribution

This is the quality bar. Speed matters, but not at the expense of relevance.
