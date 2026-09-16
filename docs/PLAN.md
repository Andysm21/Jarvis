# JARVIS — Master Plan

## 1. What we're building

An always-on personal assistant that lives on the internet, not on your laptop. You reach
it from your phone; it reaches you when it has something worth saying. It has real access
to your digital life — mail, calendar, code — and a memory that persists across every
conversation.

The measure of success is not "it responds to prompts." It's: **how many small decisions,
lookups, and follow-ups does this remove from your week?**

## 2. What "always live" actually means

Three distinct properties, often confused:

| Property | What it means | How we get it |
|---|---|---|
| **Reachable** | You can message it at 3am and get an answer | Public HTTPS webhook on an always-on VM |
| **Proactive** | It messages *you* without being asked | Cron scheduler + event watchers on Gmail/Calendar |
| **Continuous** | It remembers yesterday | Persistent SQLite memory + conversation history, not per-session context |

All three are in scope. The third is what makes it feel like an assistant instead of a chatbot.

## 3. Channels — how you and JARVIS talk

### Primary: WhatsApp
Official **WhatsApp Cloud API** on Meta's free test number. Free, no ban risk, no business
verification needed. Limited to 5 verified recipient numbers, which is exactly right for a
personal assistant.

**The one real constraint:** outside a 24-hour window from your last message to JARVIS,
Meta only permits pre-approved template messages (which cost fractions of a cent each).
Inside the window, free-form messaging is unlimited and free.

We handle this with a **window-aware router**. JARVIS tracks when you last messaged it:

- **Window open** → send freely over WhatsApp, text or voice note. Free.
- **Window closed** → route the notification to the fallback push channel instead. Also free.
- **Window closed AND it's genuinely urgent** → optionally spend the template message.

In practice you'll message it most days, so the window is usually open and this rarely fires.

### Fallback + urgent: ntfy.sh push
Free, open-source push notifications to your phone. No account needed, works on iOS and
Android, supports priority levels that can cut through Do Not Disturb. This is the escape
hatch when the WhatsApp window is closed, and the escalation path for anything truly urgent.

### Voice — both directions, free
- **JARVIS → you:** text is synthesized with `edge-tts` (free, natural-sounding neural voices) and delivered as a WhatsApp voice note.
- **You → JARVIS:** send a voice note, the webhook pulls the audio, Whisper transcribes it, the agent handles it like any message.

This is a complete spoken conversation loop at zero cost. What it is *not* is a ringing
phone. See [DECISIONS.md](DECISIONS.md#adr-002) for why FaceTime is impossible and what a
real ringing call would cost if you later want one.

## 4. Capabilities — what JARVIS can actually do

### Email (Gmail)
- Read, search, and summarize your inbox
- **Triage:** classify incoming mail as urgent / needs-reply / FYI / noise, and surface only what matters
- Draft replies in your voice, based on how you've written before
- Send — **with confirmation by default**
- Label, archive, and mute threads
- Track "you said you'd get back to them" and nudge you

### Calendar (Google Calendar)
- Read today, this week, find free slots
- Create, move, and cancel events — with confirmation
- Detect conflicts and double-bookings before they bite
- Pre-meeting prep: who is this person, what did we last discuss, what's the attached doc
- Travel-time sanity checks between back-to-back events

### Claude sessions
- Kick off Claude Code jobs against your repos from a WhatsApp message
- Report back when a session finishes, fails, or needs a decision
- Summarize what your running sessions are doing
- Relay PR review comments and CI failures to you, and your answers back

### Memory
- Facts about you, your people, your projects, your preferences
- Conversation history that survives restarts
- Open loops and commitments, with follow-up dates
- Corrections: tell it "no, I prefer X" once and it sticks

### General
- Web search for current information
- Timers, reminders, and one-off scheduled messages
- Quick capture — dump a thought, JARVIS files it correctly

## 5. Proactive behaviours — the actual value

This is the part that makes it more than a chat window. Each is a scheduled job or a watcher
that can decide, on its own, that you need to hear something.

| When | What JARVIS does |
|---|---|
| Morning (7:30am) | **Daily brief** — today's calendar, overnight email worth knowing about, open commitments, anything due |
| Continuously | **Email triage** — genuinely important mail arrives, you get a summary and a suggested reply, not a raw notification |
| 15 min before a meeting | **Prep ping** — who's attending, last conversation, relevant docs, the link |
| On calendar change | **Conflict alert** — someone booked over your existing thing |
| Evening (6pm) | **Wrap-up** — what got done, what slipped, tomorrow's shape |
| Weekly (Sunday) | **Review** — the week's pattern, dropped balls, next week's load |
| Event-driven | **Claude session done** — your build finished / your PR went red |
| Ad hoc | **Commitment nudge** — "you told Sarah you'd send the deck by Friday. It's Friday." |

Every one of these is individually toggleable and rate-limited. Design rule 3 applies: an
interrupt has to earn itself.

## 6. Explicitly out of scope (for v1)

Naming these keeps the build honest:

- **Real ringing phone calls.** Needs paid telephony. Documented upgrade, not v1.
- **FaceTime.** Not technically possible. See ADR-002.
- **Microsoft 365 / Teams.** You want it later. The connector layer is built so it slots in without refactoring — but we don't build it now.
- **Multi-user.** This is yours alone. No tenancy, no user management.
- **A web dashboard.** Chat is the interface. Maybe later.
- **Smart home / IoT.** Different project.
- **Autonomous spending.** JARVIS never touches money.

## 7. Open questions to settle before Phase 1

1. **Which brain?** Three options in [COSTS.md](COSTS.md#the-brain) ranging from free to ~$15/mo. Needs your call — it's the only real cost.
2. **Trust ceiling.** Which actions, if any, should JARVIS eventually do without asking? (Suggested start: nothing. Earn it.)
3. **Quiet hours.** When should JARVIS never ping you short of an emergency?
4. **Whose email is it?** Personal Gmail only, or a work account too?
5. **Wake word / prefix.** Should every WhatsApp message go to JARVIS, or only ones starting with a trigger?

None of these block Phase 0.
