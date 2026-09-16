# Roadmap

Seven phases. Each one ends with something that visibly works — no phase is pure
scaffolding. Build order is deliberate: the conversation loop comes before capabilities,
because a JARVIS you can talk to is testable and a pile of connectors is not.

---

## Phase 0 — Foundations
*Goal: an empty JARVIS, deployed and reachable on the internet.*

- Python project scaffold, `pyproject.toml`, linting, pre-commit
- `config.py` — settings from environment, validated with pydantic
- FastAPI skeleton with `/health`
- SQLite schema and migration runner
- Dockerfile, docker-compose, Caddyfile
- Deployed to Oracle Cloud with HTTPS and a real domain
- CI: lint and test on push

**Done when:** `curl https://your-domain/health` returns 200 from the cloud.

---

## Phase 1 — It's alive
*Goal: a real conversation over WhatsApp, with memory.*

- WhatsApp Cloud API webhook: signature verification, message parsing
- Outbound WhatsApp send
- Channel abstraction + `InboundMessage` / `OutboundMessage`
- Claude agent loop with the system prompt
- Memory: store messages, retrieve recent history
- WhatsApp 24h window tracking
- ntfy fallback channel

**Done when:** you message JARVIS from your phone, it replies sensibly, and it remembers
what you said yesterday.

**This is the milestone that matters most.** Everything after it is adding capability to a
working assistant.

---

## Phase 2 — Voice
*Goal: talk to it, hear it back.*

- `edge-tts` synthesis → WhatsApp voice note delivery
- Inbound voice notes: media download → Whisper transcription
- Groq Whisper with local `faster-whisper` fallback
- Voice/text preference: reply in the mode you used
- Audio caching to avoid re-synthesizing repeats

**Done when:** you send a voice note and get a spoken reply.

---

## Phase 3 — Google
*Goal: JARVIS can see and act on your email and calendar.*

- Google OAuth flow, token storage and refresh (`scripts/google_auth.py`)
- **Gmail:** search, read, thread summarization, labels, archive
- **Gmail:** draft and send — behind the approval gate
- **Calendar:** list, find free slots, conflict detection
- **Calendar:** create, move, cancel — behind the approval gate
- Approval gate itself: interception, plain-language prompts, persisted pending state
- Full audit log of every action

**Done when:** "what's on tomorrow, and draft a reply to Sarah's email" works end to end,
and the send waits for your yes.

---

## Phase 4 — Proactive
*Goal: JARVIS messages you first, and it's useful rather than annoying.*

- APScheduler wired in, jobs surviving restart
- Morning brief, evening wrap, weekly review
- Gmail watcher — polling on a history cursor
- Email triage classifier, with a suggested reply on important mail
- Calendar watcher — meeting prep pings, conflict alerts
- Commitment tracking and follow-up nudges
- Notification policy: quiet hours, rate limits, deduplication, batching

**Done when:** a week goes by and the briefs are good enough that you'd miss them.

---

## Phase 5 — Claude sessions
*Goal: run and monitor dev work from your phone.*

- Claude Code session connector: launch, status, results
- Kick off a session from a WhatsApp message
- Completion and failure notifications
- Relay PR review comments and CI failures; send your answers back
- Summarize running sessions

**Done when:** "start a session on the Jarvis repo to add X" works from your phone and
reports back on its own.

---

## Phase 6 — Hardening
*Goal: trustworthy enough to leave running unattended.*

- Secrets: encrypted at rest, rotation procedure documented
- Nightly encrypted memory backups to free object storage
- Structured logging, error alerting to you via ntfy
- Rate limiting and abuse protection on webhooks
- Graceful degradation — one dead connector must not take the system down
- Restore-from-backup, actually tested
- Cost monitoring with alerts

**Done when:** it survives a week with no attention, and you'd trust it while on holiday.

---

## Phase 7 — Upgrades *(optional, on demand)*

Independent, pick as needed:

- **Twilio voice** — actual ringing phone calls with live conversation (~$3–5/mo)
- **Microsoft Graph** — Outlook mail, calendar, Teams (you've flagged wanting this)
- **Web dashboard** — audit log, settings, memory browsing
- **Documents** — Drive and Docs access
- **Richer memory** — embeddings, if and only if retrieval quality demands it
- **Task systems** — Todoist, Linear, Notion

---

## Sequencing notes

- **Phases 0–2 have no dependency on you** beyond the credential setup in [SETUP.md](SETUP.md).
- **Phase 3 needs Google OAuth**, which needs a Google Cloud project — do this early, it involves a consent screen that can take a moment.
- **Phase 4 is where the value is.** Phases 1–3 build the machine; Phase 4 is what actually makes your life easier. Don't stall before it.
- **Phase 6 shouldn't wait for a formal phase.** Log and audit as you build, not after.
