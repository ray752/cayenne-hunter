# V2 Roadmap — Cayenne Hunter

Current system: daily scrape → score → push → Telegram alert. V1 is stable. Here's what's worth building next.

---

## V2.1 — Immediate Price Drop Webhooks

**What:** Trigger an alert the moment a tracked listing drops 10%+ in price — not just at 9 AM.

**Why:** High-scoring cars sell in <24h. A 10% price drop on a score-70+ car is a buy signal. Waiting until next morning may be too late.

**How:**
- Add a second skill: `cayenne-price-watch`
- Cron: every 2–4 hours (`0 */3 * * *`)
- Only checks prices on tracked listings (no full scrape — faster, less bot exposure)
- Alert threshold: 10%+ drop on any listing with score ≥ 60

**Complexity:** Low — reuses existing infrastructure, just a more frequent partial scan.

---

## V2.2 — AutoScout24 Multi-Site Coverage

**What:** Scan AutoScout24 in addition to mobile.de.

**Why:**
- AutoScout24 has less aggressive bot protection than mobile.de (Akamai)
- Some sellers list exclusively on AutoScout24
- Cross-listing detection: same car on both sites = more leverage when haggling

**How:**
- Add AutoScout24 URL to Step 2 (parallel scrape)
- Deduplicate by: same seller + same price + same mileage = same car
- Store `source: ["mobile.de", "autoscout24"]` in listing data

**Complexity:** Low-Medium — same logic, different URL and HTML structure.

---

## V2.3 — Auto-Haggle Email Generator

**What:** When a listing scores 65+, auto-generate a personalized German haggle email using the templates in `templates/HAGGLE_EMAILS_DE.md`.

**Why:** The hardest part of buying is starting the negotiation. Pre-written templates exist — just need to fill in the variables.

**How:**
- Add Step 8 to the skill prompt: "If any listing scores 65+, generate a personalized haggle email"
- Use the template framework in `templates/HAGGLE_EMAILS_DE.md`
- Variables to fill: seller name, listing price, haggle target, competing listings, specific red flags
- Output: push generated emails to `templates/haggle_outbox/YYYY-MM-DD-[listing_id].md`
- Telegram alert includes: "📧 Haggle email drafted for [listing_id] — check GitHub"

**Complexity:** Medium — requires careful prompt engineering so emails don't sound robotic.

---

## V2.4 — GitHub Pages Live Dashboard

**What:** Serve `reports/dashboard.html` via GitHub Pages as a live visual dashboard.

**Why:** The dashboard is already built (48KB, standalone HTML/CSS/JS). It just needs to be served.

**How:**
1. Enable GitHub Pages on `ray752/cayenne-hunter` (Settings → Pages → Branch: main → Folder: /reports)
2. Update the daily scan to regenerate `reports/dashboard.html` with fresh data
3. Dashboard URL: `https://ray752.github.io/cayenne-hunter/dashboard.html`
4. Add the dashboard link to daily Telegram alerts

**Complexity:** Very low — dashboard HTML already exists, just needs GitHub Pages enabled and daily regeneration.

**Note:** The dashboard currently has hardcoded data from scan #8. The skill would need to write updated dashboard HTML on each scan. Alternatively: have the dashboard fetch `market_tracker.json` directly from GitHub raw URL at load time (no build step needed).

---

## Priority Order

1. **V2.4 (GitHub Pages)** — 30 min, already done, just needs Pages enabled + dynamic data loading
2. **V2.1 (Price drop webhooks)** — 2h, high signal-to-noise, directly addresses the <24h shelf life problem
3. **V2.3 (Auto-haggle emails)** — 4h, reduces friction at the critical moment
4. **V2.2 (AutoScout24)** — 4h, 30% more coverage, backup if mobile.de blocks the bot

---

## Not Worth Building

- Mobile app — Telegram is already the perfect notification surface
- Price prediction model — 39 data points isn't enough for ML, the scoring rubric is fine
- Auto-contact sellers — spam risk, legal grey area in Germany, ruins negotiation leverage
- Email alerts — Telegram is faster and already works
