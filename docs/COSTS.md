# Costs

You asked for free. Here's the honest accounting — what's genuinely free, what has a
catch, and where the only real bill comes from.

## The baseline: what JARVIS costs to run

| Component | Choice | Cost |
|---|---|---|
| Server | Oracle Cloud Always Free ARM | **$0** — free indefinitely, not a trial |
| Domain | DuckDNS, or ~$2/yr for a real one | **$0** |
| TLS certificate | Caddy + Let's Encrypt | **$0** |
| WhatsApp | Cloud API, free test number | **$0** — see the caveat below |
| Push notifications | ntfy.sh | **$0** |
| Text-to-speech | edge-tts | **$0** — no key, no quota |
| Speech-to-text | Groq free tier, or local Whisper | **$0** |
| Database | SQLite on the instance | **$0** |
| Backups | Cloudflare R2 free tier (10 GB) | **$0** |
| Scheduling | APScheduler, in-process | **$0** |
| **Infrastructure total** | | **$0/month** |

Everything except the model is free, permanently, with no trial clock.

---

## The brain

This is the only genuine cost, and it's the decision to make before Phase 1.

### Option A — Your Claude subscription *(probably your best answer)*

If you already pay for **Claude Pro or Max**, the Claude Agent SDK can authenticate against
that subscription rather than metered API credits. JARVIS then costs **nothing beyond what
you already pay**.

- **Cost:** $0 incremental
- **Quality:** full Claude
- **Caveat:** subject to subscription rate limits, and this needs confirming against your
  current plan and its terms before we build on it. First thing to verify in Phase 0.

### Option B — Hybrid routing *(recommended fallback)*

Route by difficulty. Google's **Gemini Flash** free tier absorbs the high-volume, low-stakes
work — email classification, intent detection, summarization, notification triage. Claude
handles what you'll actually read closely: drafting in your voice, reasoning, multi-step
tool use.

Since classification massively outnumbers composition, this collapses the bill.

- **Cost:** roughly **$0–3/month**
- **Quality:** indistinguishable where it counts
- **Caveat:** two providers to maintain; Gemini's free tier has daily request caps

### Option C — Claude API for everything

Simplest to build, best output, metered.

- **Cost:** **$5–15/month** at personal volume
- **Quality:** highest
- **Caveat:** it's a real monthly bill

**Recommendation:** verify Option A first. If your subscription doesn't permit it, build
Option B — the architecture routes through a provider interface either way, so this choice
is reversible without a rewrite.

---

## The WhatsApp caveat, priced

WhatsApp is free for the way you'll actually use it, with one boundary worth understanding.

**Free, unlimited:** every message you send JARVIS, and everything JARVIS sends back within
24 hours of your last message. This covers essentially all normal use.

**Costs money:** JARVIS messaging you *unprompted* when you haven't spoken to it in over
24 hours. That requires a pre-approved template message, priced in fractions of a cent.

**Why this will rarely bite you:** replying to the morning brief reopens the window for
another 24 hours. And when the window is shut, [ADR-004](DECISIONS.md#adr-004) routes the
notification to ntfy push instead — free.

If you ever decided you wanted everything in WhatsApp regardless of window state, the
realistic worst case is a few dozen template messages a month: **under $1**.

---

## Optional upgrades, priced

| Upgrade | What you get | Cost |
|---|---|---|
| **Twilio Voice** | JARVIS calls your actual phone; live spoken conversation | ~$1.15/mo for the number + ~$0.014/min → realistically **$3–5/mo** |
| **ElevenLabs TTS** | Noticeably better voice than edge-tts | Free tier exists; **~$5/mo** for real use |
| **WhatsApp production number** | No 5-recipient cap, no window-driven routing | Meta verification + per-message pricing |
| **Bigger server** | If the free instance ever runs out | **~$5/mo** on Hetzner |

---

## Bottom line

| Configuration | Monthly |
|---|---|
| Everything free + Claude subscription (Option A) | **$0** |
| Everything free + hybrid brain (Option B) | **$0–3** |
| Everything free + Claude API (Option C) | **$5–15** |
| Option B + Twilio ringing calls | **$3–8** |

**A fully working JARVIS at $0/month is achievable** if Option A holds. If it doesn't,
Option B gets you to a few dollars — and the architecture lets you switch between them
later without touching anything outside one module.
