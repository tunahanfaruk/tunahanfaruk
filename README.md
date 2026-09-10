<div align="center">

<img src="logo.png" alt="Xorviex" width="110" />

# Xorviex

### Set the goal. Xorviex spawns the workforce.

**A Self-Architecting AI Executive** — you give it a plain-language mission,
it designs and spawns the sub-agents that mission actually needs, lets each one
research and *act* through real tools, and routes every output through
independent Supervisor AIs before anything touches the real world.

<img src="https://img.shields.io/badge/%20%20%20%20%20%20%20%20%20%20-ffffff?style=for-the-badge" height="46" alt="" /><a href="https://xorviex.ai"><img src="btn-xorviex.svg" alt="Website" height="46" /></a><img src="https://img.shields.io/badge/%20%20%20%20%20%20-ffffff?style=for-the-badge" height="46" alt="" /><a href="https://www.linkedin.com/company/xorviex/"><img src="btn-linkedin.svg" alt="LinkedIn" height="46" /></a><img src="https://img.shields.io/badge/%20%20%20%20%20%20-ffffff?style=for-the-badge" height="46" alt="" /><a href="https://x.com/xorviex"><img src="btn-x.svg" alt="X" height="46" /></a><img src="https://img.shields.io/badge/%20%20%20%20%20%20%20%20%20%20-ffffff?style=for-the-badge" height="46" alt="" />

</div>

---

## The idea in one paragraph

Most "AI agent" products hand you a fixed org chart: a researcher, a writer, a
reviewer, forever. Xorviex doesn't. You describe an outcome — *"find every
distributor in the DACH region selling a competing product and tell me how our
pricing compares"* — and the planner **designs the team**: how many agents,
what each one's job is, what tools each one gets. There is no hardcoded role
list and no fixed agent count. The org chart is generated per mission.

Then each agent goes to work. Not by writing code that gets executed — by
running a **real tool-use loop** inside a trusted worker: the model picks a
tool, the worker calls it, the genuine result comes back, the model picks the
next move. Nothing is `exec()`'d anywhere in the agent path.

---

## How it works

```
   Plain-language goal
            │
            ▼
   ┌────────────────────┐
   │  Mission Planner   │  the model designs the team: N agents,
   │        (LLM)       │  free-form roles, per-agent directives
   └─────────┬──────────┘
             │  spawn
   ┌─────────┴───────────────────────────────────┐
   ▼                  ▼                          ▼
┌────────┐        ┌────────┐               ┌──────────┐
│ Agent  │        │ Agent  │      ...      │  Agent   │   ← real tool-use loops
└───┬────┘        └───┬────┘               └────┬─────┘
    │  search · scrape · read · post · push · write
    │                │                          │
    └────────────────┴──────────┬───────────────┘
                                ▼
                  ┌──────────────────────────┐
                  │  Shared Mission Memory   │  blackboard: every agent's
                  │      (blackboard)        │  findings become peer context
                  └────────────┬─────────────┘
                               ▼
                  ┌──────────────────────────┐
                  │    Supervisor AIs        │  independent audit of every
                  │  approve · reject · retry│  output before it ships
                  └────────────┬─────────────┘
                               ▼
                        Real-world action
```

**Mission lifecycle:** `draft → spawning → running ⇄ paused → resolved | failed`
(and `failed → spawning` to retry).

---

## What makes it different

### 🧠 The AI decides — the code is only the sensor
The core principle: strategy lives in the model, not in a hand-written
if/else tree. Xorviex's Python doesn't decide *what* to do about a competitor's
price drop; it decides how to *measure* it accurately and hand the model a true
reading. Security boundaries are the deliberate exception — those stay
hand-written and reviewed.

### 🛠️ Agents act, not just research
An agent's tool belt is assembled per run: **15 read tools** always available
(live web search, page reading, headless-browser scraping, entity discovery,
social engagement data, brand & news mention search, metric history), plus
**write tools for exactly the integrations that workspace connected** — and
nothing more.

### 👁️ Every output is audited by an independent AI
Supervisor agents grade each agent's output against a role-specific rubric.
A rejection feeds back as concrete revision guidance and the agent runs again;
persistent failure escalates to `needs_human` rather than silently shipping.

### 🩹 Self-healing runtime
Every crash is fingerprinted into a deduplicated incident row. A background
monitor classifies it — environmental → alert & suppress, recurring → claim &
heal, thrashing → escalate to a human — and re-runs the agent automatically.

### 📈 Measurements outlive their mission
An append-only metric log `(subject, metric, value, source, time)` means
"is this rising or falling?" is answerable across missions. And when there's
only one reading, it reports `insufficient_history` instead of inventing a 0%
change — a rule that runs through the whole platform: **a number without a
denominator is not reported.**

### 🔁 Recurring missions
Any mission can carry a real cron expression and become a template that spawns
a fresh, independent child mission on schedule. Due-ness is anchored off the
last fire, not the current tick, so a late sweep never skips or double-fires.

### 💬 Command Chat
A conversational control surface over the whole workspace — create missions,
revise agent directives mid-flight, edit workspace memory, all through a
tool-use loop that can ask *you* a clarifying question when the target is
ambiguous.

---

## Architecture

Two independent services:

| | Stack |
|---|---|
| **Backend** | FastAPI · async SQLAlchemy · PostgreSQL · Redis · taskiq · LLM provider SDK · Docker |
| **Frontend** | Next.js 16 (App Router) · React 19 · TypeScript · Tailwind CSS v4 · Cloudflare Workers / OpenNext |

### Isolation tiers

Untrusted work never runs at the same trust level as the platform:

- **Scraper sandbox** — disposable Playwright/Chromium containers with an
  SSRF egress guard, for any URL an agent discovers mid-run.
- **Dev sandbox** — its own image, its own Docker network, its own
  docker-socket-proxy and **no `EXEC` capability**; backs the `code_fixer`
  pipeline (clone a real repo → run its real tests → ask the model for a
  patch → open a PR, never a direct push to `main`) and the paired
  `builder`/`tester` agents that ship a product increment and verify it
  concurrently.
- **Everything else** — the ordinary tool-use loop, which needs no execution
  sandbox because it executes nothing: it can only pick from a fixed catalogue.

### Security posture

- JWT in an `httpOnly` cookie — nothing in `localStorage`
- Double-submit CSRF cookie on every state-changing request
- **Email 2FA on by default** (opt-out), TOTP opt-in, one-time recovery codes
- All ten integration credentials Fernet-encrypted at rest
- Workspace-scoped, short-lived tokens minted per individual tool call
- Route protection enforced server-side in Edge middleware before a protected
  page is ever rendered

The tradeoff of giving agents write tools is documented rather than hidden:
while the loop was read-only, "no tool can act" *was* the containment argument
against indirect prompt injection. What replaces it — untrusted-content
fencing, a prompt rule that write targets come only from the directive,
workspace-scoped tokens, connected-providers-only — is narrower, and the
codebase says so at the call site.

---

## Integrations

Slack · GitHub · Notion · Google Workspace (Gmail/Calendar/Drive/Docs/Sheets) ·
Discord · Asana · Salesforce · Ayrshare · Apify · LLM provider (BYOK)

Each one is OAuth or API-key based, encrypted at rest, and only ever exposed to
agents in workspaces that explicitly connected it. Bring-your-own model keys
act as an **overflow** key — they take over only once your own quota is
exhausted, and that work is billed by that provider directly to you.

---

## Billing model

The unit is the **Operation Unit (OU)** — one atomic agent task: a scrape, an
API call, a draft written, a supervision audit. Quota is per-user and enforced
atomically, so a mission always spends its *creator's* budget, never a pooled
one.

| | Starter | Scale | Enterprise |
|---|---|---|---|
| OUs / month | 15M | 100M | Custom |
| Concurrent missions | 5 | 25 | Unlimited |
| Model tiers | Low + Mid | + High (frontier) | Custom routing |

Measured on this platform, a real mission — one goal, four to six agents, live
research plus supervision on every output — runs about **1.3M OUs end to end**.

---

## Observability

Xorviex measures itself with the same seriousness it measures anything else:

- **User Activity Trail** — an append-only record of every page opened, button
  clicked and message sent, fed by two deliberately overlapping producers (a
  single delegated browser listener and a raw-ASGI middleware). A click with no
  request, and a request with no click, are both interesting. Superuser-only,
  browsed one user at a time — never a workspace-wide view, which would turn
  observability into surveillance between colleagues.
- **User Growth Analytics** — signups by day/week/month/year, cohort retention,
  activation funnel. A period still in progress is compared against the same
  *elapsed slice* of the previous one (otherwise every morning reads as a
  collapse), a percentage against a zero baseline is `None` rather than a
  fabricated rate, and signups are never shown without engagement beside them.
- **Blog Readership** — real dwell time and scroll depth per read, where every
  average carries its own `measured_views` denominator so the number reports
  reading behaviour rather than the ad-blocker rate.

---

## Status

Xorviex is a live, closed-source commercial platform running at
**[xorviex.ai](https://xorviex.ai)**. This repository is a public overview of
what it is and how it's built — the source itself is private.

---

<div align="center">

**Built by [Tunahan Faruk Savranoğlu](https://www.linkedin.com/in/tunahan-faruk-savrano%C4%9Flu-b71016370/)**
— founder, and the engineer behind every line of it.

[xorviex.ai](https://xorviex.ai) · [LinkedIn](https://www.linkedin.com/company/xorviex/) · [X](https://x.com/xorviex) · [GitHub](https://github.com/Xorviex-Labs)

</div>
