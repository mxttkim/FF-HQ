# Personal Agent — Design v1 (for alignment before kickoff)

> Status: **proposal v1**. This replaces v0. Changes since v0: iMessage instead of Telegram, the scope
> is now a whole-life agent with fantasy as the first proving ground, the runtime moves from Cloudflare
> Workers to a container + Postgres, and an auction Draft Lab is added. Open questions are at the bottom.

## 1. Product

One iMessage thread between you and your agent. The agent runs facets of your life end to end:
fantasy team, calendar, email, expenses, bookings and purchases. **You make the final decisions.** It
texts you only when something needs you, and it gets more autonomous only where you explicitly let it.

The first proving ground is your **Yahoo NBA league: 12-team, H2H 9-cat, auction draft**. The goal there
is for you to stay in the league and watch games while the agent manages the team.

### Interaction contract
- **Everything happens over text.** Proposals, approvals, briefs, and "what did you do today?".
- **Every proposal answers four questions:** what, why, what it costs, and how to say yes or no. Each also
  states what happens if you don't answer.
- **You approve rules, not just actions.** A *standing order* ("auto-stream adds when the gain clears the
  bar, max 2 per week"; "reorder coffee beans under $25 when I say 'coffee'") turns a stream of
  approvals into one.
- **When the agent gets stuck** (login, CAPTCHA, 2FA) it sends a **handoff link**: a live view of its
  browser that you finish in one tap on your phone. Then it continues.

---

## 2. Principles

1. **One agent = one serialized actor.** Your texts, scheduled ticks, task completions and approval
   replies all enter one mailbox and are processed one at a time, so you never race your own agent.
2. **Conversation never blocks on long work.** Mock drafts, browser bookings and research run as
   **background tasks**. The agent spawns them, keeps talking to you, and gets their results back as
   events.
3. **The LLM decides; code computes and enforces.** Valuations, simulations, lineup optimization, budget
   math and the autonomy gate are deterministic code. The model never authorizes itself.
4. **Autonomy is a per-action-class dial, earned with a track record**, and standing orders are the
   unit of trust.
5. **Money moves only with minted credentials.** The agent never holds your real card. Your approval
   creates a **single-use virtual card** locked to that merchant and amount.
6. **Delegated, least-privilege access.** We use OAuth scopes where they exist and you log in through
   handoff links. The agent never sees a password, and every integration is revocable on its own.
7. **Ports and adapters at the volatile edges:** model, channel, platforms and the browser. The core
   never imports a vendor SDK, which keeps the LLM swappable.
8. **Append-only event log.** It is the source of truth for audits, debugging and evals. It also keeps
   prompt caching and Claude's preserved thinking intact.

---

## 3. Architecture

```
 You (iMessage) ⇄ iMessage provider (Sendblue) ──webhook──▶ ┌────────────── API service (Node/TS) ──────────────┐
                                                            │ verify signature + sender allowlist → Mailbox     │
 Yahoo / Google / Plaid OAuth callbacks ───────────────────▶│ OAuth callbacks, handoff + live price-sheet pages │
                                                            └───────────────────────┬──────────────────────────┘
                                                                                    ▼
 ┌──────────────────────────────── Worker (same process in v0) ─────────────────────────────────────────────┐
 │  agent-run queue (concurrency 1) ─▶ Runtime loop ─▶ ModelPort ─▶ Claude (Opus 5.5)   [OpenAI/Gemini later] │
 │                                         │                                                                   │
 │                                         ├─▶ ToolExecutor ─▶ PolicyGate ─▶ Skills (fantasy, calendar, email, │
 │                                         │                                 money, bookings, purchases, core) │
 │                                         └─▶ spawn_task ─▶ task queue (parallel) ─▶ Draft Lab · Browser agent │
 │                                                                     │                 · Research agent     │
 │  scheduler: cron + delayed jobs (wakeups, approval timeouts) ◀──────┴── results come back as mailbox events│
 └───────────────────────────────────────────────────┬───────────────────────────────────────────────────────┘
                                                     ▼
            Postgres: events · thread · profile memory · missions & standing orders · approvals · action ledger
                      · credentials (encrypted) · usage/cost · domain tables (fantasy, transactions, …)
```

- **Runtime loop.** Pop an event, then build context in a cache-stable order (tools → system → profile
  → mission → thread tail → event). Call the model, run tools through the gate, loop until done, reply
  and persist. There are hard caps on steps, tokens and dollars per day.
- **Ports.** `ModelProvider` uses normalized message parts plus an `opaque` passthrough, so Claude's
  thinking blocks replay to the same model and are dropped on a provider switch. The other ports are
  `Channel` (send, react, typing, inbound parse), `Tool` (schema, risk class, `preview()` for approval
  text, `run()`) and `BrowserAgent` (task, constraints → result or handoff).
- **Memory.** Profile facts and preferences (always in the prompt, editable over text with
  `/memory`). Missions and standing orders (scoped context plus pre-approved bounds). Typed domain
  tables that tools query. The full event log. No vector database until measured need.

---

## 4. Stack decisions

| Decision | Choice | Why | Tradeoff |
|---|---|---|---|
| Language | **TypeScript (Node 22), strict** + zod schemas | You won't read the code and an AI writes it across many sessions, so **the compiler is the code reviewer**. One language covers server, tools, browser automation (Playwright is TS-native) and web views. Every SDK on the roadmap is first-class: Anthropic/OpenAI/Google, MCP, Google APIs, Plaid, Privacy, Browserbase, Sendblue. | Python has nicer data tooling, but our simulations are simple numeric loops that run fast in V8. |
| Runtime | **Railway**: one Node service + managed Postgres. GitHub auto-deploy; dashboard works on a phone. | The life-agent scope needs long-running tasks (browser sessions, thousands of simulated auctions), many Node SDKs, and relational data (ledger, transactions). Workers' runtime limits would keep biting. **This reverses my v0 Cloudflare pick because the scope changed.** | A bit more ops than serverless. Durable jobs in Postgres make restarts harmless. |
| Durable jobs | **pg-boss** (Postgres-backed queue: cron, delays, retries, singleton keys) | Schedules, self-wakeups, approval timeouts and background tasks with no extra infrastructure. `concurrency: 1` on `agent-run` gives us the serialized actor. | Swap in Temporal/Inngest only if workflows get much more complex. |
| Channel | **iMessage via Sendblue, AI Agent plan** (~$100/mo, dedicated number) | Dedicated line, webhooks, typing indicators, tapbacks, media and SMS/RCS fallback, plus an official TS SDK. It's the most agent-ready option. | Third-party (no official Apple API). Cheaper options: Blooio from $39, LoopMessage from ~$60, Photon has a free individual tier. **Day-1 gate:** confirm the line can message *you* unprompted in our existing thread. Sendblue lists "inbound-first", which should cover a contact who texted first, but we verify. If it fails, we switch providers behind the `Channel` port in about an hour. |
| LLM | `claude-opus-5-5` with **effort per route** (low for chat/acks, high for draft/trade/purchase decisions). `claude-haiku-5-5` for bulk email triage. Server-side refusal fallback on. | One strong model at varied effort usually beats a cascade, and it keeps one cache namespace. | Re-measure cost per completed task after two weeks. |
| Browser agent | **Browserbase** (persistent logged-in contexts + live-view handoff links) driven by a Claude computer-use sub-agent behind `BrowserAgent` | Persistent contexts mean you log in once from your phone. Live view is the handoff UX. | Browser automation is the least reliable part of any agent today. We design for graceful handoff, not 100% completion. |
| Payments | **Privacy.com API** (Plus plan or above): single-use, merchant-locked card minted per approved purchase | Your approval is the credential. A card can't be reused, and a merchant can't overcharge. | Watch Mastercard Agent Pay / Visa Intelligent Commerce (rolling out via issuers in 2026) as a later upgrade. |
| Bank data | **Plaid** (or **SimpleFIN Bridge** if Plaid's personal access is gated) + receipts parsed from email | Plaid: near-real-time, webhooks, broad coverage. SimpleFIN: about $15/yr and built for individuals, but syncs only daily. | We decide at that milestone, based on which of your banks each supports. |
| Repo | **New private repo** (e.g. `mxttkim/agent`) | FF-HQ is **public**. A life agent will accumulate personal policy, config and fixtures. | None worth having. This doc moves there at kickoff. |
| Rejected | Anthropic Managed Agents | Excellent hosted loop, but Claude-only, and you want LLM-agnostic. | — |

---

## 5. Human-in-the-loop over iMessage

| Risk class | Examples | Default | Approval form |
|---|---|---|---|
| `read` | rankings, inbox triage, transactions, availability | auto | — |
| `reversible` | daily lineup, injury swap, blocking time on *your* calendar, drafting an email | auto + in digest | — |
| `consequential` | add/drop, FAAB bid, booking a table, inviting someone, sending an email as you | confirm | `Y` / `N` (or 👍/👎 tapback) on the proposal; a numbered reply when there are several |
| `money / irreversible` | purchases, paid bookings, trades | always confirm | **echo the amount**: "Reply `BUY 84` to confirm $84.00". Prevents a fat-finger "yes" to the wrong thing. |

- **Timeouts are explicit per proposal** ("auto-skips at 7pm"). The default on silence is "don't", except
  for time-critical reversible actions such as a pre-tip injury swap.
- **Money and irreversible approvals are accepted only over iMessage from your allowlisted handle, never
  over the SMS fallback**, because SMS sender IDs can be spoofed.
- **Untrusted content never authorizes anything.** That covers emails, web pages and league chat. Email
  is the number-one prompt-injection vector for life agents. Actions whose arguments came from untrusted
  content are flagged in the proposal.
- **Standing orders** are stored rules with bounds (amount, frequency, merchant, category). Anything
  outside the bounds falls back to confirm.
- **Kill switch and visibility:** `/pause`, `/resume`, `/log`, `/cost`, `/memory`, `/orders`, plus a
  weekly "what I did" recap generated from the ledger.

---

## 6. Capability map

| Domain | What it does | Integration | Default autonomy | When |
|---|---|---|---|---|
| Core | reminders, follow-ups, notes, self-scheduled check-ins | internal | auto | M0 |
| **Fantasy NBA** | Draft Lab, auction co-pilot, daily lineups, injury swaps, streaming/FAAB, matchup plans | Yahoo Fantasy API (official read/write) | lineups auto; roster moves confirm → standing orders | H1 |
| Calendar | find times, schedule, reschedule, protect focus, travel buffers | Google Calendar (or iCloud CalDAV) | own events auto; invites confirm | M5 |
| Email | triage, summaries, draft-first replies, extract bills/receipts/confirmations | Gmail (or Outlook) | read auto; send always confirm | M5 |
| Expenses | categorize, budgets, subscription audit, anomaly alerts, weekly money brief | Plaid / SimpleFIN + email receipts | read-only, auto | M6 |
| Bookings | restaurants, appointments, travel research and holds | browser agent (+ flight APIs later) | confirm; handoff when blocked | M7 |
| Purchases | research, compare, checkout | browser agent + Privacy single-use card | always confirm (amount echo); standing orders for repeat buys | M8 |

---

## 7. Fantasy: Yahoo, 12-team, H2H 9-cat, auction

### 7.1 Data
- **Yahoo API (OAuth, you authorize once from your phone):** league settings (budget, roster slots,
  categories, lineup and waiver rules), players, rosters, live `draftresults`, and per-player
  `draft_analysis` (documented as average pick, round and % drafted; we check whether average auction
  cost is included on first connect). Writes cover date-based lineups, add/drop, waivers with FAAB, and
  trades.
- **Projections:** Yahoo doesn't expose player projections, so we blend public projections with last
  season's per-game numbers and minutes context. A paid projection source is optional later.
- **Market prices:** Yahoo average auction cost if exposed. Otherwise an ADP → $ curve fit to typical
  12-team auction spend.
- **NBA 2026-27 schedule:** games per matchup week, including playoff-week density.

### 7.2 Valuation
Per-category z-scores tuned to your nine categories, with FG% and FT% volume-weighted and TO negative.
Punt-aware variants are scored too. Value over replacement for the 12 × roster pool is converted to
dollars over the league's total budget, and **market delta** (our $ − market $) flags targets and fades.

### 7.3 Draft Lab (the mock drafts)
Mock auctions + season simulations to tune the strategy before the real thing:
1. **Auction simulator.** Eleven bot managers bid around market price with noise, budget pressure,
   roster need and archetypes (balanced, stars-and-scrubs, punters). It models nomination order, $1
   minimums and roster constraints, and is calibrated so bot spending matches the market.
2. **Season simulator.** Weekly H2H across all nine categories, using projected per-game stats × real
   games per week × availability draws (injury/rest) + variance. It runs daily-lineup approximations,
   standings and playoffs.
3. **Strategy search.** Our bidder is parameterized by punt set, budget split between stars and depth,
   max-overpay tolerance, inflation response and nomination policy. Each variant runs thousands of
   auctions × season simulations and is scored on **P(win league)** and P(playoffs), with robustness
   checks such as aggressive league-mates or a star inflation spiral.
4. **Outputs, texted to you:** a strategy memo (build, targets, fades, budget plan by tier), a **price
   sheet** (max bid per player plus fallbacks), and a nomination list.

The Draft Lab needs no credentials and no infrastructure, so it starts on day 1 in parallel with
everything else. Once Yahoo is connected it re-calibrates to your exact league settings.

### 7.4 Live auction co-pilot
Yahoo's API shows completed sales but not live nominations, so the loop is:
- **You text the nominated player's name** (e.g. "jjj"). A deterministic fast path (fuzzy name match +
  live price sheet, no LLM) replies **in ~2–3 seconds** with "Max $27 · fills BLK/3PM · you have $142
  for 9 spots."
- **After each sale** (polling draft results), it recomputes inflation, your category build and max
  bids, and texts only material changes ("Wemby went $71, inflation now +6%; raise Holmgren max to $31").
- **On your nomination turn** it suggests who to put up, usually players others overpay for, to drain
  budgets.
- **A live price-sheet page** auto-refreshes on the agent's server if you want to glance instead of text.
- **Rehearsal:** we run the co-pilot through Draft Lab simulations, and through a Yahoo mock auction if
  one is available.

### 7.5 Season autopilot ("you watch, it manages")
- Daily lineups (auto). Pre-tip injury and late-scratch swaps (auto). IL slot management.
- Daily streaming/FAAB scan → proposals, then a standing order once its record earns it.
- Monday matchup plan: categories to chase or concede, and streaming slots to use.
- Trade evaluation on request. Trades always need confirmation.
- One morning text with a quiet weekly recap. Silence when nothing needs you.

---

## 8. Milestones

### Horizon 1: the season (tip-off Tue Oct 20; your draft date is the critical path)

Three tracks run in parallel so nothing waits on account setup:

| Track | Milestone | Needs | Exit criteria |
|---|---|---|---|
| A | **Draft Lab v1**: valuation, auction simulator, season simulator, strategy search, price sheet | nothing (public data) | A strategy memo + price sheet backed by thousands of simulated auctions |
| B | **M0 Foundation**: Railway + Postgres, iMessage channel, mailbox/runtime, Claude adapter, memory, scheduler, approvals, ledger, `/pause /log /cost` | your ~30-minute setup | You text it; it remembers; it pings you unprompted; Y/N approvals work |
| C | **Yahoo connect**: OAuth, league settings, rosters, market data → recalibrate the Draft Lab | Track B + Yahoo app | Draft Lab rerun on your exact league |
| C | **Auction co-pilot**: fast-path lookups, post-sale updates, nomination tips, live sheet | above | Rehearsed end to end before your draft |
| C | **Season autopilot v1**: lineups, injury swaps, streaming proposals, matchup plan, brief | above | A week managed with ≤1 tap per day from you |

### Horizon 2: the life agent (after tip-off)

| # | Milestone | Scope |
|---|---|---|
| M5 | Calendar + email | triage, draft-first replies, scheduling, confirmations → calendar |
| M6 | Money | bank feed, categorization, budgets, subscription audit, weekly money brief |
| M7 | Browser agent + bookings | Browserbase contexts, handoff links, restaurant and appointment booking |
| M8 | Purchases | Privacy single-use cards, amount-echo approvals, standing orders for repeat buys |
| M9 | Trust and quality, ongoing | outcome tracking, standing-order graduation, evals built from the ledger, second LLM adapter, MCP server/client |

---

## 9. Monthly cost (estimates)

| Item | Horizon 1 | Full scope |
|---|---|---|
| iMessage line (Sendblue AI Agent) | $100 | $100 |
| Railway (service + Postgres) | $10–20 | $15–30 |
| Claude API (capped daily) | $20–50 | $40–90 |
| Browserbase | — | $20–99 |
| Privacy.com (API-eligible plan) | — | $5–25 |
| Plaid / SimpleFIN | — | ~$0–15 |
| **Total** | **≈ $130–170** | **≈ $180–360** |

---

## 10. Your setup (phone, about 30 minutes, needed for Track B)

1. **GitHub:** create the private repo, or let me create it, and add it to this session.
2. **Railway:** sign in with GitHub → new project from the repo → add Postgres. I'll give you the exact
   environment variables to paste.
3. **Anthropic Console:** API key, with a monthly spend limit set in the Console as a second safety net.
4. **Sendblue:** AI Agent plan → dedicated number → set the webhook URL I give you.
5. **Yahoo Developer:** create an app (Fantasy Sports: Read/Write) with the redirect URI I give you.
   Then you tap "Connect Yahoo" in a link the agent texts you.

Later milestones add Google OAuth (calendar/email), Plaid or SimpleFIN, Privacy.com and Browserbase.

---

## 11. Open questions

1. **Draft date and time** (and any keepers)? This sets the whole Horizon 1 schedule.
2. **Calendar and email:** Google, or Apple/Outlook?
3. **Repo:** OK to create a private repo `mxttkim/agent` (or another name)? FF-HQ is public, so the life
   agent shouldn't live there.
4. *(Optional)* A name for your agent. It becomes its contact name and persona.
