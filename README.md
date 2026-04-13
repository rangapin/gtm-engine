# GTM Engine

Client domain in, launch-ready outbound campaign out. Built on Claude Code.

## What It Does

Give it a company domain. It researches the business, defines who they should sell to, finds matching prospects, enriches them, scores them, writes personalized email sequences, and pushes everything to your CRM and sequencer. Human review at every step.

## The Workflow

```
acme.com
  -> Research the client (web + Exa)
  -> Define their ICP (interactive)
  -> Find prospects (Apollo)
  -> Enrich + score contacts (Apollo, optional Clay)
  -> Draft email sequences (Claude)
  -> Push to CRM + sequencer (Attio, Apollo/Smartlead)
```

45 minutes from domain to live campaign. With review gates so nothing ships without your eyes on it.

## Stack

| Role | Tool |
|------|------|
| Orchestration | Claude Code |
| Prospecting + Enrichment | Apollo MCP |
| Additional enrichment | Clay MCP (optional) |
| CRM | Attio MCP |
| Email sequencer | Apollo Sequences / Smartlead |
| Research | Web search + Exa MCP |

## Quick Start

1. Clone this repo
2. Copy `.env.example` to `.env`, add your API keys
3. Connect Apollo and Attio MCPs in Claude Code settings
4. Run: `/campaign-run stripe.com`
5. Review each step, approve, repeat

## Skills

| Skill | What it does |
|-------|-------------|
| `/campaign-init` | Scaffold a new client project |
| `/research-client` | Research the client's business from their domain |
| `/icp-define` | Define ICP based on research |
| `/prospect` | Find companies and contacts matching ICP |
| `/enrich-and-score` | Enrich with emails/phones, score against ICP |
| `/draft-sequences` | Write personalized email sequences |
| `/activate` | Push to CRM and sequencer |
| `/campaign-run` | Run the full chain with review gates |

## Built For

GTM engineers running outbound on behalf of clients. One engineer can manage multiple campaigns because the grunt work is automated and the judgment calls are surfaced for human review.

## License

MIT
