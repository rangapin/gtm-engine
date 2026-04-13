---
name: campaign-run
description: Full campaign workflow from domain to launch. Chains all skills in order with review gates between each step. This is the main entry point for running a complete outbound campaign for a client.
---

# /campaign-run - Full Campaign Workflow

When the user runs `/campaign-run <domain>`:

## Overview

This playbook chains all skills in sequence:

```
/campaign-init <domain>
    |
    v  [user reviews folder structure]
/research-client <client>
    |
    v  [user reviews brief]
/icp-define <client>
    |
    v  [user reviews ICP]
/prospect <client>
    |
    v  [user reviews prospect list]
/enrich-and-score <client>
    |
    v  [user reviews scores]
/draft-sequences <client>
    |
    v  [user reviews sequences - CRITICAL GATE]
/activate <client>
    |
    v  [user confirms push - FINAL GATE]
Done.
```

## How to run

### Option A: Full auto-chain

Run the entire chain. Pause at every review gate and wait for user approval before continuing.

At the start, tell the user:

```
Starting full campaign for <domain>.

This will run 7 steps:
1. Initialize project folder
2. Research the client
3. Define ICP
4. Find prospects (uses Apollo credits)
5. Enrich and score (uses Apollo credits)
6. Draft email sequences
7. Push to CRM and sequencer (irreversible)

I'll pause at every step for your review. Nothing goes live without your approval.

Ready to start?
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
/activate <client>          # finally this
```

## Review gate behavior

At each gate:

1. **Show the output** (brief, ICP, prospect list, scores, sequences)
2. **Ask for approval** ("Does this look right?")
3. **Accept edits** (the user may want to change things)
4. **Only proceed when the user says yes**
5. **Never skip a gate**, even if the user says "just do everything"

The sequences review gate and the activation gate are the two most important. Double-check at those points.

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
