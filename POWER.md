# Making VelocityAI Powerful — Capabilities & Reference Repos

> Companion to `CLAUDE.md`. This doc is the ambition map: the capabilities that turn
> "enter a Jira ticket → get a PR" from a demo into a system teams trust in production,
> each paired with **real open-source repos to study and borrow from**.

The baseline (recorded in `CLAUDE.md`) is: UI → pull ticket → Claude Agent SDK implements
in a sandbox → PR → status back to Jira. Everything below is how we make that *powerful*.

---

## North star

> A teammate that reliably turns a well-scoped ticket into a reviewable PR, knows when it
> *can't* (and asks instead of guessing), proves its work with tests, and gets measurably
> better over time — across many repos and intake channels, safely.

The hard parts (per real builders) are **trust, verification, and ambiguous tickets** —
not the plumbing. The capabilities below are ordered by how much leverage they add.

---

## 1. Plan → Implement → Verify → Review loop (multi-agent)

Don't one-shot. Decompose into specialized stages: a **planner** turns the ticket into a
step plan, an **implementer** edits code, a **tester** runs the suite, a **reviewer**
critiques the diff before it ever reaches a human. The Claude Agent SDK supports this
natively via subagents (`AgentDefinition`) and the agent loop.

**Borrow from:**
- [`All-Hands-AI/OpenHands`](https://github.com/All-Hands-AI/OpenHands) — *(~68k★, MIT)* takes a task description and returns a PR: reads the codebase, writes a plan, executes in a sandbox, runs tests, fixes failures, hands you a diff. The closest end-to-end reference for our exact loop.
- [`princeton-nlp/SWE-agent`](https://github.com/princeton-nlp/SWE-agent) — research-grade agent-computer interface for resolving real GitHub issues; great for the agent↔repo interaction design.
- [`Aider-AI/aider`](https://github.com/Aider-AI/aider) — *(~41k★)* git-native edit/commit discipline and clean diffs.
- [`anthropics/claude-agent-sdk-demos`](https://github.com/anthropics/claude-agent-sdk-demos) — the **Research Agent** demo shows multi-agent coordination with parallel subagents; same pattern, applied to code.

## 2. Deep codebase understanding (repo map + retrieval)

The agent's output quality is gated by how well it *understands* the target repo. Build a
**semantic codebase map** (files, classes, signatures) and retrieval so the agent navigates
large/unfamiliar repos instead of guessing. deepsense.ai's build used exactly this.

**Borrow from:**
- [`Aider-AI/aider`](https://github.com/Aider-AI/aider) — its **repo map** (ranked tags of the codebase via tree-sitter) is the canonical reference for cheap, high-signal repo context.
- [`sweepai/sweep`](https://github.com/sweepai/sweep) — issue-to-PR with code search/retrieval over the repo.
- [`continuedev/continue`](https://github.com/continuedev/continue) — *(~31k★)* practical codebase indexing/retrieval patterns.
- [`TabbyML/tabby`](https://github.com/TabbyML/tabby) — self-hosted code intelligence/indexing.

## 3. Verification & self-healing (the test/lint/CI loop)

Power comes from the agent **proving** its change: run linters and the test suite after
each edit, feed failures back, and iterate until green or budget-exhausted. Then measure
yourself against a real benchmark so "did we get better?" is a number, not a vibe.

**Borrow from:**
- [`princeton-nlp/SWE-bench`](https://github.com/princeton-nlp/SWE-bench) — the standard benchmark of real GitHub issues; use it as our regression/eval harness for agent quality.
- [`All-Hands-AI/OpenHands`](https://github.com/All-Hands-AI/OpenHands) — runs your test suite in a sandbox and self-corrects on failures.
- Agent SDK `hooks` (`PostToolUse`) — run tests automatically after edits and gate progress.

## 4. Strong, fast sandbox isolation

Every run executes AI-generated code against **untrusted ticket input** — isolation is
non-negotiable (see the `claude-code-action` prompt-injection flaw in `CLAUDE.md`). Want
microVM-grade isolation with sub-200ms starts and ephemeral, stateful sessions.

**Borrow from / build on:**
- [`e2b-dev/E2B`](https://github.com/e2b-dev/E2B) — *(Apache-2.0)* open-source Firecracker-microVM sandboxes for AI agents; ~150–200ms starts, sessions up to 24h. Strong default for Path B.
- [`daytonaio/daytona`](https://github.com/daytonaio/daytona) — secure AI code-execution runtime + dev environments, <200ms starts, stateful persistence.
- [`firecracker-microvm/firecracker`](https://github.com/firecracker-microvm/firecracker) — the microVM tech underneath (per-sandbox kernel isolation).
- [`restyler/awesome-sandbox`](https://github.com/restyler/awesome-sandbox) — curated list to compare options (incl. microsandbox, gVisor).
- Managed alternatives: **Vercel Sandbox** and **Anthropic Managed Agents** (hosts the agent *and* a per-session sandbox — offloads ops entirely).

## 5. Autonomy levels & confidence-based triage

Not every ticket should be auto-implemented. Score each ticket's **complexity/clarity**
against a confidence threshold: auto-PR the clear ones, draft-only the medium ones, and
bounce the ambiguous ones back with a clarifying question. This is the single biggest
trust lever — the Anabranch builder found "too conservative barely helps, too aggressive
costs more than it saves."

**Borrow from:**
- **Anabranch** ([HN writeup](https://news.ycombinator.com/item?id=47092994)) — confidence threshold + complexity limits + safety constraints, configurable.
- [Port guide: resolve tickets with coding agents](https://docs.port.io/guides/all/automatically-resolve-tickets-with-coding-agents/) — "one-shot fix" gating patterns.

## 6. Human-in-the-loop checkpoints

Powerful ≠ unattended. Let the agent **ask before guessing** on ambiguity, propose a plan
for approval before editing, and never auto-merge. The Agent SDK ships `AskUserQuestion`
and plan mode for exactly this.

**Borrow from:**
- [`anthropics/claude-agent-sdk-demos`](https://github.com/anthropics/claude-agent-sdk-demos) — the **AskUserQuestion previews** demo (`canUseTool` callback, plan mode, HTML preview cards) — surface clarifying questions and plan approvals in our UI.

## 7. Omni-channel intake (beyond Jira)

The pipeline shouldn't care where the work came from. Abstract "ticket" into a common brief
so Jira, Linear, GitHub Issues, and Slack can all trigger a run through one engine.

**Borrow from:**
- [`sooperset/mcp-atlassian`](https://github.com/sooperset/mcp-atlassian) — Jira/Confluence MCP (`jira_get_issue`, JQL, transitions, comments).
- [`modelcontextprotocol/servers`](https://github.com/modelcontextprotocol/servers) — reference MCP servers for many sources; wire any into the SDK's `mcpServers`.
- [GitHub Copilot coding agent for Jira](https://github.blog/changelog/2026-03-05-github-copilot-coding-agent-for-jira-is-now-in-public-preview/) and [OpenAI Codex Jira↔GitHub](https://developers.openai.com/cookbook/examples/codex/jira-github) — reference UX/integration for the same intake.
- The session's **Linear MCP** + **Vercel Chat SDK** (Slack/Teams/Discord) for chat-driven triggers.

## 8. Durable background execution & parallelism

Runs take minutes and must survive restarts; teams want many tickets implemented in
parallel. Use a **durable job queue** + worker pool, resumable runs, and parallel branches.

**Borrow from:**
- [`triggerdotdev/trigger.dev`](https://github.com/triggerdotdev/trigger.dev) — open-source durable background jobs with retries/concurrency; strong fit for the run queue.
- [`bradAGI/awesome-cli-coding-agents`](https://github.com/bradAGI/awesome-cli-coding-agents) — covers parallel runners & autonomous-loop harnesses.
- **Vercel Queues** + Fluid Compute, or **Coder Tasks** (background agent per task) as managed options.
- Agent SDK **sessions** (`resume`/fork) make individual runs resumable.

## 9. Memory & project knowledge that compounds

The agent should get smarter per repo over time: learn conventions, remember past decisions,
and respect a per-repo `CLAUDE.md`. Persist a decision log / wiki the agent reads and writes.

**Borrow from:**
- Per-repo `CLAUDE.md` (the target repo's own) — the SDK auto-loads it; quality of this file ≈ quality of output.
- **Claude Project Tracker** pattern (a `.project/` folder of git-tracked tickets/decisions/wiki, no DB) — give each repo a self-maintained memory.
- Agent SDK `settingSources` to control which `.claude/` config loads.

## 10. Self-review & quality gates before humans

Shrink human review time: the agent reviews its **own** diff, runs a security pass, and
attaches a summary + risk notes to the PR. Then respond to human review comments with a
follow-up run on the same branch.

**Borrow from:**
- [`anthropics/claude-code-action`](https://github.com/anthropics/claude-code-action) — `@claude` review/implement on PRs/issues; reference for the review-comment → follow-up-run loop. (Also the source of the security lessons — read its `docs/security.md`.)
- This repo's own `pr-review-toolkit` agents + `/code-review` for a self-review gate.

## 11. Observability, evals & cost control

To make it *trustworthy*, measure it: success rate, turns per task, cost per PR, and
human-edit-distance after merge. Route models by task difficulty and cap token/turn budgets.

**Borrow from:**
- [`princeton-nlp/SWE-bench`](https://github.com/princeton-nlp/SWE-bench) — quantify resolution rate as a tracked metric per release.
- **Vercel AI Gateway** — unified multi-provider routing, fallbacks, and per-request cost/observability.
- Agent SDK `--max-turns` + time budgets + `PreToolUse` hooks to enforce cost/safety ceilings.

---

## Consolidated reference-repo table

| Repo | What to borrow | Notes |
|---|---|---|
| [`All-Hands-AI/OpenHands`](https://github.com/All-Hands-AI/OpenHands) | Full task→plan→sandbox→test→PR loop | ~68k★, MIT — closest blueprint |
| [`princeton-nlp/SWE-agent`](https://github.com/princeton-nlp/SWE-agent) | Agent↔repo interface design | Research-grade |
| [`princeton-nlp/SWE-bench`](https://github.com/princeton-nlp/SWE-bench) | Eval/benchmark harness | Track quality as a number |
| [`Aider-AI/aider`](https://github.com/Aider-AI/aider) | Repo map + git-native diffs | ~41k★ |
| [`cline/cline`](https://github.com/cline/cline) | Plan/Act modes, MCP extensibility | ~58k★ |
| [`block/goose`](https://github.com/block/goose) | Extensible local agent + MCP | ~32k★ |
| [`continuedev/continue`](https://github.com/continuedev/continue) | Codebase indexing/retrieval | ~31k★ |
| [`TabbyML/tabby`](https://github.com/TabbyML/tabby) | Self-hosted code intelligence | ~33k★ |
| [`sweepai/sweep`](https://github.com/sweepai/sweep) | Issue→PR with code search | |
| [`e2b-dev/E2B`](https://github.com/e2b-dev/E2B) | Firecracker microVM sandboxes | Apache-2.0, ~150ms starts |
| [`daytonaio/daytona`](https://github.com/daytonaio/daytona) | Secure agent runtime + stateful sandboxes | <200ms starts |
| [`firecracker-microvm/firecracker`](https://github.com/firecracker-microvm/firecracker) | MicroVM isolation primitive | |
| [`restyler/awesome-sandbox`](https://github.com/restyler/awesome-sandbox) | Compare sandbox options | Curated list |
| [`triggerdotdev/trigger.dev`](https://github.com/triggerdotdev/trigger.dev) | Durable background jobs/queue | |
| [`anthropics/claude-code-action`](https://github.com/anthropics/claude-code-action) | Issue/PR automation + security lessons | Read `docs/security.md` |
| [`anthropics/claude-agent-sdk-demos`](https://github.com/anthropics/claude-agent-sdk-demos) | Subagents, AskUserQuestion, streaming UI | Official |
| [`sooperset/mcp-atlassian`](https://github.com/sooperset/mcp-atlassian) | Jira/Confluence MCP tools | Cloud + Server |
| [`modelcontextprotocol/servers`](https://github.com/modelcontextprotocol/servers) | MCP servers for many sources | Omni-channel intake |
| [`bradAGI/awesome-cli-coding-agents`](https://github.com/bradAGI/awesome-cli-coding-agents) | Harnesses, parallel runners | Curated list |
| [`Zijian-Ni/awesome-ai-agents-2026`](https://github.com/Zijian-Ni/awesome-ai-agents-2026) | Broad agent ecosystem map | Curated list |

> Star counts are approximate (as of research, 2026-06) and only meant to signal maturity.

---

## Suggested phasing (don't build it all at once)

1. **MVP (Path A):** ticket → `claude-code-action` on a GitHub runner → PR. Proves the loop.
2. **Control (Path B):** Agent SDK worker in **E2B/Daytona** sandbox + **trigger.dev** queue.
3. **Trustworthy:** verification loop (§3) + confidence triage (§5) + human checkpoints (§6).
4. **Powerful:** repo understanding (§2), self-review gates (§10), memory (§9).
5. **Scale:** omni-channel intake (§7), parallelism (§8), evals + cost control (§11).

Each phase is independently shippable and de-risks the next.
