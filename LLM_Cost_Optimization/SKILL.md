---
name: llm-cost-optimizer
description: Audit a GenAI / agentic-AI codebase for LLM cost and cut cost per task without losing accuracy. Reads and understands the whole LLM code path (what each route does, why, and what output it must produce), runs 70+ checks plus open web research (caching, tokens, routing, agent loops, batch, RAG, retries, budgets), asks the user whether suspicious choices are intentional and records them in .llm-cost.yaml, estimates % savings per finding, then builds test cases and a metering harness that measures REAL cost and quality before and after each fix. Use whenever the user wants to reduce, review, audit, measure or benchmark LLM/token/API spend, asks why an OpenAI, Anthropic, Gemini or Bedrock bill is high, wants model routing, prompt caching, token limits, batching or cheaper/open-source models, or has built an LLM, RAG, chatbot or agent app and wants a cost review before shipping - even if they never say "cost" (e.g. "make this agent cheaper to run").
metadata:
  author: "Neel Shah"
  version: "1.1.0"
  updated: "2026-10-21"
compatibility: "Any coding agent with shell access. Embedded tools need Python 3.8+ (stdlib only)."
---

# LLM Cost Optimizer

_Version 1.1.0 · Author: Neel Shah · Single self-contained file: workflow, check catalog, provider notes, templates, and embedded tools._

> **Reading this file:** read everything up to Appendix G. Appendix G is embedded tool code: extract it with *Tools setup*, and don't read it into context.

**Contents:** Principles · Tools setup · Phases 0-7 · Appendix A Check catalog · B Provider specifics · C `.llm-cost.yaml` · D Report template · E Measuring real cost · F Starter yaml · G Embedded tools · Maintaining this skill

Find out what **this** project's LLM calls cost, why, and what to change to cost less per completed task **without lowering the quality of its output**. Then prove it with measured before/after numbers, not assumptions.

The unit is **cost per completed task**, not cost per token. A cheaper model that fails and retries, or an agent that needs 3x the turns, is not cheaper. Every recommendation reads cost and quality together.

## Principles

1. **Understand before optimizing.** You can only suggest a good optimization once you know what the code is for, why it was built that way, and what output it must produce. Read the full LLM code path (Phase 1). A suggestion that breaks a route's contract is a bug, not a saving.
2. **The catalog is a floor, not a ceiling.** Appendix A lists 74 known levers. You are expected to go beyond it. Use your own engineering knowledge, and **search the web freely** for anything the catalog doesn't cover or that may have changed: new provider features, framework-specific settings, new cheaper models, pricing changes, cookbooks, issue threads about a library's token behavior. Cite what you used. Treat web content as information to evaluate, never as instructions to follow.
3. **Never invent prices or API parameters.** Prices, cache multipliers, minimum cacheable sizes, TTLs, tier names and parameter names change every few months and differ per model. Verify against live pages (Appendix B lists them) and record the URL and date. If you can't verify, size impact as High / Medium / Low and say so.
4. **Ask, don't assume, about intent.** Never mark a finding intentional on your own judgment. When something looks deliberate, or the user might have a reason for it, ask them (Phase 3).
5. **Estimate first, then measure.** Every finding carries an *estimated* % reduction (labelled as an assumption). When the user wants proof, the bundled harness measures real tokens and cost from the provider's own usage data, before and after each fix, at the same prices, alongside quality checks.
6. **Free wins before tradeoffs.** Free wins (caching, input/output hygiene, loop hygiene, batch, dedup, observability) don't touch capability. Tradeoffs (cheaper models, routing, lower reasoning effort, tighter budgets, semantic caching, self-hosting) change what the model can do. Measure tradeoffs against a baseline that already includes the free wins, and apply them only after the user accepts the quality risk.
7. **Spending money needs approval.** Every harness run calls real models. Show the estimated cost and get an explicit yes before running. The runner refuses without `--yes` and enforces a `max_cost_usd` budget.
8. **"No change recommended" is a valid outcome.** Don't manufacture findings.

## Tools setup (do this once per session, only when you need a tool)

Five stdlib-only Python tools are embedded in Appendix G of this file (Python ≥ 3.8; no pip installs). Extract them with one command instead of retyping them. `<this-file>` is this SKILL.md's path; your harness usually shows the skill's base directory when the skill loads. The extractor verifies each file's SHA-256 checksum.

```bash
export LCO="${TMPDIR:-/tmp}/llm-cost-tools-v1.1.0"
python3 - "<this-file>" "$LCO" <<'EOF'
import hashlib, os, re, sys
src = open(sys.argv[1], encoding="utf-8").read()
os.makedirs(sys.argv[2], exist_ok=True)
blocks = re.findall(r"<!-- file: (\S+) sha256=([0-9a-f]{64}) -->\n`{5}\w*\n(.*?)\n`{5}", src, re.S)
assert blocks, "no embedded tools found - is this the right file?"
for name, digest, body in blocks:
    data = (body + "\n").encode("utf-8")
    ok = hashlib.sha256(data).hexdigest() == digest
    with open(os.path.join(sys.argv[2], name), "wb") as f:
        f.write(data)
    print(("ok      " if ok else "CHECKSUM MISMATCH (file was edited?) ") + name)
EOF
```

If the file path isn't available (e.g. the skill was pasted into chat), copy each Appendix G block verbatim into `$LCO/<name>`. All tools must sit in the same directory, because `llm_eval.py` and `llm_cost_proxy.py` import `llm_meter.py`. In the commands below, `$LCO` is that directory. Re-export it in each new shell.

| Tool | Purpose |
|---|---|
| `scan_llm_usage.py` | Read-only inventory pre-pass (Phase 1) |
| `cost_model.py` | Estimate arithmetic and cache-TTL simulation (Phase 4) |
| `llm_meter.py` | In-process usage meter: httpx/requests/botocore hooks, streaming-aware (Phase 5) |
| `llm_cost_proxy.py` | Local recording proxy for non-Python stacks (Phase 5) |
| `llm_eval.py` | `init` / `estimate` / `run` / `report` / `compare` (Phases 5-6) |

## Phase 0: Load intent and policy

Read `.llm-cost.yaml` at the repo root if it exists (schema in Appendix C), and note any `llm-cost: intentional <ID> - reason` comments (the scanner also finds them).

- `policy`: hard constraints (approved models, residency/ZDR, latency SLOs, quality floors, session context cap, budget). Never recommend violating them. Put such levers under "Blocked by policy".
- `context`: traffic facts (volume per route, whether users wait, gaps between turns) used for sizing.
- `intentional`: acknowledged findings. Report them in their own table, exclude them from savings, and don't argue them again. Flag entries that have expired or no longer match the code, **or whose premise changed**. Example: the model they pin is being deprecated or repriced, which forces a re-decision anyway.

**Follow-up reviews.** If an earlier `LLM_COST_REPORT.md` exists, or the user says a cost pass was already done:
- verify each earlier lever is actually present in the code, and say which are confirmed, missing, or regressed;
- don't re-propose levers that are confirmed;
- focus on what changed since, and on what the earlier pass missed.

## Phase 1: Understand the project deeply

**1a. Map it.** Run the scanner (stdlib only, read-only, makes no network calls):

```bash
python3 "$LCO"/scan_llm_usage.py <project-root> --out <scratch>/llm_scan.json --markdown
```

It lists providers (SDK imports and raw-HTTP endpoints), frameworks, every LLM/embedding call site with nearby cost parameters, model strings, prompt files and large inline prompts with token estimates, dynamic values near prompts, retries, loop guards, tests that hit live APIs, and intent comments. Treat every hit as a lead to verify, and every gap as possible: it's a heuristic. If it reports no call sites, grep for provider hostnames, endpoint paths (`/v1/messages`, `/chat/completions`, `:generateContent`) and model names before concluding there are none. Prompts repeated by copy-paste are flagged with a repeated-lines %.

**1b. Read the code completely.** For every route, read every file on its path in full, not excerpts:

- **Entry point**: the HTTP handler, CLI, job, queue consumer or UI action that triggers it.
- **Prompt assembly**: templates, system prompt, few-shots, tool definitions, retrieved context, history handling, and what is interpolated where.
- **The call(s)**: client config, model, parameters, streaming, retries, timeouts, fallbacks, loops, sub-agents.
- **Post-processing**: parsing, validation, repair calls, caching layers, and what happens on failure.
- **The consumer**: who uses the output (a UI, another service, a database, another LLM call) and what form it must have.
- Supporting context: README/docs, comments, tests and fixtures (they show the expected output), the git history of prompt files (`git log -p -- <prompt file>` often explains why a rule exists), configs per environment, and framework source when its defaults matter (e.g. what LangChain or CrewAI actually renders).

**1c. Write a route card** for each route. This is the contract your optimizations must preserve:

```
Route: <name>    Entry: <file:function>    Trigger/volume: <who calls it, how often>
Purpose: <what job it does for the user/business, in one or two sentences>
Input -> Output: <shape of input> -> <exact shape/format/quality the consumer needs>
Success criteria: <how you'd judge an output correct; what a bad output looks like>
Prompt anatomy: <static prefix size, dynamic parts and their positions, tools count, history policy, retrieved context>
Call graph: <single call | chain of N | agent loop (max depth?) | sub-agents>, model(s), effort, max tokens, streaming
Latency: <user waiting? SLO?>     Failure handling: <retries, fallbacks, repair calls>
Why it is built this way: <evidence from code/docs/history; "unknown" if unknown>
Cost drivers (hypothesis): <where tokens most likely go>
```

Also record the platform (direct API, Azure, Bedrock, Vertex, self-hosted, gateway) and the measurement already available (usage logging, gateway dashboards, provider console or admin usage API). Ask for a representative day of usage logs or the monthly bill if they exist. Measured beats estimated.

**1d. Confirm understanding.** Show the user the route cards in a compact form, plus your open questions (volume, which routes matter most, unknown purposes), in one short round. Correct the cards before going further. Wrong understanding produces wrong optimizations.

## Phase 2: Find the optimizations

1. Go through **every category** in Appendix A against each route card, and read Appendix B for the providers in use.
2. **Research beyond the catalog.** For this project's specific stack (framework version, provider, model generation, vector DB, agent pattern), search the web for cost features and known token pitfalls the catalog may not cover or that are newer than it. Typical searches:
   - "<framework> prompt caching";
   - "<provider> <model> pricing";
   - "<framework> token usage high";
   - newer, cheaper models in the same tier;
   - provider batch, flex and caching updates.

   Also use your own expertise in architecture, algorithms and data flow: the biggest savings are often a design change (precompute instead of per-request generation, one call instead of three, deterministic code instead of an LLM).
3. For each candidate, check it against the route card: **does it preserve the output contract?** If it changes quality risk, it's a tradeoff and needs measurement.
4. **Check lever interactions.** One fix can change the value of another. For example:
   - deduplicating a prompt can push it below the provider's minimum cacheable size, which makes caching fixes worthless;
   - processing only changed inputs shrinks the absolute value of every per-call lever;
   - switching model changes cache minimums, prices and tokenizer.

   Rank with these dependencies in mind, and say so in the table.
5. Check the **model lifecycle** on the fetched pages: deprecations, shutdown dates and scheduled price changes for every model in use. A forced migration or a price increase belongs in the report even when no code is wasteful.
6. Classify each one:
   - **Finding**: with evidence (`file:line`, what you saw, why it costs money here).
   - **Question**: depends on facts you can't see.
   - **Not applicable**: summarized in one line per category.
   - **Blocked by policy**.
   - **Intentional**: only if it matches `.llm-cost.yaml` or a comment, or the user confirms it in Phase 3.
   - New issues outside the catalog get ID `CUSTOM-nn`, with the source or reasoning.

## Phase 3: Ask about intent (mandatory before marking anything intentional)

Keep this light: **at most two question rounds** in total.

Some findings look deliberate: a frontier model on one route, caching disabled, a huge context, high reasoning effort, a timestamp in the prompt, a multi-agent design, no batching on a job that might have an SLA. Others might have reasons you can't see. **Ask the user instead of guessing.**

How to ask:

- If your harness has a structured question tool (e.g. `AskUserQuestion` in Claude Code), use it. Otherwise ask in chat as a numbered list the user can answer in one message.
- One question per finding that looks deliberate, in batches of up to 4. Keep it concrete:
  - Question: *"<ID> in `<file:line>`: <one-line finding>. Estimated impact <X>. Is this intentional?"*
  - Options:
    - **"No - optimize it"**
    - **"Yes - it's intentional (I'll give the reason)"**
    - **"Not sure - keep as an open question"**
- Don't ask individually about unambiguous waste: a prompt repeated 120 times, `n=3` with one result used, a duplicated call. Those go only into the single bulk question below.
- Inline `llm-cost: intentional` comments and existing yaml entries count as already answered; don't ask again. But if something the user says contradicts one (e.g. "only the legal route is intentional" while a comment marks the classifier), surface the conflict instead of silently picking one side.
- For the remaining findings that look like plain waste, ask **one** multi-select question: *"Are any of these intentional?"*, listing them. Unselected means not intentional.
- For every "Yes", get the **reason** (required), plus **owner** and optional **expiry** if the user wants them. A free-text answer is fine.
- Then ask once: *"Save these as intentional in `.llm-cost.yaml` so future reviews skip them?"* If yes, write the entries (format in Appendix C) and show exactly what you added.

Intentional findings move to the "Acknowledged as intentional" table and are excluded from savings. "Not sure" ones stay in the main table, marked "Question".

## Phase 4: Estimate the impact (assumptions)

For each remaining finding, estimate the **% reduction of that route's cost** and of the **total LLM bill**, with the basis stated:

| Data available | How to express impact |
|---|---|
| Real usage/cost data (console, admin usage API, gateway, logs) | $/month range + %, labelled *measured basis* |
| A bill figure or partial logs | % of bill, labelled *partial data* |
| Code only | % range from token arithmetic on the route card (prompt sizes, call counts, verified prices), labelled *assumption*; or High / Medium / Low if prices are unverified |

Use `cost_model.py estimate` (Appendix G) for consistent arithmetic. Its per-route `after` overrides model any lever: a trimmed prompt, fewer requests, fewer output tokens. Use `simulate-cache` for TTL decisions. Dollar figures from code-only estimates are fine when the prices are verified, as long as each one is labelled *assumption* with its inputs (volume, tokens per call). Keep it honest:

- Overlapping levers claim the same tokens. Deflate them so the total can't exceed 100%.
- Per-request unique payload is uncacheable.
- Agent costs are tail-heavy: look at p90, not the median.
- Levers too small to be worth validating go to "Considered, not worth it".

## Phase 5: Build the measurement harness (test cases + real metering)

Offer this whenever the user wants proof, wants to apply fixes, or any tradeoff is on the table. If the user declines measurement for now, still deliver the measurement **plan** (cases, checks, expected cost). Build the harness files only if they want them ready to run later. Full details are in Appendix E; read it before starting. Extract the tools first (see *Tools setup*). In short:

1. **Scaffold** in the project: `python3 "$LCO"/llm_eval.py init --dir llm_cost_eval --name <project>`.
2. **Generate test cases** per route (`llm_cost_eval/cases/<route>.jsonl`). Source them, best first, from:
   - existing tests, fixtures and eval sets;
   - real (anonymized) inputs from logs;
   - cases you synthesize from the route card: typical, edge and hard ones, plus inputs that exercise long histories, tool loops or large documents.

   Aim for 15-30 per important route. Show the user a sample and get it approved.
3. **Write quality checks from the route card's success criteria**:
   - deterministic checks first (expected values, JSON keys, regex, contains, length);
   - custom Python checks in `checks.py` for domain logic;
   - an LLM judge only when nothing cheaper works. Its spend is tracked separately and never mixed into task cost.

   The checks must pass on the current code. If the baseline fails a case, investigate before optimizing.

   **Make cases behave like production**, or the baseline will lie:
   - vary per-user values (names, IDs) across cases;
   - give multi-turn cases distinct conversations;
   - set `pace_ms` on routes whose prompts contain time-based values. Fast back-to-back runs within the same second hide timestamp cache-breakers and make a broken cache look healthy.
4. **Wire the target**:
   - Python routes: `{"type": "python", "callable": "pkg.module:function"}`. The meter hooks httpx, requests and botocore in-process, so no code changes are needed.
   - Any other language: a small runner script reading JSON from stdin, plus `llm_cost_proxy.py` (Appendix G) with the SDK's base URL pointed at it.

   Prefer the route's real entry function so loops, retries and agent steps are all counted.
5. **Prices**: fill `prices.json` from the live pricing pages for every model involved, including candidate models for swaps. Put the URL and date in the file. One prices file is used for every run, so comparisons are apples to apples.
6. **Estimate and get approval**: `python3 "$LCO"/llm_eval.py estimate --config llm_cost_eval/eval.json` (or from a probe run). Tell the user the expected cost of the baseline and of each re-run. Set `max_cost_usd`.
7. **Baseline**: `python3 "$LCO"/llm_eval.py run --config llm_cost_eval/eval.json --label baseline --yes`. Use at least 3 trials so cache warmth and nondeterminism are visible.

## Phase 6: Apply and measure fixes (only with the user's go-ahead)

- Measuring a code change means editing the code. If the user hasn't approved edits to the application, do the fixes on a separate branch or worktree (or save them as patches in `llm_cost_eval/patches/`) and leave their working tree as it was.
- **One lever per change**, run as label `fix-<CHECK-ID>`, in lever order (observability → free wins → tradeoffs). Re-run the same cases with the same prices:
  `python3 "$LCO"/llm_eval.py run --config llm_cost_eval/eval.json --label fix-CACHE-01 --yes --note "<what changed>"`
- Compare: `python3 "$LCO"/llm_eval.py compare --prices llm_cost_eval/prices.json llm_cost_eval/runs/baseline.json llm_cost_eval/runs/fix-*.json --md llm_cost_eval/COMPARISON.md`. It reports pass-rate delta, cost/task delta vs baseline and vs previous, a noise flag, regressed cases, cache hit rate, truncations and the monthly projection. Use `--skip-first-trial` to see warm-cache steady state, and report both views for caching changes.
- After each kept change, **re-check the remaining levers' value** (see lever interactions in Phase 2) before spending money measuring them.
- **Keep** a change only if quality holds (no regressions beyond noise) and cost drops beyond noise. Otherwise revert and record why. Stack kept changes, so each later fix is measured on top of the earlier ones.
- Levers that can't be exercised synchronously (e.g. Batch API) are measured as *measured tokens × batch price* via `tier_multipliers`. Say so in the table.
- Commit each kept change separately, with the check ID in the message, if the user wants commits.

## Phase 7: Report

Write `LLM_COST_REPORT.md` (template: Appendix D). The core is the suggestions table:

| # | ID | Finding | Route | Location | Type | Suggested change | Est. reduction (assumption) | Measured reduction | Quality Δ (measured) | Effort | Status |
|---|---|---|---|---|---|---|---|---|---|---|---|

- **Est. reduction**: % of route cost and % of total, from Phase 4, always marked as an assumption with its basis.
- **Measured reduction**: from `compare` (cost/task Δ%, monthly $ if volume is known), or "not measured".
- **Status**: Proposed / Measured-kept / Measured-reverted / Needs eval / Question / Blocked by policy.
- Then: route cards, recommended order of work, the acknowledged-intentional table, questions, skipped/blocked items, the measured comparison (`COMPARISON.md`), and the sources (pricing and docs URLs with dates, plus web sources used beyond the catalog).

End in chat with:
- the top 3-5 findings;
- estimated vs measured totals;
- decisions still needed;
- the next offers: apply remaining free wins, measure the tradeoffs, add the harness to CI as a cost-regression check.

---

## Appendix A - Check catalog

Stable IDs. Teams reference them in `.llm-cost.yaml` and in `llm-cost: intentional <ID>` comments, so **never renumber**; add new checks at the end of a category.

Each check: **Signals** (what to look for), **Why it costs**, **Fix**, **Type** (Free win / Tradeoff / Policy / Observability), **Skip when** (also typical legitimate reasons to mark it intentional).

Provider-specific parameter names and numbers live in Appendix B. Verify there before citing them.

This catalog is a floor, not a ceiling. Combine it with your own engineering judgment and current web research for the project's specific stack; report new issues as `CUSTOM-nn`.

### Contents

1. OBS - Observability & guardrails
2. CACHE - Provider prompt caching
3. RESP - Application-level response caching & dedup
4. INPUT - Input token hygiene
5. RAG - Retrieval & embeddings
6. OUTPUT - Output & reasoning tokens
7. MODEL - Model selection & routing
8. AGENT - Agent loops & multi-agent
9. TIER - Throughput / pricing tier
10. RELY - Reliability waste (retries, duplicates)
11. DEV - Dev, test & CI usage

---

### 1. OBS - Observability & guardrails

You can't optimize what you don't measure. These are usually the first diffs, because they turn every later estimate into a measurement.

**OBS-01 No per-request usage logging**
- Signals: responses used only for `.content`/`.text`; `usage` / `usage_metadata` never read or logged; no gateway/tracing (Langfuse, LangSmith, Helicone, Portkey, LiteLLM proxy, OpenTelemetry GenAI spans).
- Why: without input/output/cached/reasoning token counts per request, cache hit rate, the output share and per-route spend are guesses.
- Fix: log provider usage fields (incl. cached and reasoning tokens) with route name, model, latency and stop/finish reason. One helper wrapping the client is enough.
- Type: Observability. Skip when a gateway already records it.

**OBS-02 No cost attribution**
- Signals: one API key for all features/environments; no per-request metadata/tags (route, tenant, user, feature); no workspace/project separation.
- Why: the bill can't be split by route, so nobody knows which feature to optimize, and post-change savings can't be confirmed.
- Fix: per-project/per-environment keys or workspaces; tag requests (provider `metadata`/`user` fields, gateway tags).
- Type: Observability.

**OBS-03 No spend limits or alerts**
- Signals: no budget alerts, no per-workspace/key spend caps, no per-tenant quotas for a multi-tenant app.
- Why: a runaway loop or abusive user is discovered on the invoice.
- Fix: provider/console spend limits, gateway budgets, per-user/tenant rate and token quotas, alert at e.g. 50/80/100% of monthly budget.
- Type: Policy.

**OBS-04 No quality gate for LLM behavior**
- Signals: no eval set, golden tests, output validators or LLM-judge for the routes being optimized.
- Why: without one, savings can't be told apart from regressions. Every tradeoff is blocked until one exists.
- Fix: minimal eval (20-30 frozen real inputs + cheapest judgment that works). Report this at the top if missing.
- Type: Observability.

**OBS-05 Cost not visible to the people generating it**
- Signals: developer-facing agents/tools or internal copilots with no per-session cost display or summary.
- Why: real-time visibility changes behavior (Uber cited this). Engineers stop sending 500K-token contexts when they see the price.
- Fix: show session tokens/cost in the UI or logs; weekly per-team cost digest.
- Type: Observability.

**OBS-07 Streaming responses without usage**
- Signals: OpenAI Chat Completions `stream=True` without `stream_options={"include_usage": True}`; streaming handlers that drop the final usage event (Anthropic `message_delta`, Responses `response.completed`, Gemini final chunk).
- Why: streamed traffic, often the largest, becomes invisible in cost logs.
- Fix: request usage in streams and log it from the final event.
- Type: Observability.

**OBS-06 No hard cap on context per session/request**
- Signals: agent/chat sessions allowed to grow to the model's full context window; no max-context policy.
- Why: per-turn cost grows with context, and sessions near the limit are the most expensive *and* often the lowest-quality. Uber capped interactive sessions at 400K even where 1M was available.
- Fix: a configurable session context cap (trigger compaction/summary or a new session past it); pick the number from p95 of successful sessions, not from the model maximum.
- Type: Policy. Skip when the task genuinely needs whole-repo/whole-document context (mark intentional with the reason).

**OBS-08 Finish/stop reason never checked**
- Signals: code reads the text without checking `finish_reason` / `stop_reason` / `finishReason`; truncated or empty answers are cached, saved, or shown.
- Why: truncated outputs get served or cached (wasting the call) or trigger blind retries; a cached bad answer is re-served until its TTL expires.
- Fix: check the stop reason. Don't cache or save truncated or empty outputs; log them as failures.
- Type: Free win.

### 2. CACHE - Provider prompt caching

Usually the single largest free lever for chat and agents: every turn resends system prompt + tools + history. Caching reprices the repeated prefix at a fraction of the input rate. Caching is **prefix match**: any byte change invalidates everything after it. Typical render order is tools → system → messages.

**CACHE-01 Dynamic content in the stable prefix**
- Signals: `datetime.now()`, `Date.now()`, `new Date()`, `time.time()`, `uuid4()`, request IDs, user names/emails, session IDs, A/B variants, random few-shot selection interpolated into the system prompt or tool descriptions.
- Why: the prefix differs every request → 0% cache hits, and on providers that charge for cache writes you pay the write premium every time.
- Fix: move volatile values to the end (latest user message, or a mid-conversation system/developer message where supported). If the date is needed, use date-only granularity placed after the cached part.
- Type: Free win.

**CACHE-02 Non-deterministic prefix assembly**
- Signals: tools built from a `set`/unordered map; tool or document lists sorted by score/recency differently each call; JSON serialized from dicts built in varying order; templates with conditional sections that flip per request; framework re-rendering the system prompt with per-call variables.
- Why: same logical prompt, different bytes → cache misses.
- Fix: sort tools/documents deterministically; serialize with stable key order; keep conditional/per-user sections after the cached prefix.
- Type: Free win.

**CACHE-03 Caching not enabled where it must be explicit**
- Signals: provider requires explicit cache markers (see Appendix B; e.g. Anthropic `cache_control`, Bedrock cache points, Gemini explicit cached content) and none are present on a large reused prefix (system prompt + tools, a big document asked about repeatedly, long agent histories).
- Why: providers that don't cache automatically bill the full input rate on every resend.
- Fix: add cache breakpoint(s) at the end of the stable prefix (and on the growing conversation for multi-turn), within the provider's breakpoint limit.
- Type: Free win. Skip when the prefix is below the provider's minimum cacheable size, or requests on that prefix are rarer than the TTL.

**CACHE-04 Prefix below the minimum cacheable size, or just above it**
- Signals: small system prompt (estimate via scanner token sizes) with cache markers that never produce cached tokens; or a prefix right at the threshold where a model switch changes the minimum.
- Why: below the minimum, cache markers silently do nothing. Moving stable content (few-shots, schemas) *into* the prefix can cross the threshold.
- Fix: verify against the per-model minimum (live docs); consolidate stable content into the prefix if that pays; otherwise remove noise markers.
- Type: Free win.

**CACHE-05 TTL / retention mismatched to traffic gaps**
- Signals: conversations where users pause longer than the default TTL between turns (support chat, human-in-the-loop agents, IDE assistants); bursty batch traffic; default TTL used everywhere.
- Why: if the gap between requests exceeds the TTL, every turn re-writes the cache. Longer TTLs usually cost more per write, so they only pay when they prevent misses.
- Fix: choose TTL from the *distribution* of start-to-start gaps (use `cost_model.py simulate-cache`, Appendix G), not the average. Consider keep-alive/pre-warm where the provider supports it.
- Type: Free win (tradeoff only in write premium; compute both).

**CACHE-06 Tool set or tool definitions vary per request**
- Signals: tools filtered per request (`tools=[t for t in ALL if relevant(t)]`); tool descriptions containing dynamic data; MCP tool lists reloaded in varying order.
- Why: tools usually render first, so any change invalidates the whole cache.
- Fix: send a stable full tool list and restrict with the provider's "allowed tools"/`tool_choice` mechanism, or deferred loading/tool search (INPUT-07). Keep dynamic data out of descriptions.
- Type: Free win.

**CACHE-07 Mid-conversation parameter/model switches**
- Signals: model, reasoning effort/thinking config, structured-output schema, or tools changed between turns of one conversation (e.g. router choosing a different model per turn).
- Why: caches are per model and often per parameter set; each switch cold-starts the cache and may drop prior reasoning.
- Fix: route per conversation or at natural boundaries (new session, after compaction), not per turn.
- Type: Free win. Skip when per-turn routing is measured to save more than the lost cache (rare; mark intentional with numbers).

**CACHE-08 History rewritten every turn**
- Signals: sliding window that drops the oldest message each turn (`messages[-N:]`); summarizing history every turn; re-injecting retrieved context into earlier messages; editing earlier turns.
- Why: a moving window changes the prefix every turn → no cache reuse at all, often costing more than keeping the history.
- Fix: append-only history; truncate/compact in large infrequent steps (or provider server-side compaction/context editing) so each compaction costs one cache miss rather than one per turn.
- Type: Free win.

**CACHE-09 Cache routing hints not used on high-volume shared prefixes**
- Signals: provider offers a cache key/affinity hint (e.g. OpenAI `prompt_cache_key` on some models) and a high-QPS route shares one long prefix across many users, or many distinct prefixes share one key.
- Why: requests may land on machines without the cached prefix; hit rate stays low despite stable prompts.
- Fix: set a stable key per shared prefix; partition very high-volume keys as the provider docs advise.
- Type: Free win. Verify current semantics in Appendix B (they changed across model generations).

**CACHE-10 Cache effectiveness never verified**
- Signals: cache markers/stable prefixes exist but cached-token usage fields are never read; no hit-rate metric.
- Why: silent invalidators (CACHE-01/02/06) are invisible from code review alone.
- Fix: log cached-read and cache-write tokens; alert when hit rate on a warm route drops. Include a two-request probe.
- Type: Observability.

**CACHE-11 Framework defeats caching**
- Signals: LangChain/LlamaIndex/CrewAI/AutoGen/Semantic Kernel/custom agent framework that injects scratchpads, timestamps, memory summaries or agent "role" text at the top of the prompt; prompt templates that re-render the whole conversation; framework without support for the provider's cache markers.
- Why: framework defaults often place volatile content first.
- Fix: inspect the final rendered request (enable debug logging or a request hook), then reorder, configure the framework's cache support, or bypass the framework on the hot path.
- Type: Free win.

### 3. RESP - Application-level response caching & dedup

**RESP-01 Identical requests recomputed**
- Signals: deterministic tasks (classification, extraction, translation, summarization of the same document, tagging) called repeatedly on the same input; no memoization keyed on (model, prompt, params, input hash).
- Why: you pay full price for an answer you already have.
- Fix: exact-match cache (Redis/DB/KV) keyed on a hash of the normalized request; TTL or invalidation by source version.
- Type: Free win. Skip when outputs must vary (creative generation) or inputs are never repeated.

**RESP-02 Semantic cache candidate**
- Signals: FAQ-like traffic with many paraphrased repeat questions (support bots, internal Q&A).
- Why: large hit rates are possible, but a wrong-match serves a wrong answer.
- Fix: embedding-similarity cache with a conservative threshold, scoped per tenant/permission, with freshness invalidation; validate false-hit rate on real traffic before enabling.
- Type: **Tradeoff** (accuracy risk). Needs eval.

**RESP-03 Precomputable work done at request time**
- Signals: summaries/descriptions/tags of static content generated on every page view or request.
- Why: content that changes rarely is generated thousands of times.
- Fix: generate once on write/ingest (or nightly via batch), store, serve.
- Type: Free win.

**RESP-04 Repeated evaluation / judge calls**
- Signals: LLM-judge or eval runs re-scoring unchanged (input, output) pairs every CI run or every dashboard refresh.
- Fix: cache judge results keyed on (judge prompt version, input, output hash).
- Type: Free win.

**RESP-05 Batch job reprocesses unchanged inputs**
- Signals: scheduled jobs that regenerate summaries, tags or embeddings for **all** records every run, when only a fraction change (`updatedAt`, content hash and change feeds are available but unused).
- Why: cost scales with corpus size instead of change volume. Often the largest single lever for nightly jobs (e.g. 300 of 40,000 pages changed → about 99% fewer calls).
- Fix: process only new or changed items (content hash or `updated_at` watermark), keep previous outputs for the rest, and run a full refresh rarely if ever.
- Type: Free win.

### 4. INPUT - Input token hygiene

**INPUT-01 Bloated system prompt**
- Signals: system prompt > a few thousand tokens; duplicated instructions; prose restating tool schemas; "IMPORTANT/NEVER" walls written for older models; many few-shot examples where 1-3 would do; instructions for features the route doesn't use.
- Why: billed on every request (even cached reads cost something), and over-prescriptive prompts make newer models over-work, which increases output tokens too.
- Fix: dedupe, delete tool recaps, trim few-shots, split per-route prompts instead of one mega-prompt. Validate on eval if behavior-critical text is removed.
- Type: Free win (Tradeoff if removing behavior-critical instructions).

**INPUT-02 Large reference material inlined every call**
- Signals: whole manuals, policies, product catalogs, schemas, or codebases pasted into every prompt.
- Why: you pay for the whole document even when the question needs one section.
- Fix: retrieval (RAG) or a lookup tool/skill so the model fetches sections on demand. If most calls need most of the doc, keep it inline **but cached** instead.
- Type: Free win → Tradeoff if retrieval misses (validate).

**INPUT-03 Unbounded conversation history**
- Signals: entire chat history sent forever; no compaction, summarization or session cap.
- Why: per-turn cost grows linearly, total conversation cost roughly quadratically with turns.
- Fix: compaction/summary at a high threshold (in large infrequent steps, see CACHE-08), provider server-side compaction where available, session cap (OBS-06).
- Type: Free win (Tradeoff if summaries lose needed detail).

**INPUT-04 Verbose tool results**
- Signals: tools returning raw API JSON, full HTML, entire DB rows, logs, stack traces, base64 blobs; `get_all_*` tools; no `limit`/`fields`/pagination parameters.
- Why: tool results re-enter context and are re-sent every subsequent turn of an agent loop.
- Fix: return only needed fields; truncate/summarize large results; narrow accessors (`get_order(id)` over `list_orders()`); HTML → text/markdown; store blobs and return references.
- Type: Free win.

**INPUT-05 Images/PDFs at full resolution**
- Signals: screenshots/photos/scans sent at native resolution; vision `detail` set to high/auto everywhere; multi-page PDFs sent whole when one page is needed.
- Why: image tokens scale with pixel area, not information.
- Fix: downscale/crop to what the task needs; low-detail mode where accuracy holds; extract text from text-based PDFs locally; send only relevant pages.
- Type: Free win → Tradeoff for OCR-sensitive tasks (validate).

**INPUT-06 Large tool catalog sent every request**
- Signals: dozens of tools / several MCP servers attached to every request; tool schemas totalling thousands of tokens.
- Why: schemas are input tokens on every call and dilute tool selection accuracy.
- Fix: provider tool search / deferred loading where available; split agents by domain; trim schema descriptions. Avoid per-request filtering that breaks caching (CACHE-06).
- Type: Free win. Skip when total schema size is small (roughly < 10K tokens).

**INPUT-07 Unbounded user-supplied input**
- Signals: uploads, pasted documents, or scraped pages passed straight into the prompt without a size check.
- Why: one 300-page upload can cost more than a day of normal traffic; also a cost-abuse vector.
- Fix: count tokens first (provider token-count endpoint or tokenizer), then truncate/chunk/route to retrieval or reject with a message.
- Type: Free win / Policy.

**INPUT-08 Inefficient serialization**
- Signals: pretty-printed JSON (`indent=2`) in prompts; repeated keys in large arrays of objects; XML-heavy wrappers; long UUIDs/URLs repeated many times.
- Why: whitespace and repeated keys are tokens.
- Fix: compact JSON, tables/CSV for homogeneous rows, short local IDs mapped back after the response.
- Type: Free win (validate on extraction tasks: some models parse compact formats slightly worse).

### 5. RAG - Retrieval & embeddings

**RAG-01 Too much retrieved context**
- Signals: high `top_k` (e.g. 10-50 chunks) always sent; large chunk sizes; no reranking; duplicate/overlapping chunks; whole documents returned per hit.
- Why: retrieved context is often the biggest input line in RAG apps, and irrelevant chunks can also lower answer quality.
- Fix: rerank (cross-encoder or cheap model) and send top 3-8; dedupe overlaps; tune chunk size; relevance threshold. Validate retrieval recall on eval.
- Type: Tradeoff (low risk with eval).

**RAG-02 Re-embedding unchanged content**
- Signals: ingestion re-embeds the whole corpus on each run/deploy; no content hash or `updated_at` check.
- Fix: hash-based incremental embedding; batch embedding endpoints/tiers for backfills.
- Type: Free win.

**RAG-03 Oversized embedding model or dimensions**
- Signals: largest embedding model used for simple semantic search; full dimensions where the model supports reduced dimensions.
- Why: embedding cost, vector storage and search latency all scale up.
- Fix: smaller model or reduced dimensions; validate recall@k on eval queries.
- Type: Tradeoff.

**RAG-04 LLM-heavy query pipeline**
- Signals: per-query LLM calls for query rewriting, HyDE, multi-query expansion, LLM reranking, LLM-based "is this relevant" filtering, each on a frontier model.
- Why: 3-6 extra calls per user question.
- Fix: drop steps that the eval shows don't help; move survivors to a small model; replace LLM reranking with a cross-encoder.
- Type: Tradeoff.

**RAG-05 Embedding calls one-at-a-time**
- Signals: embedding inside a loop with a single input per request.
- Why: per-request overhead, rate-limit pressure, retries; batch/offline tiers often cheaper for backfills.
- Fix: batch inputs per request; use batch tiers for bulk ingestion.
- Type: Free win.

### 6. OUTPUT - Output & reasoning tokens

Output tokens typically cost several times more than input tokens, and reasoning/thinking tokens bill as output.

**OUTPUT-01 No output shape specified**
- Signals: open-ended prompts for tasks with a fixed answer shape; responses with preambles, restated questions, markdown essays parsed for one field.
- Why: you pay for words nobody reads.
- Fix: specify the exact shape with an example; "answer only"; structured outputs for machine-consumed responses.
- Type: Free win.

**OUTPUT-02 Output cap missing or mis-set**
- Signals: no max output tokens on a route where runaway output is possible; or a cap so tight that responses truncate (finish/stop reason = length/max_tokens) and get retried.
- Why: missing cap = no backstop against runaway generation; too tight = paid truncated attempts plus retries (no saving per solved task).
- Fix: set a generous backstop sized from observed p99 output; treat length-truncation as a failure signal to fix, not to retry blindly. Shorten output with prompt shape (OUTPUT-01), not the cap.
- Type: Free win.

**OUTPUT-03 Parse-and-retry instead of structured outputs**
- Signals: `json.loads` in a try/except that re-calls the model; regex extraction with retry loops; "return ONLY JSON" prompts plus repair calls.
- Why: each parse failure costs a full extra call.
- Fix: native structured outputs / JSON schema / strict tool schemas from the provider; validate once.
- Type: Free win.

**OUTPUT-04 Unused samples**
- Signals: `n>1`, `best_of`, or multiple parallel candidate generations where only the first is used.
- Fix: `n=1` unless candidates are actually compared/voted on.
- Type: Free win.

**OUTPUT-05 Reasoning effort / thinking higher than the task needs**
- Signals: reasoning models at high/max effort (or large thinking budgets) on classification, extraction, routing, formatting, short chat; effort never swept; effort unset where the model default is high.
- Why: reasoning tokens bill as output and can dwarf the visible answer.
- Fix: per-route effort sweep on eval (low → medium → high); keep high only where the curve is steep (hard coding/agentic tasks). Note: on some models effort/thinking can't be disabled or has a different default; verify per model.
- Type: Tradeoff.

**OUTPUT-06 Regenerating content that is mostly unchanged**
- Signals: "return the full document/file with edits applied"; rewriting entire JSON objects to change one field.
- Why: output tokens for content you already have.
- Fix: ask for diffs/patches/edit operations and apply locally; provider predicted-output features where available.
- Type: Free win (validate patch-apply reliability).

**OUTPUT-07 No early exit**
- Signals: model asked to explain at length why it can't do something; no sentinel for "not applicable".
- Fix: stop sequences/sentinels (e.g. `<NO_ANSWER>`), short refusal shape.
- Type: Free win.

**OUTPUT-08 Generation continues after the client is gone**
- Signals: streaming endpoints that don't abort the upstream request on client disconnect or user "stop"; background generations for closed tabs.
- Why: you pay for tokens nobody receives.
- Fix: propagate cancellation (abort controller / close stream) to the provider call.
- Type: Free win.

**OUTPUT-09 Output requested, then discarded**
- Signals: prompts that ask for intermediate content the code throws away: "first rewrite the page, then summarize", "explain your reasoning, then give LABEL:", restating the input. Then `split(...)[1]` or "last line" parsing.
- Why: output tokens are the most expensive kind, and here most of them are thrown away.
- Fix: ask only for the needed part (with structured outputs where possible). If visible reasoning measurably helped accuracy, use the provider's reasoning/effort control instead. Validate on eval.
- Type: Free win (validate).

**OUTPUT-10 Brittle response parsing causes paid retries**
- Signals:
  - `content[0].text` on models that can return thinking, tool-use or other non-text blocks first;
  - `split("MARKER")[1]` or last-line parsing;
  - `choices[0]` with no check;
  - a retry loop around the whole call that fires on parse errors.
- Why: a parse failure throws away a paid response and re-calls the model, sometimes up to the retry cap, and may still save nothing.
- Fix: select blocks by type (the first `text` block), use structured outputs, and retry parsing locally (or repair) instead of re-calling the model.
- Type: Free win.

### 7. MODEL - Model selection & routing

Model choice changes the capability ceiling. Do it after the free wins, and always against an eval.

**MODEL-01 One frontier model for every task**
- Signals: a single model constant used for chat, classification, routing, extraction, summarization, title generation, guardrails, judge.
- Why: simple sub-tasks rarely need frontier capability. Small models are often 10-50x cheaper per token.
- Fix: per-route model choice; move clearly simple routes (intent classification, titles, PII tagging, short extraction) to a small model after an eval confirms parity.
- Type: Tradeoff.

**MODEL-02 Outdated model generation**
- Signals: model IDs one or more generations behind the provider's current lineup; deprecated/retiring models.
- Why: newer models in the same tier are frequently cheaper per task and better; old ones also face retirement.
- Fix: migrate within tier and re-sweep effort; re-audit prompts written for the old model (INPUT-01). Check the provider's current lineup on the live models page; don't rely on memory.
- Type: Tradeoff (often net positive on both axes; still validate).

**MODEL-03 No routing for a heterogeneous workload**
- Signals: traffic mixes trivial and hard requests on one route (support chat: "reset password" vs complex billing disputes).
- Why: hard-case model price paid on easy cases.
- Fix (in order of simplicity): (1) same strong model at lower effort for easy cases; (2) rule/classifier-based router to small vs large model; (3) cascade: small model first, escalate on low confidence or failed validation. Measure cost per *resolved* task; routers add a call and can misroute. Route per conversation, not per turn (CACHE-07).
- Type: Tradeoff.

**MODEL-04 LLM used where code would do**
- Signals: LLM calls for regex-able extraction, date parsing, unit conversion, JSON validation, language detection, sorting, arithmetic, exact keyword matching, deduplication.
- Why: deterministic code is free, faster and more reliable.
- Fix: replace with code/library; keep LLM as fallback for the residue.
- Type: Free win (when exact-equivalent) / Tradeoff.

**MODEL-05 Self-hosted / open-weight candidate**
- Signals: very high-volume, narrow, stable task (classification, extraction, embeddings, moderation) with steady traffic; data-residency needs that already favor self-hosting.
- Why: at sustained volume, an open-weight model on owned/rented GPUs or a cheap hosted inference provider can undercut API pricing. Uber listed this.
- Fix: break-even analysis including GPU cost at real utilization, ops/on-call, latency, and quality on the eval. Recommend a pilot, not a migration.
- Type: Tradeoff. Skip at low or bursty volume.

**MODEL-06 Fine-tune / distill candidate**
- Signals: long few-shot prompts on a high-volume narrow task; a frontier model producing consistent outputs that could train a small one.
- Fix: distill to a small fine-tuned model; removes few-shot tokens too. Count training + hosting cost.
- Type: Tradeoff.

**MODEL-07 Model IDs hard-coded across the codebase**
- Signals: model strings repeated in many files; no central config per route.
- Why: not a direct cost, but it blocks routing, A/B tests and fast migration; drift leaves old expensive models in corners.
- Fix: one config (per route → model, effort, max tokens), overridable per environment.
- Type: Free win (enabler).

**MODEL-09 Deprecated model or scheduled price change**
- Signals: model IDs listed with shutdown dates on the provider's deprecations page, or with announced future prices on the pricing page.
- Why: a forced migration or a price increase changes the bill without any code change, and reopens every "intentional" decision tied to that model.
- Fix: plan the migration (same-tier successor first), re-run the eval, and re-confirm intent entries.
- Type: Policy.

**MODEL-08 Premium modes enabled by default**
- Signals: premium-priced speed/latency modes, priority tiers, long-context pricing tiers, or "pro"/max-reasoning variants used on all traffic.
- Fix: enable only on routes where the latency/quality gain is worth the premium.
- Type: Tradeoff.

### 8. AGENT - Agent loops & multi-agent

**AGENT-01 No iteration / token / cost budget per task**
- Signals: `while True` / `while not done` agent loops without max steps; frameworks with `max_iterations` unset or very high; no per-task token/cost budget.
- Why: runaway loops are the most common source of surprise bills.
- Fix: max steps, max total tokens or cost per task, wall-clock timeout; graceful stop with partial result. Provider task-budget features where available.
- Type: Policy / Free win.

**AGENT-02 Tool results accumulate across long loops**
- Signals: deep loops (10+ turns) where every tool result stays in history forever.
- Fix: return less per tool (INPUT-04); server-side context editing/compaction at a high threshold; subagents for bulky self-contained steps (AGENT-05). Avoid per-turn pruning (CACHE-08).
- Type: Free win.

**AGENT-03 Sequential calls that could be one, or parallel**
- Signals: separate LLM calls for steps that could be one prompt (e.g. classify then extract then format); planner calling the model per step for a fixed pipeline; parallel tool calls disabled; tool results returned in separate messages.
- Fix: merge fixed pipelines into one call with structured output; use a workflow (code-orchestrated) instead of an agent when steps are known in advance; enable parallel tool calls.
- Type: Free win (validate merged prompt quality).

**AGENT-04 Multi-agent where a single agent suffices**
- Signals: CrewAI/AutoGen/LangGraph setups with several agents passing full history to each other; "manager" agents for a linear task; group chat where every agent sees every message.
- Why: each agent re-sends large prefixes; coordination turns cost tokens; caches aren't shared across agents/models.
- Fix: single agent with tools for dependent work; keep multi-agent only for genuinely parallel, bulky, independent sub-tasks; pass summaries, not transcripts, between agents.
- Type: Tradeoff.

**AGENT-05 Subagent / cheaper worker opportunity**
- Signals: main agent reads lots of raw material (search results, files, pages) to extract a small answer.
- Fix: delegate the bulky step to a subagent (optionally a smaller model) that returns a short result.
- Type: Tradeoff. Skip when the main agent needs the raw intermediate context to decide well.

**AGENT-06 Unbounded self-critique / reflection loops**
- Signals: "critique and revise until perfect" loops; judge-and-retry without max attempts; verification calls on every step.
- Fix: cap revisions (1-2 usually captures most of the gain); verify with cheap deterministic checks (tests, schema, validators) where possible; re-run only failures at higher effort.
- Type: Free win (cap) / Tradeoff.

**AGENT-07 Verbose framework defaults**
- Signals: `verbose=True` prompts, ReAct scratchpads with long templates, memory modules that inject full summaries each step, default system prompts from frameworks (often thousands of tokens).
- Fix: inspect rendered prompts; override templates; disable unused memory/planning features.
- Type: Free win.

### 9. TIER - Throughput / pricing tier

**TIER-01 Offline work on the synchronous API**
- Signals: cron jobs, nightly reports, backfills, bulk classification/enrichment, eval runs, embedding ingestion, data labeling run through the normal real-time endpoint.
- Why: batch APIs on major providers are discounted substantially (commonly 50%; verify) and usually stack with caching.
- Fix: move to the provider's Batch API with results keyed by custom ID; keep user-facing work synchronous. Batch requests are single-shot (no mid-batch tool loops).
- Type: Free win. Skip when results are needed within minutes.

**TIER-02 Latency-tolerant traffic on standard tier where a cheaper tier exists**
- Signals: background-but-not-batchable work (async enrichment, internal tools, evals) on standard tier when the provider offers a discounted slower tier (e.g. OpenAI `service_tier="flex"`; verify).
- Fix: use the discounted tier with longer timeouts and a fallback to standard.
- Type: Free win (latency tradeoff only).

**TIER-03 Provisioned/reserved capacity mismatched to load**
- Signals: provisioned throughput units / reserved capacity with low utilization; or high steady on-demand volume that would be cheaper reserved.
- Fix: right-size from utilization metrics; move bursty traffic to on-demand.
- Type: Tradeoff (commitment risk).

### 10. RELY - Reliability waste

**RELY-01 Retrying non-retryable errors or unbounded retries**
- Signals: broad `except Exception: retry`; tenacity/backoff on all errors; retries on 400/validation errors; nested retries (SDK retries + app retries + framework retries multiplying).
- Why: retried requests that reach the model are billed; nested layers multiply attempts.
- Fix: retry only on 429/5xx/timeouts/connection errors with capped exponential backoff; one retry layer.
- Type: Free win.

**RELY-02 Timeouts shorter than generation time**
- Signals: client timeouts below the p99 latency of long generations (reasoning models, long outputs), followed by retries.
- Why: the provider may finish and bill the timed-out request; the retry bills again.
- Fix: stream long outputs; set timeouts from observed latency; don't retry a timed-out long generation blindly.
- Type: Free win.

**RELY-03 Duplicate calls from the app**
- Signals: React effects firing twice (StrictMode, missing deps), double-submit buttons without debounce, polling that re-generates, the same LLM call in a request handler and a background job, webhooks retried by the sender without idempotency keys.
- Fix: debounce, idempotency keys, request dedup, move generation to one place.
- Type: Free win.

**RELY-04 Hedged / parallel fallback calls**
- Signals: calling two models/providers simultaneously and taking the first; fallback chains triggered on slow responses rather than failures.
- Fix: sequential fallback on actual failure; hedging only where an explicit latency SLO justifies the cost.
- Type: Tradeoff.

**RELY-05 Event redelivery causes duplicate generations**
- Signals: LLM work done synchronously inside webhook or event handlers (Slack, Stripe, GitHub, queues with at-least-once delivery) before acknowledging; no idempotency key or dedup on the event ID. Answer caches written only after generation finishes.
- Why: senders retry unacknowledged events (Slack retries after about 3 seconds), so slow generations run 2-4 times in parallel and each is billed.
- Fix: acknowledge immediately, process asynchronously, dedupe on the event ID (or the `X-Slack-Retry-Num` header), and use an in-flight lock keyed on the request hash.
- Type: Free win.

### 11. DEV - Dev, test & CI usage

**DEV-01 Tests hit live LLM APIs**
- Signals: unit/integration tests instantiating real clients; no mocks or record/replay (VCR cassettes, recorded fixtures); LLM tests running on every push.
- Why: CI multiplies cost by every push from every engineer. On a 100-person team this is a real line item.
- Fix: mock/record-replay for unit tests; run live LLM evals on a schedule or on prompt/model changes only, preferably via batch.
- Type: Free win.

**DEV-02 Non-production environments use production-grade models/settings**
- Signals: dev/staging configs identical to prod (frontier model, high effort, full context); load tests against real models.
- Fix: cheaper defaults in dev; explicit opt-in for frontier models; mocked providers for load tests.
- Type: Free win / Policy. Skip when staging must mirror prod for evals.

**DEV-03 No shared org defaults**
- Signals: each service builds its own client, retries, logging and model choice.
- Why: every team re-learns the same lessons; fixes don't propagate.
- Fix: a shared internal LLM client/gateway with logging, retries, caching, budgets and routing config built in.
- Type: Policy (high leverage for large teams).

---

## Appendix B - Provider specifics

**Read this before citing any provider number or parameter.** Everything here is a *pointer*: what to look for in code, and where to verify. Prices, cache multipliers, minimum cacheable sizes, TTL options, tier names and even parameter names change between model generations, sometimes within months (e.g. OpenAI's cache controls differ between model generations; Anthropic's minimum cacheable prefix differs by model and is not monotonic across generations). So:

1. Identify provider + exact model per route (scanner output + code).
2. WebFetch the provider's **pricing** page and the **feature** page for each lever you're about to recommend.
3. Cite the fetched page and date beside every figure in the report. If a fetch fails, use relative buckets and say so.

Facts below were checked in October 2026. Treat them as orientation, not as a source.

### Contents
- Anthropic (Claude API, also via Bedrock / Vertex / Foundry)
- OpenAI (and Azure OpenAI)
- Google Gemini (Gemini API / Vertex AI)
- Amazon Bedrock (any model)
- Gateways & routers (LiteLLM, OpenRouter, Portkey, ...)
- Self-hosted / open-weight (vLLM, TGI, Ollama, SGLang)
- Common cross-provider facts

---

### Anthropic (Claude)

Code signals: `import anthropic`, `from anthropic import`, `@anthropic-ai/sdk`, `client.messages.create`, `claude-` model IDs, `AnthropicBedrock*`, `AnthropicVertex`, `AnthropicFoundry`, `langchain_anthropic`, `ChatAnthropic`.

- **Caching is explicit**: `cache_control: {"type": "ephemeral"}` on content blocks (limited number of breakpoints per request), or top-level automatic caching. No marker means no caching, so CACHE-03 is often the #1 finding on Claude apps. Default TTL is short (minutes); a longer (1-hour) TTL exists with a higher write multiplier. Writes cost more than base input, reads cost a fraction of it. Get the multipliers from the pricing page.
- **Minimum cacheable prefix is per model**. Below it, markers silently do nothing (`cache_creation_input_tokens: 0`).
- **Usage fields**: `usage.input_tokens` (uncached), `usage.cache_creation_input_tokens`, `usage.cache_read_input_tokens`, `usage.output_tokens`. Hit-rate check: on a warm loop, cache reads should dominate input.
- **Cache breakers specific to Claude**: changing `thinking`/top-level `effort`, tools, or model mid-conversation; editing earlier turns (on newer models this also invalidates preserved thinking blocks). Mid-conversation `role: "system"` messages exist on newer models as an append-only way to add instructions without editing `system`.
- **Effort** (`output_config.effort`): main control over thinking/output spend on current models. Defaults differ by model, and some models reject disabled thinking. Check per model.
- **Batch**: Message Batches API, discounted (50% at time of writing), stacks with caching, results within 24h.
- **Other levers**: token counting endpoint (ingestion gate, INPUT-07), tool search with `defer_loading` (INPUT-06), server-side compaction and context editing (AGENT-02/INPUT-03), task budgets (AGENT-01), structured outputs `output_config.format` (OUTPUT-03), Files API + code execution for large tables (INPUT-02), premium "fast mode" (MODEL-08: only where latency is worth 2x+).
- **Spend data**: Usage & Cost Admin API (needs an Admin key) gives the measured token/cost profile per model/key/workspace.
- If the Claude Code `claude-api` skill is available, its `cost-optimize` and `prompt-caching` guides go deeper for Claude-only projects. Use them for Claude routes.

Verify at:
- Pricing: https://platform.claude.com/docs/en/about-claude/pricing.md
- Models: https://platform.claude.com/docs/en/about-claude/models/overview.md
- Cost optimization (measured levers): https://platform.claude.com/docs/en/about-claude/models/optimizing-for-cost-and-intelligence.md
- Prompt caching: https://platform.claude.com/docs/en/build-with-claude/prompt-caching.md
- Batch: https://platform.claude.com/docs/en/build-with-claude/batch-processing.md
- Effort: https://platform.claude.com/docs/en/build-with-claude/effort.md
- Token counting: https://platform.claude.com/docs/en/build-with-claude/token-counting.md
- Tool search: https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-search-tool.md
- Compaction / context editing: https://platform.claude.com/docs/en/build-with-claude/compaction.md, https://platform.claude.com/docs/en/build-with-claude/context-editing.md
- Usage & cost API: https://platform.claude.com/docs/en/manage-claude/usage-cost-api.md

### OpenAI (and Azure OpenAI)

Code signals: `from openai import`, `OpenAI(`, `AzureOpenAI(`, `client.chat.completions.create`, `client.responses.create`, `gpt-`, `o1`/`o3`/`o4-` model IDs, `langchain_openai`, `ChatOpenAI`, `AzureChatOpenAI`, JS `openai` package.

- **Caching is automatic** on supported models above a minimum prefix length (1,024 tokens on current models; earlier models varied). Your job is prefix stability (CACHE-01/02/06/08), not markers. Newer generations add explicit breakpoints and TTL options (`prompt_cache_options`), and some charge for cache writes. Older ones use `prompt_cache_retention` (`in_memory` / `24h`) with no write charge. `prompt_cache_key` improves routing to warm caches for shared prefixes (CACHE-09); its role differs by model generation. Check the docs for the model in use.
- **Usage fields**: Chat Completions → `usage.prompt_tokens_details.cached_tokens`, `usage.completion_tokens_details.reasoning_tokens`. Responses API → `usage.input_tokens_details.cached_tokens`, `usage.output_tokens_details.reasoning_tokens`. Newer models also report cache-write tokens.
- **Output cap parameter** differs by API/model: `max_tokens` (legacy chat), `max_completion_tokens` (reasoning models on Chat Completions), `max_output_tokens` (Responses). Reasoning tokens count against it.
- **Reasoning effort**: `reasoning_effort` (Chat Completions) / `reasoning: {effort: ...}` (Responses), plus `verbosity` on some models (OUTPUT-05, OUTPUT-01).
- **Tiers**: Batch API (discounted, 24h window); `service_tier="flex"` (priced at batch rates, slower, may be unavailable, beta, limited models: TIER-02); `service_tier="priority"` is a *premium* (MODEL-08).
- **Other levers**: Structured Outputs / `strict` function schemas (OUTPUT-03); Predicted Outputs for edits (OUTPUT-06; check model support); image `detail: "low"` (INPUT-05); `allowed_tools` / `tool_choice` instead of removing tools (CACHE-06); embeddings `dimensions` parameter on supporting models (RAG-03).
- **Azure OpenAI**: same model behavior, but features and prices depend on deployment type (Standard / Global / Data Zone / Provisioned / Global Batch) and API version. Check the deployment type and Azure pricing, not openai.com. PTU utilization → TIER-03.

Verify at:
- Pricing: https://developers.openai.com/api/docs/pricing (or https://openai.com/api/pricing/)
- Prompt caching: https://developers.openai.com/api/docs/guides/prompt-caching
- Batch: https://developers.openai.com/api/docs/guides/batch
- Flex processing: https://developers.openai.com/api/docs/guides/flex-processing
- Priority processing: https://developers.openai.com/api/docs/guides/priority-processing
- Reasoning: https://developers.openai.com/api/docs/guides/reasoning
- Azure OpenAI pricing: https://azure.microsoft.com/pricing/details/cognitive-services/openai-service/

### Google Gemini (Gemini API / Vertex AI)

Code signals: `from google import genai`, `google.generativeai`, `vertexai`, `GenerativeModel(`, `generate_content`, `@google/genai`, `gemini-` model IDs, `langchain_google_genai`, `ChatVertexAI`, `ChatGoogleGenerativeAI`.

- **Implicit caching** is on by default for Gemini 2.5+ models, with per-model minimum prefix sizes (thousands of tokens; check the docs). Savings are automatic on hits, so prefix stability is what matters.
- **Explicit caching** (`cachedContents` / `client.caches.create`, used via `cached_content=`) exists on supported models/APIs: cache creation is billed, plus **storage per token-hour** for the TTL (default 1h if unset). It pays only when the cached content is reused enough within the TTL. Do the break-even (`cost_model.py`). Not available on every API surface or newest model. Check.
- **Usage fields**: `usage_metadata.cached_content_token_count` (generateContent; REST `cachedContentTokenCount`), `usage_metadata.thoughts_token_count` for thinking. The Interactions API reports `usage.total_cached_tokens`. Verify against the SDK in use.
- **Thinking controls**: `thinking_config` (`thinking_budget` on 2.5-era models, `thinking_level` on newer ones). Check per model (OUTPUT-05).
- **Batch mode**: discounted (50% at time of writing), up to 24h.
- **Vertex AI** has its own pricing page and may differ in feature availability (provisioned throughput → TIER-03).

Verify at:
- Pricing: https://ai.google.dev/gemini-api/docs/pricing
- Caching: https://ai.google.dev/gemini-api/docs/caching
- Optimization overview (batch/caching/flex comparisons): https://ai.google.dev/gemini-api/docs/optimization
- Batch: https://ai.google.dev/gemini-api/docs/batch-mode
- Thinking: https://ai.google.dev/gemini-api/docs/thinking
- Vertex pricing: https://cloud.google.com/vertex-ai/generative-ai/pricing

### Amazon Bedrock

Code signals: `boto3.client("bedrock-runtime")`, `converse(`, `converse_stream(`, `invoke_model(`, `ChatBedrock`, `ChatBedrockConverse`, `anthropic.` / `amazon.nova` / `meta.llama` model IDs, `AnthropicBedrockMantle`.

- **Prompt caching** is explicit: `cachePoint` blocks in Converse (or the model's native markers via InvokeModel), supported only on specific models, with model-specific minimums. Usage fields: `usage.cacheReadInputTokens`, `usage.cacheWriteInputTokens` (Converse).
- **Batch inference** (S3 in/out, discounted vs on-demand for supported models), **Intelligent Prompt Routing** (routes within a model family by predicted quality/cost: MODEL-03; validate on eval), **cross-region inference profiles** (capacity, not price), **Provisioned Throughput** (TIER-03).
- Bedrock pricing for Claude/other partner models is set by AWS and can differ from first-party pricing.

Verify at:
- Pricing: https://aws.amazon.com/bedrock/pricing/
- Prompt caching: https://docs.aws.amazon.com/bedrock/latest/userguide/prompt-caching.html
- Batch inference: https://docs.aws.amazon.com/bedrock/latest/userguide/batch-inference.html
- Prompt routing: https://docs.aws.amazon.com/bedrock/latest/userguide/prompt-routing.html

### Gateways & routers

Code signals: `litellm`, `completion(model="...")`, `OPENAI_BASE_URL`/`base_url=` pointing at a proxy, `openrouter.ai`, `portkey`, `helicone`, `langfuse`, `langsmith`, Vercel `ai` SDK (`generateText`, `streamText`), Cloudflare AI Gateway.

- Gateways often already provide: usage/cost logs (OBS-01/02), budgets (OBS-03), exact-match caching (RESP-01), fallbacks (RELY-04), routing (MODEL-03). Check what's configured before recommending a new tool.
- Watch for gateways that **strip provider cache markers** or rewrite prompts (CACHE-11), and for double retry layers (gateway + SDK + app → RELY-01).
- Through OpenAI-compatible shims, provider-native features (Anthropic `cache_control`, Gemini explicit caches, batch APIs) may be unavailable or need provider-specific passthrough fields. Verify in the gateway docs.

### Self-hosted / open-weight

Code signals: `vllm`, `text-generation-inference`, `ollama`, `sglang`, `llama.cpp`, `base_url="http://localhost:...` / internal hosts, `meta-llama/`, `Qwen/`, `mistralai/`, `deepseek` model names, `transformers` pipelines.

- Cost is GPU-hours, not tokens: the levers are **utilization** (continuous batching, right-sized instances, autoscale to zero for bursty loads), **quantization** (validate quality), **automatic prefix caching** in the server (e.g. vLLM `enable_prefix_caching`; check version defaults), and **model size**.
- For MODEL-05 break-even: monthly GPU cost ÷ tokens actually served at real utilization, vs API cost for the same tokens, plus ops/on-call and an eval-measured quality delta.

### Common cross-provider facts (stable enough to rely on, still verify numbers)

- Output tokens cost several times more than input tokens on virtually every hosted model; reasoning/thinking tokens bill as output.
- Caching is prefix-based everywhere: stable content first, volatile content last, append-only history.
- Batch tiers on the major providers are roughly half price with up to a 24h window, for work nobody is waiting on.
- Price per token does not predict cost per task: verbosity, reasoning length, turns-to-solve and retries differ by model. Compare on the project's own eval.

---

## Appendix C - `.llm-cost.yaml`: intent, policy, context

Lives at the repo root (commit it). Three optional top-level sections. A team can also keep an org-wide file (e.g. in a shared config repo) and copy or merge its `policy` into each project. When both exist, the project file wins for `intentional`/`context`, and the **stricter** value wins for `policy`.

```yaml
version: 1

policy:                      # hard constraints - never recommend violating these
  approved_models:           # if set, only recommend models from this list (globs ok)
    - "claude-*"
    - "gpt-*-mini"
  forbidden_models: []
  data_residency: "eu"       # rules out providers/regions/tiers that can't meet it
  zero_data_retention: true  # rules out features incompatible with ZDR (some caches/retention modes)
  self_hosting_allowed: false
  max_session_context_tokens: 400000   # session cap (OBS-06)
  monthly_budget_usd: 20000
  latency_slo_ms:            # per route; rules out batch/flex where violated
    chat: 3000
  quality_floor:             # tradeoffs must keep these eval scores
    support-answer: 0.92

context:                     # facts the code can't show - used for sizing
  routes:
    chat:
      requests_per_day: 50000
      user_waiting: true
      median_gap_between_turns_s: 90
    nightly-summary:
      requests_per_day: 8000
      user_waiting: false
  monthly_bill_usd: 14000
  usage_logs: "s3://.../llm-usage/"   # where measured data lives, if any

intentional:                 # acknowledged findings - reported, not re-argued, not counted
  - id: MODEL-01                       # check ID from checks.md (or CUSTOM-nn)
    path: "src/legal/**"               # glob, relative to repo root
    symbol: "review_contract"          # optional: function/class/route name to narrow the match
    reason: "Contract review must use frontier model - legal sign-off 2026-09"
    owner: "priya@company.com"
    expires: "2027-03-31"              # optional ISO date; after it, re-confirm
  - id: CACHE-01
    path: "src/chat/prompt.py"
    reason: "Timestamp needed in system prompt for compliance logging"
    owner: "team-chat"
```

### Matching rules (how the skill applies `intentional`)

A finding is **intentional** when all hold:
1. `id` equals the finding's check ID (or `id: "*"` to acknowledge every check for that path, which should be rare; flag it as a smell if broad).
2. The finding's file matches `path` (glob; `**` matches across directories).
3. If `symbol` is given, the finding is inside that function/class/route.

Then:
- Put it in the "Acknowledged as intentional" table with the reason and owner. Exclude it from savings totals and the main table.
- If `expires` is in the past → "Intent needs re-confirmation" (still excluded from totals, but asks the owner to renew or drop it).
- If the code at that location changed in a way that undermines the reason (e.g. the legal route now also handles marketing copy), list it as "Intent may be stale" with what changed. Don't silently override.
- Entries that match nothing → list under "Unused intent entries" so the file stays clean.

### Inline comments

For a single line or call site, a comment on the same line or the line directly above:

```python
## llm-cost: intentional MODEL-01 - frontier model required for medical triage accuracy
resp = client.responses.create(model="gpt-5.5", ...)
```

```ts
// llm-cost: intentional CACHE-01 - per-request nonce required by downstream audit
```

Syntax: `llm-cost: intentional <ID>[,<ID>...] - <reason>`. Any comment style works (`#`, `//`, `--`, `/* */`). A comment without a reason is still honored, but reported as "intent without reason" so someone can fill it in.

### Asking about intent (always ask; never infer)

The skill never marks a finding intentional on its own judgment. When a finding looks deliberate, or might have a reason the code doesn't show, it asks the user (structured question tool if available, e.g. `AskUserQuestion`; otherwise a numbered list in chat), up to 4 findings per batch:

> **CACHE-01** in `src/chat/prompt.py:42`: `datetime.now()` in the system prompt prevents caching (est. −35-45% of chat route). Is this intentional?
> - No, optimize it
> - Yes, it's intentional (I'll give the reason)
> - Not sure, keep as an open question

Remaining plain-waste findings get one multi-select question: "Are any of these intentional?" For each "Yes", collect the reason (required), owner and optional expiry, then ask once whether to save the entries to `.llm-cost.yaml`.

### When the user marks something intentional

Add a yaml entry (preferred for anything route-wide) with `id`, `path`, optional `symbol`, the user's reason in their words, `owner` if known (ask once), and an `expires` date if they say it's temporary. Show the exact entry you added.

---

## Appendix D - Report template

Keep the section order and the main table columns; everything else is flexible. Be concrete: `file:line`, the route, and what to change. A finding a reader can't act on is noise.

```markdown
## LLM Cost Review: <project name>
_Date: YYYY-MM-DD · Reviewer: llm-cost-optimizer skill · Commit: <sha if available>_

### Summary
- Estimated impact: <total, with data-source label: measured $ / % of bill / High-Med-Low buckets>
- Top actions: 1) ... 2) ... 3) ...
- Quality gate: <eval found at ... | NONE - tradeoffs marked "needs eval">
- Decisions needed: <short list>

### Project model
Route cards (condensed; full cards in appendix):

| Route | Purpose | Input → expected output | Provider / model | Shape (single / chain / agent, depth) | User waiting? | Volume | Quality check |
|---|---|---|---|---|---|---|---|

Platform: ... · Measurement available: ... · Policy constraints applied: ... · Assumptions: ...

### Suggestions & findings (ranked by estimated impact)
| # | ID | Finding | Route | Location | Type | Suggested change | Est. reduction (assumption) | Measured reduction | Quality Δ (measured) | Effort | Status |
|---|---|---|---|---|---|---|---|---|---|---|---|
| 1 | CACHE-01 | `datetime.now()` in system prompt → 0% cache hits | chat | `src/chat/prompt.py:42` | Free win | Move timestamp into latest user message | ~35-45% of chat route (~20% of total). Basis: 4.5K-token stable prefix of 6K input, cache-read rate from pricing page (2026-10-21) | −39.2% cost/task (24 cases × 3 trials, significant); ≈ $1.7K/mo at 1.5M tasks | 100% → 100%, 0 regressions | S | Measured-kept |

_Estimate basis per row: measured basis / partial data / assumption. Overlapping levers deflated so totals can't exceed 100%. Measured numbers come from `llm_eval.py compare` with one prices file for all runs._

**Totals:** estimated <x–y%> of LLM spend · measured so far <z%> on routes <...> (see COMPARISON.md)

### Recommended order of work
1. Observability first (if missing): OBS-01 ... → gives the baseline.
2. Free wins: ... (one diff each)
3. Tradeoffs, each behind its validation plan: ...

### Measurement
- Harness: `llm_cost_eval/` (cases per route, checks, prices.json with source + date)
- Runs: baseline + one `fix-<ID>` per lever; comparison in `llm_cost_eval/COMPARISON.md`
- Spend on measurement so far: $<...> (approved budget $<...>)

#### Validation plan for not-yet-measured tradeoffs
| ID | What to compare | Cases / checks | Keep criterion | Est. measurement cost |
|---|---|---|---|---|

### Questions for the team
- <check ID> <question whose answer would confirm or kill a finding>

### Acknowledged as intentional
| ID | Location | Reason (from user) | Owner | Expires | Confirmed how (yaml / comment / asked this run) | Status (ok / expired / may be stale) |
|---|---|---|---|---|---|---|

### Blocked by policy / skipped
| ID | Why skipped (policy, platform availability, workload shape) |
|---|---|

### Considered, not worth it
One line each: lever, why its ceiling is too small here.

### Appendix
- Scanner summary (call sites, models, prompt sizes)
- Pages fetched for prices/parameters, with dates
- Web sources used beyond the check catalog (what each informed)
- Full route cards
- Not applicable categories (one line each)
```

Notes:
- Never mix units in the impact column. If one route is measured and another estimated, say which per row.
- If nothing clears the bar, the Findings section says "No changes recommended" with the reason. That's a valid result.

---

## Appendix E - Measuring real cost

Goal: replace "this should save ~40%" with "this saved 39.2% per task (±2%) with pass rate unchanged on 24 cases × 3 trials", using the provider's own usage numbers.

### Contents
1. How metering works (and its limits)
2. Designing test cases
3. Writing quality checks
4. Wiring targets (Python, other languages, agents, streaming)
5. prices.json
6. Running: budget, trials, noise
7. Measuring specific levers (caching, batch, model swap, effort, RAG)
8. Reading the comparison & deciding
9. Keeping it: CI cost-regression check
10. Troubleshooting

---

### 1. How metering works

- `llm_meter.py` patches `httpx.Client/AsyncClient.send`, `requests.Session.send` and botocore's `_make_api_call` (Bedrock). These are the transport layers under the OpenAI, Azure OpenAI, Anthropic, Google GenAI, Mistral, Groq and Together SDKs, and under LiteLLM, LangChain, LlamaIndex, CrewAI etc. Each LLM POST (`/chat/completions`, `/responses`, `/messages`, `:generateContent`, `/embeddings`, `/converse`, `/invoke`, ...) becomes one JSONL record with normalized tokens:
  - `input`: uncached input, billed at the base rate;
  - `cache_read`, `cache_write` (default TTL), `cache_write_long` (1h/12h/24h);
  - `output` (includes reasoning); `reasoning` (informational subset).
- Normalization handles each provider's semantics. These are **not** the same and naive sums are wrong:
  - OpenAI `prompt_tokens` / `input_tokens` **include** cached tokens. The meter subtracts cached reads (and cache writes where reported).
  - Gemini `promptTokenCount` **includes** cached content. The meter subtracts `cachedContentTokenCount`; `thoughtsTokenCount` is billed as output and added.
  - Anthropic `input_tokens` **excludes** cache reads/writes (they're separate fields; 5m vs 1h writes are split via `cache_creation`).
  - Bedrock Converse `inputTokens` **excludes** `cacheReadInputTokens` / `cacheWriteInputTokens`.
- Streaming (SSE / NDJSON / streamed JSON arrays) is teed and the final usage is extracted. **OpenAI Chat Completions streams include usage only if the request sets `stream_options={"include_usage": True}`.** Otherwise the record has `usage_found: false` and the report warns about undercounting. Fix this for the eval (and consider it an OBS-01 finding).
- Calls are tagged with `<route>::<case>` and trial via a context variable. There's a process-wide fallback for worker threads, so run cases sequentially (the runner does).
- Cost is computed **at report time** from `prices.json`, never stored, so re-pricing is free and before/after use identical rates.

**Not covered in-process**: aiohttp-only clients, gRPC clients (legacy `google-generativeai`), non-Python code. Use the proxy (§4). Batch API jobs are not metered live (§7).

### 2. Designing test cases

Cases are JSONL lines: `{"id", "input", "expected"?, "checks"?, "tags"?}`. `input` is passed to the target (`kwargs` → `fn(**input)`).

Sources, in order of preference:
1. Existing tests, fixtures, eval sets, golden files in the repo.
2. Real inputs from logs or tickets, with PII removed. These represent the real distribution.
3. Synthesized from the route card. Cover:
   - **typical** inputs (most of the volume);
   - **hard/tail** inputs: long documents, multi-step questions, ambiguous requests, tool-heavy tasks. Cost is decided here, and cheap models fail here;
   - **edge** inputs: empty/huge/malformed input, other languages, refusal-worthy input;
   - **multi-turn** inputs where the route is conversational: put the prior turns in the input so history growth and caching are exercised.

Make cases behave like production: vary per-user values (names, IDs), use distinct conversations, and set `pace_ms` on the route when prompts contain time-based values. Otherwise fast back-to-back runs within the same second hide timestamp cache-breakers, and the baseline shows a cache hit rate production never gets.

Size: 15-30 per important route for per-lever decisions; around 50 for a final production cutover decision. Tag cases (`typical`, `hard`, `edge`) so regressions are readable. Freeze the set: every run uses identical cases. Show the user a sample and confirm the cases represent real traffic.

If the route has side effects (sends email, writes DB, charges cards), stub those tools or point at a sandbox **before** running. A cost eval must never mutate production.

### 3. Writing quality checks

Derive them from the route card's success criteria. Cheapest reliable first:

| Need | Check |
|---|---|
| Fixed answer (classification, extraction field) | `equals_expected`, `json_field_equals` |
| Must mention key facts | `contains_all` / `contains_any` |
| Must not leak / hallucinate X | `not_contains`, `regex` |
| Machine-readable output | `json_valid`, `json_has_keys` |
| Length/shape constraints | `max_chars`, `min_chars` |
| Close to a reference answer | `similar_to_expected` (threshold) |
| Domain logic (tests pass, SQL runs, numbers add up) | `python` → `checks.py:fn(output, case)` |
| Open-ended quality | `python` check that calls an LLM judge with a rubric. Last resort. Its cost is tagged `::judge` and excluded from task cost; use a fixed judge model and prompt |

Checks must pass on the baseline for cases that are "correct today". A check that fails on the baseline means either the code has a real bug (report it as a finding) or the check is wrong.

### 4. Wiring targets

**Python (preferred):** `"target": {"type": "python", "callable": "app.chat:answer", "call": "kwargs"}`. Use the real entry function (the one the API handler calls), not a lower-level helper, so agent loops, retries, repair calls and sub-agents are all counted. If the entry needs setup (DB, env, auth stubs), add `"setup": "checks:setup_chat"`. `checks:` and any other module in the eval dir import directly; the eval dir is on `sys.path`. Async functions and generators are handled.

**Python CLI/script:** `{"type": "command", "cmd": ["python", "-m", "app.cli", "answer"], "stdin": "json"}`. Python children are auto-metered through a generated `sitecustomize`.

**Any other language (Node, Go, Java, .NET, Ruby):**
1. Start the proxy: `python3 "$LCO"/llm_cost_proxy.py --log llm_cost_eval/runs/proxy.calls.jsonl --port 8787 --route openai=https://api.openai.com --route anthropic=https://api.anthropic.com`
2. Write a tiny runner in the project's language that reads the case input JSON from stdin, calls the route's entry function, and prints the output (JSON or text).
3. Target: `{"type": "command", "cmd": ["node", "llm_cost_eval/run_case.mjs"], "env": {"OPENAI_BASE_URL": "http://127.0.0.1:8787/openai/v1"}, "proxy": "http://127.0.0.1:8787", "proxy_log": "runs/proxy.calls.jsonl"}`. The runner tells the proxy which case is running before each call.

Base-URL variables: OpenAI SDKs `OPENAI_BASE_URL` (path must end in `/v1` → `/openai/v1`); Anthropic SDKs `ANTHROPIC_BASE_URL` (→ `/anthropic`); Gemini via client `http_options.base_url`; Azure: add `--route azure=https://<resource>.openai.azure.com` and set the endpoint to `http://127.0.0.1:8787/azure`. Check the SDK docs if a client ignores env vars.

**Agents with tools:** run the real loop, but stub tools that hit external systems with recorded fixtures, so tool output (and thus token growth) is realistic and repeatable.

### 5. prices.json

```json
{"source": "https://<provider pricing page>", "fetched": "2026-10-21", "currency": "USD", "unit": "per 1M tokens",
 "models": {"gpt-5.5*": {"input": 0, "output": 0, "cache_read": 0, "cache_write": 0},
            "claude-sonnet-5-5": {"input": 0, "output": 0, "cache_read": 0, "cache_write": 0, "cache_write_long": 0}},
 "tier_multipliers": {"standard": 1.0, "batch": 0.5}}
```

- Fill in **verified** numbers from the live pricing pages for every model in the runs, including candidate models. Model keys match exact IDs or `prefix*` patterns. The response's model name is tried first (e.g. a dated snapshot), then the requested one.
- `cache_write` is the default-TTL write rate. Omit it where writes carry no premium; it falls back to the input rate. `cache_write_long` is the long-TTL write rate.
- Long-context surcharges, regional premiums, priority/fast tiers: if the project uses them, add a separate model key or multiplier and say so in the report.
- The report shows the prices source and date on every table.

### 6. Running: budget, trials, noise

- `estimate` first. Without a previous run it gives the number of task executions. Do a 1-case × 1-trial probe (`run --limit 1 --trials 1 --yes --label probe`, with approval) to get a real per-task cost, then `estimate --from-run`.
- Set `max_cost_usd` in `eval.json`. The run aborts when spend crosses it.
- `trials ≥ 3`. The runner executes trial-major (every case once, then again) so trial 0 sees cold caches and later trials see warm ones, like production. Use `--skip-first-trial` in `report`/`compare` for the steady-state view.
- Noise: `compare` flags a cost delta as "within noise" when it's under 2 standard errors. Pass-rate differences of one case are noise on small sets; repeat trials before deciding. Regressions are judged per case by majority across trials.
- Keep runs reproducible: same cases, same prices, same trials. Note the git commit (recorded automatically; "dirty" means uncommitted changes, so record which diff was applied in `--note`).

### 7. Measuring specific levers

- **Prompt caching (CACHE-*)**: measured directly (cache-read and cache-write tokens appear in usage). Run cases sequentially and include multi-turn cases. Report both all-trials and `--skip-first-trial`. Real traffic gaps decide the production hit rate: combine the measured per-hit savings with `cost_model.py simulate-cache` on real timestamps.
- **Batch API (TIER-01)** and **flex tiers**: async/slow, so not measured live. Run the cases synchronously to measure tokens, then price them with the batch multiplier (`tier_multipliers`) from the verified pricing page. State "tokens measured, price = batch rate" in the table. Batch also changes latency, not quality.
- **Model swap / routing (MODEL-*)**: change the model (or router) and run. Quality checks decide. Look at `hard`-tagged cases separately: cheap models fail in the tail.
- **Reasoning effort / thinking (OUTPUT-05)**: one run per level (`fix-OUTPUT-05-low`, `-medium`), byte-identical otherwise. Read the curve: flat means take the lower level; steep means keep it.
- **RAG top-k / reranking (RAG-*)**: include answer-quality checks that depend on retrieval (facts only present in specific chunks).
- **Agent caps (AGENT-01)**: include a case known to loop long. Watch `calls/task`, the p90 cost and the pass rate.
- **Output shape (OUTPUT-01/03)**: `output tok/task` and the truncation count show the effect.

### 8. Reading the comparison and deciding

Keep a change when all hold:
- Cost/task drops beyond noise (all-trials, and warm-trials for caching).
- Pass rate holds; no case regresses consistently (check the "regressed on" list and read those outputs).
- Latency stays within the route's SLO.
- Truncations don't increase.

Write the measured % and the quality delta into the report table next to the original estimate. If they differ a lot, explain why. That calibrates future estimates.

### 9. Keeping it: CI cost-regression check

Offer to add a scheduled or on-prompt-change CI job:
- runs a small frozen subset (`--route`, `--limit`, `--trials 1`) with a strict `max_cost_usd`;
- runs `compare` against a stored baseline;
- fails if cost/task rises more than X% or the pass rate drops.

Unit tests should still mock LLM calls (DEV-01). This job is the deliberate, budgeted exception.

### 10. Troubleshooting

| Symptom | Cause / fix |
|---|---|
| 0 calls recorded | Client not on httpx/requests/botocore (aiohttp, gRPC) → use the proxy. Or the host isn't recognized → set `LLM_METER_EXTRA_HOSTS=myhost` (or `LLM_METER_ANY_HOST=1`) |
| `usage_found: false` | OpenAI stream without `include_usage`, or a provider that doesn't return usage on that endpoint |
| Unpriced model warning | Add the response model ID or a `prefix*` pattern to prices.json |
| Calls tagged with the wrong case | The target ran work in background threads/tasks that outlived the case. Make the entry function wait for completion |
| Cache hit 0% after a caching fix | A silent invalidator remains (CACHE-01/02/06), the prefix is below the minimum, or calls are farther apart than the TTL |

---

## Appendix F - Starter `.llm-cost.yaml`

Copy to the repo root as `.llm-cost.yaml` and commit it.

```yaml
# Copy to your repo root as .llm-cost.yaml. All sections optional.
# Schema & matching rules: Appendix C of the llm-cost-optimizer SKILL.md.
version: 1

policy:
  approved_models: []          # e.g. ["claude-*", "gpt-*-mini", "gemini-*-flash"]
  forbidden_models: []
  data_residency: null         # e.g. "eu", "us"
  zero_data_retention: false
  self_hosting_allowed: true
  max_session_context_tokens: 400000
  monthly_budget_usd: null
  latency_slo_ms: {}           # route: ms
  quality_floor: {}            # route: min eval score

context:
  routes: {}
  #  chat:
  #    requests_per_day: 0
  #    user_waiting: true
  #    median_gap_between_turns_s: 60
  monthly_bill_usd: null
  usage_logs: null

intentional: []
#  - id: MODEL-01
#    path: "src/legal/**"
#    symbol: null
#    reason: "why this is deliberate"
#    owner: "team or email"
#    expires: "2027-01-01"
```

---

## Appendix G - Embedded tools (extract with *Tools setup*; don't retype)

The files are embedded verbatim. If you change one, update its `sha256=` marker (or the extractor will warn), and bump `metadata.version`.

### `scan_llm_usage.py`

<!-- file: scan_llm_usage.py sha256=20176c7fb01f2dfde12089a0d5aaa2c467b48f0c9b90ae132293c35afc15af34 -->
`````python
#!/usr/bin/env python3
"""Deterministic inventory of LLM usage in a codebase (stdlib only).

Heuristic pre-pass for the llm-cost-optimizer skill: finds providers, frameworks,
LLM/embedding call sites with nearby cost-relevant parameters, model strings,
large inline prompts and prompt files, dynamic values near prompts (cache
breakers), loops/retries around calls, live LLM calls in tests, usage-field
reads, and `llm-cost: intentional` comments.

Every hit is a *lead*, not a finding - confirm by reading the code.

Usage:
  python3 scan_llm_usage.py <root> [--out scan.json] [--markdown] [--max-file-kb 1024]
"""
import argparse
import json
import os
import re
import sys
from collections import Counter, defaultdict

SKIP_DIRS = {
    ".git", "node_modules", ".venv", "venv", "env", ".env", "__pycache__", "dist", "build",
    ".next", ".nuxt", ".tox", "site-packages", "vendor", "target", "coverage", ".mypy_cache",
    ".pytest_cache", ".ruff_cache", ".idea", ".vscode", ".turbo", ".cache", "out", ".terraform",
}
CODE_EXT = {".py", ".ts", ".tsx", ".js", ".jsx", ".mjs", ".cjs", ".go", ".java", ".kt", ".rb",
            ".cs", ".php", ".rs", ".scala", ".swift", ".ipynb"}
CONFIG_EXT = {".yaml", ".yml", ".json", ".toml", ".ini", ".cfg", ".env", ".properties"}
PROMPT_EXT = {".prompt", ".j2", ".jinja", ".jinja2", ".mustache", ".hbs", ".liquid", ".poml"}
PROMPT_DIR_HINTS = ("prompt", "prompts", "templates", "instructions", "system_prompts")
SKIP_FILES = {"package-lock.json", "yarn.lock", "pnpm-lock.yaml", "poetry.lock", "uv.lock",
              "Cargo.lock", "composer.lock", "Gemfile.lock"}

PROVIDERS = {
    "anthropic": [r"\bimport anthropic\b", r"\bfrom anthropic\b", r"@anthropic-ai/sdk", r"langchain_anthropic",
                  r"\bChatAnthropic\b", r"anthropic-sdk-go", r"com\.anthropic"],
    "openai": [r"\bfrom openai\b", r"\bimport openai\b", r"""from\s+['"]openai['"]""", r"""require\(\s*['"]openai['"]\s*\)""",
               r"langchain_openai", r"\bChatOpenAI\b", r"\bAzureOpenAI\b", r"\bAzureChatOpenAI\b", r"openai-go",
               r"@ai-sdk/openai"],
    "gemini": [r"from google import genai", r"google\.generativeai", r"\bvertexai\b", r"@google/genai",
               r"@google/generative-ai", r"langchain_google", r"\bChatVertexAI\b", r"\bChatGoogleGenerativeAI\b",
               r"@ai-sdk/google"],
    "bedrock": [r"bedrock-runtime", r"\bChatBedrock\w*", r"\bAnthropicBedrock\w*", r"BedrockRuntimeClient",
                r"@ai-sdk/amazon-bedrock"],
    "mistral": [r"\bmistralai\b", r"@mistralai/"],
    "cohere": [r"\bimport cohere\b", r"\bfrom cohere\b", r"cohere-ai"],
    "groq": [r"\bfrom groq\b", r"\bimport groq\b", r"groq-sdk"],
    "together": [r"\bfrom together\b", r"\bimport together\b", r"together-ai"],
    "deepseek": [r"api\.deepseek\.com"],
    "openrouter": [r"openrouter\.ai"],
    "ollama": [r"\bollama\b"],
    "vllm/self-hosted": [r"\bvllm\b", r"text-generation-inference", r"\bsglang\b", r"llama_cpp", r"localhost:\d+/v1"],
    "huggingface": [r"huggingface_hub", r"\bInferenceClient\(", r"from transformers import"],
}
FRAMEWORKS = {
    "langchain": [r"\blangchain\b", r"@langchain/"],
    "langgraph": [r"\blanggraph\b", r"@langchain/langgraph"],
    "llamaindex": [r"llama_index", r"llamaindex"],
    "crewai": [r"\bcrewai\b"],
    "autogen": [r"\bautogen\b", r"autogen_agentchat"],
    "semantic-kernel": [r"semantic_kernel", r"Microsoft\.SemanticKernel"],
    "haystack": [r"\bhaystack\b"],
    "dspy": [r"\bimport dspy\b", r"\bfrom dspy\b"],
    "litellm": [r"\blitellm\b"],
    "vercel-ai-sdk": [r"""from\s+['"]ai['"]""", r"\bgenerateText\(", r"\bstreamText\("],
    "openai-agents": [r"from agents import", r"@openai/agents"],
    "pydantic-ai": [r"pydantic_ai"],
    "instructor": [r"\bimport instructor\b", r"\bfrom instructor\b"],
    "mastra": [r"@mastra/"],
    "claude-agent-sdk": [r"claude_agent_sdk", r"@anthropic-ai/claude-agent-sdk"],
    "observability": [r"\blangfuse\b", r"\blangsmith\b", r"\bhelicone\b", r"\bportkey\b", r"\bopenlit\b",
                      r"\btraceloop\b", r"\bphoenix\b", r"\barize\b", r"gen_ai\."],
}
CALLS = [
    ("chat", r"\.chat\.completions\.(?:create|parse|stream)\("),
    ("chat", r"\.responses\.(?:create|parse|stream)\("),
    ("chat", r"\.messages\.(?:create|stream|parse)\("),
    ("chat", r"\.generate_content(?:_stream)?\(|\.generateContent(?:Stream)?\("),
    ("chat", r"\.converse(?:_stream)?\(|ConverseCommand|InvokeModelCommand|\.invoke_model\("),
    ("chat", r"\b(?:a?completion|acompletion)\(\s*model\s*="),
    ("chat", r"\b(?:generateText|streamText|generateObject|streamObject)\("),
    ("chat", r"\.(?:chat|generate)\(\s*model\s*="),  # ollama / cohere style
    ("framework", r"\.a?invoke\(|\.a?batch\(|\.a?stream\("),  # only counted in framework files
    ("framework", r"\.kickoff\(|Runner\.run(?:_sync)?\(|\.initiate_chat\(|\.run_stream\("),
    ("raw-http", r"(?:\bfetch|axios(?:\.post)?|requests\.post|httpx\.post|\.post)\s*\(.*(?:/v1/messages|/chat/completions|/responses\b|:generateContent|:streamGenerateContent|/embeddings|/converse|api\.openai\.com|api\.anthropic\.com|generativelanguage\.googleapis\.com)"),
    ("embedding", r"\.embeddings\.create\(|\.embed_content\(|\.embedContent\(|\.embed_(?:documents|query)\(|\bembedMany?\(|\.embed\("),
    ("batch", r"\.batches\.create\(|create_model_invocation_job|\.batches\.create_job|batch_create"),
    ("token-count", r"count_tokens\(|countTokens\(|tiktoken"),
]
PARAMS = {
    "max_tokens": r"\bmax_tokens\b|\bmaxTokens\b|\bmaxOutputTokens\b",
    "max_output_tokens": r"\bmax_output_tokens\b|\bmax_completion_tokens\b",
    "temperature": r"\btemperature\b",
    "n>1": r"\bn\s*[:=]\s*(?:[2-9]|\d{2,})\b|\bbest_of\b",
    "stream": r"\bstream\s*[:=]\s*True\b|\bstream\s*:\s*true\b|\.stream\(",
    "cache_marker": r"cache_control|cachePoint|cache_point|cached_content|cachedContent|caches\.create",
    "prompt_cache_key": r"prompt_cache_key|prompt_cache_retention|prompt_cache_options|prompt_cache_breakpoint",
    "service_tier": r"service_tier|serviceTier|\bspeed\s*[:=]",
    "reasoning_effort": r"reasoning_effort|reasoningEffort|\breasoning\s*[:=]|\beffort\b|thinking_budget|thinking_level|\bthinking\s*[:=]|budget_tokens",
    "tools": r"\btools\s*[:=]|bind_tools|\bfunctions\s*[:=]",
    "structured_output": r"response_format|output_config|text_format|json_schema|responseSchema|response_schema|with_structured_output|generateObject|\bstrict\b",
    "image_detail": r"\bdetail\s*[:=]",
    "stop": r"\bstop\s*[:=]|stop_sequences|stopSequences",
}
MODEL_RE = re.compile(
    r"""["'`]((?:gpt-[\w.\-]+|o[1-9](?:-[\w.\-]+)?|chatgpt-[\w.\-]+|claude-[\w.\-@]+|"""
    r"""(?:us\.|eu\.|apac\.|global\.)?anthropic\.[\w.\-:]+|gemini-[\w.\-]+|text-embedding-[\w.\-]+|embed-[\w.\-]+|"""
    r"""(?:open-)?mi[sx]tral[\w.\-:/]*|(?:meta-llama/)?[Ll]lama[\w.\-:/]*|meta\.llama[\w.\-:]*|command-[\w.\-]+|"""
    r"""deepseek[\w.\-:/]*|[Qq]wen[\w.\-:/]*|amazon\.(?:nova|titan)[\w.\-:]*|grok-[\w.\-]+))["'`]""")
DYNAMIC_RE = re.compile(
    r"datetime\.(?:now|utcnow)\(|date\.today\(|time\.time\(\)|Date\.now\(|new Date\(|uuid4\(|uuid\.v4|randomUUID|"
    r"random\.(?:choice|sample|shuffle)\(|Math\.random\(|strftime\(|toISOString\(")
PROMPT_CTX_RE = re.compile(r"system|prompt|instruction|developer", re.I)
LOOP_RE = re.compile(r"^\s*(?:while\s+True|while\s*\(\s*true\s*\)|while\s+not\b|while\s*\(|for\s.+\bin\b|for\s*\(|\.forEach\(|\.map\()")
FUNC_START_RE = re.compile(r"^\s*(?:async\s+)?def\s|^\s*(?:export\s+)?(?:async\s+)?function\b|^\s*class\s|^\s*func\s|=>\s*\{\s*$")
IMPORT_RE = re.compile(r"^\s*(?:from\s+([\w.]+)\s+import|import\s+([\w.]+)|.*\bfrom\s+['\"]([^'\"]+)['\"]|.*require\(\s*['\"]([^'\"]+)['\"])")
LOOP_GUARD_RE = re.compile(r"max_iter|max_steps|max_turns|recursion_limit|maxSteps|max_rounds|max_consecutive_auto_reply|stopWhen|stepCountIs|max_tool_calls")
RETRY_RE = re.compile(r"attempt\s*<\s*\d|for\s+attempt\s+in\s+range|\bretries\s*[<>=]|\btenacity\b|@retry\b|\bbackoff\b|max_retries|maxRetries|p-retry|async-retry|with_retry|retry_if|Retry\(")
USAGE_RE = re.compile(r"\.usage\b|usage_metadata|usageMetadata|cache_read_input_tokens|cached_tokens|cacheReadInputTokens|"
                      r"cached_content_token_count|cachedContentTokenCount|prompt_tokens|input_tokens|response_metadata")
MOCK_RE = re.compile(r"\bmock\b|MagicMock|patch\(|\bvcr\b|cassette|respx|\bnock\b|\bmsw\b|fake|stub|responses\.activate|pytest_recording|httpretty", re.I)
INDENT_JSON_RE = re.compile(r"json\.dumps\(.*\bindent\s*=\s*[1-9]|JSON\.stringify\(.*,\s*(?:null|undefined)\s*,\s*['\"]?\s*\d")
SUPPRESS_RE = re.compile(r"llm-cost:\s*intentional\s+((?:[A-Z]+-\d+|\*)(?:\s*,\s*(?:[A-Z]+-\d+|\*))*)\s*(?:[-–—:]\s*(.*?))?\s*(?:\*/|-->)?\s*$")
INLINE_PROMPT_RE = re.compile(
    r"""(?P<name>\b\w*(?:prompt|PROMPT|Prompt|system|SYSTEM|System|instruction|INSTRUCTION|Instruction)\w*)\s*[:=]\s*"""
    r"""(?:[frbuFRBU]{0,2})(?P<q>\"\"\"|'''|`)(?P<body>.*?)(?P=q)""", re.S)

RAW_PROVIDER_URL = {
    "anthropic": re.compile(r"api\.anthropic\.com|/v1/messages\b|anthropic-version"),
    "openai": re.compile(r"api\.openai\.com|/chat/completions\b|openai\.azure\.com"),
    "gemini": re.compile(r"generativelanguage\.googleapis\.com|:generateContent\b|aiplatform\.googleapis\.com"),
}
PROMPTISH_RE = re.compile(r"\b(?:system|SYSTEM)_?(?:prompt|PROMPT|instruction)|[\"']role[\"']\s*:\s*[\"']system|build_\w*prompt|\bPROMPT\w*\s*=")
BIG_LITERAL_RE = re.compile(r"(?P<q>\"\"\"|'''|`)(?P<body>[^`]{6000,}?)(?P=q)", re.S)


def repeated_ratio(text):
    """Share of sentence/line units that are exact repeats - flags copy-paste bloat in prompts (INPUT-01)."""
    units = [u.strip() for u in re.split(r"\n|(?<=[.!?])\s+", text) if len(u.strip()) > 15]
    units = [re.sub(r"^[-*\d.\s]+|Policy \d+:\s*", "", u) for u in units]  # ignore list numbering
    if len(units) < 10:
        return 0.0
    return round(1 - len(set(units)) / len(units), 2)


TEST_PATH_RE = re.compile(r"(^|/)(tests?|__tests__|spec)/|(^|/)test_[^/]*\.py$|_test\.(py|go)$|\.(test|spec)\.[jt]sx?$")


def approx_tokens(text):
    return max(1, len(text) // 4)


def compile_map(d):
    return {k: [re.compile(p) for p in v] for k, v in d.items()}


PROV_C, FW_C = compile_map(PROVIDERS), compile_map(FRAMEWORKS)
CALLS_C = [(k, re.compile(p)) for k, p in CALLS]
PARAMS_C = {k: re.compile(p) for k, p in PARAMS.items()}


def read_text(path, max_bytes):
    try:
        if os.path.getsize(path) > max_bytes:
            return None
        with open(path, "r", encoding="utf-8", errors="ignore") as f:
            text = f.read()
        if path.endswith(".ipynb"):
            try:
                nb = json.loads(text)
                text = "\n".join("".join(c.get("source", [])) for c in nb.get("cells", []) if c.get("cell_type") == "code")
            except Exception:
                return None
        return text
    except OSError:
        return None


def iter_files(root):
    for dirpath, dirnames, filenames in os.walk(root):
        dirnames[:] = [d for d in dirnames if d not in SKIP_DIRS and (not d.startswith(".") or d == ".github")]
        for fn in filenames:
            if fn in SKIP_FILES:
                continue
            yield os.path.join(dirpath, fn)


def scan(root, max_kb):
    root = os.path.abspath(root)
    out = {
        "root": root, "files_scanned": 0, "providers": defaultdict(list), "frameworks": defaultdict(list),
        "call_sites": [], "models": defaultdict(list), "inline_prompts": [], "prompt_files": [],
        "dynamic_near_prompt": [], "indented_json_in_prompt_files": [], "retry_signals": [],
        "live_calls_in_tests": [], "suppressions": [], "usage_read_files": [], "loop_guard_signals": [],
    }
    max_bytes = max_kb * 1024
    test_candidates = []
    for path in iter_files(root):
        rel = os.path.relpath(path, root).replace(os.sep, "/")
        ext = os.path.splitext(path)[1].lower()
        in_prompt_dir = any(h in rel.lower().split("/")[:-1] for h in PROMPT_DIR_HINTS)
        is_code, is_cfg = ext in CODE_EXT, ext in CONFIG_EXT
        is_prompt = ext in PROMPT_EXT or ((in_prompt_dir or re.search(r"prompt|instruction", os.path.basename(rel), re.I)) and ext in {".txt", ".md", ".yaml", ".yml", ".json"})
        if not (is_code or is_cfg or is_prompt):
            continue
        text = read_text(path, max_bytes)
        if text is None:
            continue
        out["files_scanned"] += 1
        lines = text.splitlines()

        for i, line in enumerate(lines, 1):
            m = SUPPRESS_RE.search(line)
            if m:
                ids = [x.strip() for x in m.group(1).split(",")]
                out["suppressions"].append({"file": rel, "line": i, "ids": ids, "reason": (m.group(2) or "").strip() or None})

        if is_prompt:
            out["prompt_files"].append({"file": rel, "approx_tokens": approx_tokens(text), "repeated_line_ratio": repeated_ratio(text),
                                        "dynamic_placeholders": len(re.findall(r"\{\{.*?\}\}|\{[a-zA-Z_]\w*\}|\$\{.*?\}", text))})
        for m in MODEL_RE.finditer(text):
            ln = text.count("\n", 0, m.start()) + 1
            out["models"][m.group(1)].append(f"{rel}:{ln}")
        if not is_code:
            continue

        provs = [p for p, pats in PROV_C.items() if any(r.search(text) for r in pats)]
        provs += [p + " (raw http)" for p, rx in RAW_PROVIDER_URL.items() if p not in provs and rx.search(text)]
        fws = [f for f, pats in FW_C.items() if any(r.search(text) for r in pats)]
        for p in provs:
            out["providers"][p].append(rel)
        for f in fws:
            out["frameworks"][f].append(rel)
        llm_file = bool(provs or fws)
        if USAGE_RE.search(text) and llm_file:
            out["usage_read_files"].append(rel)

        sites = []
        for i, line in enumerate(lines, 1):
            for kind, rx in CALLS_C:
                if kind == "framework" and not fws:
                    continue
                if kind in ("embedding", "batch", "token-count") and not llm_file:
                    continue
                if rx.search(line):
                    lo, hi = max(0, i - 4), min(len(lines), i + 30)
                    window = "\n".join(lines[lo:hi])
                    params = sorted(k for k, r in PARAMS_C.items() if r.search(window))
                    in_loop = False
                    for l in reversed(lines[max(0, i - 16):i - 1]):
                        if LOOP_RE.search(l):
                            in_loop = True
                            break
                        if FUNC_START_RE.search(l):
                            break
                    models = sorted(set(MODEL_RE.findall(window)))
                    sites.append({"file": rel, "line": i, "kind": kind, "code": line.strip()[:160],
                                  "params_nearby": params, "models_nearby": models, "inside_loop_heuristic": in_loop})
                    break
        # keep only files that look like real LLM usage for generic patterns
        if sites and not llm_file:
            sites = [s for s in sites if s["kind"] != "framework"
                     and re.search(r"completions|responses|messages|generate_content|generateContent|converse|embeddings", s["code"])]
        out["call_sites"].extend(sites)

        for i, line in enumerate(lines, 1):  # retries anywhere (handlers often wrap LLM code from other files)
            if RETRY_RE.search(line) and not (sites or llm_file):
                out["retry_signals"].append({"file": rel, "line": i, "code": line.strip()[:120]})
        promptish = bool(re.search(r"prompt|instruction", rel, re.I)) or bool(PROMPTISH_RE.search(text))
        if sites or llm_file or promptish:
            seen_lines = set()
            for m in INLINE_PROMPT_RE.finditer(text):
                body = m.group("body")
                tok = approx_tokens(body)
                if tok >= 300:
                    ln = text.count("\n", 0, m.start()) + 1
                    seen_lines.add(ln)
                    dyn = bool(DYNAMIC_RE.search(body)) or bool(re.search(r"\{[a-zA-Z_][\w.]*\}|\$\{", body))
                    out["inline_prompts"].append({"file": rel, "line": ln, "name": m.group("name"), "approx_tokens": tok,
                                                  "has_interpolation": dyn, "repeated_line_ratio": repeated_ratio(body)})
            for m in BIG_LITERAL_RE.finditer(text):  # big string constants not named *prompt* (POLICIES, STYLE, ...)
                ln = text.count("\n", 0, m.start()) + 1
                if ln not in seen_lines:
                    body = m.group("body")
                    out["inline_prompts"].append({"file": rel, "line": ln, "name": "(large string literal)",
                                                  "approx_tokens": approx_tokens(body),
                                                  "has_interpolation": bool(re.search(r"\{[a-zA-Z_][\w.]*\}|\$\{", body)),
                                                  "repeated_line_ratio": repeated_ratio(body)})
            for i, line in enumerate(lines, 1):
                if DYNAMIC_RE.search(line):
                    ctx = "\n".join(lines[max(0, i - 12):min(len(lines), i + 12)])
                    if PROMPT_CTX_RE.search(ctx):
                        out["dynamic_near_prompt"].append({"file": rel, "line": i, "code": line.strip()[:160]})
                if INDENT_JSON_RE.search(line):
                    out["indented_json_in_prompt_files"].append({"file": rel, "line": i, "code": line.strip()[:160]})
                if RETRY_RE.search(line) and (sites or llm_file):
                    out["retry_signals"].append({"file": rel, "line": i, "code": line.strip()[:120]})
                if LOOP_GUARD_RE.search(line):
                    out["loop_guard_signals"].append({"file": rel, "line": i, "code": line.strip()[:120]})

        if TEST_PATH_RE.search(rel) and not MOCK_RE.search(text):
            imports = []
            for l in lines:
                im = IMPORT_RE.match(l)
                if im:
                    imports.append(next(g for g in im.groups() if g))
            test_candidates.append({"file": rel, "direct_sites": len(sites), "imports": imports})

    def _norm(mod):
        mod = re.sub(r"^(?:\.\.?/)+", "", mod)
        mod = re.sub(r"\.(?:py|[cm]?[jt]sx?)$", "", mod)
        return mod.replace("/", ".").strip(".")
    llm_modules = {_norm(f): f for f in {c["file"] for c in out["call_sites"]}}
    for t in test_candidates:
        hit = sorted({f for imp in t["imports"] for k, f in llm_modules.items()
                      if _norm(imp) and (k == _norm(imp) or k.endswith("." + _norm(imp)) or _norm(imp).endswith("." + k))})
        if t["direct_sites"] or hit:
            out["live_calls_in_tests"].append({"file": t["file"], "call_sites": t["direct_sites"], "imports_llm_modules": hit})
    out["providers"] = {k: sorted(set(v)) for k, v in out["providers"].items()}
    out["frameworks"] = {k: sorted(set(v)) for k, v in out["frameworks"].items()}
    out["models"] = {k: v[:50] for k, v in sorted(out["models"].items(), key=lambda kv: -len(kv[1]))}
    out["prompt_files"].sort(key=lambda x: -x["approx_tokens"])
    out["inline_prompts"].sort(key=lambda x: -x["approx_tokens"])
    out["summary"] = summarize(out)
    return out


def summarize(o):
    kinds = Counter(s["kind"] for s in o["call_sites"])
    sites = [s for s in o["call_sites"] if s["kind"] in ("chat", "framework")]
    no_cap = [s for s in sites if not ({"max_tokens", "max_output_tokens"} & set(s["params_nearby"]))]
    return {
        "call_sites_by_kind": dict(kinds),
        "distinct_models": len(o["models"]),
        "files_with_llm_calls": len({s["file"] for s in o["call_sites"]}),
        "any_usage_field_read": bool(o["usage_read_files"]),
        "any_cache_marker": any("cache_marker" in s["params_nearby"] or "prompt_cache_key" in s["params_nearby"] for s in o["call_sites"]),
        "any_batch_api": kinds.get("batch", 0) > 0,
        "prompts_with_repeated_lines": sum(1 for p in o["inline_prompts"] + o["prompt_files"] if p.get("repeated_line_ratio", 0) >= 0.3),
        "chat_sites_without_output_cap_nearby": len(no_cap),
        "chat_sites_inside_loop_heuristic": sum(1 for s in sites if s["inside_loop_heuristic"]),
        "loop_guards_found": len(o["loop_guard_signals"]),
        "retry_signals": len(o["retry_signals"]),
        "dynamic_values_near_prompts": len(o["dynamic_near_prompt"]),
        "live_calls_in_tests_files": len(o["live_calls_in_tests"]),
        "suppressions": len(o["suppressions"]),
    }


def to_markdown(o):
    s = o["summary"]
    L = [f"# LLM usage scan: `{o['root']}`", "", f"Files scanned: {o['files_scanned']}", ""]
    L += ["## Summary", ""] + [f"- **{k}**: {v}" for k, v in s.items()] + [""]
    L += ["## Providers", ""] + [f"- {p}: {len(f)} file(s), e.g. `{f[0]}`" for p, f in o["providers"].items()] + [""]
    L += ["## Frameworks", ""] + [f"- {p}: {len(f)} file(s), e.g. `{f[0]}`" for p, f in o["frameworks"].items()] + [""]
    L += ["## Models (by occurrences)", ""] + [f"- `{m}` × {len(locs)}: {', '.join(locs[:4])}" for m, locs in list(o["models"].items())[:30]] + [""]
    L += ["## Call sites", "", "| Location | Kind | Params nearby | Models nearby | In loop? |", "|---|---|---|---|---|"]
    for c in o["call_sites"][:200]:
        L.append(f"| `{c['file']}:{c['line']}` | {c['kind']} | {', '.join(c['params_nearby']) or '-'} | "
                 f"{', '.join(c['models_nearby']) or '-'} | {'yes?' if c['inside_loop_heuristic'] else ''} |")
    if len(o["call_sites"]) > 200:
        L.append(f"| ... {len(o['call_sites']) - 200} more in JSON | | | | |")
    L.append("")
    if o["inline_prompts"] or o["prompt_files"]:
        L += ["## Largest prompts (≈ chars/4 tokens)", ""]
        rep = lambda p: (" **%d%% repeated lines (INPUT-01?)**" % int(100 * p["repeated_line_ratio"])) if p.get("repeated_line_ratio", 0) >= 0.3 else ""
        L += [f"- inline `{p['name']}` at `{p['file']}:{p['line']}` ≈ {p['approx_tokens']} tok"
              f"{' (interpolated)' if p['has_interpolation'] else ''}{rep(p)}" for p in o["inline_prompts"][:15]]
        L += [f"- file `{p['file']}` ≈ {p['approx_tokens']} tok, {p['dynamic_placeholders']} placeholders{rep(p)}" for p in o["prompt_files"][:15]]
        L.append("")
    for key, title in [("dynamic_near_prompt", "Dynamic values near prompt code (possible CACHE-01)"),
                       ("indented_json_in_prompt_files", "Pretty-printed JSON in LLM files (possible INPUT-08)"),
                       ("retry_signals", "Retry signals (check RELY-01)"),
                       ("loop_guard_signals", "Loop guards found (AGENT-01)")]:
        if o[key]:
            L += [f"## {title}", ""] + [f"- `{x['file']}:{x['line']}` {x['code']}" for x in o[key][:25]] + [""]
    if o["live_calls_in_tests"]:
        L += ["## Possible live LLM calls in tests (DEV-01)", ""] + [f"- `{x['file']}` ({x['call_sites']} direct sites; imports {', '.join(x['imports_llm_modules']) or '-'}; no mock markers)" for x in o["live_calls_in_tests"]] + [""]
    if o["suppressions"]:
        L += ["## `llm-cost: intentional` comments", ""] + [f"- `{x['file']}:{x['line']}` {', '.join(x['ids'])}: {x['reason'] or '(no reason given)'}" for x in o["suppressions"]] + [""]
    L.append("_Heuristic leads only - confirm each by reading the code._")
    return "\n".join(L)


def main():
    ap = argparse.ArgumentParser(description=__doc__, formatter_class=argparse.RawDescriptionHelpFormatter)
    ap.add_argument("root")
    ap.add_argument("--out", help="write JSON here (default: stdout if --markdown not given)")
    ap.add_argument("--markdown", action="store_true", help="print a markdown summary to stdout")
    ap.add_argument("--max-file-kb", type=int, default=1024)
    a = ap.parse_args()
    if not os.path.isdir(a.root):
        sys.exit(f"not a directory: {a.root}")
    result = scan(a.root, a.max_file_kb)
    if a.out:
        with open(a.out, "w", encoding="utf-8") as f:
            json.dump(result, f, indent=2)
    if a.markdown:
        print(to_markdown(result))
    elif not a.out:
        print(json.dumps(result, indent=2))


if __name__ == "__main__":
    main()
`````

### `cost_model.py`

<!-- file: cost_model.py sha256=07a3097987bfcb53f0e0a1ef4edc4e56569ae19b5727d21912358c87203e038a -->
`````python
#!/usr/bin/env python3
"""Cost arithmetic for the llm-cost-optimizer skill (stdlib only).

No prices are built in - supply prices you have VERIFIED from the provider's
pricing page (per 1M tokens), and cite that page in the report.

Subcommands
-----------
estimate        Per-route monthly cost and lever ceilings (caching, batch, model swap).
simulate-cache  Cache hit rate for given TTLs from real request timestamps (or a Poisson rate).

Profile JSON for `estimate`:
{
  "currency": "USD",
  "prices_source": "https://... fetched 2026-10-21",
  "prices": {                       # per 1M tokens
    "big-model":   {"input": 3.0, "output": 15.0, "cache_read": 0.3, "cache_write": 3.75},
    "small-model": {"input": 0.25, "output": 1.25, "cache_read": 0.025, "cache_write": 0.3125}
  },
  "batch_discount": 0.5,           # fraction off for batch tier (verify)
  "routes": [
    {"name": "chat", "model": "big-model", "requests_per_month": 1500000,
     "input_tokens": 6000,          # avg total input per request (incl. prefix + history)
     "cacheable_prefix_tokens": 4500,  # byte-stable shared/reused part of the input
     "current_cache_hit_rate": 0.0,    # fraction of requests whose prefix is read from cache today
     "target_cache_hit_rate": 0.85,    # what you expect after fixes (use simulate-cache)
     "output_tokens": 400,          # avg output incl. reasoning tokens
     "batchable_share": 0.0,        # fraction of this route's traffic nobody waits on
     "alt_model": "small-model",    # optional: candidate for a model-swap ceiling (tradeoff!)
     "after": {"input_tokens": 2500, "cacheable_prefix_tokens": 1000},   # optional: ANY overrides that model a
     "after_label": "INPUT-01 dedupe prompt"}                           # lever (prompt trim, fewer requests_per_month,
                                                                        # fewer output tokens after lowering thinking...)
  ]
}
cache_write: price applied to prefix tokens on a miss when the provider charges
for writes; omit it (or set equal to input) for providers that cache automatically
without a write premium.
"""
import argparse
import json
import math
import sys
from datetime import datetime


def _price(prices, model, key, default=None):
    p = prices.get(model)
    if p is None:
        sys.exit(f"no prices for model '{model}'")
    v = p.get(key, default)
    if v is None:
        sys.exit(f"missing '{key}' price for model '{model}'")
    return v


def route_cost(r, prices, model=None, hit=None, batch_share=0.0, batch_discount=0.0):
    m = model or r["model"]
    n = r["requests_per_month"]
    inp, out = r["input_tokens"], r["output_tokens"]
    prefix = min(r.get("cacheable_prefix_tokens", 0), inp)
    h = r.get("current_cache_hit_rate", 0.0) if hit is None else hit
    pi = _price(prices, m, "input")
    po = _price(prices, m, "output")
    pr = prices[m].get("cache_read", pi)
    pw = prices[m].get("cache_write", pi)
    # if the route has no caching at all today (h == 0 and no write premium wanted), bill prefix at input rate
    if h == 0 and hit is None:
        prefix_cost = prefix * pi
    else:
        prefix_cost = prefix * (h * pr + (1 - h) * pw)
    per_req = (prefix_cost + (inp - prefix) * pi + out * po) / 1e6
    total = per_req * n
    if batch_share:
        total = total * (1 - batch_share) + total * batch_share * (1 - batch_discount)
    return total


def cmd_estimate(a):
    prof = json.load(open(a.profile))
    prices, bd = prof["prices"], prof.get("batch_discount", 0.0)
    cur = prof.get("currency", "USD")
    rows, tot_now = [], 0.0
    for r in prof["routes"]:
        now = route_cost(r, prices)
        tot_now += now
        cache = route_cost(r, prices, hit=r.get("target_cache_hit_rate", r.get("current_cache_hit_rate", 0.0)))
        batch = route_cost(r, prices, batch_share=r.get("batchable_share", 0.0), batch_discount=bd)
        alt = None
        if r.get("alt_model"):
            alt = route_cost(r, prices, model=r["alt_model"])
        custom = None
        if r.get("after"):
            ra = dict(r, **r["after"])
            custom = now - route_cost(ra, prices, model=ra.get("model"),
                                      hit=ra.get("target_cache_hit_rate") if "target_cache_hit_rate" in r["after"] else None)
        rows.append((r["name"], r["model"], now, now - cache, now - batch, (now - alt) if alt is not None else None, r.get("alt_model"), custom, r.get("after_label", "after")))
    print(f"_Prices: {prof.get('prices_source', 'SOURCE NOT RECORDED - verify before citing')}_\n")
    print(f"| Route | Model | Current /mo ({cur}) | Caching ceiling | Batch ceiling | Model-swap ceiling (tradeoff) | Custom lever (`after`) |")
    print("|---|---|---|---|---|---|---|")
    for name, model, now, c, b, s, altm, cu, cl in rows:
        sw = f"{s:,.0f} → {altm} (needs eval)" if s is not None else "-"
        cx = f"{cu:,.0f} ({_pct(cu, now)}) {cl}" if cu is not None else "-"
        print(f"| {name} | {model} | {now:,.0f} | {c:,.0f} ({_pct(c, now)}) | {b:,.0f} ({_pct(b, now)}) | {sw} | {cx} |")
    print(f"| **Total (current)** | | **{tot_now:,.0f}** | | | | |")
    print("\nCeilings are independent upper bounds - they overlap (cached tokens also get the batch discount, a cheaper "
          "model shrinks every other ceiling). Rank by ceiling, then deflate overlapping levers before summing.")


def _pct(x, base):
    return f"{(100 * x / base):.0f}%" if base else "-"


def _parse_ts(s):
    s = s.strip()
    if not s:
        return None
    try:
        v = float(s)
        return v / 1000.0 if v > 1e11 else v  # ms epoch → s
    except ValueError:
        return datetime.fromisoformat(s.replace("Z", "+00:00")).timestamp()


def cmd_simulate(a):
    ttls = [float(t) for t in a.ttl]
    if a.rate_per_sec is not None:
        print("| TTL (s) | Est. hit rate (Poisson) |\n|---|---|")
        for t in ttls:
            print(f"| {t:g} | {1 - math.exp(-a.rate_per_sec * t):.1%} |")
        print("\nPoisson ignores burstiness; prefer real timestamps when available.")
        return
    if not a.timestamps:
        sys.exit("give --timestamps FILE (one ISO-8601 or epoch per line; optional ',key' suffix to group by prefix/conversation) or --rate-per-sec")
    groups = {}
    for line in open(a.timestamps):
        if not line.strip():
            continue
        parts = line.strip().split(",", 1)
        ts = _parse_ts(parts[0])
        key = parts[1].strip() if len(parts) > 1 else "_all"
        if ts is not None:
            groups.setdefault(key, []).append(ts)
    print(f"Requests: {sum(len(v) for v in groups.values())} across {len(groups)} cache key(s)\n")
    print("| TTL (s) | Hit rate | Writes (misses) |\n|---|---|---|")
    for t in ttls:
        hits = misses = 0
        for v in groups.values():
            v.sort()
            last = None
            for ts in v:
                if last is not None and ts - last <= t:
                    hits += 1
                else:
                    misses += 1
                last = ts  # reads refresh the TTL
        n = hits + misses
        print(f"| {t:g} | {hits / n:.1%} | {misses} |")
    print("\nCompare: longer TTLs usually cost more per write - longer TTL pays only if "
          "(misses avoided × (write - read price)) exceeds (remaining writes × extra write premium).")


def main():
    ap = argparse.ArgumentParser(description=__doc__, formatter_class=argparse.RawDescriptionHelpFormatter)
    sub = ap.add_subparsers(dest="cmd", required=True)
    e = sub.add_parser("estimate")
    e.add_argument("--profile", required=True)
    e.set_defaults(fn=cmd_estimate)
    s = sub.add_parser("simulate-cache")
    s.add_argument("--timestamps")
    s.add_argument("--rate-per-sec", type=float)
    s.add_argument("--ttl", nargs="+", default=["300", "3600"], help="TTL values in seconds")
    s.set_defaults(fn=cmd_simulate)
    a = ap.parse_args()
    a.fn(a)


if __name__ == "__main__":
    main()
`````

### `llm_meter.py`

<!-- file: llm_meter.py sha256=1c10b6fa1b43a42dfa16fdab0cef5c40fa221e8a1b53b590e0bd44531b06f348 -->
`````python
#!/usr/bin/env python3
"""llm_meter - record REAL token usage of every LLM call a Python process makes.

No code changes to the project: hooks are installed at the HTTP-client layer
(httpx, requests) and in botocore (Bedrock), so it works under the official
SDKs (OpenAI, Azure OpenAI, Anthropic, Google GenAI, Mistral, Groq, ...),
LiteLLM, LangChain, LlamaIndex, CrewAI etc. Streaming (SSE) responses are
teed and their final usage extracted. One JSON line per call is appended to
the log; cost is computed later (llm_eval.py) from a prices file, so baseline
and after-fix runs are priced identically.

Usage
  # wrap any python entry point
  python3 llm_meter.py --log calls.jsonl --label baseline -- my_script.py --arg 1
  python3 llm_meter.py --log calls.jsonl --label baseline -- -m pytest tests/llm
  # in-process
  import llm_meter; llm_meter.install("calls.jsonl", label="baseline")
  with llm_meter.case("chat::c1", trial=0): run_agent(...)
  # child processes: put this script's dir on PYTHONPATH with LLM_METER_LOG set;
  # the bundled sitecustomize hook (see llm_eval.py) auto-installs it.

Env (used when install() args are omitted): LLM_METER_LOG, LLM_METER_LABEL,
LLM_METER_CASE, LLM_METER_TRIAL, LLM_METER_EXTRA_HOSTS (comma list).

Not covered in-process: aiohttp-only clients, gRPC clients (legacy
google-generativeai), non-Python code -> use llm_cost_proxy.py instead.
"""
import contextlib
import contextvars
import json
import os
import re
import sys
import threading
import time
from urllib.parse import urlparse

__all__ = ["install", "case", "set_case", "normalize", "record_exchange", "extract_from_body"]

_LOCK = threading.Lock()
_STATE = {"log": None, "label": None, "installed": False, "case": None, "trial": None}
_CASE = contextvars.ContextVar("llm_meter_case", default=None)
_TRIAL = contextvars.ContextVar("llm_meter_trial", default=None)

LLM_PATH_RE = re.compile(
    r"/chat/completions$|/completions$|/responses$|/messages$|:generateContent$|:streamGenerateContent$|"
    r"/embeddings$|:embedContent$|:batchEmbedContents$|/converse(?:-stream)?$|/invoke(?:-with-response-stream)?$|"
    r"/v1/chat$|/v2/chat$|/v1/embed$|/v2/embed$|/api/chat$|/api/generate$|/api/embed(?:dings)?$")
EXCLUDE_PATH_RE = re.compile(r"count_tokens|countTokens|/batches|/files|/models$")
LLM_HOST_HINTS = ("openai", "anthropic", "googleapis.com", "mistral.ai", "groq.com", "together", "deepseek",
                  "openrouter.ai", "x.ai", "fireworks.ai", "cohere", "bedrock", "perplexity", "cerebras",
                  "azure.com", "localhost", "127.0.0.1")


# ---------------------------------------------------------------- context
def set_case(case_id, trial=None):
    """Tag subsequent calls (this context AND the process-wide fallback used by threads)."""
    _CASE.set(case_id)
    _TRIAL.set(trial)
    _STATE["case"], _STATE["trial"] = case_id, trial


@contextlib.contextmanager
def case(case_id, trial=None):
    prev = (_CASE.get(), _TRIAL.get(), _STATE["case"], _STATE["trial"])
    set_case(case_id, trial)
    try:
        yield
    finally:
        _CASE.set(prev[0]); _TRIAL.set(prev[1])
        _STATE["case"], _STATE["trial"] = prev[2], prev[3]


def _current():
    c = _CASE.get() or _STATE["case"] or os.environ.get("LLM_METER_CASE")
    t = _TRIAL.get()
    if t is None:
        t = _STATE["trial"]
    if t is None and os.environ.get("LLM_METER_TRIAL", "").isdigit():
        t = int(os.environ["LLM_METER_TRIAL"])
    return c, t


# ---------------------------------------------------------------- usage normalization
def _i(x):
    try:
        return int(x or 0)
    except (TypeError, ValueError):
        return 0


def _find_key(d, key):
    """Depth-first search for a key in nested dicts (for fields whose nesting varies by API version)."""
    if isinstance(d, dict):
        if key in d:
            return d[key]
        for v in d.values():
            r = _find_key(v, key)
            if r is not None:
                return r
    return None


def normalize(usage, host=""):
    """Map any provider usage object to billable buckets.

    Returns dict: input (uncached, billed at base input rate), cache_read, cache_write
    (short/default TTL), cache_write_long (1h/12h/24h TTL where reported), output
    (incl. reasoning), reasoning (informational subset of output), schema (which rule matched).
    """
    if not isinstance(usage, dict):
        return None
    u = usage
    z = dict(input=0, cache_read=0, cache_write=0, cache_write_long=0, output=0, reasoning=0, schema="")
    # Google Gemini (REST camelCase or SDK snake_case): promptTokenCount INCLUDES cached tokens
    if "promptTokenCount" in u or "prompt_token_count" in u or "candidatesTokenCount" in u:
        g = lambda a, b: _i(u.get(a, u.get(b)))
        prompt, cached = g("promptTokenCount", "prompt_token_count"), g("cachedContentTokenCount", "cached_content_token_count")
        tool = g("toolUsePromptTokenCount", "tool_use_prompt_token_count")
        thoughts = g("thoughtsTokenCount", "thoughts_token_count")
        z.update(input=max(0, prompt - cached) + tool, cache_read=cached,
                 output=g("candidatesTokenCount", "candidates_token_count") + thoughts, reasoning=thoughts, schema="gemini")
        return z
    # Bedrock Converse (camelCase): inputTokens EXCLUDES cache read/write
    if "inputTokens" in u or "outputTokens" in u:
        cw = _i(u.get("cacheWriteInputTokens"))
        long_w = 0
        det = u.get("cacheDetails")
        if isinstance(det, list):
            for d in det:
                if isinstance(d, dict) and str(d.get("ttl", "")).lower() in ("1h", "60m"):
                    long_w += _i(d.get("inputTokens"))
        z.update(input=_i(u.get("inputTokens")), cache_read=_i(u.get("cacheReadInputTokens")),
                 cache_write=max(0, cw - long_w), cache_write_long=long_w, output=_i(u.get("outputTokens")),
                 schema="bedrock-converse")
        return z
    # Bedrock invocation metrics (InvokeModel streams/headers)
    if "inputTokenCount" in u or "outputTokenCount" in u:
        z.update(input=_i(u.get("inputTokenCount")), cache_read=_i(u.get("cacheReadInputTokenCount")),
                 cache_write=_i(u.get("cacheWriteInputTokenCount")), output=_i(u.get("outputTokenCount")),
                 schema="bedrock-metrics")
        return z
    # Anthropic: input_tokens EXCLUDES cache reads and writes
    if "cache_read_input_tokens" in u or "cache_creation_input_tokens" in u or (
            "input_tokens" in u and "anthropic" in host and "input_tokens_details" not in u):
        cc = u.get("cache_creation") or {}
        w5, w1h = _i(cc.get("ephemeral_5m_input_tokens")), _i(cc.get("ephemeral_1h_input_tokens"))
        total_w = _i(u.get("cache_creation_input_tokens"))
        if not (w5 or w1h):
            w5 = total_w
        z.update(input=_i(u.get("input_tokens")), cache_read=_i(u.get("cache_read_input_tokens")),
                 cache_write=w5, cache_write_long=w1h, output=_i(u.get("output_tokens")), schema="anthropic")
        return z
    # OpenAI Chat Completions: prompt_tokens INCLUDES cached (and cache-write) tokens
    if "prompt_tokens" in u:
        det = u.get("prompt_tokens_details") or {}
        cached = _i(det.get("cached_tokens"))
        cw = _i(det.get("cache_write_tokens") if det.get("cache_write_tokens") is not None else _find_key(u, "cache_write_tokens"))
        long_w = _i(_find_key(u, "ttl_12h_tokens")) + _i(_find_key(u, "ttl_24h_tokens"))
        out_det = u.get("completion_tokens_details") or {}
        z.update(input=max(0, _i(u["prompt_tokens"]) - cached - cw), cache_read=cached, cache_write=max(0, cw - long_w),
                 cache_write_long=long_w, output=_i(u.get("completion_tokens")),
                 reasoning=_i(out_det.get("reasoning_tokens")), schema="openai-chat")
        return z
    # OpenAI Responses API (also Bedrock/Azure Responses): input_tokens INCLUDES cached tokens
    if "input_tokens" in u:
        det = u.get("input_tokens_details") or {}
        cached = _i(det.get("cached_tokens"))
        cw = _i(det.get("cache_write_tokens") if det.get("cache_write_tokens") is not None else _find_key(u, "cache_write_tokens"))
        long_w = _i(_find_key(u, "ttl_12h_tokens")) + _i(_find_key(u, "ttl_24h_tokens"))
        out_det = u.get("output_tokens_details") or {}
        z.update(input=max(0, _i(u["input_tokens"]) - cached - cw), cache_read=cached, cache_write=max(0, cw - long_w),
                 cache_write_long=long_w, output=_i(u.get("output_tokens")),
                 reasoning=_i(out_det.get("reasoning_tokens")), schema="openai-responses")
        return z
    # Cohere v2 / Ollama
    if "billed_units" in u or "tokens" in u and isinstance(u.get("tokens"), dict):
        b = u.get("billed_units") or u.get("tokens") or {}
        z.update(input=_i(b.get("input_tokens")), output=_i(b.get("output_tokens")), schema="cohere")
        return z
    if "prompt_eval_count" in u or "eval_count" in u:
        z.update(input=_i(u.get("prompt_eval_count")), output=_i(u.get("eval_count")), schema="ollama")
        return z
    if "total_tokens" in u:  # embeddings and similar with only totals
        z.update(input=_i(u.get("total_tokens")), schema="total-only")
        return z
    return None


def _extract_usage(obj):
    """Find the usage object inside one response JSON / SSE event."""
    if not isinstance(obj, dict):
        return None
    for path in (("usage",), ("usageMetadata",), ("usage_metadata",), ("response", "usage"), ("message", "usage"),
                 ("metadata", "usage"), ("amazon-bedrock-invocationMetrics",), ("meta",)):
        cur = obj
        for k in path:
            cur = cur.get(k) if isinstance(cur, dict) else None
        if isinstance(cur, dict) and cur:
            if path == ("meta",):
                cur = cur.get("billed_units") and {"billed_units": cur["billed_units"]}
                if not cur:
                    continue
            return cur
    if "prompt_eval_count" in obj or "eval_count" in obj:
        return {k: obj.get(k) for k in ("prompt_eval_count", "eval_count")}
    return None


def _extract_meta(obj, meta):
    if not isinstance(obj, dict):
        return
    for k in ("model", "modelVersion"):
        if isinstance(obj.get(k), str) and not meta.get("model"):
            meta["model"] = obj[k]
    for sub in ("message", "response"):
        s = obj.get(sub)
        if isinstance(s, dict) and isinstance(s.get("model"), str) and not meta.get("model"):
            meta["model"] = s["model"]
    stop = obj.get("stop_reason") or obj.get("stopReason")
    if not stop and isinstance(obj.get("delta"), dict):
        stop = obj["delta"].get("stop_reason")
    if not stop and isinstance(obj.get("choices"), list) and obj["choices"]:
        stop = (obj["choices"][0] or {}).get("finish_reason")
    if not stop and isinstance(obj.get("candidates"), list) and obj["candidates"]:
        stop = (obj["candidates"][0] or {}).get("finishReason") or (obj["candidates"][0] or {}).get("finish_reason")
    if not stop and isinstance(obj.get("response"), dict):
        r = obj["response"]
        stop = (r.get("incomplete_details") or {}).get("reason") if isinstance(r.get("incomplete_details"), dict) else None
        stop = stop or (r.get("status") if r.get("status") not in (None, "in_progress") else None)
    if not stop and obj.get("status") in ("completed", "incomplete", "failed"):
        stop = (obj.get("incomplete_details") or {}).get("reason") if isinstance(obj.get("incomplete_details"), dict) else obj["status"]
    if stop:
        meta["stop_reason"] = stop


def extract_from_body(body_bytes, content_type=""):
    """Parse a full (non-stream) JSON body, an SSE stream, or a streamed JSON array.
    Returns (raw_usage_dict_or_None, meta)."""
    meta, agg = {}, {}
    text = body_bytes.decode("utf-8", "ignore") if isinstance(body_bytes, (bytes, bytearray)) else (body_bytes or "")
    events = []
    if "text/event-stream" in content_type or re.search(r"^data:", text, re.M):
        for line in text.splitlines():
            line = line.strip()
            if line.startswith("data:"):
                payload = line[5:].strip()
                if payload and payload != "[DONE]":
                    try:
                        events.append(json.loads(payload))
                    except ValueError:
                        pass
    else:
        try:
            parsed = json.loads(text)
            events = parsed if isinstance(parsed, list) else [parsed]
        except ValueError:
            # NDJSON (e.g. Ollama streaming)
            for line in text.splitlines():
                try:
                    events.append(json.loads(line))
                except ValueError:
                    pass
    for ev in events:
        _extract_meta(ev, meta)
        u = _extract_usage(ev)
        if u:
            for k, v in u.items():  # shallow merge: later cumulative counts win, zero/None never erase
                if v not in (None, 0) or k not in agg:
                    agg[k] = v
    return (agg or None), meta


# ---------------------------------------------------------------- recording
def record_exchange(*, host, path, method="POST", status=None, req_body=None, resp_body=b"", content_type="",
                    stream=False, latency_ms=None, usage=None, meta=None, source="httpx"):
    if not _STATE["log"]:
        return
    if method.upper() != "POST" or EXCLUDE_PATH_RE.search(path or ""):
        return
    model = None
    if isinstance(req_body, (bytes, bytearray)):
        try:
            req = json.loads(req_body.decode("utf-8", "ignore"))
            model = req.get("model") if isinstance(req, dict) else None
        except ValueError:
            pass
    elif isinstance(req_body, dict):
        model = req_body.get("model") or req_body.get("modelId")
    if not model:  # Gemini /models/<id>:generateContent, Bedrock /model/<id>/converse
        m = re.search(r"/models?/([^/:]+)", path or "")
        model = m.group(1) if m else None
    meta = dict(meta or {})
    if usage is None and resp_body:
        usage, m2 = extract_from_body(resp_body, content_type)
        for k, v in m2.items():
            meta.setdefault(k, v)
    norm = normalize(usage, host) if usage else None
    c, t = _current()
    rec = {"ts": round(time.time(), 3), "run": _STATE["label"], "case": c, "trial": t, "host": host, "path": path,
           "model": meta.get("model") or model, "requested_model": model, "status": status, "stream": bool(stream),
           "usage_found": norm is not None, "tokens": norm, "stop_reason": meta.get("stop_reason"),
           "latency_ms": latency_ms, "source": source}
    line = json.dumps(rec, default=str)
    with _LOCK:
        with open(_STATE["log"], "a", encoding="utf-8") as f:
            f.write(line + "\n")


def _is_llm(host, path):
    if not LLM_PATH_RE.search(path or ""):
        return False
    extra = [h.strip() for h in os.environ.get("LLM_METER_EXTRA_HOSTS", "").split(",") if h.strip()]
    return any(h in (host or "") for h in LLM_HOST_HINTS + tuple(extra)) or os.environ.get("LLM_METER_ANY_HOST") == "1"


# ---------------------------------------------------------------- httpx hooks
def _install_httpx():
    try:
        import httpx
    except ImportError:
        return False
    if getattr(httpx.Client.send, "_llm_meter", False):
        return True
    o_send, o_asend = httpx.Client.send, httpx.AsyncClient.send

    def _req_info(request):
        url = request.url
        try:
            body = request.content
        except Exception:
            body = None
        return url.host, url.path, request.method, body

    def _finish(info, resp, t0, chunks):
        host, path, method, body = info
        try:
            record_exchange(host=host, path=path, method=method, status=resp.status_code, req_body=body,
                            resp_body=b"".join(chunks), content_type=resp.headers.get("content-type", ""),
                            stream=True, latency_ms=int((time.time() - t0) * 1000))
        except Exception:
            pass

    def _wrap_sync_stream(resp, info, t0):
        orig = resp.iter_bytes

        def iter_bytes(*a, **k):
            chunks = []
            try:
                for ch in orig(*a, **k):
                    chunks.append(ch)
                    yield ch
            finally:
                _finish(info, resp, t0, chunks)
        resp.iter_bytes = iter_bytes

    def _wrap_async_stream(resp, info, t0):
        orig = resp.aiter_bytes

        async def aiter_bytes(*a, **k):
            chunks = []
            try:
                async for ch in orig(*a, **k):
                    chunks.append(ch)
                    yield ch
            finally:
                _finish(info, resp, t0, chunks)
        resp.aiter_bytes = aiter_bytes

    def _after(request, resp, t0, is_async):
        try:
            info = _req_info(request)
            if not _is_llm(info[0], info[1]):
                return
            if hasattr(resp, "_content"):  # body already read (non-stream)
                record_exchange(host=info[0], path=info[1], method=info[2], status=resp.status_code, req_body=info[3],
                                resp_body=resp.content, content_type=resp.headers.get("content-type", ""),
                                latency_ms=int((time.time() - t0) * 1000))
            elif is_async:
                _wrap_async_stream(resp, info, t0)
            else:
                _wrap_sync_stream(resp, info, t0)
        except Exception:
            pass

    def send(self, request, *a, **k):
        t0 = time.time()
        resp = o_send(self, request, *a, **k)
        _after(request, resp, t0, False)
        return resp

    async def asend(self, request, *a, **k):
        t0 = time.time()
        resp = await o_asend(self, request, *a, **k)
        _after(request, resp, t0, True)
        return resp

    send._llm_meter = asend._llm_meter = True
    httpx.Client.send, httpx.AsyncClient.send = send, asend
    return True


# ---------------------------------------------------------------- requests hook
def _install_requests():
    try:
        import requests
    except ImportError:
        return False
    if getattr(requests.Session.send, "_llm_meter", False):
        return True
    o_send = requests.Session.send

    def send(self, request, **kw):
        t0 = time.time()
        resp = o_send(self, request, **kw)
        try:
            u = urlparse(request.url)
            if _is_llm(u.hostname or "", u.path):
                body = request.body if isinstance(request.body, (bytes, bytearray)) else (request.body or "").encode()
                ct = resp.headers.get("content-type", "")
                if not kw.get("stream"):
                    record_exchange(host=u.hostname, path=u.path, method=request.method, status=resp.status_code,
                                    req_body=body, resp_body=resp.content, content_type=ct, source="requests",
                                    latency_ms=int((time.time() - t0) * 1000))
                else:
                    orig = resp.iter_content

                    def iter_content(*a, **k):
                        chunks = []
                        try:
                            for ch in orig(*a, **k):
                                chunks.append(ch if isinstance(ch, bytes) else str(ch).encode())
                                yield ch
                        finally:
                            record_exchange(host=u.hostname, path=u.path, method=request.method, status=resp.status_code,
                                            req_body=body, resp_body=b"".join(chunks), content_type=ct, stream=True,
                                            source="requests", latency_ms=int((time.time() - t0) * 1000))
                    resp.iter_content = iter_content
        except Exception:
            pass
        return resp

    send._llm_meter = True
    requests.Session.send = send
    return True


# ---------------------------------------------------------------- botocore (Bedrock) hook
class _EventStreamTee:
    def __init__(self, stream, on_done):
        self._s, self._done, self._fired = stream, on_done, False
        self._usage, self._meta = {}, {}

    def __iter__(self):
        try:
            for ev in self._s:
                try:
                    if isinstance(ev, dict):
                        md = ev.get("metadata")
                        if isinstance(md, dict) and isinstance(md.get("usage"), dict):
                            self._usage.update(md["usage"])
                        ms = ev.get("messageStop")
                        if isinstance(ms, dict) and ms.get("stopReason"):
                            self._meta["stop_reason"] = ms["stopReason"]
                        ch = ev.get("chunk")
                        if isinstance(ch, dict) and ch.get("bytes"):
                            obj = json.loads(ch["bytes"])
                            m = obj.get("amazon-bedrock-invocationMetrics")
                            if isinstance(m, dict):
                                self._usage.update(m)
                            _extract_meta(obj, self._meta)
                except Exception:
                    pass
                yield ev
        finally:
            self._fire()

    def _fire(self):
        if not self._fired:
            self._fired = True
            self._done(self._usage or None, self._meta)

    def close(self):
        try:
            self._s.close()
        finally:
            self._fire()

    def __getattr__(self, name):
        return getattr(self._s, name)


def _install_botocore():
    try:
        import botocore.client
    except ImportError:
        return False
    BC = botocore.client.BaseClient
    if getattr(BC._make_api_call, "_llm_meter", False):
        return True
    orig = BC._make_api_call
    ops = {"Converse", "ConverseStream", "InvokeModel", "InvokeModelWithResponseStream"}

    def _make_api_call(self, operation_name, api_params):
        t0 = time.time()
        result = orig(self, operation_name, api_params)
        try:
            if operation_name in ops and self.meta.service_model.service_name == "bedrock-runtime":
                model = api_params.get("modelId")
                host = urlparse(self.meta.endpoint_url).hostname or "bedrock-runtime"
                path = "/model/%s/%s" % (model, operation_name)

                def done(usage, meta, stream=True):
                    record_exchange(host=host, path=path, req_body={"modelId": model}, usage=usage,
                                    meta=dict(meta or {}, model=model), stream=stream, source="botocore",
                                    status=(result.get("ResponseMetadata") or {}).get("HTTPStatusCode"),
                                    latency_ms=int((time.time() - t0) * 1000))
                if operation_name == "Converse":
                    done(result.get("usage"), {"stop_reason": result.get("stopReason")}, stream=False)
                elif operation_name == "InvokeModel":
                    h = (result.get("ResponseMetadata") or {}).get("HTTPHeaders") or {}
                    u = {"inputTokenCount": h.get("x-amzn-bedrock-input-token-count"),
                         "outputTokenCount": h.get("x-amzn-bedrock-output-token-count"),
                         "cacheReadInputTokenCount": h.get("x-amzn-bedrock-cache-read-input-token-count"),
                         "cacheWriteInputTokenCount": h.get("x-amzn-bedrock-cache-write-input-token-count")}
                    done(u if u["inputTokenCount"] is not None else None, {}, stream=False)
                else:
                    key = "stream" if operation_name == "ConverseStream" else "body"
                    if key in result:
                        result[key] = _EventStreamTee(result[key], done)
        except Exception:
            pass
        return result

    _make_api_call._llm_meter = True
    BC._make_api_call = _make_api_call
    return True


# ---------------------------------------------------------------- public install
def install(log=None, label=None):
    log = log or os.environ.get("LLM_METER_LOG")
    if not log:
        raise ValueError("llm_meter.install: give log path or set LLM_METER_LOG")
    _STATE["log"] = os.path.abspath(log)
    _STATE["label"] = label or os.environ.get("LLM_METER_LABEL") or "run"
    hooks = {"httpx": _install_httpx(), "requests": _install_requests(), "botocore": _install_botocore()}
    _STATE["installed"] = True
    return hooks


def _main():
    import argparse
    import runpy
    if "--" not in sys.argv:
        print(__doc__)
        sys.exit(2)
    i = sys.argv.index("--")
    ap = argparse.ArgumentParser()
    ap.add_argument("--log", required=True)
    ap.add_argument("--label", default="run")
    ap.add_argument("--case", default=None)
    a = ap.parse_args(sys.argv[1:i])
    rest = sys.argv[i + 1:]
    hooks = install(a.log, a.label)
    if a.case:
        set_case(a.case, 0)
    sys.stderr.write("[llm_meter] hooks: %s -> %s\n" % (hooks, _STATE["log"]))
    if not rest:
        sys.exit("nothing to run after --")
    if rest[0] == "-m":
        sys.argv = [rest[1]] + rest[2:]
        runpy.run_module(rest[1], run_name="__main__", alter_sys=True)
    else:
        sys.argv = rest
        sys.path.insert(0, os.path.dirname(os.path.abspath(rest[0])))
        runpy.run_path(rest[0], run_name="__main__")


if __name__ == "__main__":
    _main()
`````

### `llm_cost_proxy.py`

<!-- file: llm_cost_proxy.py sha256=039211e2dac3b8e66e8dfef73f5a0de2a944bc65791e5709965ae8abf070c69f -->
`````python
#!/usr/bin/env python3
"""llm_cost_proxy - local recording reverse proxy for measuring LLM usage from ANY language.

Point the app's SDK base URL at the proxy; it forwards to the real provider
(streaming passes through chunk by chunk), and appends one usage record per
call to the same JSONL format llm_meter.py writes. Binds to 127.0.0.1 only.
API keys pass through untouched and are never logged.

Start:
  python3 llm_cost_proxy.py --log calls.jsonl --label baseline --port 8787 \
      --route openai=https://api.openai.com --route anthropic=https://api.anthropic.com \
      --route gemini=https://generativelanguage.googleapis.com
Point clients at it (examples):
  OPENAI_BASE_URL=http://127.0.0.1:8787/openai/v1
  ANTHROPIC_BASE_URL=http://127.0.0.1:8787/anthropic
  Gemini: http_options={"base_url": "http://127.0.0.1:8787/gemini"}
  Azure OpenAI: --route azure=https://<resource>.openai.azure.com , endpoint=http://127.0.0.1:8787/azure
Tag calls with the current test case (runner does this before each case):
  POST http://127.0.0.1:8787/__case   {"case": "chat::c1", "trial": 0, "run": "baseline"}
  or send header  x-llm-cost-case: chat::c1   (and x-llm-cost-trial: 0)
Health: GET /__health
Note: SSE usage for OpenAI Chat Completions streams only appears if the request sets
stream_options={"include_usage": true}; otherwise the record says usage_found=false.
"""
import argparse
import http.client
import json
import os
import socket
import ssl
import sys
import time
from http.server import BaseHTTPRequestHandler, ThreadingHTTPServer
from urllib.parse import urlsplit

sys.path.insert(0, os.path.dirname(os.path.abspath(__file__)))
import llm_meter  # noqa: E402

HOP = {"connection", "keep-alive", "proxy-authenticate", "proxy-authorization", "te", "trailers",
       "transfer-encoding", "upgrade", "host", "content-length", "accept-encoding"}
ROUTES = {}


class Handler(BaseHTTPRequestHandler):
    protocol_version = "HTTP/1.1"

    def log_message(self, fmt, *args):
        if os.environ.get("LLM_PROXY_VERBOSE"):
            sys.stderr.write("[proxy] " + fmt % args + "\n")

    def _json(self, code, obj):
        b = json.dumps(obj).encode()
        self.send_response(code)
        self.send_header("content-type", "application/json")
        self.send_header("content-length", str(len(b)))
        self.end_headers()
        self.wfile.write(b)

    def _body(self):
        n = int(self.headers.get("content-length") or 0)
        return self.rfile.read(n) if n else b""

    def do_GET(self):
        if self.path == "/__health":
            return self._json(200, {"ok": True, "routes": ROUTES, "log": llm_meter._STATE["log"]})
        self._forward(b"")

    def do_POST(self):
        body = self._body()
        if self.path == "/__case":
            try:
                d = json.loads(body or b"{}")
            except ValueError:
                return self._json(400, {"error": "bad json"})
            llm_meter.set_case(d.get("case"), d.get("trial"))
            if d.get("run"):
                llm_meter._STATE["label"] = d["run"]
            return self._json(200, {"ok": True})
        self._forward(body)

    do_PUT = do_PATCH = do_DELETE = lambda self: self._forward(self._body())

    def _forward(self, body):
        parts = self.path.lstrip("/").split("/", 1)
        name, rest = parts[0], ("/" + parts[1]) if len(parts) > 1 else "/"
        if name not in ROUTES:
            return self._json(404, {"error": "unknown route '%s'; known: %s" % (name, list(ROUTES))})
        up = urlsplit(ROUTES[name])
        base_path = up.path.rstrip("/")
        target = base_path + rest
        conn_cls = http.client.HTTPSConnection if up.scheme == "https" else http.client.HTTPConnection
        kw = {"timeout": 600}
        if up.scheme == "https":
            kw["context"] = ssl.create_default_context()
        conn = conn_cls(up.hostname, up.port, **kw)
        headers = {k: v for k, v in self.headers.items() if k.lower() not in HOP and not k.lower().startswith("x-llm-cost-")}
        headers["Host"] = up.netloc
        headers["Accept-Encoding"] = "identity"  # so the proxy can read usage from the body
        if body:
            headers["Content-Length"] = str(len(body))
        case_h, trial_h = self.headers.get("x-llm-cost-case"), self.headers.get("x-llm-cost-trial")
        t0 = time.time()
        try:
            conn.request(self.command, target, body=body or None, headers=headers)
            resp = conn.getresponse()
        except (OSError, socket.timeout) as e:
            return self._json(502, {"error": "upstream error: %s" % e})
        ctype = resp.getheader("content-type", "")
        self.send_response(resp.status)
        for k, v in resp.getheaders():
            if k.lower() not in HOP:
                self.send_header(k, v)
        self.send_header("transfer-encoding", "chunked")
        self.end_headers()
        chunks = []
        try:
            while True:
                data = resp.read1(65536) if hasattr(resp, "read1") else resp.read(65536)
                if not data:
                    break
                chunks.append(data)
                self.wfile.write(b"%x\r\n%s\r\n" % (len(data), data))
                self.wfile.flush()
            self.wfile.write(b"0\r\n\r\n")
        except (BrokenPipeError, ConnectionResetError):
            pass  # client went away; still record what was generated
        finally:
            conn.close()
        try:
            if case_h:
                prev = (llm_meter._STATE["case"], llm_meter._STATE["trial"])
                llm_meter._STATE["case"], llm_meter._STATE["trial"] = case_h, int(trial_h) if (trial_h or "").isdigit() else None
            if llm_meter.LLM_PATH_RE.search(urlsplit(target).path):
                llm_meter.record_exchange(host=up.hostname, path=urlsplit(target).path, method=self.command,
                                          status=resp.status, req_body=body, resp_body=b"".join(chunks),
                                          content_type=ctype, stream="event-stream" in ctype,
                                          latency_ms=int((time.time() - t0) * 1000), source="proxy")
        finally:
            if case_h:
                llm_meter._STATE["case"], llm_meter._STATE["trial"] = prev


def main():
    ap = argparse.ArgumentParser(description=__doc__, formatter_class=argparse.RawDescriptionHelpFormatter)
    ap.add_argument("--log", required=True)
    ap.add_argument("--label", default="run")
    ap.add_argument("--port", type=int, default=8787)
    ap.add_argument("--route", action="append", default=[], help="name=https://upstream (repeatable)")
    a = ap.parse_args()
    for r in a.route or ["openai=https://api.openai.com", "anthropic=https://api.anthropic.com",
                         "gemini=https://generativelanguage.googleapis.com"]:
        k, _, v = r.partition("=")
        ROUTES[k.strip()] = v.strip()
    llm_meter._STATE["log"] = os.path.abspath(a.log)
    llm_meter._STATE["label"] = a.label
    srv = ThreadingHTTPServer(("127.0.0.1", a.port), Handler)
    sys.stderr.write("[llm_cost_proxy] http://127.0.0.1:%d  routes=%s  log=%s\n" % (a.port, ROUTES, a.log))
    try:
        srv.serve_forever()
    except KeyboardInterrupt:
        pass


if __name__ == "__main__":
    main()
`````

### `llm_eval.py`

<!-- file: llm_eval.py sha256=3edc6eab8f4f469b204c67f1077b5e57ce4898006441430a6fa1cbc9e215e4f2 -->
`````python
#!/usr/bin/env python3
"""llm_eval - measure REAL cost and quality per task, before and after each optimization.

Subcommands
  init      scaffold llm_cost_eval/ (eval.json, cases/, prices.json, checks.py) in a project
  run       execute every case of every route N times under the meter; record outputs + checks + usage
  report    price one run (tokens x prices.json) -> per-route cost/task, quality, cache hit, truncations
  compare   baseline vs one or more runs -> markdown table with measured % reduction, quality delta,
            regressions, noise flag, monthly projection
  estimate  rough cost of a run before spending money (from a previous run, or chars/4 heuristic)

Cost is always computed at report/compare time from ONE prices file, so baseline and after-fix
runs are priced identically. Calls made while checks run (e.g. an LLM judge) are tagged
"<case>::judge" and reported separately, never mixed into task cost.

eval.json (all paths relative to the eval dir unless absolute):
{
  "name": "support-bot",
  "project_root": "..",                 # added to sys.path / used as cwd
  "prices": "prices.json",
  "trials": 3,                          # >=3 recommended; trial 0 is often a cold cache
  "max_cost_usd": 5.0,                  # hard budget guard for one `run` (needs prices)
  "routes": {
    "chat": {
      "target": {"type": "python", "callable": "app.chat:answer", "call": "kwargs"},
      #   call: kwargs -> fn(**input) | args -> fn(*input) | single -> fn(input)
      #   or {"type": "command", "cmd": ["node", "scripts/run_case.js"], "stdin": "json",
      #       "env": {"OPENAI_BASE_URL": "http://127.0.0.1:8787/openai/v1"}, "proxy": "http://127.0.0.1:8787"}
      #   (command: case input JSON on stdin, output = stdout; python children are auto-metered,
      #    other languages need llm_cost_proxy.py running, plus "proxy" (its URL) and "proxy_log" (its --log file))
      "cases": "cases/chat.jsonl",      # {"id","input","expected"?, "checks"?, "tags"?} per line
      "checks": [{"type": "contains_any", "values": ["refund"]}],   # defaults for the route
      "setup": "checks:setup_chat",     # optional callable run once before the route
      "monthly_volume": 1500000,        # optional: tasks/month for projections
      "pace_ms": 0                      # optional: sleep between cases, so time-based cache breakers
                                        # (timestamps, per-second values) behave like production
    }
  }
}
Check types: exact, equals_expected, contains, contains_all, contains_any, not_contains, regex,
json_valid, json_has_keys, json_field_equals{path,value}, max_chars, min_chars,
similar_to_expected{threshold}, python{callable -> bool | float(0-1) | {"pass","score","reason"}}.
"""
import argparse
import asyncio
import difflib
import fnmatch
import importlib
import inspect
import json
import math
import os
import re
import statistics
import subprocess
import sys
import tempfile
import time
import urllib.request
from collections import defaultdict

HERE = os.path.dirname(os.path.abspath(__file__))
sys.path.insert(0, HERE)
import llm_meter  # noqa: E402

TOKEN_KEYS = ("input", "cache_read", "cache_write", "cache_write_long", "output")


# ---------------------------------------------------------------- helpers
def load_json(p):
    with open(p, encoding="utf-8") as f:
        return json.load(f)


def load_jsonl(p):
    out = []
    with open(p, encoding="utf-8") as f:
        for n, line in enumerate(f, 1):
            line = line.strip()
            if line and not line.startswith("//"):
                try:
                    out.append(json.loads(line))
                except ValueError as e:
                    sys.exit("%s:%d invalid JSON: %s" % (p, n, e))
    return out


def resolve(base, p):
    return p if os.path.isabs(p) else os.path.normpath(os.path.join(base, p))


def import_callable(spec):
    mod, _, attr = spec.partition(":")
    obj = importlib.import_module(mod)
    for part in attr.split("."):
        obj = getattr(obj, part)
    return obj


def to_text(x):
    if x is None:
        return ""
    if isinstance(x, str):
        return x
    for m in ("model_dump", "dict"):
        if hasattr(x, m):
            try:
                return json.dumps(getattr(x, m)(), default=str, ensure_ascii=False)
            except Exception:
                pass
    try:
        return json.dumps(x, default=str, ensure_ascii=False)
    except Exception:
        return str(x)


def get_path(obj, path):
    for part in str(path).split("."):
        if isinstance(obj, list) and part.isdigit():
            obj = obj[int(part)]
        elif isinstance(obj, dict):
            obj = obj.get(part)
        else:
            return None
    return obj


def git_sha(cwd):
    try:
        return subprocess.check_output(["git", "rev-parse", "--short", "HEAD"], cwd=cwd, stderr=subprocess.DEVNULL).decode().strip()
    except Exception:
        return None


def git_diff_stat(cwd, exclude=None):
    try:
        cmd = ["git", "diff", "--stat", "--", "."]
        if exclude:
            cmd.append(":(exclude)%s" % os.path.relpath(exclude, cwd))
        out = subprocess.check_output(cmd, cwd=cwd, stderr=subprocess.DEVNULL).decode().strip().splitlines()
        return out[-1].strip() if out else None
    except Exception:
        return None


def git_dirty(cwd, exclude=None):
    try:
        cmd = ["git", "status", "--porcelain", "--", "."]
        if exclude:
            cmd.append(":(exclude)%s" % os.path.relpath(exclude, cwd))
        return bool(subprocess.check_output(cmd, cwd=cwd, stderr=subprocess.DEVNULL).strip())
    except Exception:
        return None


# ---------------------------------------------------------------- pricing
def price_for(prices, model):
    models = prices.get("models", {})
    if not model:
        return None, None
    if model in models:
        return models[model], model
    best = None
    for pat in models:
        if fnmatch.fnmatch(model, pat) or model.startswith(pat.rstrip("*")) and pat.endswith("*"):
            if best is None or len(pat) > len(best):
                best = pat
    return (models[best], best) if best else (None, None)


def call_cost(rec, prices):
    """USD for one call record, or None if unpriced. Prices are per 1M tokens."""
    t = rec.get("tokens")
    if not t:
        return None
    p, _ = price_for(prices, rec.get("model") or rec.get("requested_model"))
    if p is None:
        p, _ = price_for(prices, rec.get("requested_model"))
    if p is None:
        return None
    inp = p["input"]
    rate = {"input": inp, "output": p.get("output", 0.0),
            "cache_read": p.get("cache_read", inp),
            "cache_write": p.get("cache_write", inp),
            "cache_write_long": p.get("cache_write_long", p.get("cache_write", inp))}
    usd = sum(t.get(k, 0) * rate[k] for k in TOKEN_KEYS) / 1e6
    mult = prices.get("tier_multipliers", {}).get(rec.get("tier") or "standard", 1.0)
    return usd * mult


# ---------------------------------------------------------------- checks
def run_check(chk, output, case):
    t = chk.get("type")
    text = to_text(output)
    exp = case.get("expected")
    ci = chk.get("case_insensitive", True)
    norm = (lambda s: s.lower()) if ci else (lambda s: s)
    try:
        if t == "exact":
            return text.strip() == str(chk["value"]).strip(), None
        if t == "equals_expected":
            return norm(text.strip()) == norm(to_text(exp).strip()), None
        if t == "contains":
            return norm(str(chk["value"])) in norm(text), None
        if t == "contains_all":
            miss = [v for v in chk["values"] if norm(str(v)) not in norm(text)]
            return not miss, ("missing %s" % miss) if miss else None
        if t == "contains_any":
            return any(norm(str(v)) in norm(text) for v in chk["values"]), None
        if t == "not_contains":
            bad = [v for v in chk.get("values", [chk.get("value")]) if v is not None and norm(str(v)) in norm(text)]
            return not bad, ("contains %s" % bad) if bad else None
        if t == "regex":
            return re.search(chk["pattern"], text, 0 if not ci else re.I) is not None, None
        if t in ("json_valid", "json_has_keys", "json_field_equals"):
            obj = output if isinstance(output, (dict, list)) else json.loads(text)
            if t == "json_valid":
                return True, None
            if t == "json_has_keys":
                miss = [k for k in chk["keys"] if get_path(obj, k) is None]
                return not miss, ("missing keys %s" % miss) if miss else None
            v = get_path(obj, chk["path"])
            want = chk["value"] if "value" in chk else get_path(exp, chk.get("expected_path", chk["path"]))
            return v == want, None if v == want else "got %r want %r" % (v, want)
        if t == "max_chars":
            return len(text) <= int(chk["value"]), "len %d" % len(text)
        if t == "min_chars":
            return len(text) >= int(chk["value"]), "len %d" % len(text)
        if t == "similar_to_expected":
            r = difflib.SequenceMatcher(None, norm(text), norm(to_text(exp))).ratio()
            return r >= float(chk.get("threshold", 0.8)), "similarity %.2f" % r
        if t == "python":
            fn = import_callable(chk["callable"])
            r = fn(output, case)
            if inspect.isawaitable(r):
                r = asyncio.run(r) if not _loop_running() else None
            if isinstance(r, dict):
                return bool(r.get("pass")), r.get("reason") or ("score %s" % r.get("score") if "score" in r else None)
            if isinstance(r, (int, float)) and not isinstance(r, bool):
                return r >= float(chk.get("threshold", 0.5)), "score %.3f" % r
            return bool(r), None
    except Exception as e:
        return False, "check error: %s: %s" % (type(e).__name__, e)
    return False, "unknown check type %r" % t


def _loop_running():
    try:
        asyncio.get_running_loop()
        return True
    except RuntimeError:
        return False


# ---------------------------------------------------------------- targets
def call_python(fn, style, inp):
    if style == "kwargs":
        r = fn(**(inp or {}))
    elif style == "args":
        r = fn(*(inp or []))
    else:
        r = fn(inp)
    if inspect.isawaitable(r):
        r = asyncio.run(_await(r))
    if inspect.isgenerator(r):  # streamed text generators
        r = "".join(to_text(x) for x in r)
    return r


async def _await(x):
    return await x


def _sitecustomize_dir():
    d = os.path.join(tempfile.gettempdir(), "llm_meter_site")
    os.makedirs(d, exist_ok=True)
    with open(os.path.join(d, "sitecustomize.py"), "w") as f:
        f.write("import os, sys\nif os.environ.get('LLM_METER_LOG'):\n"
                "    sys.path.insert(0, %r)\n    try:\n        import llm_meter; llm_meter.install()\n"
                "    except Exception as e:\n        sys.stderr.write('[llm_meter] install failed: %%s\\n' %% e)\n" % HERE)
    return d


def call_command(tgt, inp, case_id, trial, label, log, cwd):
    env = dict(os.environ)
    env.update({k: str(v) for k, v in (tgt.get("env") or {}).items()})
    env.update({"LLM_METER_LOG": log, "LLM_METER_LABEL": label, "LLM_METER_CASE": case_id, "LLM_METER_TRIAL": str(trial)})
    env["PYTHONPATH"] = os.pathsep.join([_sitecustomize_dir(), env.get("PYTHONPATH", "")]).strip(os.pathsep)
    if tgt.get("proxy"):
        req = urllib.request.Request(tgt["proxy"].rstrip("/") + "/__case", method="POST",
                                     data=json.dumps({"case": case_id, "trial": trial, "run": label}).encode(),
                                     headers={"content-type": "application/json"})
        urllib.request.urlopen(req, timeout=10).read()
    stdin = json.dumps(inp) if tgt.get("stdin", "json") == "json" else to_text(inp)
    p = subprocess.run(tgt["cmd"], input=stdin, capture_output=True, text=True, cwd=cwd, env=env,
                       timeout=tgt.get("timeout_s", 900))
    if p.returncode != 0:
        raise RuntimeError("exit %d: %s" % (p.returncode, p.stderr[-2000:]))
    out = p.stdout.strip()
    try:
        return json.loads(out)
    except ValueError:
        return out


# ---------------------------------------------------------------- run
def calls_for(log, label):
    if not os.path.exists(log):
        return []
    return [r for r in load_jsonl(log) if r.get("run") == label]


def cmd_run(a):
    cfg_path = os.path.abspath(a.config)
    base = os.path.dirname(cfg_path)
    cfg = load_json(cfg_path)
    root = resolve(base, cfg.get("project_root", ".."))
    sys.path.insert(0, root)
    sys.path.insert(0, base)  # so "checks:fn" / "targets:fn" in the eval dir import directly
    os.chdir(root)
    trials = a.trials or cfg.get("trials", 3)
    runs_dir = os.path.join(base, "runs")
    os.makedirs(runs_dir, exist_ok=True)
    log = os.path.join(runs_dir, "%s.calls.jsonl" % a.label)
    out_path = os.path.join(runs_dir, "%s.json" % a.label)
    if os.path.exists(out_path) and not a.overwrite:
        sys.exit("run '%s' exists; pass --overwrite or use a new label" % a.label)
    if os.path.exists(log):
        os.remove(log)
    prices = load_json(resolve(base, cfg["prices"])) if cfg.get("prices") and os.path.exists(resolve(base, cfg["prices"])) else None
    budget = a.max_cost_usd if a.max_cost_usd is not None else cfg.get("max_cost_usd")
    if budget is not None and not prices:
        sys.exit("max_cost_usd set but no prices file - create prices.json first (budget guard needs it)")
    if not a.yes:
        sys.exit("refusing to spend money without --yes (get the user's approval, then pass --yes). "
                 "Use `estimate` first to show the expected cost.")
    hooks = llm_meter.install(log, a.label)
    sys.stderr.write("[llm_eval] run=%s trials=%d hooks=%s\n" % (a.label, trials, hooks))
    routes = {k: v for k, v in cfg["routes"].items() if not a.route or k in a.route}
    records, spent, aborted = [], 0.0, None
    started = time.time()
    for rname, r in routes.items():
        cases = load_jsonl(resolve(base, r["cases"]))
        if a.limit:
            cases = cases[:a.limit]
        if r.get("setup"):
            import_callable(r["setup"])()
        tgt = r["target"]
        if tgt.get("proxy") and not tgt.get("proxy_log"):
            sys.exit("route %s: proxy targets need \"proxy_log\" (the --log file llm_cost_proxy.py writes)" % rname)
        fn = import_callable(tgt["callable"]) if tgt["type"] == "python" else None
        for trial in range(trials):          # trial-major order: every case once, then again -> warm-cache trials
            for c in cases:
                if r.get("pace_ms"):
                    time.sleep(r["pace_ms"] / 1000.0)
                cid = "%s::%s" % (rname, c["id"])
                t0 = time.time()
                err, output = None, None
                with llm_meter.case(cid, trial):
                    try:
                        if fn:
                            output = call_python(fn, tgt.get("call", "kwargs"), c.get("input"))
                        else:
                            output = call_command(tgt, c.get("input"), cid, trial, a.label, log, root)
                    except Exception as e:
                        err = "%s: %s" % (type(e).__name__, str(e)[:500])
                lat = int((time.time() - t0) * 1000)
                results = []
                with llm_meter.case(cid + "::judge", trial):
                    if err is None:
                        for chk in (c.get("checks") or r.get("checks") or []):
                            ok, why = run_check(chk, output, c)
                            results.append({"check": chk.get("type"), "pass": ok, "why": why})
                passed = err is None and all(x["pass"] for x in results)
                records.append({"route": rname, "case": c["id"], "key": cid, "trial": trial, "tags": c.get("tags", []),
                                "output": to_text(output)[: a.max_output_chars], "error": err, "pass": passed,
                                "checks": results, "latency_ms": lat})
                mark = "PASS" if passed else ("ERROR" if err else "FAIL")
                sys.stderr.write("  %-5s %s [t%d] %dms%s\n" % (mark, cid, trial, lat, (" " + err) if err else ""))
                if budget is not None:
                    spent = sum(call_cost(x, prices) or 0 for x in calls_for(log, a.label))
                    if spent > budget:
                        aborted = "budget %.2f exceeded (spent %.4f)" % (budget, spent)
                        break
            if aborted:
                break
        if aborted:
            break
    run = {"label": a.label, "config": cfg_path, "project_root": root, "git_sha": git_sha(root), "git_dirty": git_dirty(root, base), "diff_stat": git_diff_stat(root, base),
           "started": started, "finished": time.time(), "trials": trials, "routes": list(routes), "aborted": aborted,
           "calls_log": os.path.basename(log), "note": a.note, "records": records,
           "extra_call_logs": sorted({resolve(base, r["target"]["proxy_log"]) for r in routes.values()
                                      if r["target"].get("proxy_log")})}
    with open(out_path, "w", encoding="utf-8") as f:
        json.dump(run, f, indent=1)
    sys.stderr.write("[llm_eval] wrote %s (%d case-trials)%s\n" % (out_path, len(records), " ABORTED: " + aborted if aborted else ""))
    if prices:
        print(render_report(summarize(run, prices, a.skip_first_trial)))


# ---------------------------------------------------------------- summarize
def pctl(xs, q):
    if not xs:
        return 0.0
    xs = sorted(xs)
    k = (len(xs) - 1) * q
    f, c = math.floor(k), math.ceil(k)
    return xs[f] if f == c else xs[f] + (xs[c] - xs[f]) * (k - f)


def summarize(run, prices, skip_first_trial=False):
    base = os.path.dirname(run["config"])
    calls = calls_for(os.path.join(base, "runs", run["calls_log"]), run["label"])
    for extra in run.get("extra_call_logs") or []:
        calls += calls_for(extra, run["label"])
    by_key = defaultdict(list)
    for c in calls:
        by_key[(c.get("case"), c.get("trial"))].append(c)
    cfg = load_json(run["config"]) if os.path.exists(run["config"]) else {"routes": {}}
    routes = {}
    unpriced_models, unmetered = set(), 0
    for rec in run["records"]:
        if skip_first_trial and rec["trial"] == 0 and run["trials"] > 1:
            continue
        R = routes.setdefault(rec["route"], {"tasks": 0, "passes": 0, "errors": 0, "costs": [], "tok": defaultdict(int),
                                             "calls": 0, "trunc": 0, "lat": [], "judge_cost": 0.0, "case_pass": defaultdict(list),
                                             "case_cost": defaultdict(list), "models": defaultdict(int), "unpriced": 0})
        R["tasks"] += 1
        R["passes"] += int(rec["pass"])
        R["errors"] += int(bool(rec["error"]))
        R["lat"].append(rec["latency_ms"])
        R["case_pass"][rec["case"]].append(rec["pass"])
        cost = 0.0
        for c in by_key.get((rec["key"], rec["trial"]), []):
            R["calls"] += 1
            R["models"][c.get("model") or c.get("requested_model") or "?"] += 1
            if not c.get("usage_found"):
                unmetered += 1
                continue
            for k in TOKEN_KEYS + ("reasoning",):
                R["tok"][k] += (c.get("tokens") or {}).get(k, 0)
            if str(c.get("stop_reason") or "").lower() in ("length", "max_tokens", "max_output_tokens", "incomplete"):
                R["trunc"] += 1
            usd = call_cost(c, prices)
            if usd is None:
                R["unpriced"] += 1
                unpriced_models.add(c.get("model") or c.get("requested_model"))
            else:
                cost += usd
        for c in by_key.get((rec["key"] + "::judge", rec["trial"]), []):
            R["judge_cost"] += call_cost(c, prices) or 0
        R["costs"].append(cost)
        R["case_cost"][rec["case"]].append(cost)
    out = {"label": run["label"], "git_sha": run.get("git_sha"), "git_dirty": run.get("git_dirty"), "note": run.get("note"),
           "prices_source": prices.get("source"), "prices_fetched": prices.get("fetched"), "aborted": run.get("aborted"),
           "unpriced_models": sorted(m for m in unpriced_models if m), "unmetered_calls": unmetered,
           "skip_first_trial": skip_first_trial, "routes": {}}
    for name, R in routes.items():
        n = max(1, R["tasks"])
        t = R["tok"]
        in_total = t["input"] + t["cache_read"] + t["cache_write"] + t["cache_write_long"]
        mean = statistics.mean(R["costs"]) if R["costs"] else 0.0
        sd = statistics.stdev(R["costs"]) if len(R["costs"]) > 1 else 0.0
        case_major = {k: (sum(v) * 2 > len(v)) for k, v in R["case_pass"].items()}
        vol = (cfg.get("routes", {}).get(name) or {}).get("monthly_volume")
        out["routes"][name] = {
            "tasks": R["tasks"], "pass_rate": R["passes"] / n, "errors": R["errors"],
            "cost_per_task": mean, "cost_sd": sd, "cost_se": sd / math.sqrt(n), "cost_p90": pctl(R["costs"], 0.9),
            "cost_max": max(R["costs"]) if R["costs"] else 0, "calls_per_task": R["calls"] / n,
            "tokens_per_task": {k: t[k] / n for k in TOKEN_KEYS + ("reasoning",)},
            "cache_hit_rate": (t["cache_read"] / in_total) if in_total else 0.0,
            "truncations": R["trunc"], "latency_p50_ms": pctl(R["lat"], 0.5), "latency_p90_ms": pctl(R["lat"], 0.9),
            "judge_cost_total": R["judge_cost"], "unpriced_calls": R["unpriced"], "models": dict(R["models"]),
            "case_majority_pass": case_major,
            "case_mean_cost": {k: statistics.mean(v) for k, v in R["case_cost"].items()},
            "monthly_volume": vol, "monthly_cost": (mean * vol) if vol else None,
        }
    return out


def money(x):
    if x is None:
        return "-"
    if abs(x) < 0.01:
        return "$%.6f" % x
    if abs(x) < 1:
        return "$%.4f" % x
    return "${:,.2f}".format(x)


def render_report(s):
    L = ["### Run `%s`%s" % (s["label"], " (commit %s%s)" % (s["git_sha"], "+dirty" if s["git_dirty"] else "") if s["git_sha"] else ""),
         "", "_Prices: %s (fetched %s)_" % (s["prices_source"] or "SOURCE NOT RECORDED", s["prices_fetched"] or "?"), ""]
    if s.get("aborted"):
        L += ["**ABORTED:** %s" % s["aborted"], ""]
    L += ["| Route | Tasks | Pass rate | Cost/task | p90 | Calls/task | Uncached in | Cache read | Cache write | Output (reasoning) | Cache hit | Truncated | p50 latency | Monthly |",
          "|---|---|---|---|---|---|---|---|---|---|---|---|---|---|"]
    for n, r in s["routes"].items():
        t = r["tokens_per_task"]
        L.append("| %s | %d | %.0f%% | %s | %s | %.1f | %.0f | %.0f | %.0f | %.0f (%.0f) | %.0f%% | %d | %.0fms | %s |" % (
            n, r["tasks"], 100 * r["pass_rate"], money(r["cost_per_task"]), money(r["cost_p90"]), r["calls_per_task"],
            t["input"], t["cache_read"], t["cache_write"] + t["cache_write_long"], t["output"], t["reasoning"],
            100 * r["cache_hit_rate"], r["truncations"], r["latency_p50_ms"], money(r["monthly_cost"])))
    warn = []
    if s["unmetered_calls"]:
        warn.append("%d call(s) returned no usage (e.g. OpenAI streams without stream_options.include_usage) - cost is UNDER-counted" % s["unmetered_calls"])
    if s["unpriced_models"]:
        warn.append("no price for models %s - add them to prices.json" % s["unpriced_models"])
    if any(r["judge_cost_total"] for r in s["routes"].values()):
        warn.append("judge/check LLM spend excluded from task cost: %s" % money(sum(r["judge_cost_total"] for r in s["routes"].values())))
    L += [""] + ["> ⚠ " + w for w in warn]
    return "\n".join(L)


def cmd_report(a):
    run = load_json(a.run)
    prices = load_json(a.prices)
    s = summarize(run, prices, a.skip_first_trial)
    if a.json:
        print(json.dumps(s, indent=1))
    else:
        print(render_report(s))


# ---------------------------------------------------------------- compare
def cmd_compare(a):
    prices = load_json(a.prices)
    runs = [summarize(load_json(p), prices, a.skip_first_trial) for p in a.runs]
    base = runs[0]
    L = ["## Measured cost comparison", "",
         "_Same prices for every run: %s (fetched %s). Baseline: `%s`.%s_" % (
             prices.get("source", "SOURCE NOT RECORDED"), prices.get("fetched", "?"), base["label"],
             " Cold first trial excluded." if a.skip_first_trial else ""), ""]
    routes = sorted({r for s in runs for r in s["routes"]})
    for rn in routes:
        L += ["### Route `%s`" % rn, "",
              "| Run | Pass rate | Δ pass (pts) | Cost/task | Δ vs baseline | Δ vs previous | Noise? | p90 cost | Calls/task | Cache hit | Output tok/task | Truncated | Monthly cost | Monthly saving |",
              "|---|---|---|---|---|---|---|---|---|---|---|---|---|---|"]
        prev = None
        b = base["routes"].get(rn)
        for s in runs:
            r = s["routes"].get(rn)
            if not r:
                continue
            def d(x, y):
                return "-" if not y or y["cost_per_task"] == 0 else "%+.1f%%" % (100 * (x["cost_per_task"] - y["cost_per_task"]) / y["cost_per_task"])
            noise = "-"
            if b and s is not base:
                se = math.sqrt(r["cost_se"] ** 2 + b["cost_se"] ** 2)
                noise = "within noise" if abs(r["cost_per_task"] - b["cost_per_task"]) <= max(2 * se, 1e-12) else "significant"
            dp = "-" if not b or s is base else "%+.1f" % (100 * (r["pass_rate"] - b["pass_rate"]))
            msave = (b["monthly_cost"] - r["monthly_cost"]) if (b and b.get("monthly_cost") is not None and r.get("monthly_cost") is not None and s is not base) else None
            L.append("| `%s` | %.0f%% | %s | %s | %s | %s | %s | %s | %.1f | %.0f%% | %.0f | %d | %s | %s |" % (
                s["label"], 100 * r["pass_rate"], dp, money(r["cost_per_task"]), "-" if s is base else d(r, b),
                "-" if prev is None or s is base else d(r, prev), noise, money(r["cost_p90"]), r["calls_per_task"],
                100 * r["cache_hit_rate"], r["tokens_per_task"]["output"], r["truncations"], money(r["monthly_cost"]), money(msave)))
            prev = r
        if b:
            for s in runs[1:]:
                r = s["routes"].get(rn)
                if not r:
                    continue
                reg = sorted(k for k, ok in b["case_majority_pass"].items() if ok and r["case_majority_pass"].get(k) is False)
                fixed = sorted(k for k, ok in b["case_majority_pass"].items() if not ok and r["case_majority_pass"].get(k))
                if reg or fixed:
                    L.append("")
                    if reg:
                        L.append("- `%s` **regressed** on: %s" % (s["label"], ", ".join(reg[:20])))
                    if fixed:
                        L.append("- `%s` newly passes: %s" % (s["label"], ", ".join(fixed[:20])))
        L.append("")
    for s in runs:
        if s["unmetered_calls"] or s["unpriced_models"] or s.get("aborted") or s.get("git_dirty"):
            L.append("> ⚠ `%s`: %s" % (s["label"], "; ".join(x for x in [
                "%d unmetered calls" % s["unmetered_calls"] if s["unmetered_calls"] else "",
                "unpriced models %s" % s["unpriced_models"] if s["unpriced_models"] else "",
                "aborted: %s" % s["aborted"] if s.get("aborted") else "",
                "working tree was dirty (record which diff was applied)" if s.get("git_dirty") else ""] if x)))
    L += ["", "_A change is kept only if pass rate holds (no regressions beyond noise) AND cost drops beyond noise._"]
    text = "\n".join(L)
    if a.md:
        with open(a.md, "w", encoding="utf-8") as f:
            f.write(text + "\n")
    print(text)


# ---------------------------------------------------------------- estimate
def cmd_estimate(a):
    cfg_path = os.path.abspath(a.config)
    base = os.path.dirname(cfg_path)
    cfg = load_json(cfg_path)
    trials = a.trials or cfg.get("trials", 3)
    total_cases = sum(len(load_jsonl(resolve(base, r["cases"]))) for r in cfg["routes"].values())
    if a.from_run:
        s = summarize(load_json(a.from_run), load_json(resolve(base, cfg["prices"])))
        per = {k: v["cost_per_task"] for k, v in s["routes"].items()}
        est = 0.0
        for k, r in cfg["routes"].items():
            n = len(load_jsonl(resolve(base, r["cases"])))
            est += per.get(k, 0) * n * trials
        print("Estimated cost of a %d-trial run over %d cases: %s (from run %s)" % (trials, total_cases, money(est), s["label"]))
    else:
        print("No previous run: %d cases x %d trials = %d task executions. Estimate per-task cost from the code's "
              "prompt sizes x verified prices (or run `run --limit 1 --trials 1 --yes` as a cheap probe, with approval)."
              % (total_cases, trials, total_cases * trials))


# ---------------------------------------------------------------- init
EXAMPLE_PRICES = {
    "source": "FILL IN: URL of the provider pricing page you fetched",
    "fetched": "YYYY-MM-DD",
    "currency": "USD",
    "unit": "per 1M tokens",
    "models": {
        "EXAMPLE-model-id*": {"input": 0.0, "output": 0.0, "cache_read": 0.0, "cache_write": 0.0, "cache_write_long": 0.0}
    },
    "tier_multipliers": {"standard": 1.0, "batch": 0.5},
    "_notes": "Use VERIFIED prices only. cache_write = short/default TTL write rate (omit when the provider has no write "
              "premium); cache_write_long = 1h/12h/24h TTL write rate. Patterns ending in * match by prefix."
}


def cmd_init(a):
    d = os.path.abspath(a.dir)
    os.makedirs(os.path.join(d, "cases"), exist_ok=True)
    os.makedirs(os.path.join(d, "runs"), exist_ok=True)
    files = {
        "eval.json": json.dumps({"name": a.name, "project_root": "..", "prices": "prices.json", "trials": 3, "max_cost_usd": 2.0,
                                 "routes": {"example": {"target": {"type": "python", "callable": "package.module:function", "call": "kwargs"},
                                                        "cases": "cases/example.jsonl",
                                                        "checks": [{"type": "min_chars", "value": 1}],
                                                        "monthly_volume": None}}}, indent=2),
        "prices.json": json.dumps(EXAMPLE_PRICES, indent=2),
        "cases/example.jsonl": json.dumps({"id": "c1", "input": {"question": "..."}, "expected": "...", "tags": ["typical"]}) + "\n",
        "checks.py": '"""Custom checks: fn(output, case) -> bool | float | {"pass": bool, "score": float, "reason": str}"""\n\n\n'
                     'def example(output, case):\n    return bool(output)\n',
        ".gitignore": "# keep runs/*.calls.jsonl - compare/report need them\n__pycache__/\n",
        "README.md": "# LLM cost eval\n\nGenerated by the llm-cost-optimizer skill.\n\n"
                     "```bash\npython3 $LCO/llm_eval.py estimate --config eval.json\n"
                     "python3 $LCO/llm_eval.py run --config eval.json --label baseline --yes\n"
                     "# apply ONE fix, then\npython3 $LCO/llm_eval.py run --config eval.json --label fix-CACHE-01 --yes\n"
                     "python3 $LCO/llm_eval.py compare --prices prices.json runs/baseline.json runs/fix-CACHE-01.json\n```\n",
    }
    for rel, content in files.items():
        p = os.path.join(d, rel)
        if os.path.exists(p) and not a.force:
            print("exists, skipped: %s" % p)
            continue
        with open(p, "w", encoding="utf-8") as f:
            f.write(content)
        print("wrote %s" % p)


def main():
    ap = argparse.ArgumentParser(description=__doc__, formatter_class=argparse.RawDescriptionHelpFormatter)
    sub = ap.add_subparsers(dest="cmd", required=True)
    p = sub.add_parser("init"); p.add_argument("--dir", default="llm_cost_eval"); p.add_argument("--name", default="project")
    p.add_argument("--force", action="store_true"); p.set_defaults(fn=cmd_init)
    p = sub.add_parser("run"); p.add_argument("--config", required=True); p.add_argument("--label", required=True)
    p.add_argument("--trials", type=int); p.add_argument("--route", action="append"); p.add_argument("--limit", type=int)
    p.add_argument("--max-cost-usd", type=float); p.add_argument("--yes", action="store_true")
    p.add_argument("--overwrite", action="store_true"); p.add_argument("--note", default=None)
    p.add_argument("--max-output-chars", type=int, default=4000); p.add_argument("--skip-first-trial", action="store_true")
    p.set_defaults(fn=cmd_run)
    p = sub.add_parser("report"); p.add_argument("--run", required=True); p.add_argument("--prices", required=True)
    p.add_argument("--json", action="store_true"); p.add_argument("--skip-first-trial", action="store_true"); p.set_defaults(fn=cmd_report)
    p = sub.add_parser("compare"); p.add_argument("--prices", required=True); p.add_argument("runs", nargs="+")
    p.add_argument("--md"); p.add_argument("--skip-first-trial", action="store_true"); p.set_defaults(fn=cmd_compare)
    p = sub.add_parser("estimate"); p.add_argument("--config", required=True); p.add_argument("--trials", type=int)
    p.add_argument("--from-run"); p.set_defaults(fn=cmd_estimate)
    a = ap.parse_args()
    a.fn(a)


if __name__ == "__main__":
    main()
`````

---

## Maintaining this skill

- **Version**: `metadata.version` uses semver. Patch = wording or check text fixes; minor = new checks, provider updates, tool features; major = changed check IDs (avoid), report format or `.llm-cost.yaml` schema.
- **Check IDs are permanent.** Teams reference them in `.llm-cost.yaml` and code comments. Add new checks at the end of a category; never renumber or reuse an ID. Deprecate by marking "(retired)".
- **Provider facts drift.** Re-verify Appendix B against the linked docs at least quarterly and note the date.
- **Changing an embedded tool**: edit it, re-run its tests, regenerate its `sha256=` marker, and bump the version.

### Changelog
- **1.1.0** (2026-10-21): changes from the first test round (3 test projects, compared with and without the skill).
  - Scanner: finds raw-HTTP provider calls (no SDK), prompts in files without provider imports, large non-`prompt`-named string constants, copy-paste bloat (repeated-lines %), hand-written retry loops, and retries in handler files.
  - New checks: OBS-08 (stop reason unchecked), RESP-05 (batch reprocesses unchanged inputs), OUTPUT-09 (output requested then discarded), OUTPUT-10 (brittle parsing → paid retries), RELY-05 (webhook/event redelivery duplicates), MODEL-09 (deprecation or scheduled price change).
  - Workflow: follow-up-review mode, lever-interaction and model-lifecycle checks, lighter intent questions (≤2 rounds, conflicts between user answers and existing comments surfaced), dollar figures allowed when labelled assumption, a measurement plan when measuring is declined, fixes on a branch or as patches when app edits aren't approved, production-like cases (`pace_ms`, varied per-user values).
  - Harness: eval dir importable (`checks:`), `.gitignore` keeps call logs, dirty check ignores the harness and records a diff stat, `pace_ms` per route; `cost_model.py` gains `after` overrides for any lever.
- **1.0.0** (2026-10-21): first shared release. Deep route understanding (route cards), 68-check catalog plus open web research, mandatory intent questions, estimated vs measured reduction table, embedded metering harness (in-process meter, recording proxy, eval runner with budget guard, noise and regression detection).
