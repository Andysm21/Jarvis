# JARVIS

A personal AI assistant that is always on, reachable from your phone, and allowed to act
on your behalf — read and triage your email, manage your calendar, run Claude Code jobs
on your repos, and proactively tell you the things you'd otherwise forget.

**Status: planning.** No application code yet. This repo currently holds the design.

## Start here

| Document | What's in it |
|---|---|
| [docs/PLAN.md](docs/PLAN.md) | The master plan — vision, scope, what JARVIS does day to day |
| [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) | How the system is put together, and why |
| [docs/DECISIONS.md](docs/DECISIONS.md) | Every significant technical choice, with the reasoning and the rejected alternatives |
| [docs/ROADMAP.md](docs/ROADMAP.md) | Phased build order, each phase a working milestone |
| [docs/SETUP.md](docs/SETUP.md) | The accounts and credentials **you** need to create before coding starts |
| [docs/COSTS.md](docs/COSTS.md) | Honest money breakdown — what's free, what isn't, where the bill comes from |
| [docs/SECURITY.md](docs/SECURITY.md) | This thing can send email as you. How we keep that safe |

## The one-paragraph version

JARVIS runs 24/7 on a free Oracle Cloud VM. You talk to it in WhatsApp — typing or voice
notes — and it answers in kind. Behind the chat it holds a Claude-powered agent loop with
tools for Gmail, Google Calendar, Claude Code, and its own long-term memory. It doesn't
only wait to be asked: a scheduler and a set of watchers give it reasons to message you
first — a morning brief, an email that actually matters, a meeting you haven't prepped
for, a promise you made last week and haven't kept.

## Design rules

1. **Free by default.** Every component has a zero-cost path. Paid upgrades are documented but never required.
2. **Ask before acting outward.** Reading is automatic. Sending an email, moving a meeting, or touching a repo needs your confirmation until you explicitly trust that action.
3. **Proactive, not noisy.** A notification has to earn its interrupt. Better silent than chatty.
4. **Everything is logged.** Every action JARVIS takes is recorded and reviewable.
5. **Replaceable parts.** Channels, model providers, and connectors sit behind interfaces so any one of them can be swapped without a rewrite.
