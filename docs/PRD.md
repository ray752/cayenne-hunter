# PRD: Cayenne Hunter

**Autonomous listing scraper, AI scorer, and negotiation agent for sourcing a first-gen Porsche Cayenne S (955) as a budget overlander build.**

---

## 0. Context for the AI coding agent

You are building a single, self-contained Python application. No Make.com, no Zapier, no n8n, no low-code glue. The stack is Python 3.12+, async-first, with Claude as the intelligence layer. Ship fast, keep deps minimal, deploy to a single VPS or Railway container. The end user is technical and will run this from a terminal or a cron-triggered Docker container.

---

## 1. Problem

Finding a first-gen Porsche Cayenne S (955, 2003–2006) at the right price-to-condition ratio is a daily grind across dozens of European listing sites. The sweet spot is a Cayenne S (4.5L V8 NA, 340 PS) under €7,500 with steel springs, no accident history, and ideally a dark color (black, dark grey, dark blue, dark green) that needs zero repaint for an aggressive overlander aesthetic. Bonus alpha: finding a truck where someone already sank money into mods (AT tires, spacers, lift springs, blacked trim, smoked taillights) and is selling at a loss. Manually refreshing these sites, evaluating listings, and negotiating in German is a full-time job. Automate all of it.

---

## 2. Target vehicle spec

| Attribute | Requirement |
|---|---|
| Model | Porsche Cayenne **S** (955) — the 4.5L V8 NA, 340 PS. NOT the V6 (underpowered, no sound), NOT the Turbo/Turbo S (maintenance bomb at this budget). |
| Years | 2003–2006. Prefer 2005–2006 (post-facelift refinements). Flag early 2003 models for transfer case issues. |
| Engine | 4.5L V8 naturally aspirated (250 kW / 340 PS). This is the sweet spot: cheaper than V6 on the used market, 90 PS more, better V8 sound, mechanically simpler than Turbo. |
| Exterior color | **Dark colors preferred** (no repaint needed for overlander aesthetic): Black, Dark Grey (Lava Grey, Meteor Grey), Dark Blue, Dark Green. Any color acceptable — can wrap later (€1,500–€2,500). Score dark colors higher but don't disqualify light ones. |
| Suspension | **STEEL SPRINGS ONLY.** Air suspension (Luftfederung) is an **automatic disqualifier** unless the price is low enough to fund steel conversion on day one (listed price + €1,200 must be under €7,500). |
| Transmission | Tiptronic (automatic). Flag manual but don't disqualify. |
| Mileage | Under 200,000 km. Sweet spot: 120k–170k km. Under 100k km at this price range = suspicious, investigate. |
| Budget (car) | €5,000–€7,500 for the car. Flag under €4k as scam, over €8k as stretch. |
| Budget (mods) | €1,400–€3,400 for overlander build (AT tires €700, lift springs €400, spacers €180, de-chrome €100, smoked tails €25, optional wrap €2,000). |
| Budget (all-in) | Under €10,000 total. Hard ceiling. |
| Owners | Fewer is better. 3+ owners = yellow flag. 5+ owners = red flag (score penalty). |
| TÜV | Valid TÜV/HU is a plus. Expired TÜV = negotiate harder (cost to pass). |
| Location | Germany primarily (largest market). Also: Netherlands, Belgium, Austria, Switzerland, Poland, Czech Republic. |

### Desired mods (bonus scoring, not required)

- All-terrain tires (BFGoodrich KO2, General Grabber AT3, Falken Wildpeak, etc.)
- Wheel spacers (15mm+)
- Lift springs or spacers (+25mm to +50mm)
- Blacked-out / de-chromed trim
- Roof rack, light bar, or skid plates
- Any other overlander/expedition aesthetic mods
- Note: Smoked taillights, plastidip trim etc. are too cheap (€20–€100 DIY) to affect scoring.

### Hard disqualifiers (auto-reject, score 0)

- **Air suspension** without sufficient price discount (see Suspension row above)
- Salvage title / repaired accident damage (reparierter Unfallschaden)
- Turbo or Turbo S model at this budget (maintenance trap)
- "Export" or "Bastler" (project car) listings with no photos

### Red flags (auto-penalize, -10 to -20 points)

- Missing or inconsistent service history
- 5+ total owners
- Transfer case issues mentioned (esp. early 2003 models)
- Aftermarket turbo kits or engine swaps
- Rust in subframe, wheel arches, or floor pans
- No Unfallfrei (accident-free) declaration

---

## 3. Architecture

```
┌─────────────────────────────────────────────────────┐
│                   CAYENNE HUNTER                     │
│                                                      │
│  ┌──────────┐   ┌──────────┐   ┌────────────────┐  │
│  │ Scraper  │──▶│ Scorer   │──▶│ Notifier       │  │
│  │ Engine   │   │ (Claude) │   │ (Telegram Bot) │  │
│  └──────────┘   └──────────┘   └────────────────┘  │
│       │              │               │               │
│       ▼              ▼               ▼               │
│  ┌──────────┐   ┌──────────┐   ┌────────────────┐  │
│  │ Listing  │   │ Score +  │   │ Haggle Agent   │  │
│  │ Store    │   │ Analysis │   │ (Claude + SMTP)│  │
│  │ (SQLite) │   │ Cache    │   │ [V2]           │  │
│  └──────────┘   └──────────┘   └────────────────┘  │
└─────────────────────────────────────────────────────┘
```

### Components

1. **Scraper Engine** — Async crawlers for each listing site. Outputs normalized `Listing` objects.
2. **Listing Store** — SQLite database. Single source of truth. Tracks listing lifecycle (new → scored → contacted → negotiating → deal/dead).
3. **Scorer** — Claude API call per listing. Structured output. Deterministic scoring rubric.
4. **Notifier** — Telegram bot that pings the user with scored listings, photos, and a one-tap "engage" button.
5. **Haggle Agent** (V2) — Claude-powered email negotiation agent that handles back-and-forth in German until a price threshold is met, then hands off to the human.

---

## 4. V1 — Scraper + Scorer + Notifier

### 4.1 Scraper Engine

#### Target sites (priority order)

| Priority | Source | Method | Notes |
|---|---|---|---|
| 1 (PRIMARY) | **mobile.de Suchauftrag** (email alerts) | IMAP — parse notification emails | FREE, reliable, zero bot detection. User sets up saved search once, mobile.de pushes new listings via email. Parser extracts listing ID + URL from email body. |
| 2 (PRIMARY) | **AutoScout24 saved search** | RSS feed or email alerts | AutoScout24 supports RSS for saved searches. Parse the feed for new entries. |
| 3 (FALLBACK) | **mobile.de direct** | Playwright + stealth plugin, sort by newest (`sb=doc`), page 1 only | Use optimized URL with `pw=250:` (S/Turbo only) + `sb=doc` (newest first). Only for weekly full-market snapshots or when email pipeline is delayed. |
| 4 (FALLBACK) | **Kleinanzeigen.de** | HTTP + parse | Less bot protection than mobile.de. Good for private seller deals. |
| 5 (BONUS) | **AutoUncle.de** | HTTP + parse | Aggregator — catches smaller dealers not on mobile.de |

#### Data pipeline (NOT a scraper — an alert processor)

The architecture is **push-first, pull-fallback:**

```
mobile.de Suchauftrag ──email──▶ IMAP inbox ──parse──▶ New listing IDs
AutoScout24 saved search ──RSS──▶ Feed reader ──parse──▶ New listing IDs
                                                              │
                                                              ▼
                                                    Detail page fetcher
                                                    (Playwright, 1 page at a time,
                                                     with 5–10s random delays)
                                                              │
                                                              ▼
                                                    Normalized Listing objects
                                                              │
                                                              ▼
                                                    Scorer (Claude) ──▶ Notifier
```

#### Requirements

- **Email parsing**: Use `imaplib` + `email` stdlib to connect to an IMAP mailbox. Parse mobile.de alert emails for listing URLs. Extract listing ID from URL. This is the primary data source — not HTTP scraping.
- **RSS parsing**: Use `feedparser` for AutoScout24 RSS feeds.
- **Detail page fetcher**: Playwright with `playwright-stealth` plugin for fetching individual listing detail pages. NOT for search result scraping. Rate limit: 1 request per 5–10 seconds with jitter.
- **Dedup**: Hash on `(site, listing_id)`. If already in DB, only update `last_seen_at` and `price`. Re-score if price changed >5%.
- **Image downloading**: Download first 5 images per listing. Claude needs these for visual condition scoring.
- **Schedule**: Check IMAP inbox every 15 minutes. RSS feeds every 30 minutes. Full-market Playwright scan weekly (Sunday night).

#### Normalized Listing schema

```python
@dataclass
class Listing:
    id: str                    # UUID
    source: str                # "mobile_de", "autoscout24", etc.
    source_id: str             # Original listing ID from the site
    url: str                   # Direct link to listing
    title: str                 # Raw listing title
    price_eur: int             # Price in EUR (convert from other currencies)
    currency_original: str     # Original currency if not EUR
    mileage_km: int            # Mileage in km
    year: int                  # Model year
    first_registration: str    # MM/YYYY if available
    color_stated: str          # Color as stated by seller
    transmission: str          # "automatic" | "manual" | "unknown"
    fuel_type: str             # Should always be "petrol" for Cayenne S
    power_hp: int | None       # Listed HP
    location_city: str         # City
    location_country: str      # Country code (DE, NL, AT, etc.)
    seller_type: str           # "private" | "dealer"
    seller_name: str | None
    seller_phone: str | None
    seller_email: str | None   # Critical for V2 haggle agent
    description_raw: str       # Full listing description text
    image_urls: list[str]      # All image URLs from listing
    image_paths: list[str]     # Local paths to downloaded images
    extras_raw: str            # Raw equipment/extras text
    created_at: datetime
    updated_at: datetime
    last_seen_at: datetime
    price_history: list[dict]  # [{"price": 8500, "date": "2026-03-15"}, ...]
    status: str                # "new" | "scored" | "contacted" | "negotiating" | "deal" | "dead" | "rejected"
```

### 4.2 Scorer (Claude)

Each new or price-changed listing gets scored by Claude. This is the core intelligence.

#### Scoring prompt strategy

Send Claude a structured prompt with:
1. All listing metadata (the `Listing` dataclass fields)
2. The first 3–5 images (as base64 or via URL if using a vision-capable model)
3. The full description text (in whatever language — Claude handles German/Dutch/Polish natively)
4. The scoring rubric (below)

#### Scoring rubric

Claude must return a structured JSON response. Use `response_format` / tool use to enforce schema.

```json
{
  "score_total": 82,           // 0-100. Gem threshold: 85+. Good: 70+. Skip: <50.
  "breakdown": {
    "price_value": 25,         // 0-25: Under €5k=25, €5-6k=22, €6-7k=18, €7-8k=14, €8-9k=10, €9-10k=6
    "mileage": 20,             // 0-20: Under 100k=20, 100-130k=18, 130-160k=15, 160-180k=12, 180-200k=8, 200k+=4
    "condition": 20,           // 0-20: Unfallfrei+Scheckheft=20, Unfallfrei=15, No info=8, Accident history=0
    "color_match": 15,         // 0-15: Black/dark grey=15, Dark blue/green=13, Silver/grey=8, Light/bright/other=5
    "seller_trust": 10,        // 0-10: Dealer 4.5+=10, Dealer 4+=8, Private=6, Dealer 3+=5, Dealer <3=3. Penalty: 5+ owners=-3
    "mod_readiness": 10        // 0-10: Steel springs+dark color+under €7.5k=10, Steel+budget=7. Air suspension=-10 (auto-disqualifier unless price compensates). Over €7.5k=-5.
  },
  "suspension_type": "steel",  // "steel" | "air" — CRITICAL field. Air = auto-reject unless price + €1,200 < €7,500
  "num_owners": 2,             // Number of registered owners. 5+ = red flag.
  "color_assessment": "Black Metallic — ideal for overlander aesthetic, no wrap needed.",
  "mod_inventory": [
    "BFGoodrich KO2 265/65R18",
    "20mm H&R wheel spacers",
    "+30mm lift springs (unknown brand)",
    "De-chromed window trim"
  ],
  "known_defects": [],
  "red_flags": [],
  "estimated_mod_value_eur": 2800,
  "summary": "Strong candidate. Black Cayenne S, steel springs, 155k km, Unfallfrei, dealer-sold with 4.5-star rating. Asking €5,980 — leaves €4k for mods and reserve. Best value on the market.",
  "recommended_action": "contact",   // "contact" | "watch" | "skip"
  "opening_offer_eur": 5000,
  "walk_away_price_eur": 5500,
  "language": "de"                    // Language of listing, for haggle agent
}
```

#### Claude API config

- **Model**: `claude-sonnet-4-6` for scoring (cost-efficient, fast, vision-capable). Escalate to `claude-opus-4-6` only for haggle negotiations in V2.
- **Max tokens**: 2000 for scoring response.
- **Temperature**: 0.2 (we want deterministic, analytical scoring — not creative).
- **System prompt**: Include the full vehicle spec from Section 2 as system context so Claude knows exactly what we're looking for.
- **Image handling**: Send images as base64 in the user message. Max 5 images per scoring call.

### 4.3 Notifier (Telegram)

#### Setup

- Create a Telegram bot via BotFather. Store the bot token in `.env`.
- User configures their chat ID in `.env`.

#### Notification triggers

| Event | Trigger threshold | Message content |
|---|---|---|
| 🚨 GEM | score_total >= 85 | URGENT: Full scoring breakdown, top 3 photos, direct link, "Engage" button. This is a buy signal. |
| Hot lead | score_total 70–84 | Full scoring breakdown, photos, direct link, "Engage" button |
| Warm lead | score_total 50–69 | Summary + link + score |
| 📉 Price drop | Any tracked listing drops >10% | Alert with old/new price, link, updated score |
| 💀 Gone | Previously tracked listing disappears | Alert — listing may be sold, urgency signal for remaining picks |
| ⛔ Auto-reject | Air suspension detected | Log but don't notify unless price compensates per formula |

#### Telegram message format

```
🔥 HOT LEAD — Score: 82/100

2004 Cayenne S • Black Metallic • Steel Springs
📍 Leipzig, DE • 155.000 km
💰 €5.980 → Haggle to €5.000–€5.300

✅ Unfallfrei • BOSE • Leather • Sunroof
✅ Steel springs (no air susp!)
✅ Dealer: 4.5⭐ (425 reviews)
🎨 Blue — could wrap or keep

Recommended: Contact @ €5.000 opener
Walk-away: €5.500

[View Listing](https://mobile.de/...)
[📸 Photos] [✅ Engage] [❌ Skip]
```

#### Inline keyboard actions

- **Engage**: Moves listing to `contacted` status. In V2, auto-triggers haggle agent. In V1, just marks it and sends the user the seller contact info.
- **Skip**: Moves listing to `rejected` status. Won't notify again.
- **Watch**: Keeps listing in `scored` status. Will notify on price drops.

---

## 5. V2 — Haggle Agent

### 5.1 Overview

An autonomous Claude-powered negotiation agent that conducts price negotiations with sellers via email, entirely in German, with no human involvement until a deal is within striking distance.

### 5.2 Trigger

User taps "Engage" on a Telegram notification. The system:

1. Looks up the listing's seller email (scraped from the site, or from a "contact seller" form submission).
2. If no email is available, sends the user a Telegram message asking them to provide it manually (many sites hide emails behind a contact form that requires CAPTCHA).
3. Once email is available, the haggle agent begins.

### 5.3 Email infrastructure

- **Sending**: Use a dedicated email address (e.g., `kaufen@yourdomain.de` or a Gmail alias). Send via SMTP (Gmail app password, or Mailgun/Sendgrid for deliverability).
- **Receiving**: IMAP polling on the same inbox, every 5 minutes. Parse incoming replies and match to active negotiations by `In-Reply-To` header or subject line thread ID.
- **Email identity**: German-sounding name. "Max Weber" or whatever. The email signature should look like a normal German car buyer — no company branding, just a name and phone number.

### 5.4 Negotiation protocol

The haggle agent operates as a state machine:

```
[INITIAL_CONTACT] → [WAITING_REPLY] → [NEGOTIATING] → [OFFER_MADE] → [DEAL_ZONE] → [HANDOFF]
                                            ↕
                                       [STALLED]
```

#### State: INITIAL_CONTACT

Claude composes an opening email in German. The tone is:
- Friendly, casual, knowledgeable about Porsches
- Asks 2–3 specific questions about the car (based on red flags from scoring)
- Does NOT make a price offer in the first email
- Mentions you're looking for exactly this spec and are a serious buyer

Example prompt to Claude:
```
You are Max, a car enthusiast in Germany looking for a first-gen Cayenne S in green.
Write an initial inquiry email in German to the seller of this listing:
{listing_data}

Your scoring analysis found these concerns: {red_flags}
Ask about these naturally. Be warm but not desperate. Do NOT mention a price yet.
Keep it under 150 words.
```

#### State: NEGOTIATING

After the seller replies, Claude:
1. Parses the reply for: asking price confirmation, answers to questions, tone/flexibility signals.
2. Decides whether to: ask more questions, make an offer, or disengage.
3. If making an offer, starts at `opening_offer_eur` from the scoring.

Negotiation rules (hard-coded, not left to Claude's judgment):
- **Never exceed `walk_away_price_eur`** from the scoring.
- **Max 3 counteroffers**. After 3 rounds of back-and-forth on price, either accept or walk.
- **Increment ceiling**: Each counteroffer can increase by max 5% of the opening offer.
- **Tone**: Always polite, never aggressive. Mirror the seller's formality level.
- **If seller gets personal or hostile**: Disengage gracefully. Mark as `dead`.

#### State: DEAL_ZONE

When the seller's price is within 10% of `walk_away_price_eur`, the agent:
1. Sends a Telegram message to the user: "Deal possible at €X. Seller seems flexible. Approve?"
2. Waits for user confirmation via Telegram inline button.
3. If approved: sends a "I'll take it, let's arrange inspection/pickup" email.
4. If rejected: sends a polite "I need to think about it" and marks as `stalled`.

#### State: HANDOFF

The agent sends the user:
- Full email thread
- Seller contact details
- Agreed price
- Suggested next steps (arrange inspection, bring a PPI mechanic, etc.)

The agent is DONE. Human takes over for the actual purchase.

### 5.5 Safety rails

- **Every outgoing email is logged** to the DB with full content.
- **Daily email cap**: Max 10 outgoing emails per day across all negotiations. Prevents runaway.
- **No financial commitments**: The agent can agree on a price in principle but NEVER sends deposits, bank details, or commits to a purchase without human confirmation.
- **Kill switch**: User can send `/stop` in Telegram to immediately pause all active negotiations.
- **Cooldown**: After a `dead` negotiation, the agent won't contact the same seller again for 30 days.

---

## 6. Data model (SQLite)

```sql
CREATE TABLE listings (
    id TEXT PRIMARY KEY,
    source TEXT NOT NULL,
    source_id TEXT NOT NULL,
    url TEXT NOT NULL,
    title TEXT,
    price_eur INTEGER,
    currency_original TEXT,
    mileage_km INTEGER,
    year INTEGER,
    first_registration TEXT,
    color_stated TEXT,
    transmission TEXT,
    fuel_type TEXT,
    power_hp INTEGER,
    location_city TEXT,
    location_country TEXT,
    seller_type TEXT,
    seller_name TEXT,
    seller_phone TEXT,
    seller_email TEXT,
    description_raw TEXT,
    image_urls TEXT,        -- JSON array
    image_paths TEXT,       -- JSON array
    extras_raw TEXT,
    price_history TEXT,     -- JSON array
    status TEXT DEFAULT 'new',
    created_at TEXT,
    updated_at TEXT,
    last_seen_at TEXT,
    UNIQUE(source, source_id)
);

CREATE TABLE scores (
    id TEXT PRIMARY KEY,
    listing_id TEXT NOT NULL REFERENCES listings(id),
    score_total INTEGER,
    breakdown TEXT,          -- JSON
    color_assessment TEXT,
    mod_inventory TEXT,      -- JSON array
    red_flags TEXT,          -- JSON array
    estimated_mod_value_eur INTEGER,
    summary TEXT,
    recommended_action TEXT,
    opening_offer_eur INTEGER,
    walk_away_price_eur INTEGER,
    language TEXT,
    scored_at TEXT,
    model_used TEXT          -- Track which Claude model scored this
);

CREATE TABLE negotiations (
    id TEXT PRIMARY KEY,
    listing_id TEXT NOT NULL REFERENCES listings(id),
    status TEXT DEFAULT 'initial_contact',  -- initial_contact, waiting_reply, negotiating, offer_made, deal_zone, handoff, stalled, dead
    seller_email TEXT NOT NULL,
    our_email TEXT NOT NULL,
    current_offer_eur INTEGER,
    seller_asking_eur INTEGER,
    counteroffer_count INTEGER DEFAULT 0,
    max_price_eur INTEGER,
    started_at TEXT,
    last_activity_at TEXT,
    outcome TEXT             -- "purchased" | "walked" | "seller_unresponsive" | "killed"
);

CREATE TABLE emails (
    id TEXT PRIMARY KEY,
    negotiation_id TEXT NOT NULL REFERENCES negotiations(id),
    direction TEXT NOT NULL,  -- "outbound" | "inbound"
    subject TEXT,
    body TEXT,
    sent_at TEXT,
    message_id TEXT,          -- Email Message-ID header
    in_reply_to TEXT          -- For threading
);

CREATE TABLE scrape_runs (
    id TEXT PRIMARY KEY,
    started_at TEXT,
    completed_at TEXT,
    sites_scraped TEXT,       -- JSON array
    listings_found INTEGER,
    new_listings INTEGER,
    errors TEXT               -- JSON array
);
```

---

## 7. Config

All configuration via a single `config.yaml` at the project root, with `.env` for secrets.

```yaml
# config.yaml
scraper:
  schedule_minutes: 30
  sites:
    - name: mobile_de
      enabled: true
      delay_seconds: 4
    - name: autoscout24
      enabled: true
      delay_seconds: 3
    - name: kleinanzeigen
      enabled: true
      delay_seconds: 5
    - name: marktplaats
      enabled: true
      delay_seconds: 4
    - name: olx_pl
      enabled: false
      delay_seconds: 5
    - name: willhaben
      enabled: true
      delay_seconds: 4
    - name: autouncle
      enabled: true
      delay_seconds: 3
  proxy:
    enabled: false
    type: socks5  # or http
    url: ""
  max_images_per_listing: 5
  image_storage: local  # or s3

search:
  make: "Porsche"
  model: "Cayenne"
  variant: "S"
  year_from: 2003
  year_to: 2006
  price_from: 2000
  price_to: 15000
  fuel: petrol
  countries:
    - DE
    - NL
    - BE
    - AT
    - CH
    - PL
    - CZ

scoring:
  model: claude-sonnet-4-6
  temperature: 0.2
  max_tokens: 2000
  rescore_on_price_change_pct: 5

telegram:
  notify_hot_threshold: 75
  notify_warm_threshold: 50
  notify_price_drop_pct: 5

haggle:
  enabled: false  # Set true for V2
  model: claude-opus-4-6
  max_counteroffers: 3
  counteroffer_increment_pct: 5
  daily_email_cap: 10
  cooldown_days: 30
  email_check_interval_minutes: 5
  persona_name: "Max Weber"
  persona_phone: "+49 xxx xxxxxxx"
```

```bash
# .env
ANTHROPIC_API_KEY=sk-ant-...
TELEGRAM_BOT_TOKEN=...
TELEGRAM_CHAT_ID=...
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=kaufen@yourdomain.de
SMTP_PASSWORD=...
IMAP_HOST=imap.gmail.com
IMAP_PORT=993
IMAP_USER=kaufen@yourdomain.de
IMAP_PASSWORD=...
S3_BUCKET=       # Optional
S3_ACCESS_KEY=   # Optional
S3_SECRET_KEY=   # Optional
```

---

## 8. Tech stack

| Component | Choice | Rationale |
|---|---|---|
| Language | Python 3.12+ | Fastest to vibe-code, best Anthropic SDK support |
| HTTP client | `httpx` | Async, HTTP/2, better than `requests` |
| HTML parsing | `selectolux` or `beautifulsoup4` + `lxml` | selectolux is faster, bs4 is more forgiving on messy HTML |
| Database | SQLite via `aiosqlite` | Zero ops, embedded, async. No Postgres needed for this scale |
| Claude SDK | `anthropic` (official Python SDK) | Structured outputs, vision, tool use |
| Telegram | `python-telegram-bot` v20+ | Async-native, inline keyboards |
| Email | `aiosmtplib` + `aioimaplib` | Async SMTP/IMAP |
| Scheduling | `apscheduler` | Lightweight, no external dependency |
| Config | `pydantic-settings` + `pyyaml` | Validation + yaml parsing |
| Logging | `structlog` | Structured JSON logs, easy to search |
| Containerization | Docker + `docker-compose.yml` | Single `docker compose up` to run everything |

---

## 9. Project structure

```
cayenne-hunter/
├── docker-compose.yml
├── Dockerfile
├── config.yaml
├── .env.example
├── pyproject.toml
├── README.md
├── src/
│   ├── __init__.py
│   ├── main.py                  # Entry point, scheduler setup
│   ├── config.py                # Pydantic settings, yaml loader
│   ├── db/
│   │   ├── __init__.py
│   │   ├── models.py            # Dataclasses
│   │   ├── database.py          # SQLite connection, migrations
│   │   └── queries.py           # All DB operations
│   ├── scrapers/
│   │   ├── __init__.py
│   │   ├── base.py              # Abstract base scraper
│   │   ├── mobile_de.py
│   │   ├── autoscout24.py
│   │   ├── kleinanzeigen.py
│   │   ├── marktplaats.py
│   │   ├── olx_pl.py
│   │   ├── willhaben.py
│   │   ├── autouncle.py
│   │   └── utils.py             # UA rotation, currency conversion, etc.
│   ├── scorer/
│   │   ├── __init__.py
│   │   ├── scorer.py            # Claude scoring logic
│   │   ├── prompts.py           # All Claude prompts as templates
│   │   └── schema.py            # Pydantic models for scoring response
│   ├── notifier/
│   │   ├── __init__.py
│   │   └── telegram.py          # Telegram bot + notifications
│   ├── haggler/                 # V2
│   │   ├── __init__.py
│   │   ├── agent.py             # Negotiation state machine
│   │   ├── email_client.py      # SMTP/IMAP async client
│   │   ├── prompts.py           # Negotiation prompt templates
│   │   └── templates/           # Email templates
│   │       └── signature.txt
│   └── utils/
│       ├── __init__.py
│       ├── images.py            # Image download + storage
│       ├── currency.py          # EUR conversion
│       └── logging.py           # Structlog config
└── tests/
    ├── conftest.py
    ├── test_scrapers/
    ├── test_scorer/
    ├── test_notifier/
    └── test_haggler/
```

---

## 10. Scraper implementation notes

### CRITICAL ARCHITECTURE DECISION: Don't scrape — use email alerts

**The #1 lesson from testing:** Scraping mobile.de is unreliable. They use Akamai bot detection, the Cowork VM's egress proxy blocks car listing sites, Chrome browser tools timeout, and `httpx`/`curl` get instantly blocked. Even when scraping works, it's fragile and breaks without warning.

**The correct approach is to use mobile.de's own Suchauftrag (saved search alert) system:**

1. User creates a saved search on mobile.de with the target filters (Cayenne S, 2003–2006, Petrol, under €10k, 250kW+)
2. mobile.de emails the user when new listings match — for FREE, reliably, with zero bot detection issues
3. The scraper's job becomes: **parse those notification emails** (via IMAP), extract listing IDs/URLs, fetch detail pages, and feed them into the scorer
4. Fallback: RSS feeds from AutoScout24 saved searches (they support this natively)

This flips the architecture from "pull" (scrape sites) to "push" (sites notify you). It's more reliable, cheaper, faster, and doesn't break when sites update their anti-bot systems.

### mobile.de — optimized search URL

If you DO need to scrape (e.g., for initial bulk import or periodic full-market snapshots), use the optimized URL:

```
https://suchen.mobile.de/fahrzeuge/search.html?dam=false&isSearchRequest=true&ms=20100%3B18%3B&fr=2003%3A2006&ft=PETROL&pw=250%3A&p=%3A10000&sb=doc&s=Car&vc=Car
```

**Key parameters explained:**
- `ms=20100%3B18%3B` — Porsche (20100), Cayenne (18). Verified March 2026.
- `fr=2003%3A2006` — First registration 2003–2006
- `ft=PETROL` — Fuel type petrol
- `pw=250%3A` — **Power 250kW+ (340PS+).** This filters OUT the V6 (250PS/184kW) and keeps only the S (340PS/250kW) and Turbo (450PS/331kW). Cuts results from ~137 to ~25–30.
- `p=%3A10000` — Price up to €10,000
- `sb=doc` — **Sort by date, newest first.** This is CRITICAL for incremental scanning — page 1 always has the freshest listings. If the newest listing on page 1 is already in your tracker, you can stop. No need to paginate.
- No color filter — we want all colors, score dark ones higher

**DO NOT use `ecol=GREEN`** — the green-only filter was dropped. All colors are acceptable, with dark colors (black, dark grey, dark blue, dark green) scored higher.

**Incremental scan strategy with newest-first sort:**
1. Fetch page 1 (20 listings)
2. For each listing ID: check if it exists in the tracker
3. If ALL 20 are known → market is stable, stop scanning
4. If any are NEW → fetch their detail pages, score them, alert if gem
5. Only paginate to page 2+ during weekly full-market snapshots (Sunday night)

### mobile.de technical notes

- They use Akamai bot protection. `httpx` with rotating UAs is **not sufficient** — gets blocked within 5–10 requests.
- **What works:** Real browser (Playwright + stealth plugin, or Chrome extension), or parsing email alerts (recommended).
- **Undocumented JSON API:** The search results page makes XHR calls to a JSON API endpoint. Discoverable via browser DevTools Network tab. More stable than HTML parsing but still requires browser-like headers.
- Listing detail pages contain: structured data in `<script type="application/ld+json">`, full description, all photos, seller contact info (sometimes behind a "show number" JS button).
- **Cookie consent:** GDPR wall appears on first visit. Click "Ablehnen" (decline) to dismiss. Must handle this in any browser-based approach.

### General scraper pattern

Each scraper inherits from `BaseScraper`:

```python
class BaseScraper(ABC):
    def __init__(self, config: SiteConfig, http_client: httpx.AsyncClient):
        self.config = config
        self.client = http_client

    @abstractmethod
    async def build_search_url(self, search_config: SearchConfig) -> str:
        """Build the search results URL with filters."""

    @abstractmethod
    async def parse_search_results(self, html: str) -> list[PartialListing]:
        """Extract listing previews from search results page."""

    @abstractmethod
    async def parse_listing_detail(self, html: str) -> Listing:
        """Extract full listing data from a detail page."""

    async def scrape(self, search_config: SearchConfig) -> list[Listing]:
        """Full scrape pipeline: search → paginate → detail pages."""
        url = await self.build_search_url(search_config)
        listings = []
        page = 1
        while True:
            html = await self._fetch(f"{url}&pageNumber={page}")
            results = await self.parse_search_results(html)
            if not results:
                break
            for partial in results:
                await asyncio.sleep(self.config.delay_seconds)
                detail_html = await self._fetch(partial.detail_url)
                listing = await self.parse_listing_detail(detail_html)
                listings.append(listing)
            page += 1
        return listings
```

---

## 11. Scoring implementation notes

### Prompt engineering

The system prompt for scoring should be comprehensive and include:
1. The full vehicle spec table from Section 2
2. The scoring rubric with point ranges
3. Explicit instructions to output valid JSON matching the schema
4. Examples of good vs. bad scores with reasoning

Use Claude's **tool use** feature to enforce the output schema. Define a `score_listing` tool with the exact JSON schema from Section 4.2, and instruct Claude to call that tool with its analysis.

### Image analysis

When scoring, send images to Claude's vision capability. Key things Claude should look for in photos:
- **Color accuracy**: Compare against reference photos of Jet Green and Forest Green (include 2–3 reference images in the system prompt as base64).
- **Mod identification**: Identify tire brand/model from sidewall text, spot spacers by wheel offset, identify lift from wheel gap.
- **Condition**: Paint condition, interior wear, rust, dashboard warning lights.
- **Authenticity**: Check for signs of respray (overspray on rubber seals, mismatched color in door jambs vs. body).

### Cost management

At ~$0.003 per scoring call (Sonnet, with images), and assuming 20–50 new listings per day, daily scoring cost should be $0.06–$0.15. Negligible. Don't over-optimize here.

---

## 12. Telegram bot implementation notes

### Bot commands

| Command | Action |
|---|---|
| `/start` | Register chat, confirm connection |
| `/status` | Current stats: active listings, negotiations, last scrape time |
| `/hot` | List all current hot leads (score >= 75) |
| `/search` | Trigger an immediate scrape cycle |
| `/stop` | Kill switch — pause all haggle agents |
| `/resume` | Resume haggle agents |
| `/config` | Show current config |

### Inline keyboard callbacks

Use `callback_data` format: `action:listing_id` (e.g., `engage:abc123`, `skip:abc123`, `watch:abc123`).

---

## 13. Haggle agent implementation notes (V2)

### Email threading

- Set `In-Reply-To` and `References` headers correctly to maintain thread in seller's email client.
- Parse incoming emails by matching `In-Reply-To` against our sent `Message-ID`.
- Strip email signatures and quoted text before feeding replies to Claude.

### Claude negotiation prompt

```
System: You are Max Weber, a car enthusiast in Germany negotiating to buy a Porsche Cayenne S.
You write in fluent, natural German — Umgangssprache, not formal business German.
You are knowledgeable about Porsches and know exactly what this car is worth.

Rules:
- Your maximum budget is €{walk_away_price}. NEVER agree to more.
- You've made {counteroffer_count} counteroffers so far. Max allowed: {max_counteroffers}.
- Your last offer was €{current_offer}. You can increase by max €{max_increment}.
- Be warm and enthusiastic about the car but firm on price.
- If the seller won't budge, express understanding and say you'll think about it.
- NEVER mention you are an AI, automated system, or agent.
- NEVER agree to send deposits, bank transfers, or financial details.
- Keep emails under 100 words. Real people don't write essays.

Current negotiation state: {state}
Full email thread so far:
{email_thread}

Seller's latest reply:
{latest_reply}

Compose your next email reply. Output ONLY the email body text, no subject line, no metadata.
```

### Handling "contact seller" forms

Many listings don't expose email directly. Options:
1. **Scrape the contact form and submit it** with a message that includes your email, prompting the seller to reply. This works on Kleinanzeigen and some dealers.
2. **Flag to user**: If no email and no form, notify user on Telegram and ask them to provide the seller's email manually.
3. **Phone fallback**: Some listings only have phone numbers. V2 doesn't handle phone calls. Flag these for manual follow-up.

---

## 14. Deployment

### Docker

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY pyproject.toml .
RUN pip install .
COPY src/ src/
COPY config.yaml .
CMD ["python", "-m", "src.main"]
```

```yaml
# docker-compose.yml
services:
  cayenne-hunter:
    build: .
    env_file: .env
    volumes:
      - ./data:/app/data      # SQLite DB + images
      - ./config.yaml:/app/config.yaml
    restart: unless-stopped
```

### Monitoring

- Structlog outputs JSON to stdout → pipe to any log aggregator.
- The `/status` Telegram command is your monitoring dashboard.
- If a scrape run fails entirely, send a Telegram alert.

---

## 15. Development sequence

### Phase 1: Foundation (Day 1)
1. Project scaffolding, pyproject.toml, Docker setup
2. Config loader (pydantic-settings + yaml)
3. SQLite database setup with all tables
4. Logging setup (structlog)

### Phase 2: Scraper (Days 2–3)
1. Base scraper class
2. mobile.de scraper (hardest, do first)
3. AutoScout24 scraper
4. Kleinanzeigen scraper
5. Image downloader
6. Dedup logic
7. Scheduler integration

### Phase 3: Scorer (Day 4)
1. Claude API integration
2. Scoring prompt + tool use schema
3. Score storage
4. Price change detection + re-scoring

### Phase 4: Notifier (Day 5)
1. Telegram bot setup
2. Notification formatting
3. Inline keyboard handlers
4. Bot commands

### Phase 5: Haggle Agent — V2 (Days 6–8)
1. Email client (SMTP/IMAP)
2. Negotiation state machine
3. Claude negotiation prompts
4. Telegram approval flow
5. Safety rails (daily cap, kill switch, price ceiling)

### Phase 6: Hardening (Day 9–10)
1. Error handling across all components
2. Retry logic for flaky scrapes
3. Tests for scoring, negotiation state machine, dedup
4. README + deployment docs

---

## 16. Success criteria

The system is working when:

1. **Scraper** finds all green first-gen Cayenne S listings across enabled sites within 30 minutes of them being posted.
2. **Scorer** correctly identifies Jet Green / Forest Green with >90% accuracy and meaningfully differentiates between stock cars and modded builds.
3. **Notifier** delivers a Telegram alert within 5 minutes of a hot lead being scored.
4. **Haggle agent** (V2) successfully conducts a multi-round price negotiation in natural German without the seller suspecting automation, stays within budget constraints, and hands off to the human at the right moment.
5. **The dream outcome**: A notification pings your phone showing a Jet Green Cayenne S with KO2s, spacers, a small lift, and blacked trim — listed at €9k by someone who spent €4k on mods and just wants it gone. Claude scores it 90+, you tap "Engage," and three emails later you have a handshake deal at €7,500. You show up with a trailer and drive home with the exact truck you wanted, built by someone else's wallet.
