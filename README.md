<p align="center">
  <img src="docs/hero-banner.png" alt="A mentor and a student at a lamp-lit desk of notebooks, programming books, and a glowing IDE — late-night live coding as craft." width="100%">
</p>

# Live Coding Bible

**A playbook for data-heavy civic dashboards — production tactics you can reuse, not theory.**

[![License: MIT](https://img.shields.io/badge/license-MIT-1A1A1A)](LICENSE)

By [Dr Non Arkaraprasertkul](https://github.com/Nonarkara) — architect, urban anthropologist, Senior Expert in Smart City Promotion at Thailand's Digital Economy Promotion Agency (depa), and founder of [Axiom](https://axiom.nonarkara.org).

This is independent studio writing from a Bangkok civic studio. It is **not** an official depa, ASEAN, or municipal product.

---

## What this is

A living record of tactics already running in production civic systems — flood watch, air quality, municipal control towers, satellite maps, open city indexes. Each entry exists because a real build needed it, and several exist because production broke first.

This repository is the **playbook**, not the source tree of those systems. There is no app to clone and deploy here. What you get is the pattern: the job it does, why it exists, a sketch you can port, and the anti-pattern that looks similar and fails.

It is written for two readers at once:

- **A human** about to build a civic dashboard who would rather steal a proven move than rediscover it.
- **An agent** starting a session who should read this before inventing a new architecture.

If a tactic is in this file, treat it as prior art. Check here first.

**This repo is not:**

- Source code for FloodDash, AirDash, or any municipal tower. Those live in their own repositories; some implementations stay private.
- An official warning system, city ranking, or government publication.
- A dump of workspace paths, analytics tokens, database URLs, or spreadsheet IDs. Those do not belong in a public playbook.

Related public work: [FloodDash Blueprint](https://github.com/Nonarkara/FloodDash-Blueprint), [AirDash](https://github.com/Nonarkara/airdash), [NST control tower](https://github.com/Nonarkara/nst-control-tower), [DrNon Global Satellite Toolkit](https://github.com/Nonarkara/DrNon-Global-Satellite-Toolkit), [vibecoding skills](https://github.com/Nonarkara/dr-non-vibecoding-skills).

---

## Philosophy

The signature move: **show different data sources on the same axis** so a person can see a correlation nobody pre-announced. Do not optimize milliseconds. Optimize for:

1. **Surprise** — two things on one surface that nobody thought to look at together.
2. **Reward** — the feeling of capability that comes from acting on good information.
3. **Trust** — the system earns attention by being honest about what it does not know.

Everything else follows from this.

Code is communication with future you. Write the reason next to the oddity; an agent will otherwise "clean it up." Prefer a map that *is* the page over a dashboard that hides the city in a widget. Prefer a 5-minute cache that feels live over a websocket you cannot afford. Prefer a visible stale number over a green dot that lies.

The hero at the top is the craft, not a title card: a mentor and a student, SOLID on the desk, a learning cycle that says understand → try → reflect → improve. That is the studio. The parchment is left blank on purpose. The illustration is the HUD.

---

## Ethical use

These patterns are for **public-good civic software**: honest situational awareness, open data, systems a city can run without a vendor lock-in. They are not a kit for surveillance, dark patterns, or pretending a private feed is an official alert.

**Do**

- Label freshness. Every live-looking number needs a source, an age, and a fallback tier. Stale, modelled, and missing are different states; the UI must say which.
- Keep analytics cookie-free and aggregate when you can. Prefer provider pixels that do not fingerprint. Store any beacon token as an environment variable, never in git.
- Correlate **public signals** (weather with water; news with sensors). Do not build "correlation" as a way to track individuals.
- Attribute upstream data. The number belongs to HII, GISTDA, Open-Meteo, Traffy, NASA — whoever produced it.
- Degrade in public. If the laptop sleeps or an API dies, the page still loads and says the data is old.

**Do not**

- Ship mock data as live, or hide an empty feed behind a success state.
- Commit API keys, analytics tokens, database hosts, spreadsheet IDs, tunnel credentials, or personal capture endpoints.
- Imply depa, ASEAN, a municipality, or a UN body publishes this playbook or the systems it describes, unless that system's own README says so.
- Use fleet-health pings, visitor beacons, or second-brain capture pipelines to collect more personal data than the product needs.
- Treat this file as authorization to copy a private implementation. Rebuild from the idea.

If a contribution would only work by pasting a secret, it does not belong here. Describe the pattern; leave the credential in the operator's environment.

---

## How to use the patterns

Read the index. Pick the row that matches the job. Port the sketch; do not hunt a private workspace for the original file. When you adapt a snippet, put tokens in env vars.

Agents: this README is the playbook. Do not scan a 65-repo tree looking for `tactics/*.md`. Those files are not in this repository.

| # | Tactic | In one line |
|---|--------|-------------|
| 01 | [Multi-source correlation](#01-multi-source-correlation) | Different public sources on one axis |
| 02 | [Illusion of real-time](#02-illusion-of-real-time) | A 5-minute cron that feels live |
| 03 | [Fleet health monitor](#03-fleet-health-monitor) | One worker pings every hostname |
| 04 | [Satellite-first design](#04-satellite-first-design) | The map is the page, not a widget |
| 05 | [Second-brain pipeline](#05-second-brain-pipeline) | Thought → local + searchable + human sheet |
| 06 | [Gamification layer](#06-gamification-layer) | Deep work as honest game mechanics |
| 07 | [Unified visitor analytics](#07-unified-visitor-analytics) | One cookie-free beacon, every host |
| 08 | [Plan / room architecture](#08-plan--room-architecture) | Phone-first plan; 3D as opt-in |
| 09 | [Poison-proof CDN deploy](#09-poison-proof-cdn-deploy) | Verify bytes, not the version string |
| 10 | [Three-job service pattern](#10-three-job-service-pattern) | Server + tunnel + watchdog, never one job |
| 11 | [Anti-regression ledger](#11-anti-regression-ledger) | Numbered "do not touch," each with a why |
| 12 | [Lesson docs / CPDT trace](#12-lesson-docs--cpdt-trace) | One doc per hard session; one line for the next agent |
| 13 | [Graceful degradation split](#13-graceful-degradation-split) | CDN frontend, laptop backend, mock fallback |
| 14 | [Shared data catalog](#14-shared-data-catalog) | Catalogue a source once, port the adapter forever |

### 01 Multi-source correlation

**The tactic:** Normalize unrelated public sources and plot them together. The correlation — or the lack of it — *is* the insight. Do not predict what people will find interesting. Make the comparison possible.

**Seen in production:** a daily brief with FX, crypto, equities, gold, and oil on one grid; weather + AQI on the same window as markets; aerosol + conflict events + headlines on one map; weather + flights + tourist volume on one ops view.

```js
const [fx, crypto, weather, aqi] = await Promise.all([
  fetch('/api/fx'), fetch('/api/crypto'),
  fetch('/api/weather'), fetch('/api/aqi')
]);
// Normalize to comparable units. Render on one grid.
// Never pre-conclude the correlation for the reader.
```

**Anti-pattern:** a "feature" that announces "USD is correlated with BTC." Build the surface. Trust the reader.

### 02 Illusion of real-time

**The tactic:** A five-minute cron that hits an API, stores the result in KV, and serves it to browsers looks identical to a live stream for almost every civic use. The cases that need a true websocket can pay for one.

**Cost math:** a managed websocket is tens to hundreds of dollars a month plus engineering. Cron + KV on a worker free tier is a few lines. User-visible difference is zero unless the job is arbitrage.

**Anti-pattern:** streaming infrastructure as a default because "live" is in the brief.

### 03 Fleet health monitor

**The tactic:** One worker probes every public hostname on a timer, stores a snapshot, and serves `/status`. The browser paints from `localStorage` first, then refreshes. Adding a site is a hostname in a list, not a deploy to that site.

```js
const results = await Promise.all(DOMAINS.map(probe));
await env.STATUS.put('snapshot:v1', JSON.stringify({ ts, sites }));

const cached = localStorage.getItem('studio.status.snapshot');
if (cached) paintStatus(JSON.parse(cached));
```

Amber = OK, red = fail, dim = unknown. The probe is for *your* public surfaces, not for scanning other people's networks.

### 04 Satellite-first design

**The tactic:** The map is the page. Everything else is an overlay. No boxed layout, no sidebar wider than the eye can track, no panel that cuts the city in half.

> If a UI element can only be seen by scrolling *past the map*, it does not exist for most users.

Every data surface must be reachable without leaving the map viewport — a collapsed overlay or a slide-in from an edge.

**Anti-pattern:** a dashboard with a map widget in a card grid.

### 05 Second-brain pipeline

**The tactic:** A captured note goes three places that never replace each other: local storage (instant, offline), a searchable store (vector or full-text), and a human-readable sheet or log. Each layer has a job. The pipeline is the product, not the vendor names.

```
User types a note
  → localStorage (always works offline)
  → POST /capture
      → searchable store + embedding
      → a row a human can read
```

Hosts, project refs, and sheet IDs stay in the operator's environment. They are not part of the pattern.

### 06 Gamification layer

**The tactic:** Earn attention by making progress visible, reward tangible, and the cost of distraction concrete. Not leaderboards. The quiet fact of a timer, a streak, or a broken-focus count.

1. **A thing to protect** — uninterrupted time, a streak, a daily count.
2. **A reward that isn't fake** — a painting, a line of philosophy, something worth looking at.
3. **An honest cost** — "you broke focus N times" as signal, not shame.

### 07 Unified visitor analytics

**The tactic:** One cookie-free analytics beacon covers every hostname you actually operate. Behavioral counts (visitors, pages, countries, referrers) from the host platform; contextual events only if you still need them, written to a store you control.

Keep the token in environment config. Do not paste it into a README, a tactic file, or a public snippet. GDPR-clean means you do not need a consent wall for aggregate, non-identifying traffic — it does not mean "collect everything."

**Anti-pattern:** thirty inline copies of the same tracker, each with a different hardcoded id.

### 08 Plan / room architecture

**The tactic:** Phone users land on a 2D plan — a data surface that works with one thumb. Desktop users who want immersion can enter a 3D room. The toggle persists. Music, notes, timers work from the plan.

The 3D room is not the product. The plan is the product. The room is the experience layer. Separating them means you never compromise either.

### 09 Poison-proof CDN deploy

**The tactic:** "Deploy succeeded" means the origin has new bytes. Edge nodes converge independently. New HTML against stale JS can cache the wrong payload under a new `?v=` key forever. Do not verify by curling the real URL early. Probe through throwaway keys so a stale response can only poison a key nobody will request again.

```bash
probe_asset() {  # never request the real ?v= key until content is proven
  local want=$(md5_of "public/$2") got streak=0
  for i in $(seq 1 24); do
    got=$(curl -fsS "$1/$2?${EXPECTED}&probe=$i" | md5_of /dev/stdin)
    [[ "$got" == "$want" ]] && streak=$((streak+1)) || streak=0
    [[ $streak -ge 3 ]] && return 0
  done
  return 1  # never converged — do NOT request the real key
}
```

Localhost is never a deliverable. Neither is the deploy tool's success line. A deploy is a human receiving new bytes.

### 10 Three-job service pattern

**The tactic:** An always-on service gets three supervised jobs, never one: the process (`KeepAlive`), the tunnel (its **own** config file — never the shared fallback), and a watchdog that polls health, restarts, and escalates to a human instead of looping forever. Plus a nightly backup.

A tunnel binary with no `--config` silently falls back to a shared default. Two tunnels can overwrite each other's ingress with no error. Make the shared fallback deliberately inert:

```yaml
# Shared fallback only. Every tunnel loads its own file via --config.
ingress:
  - service: http_status:404
```

A watchdog that checks the wrong thing is worse than none — it manufactures confidence. "Process is running" is not "last successful ingest was recent."

### 11 Anti-regression ledger

**The tactic:** Every project instruction file opens with a numbered, dated **do not touch** list — each item paired with *why*. Agents do not vandalise, they tidy. Anything that looks like an oddity gets cleaned up unless the reason it is deliberate is written down next to it.

```
1. Zero border-radius — enforced in globals.css. Do not remove the
   `border-radius: 0 !important` reset. It is load-bearing.
2. Three font sizes only — Display / Body / Micro. Do not introduce a fourth.
3. Mock data in src/lib/api/mock.ts — the app must render with no API keys.
```

Reverted experiments get the same treatment, dated, so a half-remembered good idea does not quietly return.

### 12 Lesson docs / CPDT trace

**The tactic:** After a hard session, write `docs/lessons/YYYY-MM-DD-the-<something>-pass.md`: what was asked (verbatim), what landed, patterns borrowed (including what was **refused** and why), honest limits, what did not make the cut, the real ship trace, and **one line for the next agent**.

**CPDT** — the ship is not the git push:

```
git pull
git add -A && git commit && git push
npm run build                  # must succeed
deploy to the real host
curl the public URL            # assert the new surface is in the body
```

A dormant project becomes productive when "why is it like this, and what did we already try" already has a written answer.

### 13 Graceful degradation split

**The tactic:** Static frontend on a CDN, talking to a function that proxies `/api/*` with no hard-coded route list, through a named tunnel, to `localhost` on a machine you own. If that machine sleeps, the site still loads — only live data goes stale, and the UI says how stale. Ship `mock.ts` so the app renders with zero keys.

An upstream dying should degrade the app, not blank it. Degradation that is not visible is a lie: a green dot plus "data 12 minutes old," never a silently stale number.

### 14 Shared data catalog

**The tactic:** One catalog across projects: source, cadence → **real latency** (two numbers — "updates every 10 min" and "data is 10–60 min old" are both true; only the second matters to the UI), auth, tier, and a live / port-me status. Before wiring a new adapter, check the catalog and port the known-good implementation.

The catalog grows from real builds, not from a documentation sprint that never happens. A flood dashboard and an air-quality dashboard can share one ingest backbone because the sources were written down somewhere neither project owned.

---

## How to contribute

Append. Do not rewrite the philosophy to taste.

A new tactic earns its number when it has already shipped somewhere, not when it sounds wise. PRs should add:

1. **One line** — the job in a sentence.
2. **Why it exists** — the incident, constraint, or civic need. No unnamed drama.
3. **The pattern** — enough to port. Sketches may use placeholders (`env.STATUS`, `/api/health`). Never real tokens, hosts, sheet IDs, or personal endpoints.
4. **The anti-pattern** — the nearby mistake.
5. **A public example, if you have one** — a live URL or a public repo. Skip private paths.

Open a pull request against `main`. Keep the voice: production, not theory; civic, not vendor pitch. If you are unsure whether a string is a secret, it is — leave it out.

Fixes to prose, ethics, and missing anti-patterns are as welcome as new tactics.

---

## License

This repository is licensed under the [MIT License](LICENSE). Copyright © 2026 Non Arkaraprasertkul.

Reuse the ideas, sketches, and prose with attribution. The MIT grant covers **this playbook**. It does not relicense upstream data, municipal identities, or private implementations described as examples.

If you build something with these patterns, I would like to see it.
