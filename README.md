# TalkThatTalk

Talk to anyone, in your own language. No sign-up. Nothing saved.

TalkThatTalk is a zero-login, ephemeral chat for travellers. A traveller opens the app, picks
their language, and gets a QR code. A local person scans it with their phone's camera — no app
install — and lands straight in the chat in their own language. Each person types and reads in
their own language, live, in the same conversation. Close the chat (or walk away for 5 minutes)
and it's gone for good — nothing is ever written to a database.

Language pairs aren't limited to "X to English" — a Korean traveller talking to a Chinese-speaking
local, or a Japanese traveller talking to a Sinhala-speaking local, works the same way as any
English pair. See [Translation engine](#translation-engine---tiered-no-card-required-by-default)
below for how.

## Who uses what

- **Traveller → Flutter app** (Android/iOS). Installed once, reused in every country. Starts a
  chat and shows the QR code.
- **Local person → Web app** (browser, no install). Scanning the QR with the phone's stock camera
  opens a link straight into the chat.

This split is deliberate: the traveller is a repeat user, so a real app is worth it. The local
person is a one-time user who must never be asked to install anything.

## How it works

**Traveller (Flutter app):** pick language → tap *Start Chat* → app shows a QR code → hand the
phone to the other person → chat, each in your own language.

**Local person (Web app):** scan the QR → pick language → tap *Join* → chat.

**Either side:** tap *End Chat*, or leave it idle for 5 minutes — the conversation is never
stored, so there's nothing left to find afterward.

## Architecture — C4 model

### Level 1: System Context

```mermaid
C4Context
  title System Context diagram for TalkThatTalk

  Person(traveler, "Traveller", "Speaks Language A, carries a smartphone")
  Person(local, "Local Person", "Speaks Language B, has a smartphone with a camera")

  System(ttt, "TalkThatTalk", "Lets two people chat, each in their own language, with no login and nothing stored")

  System_Ext(translation, "Translation tiers", "Own memory cache, MyMemory, self-hosted NLLB-200, Gemini - see Translation Engine below")

  Rel(traveler, ttt, "Starts a chat, shows the QR code, chats", "Flutter app")
  Rel(local, ttt, "Scans the QR, joins, chats", "Web browser")
  Rel(ttt, translation, "Requests a translation per message", "HTTPS/JSON")
```

### Level 2: Containers

```mermaid
C4Container
  title Container diagram for TalkThatTalk

  Person(traveler, "Traveller")
  Person(local, "Local Person")

  System_Boundary(ttt, "TalkThatTalk") {
    Container(flutter_app, "Traveller App", "Flutter (Android/iOS)", "Starts the session, generates and displays the QR code, sends/receives messages")
    Container(web_app, "Local Web App", "React + Vite, on Cloudflare Pages", "Opened via the QR link; picks language, joins, sends/receives messages")
    Container(edge_fn, "Translation Router", "Supabase Edge Function (Deno)", "Checks the memory table, then routes through MyMemory, NLLB-200, or Gemini based on confidence and quota")
    ContainerDb(db, "Sessions Table", "Supabase Postgres", "One row per chat: session id + the two language codes. Self-expires after 5 min idle, or on explicit End Chat")
    ContainerDb(memory_db, "Translation Memory Table", "Supabase Postgres", "Cached (source text, source lang, target lang) to translation pairs, seeded + grown over time")
    Container(realtime, "Realtime Broadcast", "Supabase Realtime", "Relays chat messages peer-to-peer over WebSockets; never persists them")
    Container(nllb, "Self-hosted Translation Server", "NLLB-200 (600M) via CTranslate2, on Oracle Cloud free-tier VM", "Any-to-any across 200 languages, no English pivot required")
  }

  System_Ext(mymemory, "MyMemory API", "Free, no key, translation-memory + MT fallback")
  System_Ext(gemini, "Gemini API (free tier)", "Last-resort tier; general LLM, any language pair")

  Rel(traveler, flutter_app, "Uses")
  Rel(local, web_app, "Uses")

  Rel(flutter_app, db, "Creates the session row", "Supabase client SDK")
  Rel(web_app, db, "Reads the session, joins with its language", "Supabase client SDK")

  Rel(flutter_app, realtime, "Publishes / subscribes", "WebSocket broadcast channel")
  Rel(web_app, realtime, "Publishes / subscribes", "WebSocket broadcast channel")

  Rel(flutter_app, edge_fn, "Sends outgoing message to translate", "HTTPS")
  Rel(web_app, edge_fn, "Sends outgoing message to translate", "HTTPS")
  Rel(edge_fn, memory_db, "Check cache first, write result back", "Supabase client SDK")
  Rel(edge_fn, mymemory, "Cache miss -> request translation + match score", "HTTPS")
  Rel(edge_fn, nllb, "Low match score -> fallback translation", "HTTPS")
  Rel(edge_fn, gemini, "NLLB unavailable or quota hit -> last resort", "HTTPS")
```

### Level 3: Translation request flow

```mermaid
C4Component
  title Translation request flow

  Container_Boundary(edge_fn, "Translation Router (Edge Function)") {
    Component(cache_check, "1. Check memory table", "Supabase query", "Exact/near match on (text, source lang, target lang)? Return instantly, zero cost. Hit rate climbs over time as common phrases accumulate.")
    Component(mymemory_call, "2. Call MyMemory", "HTTPS, no key", "Cache miss -> ask MyMemory. Read its match confidence score.")
    Component(route_decision, "3. Route on confidence", "Logic", "High match -> trust it, cache it. Low match -> don't use the weak MT guess, go to step 4.")
    Component(nllb_call, "4. Call self-hosted NLLB-200", "HTTPS", "Direct any-to-any translation (e.g. Korean to Chinese, no English pivot). No card, no external rate limit.")
    Component(gemini_call, "5. Call Gemini (free tier)", "HTTPS", "Only if NLLB-200 is unreachable or overloaded. Last resort - see privacy note below.")
    Component(write_back, "6. Write result to memory table", "Supabase insert", "Whichever engine answered, cache it - the next identical phrase, from any user, is a free hit.")
  }

  Rel(cache_check, mymemory_call, "miss")
  Rel(mymemory_call, route_decision, "match score")
  Rel(route_decision, nllb_call, "low confidence")
  Rel(route_decision, write_back, "high confidence")
  Rel(nllb_call, write_back, "success")
  Rel(nllb_call, gemini_call, "unreachable / error")
  Rel(gemini_call, write_back, "")
```

## Translation engine — tiered, no card required by default

Free translation APIs (Google, Azure) require a billing card on file even for their free tier.
Genuinely free, no-card options exist, but the obvious self-hosted pick, **LibreTranslate**,
doesn't support Sinhala at all — so it can't be the fallback for this app's main language pair.
The tiering below works around that, and handles direct non-English pairs (Korean↔Chinese,
Japanese↔Sinhala, etc.) the same way it handles English pairs.

| Tier | Engine | Card? | Any-to-any pairs? | Cost | Role |
| --- | --- | --- | --- | --- | --- |
| 1 | Your own memory table (Supabase) | No | Yes (whatever's cached) | Free, instant | Catches repeat phrases across all users |
| 2 | MyMemory | No | Yes, but weaker on non-English pairs | Free, 5K–50K chars/day | First external call, confidence-scored |
| 3 | Self-hosted NLLB-200 | No (model); host may ask | Yes, direct — 200 languages, no English pivot | Free, bounded by your VM | Main fallback for novel/low-confidence text |
| 4 | Gemini API (free tier) | No | Yes, strong on major languages | Free, ~1,000 req/day (Flash-Lite) | Last resort only — see caveats below |

**Why NLLB-200 instead of LibreTranslate:** LibreTranslate (Argos Translate) supports ~30
languages and Sinhala isn't one of them. NLLB-200 explicitly supports Sinhala (`sin_Sinh`) and
200 languages total, and — unlike older bilingual-model systems — translates directly between
any two of its languages without routing through English as an intermediate step.

⚠️ **License caveat (NLLB-200):** the official weights are **CC-BY-NC-4.0 — non-commercial use
only**, and Meta's model card says it isn't released for production deployment. Fine for a
student/personal-project build; revisit before any commercial launch — either license a paid
engine at that point, or properly vet a community MIT-relicensed checkpoint (e.g. Open-NLLB)
closer to launch. Giving the app away for free to end users does **not** by itself make this
non-commercial use under the license — "non-commercial" turns on intent (ads, resume/business
value, any future monetization path), not on whether users pay.

**Hosting:** running on the existing Oracle Cloud Always Free VM (ARM Ampere, multiple cores /
plenty of RAM) — comfortably fits the NLLB-200 distilled 600M model, which Render's free tier
(512 MB RAM) could not.

⚠️ **Privacy caveat (Gemini tier):** Gemini's free tier terms permit prompt data to be used for
model training/improvement. Since this app's core promise is "nothing is ever stored," routing a
message through the free Gemini API means that message content can leave the system in a way
that quietly conflicts with that promise, even though nothing is retained on the Supabase side.
Treat Gemini as a rare, clearly-scoped last resort (tier 4, only when NLLB-200 is unreachable),
not a default path — or budget for Gemini's paid tier later specifically for its no-training-use
guarantee once this matters more.

**Rate limits (Gemini free tier, subject to change):** Gemini 2.5 Flash-Lite gives the best free
volume — roughly 30 requests/minute, ~1,000 requests/day. No card required to start.

## Full stack

| Layer | Tech | Role |
| --- | --- | --- |
| Traveller client | Flutter (Android/iOS) | Installed app. Creates the session, renders the QR code, chat UI for the traveller. |
| Local client | React + Vite, on Cloudflare Pages | No install. Chat UI for the local person, reached via the QR link. |
| Realtime messaging | Supabase Realtime (Broadcast channels) | Messages travel peer-to-peer; nothing persisted. |
| Session pairing | Supabase Postgres | One `sessions` row per chat: session id + the two language codes. Deleted on explicit End Chat, or after 5 min idle via a scheduled cleanup job. |
| Translation memory | Supabase Postgres | `translations` table: cached (text, source lang, target lang) → translation. Seeded with ~100–200 common travel phrases, grows from every live translation after. |
| Translation routing | Supabase Edge Function (Deno) | Checks the memory table, calls MyMemory, reads its confidence score, falls back through NLLB-200 then Gemini in order. |
| Free translation fallback | NLLB-200 (600M, distilled), self-hosted via CTranslate2 | Runs on the Oracle Cloud free-tier VM. No card, no external cap, any-to-any across 200 languages. |
| Last-resort translation | Gemini API (free tier) | Used only if NLLB-200 is unreachable or overloaded. See privacy caveat above. |
| QR codes | `qr_flutter` (Flutter package) | Encodes the session join-URL. Scanned by the phone's own camera app — nothing to build on the web side. |

## Why no messages are stored

Messages travel peer-to-peer through a Supabase Realtime broadcast channel — the sessions
database only ever holds which two languages are paired, not what was said, and that row deletes
itself within 5 minutes either way (or immediately on End Chat). The translation memory table
stores translated *phrases* for reuse across users, not a record of who said what to whom or
when. The one caveat to this promise is the Gemini fallback tier — see above.

## Local development

### Local Web App

```
cd web
npm install
cp .env.example .env   # fill in your Supabase URL + anon key
npm run dev
```

### Traveller App (Flutter)

```
cd mobile
flutter pub get
cp .env.example .env   # fill in your Supabase URL + anon key
flutter run
```

### Self-hosted translation server (NLLB-200)

```
cd translation-server
pip install -r requirements.txt   # ctranslate2, transformers, sentencepiece
python download_model.py          # fetches facebook/nllb-200-distilled-600M
python serve.py                   # exposes a local HTTP endpoint the Edge Function calls
```

See `FULL_BUILD_GUIDE.md` in this repo for the complete step-by-step build, including the
Supabase schema (`sessions` + `translations` tables), the Edge Function's tiered routing logic,
the session-expiry cleanup job, and deployment instructions for all three pieces.

## Status

🚧 In development — not yet deployed. Web client, Flutter client, and the tiered translation
router are being built against the same Supabase backend described above.
