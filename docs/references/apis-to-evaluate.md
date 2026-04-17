# APIs to evaluate (scrapbook, not a plan)

Candidate APIs for future pipeline augments. Not committing to any of these — this is a shortlist to refer back to when a specific gap gets painful enough to pay for.

Sourced from a public post attributed to an agency running Claude Code + Instantly at scale ($7m ARR, 70+ B2B orgs). Annotated with context from our current tooling.

---

## Company discovery (`/prospect`)

- **Apollo** *(current)* — primary. 70M+ companies, includes lookalike and Boolean search.
- **AI Ark** — 70M+ company DB with lookalike search. Fallback if Apollo misses a vertical (small EU, niche B2B, or a specific industry where Apollo's firmographics degrade).
- **CompanyEnrich** — 30M+ companies. Same category as AI Ark; redundant with Apollo for most cases. Evaluate only if both Apollo and AI Ark have coverage gaps.

## People discovery (`/prospect`)

- **Apollo** *(current)* — primary.
- **BlitzAPI** — ICP waterfall for decision-makers at flagged companies. Multi-contact per account, filter by persona/seniority/dept.
- **LeadsFactory** — real-time people search at named companies (as opposed to batch). Useful when a list of target companies is fixed and you need fresh contact coverage.

## Signal collection (`/gather-signals`)

- **Current:** WebFetch + Exa over free sources — hiring pages, press releases, G2, GitHub, Product Hunt, Reddit, company blogs, LinkedIn company cards. Brittle (403s, parse variance, anti-bot) but zero cost.
- **PredictLeads** — **real gap-filler.** Structured buying signals (hiring spikes, tech stack changes, product launches, press) without the free-source fragility. Strongest candidate here for a paid augment when the free scrapers start missing signals that matter.

## Email enrichment (`/enrich-and-score`)

- **Apollo** *(current)* — primary email.
- **Clay** *(current, optional)* — waterfall orchestrator across multiple providers.
- **Limadata** — email finder; covers gaps Apollo misses. At scale, two providers beats one every time.
- **Prospeo** — alternative email finder, same role as Limadata.

## Phone enrichment (`/enrich-and-score`)

- **Apollo** *(current)* — weak phone coverage.
- **FullEnrich** — **real gap.** 20+ provider cascade, ~95% phone match. Relevant only for clients whose motion includes cold calling (most of ours don't — HeyReach and Instantly don't need phones). Revisit if a client explicitly needs phone outreach.

## Activation (`/activate`)

- **Instantly** *(current, preferred email sender)*
- **HeyReach** *(current, LinkedIn channel)*
- **Smartlead** *(current, paid email — use when it's worth the cost)*
- **Apollo Sequences** *(current email fallback, requires paid Apollo plan)*
- **lemlist** — multi-channel alternative to Instantly. Evaluate only if Instantly's deliverability or multi-channel features become blockers.

---

## What the source post didn't mention

Complementary tooling we already use that isn't in the post's stack:

- **Clay** — enrichment waterfall orchestrator (already integrated as optional)
- **HeyReach** — LinkedIn channel for `/activate`
- **Exa** — slug discovery + WebFetch fallback in `/gather-signals` and `/research-client`

## The "auto-improves like a retargeting pixel" claim

The source post ends with: "Claude pulls campaign analytics from Instantly, identifies which personas responded, builds lookalikes of your best-performing leads, and feeds them into the next campaign."

That's the promise. Our implementation: `/critique` Finding 3 (reply rate by signal type, joined with `results.csv`) + the `Cross-client weak signal type:` loop into `/gather-signals` downweight (commit `0db9ba2`, 2026-04-17). The paid stack in the source post gestures at this loop but doesn't typically close it without operator judgment — we do.

If a future API claims a built-in feedback loop, the bar is: does it cite per-signal-type reply rate, or does it just surface "accounts that engaged"? The second is table stakes; the first is what actually compounds.

---

## How to use this file

When a specific gap surfaces during a campaign (Apollo misses a vertical, phone coverage matters for a client's channel mix, signals keep 403-ing), come back here and pick the narrowest API that closes the gap. Don't buy the whole stack. Each paid provider adds ongoing cost, onboarding friction, and one more skill branch to maintain.
