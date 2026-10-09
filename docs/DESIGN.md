# Personal Agent — Design v0 (draft for alignment)

> Status: **proposal**. Nothing is built yet. Open questions are at the bottom; answers to those
> change Milestones 1–2, not the core architecture.

## 1. What we're building

A personal agent you **text**. It holds a conversation, remembers you, wakes itself up on a schedule,
and **acts on your behalf** inside guardrails you control. The first job it gets is running your fantasy
basketball team for the season: draft support, daily lineups, injury swaps, waiver and streaming moves,
and weekly matchup plans.

Reference point: Meta's Muse (launched Sep 2026) is a messaging-native agent that runs tasks on its own
cloud computer. We take the same interaction model but own the stack: our data, our keys, our autonomy
policy, and any LLM behind it.

### Non-goals for v0
- Multi-user. This is one person's agent, and every design choice assumes one principal.
- Voice, a native app, or a web UI. Text is the interface; a dashboard can come later.
- Fully autonomous live drafting. No platform exposes a draft-pick API (see §6.3).

---

## 2. Design principles (the opinions everything else follows from)

1. **One agent = one serialized actor.** Every input is an event in one mailbox: your text, a cron tick, a
   webhook, an approval tap, or a wakeup the agent scheduled for itself. Events are processed one run at a
   time, so you can never race the agent. Example: you text "don't drop Sengun" while the waiver job runs,
   and your text lands in the next turn of the same thread. It doesn't get lost in a parallel process.
2. **The LLM decides; code computes and enforces.** Valuations, lineup optimization, roster legality and
   the autonomy policy are deterministic code exposed as tools. The model never does arithmetic that
   matters and never grades its own permission.
3. **Autonomy is a dial per action class, earned with a track record.** Reads are free. Reversible moves
   run automatically. Consequential moves need approval until the ledger shows the agent is good at them.
   The gate is enforced in the tool executor, not in the prompt.
4. **Ports and adapters at the three volatile edges:** Model (Claude → others), Channel
   (Telegram → iMessage/SMS), Platform (Yahoo/ESPN/Sleeper). The core never imports a vendor SDK.
5. **Append-only everything.** The event log is the source of truth, and memory and state are derived
   views. This is good for auditing, and current Claude models require it: editing earlier turns
   invalidates prompt cache and preserved thinking blocks.
6. **Skills are packages.** Each skill is tools + prompt + schedules + state schema. Fantasy NBA is
   skill #1, and the core knows nothing about basketball.

---

## 3. Architecture

```
                 ┌──────────────────────── Cloudflare ─────────────────────────────┐
 You ⇄ Telegram ─┼─▶ Worker (router, webhook auth) ──▶ AgentDO  (your agent: 1 actor)│
                 │       ▲            ▲                  │                           │
                 │  Cron triggers   OAuth callbacks      ├─ Mailbox ─▶ Runtime loop  │
                 │                                       │              │            │
                 │                                       │   ModelPort ◀┘──▶ Claude (Anthropic SDK)
                 │                                       │              │      (OpenAI/Gemini later)
                 │                                       │   ToolExecutor ─▶ PolicyGate ─▶ Skills
                 │                                       │                     core: memory, schedule,
                 │                                       │                           web, approvals
                 │                                       │                     fantasy-nba: engine +
                 │                                       │                           PlatformPort
                 │                                       ├─ SQLite (DO storage): events, thread,
                 │                                       │   memory, missions, approvals, ledger, usage
                 │                                       └─ Alarms: self-scheduled wakeups
                 └──────────────────────────────────────────────────────────────────┘
```

### 3.1 Runtime loop (one "run")
1. Pop the next event from the mailbox. Runs are mutex-guarded, because a Durable Object can interleave
   requests across `await`s and storage-level gates don't cover a long LLM call.
2. Build context. The order is chosen for prompt-cache stability:
   `tools → system (persona + policy summary) → profile memory → active mission → thread tail → event`.
3. Call `ModelPort.generate()`. Then loop: execute tool calls through the PolicyGate, append results,
   and call again until the model finishes. There are hard caps on steps per run, tokens per run and
   dollars per day.
4. Emit outbound messages via `ChannelPort`, then persist everything (append-only).
5. The model can call `schedule_wakeup(at, why)`. That call is how "check back after the 6:30pm injury
   report" turns into real autonomy.

### 3.2 Ports (the contracts that keep us agnostic)

```ts
// Model — normalized parts, with opaque passthrough for provider-native blocks
type Part =
  | { type: "text"; text: string }
  | { type: "tool_call"; id: string; name: string; input: unknown }
  | { type: "tool_result"; callId: string; output: unknown; isError?: boolean }
  | { type: "opaque"; provider: string; raw: unknown }; // e.g. Claude thinking blocks: replayed
                                                         // verbatim to the same model, dropped otherwise
interface ModelProvider {
  id: string;
  generate(req: { system: string; messages: Msg[]; tools: ToolSpec[];
                  effort?: "low" | "medium" | "high"; maxTokens?: number }): Promise<{
    parts: Part[]; stop: "end" | "tool_use" | "max_tokens" | "refusal"; usage: Usage }>;
}

// Channel
interface Channel {
  send(msg: { text: string; buttons?: Button[] }): Promise<void>;
  parseInbound(req: Request): Promise<InboundEvent | null>; // verifies signature + sender allowlist
}

// Tools carry their own risk class; the gate, not the model, decides what runs
interface Tool<I, O> {
  name: string; description: string; schema: JSONSchema;
  risk: "read" | "reversible" | "consequential" | "irreversible";
  preview?(input: I, ctx: Ctx): Promise<string>; // human-readable diff shown in approvals
  run(input: I, ctx: Ctx): Promise<O>;
}
```

The `opaque` part is the important detail. Provider-agnostic frameworks usually flatten messages to text
and lose provider-native state such as Claude's thinking blocks. Without that state the model reasons
worse on later turns and caching breaks. We keep the native blocks and replay them to the same provider,
and drop them only when switching providers.

### 3.3 Autonomy policy (config, not prompt)

| Risk class | Examples | Default mode | On approval timeout |
|---|---|---|---|
| `read` | rankings, roster, news, schedule | **auto** | n/a |
| `reversible` | set today's lineup, bench a scratched player, reorder draft queue | **auto + notify** (in digest) | n/a |
| `consequential` | add/drop, waiver claim, FAAB bid | **confirm** (tap ✅/❌ in Telegram) | per action: lineup-adjacent → execute; roster moves → skip |
| `irreversible` / social | trade propose/accept, messaging league mates, anything with money | **always confirm** | skip |

- Constraints can be expressed per tool, e.g. `faab_bid: auto if bid <= 5`.
- **Graduation:** the decision ledger records every proposal, your verdict and the outcome. When a class
  has a good record (e.g., 20 streaming proposals approved unchanged), the agent can ask to promote it to
  `auto`. You flip the dial; it never promotes itself.
- **Kill switch:** `/pause` halts all writes instantly. `/resume` restores them.
- **Idempotency:** every write carries a ledger key (`action + args hash + date`), so retries can never
  double-drop a player.

### 3.4 Memory

| Layer | Contents | How it's used |
|---|---|---|
| Event log | every inbound/outbound message, tool call, decision | audit, debugging, eval data |
| Thread | conversation turns (append-only) | recent tail in context; server-side compaction when long |
| Profile | durable facts and preferences ("punting FT%", "never drop Wemby", "texts after 11pm are low priority") | always in system prompt; agent edits via `remember()` / `forget()`; you can view it with `/memory` |
| Missions | standing goals with their own notes and schedule ("Win league X: H2H 9-cat, punt FT%") | injected when a run belongs to the mission |
| Domain state | typed tables: league, roster snapshots, valuations, schedule | queried by tools, never dumped into the prompt |

No vector database in v0. A personal agent's profile and missions fit in context for a long time. We
add retrieval when measured context pressure says to, not before.

### 3.5 Security
- **Telegram webhook** is verified with the secret-token header, and the **sender is allowlisted to
  your chat ID**. Without the allowlist, anyone who found the bot could command it.
- **Credentials live in Worker secrets**, reachable only from inside tools. They never appear in prompts.
- **Untrusted content** (news, web pages, league chat, player blurbs) is tagged as data. A prompt
  injection can at worst cause a *proposal* you see, because consequential writes pass the code-level gate.
- **Budget caps:** daily LLM spend ceiling plus per-run step and token limits. `/cost` shows spend.

---

## 4. Stack decisions

| Decision | Choice | Why | Tradeoff / revisit when |
|---|---|---|---|
| Language | **TypeScript** | Best fit for the edge runtime, and the Anthropic, OpenAI and Google SDKs are all first-class. Tool schemas and types line up. | Python has better fantasy and NBA libraries (`espn-api`, `nba_api`). We write thin HTTP clients instead, since these APIs are simple JSON. |
| Hosting | **Cloudflare Workers + one Durable Object** (via the `agents` SDK for state, SQL and scheduling) | Webhooks, cron, durable per-agent state, alarms for self-scheduling, and **deploy from GitHub with no laptop**. Free or $5/mo. A single-threaded actor matches principle 1. | Not Node (use `nodejs_compat`). Browser automation goes through Cloudflare Browser Rendering. Revisit if we need long CPU-heavy jobs; a container on Fly or Railway would be the escape hatch. |
| Durable workflows | **Not in v0.** Approvals are rows + alarms inside the DO. | Fewer moving parts. | Add Cloudflare Workflows if multi-hour, multi-step jobs show up. |
| LLM layer | **Own a thin `ModelProvider` port**, Anthropic SDK in the adapter | Keeps provider-native features (caching, thinking replay, refusal fallbacks) and avoids framework churn. A second provider is one adapter file, about 200 lines. | We maintain the adapters. Revisit only if we need more than three providers. |
| Model | `claude-opus-5-5` everywhere, **effort tuned per route** (`low` for chat/acks, `high` for draft/trade/waiver decisions). `claude-haiku-5-5` only for bulk extraction such as news parsing, if needed. | One model gives one prompt-cache namespace. Lowering effort on the strongest model usually beats a multi-model cascade. | Measure cost per completed task after two weeks of real traffic. |
| Channel | **Telegram first** | Free, set up in 10 minutes from your phone, inline buttons for one-tap approvals, and unrestricted proactive messages. Proactive messages are the core of an autonomous agent. | It isn't iMessage. **iMessage** goes through third parties (Sendblue, LoopMessage, etc., roughly $30–100/mo, no official Apple API) and is a later adapter. **SMS** needs A2P 10DLC or toll-free verification (days to weeks of carrier paperwork). **WhatsApp** limits proactive messages outside a 24-hour window to pre-approved templates, which is bad for an agent that pings you first. |
| Repo | Build in **`agent/`** in this repo, leave `index.html` untouched | Works within this session's access today. | Extract to its own repo with `git subtree split` whenever you like; cost is near zero. |
| Tool protocol | In-process tools with JSON-Schema specs | Simplest. | Wrap skills as an **MCP server** in Horizon 2, so the same fantasy tools also work from the Claude app or Claude Code. |

**Rough running cost (estimate, verify after week 1):** Cloudflare $0–5/mo, Telegram $0. Claude at
moderate personal use (~50 conversational turns/day plus daily jobs) with prompt caching is roughly
$1–2/day on Opus 5.5. A configurable daily cap (default $3) bounds the downside.

---

## 5. Phone-only development workflow

- **Build:** Claude Code cloud sessions (like this one) write code, run typecheck, tests and a local
  Worker (Miniflare/vitest) in the container, then push to a branch and open a PR.
- **Review and ship:** you review and merge from GitHub mobile. GitHub Actions runs CI on PRs and
  `wrangler deploy` on merge to `main`, using repo secrets.
- **One-time setup on your phone (about 20 minutes):**
  1. Cloudflare account → API token ("Edit Workers" template) → GitHub repo secret `CLOUDFLARE_API_TOKEN`
     (+ `CLOUDFLARE_ACCOUNT_ID`).
  2. Telegram → @BotFather → `/newbot` → token → secret `TELEGRAM_BOT_TOKEN`.
  3. Anthropic Console → API key → secret `ANTHROPIC_API_KEY`.
  4. Fantasy platform credentials: the Yahoo OAuth flow runs entirely in the phone browser through
     a `/connect/yahoo` route on the agent. ESPN's `espn_s2`/`SWID` cookies are hard to extract on a
     phone and may need a desktop browser once.
- **Operate from Telegram:** `/status`, `/pause`, `/resume`, `/cost`, `/log [n]`, `/memory`,
  `/policy`. Logs also go to the event log, so debugging doesn't need a dashboard.

---

## 6. Skill #1 — Fantasy NBA

### 6.1 Platform reality (this is what bounds "autonomous")

| Platform | Read | Write (lineup / add-drop / waivers / trades) | Draft picks via API |
|---|---|---|---|
| **Yahoo** | official OAuth2 API | **Official**: roster `PUT` by date (NBA), transactions `POST` for add, drop, add/drop, waiver claims with FAAB, trade proposals | no |
| **ESPN** | unofficial JSON API (cookie auth for private leagues) | reverse-engineered endpoints only; brittle and a ToS gray area | no |
| **Sleeper** | official public API, **read-only** | none (private app API only) | no |
| Fantrax / others | limited | none practical | no |

Implication: **on Yahoo, full autonomy is legitimate.** On ESPN it's achievable but fragile. On Sleeper
the agent is an advisor that tells you exactly what to tap. Browser automation is a possible later
fallback.

### 6.2 Engine (deterministic code, exposed as tools)
- **Player model:** per-game projections (platform projections, blended with last season's per-game
  numbers and minutes context) → per-category **z-scores** tuned to *your league's* categories. Field
  goal and free throw percentages are volume-weighted. Values are recomputed for punt builds.
- **Draft recommender:** value over replacement, plus your roster's category and position needs, plus
  ADP (who's likely gone before your next pick), plus run detection (e.g., "4 centers went in the last
  6 picks").
- **Daily lineup optimizer:** who plays today × eligible slots × injury status → a maximum-games
  assignment (bipartite matching), including IL/IR slot handling.
- **Streaming planner:** games remaining in the matchup week × categories still winnable this week →
  pickup candidates.
- **Matchup projector:** expected category totals for both teams → which categories to chase and which
  to concede this week.
- **Trade evaluator:** category-delta for both sides, schedule-adjusted (Horizon 2).

### 6.3 Modes
- **Pre-draft:** a league-tuned big board, punt-strategy simulations, and an autodraft queue exported as
  a fallback in case you miss the draft.
- **Live draft (co-pilot):** the agent polls draft state every ~10s. When you're about two picks out it
  texts you its top 3 with one-line reasons, and after each pick it re-plans your category build. You
  make the pick in the app (no platform supports API picks). A browser-automation drafter is a Horizon 2
  experiment, not something to bet a draft on.
- **Season autopilot (schedules in your local time):**
  - ~10:00 daily: set today's lineup (`reversible` → auto) and flag questionable players.
  - About 30 minutes before the first tip, and again before late slates: re-check injury reports and
    late scratches, then swap (auto).
  - 09:00 daily brief: yesterday's results, matchup state, today's moves, pending proposals.
  - Daily waiver/streaming scan → proposals with ✅/❌ buttons (`consequential` → confirm).
  - Monday: weekly matchup plan (categories to chase or concede, number of streaming slots).

### 6.4 Data sources
The platform API (rosters, free agents, projections, transactions) plus ESPN's public site API for NBA
schedule, injuries and news. We avoid `stats.nba.com`, which routinely blocks cloud IPs.

---

## 7. Milestones — two horizons

### Horizon 1: ship before tip-off (Oct 20, 2026). The draft date is the critical path.

| # | Milestone | Scope | Exit criteria |
|---|---|---|---|
| **M0** | Foundation (days 1–2) | `agent/` scaffold, CI+deploy Action, Telegram channel (allowlist + webhook auth), AgentDO mailbox + runtime loop, Anthropic adapter, event log, profile memory, `schedule_wakeup`, PolicyGate skeleton, `/status /pause /cost` | You text it from your phone; it answers, remembers a preference tomorrow, and pings you at a time you asked for. |
| **M1** | Fantasy read + engine (days 2–4) | Platform client (read), league settings ingest, player pool + projections, z-score valuation tuned to your categories and punt, ranked board | "Give me my top 60 for this league, punting FT%" returns a numerically defensible board. |
| **M2** | Draft co-pilot (before your draft) | Live draft polling, on-the-clock texts, post-pick re-planning, autodraft queue export | You draft from your phone with it in a mock draft, then the real one. |
| **M3** | Season autopilot v1 (by Oct 20) | Lineup optimizer + auto-set, injury watch + swaps, morning brief, waiver/stream proposals with approval buttons, decision ledger | A week of games managed with ≤1 tap per day from you. |

If your draft is within four days, M1 shrinks to "board only" and M2 ships right after M0.

### Horizon 2: from fantasy bot to personal agent

| # | Milestone | Scope |
|---|---|---|
| **M4** | Trust & quality | Outcome tracking on every decision (did the recommendation beat the alternative?), policy graduation flow, an eval set built from real decisions, cost-per-task dashboard, and a **second model adapter** to prove the LLM port is really agnostic |
| **M5** | General agent | New skills (calendar, email triage, reminders, web research, purchases with always-confirm), an MCP *client* to plug in third-party tool servers, your skills exposed as an MCP *server*, iMessage channel adapter, retrieval memory if needed, and a web dashboard (the Draft HQ page in this repo is a natural shell) |

---

## 8. Proposed layout

```
agent/
  src/
    core/            runtime loop, mailbox, ports (model/channel/tool), policy gate, memory, ledger
    providers/       anthropic.ts            (openai.ts later)
    channels/        telegram.ts             (sendblue.ts / imessage later)
    skills/
      core/          remember, forget, schedule_wakeup, list_approvals, web_fetch
      fantasy-nba/   engine/ (valuation, lineup, streaming, draft), platforms/ (yahoo|espn|sleeper),
                     tools.ts, prompt.md, schedules.ts
    platform/cloudflare/   worker entry, AgentDO, cron, secrets
  test/              unit (engine is pure → heavy tests), integration (Miniflare), recorded fixtures
  wrangler.toml
.github/workflows/   ci.yml, deploy.yml
docs/DESIGN.md
```

---

## 9. Open questions (need your answers to lock M1–M2)

1. **Platform:** which one is your league on (Yahoo / ESPN / Sleeper / other)? Is that fixed, or could
   you choose? Yahoo is the only one where full autonomy is officially supported.
2. **League format:** H2H 9-cat / H2H points / roto? Number of teams? Snake or auction? Lineup type
   (daily vs weekly), and any IL slots?
3. **Draft date and time** (and whether a mock draft is possible before it).
4. **Channel:** is Telegram OK for v0, or is iMessage a hard requirement now (third-party, ~$30–100/mo)?
5. **Budget:** OK with a ~$3/day LLM cap and Cloudflare's $5/mo plan if the free tier runs out?
