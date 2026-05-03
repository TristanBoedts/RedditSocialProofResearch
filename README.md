# Reddit Social Proof & Misinformation Research Scraper

## What This Does

Collects **census-level time-series data** on Reddit comment scores to study
how visible social metrics (upvotes) influence engagement with information
of varying quality.

The key natural experiment: many subreddits hide comment scores for a
configurable window (5 minutes to 24 hours). This creates a before/after
design where the same comment transitions from score-hidden to score-visible,
enabling regression discontinuity in time (RDiT), interrupted time-series
(ITSA), and difference-in-differences (DiD) analyses.

## How It Works

Each scrape cycle does two things:

1. **Discovers new posts** — checks `/new` on each target subreddit and
   records every post not yet in the database.
2. **Polls tracked posts** — fetches ALL comments on each due post and
   records a score snapshot for every comment.

Polling frequency is throttled at the **post level** (one API call per
post per interval), so the cycle skips posts that were fetched recently
enough. When a post is fetched, all its comments are snapshotted at the
same timestamp — giving consistent time-points for time-series analysis.

| Post age  | Poll interval | Rationale |
|-----------|--------------|-----------|
| < 1 h     | every cycle (~30 s) | Dense coverage of DTI's 5-min hide window |
| 1 h – 2 d | every 60 s  | Full resolution through IAF's 12-h window + first day |
| 2 d – 5 d | every 10 min | Engagement still possible; meteoric rises detectable |
| 5 d – 14 d | every 30 min | Long tail, near-dormant |

This means every post from your collection start date is captured, and
every comment on those posts is tracked with repeated score observations
across the hidden→visible transition.

## Data Schema

```
posts        →  One row per post (title, body, URL, domain, metadata)
comments     →  One row per comment (body text, author, depth, metadata)
snapshots    →  One row per (comment × time) observation
                Contains: score, score_hidden flag, controversiality,
                reply count, parent post metrics
scrape_log   →  Health monitoring for each scrape cycle
```

Raw comment and post text is stored because misinformation classification
requires assessment of content veracity — this cannot be done from
metadata alone. Text will be deleted once classification is complete,
retaining only the derived labels.

## Quick Start

```bash
# Set credentials
export REDDIT_CLIENT_ID="..."
export REDDIT_CLIENT_SECRET="..."

# Install
pip install -r requirements.txt

# Pilot test (scrapes once, then deletes the database)
python scraper.py --pilot

# Single cycle (keeps data)
python scraper.py

# Continuous collection (1-minute intervals)
python scraper.py --daemon

# Continuous collection (5-minute intervals)
python scraper.py --daemon --interval 300

# Export to CSV for analysis
python scraper.py --export
```

## Ethics & Compliance Commands

```bash
# Pseudonymise all usernames (run BEFORE analysis)
# Saves encrypted lookup table for deletion compliance
python scraper.py --anonymise

# Purge all deleted/removed content from the database
# Run periodically to comply with Reddit API terms
python scraper.py --purge-deleted
```

## Research Design Notes

### Target Subreddits

| Subreddit | Score-Hide Duration | Purpose |
|-----------|-------------------|---------|
| r/interestingasfuck | 12 hours | Primary: long hide window |
| r/Damnthatsinteresting | 5 minutes | Control: near-zero window |

Cross-subreddit variation in hide durations is critical for the RDiT
design: it separates the treatment effect (score becoming visible) from
natural comment-ageing trends.

### Key Variables

**Treatment**: `score_hidden` (binary) — 1 during the hide window, 0 after

**Outcomes (per snapshot)**:
- `score` — net upvotes at observation time
- `num_replies` — engagement proxy
- `controversiality` — Reddit's built-in flag

**Derived measures (in analysis)**:
- Score velocity: Δscore / Δtime between consecutive snapshots
- Score acceleration: change in velocity at the hidden→visible transition
- Reply rate: Δreplies / Δtime

### Analysis Strategy

1. **Descriptive analysis** — score trajectory distributions, hidden vs. visible
2. **RDiT** — regression discontinuity at the score-reveal cutoff
3. **ITSA** — segmented regression with breakpoint at transition
4. **DiD** — misinformation vs. verified content, before/after visibility
5. **Misinformation classification** — domain credibility + manual coding
6. **Robustness checks** — temporal controls, threshold sensitivity

### Misinformation Classification (separate pipeline)

The scraper captures `post_url`, `post_domain`, `post_body`, and
`comment_body`. Classification can use:
- Domain credibility: NewsGuard, Media Bias/Fact Check, and similar
- Fact-check databases: Google Fact Check API, ClaimBuster
- Manual coding: for a validation subset

Classification is intentionally decoupled from collection.

## Files

```
scraper.py          Main scraper (collection, export, anonymisation, purge)
requirements.txt    Python dependencies
SETUP_GCP.md        GCP Free Tier deployment guide
README.md           This file
```

## Ethics

This project has WBS ethics approval. Key safeguards:
- Observational only — no intervention, no interaction with users
- Usernames pseudonymised before analysis
- Deleted content purged per Reddit API terms
- Raw text deleted after misinformation classification is complete
- Published data is aggregated and de-identified
- Sensitive subreddits (support, medical, vulnerable populations) excluded
