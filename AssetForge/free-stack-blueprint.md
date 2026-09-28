# FREE STACK BLUEPRINT — AssetForge + WGU Engine
**Source evidence:** FMHY /ai pulled live 2026-09-28 (539 resources, 15 categories) + FMHY /other/selfhosting + 3 independent free-tier verifications.
**Constraint:** $0 fixed cost. Every tool below is free to run. Only Stripe takes a % AFTER revenue exists.

---

## 1. FMHY /ai — WHAT MADE THE CUT (and what was rejected)

### ADOPTED — Core Factory (Track 1: Content Repurposing)

| Tool | FMHY section | Role | Why it won |
|---|---|---|---|
| Ollama | Self-Hosting Tools | Model runtime (already core) | Canonical Mac local-LLM runtime, first-class Apple Silicon |
| mlx-whisper (MLX community builds) | Local AI Frontends ecosystem | STT workhorse | Many-x realtime on M-series; already in msg-1 architecture |
| Kokoro-82M | Text to Speech | TTS output layer | 82M params, Apache-2.0, runs realtime locally; closes the TTS blank in the msg-1 stack |
| GPT-SoVITS | Text to Speech | Voice cloning (client-branded audio, WITH consent) | Free local zero-shot cloning; replaces ElevenLabs paid |
| Ultimate Vocal Remover (UVR) | Voice Removal/Separation | Clean raw client audio (music beds, noise) | Local, free, best-in-class stem separation |
| ebook2audiobook | Text to Speech | Long-form text→audiobook pipeline | Double duty: client repurposing + textbook audio for study |
| yt-dlp + FFmpeg | (FMHY Downloading ecosystem) | Universal ingest of client source media | Pulls any platform's media for the pipeline |
| Open WebUI | Local AI Frontends | Instant ChatGPT-style UI over Ollama | Free demo surface for client calls; has built-in RAG |
| Postiz | (verified via search) | Social scheduling, 30+ networks | Self-hosted free; closes the distribution blank |
| n8n self-hosted | (verified via search) | Orchestration spine | Unlimited workflows, free for internal business use (Sustainable Use License) |

### ADOPTED — Academic Engine accelerators

| Tool | FMHY section | Role |
|---|---|---|
| NotebookLM | Specialized Chatbots | Source-grounded study guides + audio overviews (non-sensitive material only — Google cloud) |
| ankinlm extension | Specialized Chatbots | NotebookLM → Anki export bridge |
| genanki (Python lib) | — | Programmatic Anki deck generation from our RAG engine |
| Google Illuminate | Specialized Chatbots | Papers → podcast-style discussions |
| Artificial Analysis / Aider leaderboards / models.dev | AI Benchmarks | Living model-selection dashboards before every model pull |
| WhatModelsCanIRun / canirun.ai | Self-Hosting Tools | Hardware-fit checks |
| Unsloth | Local AI Frontends | Free QLoRA fine-tuning (Track 3 vertical models) |

### REJECTED (for now) — with reasons
- **Suno / music gen** — not in the retainer value chain yet
- **ComfyUI / Fooocus image stack** — image asset generation is Stage 5+, after first revenue
- **GPT4Free / API-proxy greyware** — ToS and reliability risk; a client-facing business cannot run on it
- **Roleplay frontends (SillyTavern etc.)** — no business function
- **Arena / multi-model roulette sites** — fine for humans, wrong for pipelines; we need deterministic local inference
- **Perplexity/Exa** — research toys vs. our own RAG; we keep the engine in-house

---

## 2. THE $0 TRANSLATION OF THE FIVE-LAYER PRODUCTION MODEL

| Layer | Standard stack | $0 replacement | Limit that matters |
|---|---|---|---|
| 1. Builder | Claude Code + GitHub | GitHub Free (private repos) | — |
| 2. Spine | Vercel + Supabase + Stripe + Cloudflare | Cloudflare Pages/Workers (unlimited static bandwidth) OR Vercel Hobby; Supabase Free (500MB DB, 50k MAU); Stripe (% of revenue, $0 fixed); Cloudflare DNS free | Serverless limits only matter post-revenue |
| 3. Keeping it alive | Sentry + Better Stack + PostHog + Upstash | GlitchTip self-host OR Sentry free (5k err/mo); Uptime Kuma self-host; PostHog free (1M events/mo); Upstash free Redis (rate limiting) | All self-hosted on the M5 or free tiers |
| 4. Talking to users | Resend + Intercom + n8n/Zapier | Resend free (3k email/mo) OR Brevo free (300/day); Chatwoot self-host (support inbox); **n8n self-hosted replaces Zapier entirely** | n8n: free for internal business use; do NOT resell n8n itself as your product (license boundary) |
| 5. Discovery | WordPress + Beehiiv | Astro/Next blog on Cloudflare Pages + IndexNow (free); Beehiiv free tier (2.5k subs) | Per your rule: Layers 3–5 wait until revenue |

**Uncommon gems (the shortcuts list):**
1. Tally — unlimited free intake forms, no branding (replaces Typeform $)
2. Uptime Kuma — self-hosted status pages; instant agency credibility artifact
3. Kokoro-82M — TTS that sounds like a cloud model, 82M params, realtime local
4. WhisperX / stable-ts — word-level timestamp alignment for surgical clip-cutting (Stage 2 upgrade over vanilla mlx-whisper)
5. genanki — RAG output → .apkg Anki decks, zero manual steps
6. Pinokio — one-click installer for local AI apps (shortcut when Docker gets tedious)
7. llm.datasette.io — Simon Willison's CLI; logs every prompt to SQLite (eval evidence)
8. Aider — free terminal AI pair-programmer (build velocity)
9. Tailscale free — 3 users / 100 devices; the private mesh that makes the Mac reachable from iPhone/Watch anywhere
10. Cloudflare Tunnel free — public HTTPS into the M5 without exposing a home IP (client demo links)
11. NocoDB — free Airtable-like asset registry for client deliverables
12. Chatwoot — free self-hosted support inbox

---

## 3. INTEGRATION ARCHITECTURE — "no back-and-forth" contract

**Design principle:** humans never shuttle data. Everything moves through (a) watched folders, (b) webhooks, (c) scheduled triggers. n8n is the spine; the filesystem is the data bus; Tailscale is the network.

```
iPhone/Watch Shortcut  --SSH over Tailscale-->  ~/intake/inbox/   (or n8n webhook)
        |                                                      |
        v                                                      v
  n8n file-trigger → [classify: client/slug via Llama 3.2:3b]
        → yt-dlp fetch (if URL) → FFmpeg normalize → UVR clean (if music bed)
        → mlx-whisper transcript → Llama 3.1:8b asset generation
        → Kokoro TTS (audio assets) → package to exports/<client>/
        → Postiz schedule → Resend notify → log to SQLite (eval)
```

**Folder contract (the data bus):**
```
~/assetforge/
├── intake/inbox/          # raw drops (any of: .url, .mp4, .mp3, .m4a, .wav)
├── work/<job_id>/         # intermediate artifacts, one dir per job
├── exports/<client>/YYYY-MM-DD-<slug>/
│   ├── manifest.json      # the deliverable contract (see PRD)
│   ├── show-notes.md  blog-draft.md  thread.txt  linkedin.md
│   ├── captions/  quotes/  audio/  clips/  seo/
└── logs/jobs.db           # SQLite job ledger (resumability + eval)
```

**M5 memory budget (24GB) with the factory resident:**
- macOS + windowserver: ~4.0GB
- Ollama (Llama 3.1 8B + nomic embed): ~7.7GB
- n8n (Docker): ~1.5GB
- Postiz (Docker + Postgres + Redis): ~2.5GB — Stage 2 addition
- Python pipeline (whisper burst): ~4GB transient during jobs
- Headroom: ~4GB — UVR runs as a batch job, never concurrent with generation

Iron rule v2: ONE generation model + ONE embedder resident; transcription and TTS are burst jobs; n8n serializes jobs via workflow concurrency=1.

---

## 4. VERIFICATION LEDGER (evidence, dated 2026-09-28)

| Claim | Status | Source |
|---|---|---|
| n8n self-host free for internal business use | ✅ VERIFIED | Sustainable Use License — free unlimited internal workflows; reselling/embedding n8n as your product requires a paid embed license |
| Postiz free self-hosted, 30+ networks | ✅ VERIFIED | GitHub gitroomhq/postiz-app, open-source |
| xAI/Grok free API tier for prototyping | ✅ exists | Rate-limited; treat as burst-only |
| Groq / OpenRouter :free endpoints | ⚠️ limits rotate | Verify current caps at signup; local-first means no dependency |
| Social auto-post friction (Meta/X/LinkedIn) | ⚠️ FLAGGED | Postiz handles OAuth for connected accounts; X free write-tier is limited; expect human-approve step in v1 for some platforms |
| Tailscale personal free (3 users/100 devices) | ✅ stable for years | tailscale.com pricing |
| Stripe $0 fixed | ✅ | % per transaction only — costs nothing until money arrives |
