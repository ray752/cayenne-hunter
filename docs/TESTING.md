# Testing Guide — Cayenne Hunter

## Manual Trigger

Tell OpenClaw in Telegram:

```
run cayenne hunter
```

The skill responds immediately (no waiting for 9 AM cron). You'll receive a Telegram summary within ~2–3 minutes.

Other valid triggers: `cayenne`, `car hunt`, `car scan`, `market scan`, `porsche`

---

## Expected Behavior Per Step

| Step | What to Expect | Success Signal |
|------|---------------|----------------|
| 1 — GitHub read | Skill reads `data/market_tracker.json` and `config/scoring_rubric.json` | No error; skill logs "X active listings, last scan YYYY-MM-DD" |
| 2 — Scrape mobile.de | Browser opens the search URL, handles cookie consent | Listing cards load; IDs and prices extracted |
| 3 — Compare | Skill classifies each listing as NEW, EXISTING, or GONE | Summary shows correct counts |
| 4 — Score | New listings get 0-100 scores based on rubric | Scores appear in output; no listing scored > 100 |
| 5 — Report | `reports/CAYENNE_HUNT_REPORT.md` is generated | Report includes today's date and scan number |
| 6 — GitHub push | Both `market_tracker.json` and `CAYENNE_HUNT_REPORT.md` updated | `git log` shows new commit with `scan #X:` prefix |
| 7 — Telegram | Alert arrives in your chat | Message matches GEM/HOT/normal format |

---

## Verify GitHub Push Worked

After a scan, check the last commit:

```bash
gh api repos/ray752/cayenne-hunter/commits/main --jq '.commit.message'
```

Expected format: `scan #9: Y new, Z gone, best=XXXXXXXX score=NN`

Or browse: https://github.com/ray752/cayenne-hunter/commits/main

---

## Validate JSON Integrity

Run locally after any scan:

```bash
# Pull latest tracker
gh api repos/ray752/cayenne-hunter/contents/data/market_tracker.json \
  --jq '.content' | base64 -d | jq '.meta'

# Should return:
# { "version": "1.1", "last_scan": "<today>", "total_scans": N, ... }
```

Or clone and check:
```bash
git clone https://github.com/ray752/cayenne-hunter.git /tmp/cayenne-check
jq '.meta, (.listings | length), (.removed_listings | length)' \
  /tmp/cayenne-check/data/market_tracker.json
```

---

## Cron Schedule Verification

The skill fires at **09:00 Europe/Berlin** (`0 9 * * *`). That's:
- **08:00 UTC** in winter (CET = UTC+1)
- **07:00 UTC** in summer (CEST = UTC+2)

To verify the cron is registered in OpenClaw, check the schedule list (if OpenClaw exposes this via UI or CLI).

---

## Failure Mode Table

| Symptom | Cause | Fix |
|---------|-------|-----|
| "Page loading timeout" | mobile.de slow or bot detection | Skill retries once. If persistent: add 3–5s wait or stealth plugin |
| Cookie wall blocks everything | GDPR overlay not dismissed | Skill looks for "Ablehnen". If text changes: update skill trigger phrase |
| Empty listing data | JS hasn't rendered yet | Skill waits for listing cards. If still empty: check page selector |
| GitHub push fails | PAT expired or wrong permissions | Regenerate PAT with `repo` scope. Update `GITHUB_PAT` in `~/.openclaw/.env` |
| Telegram silent | Wrong chat ID or bot not started | User must `/start` the bot first. Verify chat ID with @userinfobot |
| All listings show as "new" | Tracker JSON corrupted | Re-seed: push the backup `market_tracker.json` from the `cayenne` source folder |
| Akamai blocks browser | Fingerprint detected | Add stealth plugin, random viewport, disable webdriver flag |
| Zero listings extracted | Major page structure change | Check mobile.de manually. Update listing card CSS selector in skill prompt |

---

## Re-seeding the Database

If `market_tracker.json` gets corrupted:

```bash
# Push the local source copy back to GitHub
gh api repos/ray752/cayenne-hunter/contents/data/market_tracker.json \
  --method PUT \
  --field message="fix: re-seed tracker from backup" \
  --field content="$(base64 < /Users/raimonds/Documents/GitHub/cayenne/market_tracker.json)" \
  --field sha="$(gh api repos/ray752/cayenne-hunter/contents/data/market_tracker.json --jq '.sha')"
```
