---
name: recoup-signing-package
description: Produces a comprehensive signing evaluation package for an artist under consideration — market position, audience quality, competitive landscape, content potential, growth trajectory, risk factors, and a structured recommendation framework. Use when asked "signing package for {artist}", "should we sign {artist}", "evaluate {artist} for signing", "signing eval", "artist evaluation", "is {artist} worth signing", "due diligence on {artist}", or any A&R signing decision request. The output is a decision-grade document an A&R team or executive can present at a signing meeting — not raw research.
---

# Signing Package

A decision-grade artist evaluation for signing consideration. The skill exists
because A&R teams don't need another data dump — they need a structured
argument for or against investing in an artist, grounded in real data, with
risks surfaced honestly.

This is NOT `recoup-artist-research` with a different title. Artist research
answers "what's happening with this artist." A signing package answers "should
we bet money on this artist, and if so, how much and under what terms?"

## When to use

- "Signing package for {artist}" / "Evaluate {artist} for signing"
- "Should we sign {artist}" / "Is {artist} worth signing"
- "Signing eval for {artist}" / "Artist evaluation for {artist}"
- "Due diligence on {artist}" / "A&R package for {artist}"
- "Prep a signing memo for {artist}" / "Build a case for signing {artist}"
- Any A&R decision that involves committing capital to an artist

For general research without the signing lens, use `recoup-artist-research`.
For ongoing tracking of an already-signed artist, use `recoup-weekly-brief`.
For pre-release marketing planning, use `recoup-release-pack`.

## Setup

```bash
export RECOUP_API_KEY="recoup_sk_..."
export RECOUP_API="https://api.recoupable.com/api"
```

## What you need from the user

Before fetching data, confirm:

1. **Artist name** (or Chartmetric ID / workspace slug)
2. **Deal type being considered** — recording, publishing, distribution, 360,
   joint venture, or "exploring" (if not sure yet)
3. **Label/company context** — genre focus, roster size, typical deal range
   (optional but improves the competitive landscape analysis)

If #1 is missing, ask. #2 and #3 improve the output but aren't blockers —
default to "recording deal, general evaluation" if not provided.

## Workflow

### 1. Resolve the artist

Prefer `cmArtistId` from workspace frontmatter. Otherwise:

```bash
curl -s "$RECOUP_API/research?q={ARTIST}&type=artists&beta=true" \
  -H "x-api-key: $RECOUP_API_KEY"
```

Use top result with `match_strength >= 1`. On ambiguous matches, surface the
options and ask the user to confirm.

### 2. Fan out all endpoints (parallel)

Pull everything available for this artist in one pass:

```bash
# Spotify metrics — listeners, followers, popularity time series
curl -s "$RECOUP_API/research/metrics?artist={ARTIST}&source=spotify" \
  -H "x-api-key: $RECOUP_API_KEY" &

# TikTok metrics — followers, posts, engagement
curl -s "$RECOUP_API/research/metrics?artist={ARTIST}&source=tiktok" \
  -H "x-api-key: $RECOUP_API_KEY" &

# Instagram metrics
curl -s "$RECOUP_API/research/metrics?artist={ARTIST}&source=instagram" \
  -H "x-api-key: $RECOUP_API_KEY" &

# YouTube metrics
curl -s "$RECOUP_API/research/metrics?artist={ARTIST}&source=youtube_channel" \
  -H "x-api-key: $RECOUP_API_KEY" &

# Global rank
curl -s "$RECOUP_API/research/rank?artist={ARTIST}" \
  -H "x-api-key: $RECOUP_API_KEY" &

# Audience demographics (TikTok + Instagram)
curl -s "$RECOUP_API/research/audience?artist={ARTIST}&platform=tiktok" \
  -H "x-api-key: $RECOUP_API_KEY" &

curl -s "$RECOUP_API/research/audience?artist={ARTIST}&platform=instagram" \
  -H "x-api-key: $RECOUP_API_KEY" &

# Top cities
curl -s "$RECOUP_API/research/cities?artist={ARTIST}" \
  -H "x-api-key: $RECOUP_API_KEY" &

# Similar artists (for competitive landscape)
curl -s "$RECOUP_API/research/similar?artist={ARTIST}&audience=high&genre=high&limit=20" \
  -H "x-api-key: $RECOUP_API_KEY" &

# Playlist position
curl -s "$RECOUP_API/research/playlists?artist={ARTIST}&sort=followers" \
  -H "x-api-key: $RECOUP_API_KEY" &

# Activity feed / milestones
curl -s "$RECOUP_API/research/milestones?artist={ARTIST}" \
  -H "x-api-key: $RECOUP_API_KEY" &

# AI-surfaced insights
curl -s "$RECOUP_API/research/insights?artist={ARTIST}" \
  -H "x-api-key: $RECOUP_API_KEY" &

wait
```

12 calls, all parallel. Total wall time ≈ slowest single call.

### 3. Analyze through the signing lens

This is where the skill diverges from raw research. Process the data through
these evaluation dimensions:

**A. Growth trajectory (is the line going up?)**
- Spotify listener 30/60/90-day trend
- TikTok follower velocity
- Rank movement direction
- Release cadence (milestones feed)

**B. Audience quality (are they real fans or passive listeners?)**
- Follower-to-listener ratio (>5% = engaged fanbase)
- Instagram engagement vs. following
- TikTok engagement patterns
- Geographic concentration (diverse = healthier, concentrated = riskier)

**C. Market position (where do they sit?)**
- Rank vs. similar artists
- Playlist coverage vs. peers (gap = upside, parity = harder to grow)
- Social following vs. streaming (mismatch = content opportunity or
  streaming risk depending on direction)

**D. Content potential (can we build around them?)**
- TikTok-to-Spotify pipeline health
- Content creation velocity (are they posting?)
- Visual/brand distinctiveness (from insights)
- Collaboration network potential (from similar artists)

**E. Risk factors (what could go wrong?)**
- Single-platform dependency (>70% from one source)
- One-hit-wonder signal (one track drives >80% of streams)
- Declining trajectory despite recent release
- Thin social presence relative to streams (bot risk)
- Genre/trend dependency (is the wave passing?)

### 4. Build the recommendation framework

Classify the opportunity:

| Signal | Score | Meaning |
|--------|-------|---------|
| 🟢 Strong sign | 80–100 | Clear growth, quality audience, low risk |
| 🟡 Conditional sign | 60–79 | Opportunity exists but conditions apply |
| 🟠 Development deal | 40–59 | Potential but needs investment / proving |
| 🔴 Pass / watch | 0–39 | Risk outweighs opportunity at current trajectory |

The recommendation should be ONE of:
- **Sign** — data supports the investment at the discussed deal level
- **Conditional sign** — data supports signing IF specific conditions are met
  (name the conditions)
- **Development deal** — the artist shows promise but isn't ready for a full
  deal. Recommend lighter terms with defined promotion triggers.
- **Pass** — data doesn't support the investment. Be specific about why.
- **Watch** — too early to evaluate. Set a specific check-back date and the
  metrics that would change the recommendation.

### 5. Write the package

Path: `artists/{slug}/signing/package-$(date +%F).md`

Create the `signing/` subdirectory under the artist workspace if it doesn't
exist. If no workspace exists, write to `signing-packages/{slug}-$(date +%F).md`.

Same-day re-run is a no-op.

Template:

```markdown
# Signing Package — {Artist Name}
**Prepared:** {YYYY-MM-DD} | **Deal type:** {recording|publishing|distribution|360|JV|exploring}
**Recommendation:** {🟢|🟡|🟠|🔴} {Sign|Conditional sign|Development deal|Pass|Watch}

## Executive Summary
{4–6 sentences. The argument for or against, with specific numbers. This
should be readable in 30 seconds and sufficient for an exec who won't read
the rest.}

## Artist Snapshot

| Metric | Current | 30d Δ | 90d Δ |
|--------|---------|-------|-------|
| Spotify monthly listeners | {N} | {±N%} | {±N%} |
| Spotify followers | {N} | {±N%} | {±N%} |
| TikTok followers | {N} | {±N%} | {±N%} |
| Instagram followers | {N} | {±N%} | {±N%} |
| YouTube subscribers | {N} | {±N%} | {±N%} |
| Global rank | {N} | {±N} | {±N} |
| Spotify popularity | {N}/100 | — | — |

## Growth Trajectory
{Analysis of where the artist is heading. Use the time-series data to
identify the shape: accelerating, linear, plateauing, declining. Reference
specific data points, not feelings. Include a note on release cadence —
are they actively putting out music?}

## Audience Analysis

### Demographics
| Platform | Top age group | Top gender | Top geo |
|----------|--------------|------------|---------|
| TikTok | {age} | {gender} | {country/city} |
| Instagram | {age} | {gender} | {country/city} |

### Audience Quality Indicators
- **Follower:listener ratio:** {N}% ({assessment})
- **Geographic diversity:** {top 3 cities with %} — {concentrated|diverse}
- **Platform balance:** {assessment of cross-platform presence}

### Top Markets
| City | Listeners | % of total |
|------|-----------|------------|
| {city 1} | {N} | {N%} |
| {city 2} | {N} | {N%} |
| {city 3} | {N} | {N%} |

Tour viability: {assessment based on geographic concentration and venue sizes
for those markets}

## Competitive Landscape

### Similar Artists (by audience overlap + genre)
| Artist | Spotify Listeners | Rank | Signed to | Notes |
|--------|------------------|------|-----------|-------|
| {peer 1} | {N} | {N} | {label} | {relevant comparison} |
| {peer 2} | {N} | {N} | {label} | {relevant comparison} |
| ... | ... | ... | ... | ... |

### Market Positioning
{Where does this artist sit relative to peers? Are they leading or trailing
the pack? Is the competitive set signed to major/indie/unsigned? What does
the peer landscape tell us about deal expectations?}

## Playlist Position

### Current Placements
| Playlist | Followers | Type |
|----------|-----------|------|
| {playlist 1} | {N} | {editorial|algorithmic|user} |
| ... | ... | ... |

### Playlist Gap Analysis
{Playlists where similar artists appear but this artist doesn't. This is
the upside — placements a label's playlist team could target.}

## Content & Social Potential
{Assessment of the artist's content game: posting frequency, TikTok
engagement, visual identity, brand distinctiveness. Is there a
TikTok-to-Spotify pipeline working? Content opportunity if not.}

## Risk Factors

{List each identified risk with severity and mitigation:}

| Risk | Severity | Evidence | Mitigation |
|------|----------|----------|------------|
| {risk 1} | 🔴 High / 🟡 Medium / 🟢 Low | {specific data} | {what signing team would need to do} |
| ... | ... | ... | ... |

## Opportunity Assessment
{What's the specific upside? Not generic "could grow" — cite the data.
If TikTok is growing 15%/month but Spotify is flat, the opportunity is
content strategy to convert social to streams. If playlists are strong
but social is weak, the opportunity is social investment. Be specific.}

## Recommendation

**{🟢|🟡|🟠|🔴} {Sign|Conditional sign|Development deal|Pass|Watch}**

{3–5 sentences restating the core argument with the key numbers.}

{If conditional or development deal, list the specific conditions:}
- Condition 1: {what needs to happen, by when}
- Condition 2: ...

{If pass or watch, state what would change the recommendation:}
- Reconsider if: {specific metric threshold, e.g., "Spotify listeners
  sustain above 500K for 3 consecutive months"}

## Comps for Deal Sizing
{If deal type was specified, reference the similar artists' label deals
(from the competitive landscape) as comps. If data isn't available, note
"deal term comps not available from public data — check internal
databases." Never invent deal terms.}

---
*Generated {ISO timestamp}. Source: Recoup research API.
Re-run with "signing package for {artist}".*
```

### 6. Print a chat summary

In chat, always print:
- The recommendation (emoji + verdict)
- The executive summary (verbatim from the report)
- Top 3 risk factors
- One-line on where the full package is saved

## What this skill refuses to do

- **No deal term invention.** If you don't have actual comp data for deal
  sizing, say so. Never invent advance amounts, royalty splits, or contract
  terms. The package provides the analytical foundation — deal terms are
  negotiated by humans.
- **No false confidence.** If the data is thin (emerging artist with <10K
  listeners), the recommendation must reflect that uncertainty. A 🟡 or 🟠
  with honest caveats is better than a 🟢 that overstates the evidence.
- **No sentiment analysis as a substitute for data.** "The artist has a really
  authentic vibe" is not analysis. Ground every claim in a number from the API.
- **No silent data gaps.** If an endpoint returns empty, note it in the relevant
  section with "(no data available)" and explain what it means for the
  assessment (e.g., "no TikTok data — can't assess social-to-streaming pipeline").
- **No overwrites.** Same-day re-run is a no-op.

## Credit awareness

12 parallel calls at ~1 credit each = ~12 credits per signing package.
This is a high-value use case — 12 credits to replace hours of manual research
is excellent ROI. No need to warn the user unless they're on a very low credit
plan.

## Graceful degradation

- **Emerging artist (thin data):** Produce the package but shift the
  recommendation toward 🟠 Development deal or 🟡 Watch. Note which sections
  have thin data. An honest thin-data package is more useful than refusing to
  produce one.
- **No TikTok presence:** Skip TikTok sections, note the gap, assess whether
  this is a risk factor (for a pop/hip-hop artist, yes; for a jazz artist,
  maybe not).
- **No audience demographics:** Skip the demographics table, note it. Audience
  quality assessment falls back to follower:listener ratio only.
- **Artist not in Chartmetric:** If the artist can't be resolved at all, tell
  the user. Suggest alternative spellings, or offer to run
  `recoup-web-intelligence` as a fallback for a lighter evaluation.

## References

- `references/endpoints.md` — full curl examples
- `references/response-shapes.md` — JSON shapes per endpoint
- `recoup-artist-research` — one-shot research (this skill builds on top of it)
- `recoup-competitive-analysis` — deeper head-to-head (use after the package
  to drill into specific comparisons)
- `recoup-audience-analysis` — deeper audience analysis (use after the package
  for geographic/demo deep dives)
- `recoup-playlist-intelligence` — playlist pitching (use to act on the gap
  analysis from this package)
