# Reddit Social Proof Scraper — GCP Free Tier Deployment

## 1. Reddit API Credentials

1. Go to https://www.reddit.com/prefs/apps/
2. Click **"create another app"**
3. Fill in:
   - **Name**: SocialProofResearch
   - **Type**: `script`
   - **Redirect URI**: `http://localhost:8080`
   - **Description**: Academic research on social proof and information quality
4. Note your **client_id** (under the app name) and **client_secret**

## 2. Create a GCP Free Tier VM

```bash
# Create project
gcloud projects create social-proof-research --name="Reddit UpvoteDownvote"
gcloud config set project social-proof-research

# Enable Compute Engine
gcloud services enable compute.googleapis.com

# Create e2-micro VM (free tier: us-central1, us-east1, or us-west1)
gcloud compute instances create reddit-scraper \
    --zone=us-central1-a \
    --machine-type=e2-micro \
    --image-family=ubuntu-2404-lts-amd64 \
    --image-project=ubuntu-os-cloud \
    --boot-disk-size=30GB \
    --boot-disk-type=pd-standard
```

## 3. Set Up the VM

```bash
gcloud compute ssh reddit-scraper --zone=us-central1-a

sudo apt update && sudo apt install -y python3 python3-pip python3-venv

mkdir -p ~/scraper && cd ~/scraper
python3 -m venv venv
source venv/bin/activate

# Upload files (from local machine):
# gcloud compute scp ./scraper.py reddit-scraper:~/scraper/ --zone=us-central1-a
# gcloud compute scp ./requirements.txt reddit-scraper:~/scraper/ --zone=us-central1-a

pip install -r requirements.txt
```

## 4. Configure Credentials

```bash
cat >> ~/.bashrc << 'EOF'
export REDDIT_CLIENT_ID="your_client_id_here"
export REDDIT_CLIENT_SECRET="your_client_secret_here"
export REDDIT_USER_AGENT="SocialProofResearch/2.0 (academic; contact: tristan.boedts@warwick.ac.uk)"
# Optional:
# export REDDIT_USERNAME="your_reddit_username"
# export REDDIT_PASSWORD="your_reddit_password"
EOF

source ~/.bashrc
```

## 5. Pilot Test

```bash
cd ~/scraper && source venv/bin/activate

# This scrapes once and deletes the database — safe pre-ethics testing
python scraper.py --pilot
```

Expected output:
```
2026-03-21 12:00:00 [INFO] PILOT MODE — data will be deleted after this run
2026-03-21 12:00:25 [INFO] Cycle complete in 24.3s — 847 new posts, 847 polled, 12453 snapshots, 0 errors
2026-03-21 12:00:25 [INFO] Pilot complete. Database deleted.
```

## 6. Start Real Collection (after ethics approval)

```bash
# Single test with data retention
python scraper.py

# Check what we got
sqlite3 social_proof_data.db "
  SELECT subreddit, COUNT(DISTINCT post_id) as posts,
         COUNT(DISTINCT comment_id) as comments
  FROM comments GROUP BY subreddit;
"

# If happy, start the daemon
python scraper.py --daemon --interval 60
```

## 7. Run as a Persistent Service (systemd)

```bash
sudo tee /etc/systemd/system/reddit-scraper.service << EOF
[Unit]
Description=Reddit Social Proof Scraper
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=$USER
WorkingDirectory=/home/$USER/scraper
Environment="REDDIT_CLIENT_ID=your_client_id_here"
Environment="REDDIT_CLIENT_SECRET=your_client_secret_here"
Environment="REDDIT_USER_AGENT=SocialProofResearch/2.0 (academic; contact: tristan.boedts@warwick.ac.uk)"
ExecStart=/home/$USER/scraper/venv/bin/python scraper.py --daemon --interval 60
Restart=always
RestartSec=30

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable reddit-scraper
sudo systemctl start reddit-scraper

# Check status
sudo systemctl status reddit-scraper
sudo journalctl -u reddit-scraper -f
```

## 8. Monitor & Maintain

```bash
# Database size
ls -lh ~/scraper/social_proof_data.db

# Quick stats
sqlite3 ~/scraper/social_proof_data.db "
  SELECT
    COUNT(DISTINCT p.post_id) as tracked_posts,
    COUNT(DISTINCT c.comment_id) as tracked_comments,
    (SELECT COUNT(*) FROM snapshots) as total_snapshots,
    ROUND((MAX(s.scraped_utc) - MIN(s.scraped_utc)) / 86400.0, 1) as days_running
  FROM posts p
  JOIN comments c ON p.post_id = c.post_id
  JOIN snapshots s ON c.comment_id = s.comment_id;
"

# Purge deleted content (run weekly for API compliance)
python scraper.py --purge-deleted

# Export for analysis
python scraper.py --export
# gcloud compute scp reddit-scraper:~/scraper/exports/*.csv ./ --zone=us-central1-a
```

## 9. Transition from 1-min to 5-min intervals

After the pilot phase (~2 weeks), reduce polling frequency:

```bash
sudo systemctl stop reddit-scraper

# Edit the service file: change --interval 60 to --interval 300
sudo nano /etc/systemd/system/reddit-scraper.service

sudo systemctl daemon-reload
sudo systemctl start reddit-scraper
```

## 10. Pre-Analysis Steps

Before exporting data for analysis:

```bash
# 1. Pseudonymise usernames
python scraper.py --anonymise
# This creates username_lookup.json — ENCRYPT IT, store separately

# 2. Purge deleted content one final time
python scraper.py --purge-deleted

# 3. Export
python scraper.py --export
```

## 11. Database Size Estimates

The new design tracks ALL posts and ALL comments (not a sample). With
9 subreddits, census collection, and 48-hour tracking windows:

| Interval | Snapshots/day | DB growth/day | 30-day total |
|----------|--------------|---------------|-------------|
| 1-min | ~2-5M | 1-3 GB | 30-90 GB |
| 5-min | ~400K-1M | 200-600 MB | 6-18 GB |

**Recommendation**: Start at 1-min for 1-2 weeks (captures the transition
at high resolution), then switch to 5-min. The 30GB free-tier disk should
handle ~2-3 weeks at 1-min or the full month at 5-min.

If you're running low on space:
- Switch to 5-min intervals early
- Reduce MAX_POST_AGE_HOURS from 48 to 36 or 24
- Export and archive older data, then prune

## 12. Cost Summary

| Resource | Free Tier | Your Usage |
|----------|-----------|------------|
| e2-micro VM | 744 hrs/month | 744 hrs (24/7) |
| Standard disk | 30 GB | 10-25 GB |
| Egress | 1 GB/month | Minimal |
| Reddit API | 100 QPM | ~30-60 QPM |

**Expected cost: $0/month** in free-tier regions.

## 13. Important Notes

- **Verify hide durations** before starting real collection. Post a
  comment in each target subreddit and time when the score appears.
- **Reddit API compliance**: descriptive User-Agent, respect rate limits,
  purge deleted content regularly.
- **Backup regularly**: `gcloud compute scp` the .db file off the VM.
- **Score-hidden inference**: PRAW doesn't expose `score_hidden` directly.
  The scraper infers it from comment age vs. subreddit hide duration.
  This is accurate as long as your hide-duration config is correct.
- **Raw text**: stored for misinformation classification only. Delete
  after classification is complete (ethics commitment).
