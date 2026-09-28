# ArchitectWizard — Project Story

## Inspiration
Federal regulators (FERC) issued **Order No. 1920** in 2024 because utilities plan construction in isolation, causing duplicated work and delays. The **Sperry Tech GridLock Challenge** asks for a tool that compares at least two utilities' future plans and flags overlaps — timeline and geographic — so they can share crews, equipment, and right-of-way.

We saw the same problem in Florida: county-level construction plans (road, bridge, substation) overlap near borders, but no live coordination layer exists. We built a wizard-guided platform that turns spoken or typed project plans into structured data, checks them against real upcoming Florida work, and recommends the nearest free window.

## What it does
**Wizard + Voice:** An interactive wizard (`wizard-voice.js`) speaks in a consistent British male voice (`pitch=0.72`, `rate=0.78`) across `index.html`, `onboarding.html`, `intake.html`, `map.html`, and `conflict_check.html`. Users click a microphone, speak a project (county, dates, company), and ElevenLabs STT fills the form.

**Conflict Detection:** After intake, `POST /api/projects/conflicts` (direct backend) checks temporal overlap in MongoDB Atlas (`hackathon.projects`). If overlaps exist, `conflict_check.html` displays conflicts, nearest-window remediation, and wizard speech. The map (`map.html`) refreshes automatically via `BroadcastChannel('sperry-refresh')` when a save occurs.

**Real Data:** We seeded 136 real-style Florida construction entries (FDOT bridge/road/substation projects, max 2 per county) with future dates (`2026–2027`) so conflict detection is meaningful.

**Live Domain:** Deployed to Render; GoDaddy domain `shadow-wizard-architect-gang.us` verified (`A` record `216.24.57.1`, `CNAME` `www` → `architectwizard.onrender.com`).

## How we built it

**Frontend:** Vanilla HTML/JS (`frontend/public/`). `wizard-voice.js` controls SpeechSynthesis settings and wizard utterances. `intake.html` has county dropdown (all Florida counties, `Dade` mapped to `Miami-Dade`), `toUTCISO()` date conversion, `localStorage` draft save, `sessionStorage` conflict load, and `BroadcastChannel` refresh. `map.js` renders Leaflet markers by county with `fetchCountyProjects()`.

**Backend:** `express` server (`backend/server.js`). `MongoClient` connects to Atlas. Compound index `{county: 1, startTime: 1, endTime: 1}` supports fast overlap queries. `POST /api/projects` validates title/county/dates, applies past-date guard (`startTime < today` rejected), and inserts. `GET /api/projects/county/:county` filters by `endTime >= now` so finished work is hidden.

**AI / NLP:** `backend/ai.js` initializes Vertex AI (`gemini-3.8-flash`, `vertexai: true`). `extractProjectFields()` parses text/audio into structured fields. `/api/ai/check-conflicts` runs extraction + overlap + `nearestWindow` computation.

**Voice Template:** `wizard-voice.js` prefers British male voices (`Daniel`, `Thomas`, `Arthur`) with `pitch: 0.72`, `rate: 0.78`.

**Deployment:** `package.json` with `npm run deploy`; `.env` holds `MONGO_URI`, `ELEVENLABS_API_KEY`, `GEMINI_API_KEY`, `GOOGLE_APPLICATION_CREDENTIALS`, `PRODUCTION_URL`. Render auto-deploys `main`; build uses `npm install`, start uses `node backend/server.js`.

## Challenges we ran into

1. **Gemini 429 / free tier rate limit:** Free Gemini tier returned `429` under load. We migrated `backend/ai.js` to Vertex AI (`vertexai: true`) and switched the intake flow from `/api/ai/check-conflicts` to direct `POST /api/projects` + `GET /api/projects/conflicts` to reduce AI calls.

2. **Timezone date shifts:** User dates were shifting by local timezone when converted to MongoDB. Added `toUTCISO()` in intake and `fmtUTC()` in display so `startTime` and `endTime` stay consistent across pages.

3. **Root CNAME error:** GoDaddy DNS rejected `CNAME` on root domain (`shadow-wizard-architect-gang.us`). Used Render's provided `A` record (`216.24.57.1`) for `@` and kept `CNAME` for `www`.

4. **Past projects showing up:** Before a filter, finished projects appeared on the map and blocked new conflicts incorrectly. Added `endTime: { $gte: now }` to county endpoint and relied on overlap logic (with future-start guard) for conflicts.

5. **Conflict page refresh empty:** `sessionStorage.removeItem('conflictCheck')` on page load erased results on refresh. Removed immediate deletion; `intake.html` clears it via `pageshow` when returning.

6. **Real data missing:** Original `projects.json` only covered `Miami-Dade` and `Orange`. Built `seed_real.py` to cap at 2 real-style FDOT entries per Florida county (136 total), then inserted via `node` MongoDB script.

## Accomplishments that we're proud of

- **Live domain** with full GoDaddy DNS verification.
- **Consistent wizard voice** across all 5 pages using shared `wizard-voice.js`.
- **Real-time map refresh** via `BroadcastChannel` — no reload needed.
- **136 real Florida projects** in MongoDB with future dates, enabling meaningful overlap detection.
- **All Florida counties** available in dropdown, with `Dade` normalized.
- **Past-date guard + UTC date handling** eliminates timezone bugs and invalid past bookings.

## What's next for ArchitectWizard

- **Modal flow:** Replace separate `conflict_check.html` page with an overlay modal inside `intake.html` for smoother flow.
- **Stronger sanitization:** Strict county regex, XSS sanitization, rate limits on endpoints.
- **Full Vertex migration:** Fully remove Gemini dependency if rate limits persist at scale.
- **Real-time polling:** Optional `setInterval` or Server-Sent Events for users who keep the map tab open without other tabs.
- **Cross-utility comparison:** Extend conflict logic to compare two explicit utility sources (e.g., DESC vs. Georgia Power analog) rather than single-county aggregation.


## Math / Metrics (optional)
Overlap score ranking: closer distance \( d \) = higher coordination value.

$$
\text{Priority} = \begin{cases} \text{Critical} & d < 1.6\ \text{km} \\ \text{High} & 1.6 \le d < 8\ \text{km} \\ \text{Medium} & d < 40\ \text{km} \\ \text{Ignore} & d \ge 40\ \text{km} \end{cases}
$$

We use temporal overlap as primary signal (`start \le q_{end} \land end \ge q_{start}`) and geographic proximity as secondary ranking.
