# GTM Engine

Client domain in, launch-ready outbound campaign out. Built as a set of [Claude Code](https://docs.claude.com/claude-code) skills — no build step, no runtime, no custom server. The "code" is `.md` files that orchestrate MCP tool calls. State lives on disk under `clients/<name>/`.

## What it does

Give it a domain. It:

1. Researches the business from their site
2. Defines the ICP with you, interactively
3. Finds companies + contacts matching it (Apollo)
4. Scrapes free public sources for campaign-relevant signals (hiring surges, funding, launches)
5. Enriches contacts with emails + phones, scores them against the ICP
6. Drafts personalized sequences per prospect (anti-AI-voice rules, persona-calibrated)
7. Pushes to your CRM and sequencer — always paused, for visual review before any send

~25 minutes from domain to paused campaign. Review gates at every step. The file system is the state; you can inspect and edit any file between steps.

## Who this is for

GTM engineers running outbound on behalf of clients — where one operator needs to manage several active campaigns and the bottleneck is per-campaign grunt work (prospect discovery, enrichment, segmentation, copy per prospect), not strategy. If you're running one campaign for your own company, this is overkill.

## Prerequisites

- [Claude Code](https://docs.claude.com/claude-code) installed and authenticated
- Node.js 20+ (for `npx`-based MCP servers)
- A paid Apollo.io account — required for prospecting + enrichment. Free tier doesn't return emails.
- At least one activation channel: HeyReach (LinkedIn), Instantly or Smartlead (email), or Gmail drafts (test / low-volume manual)
- Attio account for CRM (optional — falls back to CSV export)

## Install

1. **Clone:**
   ```bash
   git clone https://github.com/rangapin/gtm-engine.git
   cd gtm-engine
   ```

2. **Set up env:**
   ```bash
   cp .env.example .env
   ```
   Fill in the keys you have. Skip optional ones — skills detect what's connected and adapt.

3. **Connect MCP servers.** Each provider has a different path. Only Apollo + one activation channel are strictly required.

   | MCP | Required? | How to connect |
   |-----|-----------|----------------|
   | Apollo | **yes** | Hosted — add in Claude Code's MCP settings, OAuth in browser |
   | Attio | recommended | Hosted — same pattern as Apollo |
   | Exa | recommended | Hosted — used for research + signal gathering (G2, ProductHunt, LinkedIn slug discovery) |
   | HeyReach (LinkedIn) | one channel required | Local: `npx @heyreach/heyreach-mcp-server --api-key YOUR_KEY`. Use a **space**, not `=` — the `--api-key=VALUE` form truncates base64 padding on some shells |
   | Instantly (email, preferred) | or | Remote MCP URL: `https://mcp.instantly.ai/mcp/YOUR_API_KEY` |
   | Smartlead (email, alt) | or | `npx smartlead-mcp-by-leadmagic install`, needs `SMARTLEAD_API_KEY` in env |
   | Clay | optional | Hosted — adds enrichment depth |
   | Gmail | optional | Hosted — test/manual mode; creates unsent drafts only |

4. **Verify the skills are visible.** In Claude Code, type `/` — you should see `/campaign-run`, `/prospect`, etc. If not, make sure you're in this repo's working directory. Skills load from `.claude/skills/`.

## Quickstart

Run the full pipeline with review gates between each step:

```
/campaign-run stripe.com
```

Expect ~25 minutes of answering questions about ICP, editing prospect lists, and reviewing copy before the paused campaign lands in your sender's UI.

Or run individual steps:

```
/campaign-init stripe
/research-client stripe
/icp-define stripe
/prospect stripe
/gather-signals stripe
/enrich-and-score stripe
/draft-sequences stripe
/activate stripe
```

Check status anytime:

```
/campaign-status         # all clients
/campaign-status stripe  # one client
```

After a campaign has been live ≥7 days, close the loop:

```
/capture-results stripe  # scaffolds results.csv for manual outcome entry
/critique stripe         # evidence-derived per-step ratings
```

## The pipeline

```
domain
  → /campaign-init     scaffold clients/<name>/
  → /research-client   scrape site + Exa; writes brief.md
  → /icp-define        interactive; writes icp.json with messaging angles
  → /prospect          Apollo search; writes prospects.csv
  → /gather-signals    free-source scrape; writes signals.csv
  → /enrich-and-score  Apollo enrich + transparent scoring; writes prospects.enriched.csv
  → /draft-sequences   per-prospect sequences with anti-AI-voice lint; writes sequences/*.md
  → /activate          push to CRM + sender (always paused)

after the campaign runs:
  → /capture-results   you fill results.csv with replies / bounces / meetings
  → /critique          outcome-gated evaluation; writes critique.md
```

## Safety rails

- **Paused-first.** Every HeyReach / Instantly / Smartlead campaign and Gmail draft lands un-sent. You resume in the sender UI after visual review. No skill will call `resume_campaign` or `send_draft`.
- **Two-half budget gate.** You see the actual top-10 rows of `prospects.csv` *before* any Apollo enrichment credits are spent. The spend happens in `/enrich-and-score`, with the current row count displayed and a per-credit estimate.
- **Global contacted-ledger dedup.** `/activate` checks `clients/_contacted.csv` before pushing — prevents re-contacting anyone across clients or re-runs. Matches on `apollo_person_id → email → linkedin → company_domain|person_name`.
- **Tiered review gates.** Silent on free steps (research, ICP), budget gate at the credit-spend boundary, full quality gate after copy is drafted, irreversible gate before activate.
- **Transparent scoring.** Every ICP score has a one-line `score_reasoning` you can read and argue with. No black-box ML.

## The feedback loop

This repo's architectural bet is that per-run critiques compound into better skills over time.

1. A campaign activates. Real prospects get messaged.
2. Wait 7+ days, then run `/capture-results` — it scaffolds `results.csv` with rows for each non-disqualified prospect. You fill in replies, bounces, meetings.
3. Run `/critique`. It's outcome-gated: requires `fill_rate ≥ 50%` on a ≥7-day campaign to give full ratings. It then proposes evidence-derived caps per skill (e.g. any step with 0 replies across ≥10 sends caps `/draft-sequences` at ≤3).
4. Every future skill run reads `clients/*/critique.md` to extract the top 3 recurring weaknesses and bias against them.

**Without running at least one campaign end-to-end and capturing results, the learning loop is theoretical.** The repo doesn't ship with any critique history.

## File layout

```
.claude/skills/        # skill definitions (the "code")
CLAUDE.md              # repo conventions + skill contract + file schemas (authoritative)
clients/               # per-client working state (gitignored except critique.md + assumptions.md)
  <name>/
    brief.md
    icp.json
    prospects.csv
    prospects.enriched.csv
    signals.csv
    sequences/
    results.csv
    critique.md
    logs/
  _contacted.csv       # global contacted-prospect ledger (cross-client dedup)
```

The file system **is** the state. No database, no in-memory globals, nothing persisted outside the repo. Edit any file between skills and continue.

## Cost expectations

- **Apollo:** ~1 credit per company search result; ~1 credit per `people_match` (email reveal). A 100-prospect campaign ≈ 200 credits on the cheapest paid tier.
- **Clay** (optional): per-data-point; highly variable.
- **Exa:** one campaign usually fits inside free-tier search quota.
- **HeyReach / Instantly / Smartlead:** flat subscription, not per-send.
- **Claude Code:** counts against your Anthropic API usage. Expect ~1–3M tokens per full campaign including all review gates.

## Customizing for your stack

Skills branch on which MCPs are connected — missing tools are downgraded, not errored. See the "MCP availability is conditional" section in [CLAUDE.md](./CLAUDE.md) for per-MCP fallback rules. Some examples:

- **No Attio** → `/activate` exports `attio-import.csv` for manual upload.
- **No Exa** → `/research-client` works from the client's own site only; `/gather-signals` skips G2 / ProductHunt / LinkedIn slug discovery.
- **No HeyReach + no Instantly + no Smartlead** → `/activate` offers Gmail drafts (for low-volume manual send) or CSV export.
- **No Clay** → `/enrich-and-score` runs Apollo-only.

Swapping in a different sender (e.g. another LinkedIn or email tool) is a matter of:
1. Installing the provider's MCP.
2. Editing the channel section in `.claude/skills/activate/SKILL.md` (Steps 5 and 6) — specifically the canonical-token → platform-variable mapping table.
3. Confirming paused-first discipline survives (no `resume`, no `send`).

## Skills

| Skill | Purpose |
|-------|---------|
| `/campaign-init` | Scaffold a new client project folder |
| `/research-client` | Research the client's business from their domain |
| `/icp-define` | Interactive ICP definition with messaging angles |
| `/prospect` | Find companies + contacts matching the ICP (Apollo) |
| `/gather-signals` | Scrape free public sources for campaign-relevant signals |
| `/enrich-and-score` | Enrich contacts with emails/phones, score against ICP |
| `/draft-sequences` | Write personalized sequences per prospect |
| `/activate` | Push to CRM + sender, always paused |
| `/capture-results` | Scaffold results.csv for manual outcome entry |
| `/critique` | Outcome-gated per-step evaluation; writes critique.md |
| `/campaign-run` | Chain the whole pipeline with review gates |
| `/campaign-status` | Inspect progress across one or all clients |

For the skill contract (how skills read each other's output, what review gates look like, what to preserve when editing), see [CLAUDE.md](./CLAUDE.md).

## Troubleshooting

- **Skills don't show up under `/`.** You're not in the repo directory, or the skills dir is at a path Claude Code doesn't scan. Check that `.claude/skills/<name>/SKILL.md` exists for each skill.
- **Apollo returns no emails.** You're on the free tier. Prospecting works (company + person metadata); `people_match` (email reveal) requires paid.
- **HeyReach MCP fails to authenticate.** Check that you passed `--api-key VALUE` with a space, not `=`. The `=` form truncates base64 padding.
- **`/activate` says "already contacted" for everyone.** Check `clients/_contacted.csv` — if this is a re-prospect of the same ICP, the ledger is working as designed. Override per-row if the message/offer has genuinely changed.
- **`/critique` blocks and redirects to `/capture-results`.** The campaign is ≥7 days old but `results.csv` has <50% fill. Fill it. Outcome-blind critiques produce bad signal.

## License

MIT. (Add a `LICENSE` file to the repo root before publishing — the current repo doesn't have one.)
