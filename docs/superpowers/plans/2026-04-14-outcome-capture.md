# Outcome Capture Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a `results.csv` outcome-capture layer to gtm-engine so `/critique` can evaluate whether campaigns work, not just how Claude executed them.

**Architecture:** One new skill (`/capture-results`) writes a pre-filled `results.csv` skeleton from `prospects.enriched.csv`; user fills outcome columns manually in editor; `/critique` reads `results.csv` when present and computes reply-rate-by-tier and replies-by-step; `/campaign-status` surfaces fill-state as a flag. Manual-fill v1 — no MCP auto-pull. Backward compatible (every skill behaves identically when `results.csv` is absent).

**Tech Stack:** Markdown SKILL.md files (no code, no tests, no runtime). The "test suite" is a smoke-test against the existing `clients/theline/` folder. Implementation = prompt-editing.

**Reference spec:** `docs/superpowers/specs/2026-04-14-outcome-capture-design.md`

---

## Context for the implementing engineer

This repo has no build, no tests, no runtime. Skills are `SKILL.md` files in `.claude/skills/<name>/` — they're prompts Claude reads at skill-invocation time. "Implementation" = writing or editing these Markdown files. "Testing" = invoking the skill in Claude Code against an existing client folder and inspecting the output files.

Read first:
- `CLAUDE.md` (project root) — skill contract, file schemas, gate tiering.
- `.claude/skills/critique/SKILL.md` — existing 8-step-contract skill you'll extend.
- `.claude/skills/campaign-status/SKILL.md` — read-only skill you'll extend.
- `.claude/skills/activate/SKILL.md` — for understanding channel semantics (not edited).
- `clients/theline/prospects.enriched.csv` — actual data shape for the smoke test.
- `clients/theline/logs/activate.log.md` — actual activate log shape.

Two implementation notes that differ from the spec:

1. **Join key fallback.** `CLAUDE.md`'s schema mandates `apollo_person_id` in `prospects.enriched.csv`, but `clients/theline/prospects.enriched.csv` pre-dates that and lacks it. In `results.csv`, `apollo_person_id` is a column but may be blank for legacy clients. The *actual* join key used for merging on re-run is composite: `company_domain + person_name` (both always populated). This lets the skill work on both legacy and new clients.

2. **Channel parsing.** The spec says "parse the Channel: line" from activate log. Real logs (see `theline`) use free-form markdown like `**Mode:** Attio (CRM push) + Gmail (unsent drafts of email 1)`. Use a keyword scan over the log body: `heyreach`/`linkedin` → `linkedin`; `smartlead`/`instantly` → `email`; `gmail` → `gmail`. If zero or multiple channel keywords match, prompt the user once.

---

## File Structure

- **Create:** `.claude/skills/capture-results/SKILL.md` (new skill)
- **Modify:** `.claude/skills/critique/SKILL.md` (add Step 2.5 + outcome section in output)
- **Modify:** `.claude/skills/campaign-status/SKILL.md` (add stage row, quick stat, flags, next-step mapping)
- **Modify:** `CLAUDE.md` (schema, pipeline, Instantly mention)

---

## Task 1: Create `/capture-results` skill

**Files:**
- Create: `.claude/skills/capture-results/SKILL.md`

- [ ] **Step 1: Write the skill file**

Write `.claude/skills/capture-results/SKILL.md` with the exact content below:

````markdown
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
````

- [ ] **Step 2: Smoke-test mentally**

Re-read the file. Verify:
- All 8 contract steps from CLAUDE.md are present (load context, prior-run signal, tools, output file, assumptions, review gate [acknowledged as "silent tier, none"], log, confirm next).
- Pipe-delimited composite key `company_domain|person_name` is used for re-run matching, not `apollo_person_id`.
- The keyword-scan for channel explicitly handles the LinkedIn URL false positive (word-boundary).
- No steps require MCP calls or external writes.

- [ ] **Step 3: Commit**

```bash
git add .claude/skills/capture-results/SKILL.md
git commit -m "feat: add /capture-results skill for manual outcome recording"
```

---

## Task 2: Extend `/critique` with outcome signal

**Files:**
- Modify: `.claude/skills/critique/SKILL.md`

- [ ] **Step 1: Insert Step 2.5 between existing Step 2 and Step 3**

In `.claude/skills/critique/SKILL.md`, find the section `### Step 3: Extract skill-edit suggestions` and insert a new section immediately before it:

````markdown
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

**Output both findings to the user in the terminal** (so they see it before filling out Step 3 skill-edit suggestions). Also save them for the written critique file in Step 4.
````

- [ ] **Step 2: Add `## Outcome signal` section to the critique file template in Step 4**

In the same file, find the template block under `### Step 4: Write the critique file` (the markdown block containing `# Critique — <client>`). After the `## Per-step ratings` table and before `## Skill-edit candidates`, insert a new section:

```markdown
## Outcome signal

<either the two findings from Step 2.5 verbatim, OR the line "No results.csv yet — run /capture-results when outcomes start landing.">
```

The full template block after the edit should read:

````markdown
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
````

Note the `capture-results` row added to the per-step ratings table — so critique also rates whether this skill did its job (channel-guess quality, join-key robustness, etc.).

- [ ] **Step 3: Extend Step 5 cross-client pattern detection**

In the same file, find `### Step 5: Surface patterns (if multiple clients exist)`. After the existing logic for step ratings ≤2 across 2+ clients, add a new pattern check:

```markdown
Additionally, if ≥2 clients have `results.csv` files where the same step number shows 0 replies across ≥10 sends (per Finding 2 logic), flag it as a cross-client template weakness:

```
Cross-client template weakness: step 3 got 0 replies across 2 clients (24 total sends).
This step's template is a candidate for rewrite — the weakness is not client-specific.
```

Don't propose the rewrite here — surface only.
```

- [ ] **Step 4: Update the "Step 1: Load context" list**

In the same file, find `### Step 1: Load context` and update the file list to include `results.csv` as an optional input:

```markdown
Read the artifacts that exist in `clients/<client-name>/`:
- `brief.md` (from research-client)
- `icp.json` (from icp-define)
- `prospects.csv` (from prospect)
- `prospects.enriched.csv` (from enrich-and-score)
- `sequences/` (from draft-sequences)
- `logs/activate.log.md` (from activate)
- `results.csv` (from capture-results, optional)
```

- [ ] **Step 5: Update the top-of-file description/intro if needed**

Check the top of `.claude/skills/critique/SKILL.md`. The skill's role now includes outcome-signal analysis. Update the intro paragraph (the one beginning "This skill is the **eval loop primitive**...") to acknowledge the two layers:

Replace:
> Each run of the pipeline produces outputs of variable quality. Without a structured critique, "the copy was so-so" stays a vibe — not actionable signal. This skill turns that vibe into a one-page artifact the user (or Claude) can diff across clients to spot recurring weak spots.

With:
> Each run produces two kinds of signal: (1) how well Claude executed each step (the per-step 1-5 rating), and (2) whether the campaign actually worked (reply rate by tier, template performance). Without a structured critique, both stay vibes — not actionable. This skill turns both into a one-page artifact the user (or Claude) can diff across clients to spot recurring weak spots. The outcome signal (2) is only available when `/capture-results` has produced a `results.csv`; when absent, the skill still functions and captures (1) normally.

- [ ] **Step 6: Commit**

```bash
git add .claude/skills/critique/SKILL.md
git commit -m "feat(critique): add outcome-signal analysis from results.csv"
```

---

## Task 3: Extend `/campaign-status` with results-tracking awareness

**Files:**
- Modify: `.claude/skills/campaign-status/SKILL.md`

- [ ] **Step 1: Add new stage row**

In `.claude/skills/campaign-status/SKILL.md`, find the stage table under `### Step 2: Derive state from the file system`. Add a new row below the existing "activated" row:

```markdown
| Files present | Stage |
|---|---|
| only folder + `logs/init.log.md` | initialized |
| + `brief.md` | research done |
| + `icp.json` | ICP defined |
| + `prospects.csv` (or `prospects.seed.csv`) | prospected |
| + `prospects.enriched.csv` | enriched + scored |
| + `sequences/*.md` | sequences drafted |
| + `logs/activate.log.md` with non-placeholder content | activated |
| + `results.csv` | activated + tracking outcomes |
| + `critique.md` with mtime > `results.csv` mtime | critiqued |
```

- [ ] **Step 2: Add results.csv quick stat logic**

In the same file, find `### Step 3: Pull quick stats per client` and append to the bullet list:

```markdown
- `results.csv` if present — count total rows, count rows where `replied` is non-null (outcomes_filled), count rows where `replied=true` (reply_count), note mtime as "last outcome edit: <N> days ago".
```

- [ ] **Step 3: Add flags for stale / underfilled results**

In the same file, find the single-client format block in `### Step 4: Report` (the code fence starting `Client: <client-name>`). Below the "Sequences drafted" line, add:

```markdown
- Outcomes: <outcomes_filled>/<total> filled (<pct>%), <reply_count> replies, last edit <N> days ago
```

(Only when `results.csv` exists — skip the line otherwise.)

Then in the "Flags" bullets handling (implicit today — currently described only as "if any logs flag errors or placeholders, surface them as 'Flags' bullets"), add explicit rules. Add a new sub-section immediately after Step 4:

```markdown
### Step 4a: Compute flags

Compute the following flags for each client and surface them as bullets under "Flags:" in the single-client report (or as inline notes in the multi-client format). If none apply, omit the "Flags:" section.

**Outcome-capture flags** (compute only when `results.csv` exists):

- If stage is `activated + tracking outcomes` AND `outcomes_filled / total < 0.5` AND `days since activate log mtime >= 7`:
  `results.csv <pct>% filled (<filled>/<total>), <D> days since activate — outcomes may be getting missed. Run /capture-results to re-check or fill in the CSV.`

- If `days since results.csv mtime > 14` AND `outcomes_filled / total < 1.0`:
  `results.csv stale (<D> days since last edit, <pct>% filled), run /capture-results to refresh or mark remaining prospects.`

**Activated-but-no-results flag** (compute when `results.csv` is absent):

- If stage is `activated` AND `days since activate log mtime >= 3`:
  `activated <D> days ago but no results.csv yet — run /capture-results <client> to start tracking outcomes.`

Other flags may exist (errors from logs, RESUME.md notes, etc.) — preserve existing behavior for those.
```

- [ ] **Step 4: Update next-step mapping**

In the same file, find the table under `## Next-step mapping`. Replace the "activated" row and add two new rows:

```markdown
| Stage | Next skill |
|---|---|
| initialized | `/research-client <client>` |
| research done | `/icp-define <client>` |
| ICP defined | `/prospect <client>` |
| prospected | `/enrich-and-score <client>` |
| enriched + scored | `/draft-sequences <client>` |
| sequences drafted | `/activate <client>` |
| activated | `/capture-results <client>` |
| activated + tracking outcomes | `/critique <client>` |
| critiqued | no next step — campaign live, check sequencer UI and keep outcomes updated |
```

- [ ] **Step 5: Commit**

```bash
git add .claude/skills/campaign-status/SKILL.md
git commit -m "feat(campaign-status): surface results.csv fill state and staleness"
```

---

## Task 4: Update CLAUDE.md

**Files:**
- Modify: `CLAUDE.md`

- [ ] **Step 1: Update the pipeline order in the Skills section**

Find the "Skills" section near the top. Replace:

```markdown
- Pipeline: `campaign-init` → `research-client` → `icp-define` → `prospect` → `enrich-and-score` → `draft-sequences` → `activate`
```

With:

```markdown
- Pipeline: `campaign-init` → `research-client` → `icp-define` → `prospect` → `enrich-and-score` → `draft-sequences` → `activate` → `capture-results`
```

Also update the "Ten skills" count if it's mentioned numerically — it becomes eleven. (Check the line "Live at `.claude/skills/<skill-name>/SKILL.md`. Ten skills:" — change to "Eleven skills:".)

- [ ] **Step 2: Add `results.csv` to the File schemas section**

Find the `## File schemas (single source of truth)` section. After the `### sequences/` subsection and before `### Canonical sequence tokens`, insert a new subsection:

````markdown
### `results.csv` (output of `capture-results`, user-filled)

One row per non-disqualified prospect from `prospects.enriched.csv`. Pre-filled columns come from the skill; outcome columns are blank and the user fills them as outcomes land.

```
apollo_person_id, person_name, person_email, company_name, company_domain,
icp_score, tier, channel,
sent_at, replied, replied_on_step, bounced, meeting_booked, outcome_notes
```

- `tier`: derived from `icp_score` at skeleton-creation time. `High` (≥70), `Med` (40-69), `Low` (1-39). Snapshot — does not update if the prospect is re-scored later.
- `channel`: `linkedin` / `email` / `gmail` — derived from the `activate.log.md` body via keyword scan (`heyreach`/`linkedin` → linkedin, `smartlead`/`instantly` → email, `gmail` → gmail). If ambiguous, the skill prompts the user.
- `replied`: `true` / `false` / blank. Blank means no outcome recorded yet; `false` means "sent, no reply". Both count as "outcome_filled" for fill-rate calculations.
- `replied_on_step`: integer step number (1..N) if `replied=true`, else blank.
- `apollo_person_id`: may be blank for clients whose `prospects.enriched.csv` predates that column. The skill uses composite key `company_domain + person_name` for merge-on-rerun.
- Join key with `prospects.enriched.csv`: `company_domain + person_name` (robust to `apollo_person_id` absence).

`/critique` reads this file when present and adds an "Outcome signal" section with reply-rate-by-tier and replies-by-step findings. When absent, `/critique` behaves as before.
````

- [ ] **Step 3: Add Instantly alongside Smartlead in MCP availability**

Find the `## MCP availability is conditional` section. Find the Smartlead line:

```markdown
- **Smartlead** — email channel for `activate` via the LeadMagic MCP (`npx smartlead-mcp-by-leadmagic install`, needs `SMARTLEAD_API_KEY`). Paid product — skip if not worth the cost yet.
```

Add a new line directly after it:

```markdown
- **Instantly** — email channel for `activate` via the Instantly v2 API MCP. Current preferred email sender for prod runs. Paused-first discipline applies.
```

- [ ] **Step 4: Commit**

```bash
git add CLAUDE.md
git commit -m "docs: add results.csv schema, capture-results to pipeline, Instantly MCP"
```

---

## Task 5: Smoke-test on existing client data

**Files:**
- Read-only: `clients/theline/prospects.enriched.csv`, `clients/theline/logs/activate.log.md`
- Will be created / modified by the skills under test: `clients/theline/results.csv`, `clients/theline/assumptions.md`, `clients/theline/logs/capture-results.log.md`, `clients/theline/critique.md`

This task is manual — the implementing engineer (or the user) runs the skills in Claude Code and inspects output. No automated assertions are possible because the "implementation" is prompt files.

- [ ] **Step 1: Baseline — verify backward compat of `/critique`**

Before any new files exist, run `/critique theline` in Claude Code. Confirm:
- It asks for per-step ratings as today.
- The `Outcome signal` section in the written `critique.md` contains the fallback line: `No results.csv yet — run /capture-results when outcomes start landing.`
- No errors referencing a missing `results.csv`.

If the critique step-row table now includes `capture-results` (per Task 2 Step 2), it's expected to show as empty/skipped since the skill hasn't been run yet.

- [ ] **Step 2: First-run `/capture-results`**

Run `/capture-results theline`. Confirm:
- It loads `prospects.enriched.csv` and filters DQ rows (theline's current file has 0 DQ rows — all should be kept).
- It detects channel as `gmail` from the activate log (log body contains "Gmail drafts").
- It writes `clients/theline/results.csv` with the 13-column header and one row per non-DQ prospect.
- Pre-filled columns are populated from enriched.csv; `apollo_person_id` is blank (theline's enriched.csv lacks it).
- Outcome columns (`sent_at`, `replied`, `replied_on_step`, `bounced`, `meeting_booked`, `outcome_notes`) are all blank.
- `clients/theline/assumptions.md` gains one line: `capture-results: I assumed channel=gmail because ...`
- `clients/theline/logs/capture-results.log.md` is created with the run entry.
- The user-facing print names the file path and next step (`/critique theline`).

- [ ] **Step 3: Re-run `/capture-results` with no edits**

Re-run `/capture-results theline` without editing `results.csv`. Confirm:
- It does NOT overwrite the existing file.
- It prints the summary: 4 rows, 0/4 filled, 0 replies, last edit ~0 days ago (just created).
- No new prospects appended (nothing new in enriched.csv).
- A new log entry is appended.

- [ ] **Step 4: Manually fill 3+ outcome rows**

Edit `clients/theline/results.csv` directly. Fill in at least 3 rows' `replied` column — mix of `true` (with `replied_on_step` populated) and `false`. Example:

| person_name | ... | replied | replied_on_step |
|---|---|---|---|
| Alexey Molchanov | ... | true | 1 |
| Kseniia Iurtaeva | ... | false | |
| Dimitrios Pantazis | ... | false | |
| Athanasios Papaterpos | ... | (blank) | |

Save.

- [ ] **Step 5: Run `/critique theline` with outcome data present**

Run `/critique theline`. Confirm:
- It now shows a "Outcome signal for this run:" block in the terminal during Step 2.5.
- Reply rate by tier displays the non-empty tier rows (theline has all High-tier prospects given scores 95/80/60/60, so expect High: 1/1 and Med: 0/2 — or adjust based on your filled values).
- Small-sample note appears (n < 10).
- Written `critique.md` includes the "Outcome signal" section with the two findings verbatim.
- Skill-edit candidates section still works.

- [ ] **Step 6: Run `/campaign-status theline`**

Run `/campaign-status theline`. Confirm:
- Stage is reported as `critiqued` (since `critique.md` now exists and is newer than `results.csv`).
- The Outcomes line is present: `Outcomes: 3/4 filled (75%), 1 replies, last edit X days ago`.
- No stale flag (edit is recent).
- Next-step mapping points to "no next step" since stage is `critiqued`.

- [ ] **Step 7: Test the stale flag**

Manually touch `clients/theline/results.csv` to a date 20 days ago if feasible (or just visually inspect the logic by reading the SKILL.md to confirm the branch). If the repo is live, you can simulate by running `/campaign-status theline` and checking the flag wording is correct when conditions hold. This step is a lightweight sanity check — don't actually backdate files in git.

- [ ] **Step 8: Commit the spec-derived smoke-test notes (optional)**

If the smoke test reveals any deviations, document them in a `## Smoke-test notes` section at the bottom of the plan file (this file) and commit. Otherwise, no commit needed for this task.

---

## Self-Review

After writing the plan, cross-check against `docs/superpowers/specs/2026-04-14-outcome-capture-design.md`:

**Spec coverage:**
- [x] New `/capture-results` skill → Task 1
- [x] `/critique` Step 2.5 extension with both findings → Task 2 Steps 1-2
- [x] Cross-client template weakness flag → Task 2 Step 3
- [x] `/critique` Step 1 updated to include results.csv → Task 2 Step 4
- [x] `/critique` intro paragraph reflects two signal layers → Task 2 Step 5
- [x] `/campaign-status` stage row + quick stat + flags + next-step → Task 3 Steps 1-4
- [x] CLAUDE.md pipeline update, schema, Instantly → Task 4
- [x] Backward compat verified (no results.csv case) → Task 5 Step 1
- [x] Smoke test on real client data → Task 5 Steps 2-6

**Placeholder scan:** No "TBD", "TODO", or "similar to task N" found. All code/prompt steps include the actual content. Channel-scan false-positive on `linkedin.com` URLs is explicitly handled.

**Type / name consistency:**
- Column order consistent between Task 1 (skill file) and Task 4 (CLAUDE.md schema): `apollo_person_id, person_name, person_email, company_name, company_domain, icp_score, tier, channel, sent_at, replied, replied_on_step, bounced, meeting_booked, outcome_notes`. ✓ (Note: added `company_domain` — not in original spec column list but needed for the join-key fallback; spec's 13 columns become 14 with company_domain. Worth updating the spec or calling this out.)
- Tier bands consistent: High ≥70, Med 40-69, Low 1-39 across skill, schema, and `/campaign-status`. ✓
- "Silent tier" designation matches CLAUDE.md gate tiering section. ✓

**Deviation from spec flagged:** Column count is 14, not 13 — added `company_domain` for the composite join key. Update the spec file inline to match (or accept as an implementation discovery and note in the spec's open-questions section).
