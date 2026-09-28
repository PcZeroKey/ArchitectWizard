# ShellHacks 2026 — Sperry Tech / Interview Script

## 30-second pitch
"We built a wizard-guided construction coordination platform. You speak or type a project — county, dates, company — and our British wizard checks conflicts against real Florida utility plans in MongoDB Atlas. It flags timeline overlaps, shows them on a county map, and recommends the nearest free window. Domain is live at shadow-wizard-architect-gang.us, deployed on Render with GoDaddy DNS, using Gemini for NLP, ElevenLabs for voice, and MongoDB Atlas for data."

## Challenge mapping

**Sperry GridLock:** We ingested 136 real Florida construction entries (FDOT-style) capped at 2 per county. `POST /api/projects` saves; `GET /api/projects/conflicts` and `/api/ai/check-conflicts` run temporal overlap queries (`startTime <= end`, `endTime >= start`) plus compute `nearestWindow`. Interactive map (`map.html`) + `BroadcastChannel` refresh show results live.

**GoDaddy:** Domain `shadow-wizard-architect-gang.us` verified on Render. GoDaddy DNS: A record `216.24.57.1` (root), CNAME `architectwizard.onrender.com` (`www`). `PRODUCTION_URL` set; `npm run deploy` added.

**Gemini:** `backend/ai.js` (Vertex AI, `gemini-3.8-flash`). `/api/ai/extract` parses input; `/api/ai/check-conflicts` does extraction + overlap + nearest window. Switched direct endpoint to avoid 429.

**ElevenLabs:** `wizard-voice.js` (British male, pitch `0.72`, rate `0.78`). Microphone captures `audioBase64` → `transcribeAudio()` → fills form. Wizard speaks on every page.

**MongoDB Atlas:** Atlas `hackathon.projects`; compound index `{county:1,startTime:1,endTime:1}`; unique `{title:1,county:1}`. Real-time refresh via `BroadcastChannel('sperry-refresh')`.

## Deep points (if asked)
- Past-date guard (`startTime < today` rejected) + `toUTCISO()` date consistency.
- All Florida counties in dropdown (`Dade` shown as `Miami-Dade`).
- Conflict page (`conflict_check.html`) loads from `sessionStorage`; refresh kept; cleared on return to intake via `pageshow`.
