---
name: recoup-roster-health
description: Scans an entire artist roster and produces a portfolio-level health report — ranked by growth velocity, flagging at-risk artists, surfacing opportunities, and measuring concentration risk. Use when asked "how is my roster doing", "roster health check", "portfolio overview", "which artists need attention", "roster report", "label health check", "who's growing fastest", or any request to compare or rank multiple artists at once. Outputs a single markdown report that replaces 20+ individual lookups with one portfolio-level view.
---

# Roster Health Check

A portfolio-level scan across your entire roster. The skill exists because
labels and managers don't think artist-by-artist — they think portfolio.
Running `recoup-artist-research` on 20 artists individually produces 20
unconnected documents nobody compares. This skill produces ONE ranked,
cross-referenced health report.

## When to use

- "How is my roster doing" / "Roster health check"
- "Portfolio overview" / "Label health check"
- "Which artists need attention" / "Who's growing fastest"
- "Roster report" / "Compare my artists"
- "Rank my artists by growth" / "Who should I focus on"
- Any request that involves comparing or evaluating multiple artists at once

For single-artist deep dives, use `recoup-artist-research`.
For recurring single-artist tracking, use `recoup-weekly-brief`.

## Setup

```bash
export RECOUP_API_KEY="recoup_sk_..."
export RECOUP_API="https://api.recoupable.com/api"
```

## Resolving the roster

The skill needs a list of artists. Resolve it in this order:

1. **Workspace scan** (preferred): List all directories under `artists/` that
   contain a `RECOUP.md` or `context/artist.md` file. Extract `cmArtistId`
   from frontmatter when present.
2. **Explicit list**: The user provides artist names directly ("scan Drake,
   Bad Bunny, and Peso Pluma").
3. **Org query**: If the user says "my roster" and no workspace folders exist,
   ask them to list their artists or point to a CSV/text file.

If the roster has **more than 30 artists**, warn the user about credit cost
(see Credit awareness below) and ask whether to proceed with the full list or
a subset (e.g., top 15 by recent activity).

## Workflow

### 1. Resolve artist IDs

For each artist without a `cmArtistId`, resolve via search:

```bash
curl -s "$RECOUP_API/research?q={ARTIST}&type=artists&beta=true" \
  -H "x-api-key: $RECOUP_API_KEY"
```

Use the top result with `match_strength >= 1`. Skip unresolvable artists and
note them in the report's "Unresolved" section.

### 2. Fan out metrics per artist (batch parallel)

For each artist, fire 3 endpoints in parallel:

```bash
# Spotify metrics — listeners, followers, popularity
curl -s "$RECOUP_API/research/metrics?artist={ARTIST}&source=spotify" \
  -H "x-api-key: $RECOUP_API_KEY" &

# TikTok metrics — followers, posts, views
curl -s "$RECOUP_API/research/metrics?artist={ARTIST}&source=tiktok" \
  -H "x-api-key: $RECOUP_API_KEY" &

# Global rank
curl -s "$RECOUP_API/research/rank?artist={ARTIST}" \
  -H "x-api-key: $RECOUP_API_KEY" &

wait
```

Process artists in batches of 5 to avoid overwhelming the API. Wait for each
batch to complete before starting the next.

### 3. Compute health scores

For each artist, compute a composite health score (0–100) from:

| Signal | Weight | How to compute |
|--------|--------|----------------|
| **Spotify listener velocity** | 30% | Latest monthly listeners vs. 30-day-ago value from time series. Positive Δ% = healthy. |
| **Spotify follower growth** | 15% | Latest followers vs. 30-day-ago. Followers grow slower than listeners — even small positive Δ is good. |
| **TikTok activity** | 20% | Latest TT followers vs. 30-day-ago. TikTok is the leading indicator for streaming growth. |
| **Global rank** | 15% | Lower rank = better. Rank improving = healthy. If null, use 50 as neutral score. |
| **Popularity index** | 20% | Spotify popularity (0–100). Above 50 = mainstream traction. |

Scoring per signal:
- **Strong growth** (>10% monthly gain): 90–100
- **Healthy growth** (2–10% gain): 70–89
- **Stable** (±2%): 50–69
- **Declining** (2–10% loss): 30–49
- **At risk** (<10% loss): 0–29
- **No data**: 50 (neutral, noted as "thin data")

Composite = weighted average of all signal scores.

### 4. Classify artists into tiers

| Tier | Composite score | Emoji | Meaning |
|------|----------------|-------|---------|
| 🟢 Growing | 70–100 | 🟢 | Active growth — maintain momentum |
| 🟡 Stable | 50–69 | 🟡 | Holding steady — look for catalysts |
| 🟠 Watch | 30–49 | 🟠 | Declining trends — needs intervention |
| 🔴 At risk | 0–29 | 🔴 | Significant decline — urgent attention |

### 5. Compute portfolio-level metrics

- **Portfolio health score**: Average of all artist composites
- **Growth leaders** (top 3 by velocity): Who's carrying the roster
- **Concentration risk**: What % of total streams come from top artist? >50% = concentrated
- **At-risk count**: How many artists are 🟠 or 🔴
- **TikTok/streaming correlation**: Artists with TikTok growth but flat streaming = content opportunity

### 6. Write the report

Path: `reports/roster-health-$(date +%F).md`

If no `reports/` directory exists, create it. If today's report already exists,
**stop** — tell the user it already exists.

Template:

```markdown
# Roster Health Report
**Date:** {YYYY-MM-DD} | **Artists scanned:** {N} | **Portfolio health:** {score}/100

## TL;DR
{3–4 sentences. Portfolio-level summary: overall trajectory, biggest movers,
biggest risks, one recommended action. Specific numbers, not vibes.}

## Portfolio Summary

| Metric | Value |
|--------|-------|
| Artists scanned | {N} |
| Portfolio health score | {score}/100 |
| 🟢 Growing | {N} |
| 🟡 Stable | {N} |
| 🟠 Watch | {N} |
| 🔴 At risk | {N} |
| Concentration risk | {top artist}% of total listeners |

## Artist Rankings

| Rank | Artist | Health | Spotify Listeners | Δ 30d | TikTok | Δ 30d | Global Rank | Score |
|------|--------|--------|------------------|-------|--------|-------|-------------|-------|
| 1 | {name} | 🟢 | {N} | {±N%} | {N} | {±N%} | {rank} | {score} |
| 2 | ... | ... | ... | ... | ... | ... | ... | ... |
| ... | ... | ... | ... | ... | ... | ... | ... | ... |

## 🟢 Growth Leaders
{Top 3 artists by velocity. For each: what's driving growth (playlist adds,
TikTok viral moment, release momentum), and what to do to maintain it.}

## 🔴 Needs Attention
{Artists scoring 🟠 or 🔴. For each: what's declining, possible causes
(no recent releases, lost playlist placements, social dormancy), and one
specific recommended action.}

## Concentration Risk
{Analysis of how concentrated the portfolio is. If top artist represents
>50% of total listeners, flag it. Suggest diversification strategies if
applicable.}

## Opportunities
{Cross-roster patterns: artists with TikTok momentum that hasn't translated
to streaming yet, artists gaining playlists but not posting content,
complementary collaborations within the roster.}

## Unresolved Artists
{Any artists from the input list that couldn't be matched to Chartmetric IDs.
Note the search query used and suggest corrections.}

---
*Generated {ISO timestamp}. Source: Recoup research API.
Re-run with "roster health check".*
```

### 7. Print a chat summary

Always print a condensed version in chat — the user shouldn't have to open the
file to get the headlines:

- Portfolio health score
- Top 3 growth leaders (name + Δ%)
- Any 🔴 at-risk artists (name + issue)
- One recommended action

## What this skill refuses to do

- **No data invention.** If metrics return null for an artist, score them as
  "thin data" (50/neutral) and note it. Never invent listener counts.
- **No silent overwrites.** Same-day re-run is a no-op.
- **No individual deep dives.** This is a portfolio scanner. If the user wants
  a single-artist deep dive after seeing the rankings, point them to
  `recoup-artist-research`.
- **No financial projections.** Health scores measure streaming/social velocity,
  not revenue. Don't extrapolate to dollar amounts — that's the catalogs
  plugin's domain.

## Credit awareness

Each artist costs ~3 credits (Spotify metrics + TikTok metrics + rank).

| Roster size | Estimated credits |
|-------------|------------------|
| 5 artists | ~15 credits |
| 10 artists | ~30 credits |
| 20 artists | ~60 credits |
| 30 artists | ~90 credits |

For rosters >30 artists, warn the user before proceeding.

## Graceful degradation

- **Thin Chartmetric data** (emerging/unsigned artists): Score as 50/neutral,
  flag in the rankings table with "(thin data)". Don't drop them from the report.
- **TikTok metrics unavailable**: Score TikTok signal as 50/neutral, note in
  the artist row. Don't fail the whole report because one platform is missing.
- **Partial roster failure**: If some artists fail to resolve but others succeed,
  produce the report for the successful ones and list failures in the
  Unresolved section.

## References

- `references/endpoints.md` — full curl examples
- `references/response-shapes.md` — JSON shapes per endpoint
- `recoup-artist-research` — single-artist deep dive (use after roster scan
  to drill into specific artists)
- `recoup-weekly-brief` — set up recurring tracking for artists that need
  ongoing attention
