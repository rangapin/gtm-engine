---
name: activate
description: Push approved prospects to Attio (CRM) and launch the outbound campaign via HeyReach (LinkedIn), Smartlead (email), or Gmail drafts (manual/test mode). Final step before campaign goes live. Use after /draft-sequences and user approval.
---

# /activate - Push to CRM and Sequencer

When the user runs `/activate <client-name>`:

## What to do

### Step 1: Load context

Read:
- `clients/<client-name>/prospects.enriched.csv` (the scored prospect list)
- `clients/<client-name>/sequences/` (the approved email sequences)
- `clients/<client-name>/icp.json` (for reference)

### Step 1.5: Dedup against the global contacted ledger

Before anything touches real channels, check this client's prospect list against `clients/_contacted.csv` — the global contacted ledger. This is the only skill that reads across client folders, specifically to prevent re-contacting someone on a different campaign or a re-run.

1. **Read the ledger.** If `clients/_contacted.csv` doesn't exist, treat as empty and skip to Step 2 (first-ever activation).

2. **Build a match key per prospect** in this client's `prospects.enriched.csv` (filter to non-disqualified rows above the activation threshold). Use the first non-empty key in priority order:
   - `apollo_person_id`
   - `person_email`
   - `person_linkedin`
   - `company_domain|person_name` (composite fallback for legacy rows)

3. **Match against the ledger** using the same priority. A prospect is a dupe if any key matches any ledger row's corresponding column.

4. **If dupes exist, show the user:**

   ```
   Dedup check: N of M prospects are already in the global contacted ledger.

   [Up to 10 rows: person_name, title, company | prior client | channel | date]

   Default: exclude all dupes from this campaign.

   Options:
   - (press Enter) Exclude all — recommended
   - Override specific rows (list numbers to keep, e.g. "2, 5, 9")
   - Type "override all" to contact everyone, ignoring prior contact (reputation risk)
   ```

5. **Apply the user's choice.** The filtered set is the working list for every step below. Step 2's pre-flight counts reflect the post-dedup count.

6. **If zero dupes**, say one line ("Dedup check: 0 of N prospects already contacted") and continue — no prompt.

7. **Record the dedup outcome** for Step 8's log: `dedup_blocked: N; dedup_overridden: N; dedup_key_hits: {apollo_id: N, email: N, linkedin: N, composite: N}`.

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
- **Gmail MCP connected?** → **Manual / test mode.** Creates unsent email drafts in the authed Gmail account — one draft per prospect, email 1 of the sequence only. Good for: workflow smoke tests, low-volume outbound (<10 prospects), or when the user wants eyes-on-every-email before send. Not a real sequencer — no follow-up automation, no deliverability warming, no reply tracking.
- **Both HeyReach + Smartlead connected?** → Multi-channel (LinkedIn connect → LinkedIn message → email) has higher response rates than either alone. Recommend both unless the user has a reason not to.
- **Neither HeyReach nor Smartlead?** → Offer Gmail drafts (if connected) for small volume, Apollo Sequences (if available on their plan) for paid Apollo users, or CSV fallback as last resort.

Default preference when not specified: **HeyReach (LinkedIn) first** when the prospect list has strong LinkedIn URL coverage and patchy email coverage, which is the common case on free-tier enrichment plans. Gmail drafts if the user says the word "test" or the prospect count is ≤5.

### Step 5: Push to LinkedIn (HeyReach)

Use the HeyReach MCP. Typical flow (verify exact tool names against the installed MCP — they vary by provider):

1. List LinkedIn sender accounts → ask the user which sender to use. LinkedIn sender matters: the message comes from this person's profile.
2. Create a new campaign named `<client-name>-<YYYY-MM-DD>`. **Always create paused.**
3. **Adapt email drafts to LinkedIn format:**
   - Shorter. Connection-request note ≤300 chars. Follow-ups 2-4 short sentences.
   - No subject line on LinkedIn messages. Drop `{{EMAIL_SUBJECT}}`.
   - Map canonical tokens to HeyReach variable syntax (see mapping table below). If unsure, inspect an existing HeyReach campaign or template via the MCP to confirm the exact token format before pushing.
4. Load the LinkedIn sequence steps (connection request note + 2–3 follow-up messages) from `clients/<client-name>/sequences/`.
5. Add leads to the campaign using LinkedIn URLs from `prospects.enriched.csv`. HeyReach deduplicates by profile URL. Pass per-prospect hook values (`HOOK_SHORT`, `HOOK_EXPANDED`) as custom variables so they survive token substitution.
6. Leave paused. Show the user the HeyReach campaign URL so they can review in the UI and start the campaign manually.

**Canonical token → platform mapping:**

| Canonical (from draft-sequences) | Smartlead | HeyReach | Notes |
|---|---|---|---|
| `{{FIRST_NAME}}` | `{{first_name}}` | `{{FirstName}}` | Native merge tag on both |
| `{{LAST_NAME}}` | `{{last_name}}` | `{{LastName}}` | Native merge tag on both |
| `{{COMPANY}}` | `{{company_name}}` | `{{CompanyName}}` | Native merge tag on both |
| `{{SENDER_NAME}}` | `{{sender_name}}` | sender profile name | HeyReach uses sender account, not a variable |
| `{{HOOK_SHORT}}` | custom var | custom var | Must be passed per-lead at enrollment |
| `{{HOOK_EXPANDED}}` | custom var | custom var | Must be passed per-lead at enrollment |
| `{{EMAIL_SUBJECT}}` | subject field | n/a | Email only |

HeyReach variable syntax varies by MCP version. **Before pushing**, call the MCP to fetch an existing campaign or template and confirm the exact token format (e.g., `{{FirstName}}` vs `{{first_name}}` vs `[[FirstName]]`). If the mapping is wrong, literal `{{...}}` strings will ship into real LinkedIn messages.

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

4. `smartlead_add_leads_to_campaign` — enroll the contacts from `prospects.enriched.csv`. Map canonical tokens to Smartlead syntax (see token mapping table in Step 5). Pass personalization fields as Smartlead custom variables: `first_name`, `last_name`, `company_name`, plus per-prospect hook values (`HOOK_SHORT`, `HOOK_EXPANDED`, `EMAIL_SUBJECT`) so sequence merge tags resolve.

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

**Option C: Gmail drafts (test / manual mode)**

Creates unsent drafts in the authed Gmail account. **Email 1 only** — follow-ups stay in `sequences/*.md` and the user re-drafts them closer to send time if/when email 1 gets a reply.

Before creating any draft:

1. Call `gmail_get_profile` to confirm the authed account. **Flag to the user if the Gmail account differs from the client's domain** (e.g., authed as `personal@gmail.com` but the client is `acme.com`). Offer to abort so they can connect the right account. This is a foot-gun — drafts land in the "wrong" inbox if the user doesn't notice.

2. For each prospect in `prospects.enriched.csv`:
   - **If `person_email` is populated:** create the draft with `to = person_email`, subject = email 1 subject from the sequence file, body = email 1 body.
   - **If `person_email` is blank (pending enrichment):** create the draft with `to` omitted, and prefix the subject with `[DRAFT — <person_name>, email TBD]` so it sorts recognizably in the drafts folder. Include the LinkedIn URL in the first line of the body so the user can hand-resolve the address later.

3. Use `gmail_create_draft` per prospect. Capture the returned `draftId` for the log.

4. Present the final confirmation:
   ```
   Gmail drafts created in <authed-email> (UNSENT):
   - <N>/<total> with To: populated
   - <M> with blank To: (email pending enrichment)

   Review and send manually:
   https://mail.google.com/mail/u/0/#drafts
   ```

   **Do not call `gmail_send` or `gmail_send_draft`.** Drafts stay unsent. This is the irreversible gate — the user sends each one from Gmail after visual review.

**Option D (last resort): CSV export**

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

Gmail drafts (if used):
- <N> drafts created in <authed-email>
- <M>/<N> have To: populated; rest pending email enrichment
- Status: UNSENT — user sends manually from Gmail

Next steps:
- Open each sequencer UI, review the messages in context, then resume
- Check CRM for bounced/unsubscribed contacts after first send window
- Re-run /prospect + /enrich-and-score in 2-4 weeks for fresh prospects
```

### Step 7.5: Append to the contacted ledger

After the push is confirmed successful (CRM + sequencer reported OK, campaign visible in the sender UI), append one row per actually-activated prospect per channel to `clients/_contacted.csv`. This is what the *next* `/activate` (for any client) will dedupe against.

1. **Create the file with headers if it doesn't exist:**
   ```
   apollo_person_id,person_email,person_linkedin,company_domain,person_name,client_name,activated_at,channel,campaign_id
   ```

2. **For each activated prospect**, append one row per channel used (a prospect activated on LinkedIn AND email generates two rows). Fields come from `prospects.enriched.csv` plus activation metadata:
   - `apollo_person_id`, `person_email`, `person_linkedin`, `company_domain`, `person_name` — from the enriched row (blank if absent; that's fine)
   - `client_name` — current client
   - `activated_at` — ISO 8601, now
   - `channel` — `linkedin` / `email` / `gmail`
   - `campaign_id` — HeyReach or Smartlead campaign ID/name; blank for Gmail drafts

3. **Include overridden dupes.** If the user overrode a dedup warning in Step 1.5 and re-contacted a prospect already in the ledger, that prospect gets a fresh row here — the ledger records every intent-to-contact, not just first-touch.

4. **Exclude dedup-blocked prospects.** They weren't activated, so they don't get a new row. Their original ledger entry stands.

5. **Append-only.** Never rewrite or dedupe historical rows. The ledger is a permanent audit trail.

### Step 8: Log

Write to `clients/<client-name>/logs/activate.log.md`:

```markdown
# Activation Log
- **Date:** <timestamp>
- **Dedup (Step 1.5):**
  - Prospects considered: <M>
  - Dupes found in `_contacted.csv`: <N>
  - Dupes excluded: <N>
  - Overridden by user: <N>
  - Key hits: apollo_id=<n>, email=<n>, linkedin=<n>, composite=<n>
- **Ledger rows appended (Step 7.5):** <N>
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

- **Attio MCP** (if connected): upsert-record (by domain for companies, by email for people), create-record (for people without email), create-note
- **HeyReach MCP**: list senders, create campaign, add leads (by LinkedIn URL), set sequence, pause/start (verify exact tool names against the installed MCP — they vary by provider)
- **Smartlead MCP (LeadMagic)**: smartlead_fetch_all_email_accounts, smartlead_create_campaign, smartlead_update_campaign_sequences, smartlead_add_leads_to_campaign
- **Apollo.io** (Apollo-sequences fallback): apollo_contacts_create, apollo_contacts_search, apollo_emailer_campaigns_search, apollo_emailer_campaigns_add_contact_ids, apollo_email_accounts_index
- **Gmail MCP** (test/manual mode): gmail_get_profile, gmail_create_draft. Never use gmail_send or gmail_send_draft — drafts must stay unsent.

## Safety rails

This skill handles irreversible actions. Extra caution:

1. **Always confirm before pushing.** Never auto-push to CRM or any sequencer.
2. **Check for duplicates.** Step 1.5 runs global dedup against `clients/_contacted.csv` before pre-flight — this is the primary defense against re-contacting someone across clients or re-runs. Platform-level dedup is the second line: `run_dedupe: true` in Apollo, Attio upsert by domain/email, Smartlead by email, HeyReach by LinkedIn URL within a campaign.
3. **Always create paused / unsent.** Every HeyReach campaign, Smartlead campaign, Apollo enrollment, and Gmail draft is created in paused/unsent state. The user resumes or sends manually in the respective UI after visual review. No exceptions.
4. **LinkedIn-specific rails.** Keep daily send volumes within HeyReach's defaults. Warn if the chosen sender account is new (<30 days) or recently restricted. Keep connection-request notes under 300 characters.
5. **Gmail-specific rails.** The authed Gmail account may not match the client's domain — always call `gmail_get_profile` first and flag a mismatch to the user before creating drafts. Never call `gmail_send` or `gmail_send_draft`; drafts stay unsent until the user sends them from the Gmail UI.
6. **Log everything.** If something goes wrong, the log should tell you exactly what was pushed, to which channel, and when.
7. **No bulk delete.** If the user made a mistake, don't try to bulk-delete from CRM or sequencers. Flag the issue and let them handle it in the respective UI.
