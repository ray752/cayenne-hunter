# Cayenne Hunter

Autonomous daily market scanner for a Porsche Cayenne S (955) — 2003–2006, 4.5L V8 NA 340 PS.

Runs on OpenClaw at 9 AM Berlin time. Scrapes mobile.de, scores listings 0–100, tracks market velocity, and sends Telegram alerts.

## Structure

```
data/market_tracker.json      ← persistent database (listings, price history, market velocity)
reports/CAYENNE_HUNT_REPORT.md ← latest daily scan report
reports/dashboard.html         ← visual dashboard
config/scoring_rubric.json     ← 0-100 scoring formula
templates/HAGGLE_EMAILS_DE.md  ← German negotiation email templates
docs/PRD.md                    ← full product requirements
```

## Scoring

| Score | Signal | Avg shelf life |
|-------|--------|----------------|
| 78+   | 🚨 GEM — buy immediately | <24h |
| 70–77 | ⚡ HOT — act within hours | ~4 days |
| 60–69 | ⚠️ WATCH — verify condition | ~2 days |
| <60   | 🐢 Low priority | — |

## Target Spec

- Model: Cayenne S (955) — NOT Turbo, NOT V6
- Engine: 4.5L V8 NA, 250 kW / **340 PS**
- Years: 2003–2006
- Suspension: **Steel springs only** (auto-reject air suspension)
- Budget: €4,000–€7,500 (car) / under €10,000 all-in
- Color: Black > Dark Grey > Dark Blue
- Mileage: under 200k km
