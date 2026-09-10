# TalkThatTalk

Talk to anyone, in your own language. No sign-up. Nothing saved.

TalkThatTalk is a zero-login, ephemeral chat for travellers. Start a chat, get a QR code, hand your
phone (or point a camera) at the other person — they scan, pick their own language, and you're
talking. Each person types and reads in their own language, live. Close the chat (or walk away for
5 minutes) and it's gone for good — nothing is ever written to a database.

## How it works

1. **Start** — pick your language, tap *Start Chat*. You get a QR code.
2. **Join** — the other person scans it, picks their language, taps *Join*.
3. **Talk** — you type in your language, they read it in theirs, and vice versa — same
   conversation, two languages.
4. **Gone** — tap *End Chat*, or just leave it idle for 5 minutes. The conversation is never stored,
   so there's nothing left to find afterward.

## Stack

| Layer | Tech |
|---|---|
| Frontend | React + Vite, deployed on Cloudflare Pages |
| Realtime messaging | Supabase Realtime (Broadcast channels — messages never touch the database) |
| Session pairing | Supabase Postgres (a single `sessions` table holding language prefs only) |
| Translation | Google Gemini API, called from a Supabase Edge Function |
| QR codes | `qrcode.react`, generated client-side |

No servers to manage, no card required anywhere in the free tiers used.

## Why no messages are stored

The privacy promise ("the chat is gone forever") only means something if it was never really
written down. Messages travel peer-to-peer through a Supabase Realtime broadcast channel — the
database only ever holds which two languages are talking to each other, not what was said, and that
pairing record deletes itself within 5 minutes either way.

## Local development

```bash
cd web
npm install
cp .env.example .env   # fill in your Supabase URL + anon key
npm run dev
```

See `FULL_BUILD_GUIDE.md` in this repo for the complete step-by-step build, including the Supabase
schema, Edge Functions, and deployment instructions.

## Status

🚧 In development — not yet deployed.
