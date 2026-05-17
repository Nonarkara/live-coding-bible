# Dr Non's Playbook — How I Build Things

**Read this first. Every session. It saves tokens.**

This is the living record of Dr Non's recurring techniques and tactics. Not theory — each entry is a pattern already deployed in production, with exact file references. When you're about to build something, check here first. The answer is probably already built.

---

## Who This Is For

Claude Code sessions starting on any project in this workspace. If you're in a new session and haven't read this, stop and read it. You'll spend the first 20 minutes re-discovering things that are already solved.

## The Core Philosophy (30 seconds)

Dr Non's signature move: **show different data sources on the same axis to reveal correlations that nobody thought to look for**. He doesn't optimize milliseconds. He optimizes for:
1. **Surprise** — the moment a user sees two things together that they never expected to correlate
2. **Reward** — the feeling of capability that comes from acting on good information
3. **Trust** — the system earns the user's attention by being honest about what it doesn't know

Everything else follows from this.

---

## Tactics Index

| # | Tactic | In One Line | Reference |
|---|--------|-------------|-----------|
| 01 | [Multi-Source Correlation](#01-multi-source-correlation) | Different data sources on one axis | `tactics/01-correlation.md` |
| 02 | [Illusion of Real-Time](#02-illusion-of-real-time) | 5-min cron that feels live | `tactics/02-realtime.md` |
| 03 | [Fleet Health Monitor](#03-fleet-health-monitor) | Ping all subdomains → one dashboard | `tactics/03-fleet-health.md` |
| 04 | [Satellite-First Design](#04-satellite-first-design) | Map is the page, not a widget | `tactics/04-satellite-first.md` |
| 05 | [Second Brain Pipeline](#05-second-brain-pipeline) | Thought → Supabase vector + Sheets | `tactics/05-second-brain.md` |
| 06 | [Gamification Layer](#06-gamification-layer) | Deep work as game mechanics | `tactics/06-gamification.md` |
| 07 | [Unified Visitor Analytics](#07-unified-visitor-analytics) | One token, every subdomain | `tactics/07-visitor-analytics.md` |
| 08 | [Plan / Room Architecture](#08-plan--room-architecture) | Phone-first dashboard + 3D opt-in | `tactics/08-plan-room.md` |

---

## 01 Multi-Source Correlation

**The tactic:** Normalize data from unrelated sources and plot them together. The correlation — or lack of it — IS the insight. You don't predict what people will find interesting. You make the comparison possible and let them discover it.

**Examples in production:**
- `nonarkara-org/app.js` — USD/THB + BTC + SET + Gold + Brent + PTT on one brief grid
- `nonarkara-org/app.js` — Bangkok weather + AQI on the same time window as market data
- `conflict-tracker/v3-global/` — satellite aerosol layer + conflict events + news headlines on one map
- `phuket/dashboard/` — weather + AQI + flight arrivals + tourist volume on one ops view

**The pattern:**
```js
// 1. Fetch all sources in parallel (never sequential)
const [fx, crypto, weather, aqi] = await Promise.all([
  fetch('/api/fx'), fetch('/api/crypto'),
  fetch('/api/weather'), fetch('/api/aqi')
]);

// 2. Normalize to the same units if comparing (%, or same scale)
// 3. Render on the same grid — user draws their own conclusions
// 4. Never pre-conclude the correlation for them
```

**Anti-pattern:** Building a "feature" that says "USD is correlated with BTC." That's lazy. Build the surface. Let the user see it. Trust their intelligence.

---

## 02 Illusion of Real-Time

**The tactic:** A 5-minute cron job that hits an API, stores the result in KV, and serves it to browsers looks identical to a live stream — for 99% of use cases. The 1% that needs true real-time pays for it.

**Full documentation:** `_toolkit/claude-skills/claude-skills/ninja-innovation/SKILL.md`

**Examples in production:**
- `nonarkara-org/worker/src/index.js` — `/daily-brief` endpoint, all market quotes cached in KV for 5 min
- `nonarkara-org/worker/src/index.js` — `/status` fleet health, KV-cached, cron every 5 min
- `conflict-tracker/v3-global/` — ACLED + GDELT + NASA FIRMS fetched once per cron, served instantly

**The cost math:**
- Real-time WebSocket stream: ~$50–200/month (Pusher/Ably), engineering overhead
- 5-min cron + KV: $0 (Cloudflare free tier), 3 lines of Worker code
- User-visible difference: zero, unless their job is arbitrage trading

---

## 03 Fleet Health Monitor

**The tactic:** A single Cloudflare Worker pings every subdomain every 5 minutes, stores results in KV, serves a `/status` JSON endpoint. The browser polls every 3 minutes. Every project's status dot updates without touching those projects.

**Reference implementation:** `nonarkara-org/worker/src/index.js`

**The 3 pieces:**
```js
// 1. Worker cron: probe all domains in parallel
const results = await Promise.all(DOMAINS.map(probe));
await env.STATUS.put('snapshot:v1', JSON.stringify({ ts, sites }));

// 2. Client: cache in localStorage, poll every 3 min
const cached = localStorage.getItem('nonarkara.status.snapshot');
if (cached) paintStatus(JSON.parse(cached));   // instant on load

// 3. UI: amber dot = OK, red = fail, dim = unknown
```

**Why it matters:** 28+ subdomains. One worker. Zero per-project code changes needed to add a new site — just add its hostname to the DOMAINS array.

---

## 04 Satellite-First Design

**The tactic:** The map/satellite layer is the page. Not a component inside a page — the page. Everything else is an overlay on top of the map. This means: no boxed layout, no sidebar wider than the eye can track, no panels that cut the map in half.

**Full documentation:** `_toolkit/claude-skills/claude-skills/non-app-pattern/SKILL.md` (§ Reference implementation)

**Examples:**
- `phuket/dashboard/phuket-dashboard/` — Leaflet + deck.gl map fills viewport; three panels pin to corners
- `conflict-tracker/v3-global/` — MapLibre fills screen; TVs are overlays
- `asean/kuching-ioc/` — Leaflet base; data layers toggle as transparent overlays
- `consulting/chula/apps/web/` — Map center; everything else positions around it

**The rule that breaks most dashboards:**
> If a UI element can only be seen by scrolling *past the map*, it does not exist for 90% of users.

Every data surface must be reachable without leaving the map viewport, either as a collapsed overlay or a slide-in panel from an edge.

---

## 05 Second Brain Pipeline

**The tactic:** Every thought, note, or observation captured in the plan view goes three places simultaneously: localStorage (instant, offline), Supabase `captures` table (searchable via pgvector), Google Sheets (human-readable, shareable). The three layers serve different purposes and never replace each other.

**Reference implementation:**
- Capture endpoint: `nonarkara-org/worker/src/index.js` → `POST /capture`
- Supabase schema: `nonarkara-org/worker/migrations/second-brain-schema.sql`
- Google Apps Script: `nonarkara-org/worker/apps-script/second-brain-sheet.js`
- Client call: `nonarkara-org/app.js` → NOTE button handler

**The pipeline in one diagram:**
```
User types note
  → localStorage (instant, always works offline)
  → POST /capture to Worker
      → Supabase captures table (pgvector for semantic search)
      → Google Sheets row (for human review, formula analysis)
      → OpenAI embedding (1536-dim, stored back to Supabase)
```

**Supabase project:** `qoagbsslzgaflwjmguej.supabase.co` (Second Brain v2)
**Sheets:** `1DM66spLCh_PKJ0hncFBWVft_H-3UiePONynjTltSvQg`

---

## 06 Gamification Layer

**The tactic:** The interface earns the user's attention by making progress visible, making reward tangible, and making the cost of distraction concrete. Not leaderboards or badges — the quiet satisfaction of a Pomodoro timer that shows your completion rate, a step count, or a focus streak.

**The three elements:**
1. **A thing to protect** — uninterrupted focus time, a streak, a daily count
2. **A reward that isn't fake** — museum art during a Pomodoro session, rotating philosophical quotes, a sense of aesthetic pleasure
3. **An honest cost** — the Pomodoro shows "you broke focus X times this week" not as shame but as signal

**Examples in production:**
- `nonarkara-org/app.js` — Pomodoro with 21 Nonist quotes, FRAME mode with 47 museum paintings
- `nonarkara-org/art-manifest.json` — 47 CC0 paintings from Met + AIC with Non-voiced notes
- `TKC/talent-support-dashboard/` — DQ3 board game as HR management system

**The Pomo quote pattern:**
```js
// Rotate a quote every 40s using Web Animations API (not CSS transitions —
// those break when parent transitions from display:none)
el.animate([{ opacity: 0 }, { opacity: 1 }], { duration: 600, fill: 'forwards' });
```

---

## 07 Unified Visitor Analytics

**The tactic:** One analytics token (`0da324da0e204440a172088c0fafc92c`) covers all subdomains via Cloudflare Web Analytics. One Google Sheets tracker module covers custom events. Both are cookie-free, GDPR-clean, and require zero consent banners.

**The canonical module:** `_shared/lib/visitor-tracking.js`

**The two layers:**
1. **Cloudflare Web Analytics** (behavioral) — unique visitors, top pages, countries, referrers. Zero JS needed on CF Pages sites; one `<script defer>` on external hosts.
2. **Google Sheets pipeline** (contextual) — country, city, IP, device, timezone, referrer — every detail you can't get from Cloudflare alone.

**Current status:** Beacon added to all 16 active subdomains. See `_shared/lib/visitor-tracking.js` for the canonical module that replaces the 30+ inline implementations.

---

## 08 Plan / Room Architecture

**The tactic:** Phone users (the majority) land on a 2D plan view — a clean data dashboard that works with one thumb. Desktop users and those who want immersion can enter the 3D room view. The toggle persists in localStorage. Music, notes, Pomodoro — everything works from the plan view.

**Full documentation:** `_toolkit/claude-skills/claude-skills/non-app-pattern/SKILL.md`

**Reference:** `nonarkara-org/app.js` — the entire thing. Single HTML file + single JS module.

**The insight:** The 3D room is not the product. The plan view is the product. The room is the experience layer for when someone wants to feel the depth of what they're looking at. Separating them means you never compromise either.

---

## Shared Resources Quick Reference

| Resource | Path | What it is |
|---|---|---|
| Brand logos | `_shared/brand-assets/` | AXIOM, DEPA, RETL, SLIC, PMUA, Smart City Thailand |
| Portraits | `_shared/photos/slic/Photos/` | `profile-speaker.jpg`, `profile-group-depa.jpg`, etc. |
| Knowledge base | `_shared/knowledge-base/` | 100+ PDFs: smart city, SLIC methodology, ASEAN |
| Geodata | `_shared/geodata/` | GeoJSON boundaries, telemetry schemas |
| Design tokens | `_shared/design-tokens/dr-non-brand.css` | Canonical CSS variables — the single source of truth |
| Visitor tracker | `_shared/lib/visitor-tracking.js` | Canonical module — import, don't copy |
| Tech stack DB | `_toolkit/tech-stack-database/` | All projects, APIs, costs in CSV + Excel |

---

## What Doesn't Exist Yet (Build Next)

- `GET /query?q=...` semantic search endpoint on the Worker (needs pgvector `match_captures` function already in Supabase)
- Obsidian daily-note sync from Supabase captures (obsidian-capture-bot is built, just not wired)
- iOS app using the same Worker endpoints as nonarkara.org (Swift code skeleton in `non-app (council)/ios-reference/`)
- Apple Watch step count integration (HealthKit → Worker → captures table)

---

*This file grows. When you build something new that uses a pattern worth keeping, add it here. Append, don't rewrite.*
