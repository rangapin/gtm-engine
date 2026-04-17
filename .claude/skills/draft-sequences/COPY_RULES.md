# Copy rules — cold B2B outbound

Read this every time you draft or lint a sequence. Rules are imperative. Examples are deliberately short.

**Tensions to hold:** several of these rules conflict with each other in edge cases. Permission-ask CTAs read weak to founders who prefer directness. Humor misfires with risk-averse buyers (finance, legal, healthcare compliance). Permission-ask openers underperform when the prospect has signaled urgency — e.g., they just posted a job listing for exactly what you solve, where directness reads as helpfulness, not intrusion. "Start in their world" has one exception: when the prospect already knows your category and is comparing vendors, a direct "We're the X that does Y differently because Z" respects their time more than empathy-first framing. Resolve clashes using the prospect's persona and the client's voice, not by picking a rule and ignoring the rest. When rules clash, note the tradeoff to the user at the review gate.

Full research this distills: see `COPY_RULES_SOURCES.md`.

## The three things that don't reduce to rules

Everything below flows from these. When a specific rule feels wrong for a specific situation, check it against these first:

1. **Relevance is not personalization.** Inserting a name or company is not the same as demonstrating you understand *why this message matters to this person right now*. If the same sentence could be sent to a different prospect with just the name swapped, it's personalization theater — rewrite or cut.
2. **The email is not the sale.** It exists only to start a conversation. Any email that tries to close is doing the wrong job, and it reads that way to the prospect.
3. **Credibility is fragile and binary.** One AI tell, one unverifiable claim, one overused template phrase flips the reader from "maybe" to "spam" with no recovery. You don't get a second impression.

---

## Length

- **Target 50–80 words per email. Hard ceiling 100.** 2.4× reply lift vs 200+ word emails.
- **First-touch: 25–50 words if possible.** The fewer words, the stronger each must be.
- **Write the email, cut it in half, cut it again, ship what survives.**
- **Read it on your phone before sending. If it scrolls, it's too long.**
- **Last email of a sequence: 2–3 sentences.**
- **Smaller lists beat bigger ones.** Elite-tier campaigns run under 50 recipients each. Top performers spend 2–4 minutes of research and editing per email and fewer minutes per campaign — they don't volume their way to reply rates.

## Subject lines

- **Lowercase, 1–5 words.** Title case signals "marketing email."
- **Use prospect-specific context, not your category.** `snowflake → bigquery?` ✓ / `Data integration solution` ✗
- **Questions are allowed and lift opens ~21%.** Use when the body pays off the curiosity.
- **Never use fake `Re:` or `Fwd:` prefixes.** Short-term open bump; long-term trust + domain reputation damage.
- **Follow-ups inherit the subject with `Re: ` or shift to a fresh angle-matched subject.** Do not resurrect Email 1's subject verbatim on Email 3.
- **Curiosity for sophisticated prospects; clarity for the time-poor.** "a thought on retention" intrigues a senior IC scanning their inbox. "your Marketo routing issue" respects a CFO's 6 AM scan. Mismatching the two kills opens.
- **If you open a curiosity gap, the body must pay it off.** "a thought on churn" that opens into a generic pitch creates resentment, not engagement. No bait-and-switch — the curiosity you promise is the curiosity you owe.

## Structure and format

- **Plain text only.** No HTML, no bullet points, no bold, no colors, no images. Humans don't format personal email.
- **Short paragraphs (1–2 sentences each) with white space between.**
- **5th-grade reading level.** A-grade emails (per Lavender) at that level lift reply rates 67%.
- **Three-paragraph "signal → pitch → CTA" skeleton is burned.** Recipients now pattern-match the rhythm even when the words are clean. If every email in a sequence follows HOOK_EXPANDED → proof stat → question, it reads AI regardless of vocabulary.
  - **Two-sentence email:** observation + question. No proof element needed. *"Your job posting for a RevOps Manager mentions CRM consolidation as a priority. We just finished that exact project for a team your size — worth comparing notes?"*
  - **Question-only email:** a single well-crafted question, zero pitch. Works as second or third touch, not first. *"How are you handling lead routing now that you're on HubSpot and Salesforce simultaneously?"*
  - **Insight-first email:** lead with a non-obvious observation or specific data point, then offer to discuss. No pitch at all. *"After talking to 30+ VPs who consolidated from Marketo to HubSpot, the one thing nobody anticipated was attribution breaking silently for 60–90 days. I wrote up the three most common failure modes — want me to send it over?"*
- **Vary the structure across a sequence.** Email 1 two-sentence, Email 2 insight-first, Email 3 question-only — they shouldn't all have the same rhythm. Tone shifts between emails signal AI generation.
- **Every sentence exists only to get the next sentence read.** (Sugarman's slippery slide.) If a sentence doesn't pull the reader into the one that follows, cut it. This is why preambles, meta-commentary about the email itself, and throat-clearing setups fail — they stop the slide.
- **Write to one person, not a segment.** Ogilvy's rule, applied: a draft that works for "VPs of Engineering at Series B SaaS" works for none of them. Pick one specific prospect in the group, write directly to that person, then see what survives when you generalize for the segment.
- **Sign-off format: `<first_name>\n<role> @ <domain>`.** Role+domain on second line, not a bare first name (reads as mass-email).

## Openers

- **Earn relevance in the first 7 words.** The recipient's keep/delete decision happens in the preview pane.
- **Timeline hooks (recent event, ≤30 days ideal, 60 days hard ceiling) outperform problem hooks 2.3×.** Funding round, acquisition, product launch, job posting, earnings-call mention, leadership change. Signals older than 60 days are dead — don't reach for them to force a hook. Stale signals read worse than no signal at all.
- **The signal must imply a problem, not state a fact.** "You raised Series B — congrats" ✗ / "After your Series B you're probably hiring 3 SDRs in Q2; onboarding them breaks the playbook that worked at 2" ✓
- **Permission-ask openers ("not sure if this is relevant, but…") work when you lack a strong signal.** They underperform for C-suite where directness signals respect for time, and for in-market prospects (they just posted the exact role you solve) where permission reads as indecisiveness.
- **Pattern-interrupt openers (humor, unexpected observation) must connect to the pitch within 2 sentences, or they become gimmicks.**
- **Reserve pattern interrupts for less-prospected personas.** VP Sales at mid-market SaaS has seen every clever opener already — they fatigue fastest on humor and provocation. VP Operations, Head of Customer Success, and internal-comms leads receive fewer cold emails and still respond to a well-built pattern interrupt.
- **Direct-premise openers work when your brand carries weight or the prospect's need is obvious.**

## The CTA

- **Ask permission; don't assume interest.** "Want the teardown?" ✓ / "Relevant, or is this noise?" ✓ / "Let's get 30 min on the calendar" ✗
- **One CTA per email.** Any additional ask dilutes the primary one.
- **Frame around what the prospect is losing, not what you're offering.** Loss aversion outpulls promise-of-gain.
- **Interest CTAs beat meeting CTAs on first touch.** Good forms: "Relevant?" / "Useful to dig in?" / "Want the teardown?" / "Or is this noise?" / plain-English questions tied to the prospect's situation. **Vary the CTA form across a sequence — never reuse the same construction twice in the same sequence or across templates in the same client run.** Specifically banned: "Worth a look?", "Worth exploring?", "Worth comparing notes?", "Worth [anything]?" — the most overused template-shape tell in 2025 outbound.
- **No calendar links in Email 1.** Links are spam signals and create friction before the prospect has decided they care.
- **Describe what the prospect can do that they couldn't do before — not what your product does.** Feature dumps assume the prospect cares about your solution before you've proven they have the problem. ✓ "Know which deals are actually going to close this quarter — without interrogating your reps Friday afternoon." ✗ "Our platform features AI-powered analytics, real-time dashboards, and enterprise-grade security."
- **Close on a high note.** End with something that leaves the prospect feeling lighter, not pressured. A small piece of useful information, a genuine compliment tied to a specific observation, or a humble exit lifts reply probability. ✓ "Either way, your Q3 take on pipeline coverage was the cleanest framing I've read on that." ✗ "Please let me know either way." The last sentence is the one they remember while deciding whether to reply. (For the breakup-email variant, see Follow-ups.)

## Forbidden words and phrases (AI tells)

Delete on sight. These are the highest-confidence AI-generated markers in 2025:

- impressed, intrigued, fascinated, delighted
- innovative, cutting-edge, game-changing, revolutionary
- streamline, leverage, empower, unlock
- "I hope this email finds you well"
- "I came across your profile"
- "Congrats on the recent [anything]" as the entire opener
- "Quick question" as subject line (most overused in cold email)
- "I'd love to pick your brain"
- "As a fellow [title]"
- "Worth a look?" / "Worth exploring?" / "Worth comparing notes?" / "Worth [anything]?" as a CTA
- "Different angle:" / "A different angle:" / "Different framing:" as an Email 2 opener (telegraphs the template structure)
- "Closing the loop" / "Circling back before I close this out" as a breakup opener
- "We can be live in [X] days/weeks" / "We can deploy in [X]" / any unsolicited speed-to-launch claim in Email 1
- "Most [X] teams…" / "Most [title]s at that growth rate…" as a generalization opener (presumptuous; reads as fabricated)
- **High-frequency AI words** (ship rate 20–48× higher in LLM text than human writing): delve, tapestry, multifaceted, foster, harness, underscore, holistic, transformative, comprehensive, nuanced, pivotal, seamless, robust, utilize, landscape, furthermore, moreover, consequently, notably, particularly
- "It's not X, it's Y" — the single most-recognized LLM construction in 2025 LinkedIn/email detection; delete on sight, including close variants ("This isn't a [X] problem — it's a [Y] problem")
- "It's worth noting" / "It is important to note" / "It's worth mentioning" — replace with the note itself, or cut
- "In today's fast-paced [landscape/world/environment/market]" — AI landscape opener, no human has ever written this unprompted
- "Here's the thing" / "Let that sink in" — performative-blog tells
- Hollow rhetorical questions that answer themselves ("Want better results? Of course you do.") — cut the question, keep the point
- Meta-commentary openers: "The reason I'm reaching out is…" / "I wanted to reach out because…" / "I'm writing to…" / "The purpose of this email is…" — start with the point, not the scaffolding about the email
- "In conclusion" / "The takeaway is" / "To summarize" — cold emails don't need a recap

Also forbidden: starting any sentence with "I" as the literal first word of Email 1.

**Burned opener patterns (structurally, not just by word choice):**

These used to work and are now anti-signals — they tell the reader it's automated, even when the words are different:

- *"I saw you were hiring for [role]..."* — Every tool auto-detects job postings. The hook must imply a problem; this just states a fact.
- *"Congrats on the funding round!"* — Every sequencer sends this. If your opener is indistinguishable from 50 others, delete it.
- *"I noticed you recently [generic milestone]..."* — "Recently" without a date is a tell.
- *"I have an idea for [Company]..."* — Presumptuous before earning credibility.
- AI flattery that references a LinkedIn post or blog article without genuine insight — hallucinates ~15% of the time (invents articles the person never wrote); recipients know.

## Persona calibration

**C-suite / founders (CEO, CTO, CRO, CFO):**
- Under 50 words
- Informal tone (78% higher positive-reply rate vs formal)
- Lead with business outcome, not operational detail
- No links in Email 1
- Timeline hooks specifically

**VPs and directors:**
- 50–80 words
- Connect strategic outcome to a specific operational mechanism (one metric, brief case study reference)
- Can use slightly more detail than C-suite

**Managers and ICs (incl. technical personas):**
- Frame them as internal champions — give them evidence they can bring to their VP
- Lead with the workflow problem they personally feel, not the business outcome their boss cares about
- Replace marketing language with precise operational language (for technical personas)

**Finance personas (CFO, VP Finance):**
- Most poorly written in B2B outbound; A-grade copy lifts reply rate ~79%. Biggest improvement opportunity.
- Numbers, no adjectives. Zero fluff.

## Follow-ups

- **3–4 emails total. No more.** Beyond #4, spam complaints triple and reply rates drop 55%.
- **Default cadence: 3-7-7.** Day 0, Day 3, Day 10, optional Day 17 breakup. Captures ~93% of total replies by Day 10.
- **Never send follow-ups <3 days apart.**
- **Shift the angle every touch.** Different proof point, different framing, different entry into the same problem. Do not repeat Email 1's pitch.
- **Email 3 should make "not right now" an easy reply.** A qualifying question that gives the prospect a graceful off-ramp captures the "interested but not today" bucket. ✓ "Is this on your roadmap for Q2, or just not the right time?" ✓ "Worth the conversation in six months, or should I assume this isn't a fit?" — both invite a one-word answer and close nothing they'd regret closing.
- **"Just circling back" is acceptable exactly once — and only if the next sentence contains something new.**
- **Breakup email: zero pitch, zero guilt, graceful exit.** A breakup that doesn't announce itself reads better than one that does. ✓ "Last note from me. If this ever becomes relevant, I'm easy to reach." ✓ "If the timing shifts, I'm here." Never: "Closing the loop" / "Following up one last time" / "Circling back before I close this out" — these announce the template structure and erase the goodwill the preceding emails tried to build.

## Specificity and claims

- **Precise numbers when verifiable; hedged round numbers when generalizing.**
  - Verifiable: "We reduced [named company]'s cycle from 14 days to 6."
  - Generalized: "Teams similar to yours typically see roughly 30–40% reduction."
- **Extraordinary claim → extraordinary evidence.** Inverse proportion. 5× pipeline claims with no anchor destroy credibility on sight.
- **When no case study exists, use mechanism specificity.** "We parse your dbt DAG to flag the 3 most expensive queries" is credible without a logo parade.

## Social proof

- **One proof element per email. Match it to the recipient's role, company size, industry.**
- **Named person + specific outcome > logo parade.** "We work with Stripe" is meaningless; "We helped Stripe's RevOps cut reconciliation 60%" is something.
- **Do not deploy F500 case studies to SMB prospects, or vice versa.**
- **Proof must bridge to the prospect's specific situation, not just name-drop.** Compare:
  - ✗ "Lyra works with Siemens and BSH." (logo parade — no connective tissue to this prospect)
  - ✓ "Siemens ran a similar Poland-wide restructuring last year; happy to share the manager-support framework they used." (logo + bridge to the prospect's situation)
- **The `proof_shape` field in `icp.json.messaging_angles` is the spec.** A `cost_reduction` angle with `proof_shape: "dollar-denominated outcome from a comparable customer"` means every email on that angle must land one. If the proof doesn't exist for this prospect, change the angle — don't paper over the gap with a generic logo drop.

## Mechanics and locale

- **Em-dashes: cap at 1 per email.** Two or more em-dashes in a 50-word email is an AI-voice tell. Rewrite with periods or commas. The temptation to chain clauses with em-dashes is the single strongest stylistic fingerprint of LLM-generated prose.
- **Currency matches the prospect's region.** Polish prospects see `PLN` or `€`; US see `$`; UK see `£`. Mixing currencies across templates in the same client run is a template-discipline tell — pick the locale at `/icp-define` time and hold it across every email.
- **No unsolicited speed-to-launch claims in Email 1.** "We can be live inside a week" reads like a vendor pitch answering an objection the prospect never raised. Implementation speed belongs in Email 2+ or in a reply, not cold.
- **No SaaS landing-page grammar.** "[Product] is how [Company] does [Thing]" is a homepage construction. In an email it reads as a pitch deck that forgot to change clothes. Use peer-to-peer phrasing: "Siemens' HR team uses us for…" beats "Lyra is how Siemens handles…".

## Rhythm and structure (AI-sameness filters)

The hardest tells to scrub aren't words — they're shape. LLMs default to uniform sentence length (~27 words), identical paragraph blocks, and the "general statement → explanation → hedge → safe conclusion" arc. These patterns survive word-swaps and still trigger "this is AI" on the reader.

- **Vary sentence length deliberately.** A 20-word sentence followed by a 4-word one reads human. Two back-to-back 25-word sentences read like AI even when the vocabulary is clean.
- **Contractions everywhere.** "We've" not "we have"; "you're" not "you are"; "don't" not "do not". AI defaults to expanded forms; humans contract by default.
- **At least one first-person "I" or "we" anchor per email.** Not as the opener word of Email 1 (separate rule), but somewhere in the body. Grounds the message in a specific sender, not a disembodied pitch.
- **Break paragraph symmetry.** Don't write three 2-sentence paragraphs in a row. A one-line paragraph between two longer ones reads human.
- **Cut hedging unless genuinely uncertain.** AI hedges reflexively ("might," "could potentially," "may be worth considering"). Take a stance and let the prospect push back.
- **Opinion beats description.** A claim the sender is willing to defend ("the hardest part usually lands 6–9 months after the hiring surge, not during it") beats a neutral observation ("there can be various challenges during growth periods").

## Humor

- **Humor aimed at self or situation, never the prospect.**
- **Works for creative/marketing/technical buyers; misfires for finance, legal, healthcare compliance.**
- **One beat max.** Sustained humor reads as trying too hard.

## The mobile test

- Open the draft on a phone-width window.
- If the body exceeds one screen without scrolling: too long.
- If the reader needs to zoom or squint at dense paragraphs: reformat.
- If the signature takes more vertical space than the message: trim the signature.

## The human test (final pass)

Read the email aloud.

- **Remove the CTA and re-read.** If the body still delivers something useful to the prospect — a specific observation, a data point, a framing that helps them think — it passes. If the email collapses the moment the ask is gone, it's a pitch wearing friendly clothes. Rewrite the body to stand on its own, or cut the email. This test is falsifiable in a way "does it sound human?" isn't.
- Does it sound like a LinkedIn post? Rewrite.
- Does it sound like a press release? Rewrite.
- Does any sentence exist only to scaffold the pitch (meta-commentary about the email itself, "thought you'd want a look at…")? Delete.
- Would you send this, word for word, to someone you actually want a response from? If not, neither will the client.
