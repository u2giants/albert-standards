# albert-standards

Prompt templates for onboarding AI coding tools to new projects and optimizing
them for existing ones. These are Albert's operational standards for how AI agents
should set up, document, and operate within any new project.

## Files

| File | Purpose |
|------|---------|
| [NEW-PROJECT-PROMPT.md](NEW-PROJECT-PROMPT.md) | Paste into any AI tool when starting a new project. The AI will ask for required credentials upfront, then set up the full project structure including documentation, CI/CD, and deployment. |
| [OPTIMIZE-EXISTING-PROJECT-PROMPT.md](OPTIMIZE-EXISTING-PROJECT-PROMPT.md) | Paste when onboarding an AI to an existing project for the first time. Triggers a codebase audit, doc cleanup, and standards alignment pass. |
| [1PASSWORD.md](1PASSWORD.md) | How to pull secrets from 1Password via the 1Password MCP server (for AI agents) and the `op` CLI (for humans, scripts, and CI), using Service Account tokens and `op://` secret references. |

## How to use

Copy the relevant prompt in full and paste it as the first message to Claude Code
(or any AI coding tool) at the start of a project session. The prompt is self-contained
and instructs the AI on operating model, documentation expectations, credential gathering,
and workflow conventions.

## What these enforce

- AI asks for all required credentials upfront (not piecemeal)
- Every project gets `AGENTS.md`, `CLAUDE.md`, and standard docs
- Single-branch (`main`) trunk workflow with GitHub → Coolify deployment
- No manual server editing; GitHub is source of truth
- AI executes tasks autonomously — no runbooks handed back to the owner
