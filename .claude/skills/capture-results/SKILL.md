---
name: capture-results
description: Generate a results.csv skeleton from prospects.enriched.csv so the user can manually record campaign outcomes (replies, bounces, meetings). Reads and preserves user edits on re-run. Feeds /critique's outcome-signal analysis. Use after /activate when replies start landing, and any time you want to refresh the skeleton with newly added prospects.
---

# /capture-results — Manual outcome capture

When the user runs `/capture-results <client-name>`:

This skill is the **outcome primitive** for gtm-engine. Without a results file, `/critique` can only rate skill execution quality — it can't tell you whether the campaign worked. This skill writes a pre-filled CSV skeleton that the user fills in as replies land. It never overwrites user edits.

**Gate tier:** silent. No credit spend, no external writes — fully local file I/O. Per CLAUDE.md gate tiering, no review gate is required.

## What to do

### Step 1: Load context

Read from `clients/<client-name>/`:
- `prospects.enriched.csv` (required — the source of prospects)
- `logs/activate.log.md` (optional — for channel derivation)
- `results.csv` (optional — if present, we're in re-run mode)

If `prospects.enriched.csv` is missing, stop and tell the user: "No enriched prospects for `<client>`. Run `/enrich-and-score <client>` first."

### Step 2: Read prior-run signal

Glob `clients/*/critique.md` across all clients. Extract the 3 most-repeated weaknesses flagged for `capture-results` specifically (if any). On early runs there will be none — state that plainly: "No prior critiques mention capture-results yet."

When prior flags exist, state them: "Prior runs flagged: X, Y, Z — I'll bias against these." Then adjust behavior accordingly (e.g., if past critiques flag "channel guessed wrong", default to prompting rather than keyword-scanning).

### Step 3: Branch on whether `results.csv` exists

**If `results.csv` is absent → first-run mode (Step 3A).**
**If `results.csv` exists → re-run mode (Step 3B).**

### Step 3A: First-run — write skeleton

1. Load all rows from `prospects.enriched.csv`. Filter out rows where `disqualified=true`.
2. Derive channel:
   - Read `logs/activate.log.md` body as plain text.
   - Lowercase it, then scan for channel keywords: `heyreach` or `linkedin` → `linkedin`; `smartlead` or `instantly` → `email`; `gmail` → `gmail`.
   - Ignore the word "linkedin" when it appears inside `linkedin.com` URLs (use a word-boundary check, not substring).
   - If exactly one channel keyword class matches → use it.
   - If multiple classes match OR zero match → prompt the user once: "Channel for this run? (linkedin / email / gmail)". Accept the answer.
   - If `activate.log.md` doesn't exist → prompt with the same question; note in the log that channel had to be asked.
3. For each non-DQ prospect, compute `tier` from `icp_score`:
   - `icp_score >= 70` → `High`
   - `40 <= icp_score <= 69` → `Med`
   - `1 <= icp_score <= 39` → `Low`
   - `icp_score == 0` (shouldn't happen — DQ rows already filtered, but defend) → `Low`
4. Write `clients/<client>/results.csv` with header and one row per prospect. Columns in order:

   ```
   apollo_person_id,person_name,person_email,company_name,company_domain,icp_score,tier,channel,sent_at,replied,replied_on_step,bounced,meeting_booked,outcome_notes
   ```

   Pre-filled from enriched.csv: `apollo_person_id` (blank if not present in source), `person_name`, `person_email`, `company_name`, `company_domain`, `icp_score`, `tier`, `channel`.
   Blank: `sent_at`, `replied`, `replied_on_step`, `bounced`, `meeting_booked`, `outcome_notes`.

5. Append one line to `clients/<client>/assumptions.md` (create the file if missing):
   `capture-results: I assumed channel=<X> because <reason>.`
   Example reason: "the activate log mentions Gmail drafts" or "the user specified it when prompted".
6. Append to `clients/<client>/logs/capture-results.log.md`:
   - Date
   - Prospect count (included / excluded for DQ)
   - Channel chosen + derivation source (keyword-scan / user-prompt)
   - Any prior-critique flags that were applied
7. Print to the user:

   ```
   Skeleton written: clients/<client>/results.csv
   N prospects (X DQ'd and excluded)
   Channel: <X>

   Fill outcome columns (replied, replied_on_step, bounced, meeting_booked, outcome_notes, sent_at)
   directly in the CSV as replies land. Run /critique <client> when you're ready to analyze.
   ```

### Step 3B: Re-run — merge + summarize

1. Read existing `results.csv`. Preserve every row verbatim — never overwrite user edits.
2. Load `prospects.enriched.csv` (non-DQ only). Build a set of composite keys: `company_domain|person_name` for each enriched prospect.
3. Do the same for results.csv rows.
4. Compute the diff: enriched keys NOT in results. If any, ask the user: "M new prospects in prospects.enriched.csv are missing from results.csv. Append them? (yes/no)". If yes, append rows with the same pre-filled / blank pattern as first-run mode (reusing the channel from existing results rows — they'll all share the same channel value).
5. Compute summary stats:
   - Total rows in results.csv
   - Rows with `replied` non-null (i.e., user has recorded either true or false)
   - Rows with `replied=true` (reply count)
   - File mtime of `results.csv` → days since last edit
6. Append to `logs/capture-results.log.md` (date, re-run mode, rows added).
7. Print:

   ```
   results.csv for <client>: <total> rows
   Outcomes filled: N/<total> (<pct>%)
   Replies: K
   Last edit: D days ago

   [if rows were appended: Appended M new prospects.]
   [if fill < 50% AND mtime < 7 days: Looks active — keep filling.]
   [if fill < 50% AND mtime > 7 days: Looks stale — worth a check.]

   When ready, run /critique <client>.
   ```

### Step 4: No review gate

Silent tier. Skill ends after the print in Step 3A.7 or Step 3B.7. Don't prompt the user to "approve" anything — the file is theirs to edit directly.

## Output files

- `clients/<client>/results.csv` — the skeleton (created or appended).
- `clients/<client>/assumptions.md` — one line appended per run.
- `clients/<client>/logs/capture-results.log.md` — one entry appended per run.

## What NOT to do

- Don't overwrite existing `results.csv` rows. If the user has filled any outcome, it's sacred.
- Don't call Apollo, Clay, Instantly, Smartlead, or HeyReach MCPs. v1 is fully manual.
- Don't interactively ask about per-prospect outcomes ("did Sarah reply?"). That's what the CSV is for — the user edits in their editor.
- Don't compute reply rates or any analysis here — that's `/critique`'s job.
- Don't suggest skill improvements or critique anything. Capture ≠ evaluate.

## When to run

- **Right after /activate** — to get the skeleton early, so the user can fill as replies come in.
- **Periodically during the campaign** — re-run to pick up newly added prospects or refresh the stats summary.
- **Before /critique** — if results.csv doesn't exist yet, `/critique` will still work but skip the outcome section. Running capture-results first means you get the full signal.
