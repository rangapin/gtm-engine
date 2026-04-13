---
name: campaign-status
description: Report state of one or all client campaigns. Shows which steps are complete, what ran most recently, open flags from RESUME.md, and the next recommended step. Use when resuming work, checking multi-client progress, or before starting a new campaign action.
---

# /campaign-status - Inspect Client Campaign State

When the user runs `/campaign-status` (all clients) or `/campaign-status <client-name>` (one client):

## What to do

### Step 1: Pick scope

- No argument: scan every directory under `clients/` and report on all of them.
- `<client-name>` argument: report on that client only.

If `clients/` is empty or missing, tell the user and suggest `/campaign-init <domain>`.

### Step 2: Derive state from the file system

The file system IS the state (per `CLAUDE.md` - "The skill contract"). Determine each client's stage by which files exist:

| Files present | Stage |
|---|---|
| only folder + `logs/init.log.md` | initialized |
| + `brief.md` | research done |
| + `icp.json` | ICP defined |
| + `prospects.csv` (or `prospects.seed.csv`) | prospected |
| + `prospects.enriched.csv` | enriched + scored |
| + `sequences/*.md` | sequences drafted |
| + `logs/activate.log.md` with non-placeholder content | activated |

The last file written tells you the last completed step. **Do not call MCP tools from this skill** — read-only from disk.

### Step 3: Pull quick stats per client

For each client, read (do not summarize raw JSON to the user):
- `brief.md` first line or H1 — one-sentence client description if available.
- `prospects.enriched.csv` if present — count rows, count non-disqualified, score distribution (High 70+, Med 40-69, Low 1-39, DQ).
- `sequences/` if present — count sequence files (excluding `_SEGMENT_*_template.md`, which are templates not per-prospect drafts).
- `logs/RESUME.md` if present — this is a human-written flag file. Surface its "Next action" and "Open flags" sections verbatim.
- Most recent `logs/*.log.md` modification time — "last activity: <N> days ago".

### Step 4: Report

**Single client format:**

```
Client: <client-name>
Stage: <stage from table above>
Last activity: <N> days ago (<filename>)

Counts:
- Prospects: <N> (enriched: <M>, non-DQ: <K>)
- Score distribution: High <N> | Med <N> | Low <N> | DQ <N>
- Sequences drafted: <N>

Next step: /<skill-name> <client-name>
    <one line: what that step will do>

<if RESUME.md exists, inline its content under a "Manual resume notes" header>

<if any logs flag errors or placeholders, surface them as "Flags" bullets>
```

**Multi-client format:**

```
Clients: <N>

1. <client-a>    stage=<stage>    last=<N>d ago    next=/<skill>
2. <client-b>    stage=<stage>    last=<N>d ago    next=/<skill>
3. ...

Run /campaign-status <client-name> for detail on one client.
```

Sort by most recent activity (freshest first).

### Step 5: Do not log

This is a read-only skill. Do not write log files — `campaign-status` should leave no trace in `clients/`.

## Next-step mapping

After determining stage, recommend:

| Stage | Next skill |
|---|---|
| initialized | `/research-client <client>` |
| research done | `/icp-define <client>` |
| ICP defined | `/prospect <client>` |
| prospected | `/enrich-and-score <client>` |
| enriched + scored | `/draft-sequences <client>` |
| sequences drafted | `/activate <client>` |
| activated | no next step — report "campaign live, check sequencer UI" |

If a `RESUME.md` exists with a "Next action" section, prefer that over the table — it reflects out-of-band context the file-system check can't see.

## What NOT to do

- Don't call Apollo, Clay, HeyReach, Smartlead, or Attio MCP. State comes from the file system, not live platforms.
- Don't write anything to `clients/`. No logs, no status files, no side effects.
- Don't dump raw CSV contents. Summarize to counts and distributions.
- Don't assume activation just because `activate.log.md` exists — check that its content is not a placeholder (e.g., a CSV-fallback attempt left behind). `RESUME.md` is the tiebreaker.
