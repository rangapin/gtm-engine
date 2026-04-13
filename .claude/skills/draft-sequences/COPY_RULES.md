# Copy rules — cold B2B outbound

Read this every time you draft or lint a sequence. Rules are imperative. Examples are deliberately short.

**Tensions to hold:** several of these rules conflict with each other in edge cases (e.g., permission-based CTAs read weak to founders; humor misfires with risk-averse buyers). Resolve using the prospect's persona + the client's voice, not by picking a rule and ignoring the rest. When rules clash, note the tradeoff to the user at the review gate.

Full research this distills: see `COPY_RULES_SOURCES.md`.

---

## Length

- **Target 50–80 words per email. Hard ceiling 100.** 2.4× reply lift vs 200+ word emails.
- **First-touch: 25–50 words if possible.** The fewer words, the stronger each must be.
- **Write the email, cut it in half, cut it again, ship what survives.**
- **Read it on your phone before sending. If it scrolls, it's too long.**
- **Last email of a sequence: 2–3 sentences.**

## Subject lines

- **Lowercase, 1–5 words.** Title case signals "marketing email."
- **Use prospect-specific context, not your category.** `snowflake → bigquery?` ✓ / `Data integration solution` ✗
- **Questions are allowed and lift opens ~21%.** Use when the body pays off the curiosity.
- **Never use fake `Re:` or `Fwd:` prefixes.** Short-term open bump; long-term trust + domain reputation damage.
- **Follow-ups inherit the subject with `Re: ` or shift to a fresh angle-matched subject.** Do not resurrect Email 1's subject verbatim on Email 3.

## Structure and format

- **Plain text only.** No HTML, no bullet points, no bold, no colors, no images. Humans don't format personal email.
- **Short paragraphs (1–2 sentences each) with white space between.**
- **5th-grade reading level.** A-grade emails (per Lavender) at that level lift reply rates 67%.
- **Three-paragraph "signal → pitch → CTA" skeleton is burned.** Recipients and spam filters now pattern-match it as template. Alternatives:
  - Two-sentence email (observation + question)
  - Question-only email (second touch onward)
  - Insight-first email (non-obvious observation, no pitch)
- **Sign-off format: `<first_name>\n<role> @ <domain>`.** Role+domain on second line, not a bare first name (reads as mass-email).

## Openers

- **Earn relevance in the first 7 words.** The recipient's keep/delete decision happens in the preview pane.
- **Timeline hooks (recent event, ≤30 days old) outperform problem hooks 2.3×.** Funding round, acquisition, product launch, job posting, earnings-call mention, leadership change.
- **The signal must imply a problem, not state a fact.** "You raised Series B — congrats" ✗ / "After your Series B you're probably hiring 3 SDRs in Q2; onboarding them breaks the playbook that worked at 2" ✓
- **Permission-ask openers ("not sure if this is relevant, but…") work when you lack a strong signal.** They underperform for C-suite where directness signals respect for time.
- **Pattern-interrupt openers (humor, unexpected observation) must connect to the pitch within 2 sentences, or they become gimmicks.**
- **Direct-premise openers work when your brand carries weight or the prospect's need is obvious.**

## The CTA

- **Ask permission; don't assume interest.** "Worth a 10-min look?" ✓ / "Let's get 30 min on the calendar" ✗
- **One CTA per email.** Any additional ask dilutes the primary one.
- **Frame around what the prospect is losing, not what you're offering.** Loss aversion outpulls promise-of-gain.
- **Interest CTAs beat meeting CTAs on first touch.** "Relevant?" / "Worth exploring?" / "Want me to send the teardown?"
- **No calendar links in Email 1.** Links are spam signals and create friction before the prospect has decided they care.

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

Also forbidden: starting any sentence with "I" as the literal first word of Email 1.

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
- **"Just circling back" is acceptable exactly once — and only if the next sentence contains something new.**
- **Breakup email: zero pitch, zero guilt, graceful exit.** "Closing the loop — if this ever becomes relevant, my inbox is open." ✓

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

- Does it sound like a LinkedIn post? Rewrite.
- Does it sound like a press release? Rewrite.
- Does any sentence exist only to scaffold the pitch (meta-commentary about the email itself, "thought you'd want a look at…")? Delete.
- Would you send this, word for word, to someone you actually want a response from? If not, neither will the client.
