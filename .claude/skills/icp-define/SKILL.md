---
name: icp-define
description: Define the Ideal Customer Profile for a client based on the research brief. Interactive process with the user. Outputs a structured ICP file. Use after /research-client.
---

# /icp-define - Define the Client's ICP

When the user runs `/icp-define <client-name>`:

## What to do

### Step 1: Load context

Read:
- `clients/<client-name>/brief.md` (the research from /research-client)
- `clients/<client-name>/icp.json` (the empty template from /campaign-init)

### Step 2: Draft the ICP from the brief

Based on the research brief, propose a draft ICP. Present it to the user as a conversation, not a data dump:

```
Based on the research, here's my proposed ICP for <client-name>:

**Target industries:** <your proposal, e.g. "SaaS, fintech, e-commerce - based on their case studies">
**Company size:** <your proposal, e.g. "50-500 employees - their pricing suggests mid-market">
**Regions:** <your proposal, e.g. "US and EU - their site is in English, they mention GDPR compliance">
**Target titles:** <your proposal, e.g. "VP of Engineering, CTO, Head of Platform - these are the buyers in their case studies">
**Buying signals:** <your proposal, e.g. "Recently raised Series A/B, hiring backend engineers, using AWS">
**Disqualifiers:** <your proposal, e.g. "Under 20 employees (too small), agencies (not their market), already using competitor X">

**Offer summary:** <one sentence describing what the client sells>
**Value proposition:** <one sentence describing why someone would buy>

What would you change? I can adjust any of these.
```

### Step 3: Iterate with the user

The user may:
- Accept as-is
- Modify specific fields
- Add context you missed ("actually their sweet spot is companies migrating from X to Y")
- Disagree with your assessment

Apply their changes. If they give vague input, ask clarifying questions. One question at a time, not a barrage.

### Step 4: Write the ICP

Once confirmed, write `clients/<client-name>/icp.json`:

```json
{
  "client_domain": "acme.com",
  "client_name": "acme",
  "target_industries": ["SaaS", "fintech", "e-commerce"],
  "target_company_sizes": ["51-200", "201-500"],
  "target_regions": ["United States", "United Kingdom", "Germany"],
  "target_titles": ["VP of Engineering", "CTO", "Head of Platform"],
  "target_signals": [
    "Recently raised Series A or B",
    "Hiring backend engineers",
    "Using AWS or GCP",
    "Posted job for DevOps/Platform role"
  ],
  "disqualifiers": [
    "Under 20 employees",
    "Agencies or consultancies",
    "Already using <competitor>"
  ],
  "offer_summary": "One sentence about what they sell",
  "value_proposition": "One sentence about why someone buys"
}
```

### Step 4.5: Draft messaging angles

Before confirming, propose 3 messaging angles for this ICP. Angles are distinct framings that different slices of the ICP respond to — not restatements of the value prop in different words.

Present it as a short proposal, not a JSON dump:

```
Three messaging angles for <client-name>:

1. **Cost reduction** — lands with <which slice>, pain shows up as <symptom>, proof shape: <what kind of outcome>
2. **Speed to ship** — lands with <which slice>, pain shows up as <symptom>, proof shape: <what kind of outcome>
3. **Risk mitigation** — lands with <which slice>, pain shows up as <symptom>, proof shape: <what kind of outcome>

Each angle becomes a distinct sequence template. /draft-sequences picks the right angle per prospect.

Keep these three? Swap any for something more specific to <client-name>?
(Common alternates: competitive_displacement, hiring_freeze_efficiency, compliance_deadline, category_creation)
```

The user may:
- Accept the three defaults.
- Replace one or more with client-specific angles the brief makes obvious.
- Ask for a different count (minimum 3, maximum 5).

**Calibration rule:** If two proposed angles would resonate with the same slice of the ICP (e.g., cost_reduction and speed_to_ship both aimed at engineering leaders under margin pressure), collapse them or replace one. Overlapping angles defeat the purpose — the drafter needs distinct slices to pick between.

Once confirmed, append `messaging_angles` to the `icp.json` file per the schema in `CLAUDE.md` → File schemas → `icp.json`. Each angle has `name` (snake_case), `hypothesis` (one sentence), `pain_hooks` (2-3 symptoms), `proof_shape` (one phrase).

Record the angle set in `clients/<client-name>/assumptions.md` as a single line:
```
icp-define: angles = [cost_reduction, speed_to_ship, risk_mitigation] — defaults accepted.
```
or, if customized:
```
icp-define: angles = [competitive_displacement, compliance_deadline, hiring_freeze_efficiency] — brief called out SOC2 renewal cycle and Q2 hiring freeze.
```

### Step 5: Confirm

```
ICP saved to clients/<client-name>/icp.json
Next step: run /prospect <client-name> to find companies matching this ICP.
```

### Step 6: Log

Write to `clients/<client-name>/logs/icp.log.md`:

```markdown
# ICP Definition Log
- **Date:** <timestamp>
- **Based on brief:** clients/<client-name>/brief.md
- **ICP written to:** clients/<client-name>/icp.json
- **Industries:** <list>
- **Sizes:** <list>
- **Titles:** <list>
- **Angles:** <names only, e.g. cost_reduction, speed_to_ship, risk_mitigation>
- **Status:** User approved
```

## No external tools needed

This skill uses the brief (already gathered) and user input. No MCP calls.

## Common mistakes to avoid

- Don't make the ICP too broad. "All SaaS companies" is not an ICP. Be specific about what makes someone a GOOD fit, not just a possible fit.
- Don't list 15 target titles. Pick the 3-5 that are actually the buyers or champions.
- Signals matter more than firmographics. "Recently raised Series B" is more actionable than "51-200 employees."
- Disqualifiers save time. Be explicit about who NOT to target.
- The offer summary and value proposition should use the client's own language from their website, not generic marketing speak.
- Don't let two angles target the same slice of the ICP. They need to pick different slices, or they collapse into one angle.
