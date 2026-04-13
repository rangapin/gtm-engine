---
name: activate
description: Push approved prospects to Attio (CRM) and launch the outbound campaign via HeyReach (LinkedIn) or Smartlead (email). Final step before campaign goes live. Use after /draft-sequences and user approval.
---

# /activate - Push to CRM and Sequencer

When the user runs `/activate <client-name>`:

## What to do

### Step 1: Load context

Read:
- `clients/<client-name>/prospects.enriched.csv` (the scored prospect list)
- `clients/<client-name>/sequences/` (the approved email sequences)
- `clients/<client-name>/icp.json` (for reference)

### Step 2: Pre-flight check

Before pushing anything, run through this checklist with the user:

```
Pre-flight check for <client-name>:

Prospects: <N> contacts ready to push
Sequences: <N> email sequences drafted
Emails verified: <N>/<total> have email addresses

CRM target: Attio
Channel: LinkedIn (HeyReach) / email (Smartlead or Apollo Sequences) / both

IMPORTANT: This will create real records in your CRM and load real sequences into your sequencer.

Ready to proceed? (yes/no)
```

Do NOT proceed without explicit "yes."

### Step 3: Push to CRM (Attio)

**If Attio MCP is connected:**

Use Attio MCP tools to:
1. Create company records for each unique company in the prospect list
2. Create person records for each contact, linked to their company
3. Add a note to each record with the ICP score and score reasoning
4. Tag or label records with the client name for filtering

For each company, use `create-record` with:
- Object type: companies
- Fields: name, domain, industry, employee count, location

For each person, use `create-record` with:
- Object type: people
- Fields: name, email, phone, title, company (linked)

Add a note via `create-note` with the ICP score and reasoning.

**If Attio MCP is NOT connected:**

Tell the user:

```
Attio MCP is not connected. You have two options:
1. Connect Attio MCP in your Claude settings and re-run /activate
2. I can export the data as a CSV formatted for Attio import

Which do you prefer?
```

If CSV export, write `clients/<client-name>/attio-import.csv` with Attio-compatible columns.

### Step 4: Pick the channel

Before pushing, ask the user which channel(s) to activate on, based on what's connected:

- **HeyReach MCP connected?** → LinkedIn outreach is an option. Good for B2B outbound to founders/marketers, especially when email coverage is patchy. Leads enrolled by LinkedIn URL (no email required).
- **Smartlead MCP connected?** → Email outreach. Good when email coverage is ≥70% and the audience reads cold email.
- **Both connected?** → Multi-channel (LinkedIn connect → LinkedIn message → email) has higher response rates than either alone. Recommend both unless the user has a reason not to.
- **Neither connected?** → Apollo Sequences (if available on their plan) or CSV fallback.

Default preference when not specified: **HeyReach (LinkedIn) first** when the prospect list has strong LinkedIn URL coverage and patchy email coverage, which is the common case on free-tier enrichment plans.

### Step 5: Push to LinkedIn (HeyReach)

Use the HeyReach MCP. Typical flow (verify exact tool names against the installed MCP — they vary by provider):

1. List LinkedIn sender accounts → ask the user which sender to use. LinkedIn sender matters: the message comes from this person's profile.
2. Create a new campaign named `<client-name>-<YYYY-MM-DD>`. **Always create paused.**
3. Load the LinkedIn sequence steps (connection request note + 2–3 follow-up messages) from `clients/<client-name>/sequences/`. If drafts were written for email, adapt: shorter, no subject line, no `{{first_name}}` placeholders that HeyReach doesn't recognize (use the HeyReach variable syntax — verify in their docs).
4. Add leads to the campaign using LinkedIn URLs from `prospects.enriched.csv`. HeyReach deduplicates by profile URL.
5. Leave paused. Show the user the HeyReach campaign URL so they can review in the UI and start the campaign manually.

**Safety rails specific to LinkedIn:**
- LinkedIn connection-request spam gets accounts restricted. Keep daily send volume low (HeyReach's default limits are sane — don't override).
- Connection-request notes have a 300-char limit. Adapt email copy accordingly.
- If the sender account has had recent restrictions or is new (<30 days old), warn the user and suggest they use an older account.

### Step 6: Push to Email Sequencer

**Option A (default): Smartlead MCP**

Smartlead is the default sequencer. Use the LeadMagic Smartlead MCP (`npx smartlead-mcp-by-leadmagic install`, `SMARTLEAD_API_KEY`). The flow:

1. `smartlead_fetch_all_email_accounts` — list available sending accounts. Ask the user which sender to use for this campaign.

2. `smartlead_create_campaign` — create a new campaign named `<client-name>-<YYYY-MM-DD>`. **Always create paused.** Do not auto-start.

3. `smartlead_update_campaign_sequences` — load the drafted email steps from `clients/<client-name>/sequences/` into the campaign. Preserve per-step delay and subject line from the drafts.

4. `smartlead_add_leads_to_campaign` — enroll the contacts from `prospects.enriched.csv`. Pass personalization fields (first_name, company_name, score_reasoning → custom variable) so sequence merge tags work.

5. Present the final confirmation:
   ```
   Campaign "<client>-<date>" created in Smartlead (PAUSED).
   - Sender: <sender-email>
   - <N> leads enrolled
   - <M> email steps loaded

   Campaign is paused. Open Smartlead to review and start sending:
   https://app.smartlead.ai/app/email-campaign/<campaign-id>
   ```

   **Do not call `smartlead_resume_campaign`.** The user resumes in the Smartlead UI after visual review. This is the final irreversible gate.

**Option B: Apollo Sequences** (only if user explicitly prefers Apollo over Smartlead)

1. `apollo_emailer_campaigns_search` — find the target sequence. Apollo's API can't create sequences, so one must already exist.
2. `apollo_contacts_search` / `apollo_contacts_create` (with `run_dedupe: true`) — ensure contacts exist.
3. `apollo_email_accounts_index` — pick sender.
4. Confirm with user, then `apollo_emailer_campaigns_add_contact_ids` to enroll.

**Option C (last resort): CSV export**

Only if neither MCP is available. Write:
- `clients/<client-name>/smartlead-contacts.csv` (contact list for import)
- `clients/<client-name>/smartlead-sequences.md` (email copy to paste into Smartlead)

### Step 7: Post-activation summary

```
Campaign activated for <client-name>:

CRM (Attio):
- <N> companies created/updated
- <N> contacts created/updated
- All tagged with "<client-name>" label

LinkedIn (HeyReach, if used):
- <N> leads enrolled in "<campaign-name>"
- Sender account: <linkedin-profile>
- Status: PAUSED — resume in HeyReach UI after review

Email (Smartlead / Apollo, if used):
- <N> contacts enrolled in "<sequence-name>"
- Sending from: <sender-email>
- Status: PAUSED — resume in sequencer UI after review

Next steps:
- Open each sequencer UI, review the messages in context, then resume
- Check CRM for bounced/unsubscribed contacts after first send window
- Re-run /prospect + /enrich-and-score in 2-4 weeks for fresh prospects
```

### Step 8: Log

Write to `clients/<client-name>/logs/activate.log.md`:

```markdown
# Activation Log
- **Date:** <timestamp>
- **CRM:** Attio
  - Companies pushed: <N>
  - Contacts pushed: <N>
  - Method: MCP / CSV export
- **LinkedIn channel:** HeyReach (if used)
  - Leads enrolled: <N>
  - Campaign name: <name>
  - Status: paused (always)
  - Method: MCP / CSV export
- **Email channel:** Smartlead / Apollo (if used)
  - Contacts enrolled: <N>
  - Sequence name: <name>
  - Status: paused (always)
  - Method: MCP / CSV export
- **Output files:** <list any export files created>
- **Status:** Campaign loaded, paused, awaiting user-initiated start
```

## Tools used

- **Attio MCP** (if connected): create-record, create-note
- **HeyReach MCP**: list senders, create campaign, add leads (by LinkedIn URL), set sequence, pause/start (verify exact tool names against the installed MCP — they vary by provider)
- **Smartlead MCP (LeadMagic)**: smartlead_fetch_all_email_accounts, smartlead_create_campaign, smartlead_update_campaign_sequences, smartlead_add_leads_to_campaign
- **Apollo.io** (Apollo-sequences fallback): apollo_contacts_create, apollo_contacts_search, apollo_emailer_campaigns_search, apollo_emailer_campaigns_add_contact_ids, apollo_email_accounts_index

## Safety rails

This skill handles irreversible actions. Extra caution:

1. **Always confirm before pushing.** Never auto-push to CRM or any sequencer.
2. **Check for duplicates.** Use `run_dedupe: true` in Apollo. Check Attio for existing records. Smartlead deduplicates by email; HeyReach deduplicates by LinkedIn URL within a campaign.
3. **Always create paused.** Every HeyReach campaign, Smartlead campaign, and Apollo enrollment is created in paused state. The user resumes manually in the respective UI after visual review. No exceptions.
4. **LinkedIn-specific rails.** Keep daily send volumes within HeyReach's defaults. Warn if the chosen sender account is new (<30 days) or recently restricted. Keep connection-request notes under 300 characters.
5. **Log everything.** If something goes wrong, the log should tell you exactly what was pushed, to which channel, and when.
6. **No bulk delete.** If the user made a mistake, don't try to bulk-delete from CRM or sequencers. Flag the issue and let them handle it in the respective UI.
