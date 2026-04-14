capture-results: I assumed channel=email because the user confirmed (activate log ambiguous — mentioned gmail, heyreach, smartlead; actual mode was Gmail drafts which user clarified as "email in draft").
capture-results: I excluded 1 non-DQ row (C4 Carbon) because person_name was blank — no contact surfaced by Clay, so no outcome to track.
gather-signals: For molchanovs.com, I tagged hiring_surge as high because LinkedIn shows +36% YoY headcount (52 employees) — exceeds 20% threshold.
gather-signals: For molchanovs.com, I tagged "Beginner Online Freediving Course" as product_launch high because launch date (Jan 2026) is within 90 days of detection.
gather-signals: For alchemy.gr, I did NOT emit a hiring_surge signal because LinkedIn shows -20% YoY (3 employees, shrinking). Negative growth isn't a "signal" in the positive sense.
gather-signals: For alchemy.gr content_velocity high, I based it on 5 posts in 7 days (Mar 23-30 2026) which looks like a cadence jump vs prior months, but I don't have baseline cadence data to confirm ≥2× — the call is directional.
gather-signals: For c4carbon.com, I used linkedin_company_page as source but LinkedIn follower growth (+53% YoY) is a community_signal, not hiring_surge — headcount is only 4 and stable.
gather-signals: Signals with detected_at >180 days (Alchemy Aurora fins, Alchemy Association Oceania partnership, C4 Best Choice award, etc.) were forced to confidence=low per the global decay rule.
gather-signals: Used prospects.enriched.csv instead of prospects.csv/seed.csv for smoke test because the seed file has 20 companies (~80 min full scan). Scoped to the 3 non-DQ domains in enriched.csv. One-time deviation for smoke testing.
icp-define: angles retro-filled on 2026-04-14 with [audience_quality, channel_diversification, community_authenticity] — customized for theline's sponsorship-sales offer (SaaS defaults didn't fit). Review before next draft-sequences run.
