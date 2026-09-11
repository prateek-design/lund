# AI Interface — full-page AI agent section (replaces the dashboard side panel)

**Status:** proposed — plan approved for review, not yet implemented; amended 2026-09-09 after a repo-verified review (issues G1–G10 + blast-radius fixes folded in); amended again 2026-09-09 after the security & scalability review (SR1–SR12 below, risks R11–R14)
**Date:** 2026-09-09
**Basis:** repo at `feature/multi-cli` HEAD; AI SDK 7 facts verified from official docs 2026-09-09 (harness-adoption context inlined in Appendix A — its source files are gitignored)
**Architecture companion:** [`plans/ai-interface-architecture.md`](ai-interface-architecture.md) — system/turn/sub-agent/data-model diagrams and the design rationale in one place

---

## 1. What and why

Today the dashboard's AI assistant is a **side panel**: shell chrome mounted in `App.tsx` (deliberately not a route), backed by `@bot/agent` (:7250) — a ~1,600-line hand-rolled turn loop over Connectra with hand-rolled SSE, an ambient page-context capture, and a bridge (8 last-value channels + 4 handler seams) into the registry/cron markdown editors.

This plan **removes the side panel completely** and replaces it with a dedicated, full-page **"AI Interface"** sidebar section — a ChatGPT-web × Claude-Code hybrid:

- **Sessions** — new / resume / stop / rename / **archive (never delete)**; user-specific threads; AI-titled.
- **Streaming chat** with model switching over the **full Bifrost catalog** via Connectra (today's picker only reads the fallback chain).
- **Debug mode** — per model call: the exact request sent to the model (system prompt, messages, tools), thinking blocks, raw response, failure reason.
- **Observability** — a claude-code `/context`-style panel: context-window fill per model, token breakdown by category, cumulative thread tokens/cost.
- **Sub-agents** — Claude-Code-style delegation: the model decides from the prompt and the configured agent roster when to hand work to a sub-agent, spawns separate loops in parallel, and folds their reports back into the turn (§2.7, Phase 2).
- **Branching** — fork a conversation from any message; editing a user message = edit-and-fork (the message store is append-only by DB trigger, so edits are modeled as branches).
- MCP + connectors in chat, guardrails kept, `/` + `@` composer tokens kept.
- Later: background coding agents (HarnessAgent) that keep running after the browser closes.

### Settled decisions (user-confirmed 2026-09-09)

| # | Decision |
|---|---|
| D1 | Rebuild the chat backend on the **open-source Vercel AI SDK 7** (`ToolLoopAgent`) — no hand-rolled turn loop. |
| D2 | **Guardrails (`@bot/guardrails`) and Connectra stay** as the enforcement and model-gateway layers. Model calls never go direct to providers. |
| D3 | **Assisted editing is dropped entirely** — Ask-AI editor rewrites, `propose_edit` proposals, cron drafts, and the panel↔editor bridge all go. |
| D4 | **Page-context capture is removed entirely** (ambient route/form snapshots, `get_current_page`, `get_submitted_form_state`). |
| D5 | Threads **archive, never delete**. |
| D6 | Non-productionized: breaking changes and destructive drops are acceptable (each flagged below). Runs on the existing dedicated EC2. |
| D7 | **Phased, core first** (see §6). |
| D8 | `/` + `@` composer tokens and their pickers **stay** (explicitly requested). |
| D9 | Integration-gate test tier: **full**. |

> **Supersedes (in part):** the 2026-09-09 harness-adoption decision's "Lane B keeps `@bot/agent`'s hand-rolled loop" — the chat lane now rebuilds on AI SDK primitives. HarnessAgent remains the plan for the background-coding lane only (Phase 4) — the superseded decision and everything this plan relies on from it are reproduced in **Appendix A** (the original lives in a gitignored working directory, so this plan is the in-repo record).

### Grill amendments (user-confirmed 2026-09-09, second review)

| # | Decision |
|---|---|
| G1 | `agent_proposal` drop is **two commits**: commit 1 removes every proposal code path (table stays, unused), commit 2 carries the DROP migration — the migration-safety rule holds, no D6 exception needed. |
| G2 | `agent_model_call` **retention ships in Phase 1**: the central worker's daily purge sweep nulls `request`/`response` bodies older than `MODEL_CALL_RETENTION_DAYS` (default 30), keeping summary columns joinable to `ai_usage_event`. Supersedes R5's deferral. |
| G3 | Whole Phase 1 lands on **one feature branch**, merged once after the full integration gate — `main` never sees the panel-removed/page-absent gap between Chunks A and D. |
| G4 | Catalog-membership model validation is **stale-while-error**: a failed refresh serves the last good catalog indefinitely; never-fetched → 503 `service_not_ready` (preserving §4's named-model-never-silently-Auto invariant). |
| G5 | Context-window sizes come from **widening `@bot/shared`'s `pricing.ts`** to keep the LiteLLM table's `max_input_tokens`/`max_output_tokens` it currently discards (the table is already fetched/cached/daily-refreshed) — no static `MODEL_CONTEXT_WINDOWS` map; small static fallback only for unlisted models. Retires R6. |
| G6 | The context panel's fill bar shows fill against the **currently-picked model only** (Claude Code `/context` behavior). |
| G7 | `agent_model_call.run_id` is **nullable** (partial unique on `(run_id, seq)` where non-null) so run-less calls — thread titles — get honest debug rows. |
| G8 | Edit-as-fork stays **two client calls** (branch, then send); an orphan branch on mid-failure is accepted as harmless. |
| G9 | Cumulative tokens/cost = **own-thread runs only** (Claude Code `/cost` behavior: a fork's counter starts at zero); context-window *fill* is lineage-based regardless, since ancestors really are in the prompt. |
| G10 | `count_tokens` under Auto counts against the **first fallback-chain entry** (what Auto actually routes to), flagged `estimated: true`. |
| G11 | The AI Interface gets **Claude-Code-style sub-agents** (§2.7): the model delegates via a `delegate` tool whose schema carries the roster — **registry `agent` entries** (description → tool schema, body → sub-agent system prompt) plus one built-in `general-purpose` fallback. |
| G12 | Sub-agent execution v1 is **blocking-parallel**: delegate calls in one step run concurrently (cap 4), the step waits for all reports, **depth capped at 1**; the whole tree lives inside the turn. Background-across-turns work stays in the background-agents phase. |
| G13 | Sub-agent model = the registry entry's **optional model pin, else inherit** the thread's current pick (Auto included). |
| G14 | Sub-agents are **Phase 2**; branching/edit-as-fork slides to Phase 3 (migration 0071), background coding agents to Phase 4, connectors-in-chat to Phase 5. |

### Security & scalability review amendments (adopted 2026-09-09, third review)

Every SR below is repo-verified (file:line ground truth checked at review time) and folded
into the sections it touches; the scale-out roadmap the review produced lives in the
architecture companion (§9 there).

| # | Decision |
|---|---|
| SR1 | Thread-title input runs the deterministic `redact(['secrets'])` pass before the `generateText` call — today's title path (`thread-title.ts:155-163`, fired before `runTurn`) sends raw user text through **zero** guardrail gates, and the recorder would persist it; §2.4's "post-guardrail payload" claim must hold for every recorded call, title calls included. **Pulled forward to Phase 0**: the bypass is live in the current code, so the redact pass lands on `main` first and the Phase-1 port carries it. |
| SR2 | `agent_model_call.thread_id` is **NOT NULL FK RESTRICT** — the ownership anchor for rows with `run_id NULL` (title calls). Every read owner-joins through the thread (the `resolveOwnedRun` pattern; the repo finders are deliberately not owner-scoped). Without it, `GET /agent/model-calls/:publicId` is either unauthorizable for title rows or an IDOR. |
| SR3 | The `delegate` roster is **ownership-filtered** like `filterSkillPickerItems` (`agent/src/tools.ts:86-90`): org entries + the caller's own user-scoped entries, never other users' user-scoped entries — registry visibility is deliberately unfiltered (RG5), so "visible to the user" alone means *all* entries. |
| SR4 | Delegation **pins the registry entry version** at call time and writes an audit row naming entry + version — a later body edit never changes an in-flight or historical sub-run's provenance. |
| SR5 | Sub-runs get **no connectors token**: `conn:i` (which carries `tools_providers: '*'`) is minted only for the parent, lazily on first connector use. A registry `agent` body is developer-authored content executing under the invoking user's credentials — third-party SaaS actions don't ride along by default. |
| SR6 | **Per-turn sub-run budget: 8 total** (G12's cap 4 is per-*step* concurrency — 12 steps could otherwise spawn dozens); sub-runs (`parent_run_id IS NOT NULL`) are **excluded** from `PER_OWNER_CONCURRENT_RUN_CAP` (3), which would otherwise count children and let one delegating turn block the owner's other threads — or deadlock its own children. |
| SR7 | The usage endpoint **caches `count_tokens`** per (thread, last message seq, model), client-debounced — each uncached read is 3 fully-metered gateway calls (each writes an `ai_usage_event` + `ai_audit_log` row and accrues `requests: 1` against any `max_requests` cap). |
| SR8 | **Per-user rate limiting ships in Phase 1**, in-app keyed on `requireActor`'s user id — Kong's 120/min bucket on `/agent` is keyed on `x-client-id`, one machine client, i.e. a fleet ceiling every user shares. |
| SR9 | Migration 0069 also adds an **`(thread_id, id)` index on `agent_message`** (the fold's keyset query — `WHERE thread_id = $1 AND id < $2 ORDER BY id DESC` — has no covering index today, only `(thread_id, created_at)`), and the G2 retention step runs **batched** (LIMIT loops): UPDATE-nulling TOASTed jsonb leaves dead tuples for autovacuum on the same box, and the existing purge loop already materializes every deleted id. |
| SR10 | Infra hardening lands with Phase 1: Kong `agent-service` gets `read_timeout: 360000` (the one-liner `docs/modules/kong.md` already names; a **service**-level change — the route table and `kong-partition.test.ts` counts are untouched), the agent's `Bun.serve idleTimeout` rises above the 5-min wall clock (240 → 360; its own comment already claims it must), Connectra's `Bun.serve` gets an explicit `idleTimeout` (today it runs on Bun's default with no heartbeat on `/llm/*`), and the idle-SSE-past-60s integration test the Kong doc calls for is written. Message `text` is length-validated at the route (≤ 8192 bytes) instead of 500ing on `ck_agent_message_text_bytes`. |
| SR11 | The **`knownSecrets` vault-value denylist is wired** into the outbound scan seams — `@bot/guardrails` already reads `RuleContext.knownSecrets` but no caller anywhere supplies it, so only format-shaped secrets are ever caught; it is the compensating control for a debug store that persists request/response bodies. The 5432 loopback binding is **pulled forward to Phase 0** (it is live exposure today, unrelated to this feature). |
| SR12 | G5's widening must also **loosen `parseEntry`'s gate** in `pricing.ts`: it returns `null` for any LiteLLM entry missing either cost field, so a model listed with a context window but no price would vanish — keep window fields independently of priceability. |

### Verified ground facts this plan rests on

- `ai@7.0.74` is already pinned in `modules/agent` and `modules/dashboard` (types-only today). `ai@7.0.93` exports `ToolLoopAgent` (stable), `wrapLanguageModel`, `LanguageModelMiddleware`, `prepareStep`, `stopWhen`/`stepCountIs`, `dynamicTool`, `jsonSchema` — but **no MCP client** from the main entry.
- `@ai-sdk/anthropic` is not yet a dependency anywhere in the repo (new dep; compat is Risk R1).
- Latest migration is `0068`; this feature takes `0069` (+ `0070` for sub-agents in Phase 2, `0071` for branching in Phase 3).
- The Kong `/agent` route table is reused as-is — `kong-partition.test.ts` asserts exact route counts, and every new path lives under `/agent/*`. **No route-table changes**; the one Kong change is SR10's service-level `read_timeout` on `agent-service` (no route added or removed, so the partition test is untouched; recorded in `feature/kong.md` §15 + `docs/modules/kong.md` per keep-in-sync).
- Connectra's `POST /llm/anthropic/v1/messages` streams SSE byte-for-byte (`teeUsageStream`); `count_tokens` is proxied and currently unused; the Bifrost catalog (`GET /llm/api/connectors/catalog`) carries **no context-window metadata** — but `@bot/shared`'s `pricing.ts` already daily-refreshes the LiteLLM table that does (G5).
- **`@ai-sdk/react@4.0.77` hard-pins `ai: "7.0.74"` exactly** (with its own `@ai-sdk/provider` lineage) — bumping `ai` to 7.0.93 without a lockstep `@ai-sdk/react` bump forks two `ai` copies / two `LanguageModel` spec identities in the dashboard tree. `ai@7`'s zod peer (`^3.25.76 || ^4.1.8`) must also be satisfied once ajv leaves. Both are day-1 spike exit criteria (R1).
- `agent_message` kind `'reasoning'` is **already in `ck_agent_message_kind`** (no CHECK widening needed) but has **no production writer today** — the engine's reasoning rows are new wiring, not a port.
- The boot sweep is `AgentRepo.reclaimLostRuns` (called from `modules/agent/src/index.ts`, not `runs.ts`); it terminalizes `agent_run` only, so `agent_model_call` orphans need their own sweep extension.
- **MCP tool reachability from chat (verified; load-bearing for §2.7 and R13):** the delegated bearer **cannot** reach `get_secret`/`get_repo_access` — three independent layers: (a) the agent client's scopes lack `sec:rv`/`repo:rv`, (b) `@bot/mcp` registration-filters unscoped tools so they 404 like nonexistent ones, (c) `@bot/agent`'s `assembleToolset` drops `get_secret` by name and refuses any tool name outside its schema set. `list_secrets` (metadata only, never values) **is** reachable for `sec:r` holders. Sub-agents inherit exactly this posture, narrowed further by SR5 (no connectors token). **All three layers are deliberate — removing any one is a security regression, not a cleanup.**

---

## 2. Backend — rebuild `modules/agent` on AI SDK 7

### 2.1 What is replaced vs kept

**Replaced (deleted):**

- `turn.ts`'s round loop (frame parsing, tool dispatch, repeat detection, step management) → **`ToolLoopAgent`** in a new `engine.ts`.
- `connectra.ts`'s hand-rolled Anthropic SSE client → the **`@ai-sdk/anthropic` provider** (new dependency).
- `tool-schema.ts` (ajv) → SDK `jsonSchema()` + `dynamicTool()` input validation (ajv dep dropped).

**Kept — the persistence spine.** This is the re-answer to the two documented rejections of AI SDK server primitives:

- `runs.ts` unchanged: detached-run registry (a turn survives client disconnect), 15s heartbeats. (The boot sweep is `AgentRepo.reclaimLostRuns`, called from `index.ts` — it terminalizes `agent_run` only, hence step 3's separate `agent_model_call` sweep extension.)
- `stream.ts` unchanged: hand-rolled SSE (`createRunEventResponse`, `x-accel-buffering: no`), **run-addressed reconnect with replay from persisted rows**, the fold. `createUIMessageStreamResponse` stays rejected — it has no seam for a timer-driven heartbeat independent of chunks (DA2), and replay parity requires chunks derived from persisted rows, not a live SDK stream. The SDK is adopted for the **model-call/tool loop only**.
- `guardrail*.ts`, `delegation.ts`, `thread-title.ts` (ported to `generateText`), and `mcp-client.ts` — kept over an SDK MCP client because `ai@7`'s main entry has none, ours carries per-run delegated bearers per call, and it is already matched to `@bot/mcp`'s JSON-per-POST transport.

**New module layout (`modules/agent/src/`):**

```
provider.ts        — Connectra-backed LanguageModel factory        (new)
model-call-log.ts  — debug recorder middleware                     (new)
engine.ts          — ToolLoopAgent assembly + persistence pump     (replaces turn.ts)
tools.ts           — rewritten: MCP tools as dynamicTool()s; token/@-record resolution kept
connectra.ts       — shrinks to error classification + catalog/count_tokens REST helpers
runs.ts, stream.ts, guardrail*.ts, delegation.ts, mcp-client.ts, thread-title.ts — kept
app.ts             — new route table
```

### 2.2 Connectra as the model provider (`provider.ts`)

Per-run factory (tokens are per-run delegated bearers; the provider object is cheap):

```ts
createAnthropic({
  baseURL: `${CONNECTRA_PUBLIC_URL}/anthropic`,   // → POST .../anthropic/v1/messages
  headers: {
    authorization: `Bearer ${delegatedToken}`,
    'x-onexo-correlation-id': correlationId,
  },
  fetch: connectraFetchShim,
})
```

The Anthropic route (not openai-compatible) is chosen because debug mode needs thinking blocks with signatures end-to-end, and Connectra's metering already speaks that accumulator.

`connectraFetchShim` does what the provider can't:

1. Injects **`fallbacks: []`** when the user explicitly picked a model — preserving `feature/agent-model-picker.md` §4's "a picked model runs or fails visibly" semantics. Auto leaves the key absent so Connectra's fallback chain applies. (A **port** of `turn.ts`'s shipped `modelWasExplicit` behavior into the shim — forced because `@ai-sdk/anthropic` owns body serialization — not new semantics; the gateway side is already live-verified on `/llm/anthropic/*`.)
2. Maps non-2xx through `classifyConnectraError` to typed errors, so `stream.ts`'s existing `CONNECTRA_STOP_REASON_PATTERN` keeps working unmodified.

Model ids are always the full `provider/model` string (Connectra requirement). The model is wrapped: `wrapLanguageModel({ model, middleware: [modelCallRecorder] })` (§2.4). Thinking is requested via `providerOptions.anthropic.thinking` when the picked model supports it; deltas surface as SDK `reasoning` parts and persist as `kind:'reasoning'` rows.

### 2.3 The engine (`engine.ts`)

One `ToolLoopAgent` per run: `stopWhen: stepCountIs(12)`, `maxOutputTokens: 8192`, timeout total 5 min / chunk 30 s, `abortSignal` from the ActiveRun (`POST /stop` unchanged), `onStepFinish` → `incrementStepCount` + `agent_run_event`/audit rows. Thread history folds **once** at turn start (`foldMessages` → ModelMessages).

The **persistence pump** — a detached async function, exactly like today's `runTurn` — consumes `result.fullStream` and translates parts into persisted `agent_message` rows + published chunks:

- `text-delta` → batched `kind:'text'` rows through the existing mid-stream batch validator (optimistic release + retractions);
- `reasoning-delta` → `kind:'reasoning'` rows;
- `tool-call` / `tool-result` / `tool-error` → `kind:'tool_call'` rows with `AgentToolCallRecord`;
- `finish` → inbound whole-response gate → `terminalizeRun` + `publishTerminal`.

The HTTP response only ever *subscribes* to the run — nothing about reconnect/replay changes.

**The four guardrail points, in SDK terms (semantics unchanged):**

| Point | Today | New seam |
|---|---|---|
| 1. Input (whole-turn gate) | before round 1 in `turn.ts` | before agent construction in `engine.ts` |
| 2. Outbound per-part, memoized | per-round prompt rebuild | **`prepareStep`** — scan/redact system + message text; also filters spans retracted earlier in the same run (replaces the per-round re-fold). The recorder middleware sits *below*, so the debug table records the **post-guardrail** payload — exactly what was sent |
| 3. Mid-stream batch validator + retractions | `flushTextBatch` | the persistence pump (same `validateBatch`, same drain-before-replay) |
| 4. Inbound whole-response | after the round loop | after `fullStream` completes |
| (tool-result gate) | `callToolGated` | inside each `dynamicTool().execute` wrapper |

`GUARDRAIL_SCAN_CAP_BYTES === 8192 === ck_agent_message_text_bytes` stays untouched.

Points 1 and 4 are **`GUARDRAILS_SEMANTIC`-gated today** (2 and 3 run unconditionally) — "semantics unchanged" includes preserving that conditionality, not promoting them to always-on. Stated at its true strength: the flag **defaults to false**, so on a default install only the deterministic regex tiers run, and the deterministic secrets tier is format-shaped patterns only — an arbitrary secret *value* survives redaction unless it matches a pattern. SR11 (wiring the `knownSecrets` vault denylist, which the engine already reads but no caller supplies) is the compensating control this plan adds.

### 2.4 Schema — migration `0069_ai_interface.sql`

**New `agent_model_call` — the debug store.** One row per model call:

| Column | Notes |
|---|---|
| `public_id` | 16-hex, unique |
| `run_id`, `seq` | FK RESTRICT, **nullable** (G7 — a thread-title call has no `agent_run` row); partial `UNIQUE (run_id, seq) WHERE run_id IS NOT NULL` |
| `thread_id` | bigint **NOT NULL**, FK RESTRICT (SR2) — the ownership anchor; every read owner-joins through the thread (`resolveOwnedRun` pattern), so run-less title rows stay authorizable |
| `status` | `active` / `completed` / `failed` / `aborted` |
| `model`, `observed_model` | requested `provider/model`; what the response actually reported |
| `request` jsonb NOT NULL | **full post-guardrail payload**: system, messages, tools, params |
| `response` jsonb | raw content blocks incl. thinking + signature, stop_reason |
| `thinking` text | extracted reasoning text (UI convenience) |
| `error` text | classified failure reason |
| token cols, `latency_ms`, `correlation_id` NOT NULL | joins to `ai_usage_event` and Bifrost's log detail |

Insert-at-call-start / update-once-at-end, so a process death leaves an honest `active` orphan (boot sweep terminalizes it alongside `reclaimLostRuns`). Written by `model-call-log.ts`, a `LanguageModelMiddleware` in `wrapStream` — one seam, every call, thread-title calls included (possible because of G7's nullable `run_id`). **Always persisted; "debug mode" is purely a client display toggle.** Title calls are only recordable because SR1 redacts their input first — without it, the "post-guardrail payload" property fails on exactly the rows that have no run.

**Retention (G2, supersedes R5's deferral):** the central worker's daily purge sweep gains a step that nulls `request`/`response` bodies on rows older than `MODEL_CALL_RETENTION_DAYS` (default 30) — summary columns survive and stay joinable to `ai_usage_event`. The step runs **batched** (LIMIT loops, SR9), never one unbounded statement. 0069 also adds the `(thread_id, id)` index on `agent_message` (SR9) — the fold's keyset query has no covering index today.

**Drops and archive-only:**

- **`agent_proposal`: DROP** — sequenced as **two commits** (G1): commit 1 removes every proposal code path — `modules/agent` routes/tools, **`@bot/repo`'s `agent.repo.ts` methods (`createProposal`, `findProposalByPublicId`, `updateProposalDecisions`, `selectProposalOne`), `types.ts` proposal types, `repo/src/index.ts` exports**, the contracts types + **barrel re-exports in `contracts/src/index.ts`**, and the **`contracts.test.ts` assertions** (excerpt cap, `<<<REWRITE>>>` sentinel, the exact sixteen-route `/agent/*` map, the exact-array `AGENT_MESSAGE_KINDS`) — leaving the table present but unused; commit 2 carries the DROP migration. The migration-safety rule holds; D6 is not needed here.
- **`DELETE /agent/threads/:publicId`: removed** (+ `deleteThread`, `AgentThreadDeleteResponse`). Archive (`PATCH { archived: true }`) is the only removal verb. Scope `a:d` becomes route-less; no scope migration.
- `agent_message` kinds `proposal`/`cron_draft` are never written again: CHECK values stay, fold cases are deleted (wipe any dev rows — flagged).
- `agent_thread` / `agent_run` / `agent_message` / `agent_audit_log` / `agent_run_event` all survive.

### 2.5 Branching (Phase 3, migration `0071`) — lineage pointers, no row copies

```sql
ALTER TABLE agent_thread ADD COLUMN parent_thread_id bigint REFERENCES agent_thread(id) ON DELETE RESTRICT;
ALTER TABLE agent_thread ADD COLUMN branched_from_message_id bigint REFERENCES agent_message(id) ON DELETE RESTRICT;
-- CHECK: both null or both set
```

A branch's history = ancestor messages with `id <=` the branch point (recursive), then own rows — one new repo method `listMessagesForLineage(threadId)` feeding the existing fold. Rows are never copied, so `public_id`s and retraction `(run_id, seq)` ranges stay valid, and the append-only trigger remains the enforcement. **Retraction carve-out:** the lineage query must *also* include ancestor-thread `kind:'retraction'` rows written **after** the branch-point id — retractions are appended rows the fold uses to omit ranges, so cutting them off would re-send guardrail-retracted content in the branch; including them is always safe (they only ever narrow content).

- **Branch-from-message:** `POST /agent/threads/:publicId/branch { atMessagePublicId, mode: 'at' }` → new thread.
- **Edit-as-fork:** the client branches with `mode: 'before'` the edited user message, then sends the edited text as a normal message. No update ever touches the original row. (Deliberately two calls, G8 — a mid-failure orphan branch is a harmless thread the user can type into.)

Ownership is verified at branch time (all ancestors share one owner by construction). Archived ancestors stay readable — this is why archive-only matters here.

### 2.6 API changes (all under Kong `/agent`; contracts in `modules/contracts/src/agent.ts` + `ROUTE_SCOPES`)

**New routes:**

| Route | Scope | Purpose |
|---|---|---|
| `GET /agent/models` | `a:r` | Full Bifrost catalog via Connectra (`ai:r` client-credentials token — added to `AGENT_SCOPES` in `agent-client-seed.ts`, see §5), + `contextWindow` sourced from the **widened `@bot/shared` `pricing.ts`** (G5; static fallback only for unlisted models), + `deprecated`. Replaces `GET /agent/pickers/models`; message `model` validation switches to catalog membership — ~60 s in-process cache, **stale-while-error** (G4): a failed refresh serves the last good catalog, never-fetched → 503 `service_not_ready` (never a silent fall-through to Auto, per §4's invariant) |
| `GET /agent/runs/:publicId/calls` | `a:r` | Model-call summaries (seq, model, status, tokens, latency, error) — no bodies |
| `GET /agent/model-calls/:publicId` | `a:r` | Full debug detail: request, response, thinking, error, correlation id — owner-joined via `thread_id` (SR2) |
| `GET /agent/threads/:publicId/usage` | `a:r` | `AgentThreadUsage` (below) — `count_tokens` results cached per (thread, last message seq, model), client-debounced (SR7) |
| `POST /agent/threads/:publicId/branch` | `a:w` | Phase 3 |
| `GET /agent/runs/:publicId/children` | `a:r` | Phase 2 — sub-run summaries (agent name, status, model, tokens) for the task rows; drill-in reuses the existing run-addressed detail/trace/calls routes, which work per sub-run unchanged |

`AgentThreadUsage`: next-turn token **breakdown by category** computed at read time via the proxied `count_tokens` called three ways (system-only / system+tools / full; `estimated: true` bytes/4 fallback). Under Auto, counting targets the **first fallback-chain entry** — what Auto actually routes to — flagged `estimated: true` (G10). **Cumulative** tokens + cost = **this thread's own runs only** (G9, Claude Code `/cost` semantics — a branch's counter starts at zero; ancestors appear in *fill*, which is lineage-based, but never in *spend*), joined `agent_run.correlation_id → ai_usage_event.correlation_id` (read directly off `Repositories` — same DB); per-run rows. Window sizes come from the widened pricing table (G5).

**Removed routes:** proposals (GET/PATCH), thread DELETE. Pickers for skills/records/context/templates **stay** (D8).

**Removed contract types:** `AgentPageContext`, `AgentPageRoute`, `AgentSelectionExcerpt` (+cap), `AGENT_REWRITE_SENTINEL`, `AgentEditorContext`, `AgentProposal*`, `AgentThreadDeleteResponse`, `AgentMessageCreateRequest.{origin, pageContext}` (keep `tokens`, `idempotencyKey`, `text`, `model`), `AgentDataParts.{proposal, pageContext, cronDraft}` (keep `tokens`).

All routes stay owner-checked via `requireActor` (client-credentials callers 403) — user-specific by construction. A per-user in-app rate limiter keyed on `requireActor`'s user id fronts the mutating and heavy read routes (SR8) — Kong's 120/min `/agent` bucket is fleet-wide, not per-user.

### 2.7 Sub-agents (Phase 2, migration `0070`) — G11–G14

Claude-Code-style delegation inside the chat loop: the model decides, from the prompt and the
roster, when to hand work to a sub-agent; sub-agents run as separate model loops and only
their final reports re-enter the parent turn.

**The `delegate` tool.** One `dynamicTool()` on the parent engine. Its schema carries the
roster (G11): org-scoped registry `agent` entries plus the caller's **own** user-scoped ones
(SR3 — the `filterSkillPickerItems` rule; registry visibility is unfiltered by design, so
"visible" alone would mean every user's entries) — name + description in the tool's input
enum/description, refreshed per run through the same `RegistryReader` seam —
plus one built-in `general-purpose`. The model picks the agent and writes the task prompt;
that is the entire delegation trigger, exactly the Claude Code mechanism (roster in the tool
schema, not in the system prompt).

**A sub-agent is an `agent_run` row.** Migration `0070` adds
`agent_run.parent_run_id bigint REFERENCES agent_run(id) ON DELETE RESTRICT` (null = top-level).
The sub-run gets its own `ToolLoopAgent` with: system prompt = the registry entry's body
(`general-purpose` = a built-in constant), model = the entry's optional pin, else the thread's
current pick (G13), the same MCP toolset **minus `delegate` and the connectors tools** (depth
cap 1, G12; SR5 — no `conn:i` token is minted for a sub-run), fresh
per-sub-run delegated bearers via the existing `delegation.ts` mint, its own step cap and the
parent's remaining wall clock, and the parent's `abortSignal` chained in — **Stop cascades**:
terminalizing the parent terminalizes every child. The delegation pins the entry **version**
and writes an audit row naming entry + version (SR4). A turn spawns at most **8 sub-runs
total** (SR6 — G12's 4 is per-step concurrency), and `parent_run_id IS NOT NULL` rows are
excluded from `PER_OWNER_CONCURRENT_RUN_CAP` so a delegating turn neither blocks the owner's
other threads nor deadlocks its own children.

**Persistence composes for free.** Sub-run transcript rows are ordinary `agent_message` rows
under the sub-run's `run_id` (the column and `UNIQUE(run_id, seq)` already exist);
`foldMessages` for the **parent thread excludes rows whose run has a `parent_run_id`** — only
the delegate tool's result (the report) enters the parent history, as a normal `tool_call`
row. The recorder middleware wraps the sub-run's model too, so `agent_model_call` rows land
per sub-run; guardrail verdict rows carry the sub-run's `agent_run_id`; the boot sweep's
`status='active'` reclaim needs no change (sub-runs are runs). The run-addressed SSE replay
already works per run, so the drill-in transcript view is the existing reconnect path pointed
at a child run id.

**Guardrails: all four points run per sub-run loop** — input gate on the delegation prompt
(the model wrote it, but the registry body it executes under is user-authored content), the
`prepareStep` outbound scan, the mid-stream batch validator, and the inbound whole-response
gate on the report before it re-enters the parent. Same conditionality as §2.3. The trust
boundary, stated plainly: any `reg:w` holder (the **developer** role included, not admins
only) can author a body that other users' chats will execute under their own delegated
bearers — registry bodies are size-checked (64 KiB) but never sanitized at write time, so
SR3/SR4/SR5 are the containment, not the scanner (deterministic injection rules are regexes;
the semantic tier is default-off with unmeasured accuracy).

**Execution (G12):** delegate calls emitted in one assistant step run concurrently, capped at
4; the step waits for all reports (blocking-parallel). No cross-turn background tasks in this
phase — that is the background-agents phase's lifecycle.

**Usage:** sub-runs are thread runs, so G9's own-thread cumulative already counts them; the
context panel's fill is unaffected (sub-run rows are excluded from the parent fold, so they
never enter the parent prompt).

**UI:** each delegate call renders as a task row (agent name, live status, token count) in the
transcript's tool-call part; clicking opens the sub-run's transcript + debug inspector via
the run-addressed routes. `GET /agent/runs/:publicId/children` feeds the rows.

---

## 3. Frontend — `modules/dashboard`

### 3.1 Removal map

- **Delete:** `components/agent-panel.tsx`, `agent-panel-tabs.tsx`; `lib/agent-bridge.ts`, `lib/agent-page-context.ts` (+tests); the tab-order half of `lib/agent-tabs.ts`; storage keys `AGENT_PANEL_COLLAPSED/WIDTH`, `AGENT_TAB_ORDER`, `AGENT_ACTIVE_TAB`. In `App.tsx`: `AgentPanelToggle`, the panel state, the `setActivePageRoute` effect, the `<AgentPanel/>` mount. `nav-data.ts`'s `navSectionForHash` (App was its only consumer).
- **Assisted editing (D3):** delete `markdown-editor-selection-toolbar.tsx`, `markdown-editor-rewrite-session.tsx`, `agent-rewrite-card.tsx`, `agent-proposal-card.tsx`, `agent-cron-draft-card.tsx`; **also** `features/registry/{agent-field-proposals,agent-review-editor,agent-unresolved-proposal-dialog}.tsx` and `lib/markdown-editor-inline-ai-plugins.ts`; lib (+tests) `agent-proposal-review.ts`, `agent-proposal-base-body.ts`, `agent-cron-draft.ts`, `agent-rewrite-sentinel.ts`, `agent-excerpt-preview.ts`, `markdown-editor-selection.ts`, `markdown-rewrite-diff.ts` (each with its paired `*.test.ts`); strip bridge/proposal wiring from `features/registry/{registry-edit-panel,registry-create-page,registry-content-form}.tsx` (+ delete `use-agent-proposal.ts`) and `features/crons/{cron-create-page,cron-detail-page,cron-edit-panel}.tsx`; stop mounting the selection toolbar and `INLINE_AI_PLUGINS` in **`markdown-editor-impl.tsx`** (the actual consumer — not `markdown-editor(-toolbar).tsx`, which carries no assisted-editing imports); drop proposal calls from `agent-api.ts`.
- **Kept/moved files import deleted files — strip in the same change:** `lib/agent-message-parts.ts` (kept) imports `agent-rewrite-sentinel`; `agent-composer.tsx` and `agent-message.tsx` (moved) import `agent-excerpt-preview`; `agent-transcript.tsx` and `agent-message.tsx` import the deleted rewrite/proposal/cron-draft cards; `agent-thread-view.tsx` imports `classifyRewriteText`.
- ⚠️ **Bridge deletion and every consumer edit must land atomically** or `tsc` breaks repo-wide — the acceptance gate is a grep for **zero `agent-bridge` imports** (17 non-self importers today, not 16; the count is not the gate).

### 3.2 Move panel chat internals → `src/features/ai-interface/`

Per dashboard CLAUDE.md rule 2 (one-view compositions live in the feature dir): `agent-thread-view.tsx`→`chat-view.tsx` (strip all bridge/pageContext/rewrite wiring; origin always composer), `agent-composer.tsx`→`composer.tsx` (drop excerpt props; centered max-width), `agent-transcript.tsx`→`transcript.tsx`, `agent-message.tsx`→`message.tsx` (drop proposal/cron-draft branches), plus picker-popover, model/skills dropdowns, tool-call-part, empty-state; `agent-run-trace-drawer.tsx` grows into `turn-debug-inspector.tsx`. **`token-badge.tsx` stays in `src/components/`** — it's the shared picker-kind badge vocabulary imported by the kept `lib/agent-reply-badges.ts`, and lib→features is a layering inversion per dashboard CLAUDE.md convention 3.

**Kept in `src/lib/`** (edited): `agent-api.ts` (+ `fetchAgentModels`, `fetchAgentRunModelCalls`, `fetchAgentModelCallDetail`, `fetchAgentThreadUsage`), `agent-sse.ts`, `agent-transport.ts` (delete `capturePageContext`/`pageContext`/`origin`), `agent-stream-smoothing.ts`, `agent-thread-detail.ts`, `agent-retraction.ts`, `agent-message-parts.ts`, `agent-model-selection.ts`, `agent-reply-badges.ts` (token badges only), `mention-trigger.ts`, `caret-position.ts`. New `lib/ai-threads.ts` takes `DRAFT_THREAD_CREATE_NAME` / `DRAFT_NAME_PATTERN` (lockstep with `modules/agent/src/thread-title.ts`) / `displayThreadName` / `filterThreadsForWorkspace`, plus recency grouping and name search.

**Kept in `src/components/`** (cross-feature): `context-manifest-panel.tsx`, `token-cost-summary.tsx`, `tool-call-list.tsx`; promote `features/ai-gateway/json-tree-view.tsx` → `components/` (second consumer: the debug inspector).

Storage: rename const `AGENT_MODEL`→`AI_MODEL` **keeping the literal key value** (sticky model pick survives); add `AI_DEBUG`.

### 3.3 The new page

```
ai-interface-view.tsx        a:r gate, 2-pane full-bleed layout
├── thread-sidebar.tsx       New chat · client-side search · Recents/Archived toggle
│   └── thread-list-item     rename/archive menu, recency group headers; Sheet on mobile
├── chat-view.tsx            keyed by threadPublicId (full remount per thread)
│   ├── chat-header.tsx      title rename · model badge · debug Switch · "Context" button
│   ├── transcript.tsx       centered max-w-3xl, stick-to-bottom
│   │   └── message.tsx      tool-call parts · token badges · turn-debug-inspector
│   └── composer.tsx         sticky bottom · /+@ pickers · model dropdown · Stop
├── new-chat-landing.tsx     first send = createThread → hash replace → queued send
├── context-panel.tsx        the /context Sheet
└── turn-debug-inspector.tsx per-run inspector (grown from agent-run-trace-drawer)
```

Archived thread open = read-only banner + Unarchive. `App.tsx`'s `<main>` gets a conditional `overflow-hidden`/no-padding for `view === 'ai'` (the one shell-level special case — a class toggle, never a second layout branch).

### 3.4 Routing

New `lib/ai-route.ts` (+test, `cron-route.ts` pattern):

```
#ai                      landing / new chat
#ai?filter=archived      archived list
#ai?q=<search>           search text (replace, not push)
#ai/<threadPublicId>     thread open (16-hex, deep-linkable — URL is source of truth)
```

`use-hash-route.ts`: add `'ai'` to the `View` union with a **boundary-safe** match (`#ai` | `#ai/` | `#ai?` — never bare `startsWith`, which would collide with `#ai-gateway`). `nav-data.ts`: `WIRED_HASHES` += `#ai`; new group `{ label: 'AI Interface', href: '#ai', icon: Sparkles, requiresScope: 'a:r' }`. `app-sidebar.tsx` `VIEW_HASH` += `ai`; `App.tsx` `VIEW_TITLE` + `CurrentView` case.

### 3.5 Debug mode + /context panel (UI)

- **Debug toggle** (localStorage `AI_DEBUG`) in the chat header. Per-run inspector renders under each run's **last assistant message** (the transcript already computes that boundary): lazy-fetch on open; one collapsible per model call with tabs **Request** (system prompt as monospace pre + messages/tools as a JSON tree) / **Thinking** / **Response** / **Error** (only when present), plus the existing TokenCostSummary + ToolCallList + ContextManifestPanel row.
- **Context panel** (Sheet): one segmented context-window fill bar against the **currently-picked model's** window (G6, Claude Code `/context` behavior — switching the picker re-renders against the new window; system prompt / tools / messages / free), category legend, cumulative thread tokens + cost (own-thread runs only, G9). Cost comes **only** from the usage endpoint — never summed from `AgentRun.costUsd`, which is `undefined` by design.
- Both are built against `installAgentApiMock` fixtures first (`setRunModelCalls`, `setThreadUsage`) so frontend chunks land independently; contract types land before these chunks.

---

## 4. Test changes

**Playwright — delete:** `agent-panel.pw.ts`, `agent-page-route.pw.ts`, `agent-assisted-editing.pw.ts`, `agent-cron-draft*.pw.ts` (3), `markdown-editor-inline-ai.pw.ts`, `markdown-editor-selection-decision.pw.ts`, `markdown-editor-block-diff.pw.ts`.
**Rewrite:** `agent-pickers`→`ai-pickers`, `agent-token-attachment-badges`→`ai-token-badges`, `agent-message-overflow`→`ai-message-overflow`; re-point `run-telemetry.pw.ts` at `#ai/<id>`.
**New:** `ai-interface-route`, `ai-thread-sidebar`, `ai-chat`, `ai-debug-mode`, `ai-context-panel`, `ai-mobile` (light + dark per convention). Update `e2e/mock-api.ts` fixtures (add `setRunModelCalls`/`setThreadUsage`; `agentRewriteChunks` goes dead with rewrite).
⚠️ **`e2e/markdown-editor-helpers.ts` must be reworked, not deleted:** its `selectBlock` asserts the selection toolbar's visibility and is imported by **16 specs including non-agent ones** (`crons`, `cron-templates`, `registry`, `registry-versions`, `registry-templates`, `settings-org`, `markdown-editor`) — drop the toolbar assertion from the helper in the same change as Chunk A or those suites break.

**Backend:** `turn-engine.test.ts` / `turn-telemetry.test.ts` replaced by `engine` tests (telemetry assertions ported); `tool-schema.test.ts`, `token-resolution` page-context cases, `selection-excerpt-prompt.test.ts` deleted; `sse.integration.test.ts` kept; new provider/shim/recorder/prepareStep-retraction tests.

---

## 5. Keep-in-sync (same commit, per root CLAUDE.md)

`@bot/contracts` (`agent.ts`, `route-scopes.ts`, **barrel re-exports in `index.ts`**, `contracts.test.ts` assertions) → **`@bot/repo`** (`agent.repo.ts` proposal methods, `types.ts`, `index.ts` exports, schema entry) → Bruno `api_collection/agent/` (delete proposal/DELETE requests; **rewrite `thread-messages.bru`'s body + docs prose** — `origin`/`pageContext`/`selectionExcerpt` are baked in, not just separate files; repoint `pickers-models.bru`; fix `README.md`'s `a:d` triad mention) → `docs/api.md` + `docs/security.md` + `docs/modules/guardrails.md` (propose_edit gate) → `docs/modules/agent.md` (rewrite; record the re-answer to the two SDK rejections — cite them precisely as DA2 (heartbeat seam) and C19/R8/DA18 (drain-before-replay / fold / detached runs), not both under "DA2") → feature docs (new `feature/ai-interface.md`; amend `feature/agent-model-picker.md` §4 for catalog sourcing; **mark `feature/ajv-tool-arg-validation.md` retired** — dropping ajv retires that whole spec; add a disposition note to `feature/agent-tool-execution.md` and the 10 `feature/dashboard-agent*.md` files describing the deleted panel) → **`agent-client-seed.ts`: add `ai:r` to `AGENT_SCOPES`** (the seed full-replaces `client_scope` on every boot — an out-of-band grant is wiped on restart) → dashboard `CLAUDE.md` structure section (the feature-dir list is stale anyway; add `ai-interface/`) → **`feature/kong.md` §15 + `docs/modules/kong.md`** (SR10's `agent-service` `read_timeout` is a recorded deviation) → **`setup.md` + `docker-compose.yml`** (SR11's loopback-bound 5432, Phase 0) → `docs/modules/guardrails.md` (SR11's `knownSecrets` wiring — the doc currently records that no caller supplies it) → migration-safety note recording G1's two-commit sequencing → route/coverage tests.

---

## 6. Phasing

### Phase 0 — live-today hardening (lands on `main` first, independent of the rebuild)

Three review findings are live in the **current** system, not properties of this plan — they
are adopted as the plan's first work items so they don't wait on the feature branch (G3 merges
Phase 1 only as a whole; these can't):

1. [backend] **Bind the compose Postgres to loopback**: `docker-compose.yml` publishes 5432
   unbound on all interfaces with the default password (the same file already binds Redis to
   `127.0.0.1` and remarks on the difference). One line + `setup.md`. Pulled forward from SR11.
2. [backend] **Redact title-call input in the current `thread-title.ts`**: the bypass exists
   today — `thread-title.ts:155-163` reaches the model through zero guardrail gates on every
   new thread's first message. Apply the deterministic `redact(['secrets'])` pass now; the
   Phase-1 `generateText` port (step 5) carries it over. Pulled forward from SR1.
3. [ops, no code] **Seed a baseline org-level `ai_limit` row** via the existing dashboard
   editor (`PUT /llm/api/limits`): nothing seeds limits or policies today, so an unconfigured
   install allows everything and only meters — a coarse org-level token/cost cap bounds spend
   before the chattier UI ships.

### Phase 1 — core (this build)

**All of Phase 1 lands on one feature branch, merged once after the full integration gate (G3)** — `main` never sees the panel-removed/page-absent gap between Chunks A and D.

1. **[backend] Day-1 spike test (tripwire for R1/R2):** `ai@7.0.93` + `@ai-sdk/anthropic` `streamText` → local Connectra `/anthropic/v1/messages` round-trip with tools + thinking; verify metering records model/tokens. **Exit criteria include:** exactly one `ai` copy workspace-wide (lockstep `@ai-sdk/react` bump — it hard-pins `ai: 7.0.74` today) and the zod peer (`^3.25.76 || ^4.1.8`) resolved.
2. [backend] `provider.ts` + fetch shim (fallbacks injection, error classification) + tests.
3. [backend] **Proposal code removal (G1 commit 1):** delete every proposal code path across `modules/agent`, `@bot/repo`, contracts (types, barrel, tests) — table still present, unused. Then **migration 0069 (G1 commit 2):** `agent_model_call` (nullable `run_id`, G7; **`thread_id` NOT NULL, SR2**) + drop `agent_proposal` + **`(thread_id, id)` index on `agent_message` (SR9)**; `AgentRepo` methods (`createModelCall`, `finishModelCall`, `listModelCallsForRun`, `findModelCallByPublicId` — all owner-joined) + boot-sweep extension + **batched retention step in the daily purge sweep (G2/SR9, `MODEL_CALL_RETENTION_DAYS` default 30)**.
4. [backend] `model-call-log.ts` recorder middleware.
5. [backend] `engine.ts` + rewritten `tools.ts`; port `thread-title.ts` to `generateText` **carrying Phase 0's SR1 redact-before-send**; **wire `knownSecrets` into the outbound scan seams (SR11)**; delete `turn.ts`/`tool-schema.ts` + dead tests. Widen `@bot/shared` `pricing.ts` to keep the window fields (G5), **loosening `parseEntry`'s both-costs gate (SR12)**.
6. [backend] `app.ts` route rewrite + contracts + `ROUTE_SCOPES`; **add `ai:r` to `AGENT_SCOPES` in `agent-client-seed.ts`** (seed full-replaces on boot); **per-user in-app rate limiter (SR8)**, route-level 8192-byte `text` validation (SR10), `count_tokens` cache on the usage endpoint (SR7); Bruno + `docs/api.md`.
7. [backend] **Infra hardening (SR10/SR11):** Kong `agent-service` `read_timeout: 360000` + `feature/kong.md` §15 / `docs/modules/kong.md`; agent `Bun.serve idleTimeout` 240 → 360; explicit `idleTimeout` on Connectra's `Bun.serve`; idle-SSE-past-60s integration test. (The compose 5432 loopback binding already landed in Phase 0.)
8. [frontend] **Chunk A** — removal (panel + page-context + assisted editing, atomic; grep gate: zero `agent-bridge` imports; full suite run after).
9. [frontend] **Chunk B** — route/nav/shell page (deep links, scope gating, `#ai-gateway` non-collision).
10. [frontend] **Chunk C** — thread sidebar (create/rename/archive/unarchive/search/recency groups/empty states).
11. [frontend] **Chunk D** — streaming chat core on the new page (send/stream/stop/resume-reconnect/retractions).
12. [frontend] **Chunk E** — composer tokens + pickers + catalog-backed model picker.
13. [frontend] **Chunks F/G/H** — debug inspector · context panel · narrow viewport.
14. [backend] Docs pass (§5).

### Phase 2 — sub-agents (§2.7, migration 0070)
`agent_run.parent_run_id`; the `delegate` dynamicTool with the registry-fed roster
(**ownership-filtered, SR3**) + built-in `general-purpose`; sub-run engine assembly (entry
body as system prompt **pinned by version with an audit row, SR4**, model pin-else-inherit,
toolset minus `delegate` **and connectors (SR5)**, chained abort, blocking-parallel cap 4 /
depth 1 / **per-turn budget 8 + owner-cap exclusion, SR6**);
parent-fold exclusion of child-run rows + fold tests; `GET /agent/runs/:publicId/children` +
contracts + Bruno; UI task rows + drill-in via the run-addressed replay; guardrail-per-sub-run
tests; docs (`feature/ai-interface.md` §sub-agents, `docs/modules/agent.md`,
`docs/modules/skills.md` registry-consumption note — the `agent` kind gains a second reader).

### Phase 3 — branching / edit-as-fork
Migration 0071 lineage columns, `listMessagesForLineage` (including the post-cut retraction rows, §2.5), branch route, fold-over-lineage; UI: hover "Branch in new chat", edit-user-message = edit-and-branch prefill (two calls, G8), branch-indicator chip.

### Phase 4 — background coding agents (sketch)
HarnessAgent per Appendix A: `ExecutionEngine` seam in the runner daemon, dashboard-initiated task lane, Claude-Code-adapter-only, on the dedicated EC2, Connectra fail-CLOSED, host-exec → Docker-per-session isolation, DB-tail SSE. `modules/agent` untouched by the lane; the AI Interface gains a `#ai/tasks` pane reading the task API. The POC Phases 0–1 tripwire stands (Appendix A §A4). The lane gets its **own security pass when planned in full** — it adds process execution, so Appendix A3's rails (fail-CLOSED, adapter restriction, isolation path) are entry conditions, not options, and R11–R14 apply to it unchanged.

### Phase 5 — connectors-in-chat
Per-thread connector enablement UI; tool-approval flow as a **DB-parked approval on our own persistence** — the `guardrail_hold` pattern: a new `agent_message` kind (additive CHECK widening) parks the tool call, a later user action resumes it. **Not** the SDK's `toolApproval`: that approval-as-later-message mechanism is documented for HarnessAgent, and `ToolLoopAgent` approvals are non-durable ("lost on crash", Appendix A §A5); the durable SDK path (WorkflowAgent) is already rejected on the storage invariant (§A5). Skills/subagents/custom-agents surfacing from the registry.

---

## 7. Verification

- Integration-gate tier: **full** — typecheck + all module tests + dashboard lint/build + Playwright + smoke, run once after all subtasks are accepted.
- Backend: the spike test (step 1) gates everything; `bun test modules`; `bun run smoke` before commit.
- End-to-end: `bun run dashboard:dev`, open `#ai`, send a message against local Connectra/Bifrost, toggle debug and inspect a turn, open the Context panel, archive/unarchive, reload mid-run to prove resume.
- Ground truth for the Phase 4 (HarnessAgent) claims: **Appendix A** below — inlined because the source files live in gitignored working directories.
- Phase 2 end-to-end: author a registry `agent` entry, send a prompt that warrants delegation, watch parallel task rows stream, drill into a sub-run's transcript + debug inspector, Stop mid-delegation and confirm the cascade terminalizes children. Confirm a second user's user-scoped entry does **not** appear in the roster (SR3).
- Review-driven checks: SR10's idle-SSE test (a stream carrying only heartbeats survives Kong past 60 s); SR2 (a title call's model-call row is fetchable by its thread owner, 404 for anyone else); SR1 (a pattern-shaped secret in a first message is redacted in the recorded title-call request); SR6 (a delegating turn does not trip the owner's own concurrency cap).
- Phase 0: 5432 refuses connections from a non-loopback address after the compose change; a unit test on the current `thread-title.ts` asserts a pattern-shaped secret never appears in the outbound title prompt; the org `ai_limit` row denies with 429 once tripped.

## 8. Risks

| # | Risk | Mitigation |
|---|---|---|
| R1 | **Blocking:** `@ai-sdk/anthropic` compat with `ai@7.0.x`'s LanguageModelV4 spec — never installed in this repo; **and `@ai-sdk/react@4.0.77` hard-pins `ai: 7.0.74`**, so a naive bump forks two `ai` copies | Day-1 spike test; pin after checking its `@ai-sdk/provider` peer; spike exit criteria: single `ai` copy (lockstep `@ai-sdk/react` bump) + zod peer resolved |
| R2 | **High:** thinking blocks + tool streaming byte-for-byte through Connectra/Bifrost — believed safe (passthrough), never exercised | Same spike; integration test kept |
| R3 | Fetch-shim body mutation couples to provider serialization | Contain in one function with a body-is-JSON test |
| R4 | In-memory loop vs per-round re-fold: mid-run retractions must be filtered in `prepareStep` or retracted content resends within the same turn | Dedicated test |
| R5 | `agent_model_call.request` stores full history × tools per call — O(thread²) growth | **Resolved by G2:** retention ships in Phase 1 (daily sweep nulls bodies past `MODEL_CALL_RETENTION_DAYS`) |
| R6 | ~~Static `MODEL_CONTEXT_WINDOWS` staleness~~ | **Retired by G5:** windows come from the already-self-refreshing LiteLLM table via `pricing.ts`; static fallback only for unlisted models — with SR12's `parseEntry` loosening, since the parser today drops any entry missing either cost field |
| R7 | Agent auth client lacks `ai:r` (confirmed — seed has `ai:i`/`ai:dg` only) | Edit `AGENT_SCOPES` in `agent-client-seed.ts` — the seed full-replaces on boot, so out-of-band grants are wiped |
| R8 | Frontend: bridge fan-out atomicity; `#ai` vs `#ai-gateway` matching (`use-hash-route` matches by `startsWith` — the `#ai` branch must sit after `#ai-gateway` or test boundaries); contracts-first ordering for debug/usage chunks | Chunk A atomic + zero-imports grep gate; boundary-safe matcher (`#ai` \| `#ai/` \| `#ai?`) + `navGroupHrefForHash` longest-prefix check; land types before chunks F/G |
| R9 | Branch lineage cut dropping later retraction rows would resend guardrail-retracted content | Designed out in §2.5 (lineage includes post-cut retraction rows); dedicated fold-over-lineage test |
| R10 | Sub-agents (§2.7): a child-run row leaking into the parent fold corrupts the parent prompt; a registry `agent` body is user-authored content elevated to a sub-run system prompt | Fold-exclusion is one predicate (`parent_run_id IS NOT NULL`) with dedicated fold tests incl. replay parity; the outbound guardrail scan covers the sub-run system prompt like any other prompt part — plus SR3 (roster ownership filter), SR4 (version pin + audit), SR5 (no connectors token) as the actual containment |
| R11 | **Single-instance ceiling is inherited, not incidental:** the kept `runs.ts` registry is a process-local Map with in-process pub/sub, and `reclaimLostRuns` terminalizes every `active` run fleet-wide with no instance filter — a second instance kills the first's runs at boot. Phase 1 capacity = one Bun process on the shared EC2 (7 PM2 apps + 9 containers) | Accepted under D6; the scale-out path is recorded in the architecture companion §9 so Phase-1 code preserves the chunks-derived-from-rows invariant it depends on |
| R12 | Delegated-bearer blast radius: 30-min TTL (schema cap ≤1800s), one fleet-wide audience, Kong verifies locally with no jti blocklist/introspection — a leaked run bearer is valid at `/mcp`, `/api/*`, `/agent`, `/llm/*` for its full TTL, outliving the 5-min turn by ~25 min with no kill switch; Phase 2 multiplies mints | SR5 stops minting the connectors token per sub-run; `delegation_token_id` is already stored per run for a future revoke-on-terminalize hook (real enforcement needs an `onexo-auth` plugin change — recorded, not built) |
| R13 | The debug store is a high-value plaintext prompt/response sink — arbitrary user-pasted secret *values* are not format-shaped and survive redaction — next to a compose Postgres publishing 5432 unbound with a default password | SR11 (`knownSecrets` denylist wired + 5432 loopback-bound); G2 bounds the exposure window to `MODEL_CALL_RETENTION_DAYS` |
| R14 | Archive-never-delete (D5) + the append-only trigger = **no data-subject-deletion path, permanently** (`agent_message` rows cannot be deleted without superuser/TRUNCATE; model-call summary columns and `ai_usage_event` are joinable forever, neither has retention) | Accepted consequence of D5, recorded here so it is a decision, not an oversight; revisit before any non-internal deployment |

---

## Appendix A — harness-adoption decision extract (2026-09-09)

The harness-adoption decision and its supporting dossier live in gitignored working
directories, so everything this plan relies on from them is reproduced here — this appendix
is the in-repo record.

### A1. The decision this plan supersedes (in part)

Adopt the Vercel AI SDK 7 **HarnessAgent** via a strangler (Option 4 of the evaluated
alternatives), for the **dashboard-initiated background-task lane only**; the chat lane
(`@bot/agent`) stays untouched. The load-bearing sentence on the chat lane:

> **Lane B: NO HarnessAgent.** `@bot/agent` stays (detached runs + drain-before-replay
> reconnect already exist; a bridge+sandbox per chat turn is pure overhead). Selective AI SDK
> plumbing adoption only if the two documented rejections (heartbeat-interleaving seam,
> run-addressed reconnect) are re-answered.

This plan takes exactly that escape hatch: §2.1 re-answers both rejections (kept `stream.ts`
and `runs.ts`) and adopts the SDK for the model-call/tool loop only. HarnessAgent remains
Phase 4's mechanism.

### A2. The two documented rejections of AI SDK server primitives

1. **Heartbeat seam (DA2, `feature/dashboard-agent.md`):** `createUIMessageStream`/
   `createUIMessageStreamResponse` were verified present under `ai@7` but neither gives a seam
   to interleave a heartbeat independent of message chunks — DA2 requires the 15 s heartbeat
   "never chained to model output". `stream.ts` hand-rolls the SSE transport, emitting the
   identical `UIMessageChunk` payloads; only the SDK *types* are used.
2. **Replay parity / run-addressed reconnect (C19 / R8 / DA18):** reconnect replays from
   persisted rows, and C19's rule is drain-before-replay — in-flight batch validation drains
   to zero *before* the replay query, "wait, then query, never a race against an undecided
   verdict". A live SDK stream cannot satisfy this; chunks must derive from rows. DA18 is the
   detached-run registry (a turn survives its client), DA31 its one-instance/in-process
   constraint — which this plan inherits by keeping `runs.ts`.

### A3. Phase 4 (background coding agents) — supporting decisions

- **`ExecutionEngine` seam** sits above `runCli`, at the `runResumable` call-site level —
  **not** inside `CliAdapter`.
- **Claude-Code-adapter-only**: the Codex adapter would need `permissionMode: 'allow-all'`,
  which is unacceptable under the multi-CLI safety rules (MC14).
- **Per-conversation engine pin** (the `cli-pin.ts` analog) and the **MC-CHECK admission
  gate** (`feature/multi-cli.md` §7) are named supporting decisions, not optional.
- **Connectra fail-CLOSED** for the lane — noting the claude CLI path is currently fail-OPEN
  (A8 unbuilt), so this is work, not a property.
- **Isolation path**: host-exec first, then Docker-per-session.
- **DB-tail SSE** carries two conditions: `x-accel-buffering: no` (the gateway buffers SSE
  otherwise) and a raised per-lane RunEvents cap (the 2000-row cap is too small for
  multi-hour tasks).

### A4. The POC tripwire

Phases 0–1 of the HarnessAgent proof-of-concept are the gate: if the strangler costs more
than ~1 engineer-week to reach the tripwire milestones, fall back to Option 1 (keep the
existing CLI-adapter lane unchanged).

### A5. SDK agent-class facts this plan's phasing depends on

- **`ToolLoopAgent`** (`ai` main entry): in-memory tool loop; **approvals not durable — lost
  on crash**. This is why Phase 5's tool-approval flow is DB-parked on our persistence, not
  the SDK's `toolApproval`.
- **HarnessAgent** is where approval-as-later-message is documented — not `ToolLoopAgent`.
- **WorkflowAgent** (durable approvals) requires the Workflow DevKit, whose
  `@workflow/world-postgres` brings its own pg-boss/Drizzle tables — the storage-invariant
  violation that already disqualified Mastra. Rejected.
