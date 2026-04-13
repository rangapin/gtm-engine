---
name: draft-sequences
description: Write personalized email sequences for each prospect based on their enrichment data and the client's positioning. Outputs ready-to-send sequences. Use after /enrich-and-score.
---

# /draft-sequences - Write Outbound Email Sequences

When the user runs `/draft-sequences <client-name>`:

## What to do

### Step 1: Load context

Read:
- `clients/<client-name>/prospects.enriched.csv` (enriched + scored prospects)
- `clients/<client-name>/brief.md` (client research)
- `clients/<client-name>/icp.json` (ICP definition)

Filter to non-disqualified prospects with `icp_score >= 40` (or whatever threshold the user prefers). Ask:

```
I'll draft sequences for <N> prospects scoring 40+.
Want to adjust the score threshold, or should I proceed?
```

### Step 2: Define the sequence structure

Default structure is a 3-email sequence. Present it to the user:

```
Default sequence structure:
- Email 1: Cold open. Personalized hook based on the prospect's signal/context. Introduce the value prop. Soft CTA (reply, not book a call).
- Email 2: Follow-up (3 days later). Different angle. Social proof or case study if available. Same soft CTA.
- Email 3: Breakup (5 days later). Short, direct. "Is this relevant?" Last chance.

Want to adjust the structure, timing, or approach?
```

Let the user modify the sequence structure if they want. Some clients prefer 4-5 emails, some want LinkedIn touches mixed in, some want a harder CTA.

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

### Step 4: Write the sequences

For each prospect (or group of similar prospects), write the full sequence.

**Personalization rules:**
- Email 1 MUST reference something specific about the prospect's company or situation. Not "I see you're in fintech" (generic). Instead, reference a specific signal: "Saw you just raised your Series B" or "Noticed you're hiring a Head of Platform."
- Email 2 should take a different angle. If Email 1 was about a signal, Email 2 might be about a pain point or a peer's experience.
- Email 3 is short. 2-3 sentences max.

**Formatting rules:**
- No HTML. Plain text only.
- Short paragraphs (1-2 sentences each).
- No more than 100 words per email.
- Subject lines: 3-6 words, lowercase, no punctuation tricks.
- No "Dear <name>" or "I hope this finds you well." Ever.
- Sign off with the sender's first name only.

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

If multiple prospects share the same signal or profile (e.g., all are CTOs at Series B fintech companies), you can write one template for the group and customize the opening line per person. Flag this to the user:

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
- **Output:** clients/<client-name>/sequences/
- **Status:** User reviewing
```

## No external tools needed

This skill is pure writing. No MCP calls. Claude does the copywriting based on the enrichment data and brief.

## Writing quality standards

**Good first line examples:**
- "Noticed Acme just brought on a Head of Data - usually means the data stack is getting a rethink."
- "Three of your competitors switched to <product category> in Q1. Curious if that's on your radar."
- "Your Series B announcement caught my eye - we help companies at that stage solve <specific problem>."

**Bad first line examples (never write these):**
- "I hope this email finds you well."
- "I'm reaching out because..."
- "I noticed your company is doing great things in the <industry> space."
- "As a fellow <title>, I understand the challenges of..."
- "I'd love to pick your brain about..."

**The test:** Would a real human send this email to someone they actually want a response from? If it sounds like a mass email, rewrite it.
