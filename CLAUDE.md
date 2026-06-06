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

## Helper agents

Seven subagents live in `.claude/agents/`, each owning one subsystem of the pipeline:
`solution-architect`, `jira-integration-specialist`, `agent-sdk-orchestrator`,
`sandbox-runtime-engineer`, `github-pr-manager`, `frontend-ui-builder`, `security-reviewer`.
