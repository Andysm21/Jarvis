# Architecture

## 1. System shape

```
                   ┌──────────────────────────────────────────────┐
   YOU             │              ORACLE CLOUD VM (free)          │
   📱 WhatsApp ───▶│                                              │
   🔔 ntfy    ◀────│   ┌──────────────────────────────────────┐   │
                   │   │        FastAPI  (webhooks + API)     │   │
                   │   └──────────────┬───────────────────────┘   │
                   │                  │                           │
                   │   ┌──────────────▼───────────────────────┐   │
                   │   │           CHANNEL LAYER              │   │
                   │   │  whatsapp · ntfy · voice(TTS/STT)    │   │
                   │   │  normalises everything to a Message  │   │
                   │   └──────────────┬───────────────────────┘   │
                   │                  │                           │
                   │   ┌──────────────▼───────────────────────┐   │
                   │   │              BRAIN                   │   │
                   │   │   Claude agent loop · system prompt  │   │
                   │   │   tool dispatch · approval gate      │   │
                   │   └───┬──────────────────────────┬───────┘   │
                   │       │                          │           │
                   │  ┌────▼─────────┐        ┌───────▼────────┐  │
                   │  │  CONNECTORS  │        │     MEMORY     │  │
                   │  │  gmail       │        │  SQLite        │  │
                   │  │  gcal        │        │  facts         │  │
                   │  │  claude_code │        │  history       │  │
                   │  │  websearch   │        │  commitments   │  │
                   │  └──────────────┘        └────────────────┘  │
                   │                                              │
                   │   ┌──────────────────────────────────────┐   │
                   │   │          PROACTIVE ENGINE            │   │
                   │   │  APScheduler cron · Gmail watcher    │   │
                   │   │  calendar watcher · nudge engine     │   │
                   │   └──────────────────────────────────────┘   │
                   └──────────────────────────────────────────────┘
```

## 2. The four layers

### Channel layer
Owns every way a message enters or leaves. Its job is **normalization**: an inbound
WhatsApp text, an inbound voice note, and a scheduled internal trigger all become the same
`InboundMessage` object before the brain sees them. Outbound works in reverse — the brain
emits an `OutboundMessage` and the router decides delivery (WhatsApp text / WhatsApp voice
note / ntfy push) based on content, urgency, and WhatsApp window state.

This is why adding Telegram or SMS later is a single new file, not a refactor.

```python
class InboundMessage:
    text: str              # transcribed if it arrived as audio
    source: Channel
    sender_id: str
    attachments: list[Attachment]
    received_at: datetime
    raw: dict              # original payload, for debugging

class OutboundMessage:
    text: str
    urgency: Urgency       # AMBIENT | NORMAL | IMPORTANT | URGENT
    prefer_voice: bool
    requires_response: bool
```

`urgency` drives routing. `AMBIENT` may be batched into the next brief instead of sent at
all; `URGENT` bypasses quiet hours and escalates to a high-priority ntfy push if WhatsApp
can't deliver.

### Brain
A Claude agent loop. Receives a normalized message plus assembled context, decides on tool
calls, executes them through the approval gate, and produces a reply.

Context assembled per turn:
- System prompt (persona, capabilities, current policy, quiet hours)
- Relevant long-term memory (retrieved, not dumped — keep it tight)
- Recent conversation history
- Live situational snapshot: current time and timezone, today's calendar, unread count
- The message itself

**The approval gate** sits between tool *decision* and tool *execution*. Tools are tagged
`READ`, `WRITE`, or `SENSITIVE`. Reads execute immediately. Writes and sensitives get
intercepted: JARVIS messages you a plain-language description of what it's about to do and
waits for a yes. Pending approvals are persisted, so a restart doesn't lose them and you
can approve twenty minutes later.

### Connectors
One module per external service, each exposing plain Python functions that are registered
as Claude tools. Connectors never touch the channel layer or the brain directly — they're
leaf nodes. They own their own auth, their own rate-limit handling, and their own retry
logic.

Planned: `gmail`, `gcal`, `claude_code`, `websearch`. Later: `msgraph` (Outlook/Teams).

### Memory
SQLite. Not a vector database — at personal scale, structured tables plus SQLite's built-in
full-text search outperform embeddings for both accuracy and simplicity. Revisit only if
retrieval quality actually degrades.

Tables:

| Table | Holds |
|---|---|
| `messages` | Every inbound and outbound message, with channel and timestamp |
| `facts` | Durable knowledge: preferences, people, projects, recurring context |
| `commitments` | Things you said you'd do, with who/what/when and a status |
| `entities` | People and orgs, with aliases, so "Sarah" resolves consistently |
| `actions` | Audit log — every tool call, its arguments, result, and approval state |
| `approvals` | Pending and resolved confirmation requests |
| `state` | Key-value runtime state: WhatsApp window expiry, watcher cursors, job runs |

### Proactive engine
Two sources of self-initiated action:

**Scheduled** — APScheduler running in-process, cron-style. Morning brief, evening wrap,
weekly review, commitment sweeps.

**Event-driven** — watchers that poll or subscribe:
- *Gmail:* start with polling the History API on a cursor (simple, no public-endpoint requirement). Upgrade to Pub/Sub push if latency matters.
- *Calendar:* incremental sync tokens to catch changes cheaply.
- *Claude Code:* session status transitions.

Both funnel into the same place: a candidate notification. Every candidate passes through
the **notification policy** before it can reach you — checks quiet hours, per-category rate
limits, duplicate suppression, and whether it's worth more than being folded into the next
scheduled brief. This single chokepoint is what keeps JARVIS from becoming annoying.

## 3. Stack

| Concern | Choice | Why |
|---|---|---|
| Language | **Python 3.12** | Best libraries for Google APIs, audio, and the Claude SDK in one place |
| Web | **FastAPI + uvicorn** | Async, fast, trivial webhook handling, auto-generated docs |
| Agent | **Claude Agent SDK** | Tool use, streaming, and the agent loop already solved |
| Storage | **SQLite** (WAL mode) | Zero-ops, fast enough by orders of magnitude, one file to back up |
| Scheduling | **APScheduler** | In-process cron, no extra infrastructure |
| TTS | **edge-tts** | Free, no key, genuinely good neural voices |
| STT | **Groq Whisper** free tier, local `faster-whisper` fallback | Fast and free; local fallback means no hard dependency |
| Deploy | **Docker + docker-compose** | Portable — moving off Oracle later is a redeploy, not a rewrite |
| Proxy/TLS | **Caddy** | Automatic Let's Encrypt certificates with near-zero config |

## 4. Repository layout

```
jarvis/
├── app/
│   ├── main.py                 # FastAPI app, webhook routes, lifespan
│   ├── config.py               # Settings from env, validated with pydantic
│   │
│   ├── channels/
│   │   ├── base.py             # Channel protocol, Inbound/OutboundMessage
│   │   ├── whatsapp.py         # Cloud API send/receive, window tracking
│   │   ├── ntfy.py             # Push fallback
│   │   ├── router.py           # Urgency + window -> delivery decision
│   │   └── voice.py            # TTS out, STT in
│   │
│   ├── brain/
│   │   ├── agent.py            # The Claude loop
│   │   ├── prompts.py          # System prompt, assembled per turn
│   │   ├── context.py          # Gathers memory + situation into context
│   │   └── tools/              # Tool definitions, registry, schemas
│   │
│   ├── connectors/
│   │   ├── google_auth.py      # Shared OAuth, token refresh
│   │   ├── gmail.py
│   │   ├── gcal.py
│   │   ├── claude_code.py
│   │   └── websearch.py
│   │
│   ├── memory/
│   │   ├── db.py               # Connection, migrations
│   │   ├── models.py
│   │   └── retrieval.py        # What's relevant to this turn
│   │
│   ├── proactive/
│   │   ├── scheduler.py        # APScheduler setup and job registry
│   │   ├── policy.py           # Quiet hours, rate limits, dedup
│   │   ├── briefs.py           # Morning / evening / weekly
│   │   └── watchers/           # gmail.py, calendar.py, sessions.py
│   │
│   └── approvals/
│       ├── gate.py             # Intercepts WRITE and SENSITIVE tool calls
│       └── policy.py           # Per-action trust settings
│
├── tests/
├── docs/
├── scripts/                    # oauth bootstrap, db init, deploy
├── Dockerfile
├── docker-compose.yml
├── Caddyfile
├── pyproject.toml
└── .env.example
```

## 5. Two flows, end to end

**You send a voice note: "what's my day look like, and did Sarah ever reply?"**

1. Meta POSTs the webhook → FastAPI verifies the signature
2. `whatsapp.py` sees an audio attachment, downloads it from Meta's media endpoint
3. `voice.py` transcribes via Whisper → `InboundMessage(text=...)`
4. WhatsApp window is stamped open for the next 24h
5. `context.py` assembles memory, recent history, today's date and calendar
6. Brain calls `gcal.list_events(today)` and `gmail.search(from:sarah)` — both `READ`, so they run immediately
7. Brain composes a reply; because the message arrived as voice, `prefer_voice=True`
8. `voice.py` synthesizes it; router sends it back as a WhatsApp voice note
9. Both messages are written to `messages`; both tool calls to `actions`

**An important email lands at 2pm**

1. Gmail watcher polls, sees new history since its cursor
2. Brain classifies it: sender is known, subject matches an open commitment, contains a question → `IMPORTANT`
3. Notification policy: not quiet hours, under the hourly cap for this category, not a duplicate → allow
4. WhatsApp window state checked — open (you messaged this morning) → deliver over WhatsApp, free
5. You get a summary, why it matters, and a drafted reply
6. You answer "send it"
7. `gmail.send` is a `WRITE` tool → approval gate shows you the exact final text
8. You confirm → it sends → logged to `actions` → the commitment is closed
