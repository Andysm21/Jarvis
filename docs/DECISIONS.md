# Architecture Decision Records

Each entry: the decision, why, and what we rejected. Written so that in six months we
remember the reasoning and don't relitigate it by accident.

---

## ADR-001 — Free-tier-first, with documented paid upgrades

**Decision.** Every component must have a genuinely free path. Paid options are documented
as upgrades but never required to have a working system.

**Why.** Stated requirement. It also forces honest design: the constraints of free tiers
(the WhatsApp 24h window, Oracle's capacity limits) surface real architectural questions
early rather than after launch.

**Consequence.** One genuine cost remains — the language model. Addressed in ADR-008.

---

## ADR-002 — No FaceTime. Free voice is voice notes; ringing calls are a paid upgrade

**Decision.** Two-way voice runs over WhatsApp voice notes: `edge-tts` for synthesis,
Whisper for transcription. JARVIS does not place ringing calls in v1.

**Why FaceTime is off the table.** Apple publishes no API for initiating FaceTime calls.
There is no entitlement, no SDK, no server-side path. This isn't a tier or a permission
problem — the capability does not exist. Anyone claiming otherwise is describing a
`facetime://` URL scheme, which requires an unlocked, logged-in Apple device to already be
executing the link.

**Rejected — Mac + Shortcuts bridge.** An always-on Mac could open `facetime://` links on
command. It produces a genuine FaceTime ring, but: it needs hardware you'd have to leave
running and logged in, it breaks on reboot and on macOS updates, and critically there is
**no AI on the other end of the call** — it just rings. A ringing phone with nobody there
is a worse notification than a push.

**Rejected for now — Twilio Voice.** This is the real thing: JARVIS calls your actual
phone, you answer, and you have a live spoken conversation (Twilio Media Streams →
Whisper → Claude → TTS → back). It works properly. It is not free: roughly **$1.15/month**
for a number plus **~$0.014/minute**. Realistically $3–5/month.

**Upgrade path.** The channel layer treats voice as just another `Channel` implementation.
Adding Twilio later is a new file plus a webhook route — no changes to the brain, the
connectors, or the memory layer.

---

## ADR-003 — WhatsApp Cloud API on Meta's free test number

**Decision.** Official Meta Cloud API, using the free test phone number Meta provisions
with every WhatsApp Business app.

**Why.** It's free, it's officially supported, and it carries zero account-ban risk. The
test number's cap of 5 verified recipients is a non-issue for a personal assistant — you
need one.

**Rejected — unofficial bridges** (Baileys, `whatsapp-web.js`). These drive your real
WhatsApp account by automating the web client. Unlimited and free, but they violate
WhatsApp's Terms of Service and carry a real risk of permanent number bans. Not worth it
for your primary phone number.

**Rejected — production WhatsApp number.** Requires Meta Business verification, a real
phone number you own, and app review. More friction, no benefit at this scale.

**The constraint we accept.** Meta's customer service window: JARVIS may send free-form
messages only within 24 hours of your last message to it. Outside that window, only
pre-approved template messages are permitted, and those cost a fraction of a cent each.

**How we handle it.** ADR-004.

---

## ADR-004 — Window-aware notification routing, with ntfy.sh as the free fallback

**Decision.** Track WhatsApp window expiry in `state`. Route every outbound proactive
message on window state plus urgency:

| Window | Urgency | Route |
|---|---|---|
| Open | any | WhatsApp — free, unlimited |
| Closed | `AMBIENT` / `NORMAL` | Hold for the next brief, or push via ntfy |
| Closed | `IMPORTANT` | ntfy push |
| Closed | `URGENT` | High-priority ntfy push; optionally spend a WhatsApp template |

**Why ntfy.sh.** Free, open source, no account required, iOS and Android apps, supports
priority levels that can break through Do Not Disturb. It's also self-hostable on the same
VM if you'd rather not depend on the public instance.

**Practical note.** You'll message JARVIS most days — replying to the morning brief alone
reopens the window. The closed-window path will fire rarely.

---

## ADR-005 — Oracle Cloud Always Free as primary host

**Decision.** Deploy to an Oracle Cloud **Always Free** ARM (Ampere A1) instance:
up to 4 vCPUs, 24 GB RAM, 200 GB storage — free indefinitely, not a trial.

**Why.** It is the only major cloud offering a genuinely always-on VM for free, forever.
The resources are generous enough to run local Whisper transcription, so we're not even
dependent on a free API tier for speech.

**Known risk — capacity.** Oracle's free ARM capacity is frequently exhausted in popular
regions; sign-up can return "Out of host capacity" for days. Two mitigations:
1. Try a less-congested region at sign-up.
2. Fall back to Oracle's Always Free **AMD micro** instance (1/8 OCPU, 1 GB RAM) — always
   available, and adequate if we use hosted Whisper instead of local.

**Fallback stack, also 100% free**, if Oracle proves unworkable:
- **Google Cloud Run** — 2M requests/month free, scales to zero. Cold start of a few seconds is acceptable for chat.
- **Turso** (hosted SQLite) or **Supabase** (Postgres) for storage, since Cloud Run has no persistent disk.
- **GitHub Actions cron** or **Cloud Scheduler** (3 free jobs) to drive scheduled work.

**Rejected — Fly.io / Railway / Render.** Fly and Railway no longer have meaningful free
tiers. Render's free web services sleep after 15 minutes with ~50s cold starts, which
undermines "always live."

**Portability.** Everything ships as Docker. Moving hosts is a redeploy.

---

## ADR-006 — SQLite, not Postgres, and not a vector database

**Decision.** A single SQLite file in WAL mode, with FTS5 for text search.

**Why.** Single-user, low-write, read-mostly workload. SQLite handles this with orders of
magnitude of headroom, needs no server process, and backs up by copying one file.

**Why not a vector DB.** At personal scale, structured tables plus full-text search give
better retrieval than embeddings, and are debuggable — you can read a row and understand
why it surfaced. Embeddings are a remedy for a retrieval problem we don't have yet.
Revisit if and only if recall measurably degrades.

**Backups.** Nightly encrypted dump to a free object store (Cloudflare R2's free tier, or
a private GitHub repo). Losing JARVIS's memory is the worst non-security failure mode here.

---

## ADR-007 — Python

**Decision.** Python 3.12.

**Why.** Google's API client libraries, `faster-whisper`, `edge-tts`, and the Claude Agent
SDK are all first-class in Python. TypeScript would need workarounds for the audio stack.
The performance difference is irrelevant at one user.

---

## ADR-008 — The brain: hybrid routing to keep model cost near zero

**Decision.** Route by task difficulty rather than using one model for everything.

- **Cheap tier** — Google **Gemini Flash**, free tier (generous daily request allowance). Handles email classification, intent detection, summarization, routing. This is the overwhelming majority of calls.
- **Smart tier** — **Claude** for actual reasoning, drafting in your voice, multi-step tool use, and anything you'll read closely.

**Why.** Model calls are the only unavoidable cost, and most of JARVIS's calls are trivial
classification work that does not need a frontier model. Routing collapses the bill to near
zero without compromising the output you actually read.

**Alternative worth considering.** If you hold a Claude Pro or Max subscription, the Claude
Agent SDK can authenticate against it rather than metered API credits — no incremental cost
at all. This is likely the best option for you; it needs confirming against your current
subscription. See [COSTS.md](COSTS.md).

**Design consequence.** The brain talks to a provider interface, not to a vendor SDK
directly. Swapping or re-routing models must never require touching tools or connectors.

---

## ADR-009 — Confirm-before-acting, by default, forever

**Decision.** Tools are classified `READ`, `WRITE`, or `SENSITIVE`. Reads execute freely.
Writes and sensitives are intercepted by an approval gate that asks you first, in plain
language, showing exactly what will happen.

**Why.** This system holds send-as-you access to your email. The failure mode of an
over-eager agent isn't a bad answer — it's an email you didn't write, sent to your boss,
that you cannot recall. The cost of confirming is a few seconds. The cost of not confirming
is unbounded.

**Earned trust.** Specific actions can be individually promoted to auto-execute once
they've proven themselves ("archive newsletters without asking"). This is opt-in per
action, never a global switch, and always reversible.

**Never auto-approved, under any circumstance:** sending email to a new recipient,
deleting anything, financial actions, changing JARVIS's own permissions.

---

## ADR-010 — Connectors are leaf nodes

**Decision.** Connector modules expose plain functions and never import from the channel
layer, the brain, or each other.

**Why.** Testability and swappability. Adding Microsoft Graph later — which you want — must
be a new leaf module registering new tools, with no changes anywhere else. The moment a
connector knows about a channel, that property is lost.

---

## ADR-011 — A single notification chokepoint

**Decision.** No component sends you an unsolicited message directly. Every proactive
candidate goes through `proactive/policy.py`, which enforces quiet hours, per-category rate
limits, duplicate suppression, and batching.

**Why.** The way an assistant like this fails is not by being wrong — it's by being
annoying enough that you mute it. One chokepoint means noise is a tuning problem in one
file, rather than an emergent property of a dozen independent notifiers.
