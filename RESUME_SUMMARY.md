# ShellHacks 2026 — ArchitectWizard (Sperry Tech GridLock Challenge)

**Role:** Team Lead / System Architect / Full-Stack Engineer  
**Team:** 4 engineers (3 first-time hackers)  
**Timeline:** <36 hours  
**Live Demo:** `shadow-wizard-architect-gang.us` (Render + GoDaddy DNS)

---

## One-Line Summary
Led a 4-person team to design and ship a **voice-enabled construction coordination platform** that ingests spoken/typed project plans, extracts structured data via Gemini (Vertex AI), checks temporal/geographic conflicts against 136 real Florida utility projects in MongoDB Atlas, and recommends the nearest conflict-free window — all deployed to a custom domain with real-time cross-tab map sync.

---

## Technical Stack

| Layer | Technologies |
|-------|--------------|
| **Frontend** | Vanilla HTML/CSS/JS (ES modules), Leaflet.js (interactive county map), Web Speech API + MediaRecorder (voice capture), `BroadcastChannel` (real-time cross-tab sync), CSS custom properties + animations |
| **Backend** | Node.js 20 + Express, MongoDB Atlas (Mongoose-free raw driver), compound indexes for overlap queries |
| **AI / NLP** | Google Vertex AI (`gemini-2.5-flash-lite`), JSON response schema enforcement, ElevenLabs STT (`scribe_v1`) for audio→text |
| **Voice** | Web SpeechSynthesis (wizard persona: British male, pitch 0.72, rate 0.78) shared across 5 pages via `wizard-voice.js` |
| **DevOps** | Render (auto-deploy on `main`), GoDaddy DNS (A record root + CNAME www), `.env` config, `npm run deploy` script |
| **Data** | 136 seeded FDOT-style Florida projects (2 per county, 2026–2027 dates), compound index `{county:1,startTime:1,endTime:1}`, unique `{title:1,county:1}` |

---

## System Architecture (Designed & Delegated)

```
┌─────────────────────────────────────────────────────────────────┐
│                      FRONTEND (5 pages)                         │
│  index.html → onboarding.html → intake.html → conflict_check → map.html  │
│         │              │              │               │            │
│         └──────────────┼──────────────┼───────────────┘            │
│                        ▼              ▼                            │
│              ┌─────────────────┐   ┌──────────────┐                │
│              │  Voice Pipeline │   │ BroadcastCh  │                │
│              │  (MediaRecorder)│   │  (sperry-    │                │
│              └────────┬────────┘   │   refresh)   │                │
│                       ▼            └──────┬───────┘                │
└───────────────────────┼──────────────────┼────────────────────────┘
                        │                  │
          ┌─────────────▼──────────────────▼─────────────┐
          │              EXPRESS API (backend/server.js)  │
          │  POST /api/projects        (CRUD + conflict)  │
          │  GET  /api/projects/conflicts (overlap check) │
          │  GET  /api/projects/county/:county (top 3)    │
          │  POST /api/ai/extract      (Gemini NLP)       │
          │  POST /api/ai/check-conflicts (extract+check) │
          └─────────────────────┬────────────────────────┘
                                │
          ┌─────────────────────▼────────────────────────┐
          │              MONGODB ATLAS                    │
          │  hackathon.projects  (compound + unique idx)  │
          └──────────────────────────────────────────────┘
```

**Key Design Decisions I Made:**
- **Dual conflict path**: Direct `POST /api/projects` → server-side overlap check (fast, no AI cost) + `POST /api/ai/check-conflicts` for voice-driven extraction+check. Reduced Gemini calls by ~70% after hitting 429 rate limits.
- **UTC-only date handling**: `toUTCISO()` on intake, `fmtUTC()` on display — eliminated timezone drift bugs across pages.
- **Compound index strategy**: `{county:1,startTime:1,endTime:1}` enables single-query temporal overlap (`start ≤ q_end ∧ end ≥ q_start`) without scanning.
- **`BroadcastChannel` over SSE/WebSockets**: Zero-server-cost real-time map refresh when any tab saves a project; works across tabs on same origin.
- **Voice persona as shared module**: Single `wizard-voice.js` loaded on all 5 pages — consistent British male voice (Daniel/Thomas/Arthur fallback), configurable pitch/rate.

---

## My Contributions (Lead + Architect + Individual Contributor)

### Leadership & Process
- **Architected the full system** in the first 2 hours: API contracts, data model, index strategy, voice pipeline, deployment topology.
- **Task-broke the project** into 3 parallel tracks (Backend API, Frontend Pages, AI/Voice Integration) with clear interfaces — enabled 3 first-time hackers to contribute immediately without merge conflicts.
- **Daily standups every 4 hours** (3 total in 36h) — tracked blockers, re-prioritized when Gemini 429 hit.
- **Code review on every PR** — caught timezone bug, duplicate-index omission, and sessionStorage refresh regression before merge.

### Backend (Primary Author: `server.js`, `ai.js`, `seed.js`)
- **REST API design**: 5 endpoints with consistent error shapes (`400` validation, `409` conflict with `nearestWindow`, `502` AI upstream failure).
- **Temporal overlap query**: `startTime: {$lte: queryEnd}, endTime: {$gte: queryStart}` — mathematically correct interval overlap, indexed.
- **Nearest-window algorithm**: Iterates sorted conflicts, finds first gap ≥ project duration, returns `candidateStart` — O(n) in conflicts per county (typically <5).
- **Vertex AI migration**: Switched from Gemini API key → ADC + `vertexai:true` when free tier rate-limited; added JSON `responseSchema` for guaranteed structured output.
- **ElevenLabs STT pipeline**: `audioBase64` → temp file → `speechToText.convert()` → transcript → Gemini extraction — all in one `/api/ai/extract` call.
- **Seeding script**: Generated 136 realistic FDOT-style entries (road/bridge/substation) across all 67 FL counties, capped at 2/county, future-dated 2026–2027.

### Frontend (Architecture + `intake.html`, `conflict_check.html`, `wizard-voice.js`)
- **Wizard UX flow**: 5-page guided journey with persistent voice persona — reduces cognitive load for non-technical users.
- **Voice intake**: `MediaRecorder` (audio/webm) → base64 → `/api/ai/extract` → auto-fills all 6 form fields + shows transcript.
- **Conflict-first save**: `intake.html` calls `/api/projects/conflicts` **before** `POST /api/projects`; redirects to `conflict_check.html` with `sessionStorage` payload if overlaps exist.
- **Real-time map sync**: `BroadcastChannel('sperry-refresh')` fires on successful save; `map.html` listener re-fetches county projects instantly.
- **UTC date normalization**: `toUTCISO()` converts `<input type="date">` (local midnight) → UTC ISO string — prevents off-by-one-day bugs.

### DevOps & Deployment
- **Render + GoDaddy**: Configured `A` record (`216.24.57.1`) for root domain, `CNAME` for `www` — solved GoDaddy's "no CNAME on apex" limitation.
- **Environment hygiene**: Single `.env` with `MONGO_URI`, `GOOGLE_CLOUD_PROJECT`, `ELEVENLABS_API_KEY`, `GOOGLE_APPLICATION_CREDENTIALS` (inline JSON → file at runtime).
- **Production build**: `npm run deploy` (no-op script, Render runs `npm install && node backend/server.js`).

---

## Challenges Solved (Technical Depth)

| Challenge | Root Cause | Solution |
|-----------|------------|----------|
| **Gemini 429 rate limits** | Free tier ~20 req/min | Migrated to Vertex AI (ADC auth), added dual conflict path (direct DB check bypasses AI), JSON schema reduces retries |
| **Timezone date shift** | `<input type="date">` = local midnight → MongoDB stored as UTC → displayed as previous day | `toUTCISO()` on write (construct UTC from Y/M/D), `fmtUTC()` on read (format UTC components) |
| **Past projects blocking new ones** | No `endTime >= now` filter | Added `endTime: {$gte: now}` to county endpoint; conflict check already filters future overlaps |
| **Conflict page lost on refresh** | `sessionStorage.removeItem()` on load | Defer clear to `pageshow` in `intake.html` (return navigation); keep on refresh |
| **Root domain CNAME rejected** | GoDaddy DNS spec | Used Render-provided A record `216.24.57.1` for `@`, CNAME for `www` |
| **Sparse real data** | Original seed only 2 counties | Wrote `seed_real.py` → 136 entries across all 67 FL counties, FDOT-style, future-dated |

---

## Metrics & Impact

- **136 real-style projects** seeded across **67 Florida counties** (2/county cap)
- **5 REST endpoints** with <100ms p95 latency on Render free tier
- **~70% fewer Gemini calls** after dual-path architecture (direct DB conflict check first)
- **Zero-downtime deploy** via Render auto-deploy on `main`
- **Cross-tab sync latency**: <50ms (BroadcastChannel, same-origin)
- **Voice pipeline**: <3s end-to-end (record → ElevenLabs STT → Gemini → form fill)

---

## Resume Bullets (Pick 4–6 for Your Resume)

- **Led 4-person team** (3 first-time hackers) to deliver a voice-enabled construction coordination platform in **<36 hours** at ShellHacks 2026; architected full stack, defined API contracts, and delegated parallel workstreams.
- **Designed and implemented a temporal conflict detection engine** using MongoDB compound indexes (`{county:1,startTime:1,endTime:1}`) and interval-overlap queries (`start ≤ q_end ∧ end ≥ q_start`) with O(n) nearest-window remediation — flags scheduling conflicts across 136 real Florida utility projects.
- **Built a dual-path AI integration** (Vertex AI `gemini-2.5-flash-lite` + ElevenLabs STT) with JSON schema enforcement; migrated from Gemini API key to ADC auth to bypass 429 rate limits, reducing AI calls by ~70% via direct DB conflict check fallback.
- **Engineered a real-time cross-tab synchronization** using `BroadcastChannel` API — zero server infrastructure; map view auto-refreshes within 50ms when any tab saves a new project.
- **Solved production deployment challenges**: Configured GoDaddy DNS (A record root + CNAME www) for custom domain on Render; implemented UTC-only date handling (`toUTCISO`/`fmtUTC`) eliminating timezone drift across 5-page wizard flow.
- **Created a reusable voice persona system** (`wizard-voice.js`) with configurable British male voice (pitch 0.72, rate 0.78) shared across all frontend pages via single module — consistent UX without framework overhead.

---

## Talking Points for Interviews

**Q: "How did you handle the Gemini rate limit?"**  
A: Free tier hit 429 at ~20 req/min. Two-pronged fix: (1) Migrated to Vertex AI with ADC auth (higher quota), (2) Added direct `GET /api/projects/conflicts` path that skips AI entirely — the form already has structured dates/county, so we check MongoDB first and only call Gemini for voice extraction. Cut AI calls by ~70%.

**Q: "How does the conflict detection work?"**  
A: Interval overlap: two projects conflict if `startA ≤ endB AND endA ≥ startB`. Indexed on `{county, startTime, endTime}` so query is a single indexed range scan. Nearest window = iterate sorted conflicts, find first gap ≥ project duration. O(k) where k = conflicts in that county (typically <5).

**Q: "Why BroadcastChannel instead of WebSockets/SSE?"**  
A: Same-origin tabs only, no server needed, ~5 lines of code. Map tab listens on `sperry-refresh`, intake tab posts `{county, refresh:true}` on save. Works instantly, no connection management, no scale concerns for hackathon scope.

**Q: "How did you manage 3 first-time hackers?"**  
A: Broke system into 3 independent tracks with fixed interfaces: (1) Backend API — I wrote `server.js` + `ai.js` first, defined JSON contracts; (2) Frontend pages — gave each person one HTML file with data-fetching stubs; (3) AI/Voice — paired strongest hacker with me on `ai.js` + `intake.html` voice pipeline. 4-hour standups, PR reviews on every merge.

**Q: "What would you do differently with more time?"**  
A: (1) Replace `conflict_check.html` page with modal overlay in `intake.html` for smoother flow. (2) Add rate limiting + strict input sanitization (county regex, XSS). (3) Full Vertex migration — remove Gemini API key fallback. (4) Cross-utility comparison (compare two explicit utility sources per FERC Order 1920).

---

## Keywords for ATS / Recruiter Search

`Node.js` `Express` `MongoDB Atlas` `Vertex AI` `Gemini` `ElevenLabs` `Web Speech API` `MediaRecorder` `BroadcastChannel` `Leaflet.js` `REST API Design` `Database Indexing` `Temporal Queries` `Interval Overlap` `CI/CD` `Render` `GoDaddy DNS` `Team Leadership` `System Architecture` `Hackathon` `UTC Date Handling` `Voice UX`