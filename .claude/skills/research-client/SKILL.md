---
name: research-client
description: Research a client's business from their domain. Scrapes their website, analyzes their positioning, identifies what they sell and who they sell to. Outputs a structured brief. Use after /campaign-init.
---

# /research-client - Research the Client's Business

When the user runs `/research-client <client-name>`:

## What to do

### Step 1: Load context

Read `clients/<client-name>/brief.md` and `clients/<client-name>/icp.json` to get the domain.

### Step 2: Research the company

Use **web search** and **Exa** to gather information about the client. Search for:

1. **Their website** - what do they sell? What's on their homepage, product page, pricing page?
2. **Their LinkedIn company page** - company size, industry, description
3. **Recent news or press** - funding, product launches, partnerships
4. **Their competitors** - who else operates in their space?
5. **Their customers** - any case studies, testimonials, or logos on their site?

Use Exa (`web_search_exa`) for deeper semantic searches like:
- `"<company name> case study customer"` to find who their customers are
- `"<company name> vs"` to find competitors
- `category:company <domain>` to find the company profile

Use web search for:
- `site:<domain>` to understand their product
- `"<company name>" funding OR raised OR series` for funding info
- `"<company name>" reviews OR customers` for social proof

### Step 3: Synthesize into the brief

Update `clients/<client-name>/brief.md` with your findings:

```markdown
# Client Brief: <Client Name>

**Domain:** <domain>
**Researched:** <date>
**Status:** Research complete

## Company Overview
<2-3 sentences: what this company is, when founded, HQ, size, funding stage>

## What They Sell
<Clear description of their product/service. Be specific about the value proposition, not just the category.>

### Product/Service Details
<Key features, pricing model if visible, deployment model (SaaS, on-prem, etc.)>

## Who They Sell To
<Based on their website copy, case studies, and positioning:>

### Industries
<Which industries do their customers come from?>

### Company Size
<SMB, mid-market, enterprise? What signals this?>

### Buyer Personas
<Who is the decision maker? Who is the user? What titles appear in their case studies?>

### Geography
<Where do they operate? US-only? Global? EU focus?>

## Positioning & Messaging
<How do they position themselves? What language do they use? What pain points do they address?>

### Key Claims
<Bullet their main value propositions as stated on their site>

### Competitor Landscape
<Who are the main alternatives? How does the client differentiate?>

## Signals That Indicate a Good Prospect for Them
<Based on what you learned about their customers, what observable signals would indicate a company needs what this client sells? Think: hiring patterns, tech stack, funding stage, growth indicators, industry events.>

## Notes
<Anything else relevant: red flags, gaps in their positioning, things to ask the client about>
```

### Step 4: Review gate

Present the completed brief to the user. Ask:

```
Here's what I found about <client-name>. Please review:

[show the brief]

Does this look right? Anything to add, correct, or remove before we define the ICP?
```

Wait for the user to confirm or make edits. Apply any edits to the brief.

### Step 5: Log

Write to `clients/<client-name>/logs/research.log.md`:

```markdown
# Research Log
- **Date:** <timestamp>
- **Sources searched:** <count>
- **Tools used:** web_search, Exa
- **Brief written to:** clients/<client-name>/brief.md
- **Status:** Complete, user reviewed
```

## Tools used

- **Web search** (built-in)
- **Exa web_search_exa** for semantic company research

## Tips for better research

- Check the footer of the client's website for case study links, partner logos, and industry badges
- Look at their job postings to understand what teams they're building (this tells you a lot about their go-to-market motion)
- Their "About" or "Customers" page often reveals the ICP more honestly than their homepage
- If they have a blog, recent posts reveal what topics they want to be known for
- G2/Capterra reviews from their customers reveal pain points and use cases
