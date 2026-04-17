---
name: draft-sequences
description: Write personalized email sequences for each prospect based on their enrichment data and the client's positioning. Outputs ready-to-send sequences. Use after /enrich-and-score.
---

# /draft-sequences - Write Outbound Email Sequences

When the user runs `/draft-sequences <client-name>`:

## What to do

### Step 1: Load context

Read:
- `.claude/skills/draft-sequences/COPY_RULES.md` — **the cold-email craft rules. Load this every time; it's the drafting spec.**
- `clients/<client-name>/prospects.enriched.csv` (enriched + scored prospects)
- `clients/<client-name>/brief.md` (client research)
- `clients/<client-name>/icp.json` (ICP definition + `messaging_angles` used to pick a per-prospect angle)

If `messaging_angles` is absent from `icp.json` (legacy ICP predating the angles layer), fall back to the pre-angles flow: write sequences without an angle header and note `angles: absent` in the drafting log. Don't halt — legacy clients still work.

`COPY_RULES_SOURCES.md` sits alongside for traceability — do NOT load it at drafting time (too long). Only read it if the user wants to debate a specific rule against its evidence.

Filter to non-disqualified prospects with `icp_score >= 40` (or whatever threshold the user prefers). Ask:

```
I'll draft sequences for <N> prospects scoring 40+.
Want to adjust the score threshold, or should I proceed?
```

### Step 2: Define the sequence structure

Default structure is a 3–4 email sequence on the **3-7-7 cadence** (from `COPY_RULES.md`). Present it to the user:

```
Default sequence structure (3-7-7 cadence):
- Email 1 (Day 0): Cold open. Timeline-based hook from the prospect's signal. Permission-ask CTA.
- Email 2 (Day 3): Angle-shift follow-up. Different proof point or framing of the same problem. Do not repeat Email 1's pitch.
- Email 3 (Day 10): Qualifying question that makes "not right now" an easy reply. Or insight-first with no pitch.
- Email 4 (Day 17, optional): Breakup. Zero pitch, zero guilt. Graceful exit.

Want to adjust structure, timing, or drop the breakup?
```

Let the user modify the sequence structure if they want. Some clients prefer 4-5 emails, some want LinkedIn touches mixed in, some want a harder CTA. **Hard rule from research: do not exceed 4 emails.** Past #4, spam complaints triple and reply rates drop 55%.

### Step 3: Calibrate the voice

Before writing, establish the writing voice. Ask:

```
How should these emails sound?

The client's own tone (from their website) is: <describe their tone based on the brief>

Should the emails:
- Match their brand voice?
- Sound more casual/personal?
- Be formal and professional?
- Something else?

Any specific phrases to use or avoid?
```

### Step 3.5: Pick an angle per prospect

For each prospect (or group of similar prospects), pick **one** angle from `icp.json.messaging_angles`. The picker is interpretive, not formulaic — read the prospect's title + signals + score_reasoning and choose the angle whose `hypothesis` best matches.

Heuristics (not rules — override when the prospect clearly fits a different slice):
- Finance titles (CFO, VP Finance, Controller) → usually `cost_reduction`.
- Engineering/platform leadership (CTO, VP Eng, Head of Platform) → usually `speed_to_ship` unless signals say otherwise.
- Security, compliance, or risk titles → `risk_mitigation`.
- Strong signal of a named competitor being ripped out or a losing incumbent → `competitive_displacement` if it's in the angle set.
- Recent hiring freeze, layoff, or "do more with less" signal → `hiring_freeze_efficiency` if in the set.

When two angles both seem to fit: pick the one whose `proof_shape` you can actually satisfy for this prospect from the enrichment + signals data. No point picking `cost_reduction` if you have no dollar-denominated proof to cite.

When no angle fits: flag to the user rather than forcing a bad match. Example: `"<prospect> is a design lead — none of the 3 angles (cost_reduction, speed_to_ship, risk_mitigation) fit. Skip this prospect, or want me to propose a new angle for the ICP?"`

Record the picked angle in two places:
1. In the sequence file header (see Step 6, the `Angle:` field).
2. In the drafting log (Step 8, per-prospect or per-segment).

### Step 4: Write the sequences

For each prospect (or group of similar prospects), write the full sequence **against the rules in `COPY_RULES.md`**. That file is the drafting spec — don't try to remember the rules from memory, read them.

**The short version (full text in `COPY_RULES.md`):**
- Target 50–80 words per email. First-touch 25–50 if possible. Hard ceiling 100.
- Subject lines lowercase, ≤5 words, prospect-specific not category-generic. Never fake `Re:`.
- Plain text only — no HTML, no bullets, no bold. 5th-grade reading level.
- Openers: timeline hook from the prospect's signal. Earn relevance in the first 7 words. No "I" as the first word.
- CTA: ask permission, don't assume interest. One CTA per email. No calendar links in Email 1. The body must stand on its own with the CTA removed — if it doesn't, it's a pitch, not a note.
- Angle-shift every follow-up. Do not repeat Email 1's pitch.
- Persona-calibrate: C-suite under 50 words + outcome-led; VPs connect strategy to mechanism; ICs/technical use operational language; finance wants numbers, no adjectives.
- Sign-off: `<first_name>\n<role> @ <domain>`. Not a bare first name.
- Forbidden words and phrases (AI tells — full list in `COPY_RULES.md`): impressed, innovative, streamline, leverage, delighted, cutting-edge, game-changing, revolutionary, "I hope this email finds you well", "I came across your profile", "quick question" (subject), "I'd love to pick your brain", **"Worth a look?" / "Worth exploring?" / "Worth [anything]?" as CTA**, **"Different angle:" as Email 2 opener**, **"Closing the loop" as breakup opener**, **"We can be live in [X]" speed-to-launch claims in Email 1**, **"Most [X] teams…" generalization openers**, **"[Product] is how [Company] does [Thing]" SaaS-pitch construction**.
- CTA variety: never reuse the same CTA construction across emails in a sequence, or across templates in the same client run. Vary form ("Relevant?" / "Useful to dig in?" / plain questions / no CTA at all on insight-only emails).
- Em-dash cap: ≤1 per email. Rewrite extras with periods or commas.
- Locale: currency matches prospect region (PLN/€ for PL, $ for US, £ for UK). Pick at `/icp-define` time and hold across every email in the client run.
- Proof-on-prospect: every email must bridge one proof element to this prospect's specific situation. Logo drops without connective tissue to the prospect don't count.

**Tension awareness.** Several rules in `COPY_RULES.md` conflict in edge cases — e.g., permission-ask CTAs ("not sure if this is relevant, but…") read weak to founders who prefer directness; humor lands for marketing personas but misfires for finance. When rules clash, resolve using the prospect's persona and the client's voice, and flag the tradeoff at the review gate rather than picking one rule silently.

**Personalization variables (canonical tokens):**

Use UPPER_SNAKE_CASE. The `/activate` skill maps these to the target platform's syntax (Smartlead, HeyReach, etc.) before pushing.

Per-prospect tokens (the sequencer fills these):
- `{{FIRST_NAME}}` - prospect's first name
- `{{LAST_NAME}}` - prospect's last name
- `{{COMPANY}}` - prospect's company name
- `{{SENDER_NAME}}` - the person sending (client's rep)

Per-prospect hook tokens (you fill these when writing; stored per-prospect in the sequence file):
- `{{HOOK_SHORT}}` - one-line opener referencing the prospect's specific signal
- `{{HOOK_EXPANDED}}` - 2-3 sentence version for email body
- `{{EMAIL_SUBJECT}}` - per-prospect subject line

The grouped-sequences pattern (see Step 5) uses segment template files for the reusable body, and per-prospect files that only define the hook values. See existing `clients/claybootcamp/sequences/` for the reference pattern.

The real personalization (the hook, the signal reference) must be written specifically per-prospect, not left generic. This is what makes it work.

### Step 5: Group similar prospects

If multiple prospects share the same signal or profile (e.g., all are CTOs at Series B fintech companies), you can write one template for the group and customize the opening line per person. Grouping by angle is a natural segmentation — one `_SEGMENT_<angle_name>_template.md` per angle often maps cleanly onto the prospect set. Flag this to the user:

```
I can write:
- Fully individual sequences for each prospect (highest quality, most work)
- Grouped sequences by profile type, with per-person opening lines (good balance)

Which approach?
```

### Step 6: Write output files

For each prospect (or group), write to `clients/<client-name>/sequences/`:

```
clients/<client-name>/sequences/
  <company-domain>_<person-name>.md     # individual (full sequence inline)
  OR (grouped pattern)
  _SEGMENT_<A|B|C>_template.md          # reusable body with {{...}} tokens
  <company-domain>_<person-name>.md     # per-prospect hook values only
```

Underscore-prefixed files (`_SEGMENT_*_template.md`) are grouped templates referenced by per-prospect hook files. The underscore sorts them to the top of the directory listing.

Each file format:

```markdown
# Sequence: <Person Name> at <Company>

**Prospect:** <name>, <title> at <company>
**Score:** <icp_score>
**Angle:** <angle_name> — <one-sentence why this angle for this prospect>
**Signal:** <what triggered the outreach>
**Email:** <email>

## Email 1 - Cold Open
**Subject:** <subject line>
**Send:** Day 1

<email body>

## Email 2 - Follow-up
**Subject:** Re: <original subject> OR <new subject>
**Send:** Day 4

<email body>

## Email 3 - Breakup
**Subject:** Re: <original subject> OR <new subject>
**Send:** Day 9

<email body>
```

### Step 6.5: Lint pass

Before showing drafts to the user, re-read every email against `COPY_RULES.md`. For each email, check:

1. **Length.** Word count ≤100, Email 1 ideally ≤50. If over, cut.
2. **Subject line.** Lowercase? ≤5 words? Prospect-specific (not generic category)? No fake `Re:`?
3. **Forbidden words and phrases.** Grep the body against the full Forbidden list in `COPY_RULES.md`. Minimum hit-list: impressed, innovative, streamline, leverage, delighted, cutting-edge, game-changing, revolutionary, "hope this email finds you well", "came across your profile", "quick question" (as subject), "pick your brain", **"worth a look", "worth exploring", "worth comparing" (as CTA), "different angle:", "a different angle:", "different framing:" (as opener), "closing the loop" (as breakup opener), "we can be live", "we can deploy" (speed-to-launch in Email 1), "most [x] teams", "[Product] is how"**. Any hit → rewrite that line.
4. **Opener.** First 7 words earn relevance? Timeline hook if a recent signal exists? No "I" as the literal first word of Email 1?
5. **CTA.** One CTA, permission-asking? No calendar link in Email 1?
6. **Angle-shift.** Email 2 opens a different angle from Email 1? Email 3 is the qualifying question or insight-first, not "just following up"?
7. **Signature.** `<first_name>\n<role> @ <domain>`, not a bare first name?
8. **Mobile test.** Would this fit on a phone screen without scrolling?
9. **Aloud test.** Read it in your head as if it arrived from a stranger. Does any sentence exist only to scaffold the pitch? Delete it.
10. **Angle alignment.** Does the email body actually express the chosen angle? A prospect tagged `cost_reduction` whose email talks about shipping velocity is mis-angled — either rewrite the body or change the angle tag, don't let them drift.
11. **Angle field present.** Does the sequence file header include `**Angle:** <name> — <why>`? If it's missing, add it before delivering the draft. This field is required for downstream critique and outcome correlation.
12. **Em-dash count.** Count em-dashes (`—`, not hyphens `-`) per email. Cap at 1 per email. Rewrite extras with periods or commas. Two+ em-dashes in a 50-word email is the single strongest AI-voice tell.
13. **CTA variety across the sequence.** List the CTA phrases across all emails for this prospect (or all templates in the client run). If any two match in construction ("Worth X?" + "Worth Y?", "Interested in X?" + "Interested in Y?"), rewrite one. No sequence should repeat its CTA form.
14. **Locale consistency.** Check currency symbols used in each email against the client's locale (derived from `icp.json.target_regions` — PL → PLN/€, US → $, UK → £). Mixed currencies across templates in the same run must be unified before delivery.
15. **Proof-on-prospect bridge.** For each email that names a customer logo (Siemens, Stripe, etc.), check the next sentence ties it to something specific about *this* prospect's situation — company, signal, or named constraint. A logo drop without a bridge ("We work with Siemens and BSH.") must be rewritten or cut.
16. **Feature-dump check.** For each email, check every sentence: does it describe what the *product does*, or what the *prospect can do that they couldn't do before*? Any feature/capability listing must be rewritten as a prospect-facing outcome. ✗ "Our platform features AI analytics and real-time dashboards." ✓ "Know which deals are closing this quarter without interrogating your reps Friday afternoon."
17. **Close-on-high-note check.** The last sentence of each email sets the memory the prospect replies from. It must leave them lighter, not pressured. Kill pressure-closers: "please let me know either way," "looking forward to your reply," "awaiting your response," "will you let me know?" Replace with a humble exit, a small useful nugget, or silence. A breakup that says "I'll survive if this isn't for you" outperforms one that says "last chance."
18. **Timeline signal freshness.** For any opener that references a specific event (funding, hiring, launch, leadership change), confirm the event date is ≤60 days old. If it's older, either find a fresher signal or drop the time-anchored phrasing ("recently," "this week," "just," "after your") — reaching for a stale signal reads worse than no signal.
19. **CTA-removal test.** Re-read each email with the CTA mentally deleted. If the remaining body still gives the prospect something of value — a specific observation, a data point, a useful framing — it passes. If the email collapses without the ask, it's a pitch wearing friendly clothes. Rewrite the body to stand on its own, or cut the email. This test is falsifiable in a way the "aloud test" (#9) isn't; run both.

Fix violations before the review gate. If a rule conflicts with the client's voice (e.g., permission-ask feels weak for this founder), leave it and flag the tradeoff to the user — don't pretend the rule is absolute. Log which rules you applied vs bent in the drafting log.

### Step 7: Review gate

This is the most important review gate. Present 2-3 sample sequences to the user:

```
Here are sample sequences for review:

---
[Show 2-3 full sequences]
---

Full sequences saved to clients/<client-name>/sequences/

Please review carefully:
- Does the tone match what the client wants?
- Are the hooks relevant and specific?
- Are the CTAs appropriate?
- Anything to change in structure, length, or approach?

I can revise any sequence before we push to the sequencer.
```

The user MUST review sequences before activation. This is where quality lives or dies.

### Step 8: Log

Write to `clients/<client-name>/logs/sequences.log.md`:

```markdown
# Sequence Drafting Log
- **Date:** <timestamp>
- **Prospects sequenced:** <N>
- **Sequence structure:** <N> emails, <timing>
- **Approach:** individual / grouped by <criteria>
- **Voice:** <description>
- **Rules applied:** (list key COPY_RULES.md rules that shaped these drafts, e.g. timeline-hook opener, permission-ask CTA, 3-7-7 cadence)
- **Rules bent (and why):** (list any rules deliberately broken for this client's voice or persona — e.g. "used direct CTA over permission-ask for founder audience who prefers directness")
- **Angles used:** (distribution — e.g. `cost_reduction: 8, speed_to_ship: 12, risk_mitigation: 4, unmatched: 1`, or `absent` for legacy ICPs without messaging_angles)
- **Unmatched prospects:** (list any where no angle fit and sequence was skipped or flagged)
- **Output:** clients/<client-name>/sequences/
- **Status:** User reviewing
```

## No external tools needed

This skill is pure writing. No MCP calls. Claude does the copywriting based on the enrichment data and brief.

## Writing quality standards

The authoritative rules live in `COPY_RULES.md`. Example pairs and research citations live in `COPY_RULES_SOURCES.md`.

**The test:** Would a real human send this email to someone they actually want a response from? If it sounds like a mass email, rewrite it.
