# Dev Audit Skills

A collection of deep, self-contained [Claude](https://claude.com/claude-code) skills for real engineering work. Each skill is a **single `SKILL.md` file** that carries its entire workflow, a large stable-ID check catalog, and its own bundled **stdlib-only Python tools** (embedded as checksummed blocks and extracted at runtime — no `pip install`, no network unless you explicitly opt in). Every skill ships an offline self-test.

Works with Claude Code and any coding agent that has shell access.

## Skills

| Skill | What it does |
|---|---|
| [**LLM Cost Optimizer**](LLM_Cost_Optimization/SKILL.md) | Audit a GenAI / agentic app and cut cost per task **without losing quality** — then prove it with measured before/after numbers. |
| [**App Security Auditor**](App_Security_Auditor/SKILL.md) | Find, verify and prioritize real security vulnerabilities in any app, with deep coverage of vibe-coded and AI/LLM/agent apps. |

---

### 🪙 LLM Cost Optimizer

Find out what **your** LLM calls cost, why, and what to change to cost less per completed task — without lowering output quality. The unit is **cost per completed task**, not cost per token: a cheaper model that retries, or an agent that needs 3× the turns, is not cheaper.

- Reads the full LLM code path and writes a **route card** (contract) per route before suggesting anything.
- **74+ checks** across caching, token hygiene, routing, agent loops, batching, RAG, retries and budgets — plus open web research for your exact stack (new models, pricing changes, framework token pitfalls).
- Asks before assuming intent; records accepted trade-offs in `.llm-cost.yaml`.
- Estimates % savings per finding, then the bundled harness **measures real tokens and cost** from the provider's own usage data, before and after each fix, alongside quality checks.

**Bundled tools (Python 3.8+, stdlib only):** `scan_llm_usage.py` (inventory), `cost_model.py` (estimates + cache simulation), `llm_meter.py` (in-process usage meter), `llm_cost_proxy.py` (recording proxy for non-Python stacks), `llm_eval.py` (test cases + metering harness with a spend guard).

**Use it when** you want to reduce, review or benchmark LLM/token/API spend, wonder why an OpenAI/Anthropic/Gemini/Bedrock bill is high, want model routing, prompt caching, token limits or batching, or just shipped an LLM/RAG/agent app and want a cost review before launch — even if you never say "cost" (e.g. *"make this agent cheaper to run"*).

---

### 🛡️ App Security Auditor

Find the places where someone with less authority than they should have can read, change, destroy or exfiltrate something they shouldn't — then get the evidence, the blast radius, the priority, and the smallest fix. Something is only a **finding** when it crosses a concrete trust boundary with a concrete bad outcome. Defensive, source-first.

Tuned for **vibe-coded** apps (AI-generated Next.js / Supabase / Firebase / Stripe / LLM stacks) and their recurring failures — secrets in the client bundle, RLS off or `using(true)`, open Firebase rules, UI-only auth, client-trusted prices, prompt injection into real sinks, the agent "lethal trifecta", known-CVE frameworks, supply-chain worms — but it reasons about the specific app in front of it.

- **200+ checks** across 20 groups (secrets, auth, authZ, BaaS/RLS, injection, web/HTTP, API, files/SSRF, crypto, business logic, data exposure, LLM, agents, MCP, dependencies, CI/CD, cloud/IaC, framework config, mobile, AI-coding-agent hygiene).
- Maps to **OWASP Top 10 2025, LLM Top 10, Agentic Top 10 and MCP Top 10**, plus live research for current CVEs and provider advisories.
- A **verification gate**: no candidate becomes a finding until the boundary and the result are shown. Severity reflects demonstrated impact; off-repo facts become `needs_validation` with a safe way to check them.

**Bundled tools (Python 3.8+, stdlib only):**

| Tool | Purpose | Network |
|---|---|---|
| `sec_scan.py` | Static code + config + secrets + git-history scanner, route/trust inventory | none |
| `dep_audit.py` | Dependency & supply-chain audit (lockfiles, advisories, typosquats) | opt-in `--online` |
| `sec_probe.py` | Safe read-only prober for a deployment **you own** | gated |
| `llm_redteam.py` | Behavioral safety harness for an LLM feature | gated (calls your model) |
| `sec_report.py` | Validate & render `findings.json` → report / SARIF | none |

**Safety-first:** static tools are always safe (read-only, offline); anything live or paid is gated behind explicit flags and your confirmation. Secrets are redacted, never printed.

**Use it when** you want to find, review, audit, pentest or harden security, ask *"is my app safe / production-ready?"*, worry about exposed API keys, Supabase/Firebase rules, RLS, auth, prompt injection, insecure dependencies or leaked secrets, or shipped a vibe-coded / AI app and want a security pass before launch.

---

## How the skills are built

Every skill follows the same contract:

- **One file.** The whole skill — workflow, catalog, provider notes, templates, and tool source — lives in `SKILL.md`.
- **Embedded tools.** Python tools are stored as fenced blocks with `<!-- file: NAME sha256=... -->` markers and extracted on demand by a short, checksum-verifying one-liner in the skill's *Tools setup* section. Stdlib only — no installs.
- **Offline-safe by default.** Nothing leaves your machine unless you pass an explicit flag (and the skill tells you first).
- **Self-tested.** Each skill includes an offline self-test that builds throwaway fixtures (a vulnerable one and a hardened twin) and asserts the tools catch seeded issues and stay quiet on clean code.
- **Stable check IDs.** Catalog IDs are never renumbered, so you can reference and accept findings over time.

## Installing a skill

Point your agent at the skill folder, or drop the `SKILL.md` where your agent loads skills from. For Claude Code, place each folder under your skills directory (e.g. `~/.claude/skills/`) or your project's `.claude/skills/`. The skill extracts its own tools the first time it needs them; run the self-test it points you to before relying on it.

## License

MIT — see [LICENSE](LICENSE).

---
