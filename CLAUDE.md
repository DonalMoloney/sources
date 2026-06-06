# VelocityAI

## Project Goal

Research and build a UI app where a user can enter (or select) a Jira ticket and
invoke a target repository to implement the ticket automatically using the Claude
Agent SDK.

### Core flow
1. User enters a Jira ticket key (e.g. `PROJ-123`) in the UI.
2. App pulls the ticket details (summary, description, acceptance criteria,
   comments) from Jira via the Jira REST API / MCP.
3. App invokes the Claude Agent SDK against a target repo to implement the change.
4. Agent produces a branch / diff / PR with the implementation.
5. (Optional) Status flows back to Jira (comment, transition, link to PR).

### Open research questions
- UI stack (e.g. Next.js on Vercel) and how the user authenticates to Jira.
- Jira integration: REST API token vs. Atlassian MCP server vs. OAuth app.
- How to scope and sandbox the Claude Agent SDK run against the repo.
- Where the agent runs (local, Vercel Sandbox, CI) and how repo access is granted.
- Mapping ticket → branch/PR and reporting results back to Jira.

## Research findings (2026-06-06)

This pattern is well-established; we are not inventing it. Key precedents and building blocks:

- **deepsense.ai "AI Teammate"** — closest blueprint: Jira webhook → FastAPI endpoint
  → clone `main` → build a semantic codebase map → Claude implements in a loop with
  lint/test feedback → `PyGitHub`/`python-gitlab` open the PR → status back to Jira.
- **Anabranch** (HN build) — scores ticket complexity against a confidence threshold,
  only auto-implements "straightforward" tickets, PR for human review. Lesson: the hard
  parts are trust/reliability and ambiguous tickets, not the plumbing.
- **GitHub Copilot coding agent for Jira** and **OpenAI Codex Jira↔GitHub** — vendors
  already ship this; useful reference/competition.

### Engine: Claude Agent SDK
- `query({ prompt, options })` (TS) / `query(...)` (Python) runs the Claude Code agent
  loop headlessly. Relevant options: `cwd` (cloned repo), `allowedTools`
  (`Read/Edit/Write/Bash`), `permissionMode: "acceptEdits"`, `mcpServers` (wire Jira in),
  `hooks` (`PreToolUse`/`PostToolUse` guardrails + audit), `sessions` (resume/fork).
- **Managed Agents** is the alternative when we don't want to operate the sandbox —
  Anthropic hosts the agent + a per-session sandbox behind a REST API.

### Jira integration
- `sooperset/mcp-atlassian` (MCP): `jira_get_issue`, `jira_search` (JQL),
  `jira_create_issue`, `jira_update_issue`, `jira_transition_issue`, comments. Auth via
  API token (Cloud) or PAT (Server); run via `uvx`/Docker; plugs into SDK `mcpServers`.

### Two implementation paths
- **Path A (fast MVP):** UI pulls ticket → mirror as GitHub issue / `workflow_dispatch`
  → `anthropics/claude-code-action@v1` implements on a GitHub runner (runner = sandbox)
  → PR → status back to Jira. Minimal custom code.
- **Path B (full control):** UI → API → job queue → worker runs the Agent SDK against a
  cloned repo in Vercel Sandbox / Managed Agents → `gh`/Octokit opens the PR → Jira
  transition + comment.

### Security (load-bearing)
Jira ticket text is **untrusted input**. A disclosed `claude-code-action` flaw let a
malicious issue hijack a repo via prompt injection. Therefore: sandbox every run, scope
GitHub/Jira tokens to least privilege, treat ticket content as data (not instructions),
and **never auto-merge** — always a PR for human review.

### Recommended stack (working assumption)
Next.js on Vercel (UI + API routes) · Claude Agent SDK in TypeScript (orchestration) ·
`mcp-atlassian` (Jira) · `gh`/Octokit (PRs) · job queue for long runs (runs are minutes,
not request-time) · Vercel Sandbox or Managed Agents for isolation.

## Reference repositories

Verified GitHub repos to study and build on (checked 2026-06-06), grouped by role in the pipeline.

### Engine — Claude Agent SDK
- [`anthropics/claude-agent-sdk-python`](https://github.com/anthropics/claude-agent-sdk-python) — Python SDK; the headless `query(...)` agent loop for a Path B worker.
- [`anthropics/claude-agent-sdk-typescript`](https://github.com/anthropics/claude-agent-sdk-typescript) — TypeScript SDK; matches the recommended Next.js/TS stack.
- [`anthropics/claude-agent-sdk-demos`](https://github.com/anthropics/claude-agent-sdk-demos) — official demos, incl. a multi-agent research system and an autonomous coding agent (initializer + coder, progress persisted via git).
- [`anthropics/claude-code-action`](https://github.com/anthropics/claude-code-action) — GitHub Action that runs the agent on a runner; the backbone of Path A (also the source of the disclosed prompt-injection flaw — see Security).
- [`anthropics/claude-quickstarts`](https://github.com/anthropics/claude-quickstarts) — deployable starter apps on the Claude API.
- [`anthropics/claude-cookbooks`](https://github.com/anthropics/claude-cookbooks) — recipes for tool use, agents, and prompt patterns.

### Jira / Atlassian integration
- [`sooperset/mcp-atlassian`](https://github.com/sooperset/mcp-atlassian) — our chosen MCP server for Jira/Confluence (Cloud + Server, OAuth or API token); plugs into the SDK `mcpServers` option.

### PR plumbing
- [`octokit/octokit.js`](https://github.com/octokit/octokit.js) — official JS/TS GitHub client (matches the TS stack) for opening PRs.
- [`PyGithub/PyGithub`](https://github.com/PyGithub/PyGithub) — Python GitHub client for a Python worker.
- [`python-gitlab/python-gitlab`](https://github.com/python-gitlab/python-gitlab) — GitLab equivalent for non-GitHub targets.

### Precedents / patterns
- [`rpostulart/Claude-Project-Tracker`](https://github.com/rpostulart/Claude-Project-Tracker) — "Jira for AI agents": a `.project/` folder where the agent files its own tickets and decisions; a useful self-tracking pattern for long multi-session runs.
- [OpenAI Codex Jira↔GitHub cookbook](https://developers.openai.com/cookbook/examples/codex/jira-github) — label-triggered Jira→PR flow via a coding agent in a GitHub Action; direct competitor/reference for Path A.

## Helper agents

Seven subagents live in `.claude/agents/`, each owning one subsystem of the pipeline:
`solution-architect`, `jira-integration-specialist`, `agent-sdk-orchestrator`,
`sandbox-runtime-engineer`, `github-pr-manager`, `frontend-ui-builder`, `security-reviewer`.
