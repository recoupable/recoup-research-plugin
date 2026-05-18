---
name: recoup-artist-research
description: Full artist research sweep via the Recoup research API. Use when asked to research an artist, get an overview, understand how an artist is performing, or prepare an artist brief. Triggers on "research [artist]", "how is [artist] doing", "tell me about [artist]", "artist overview", "artist brief", or any request for a comprehensive look at an artist's streaming, social, audience, and competitive position. This is the default entry point for all artist research — it orchestrates profile, metrics, audience, playlists, similar artists, and insights into a synthesized brief.
---

# Artist Research

Full-stack artist research through the Recoup API. This skill is the **default
entry point** — it runs the complete research sweep and synthesizes findings
into an actionable brief.

All endpoints live under `https://recoup-api.vercel.app/api/research` and
authenticate with `x-api-key`.

```bash
export RECOUP_API_KEY="recoup_sk_..."   # already set in Recoup sandboxes
export RECOUP_API="https://recoup-api.vercel.app/api"
```

Reference docs: <https://developers.recoupable.com>

## Decision tree

Start here based on what the user asks:

- **"Research [artist]"** → full sweep (below)
- **"How is [artist] doing?"** → `metrics?source=spotify` + `cities` + `insights` → synthesize
- **"Tell me about [artist]"** → `profile` → `insights` → `career` → synthesize
- **"What do we know about [artist]?"** → check workspace `context/artist.md` first; if empty, run full sweep
- **Playlist questions** → hand off to `recoup-playlist-intelligence`
- **Audience/market questions** → hand off to `recoup-audience-analysis`
- **Competitor/roster comparison** → hand off to `recoup-competitive-analysis`
- **Trending/discovery questions** → hand off to `recoup-trend-detection`
- **People/outreach questions** → hand off to `recoup-people-outreach`

Before researching: check if the artist already has a workspace
`context/artist.md` — don't re-research what's known.

## Full research sweep

Run these in parallel where possible, then synthesize:

```bash
# 1. Search — resolve the artist name to a Chartmetric ID
curl -s "$RECOUP_API/research?q={ARTIST}&type=artists&beta=true" \
  -H "x-api-key: $RECOUP_API_KEY" | jq

# 2. Profile — bio, genres, social URLs, label, career stage
curl -s "$RECOUP_API/research/profile?artist={ARTIST}" \
  -H "x-api-key: $RECOUP_API_KEY" | jq

# 3. Spotify metrics — followers, listeners, popularity trend
curl -s "$RECOUP_API/research/metrics?artist={ARTIST}&source=spotify" \
  -H "x-api-key: $RECOUP_API_KEY" | jq

# 4. Top listener cities
curl -s "$RECOUP_API/research/cities?artist={ARTIST}" \
  -H "x-api-key: $RECOUP_API_KEY" | jq

# 5. Similar artists — competitive landscape
curl -s "$RECOUP_API/research/similar?artist={ARTIST}&audience=high&genre=high&limit=20" \
  -H "x-api-key: $RECOUP_API_KEY" | jq

# 6. Playlist placements — editorial + algorithmic reach
curl -s "$RECOUP_API/research/playlists?artist={ARTIST}&sort=followers" \
  -H "x-api-key: $RECOUP_API_KEY" | jq

# 7. Audience demographics — IG or TikTok
curl -s "$RECOUP_API/research/audience?artist={ARTIST}&platform=instagram" \
  -H "x-api-key: $RECOUP_API_KEY" | jq

# 8. AI-surfaced observations
curl -s "$RECOUP_API/research/insights?artist={ARTIST}" \
  -H "x-api-key: $RECOUP_API_KEY" | jq

# 9. Social + streaming URLs
curl -s "$RECOUP_API/research/urls?artist={ARTIST}" \
  -H "x-api-key: $RECOUP_API_KEY" | jq
```

## Synthesis template

After gathering data, synthesize into this structure:

### Artist Brief: {Name}

**Career Stage:** {from similar or profile} | **Momentum:** {recent_momentum}
**Global Rank:** {from rank endpoint} | **Label:** {from profile}

**Streaming Snapshot:**
- Spotify: {monthly_listeners} listeners / {followers} followers ({follower:listener ratio}%)
- TikTok: {followers} followers
- Instagram: {followers} followers
- YouTube: {subscribers} subscribers

**Geographic Hotspots:** {top 5 cities from /cities}

**Audience Profile:** {age/gender breakdown from /audience}

**Playlist Position:**
- {count} editorial placements, {count} total
- Total playlist reach: {sp_playlist_total_reach from profile}
- Notable placements: {top 3 by reach}

**Competitive Position:** {career stage vs similar artists, notable gaps or strengths}

**Key Insights:** {from /insights endpoint}

**Recommendations:** {synthesized from all data — what to do next}

## Critical gotchas

These failure modes will eat your time:

- **Search: `match_strength < 1` = not found.** Real matches score 100s–50,000s; noise is 0.005–0.1. Don't pass sub-1 IDs into detail endpoints.
- **`/research/profile` returns `null` for many fields on less-covered artists.** Fall back to individual endpoints — they hit platform data directly.
- **`/research/metrics` uses `youtube_channel` or `youtube_artist`**, not plain `youtube`.
- **`/research/audience?platform=` accepts only `instagram | tiktok | youtube`.**
- **For URLs, route through `/research/lookup?url=` first.** `/profile?artist=<URL>` works sometimes but 404/406s for others.
- **POST endpoints have real latency.** `/enrich` 60–90s, `/deep` 2+ min. Set client timeouts to ≥3 min.
- **Don't guess field names.** `recent_momentum` not `trend`; platform counts are top-level (`sp_followers`, `ins_followers`), no `metrics` wrapper.

## Graceful degradation

If Chartmetric data is unavailable (search returns empty, match_strength < 1,
or lookup fails):

1. `POST /research/web` — ranked web results
2. `POST /research/enrich` — structured facts (~60–90s)
3. `POST /research/deep` — cited narrative (~2+ min)

For very emerging artists, Chartmetric may not have data — web + enrich + deep
is the fallback. See `recoup-web-intelligence` skill for details.

## Saving research

If working in an artist workspace, save research results to `research/` with
timestamps:

```
research/artist-intel-2026-05-17.md
```

Don't overwrite `context/artist.md` with research data. Static context (who the
artist IS) is separate from dynamic research (how they're performing NOW). If
the research reveals something that should update the static profile, suggest it
— don't auto-update.

## Endpoint quick reference

Full curl examples, filter rules, latency budgets, and platform source enums
are in **[references/endpoints.md](../../references/endpoints.md)**.

| Endpoint | Returns |
| -------- | ------- |
| `GET /research?q=&type=&beta=true` | search → Chartmetric IDs |
| `GET /research/profile?artist=` | bio, label, genres, aggregate counts |
| `GET /research/metrics?artist=&source=` | platform time-series |
| `GET /research/audience?artist=&platform=` | age/gender/country |
| `GET /research/cities?artist=` | top listener cities |
| `GET /research/similar?artist=` | peer artists |
| `GET /research/playlists?artist=` | placements |
| `GET /research/tracks?artist=` | catalog |
| `GET /research/career?artist=` | career timeline |
| `GET /research/insights?artist=` | AI observations |
| `GET /research/milestones?artist=` | activity feed |
| `GET /research/urls?artist=` | social/streaming URLs |
| `GET /research/rank?artist=` | global rank (single int) |
| `GET /research/instagram-posts?artist=` | top IG posts |
| `GET /research/venues?artist=` | venue history |
| `GET /research/lookup?url=` | URL → artist resolution |

## References

- **[references/endpoints.md](../../references/endpoints.md)** — full curl examples, pagination, latency
- **[references/response-shapes.md](../../references/response-shapes.md)** — actual JSON shapes
- **[references/workflows.md](../../references/workflows.md)** — multi-step workflow chains
