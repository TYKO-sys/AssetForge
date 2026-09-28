# PRD — AssetForge v1: Local Content Repurposing Retainer Engine

**Status:** APPROVED 2026-09-28 (Phase 1 checkpoint passed; Amendment 1 below locks open questions)
**Users:** (1) Solo operator (founder, on the M5/24GB) — runs the factory. (2) Retainer clients — receive asset packs.
**Stack:** Python 3.12 · mlx-whisper · Ollama (llama3.1:8b, llama3.2:3b, nomic-embed-text) · Kokoro-82M TTS · UVR · yt-dlp · FFmpeg · n8n self-host · Postiz self-host · Tailscale · Resend free · GitHub Free.
**Cost constraint:** $0 fixed. Stripe enters only at invoicing time (% of revenue).

## Problem statement
Long-form audio/video (podcasts, streams, interviews) is the most expensive content to produce and the least repurposed. Operators with 2+ hours of source media need ~30 derivative assets (show notes, blog, threads, captions, clips, audio versions) but manual repurposing costs 6–10 human hours per episode or $500+/mo in SaaS subscriptions. AssetForge converts one source file into a structured, client-ready asset pack on a single MacBook with zero marginal cost and zero manual shuttling.

## Success metrics (v1)
- 2h episode → full asset pack in **≤ 30 min unattended** on the M5
- Marginal cost per episode: **$0** (electricity excluded)
- Human touch per episode: **≤ 15 min** (review + approve)
- Pipeline reliability: **≥ 95% jobs complete without manual rescue** over the first 20 jobs
- Business: **1 paying retainer within 6 weeks** of v1 acceptance

## User stories + acceptance criteria
1. **As the operator, I drop a source (file or URL) and nothing else.**
   AC: file placed in `~/assetforge/intake/inbox/` (locally, or from iPhone/Watch via Shortcut-over-Tailscale) triggers the full pipeline with no further input; URL files (.url) fetch via yt-dlp first.
2. **As the operator, I get one notification when the pack is ready.**
   AC: Resend email arrives with pack path + manifest summary; failed jobs notify with the failing stage + job_id.
3. **As the operator, I can resume a failed job without redoing completed stages.**
   AC: each stage writes state to `work/<job_id>/`; re-run picks up at the failed stage (idempotent, content-hash keyed).
4. **As a client, I receive a structured pack I can navigate without instructions.**
   AC: `manifest.json` lists every asset with type, path, platform target, and word/clip length; folder names are human-readable; no internal artifacts leak into `exports/`.
5. **As the operator, I approve-before-post for platforms with API friction.**
   AC: Postiz queue is filled by the pipeline in DRAFT status; one manual approve step publishes; auto-publish only where the platform's API is frictionless.
6. **As the operator, every generated asset carries provenance.**
   AC: each asset embeds source reference + timestamp range it was derived from (the academic-engine discipline, reused).

## Scope — v1 SHIPS
- Ingest: local drop + SSH-over-Tailscale drop + URL fetch (yt-dlp)
- Audio prep: FFmpeg loudness-normalize; UVR pass when music bed detected (heuristic v1: operator flag in filename, e.g. `music_`)
- Transcription: mlx-whisper (large-v3-turbo), segment timestamps
- Generation (llama3.1:8b): show notes, blog draft, X thread (5–8 posts), LinkedIn post, IG caption, 8 quotes, SEO title+description, 5 clip recommendations with timecodes
- Audio assets: Kokoro TTS narration of show notes (v1.1; ships one week after core)
- Packaging: folder contract + manifest.json + provenance
- Delivery: email notification + pack folder; Postiz draft queue
- Academic bridge: same pipeline ingests WGU lecture audio → study notes (the double-duty proof)

## Scope — v1 does NOT
- Video clip rendering (WhisperX word-alignment + FFmpeg cutting = v1.2)
- Voice cloning (GPT-SoVITS) — ships only with a documented client-consent flow
- Web client portal (Track 3 productization, post-revenue)
- Multi-tenant client isolation beyond folder separation
- Image asset generation (ComfyUI stack — Stage 5)
- Auto-publishing to all platforms (approve-first everywhere in v1)

## Data model
Folder contract per the blueprint (§3). `manifest.json` schema:
```json
{
  "job_id": "sha256[:16]",
  "client": "slug", "source_ref": "url|file", "duration_s": 7123,
  "stages": {"fetch":"ok","clean":"skip","transcribe":"ok","generate":"ok","tts":"ok","package":"ok","schedule":"ok"},
  "assets": [{"type":"thread","path":"thread.txt","platform":"x","tokens":412,"provenance":"[00:12:30–00:14:05]"}],
  "model_manifest": {"stt":"whisper-large-v3-turbo-mlx","llm":"llama3.1:8b-instruct-q4_K_M","tts":"kokoro-82m"}
}
```
Job ledger: SQLite `logs/jobs.db` (job_id, stage states, timings, token counts) — resumability + the eval dataset.

## Edge cases + failure states
- **3h+ episodes:** chunked transcription (10-min segments), stitched; generation uses map-reduce summarization
- **Multi-speaker:** v1 labels speakers heuristically only if diarization confidence is high; else single-voice transcript with timestamps
- **Music bed present:** UVR stage; if stems still dirty → job fails at `clean` with actionable message, operator reflags
- **Ollama model not resident (swapped out):** pipeline pre-flight checks `ollama ps`, cold-loads, logs the added latency
- **n8n/Docker down:** launchd + `docker compose restart: unless-stopped`; intake folder buffers (jobs process on recovery — the folder contract makes the bus durable)
- **Disk pressure:** intake guard refuses jobs when free space < 20GB; logs cleanup policy (work/ dirs purged after 14 days)
- **Watch/phone offline drop:** Shortcut queues locally and syncs on reconnect (Shortcuts "Wait for network" pattern)
- **Social token expiry:** Postiz draft-only in v1 — expired tokens surface in the notification, never block packaging
- **Hallucination guard:** every generated asset must cite provenance ranges; generation prompt rejects "insufficient source" with explicit flag in manifest

## Open questions (need your answers to lock the spec)
1. First niche/client profile — podcasters, coaches, real-estate, local services? (Determines asset mix + prompt library.)
2. Do you already have a client pipeline (warm leads), or does AssetForge v1 need a demo-first strategy (produce one free spec pack for a target client)?
3. Which platforms actually matter for client #1 (X, LinkedIn, IG, TikTok, YouTube Shorts)? Postiz drafts all; we prioritize prompts for 3.
4. Voice cloning consent — skip entirely in v1? (My recommendation: yes, skip.)
5. Academic intake: do you want WGU lecture audio auto-converted into your RAG engine on the same bus (yes = one more intake route)?

---

## AMENDMENT 1 — 2026-09-28 (user pre-approved catalog integration; Phase 1 open questions resolved)

**Open question resolutions:**
1. **Niche = real estate agents/brokers (beachhead).** Rationale: they already produce long video (listing walkthroughs, market updates) that dies on one platform; they already pay for lead-gen ($200–1000/mo to Zillow-class platforms); they are local, publicly reachable, and chronically time-poor. Secondary vertical (post-validation): podcasters.
2. **Go-to-market = demo-first** (no existing pipeline): produce unsolicited sample packs from prospects' public videos — the pack IS the pitch. At $0 marginal cost, demo volume is unlimited.
3. **Platforms (priority order): Instagram → Facebook → YouTube Shorts → LinkedIn.** Postiz drafts all four; the prompt library is tuned for IG/FB first.
4. **Voice cloning: skipped for v1.** Confirmed.
5. **Academic intake: YES.** WGU lecture audio enters the same intake bus → transcript → study-notes asset branch + RAG ingestion route. Scope added to v1.

**Catalog integration (199 verified repos from user's curated_catalog.md — source: https://github.com/user-attachments/files/32708278/curated_catalog.md):**

| Addition | Stage | Role |
|---|---|---|
| WyattBlue/auto-editor | v1.2 | Auto-cut silence/filler before clip extraction — clip prep cost halves |
| harry0703/MoneyPrinterTurbo | v2 | Text→short-video generation; video asset tier |
| firecrawl/firecrawl | Stage 4 | Prospect sites/socials → clean LLM-ready data for personalized outreach |
| dgtlmoon/changedetection.io | Stage 4 | Client competitor/website monitoring — sellable AAA line item |
| trycompai/crm | Stage 4 | Open-source CRM for retainer client management |
| calcom/cal.diy | Stage 4 | Self-hosted booking page — calls get scheduled without human shuttling |
| browser-use/browser-use | Stage 4/5 | Browser automation for onboarding + posting where platform APIs friction |
| crewAIInc/crewAI | Stage 4 | Multi-agent orchestration (Track 2 service builds) |
| stanfordnlp/dspy | Engine | Prompt optimization + eval; WGU double-duty |
| pipecat-ai/pipecat | Stage 5 | Voice agents (Track 2 phone intake) |
| wasp-lang/open-saas + shadcn/ui | Stage 5 | Track 3 SaaS scaffold |
| plausible + Openpanel | Stage 5 | Layer-5 analytics |
| bitwarden/clients | Stage 0 | Secrets management for the API keys the business accumulates |
| anthropics/courses, claude-cookbooks, prompt-eng tutorial | Academic | Degree study material |
| OSINT set (sherlock, maigret, holehe, theHarvester, osintgram) | Stage 4 | Lead research (find decision-maker handles); ToS-care noted |
| Security set (60 repos: hacktricks, nuclei, OWASP suite, etc.) | Academic | WGU security competency practice labs — NOT business infrastructure |
| gophish/gophish | Optional | Opted-in email lists only; cold outreach stays DM-first (deliverability + ToS) |

**Stack validations from catalog:** n8n, Postiz, Ollama, Open WebUI, Unsloth, Jan, KoboldCPP, ComfyUI, Fooocus, whisper, shadcn/ui — all already present in user's saved posts; adopted picks confirmed independently.

**Rejected from catalog (with reasons):** pentest/attack tooling (metasploit, sqlmap, etc.) — legal exposure for a content business, academic labs only; anonymous/offense OSINT (torbot, deauther) — no business function; vibe-trading/quant repos (Vibe-Trading, FinceptTerminal) — out of scope; `app.example.com`-class OCR noise.

---

## AMENDMENT 2 — 2026-09-28 (Phase 3 architecture lock, pre-authorized "go")

The original spec said "n8n orchestration" for the pipeline itself. Implementation reality:
Docker cannot reach Apple Metal, and mlx-whisper + Ollama must run on the host GPU. The spec
is amended before the code shipped (per the spec-first rule):

1. **Split-brain orchestration.** A host daemon (`af run --watch`, managed by launchd) owns
   every Metal stage: fetch → normalize → clean → transcribe → segment → generate → package.
   n8n (Docker) owns post-pack fan-out only (email/Postiz/client comms), triggered by an
   optional localhost webhook. If n8n is down, packs still complete and notify.
2. **Postiz staging in v1.0.** Distribution = `postiz-outbox.json` + approve in Postiz UI.
   Direct Postiz API integration is a Stage 2 config flip (disabled node ships in
   `workflows/assetforge-v1.json`), not new code.
3. **Provenance gate is a ship gate.** An asset whose quotes/timecodes fail validation
   against the transcript ships flagged `provenance_ok: false` — it is never silently passed.
4. **Academic bridge confirmed (Amendment 1 §5).** `af study <audio>` runs the same bus and
   emits `study-pack.md` + optional RAG ingestion into the `wgu_knowledge` collection.
5. **Filename contract v1:** `{client}__{source}__{YYYY-MM-DD}__{slug}[_music__].{ext}` —
   4–5 double-underscore tokens; `music` token triggers the UVR clean stage; `.url` files
   carry one fetch URL per line. See `src/contract.py` (authoritative parser).
