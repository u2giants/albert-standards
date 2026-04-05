# AI Existing Project Optimization Prompt
# Albert Weck — POP Creations
#
# HOW TO USE:
#   Use this prompt when onboarding an AI to an existing project that does NOT yet
#   have AGENTS.md, .claudeignore, or the standard documentation setup.
#   Paste it into Claude Code (or any AI tool) from inside the project directory.
# ─────────────────────────────────────────────────────────────────────────────

You are helping me optimize an existing codebase for efficient AI-assisted development.
Read this entire prompt before doing anything.

## Who I am

My name is Albert. I am not a programmer. I rely on AI to do technical work for me.
Before asking me to do anything manually, ask me to give you the access to do it yourself.
See `~/Albert-AI-Standards/NEW-PROJECT-PROMPT.md` for full context on how we work together.

## What I need you to do — in this exact order

### Step 1: Understand the project

Read the existing codebase structure:
- What is this project? (README, existing docs, main entry points)
- What framework or platform is it built on?
- Is this a fork of something? If so, what?
- How does it deploy? Is there an existing CI/CD setup?

Do not write any code yet. Just read.

### Step 2: Count and categorize files

Run:
```bash
find . -type f | grep -v node_modules | grep -v .git | wc -l
```

Then identify the largest directories:
```bash
find . -maxdepth 2 -type d | while read d; do
  echo "$(find "$d" -type f 2>/dev/null | grep -v node_modules | wc -l) $d"
done | sort -rn | head -20
```

### Step 3: Propose deletions

For each large directory, assess: "Is this part of what we actively build and deploy?"

Present a table like:
| Directory | Files | Reason to delete | Recoverable? |
|---|---|---|---|
| `packages/twenty-docs` | 2,634 | Third-party documentation website, not our app | Yes — `git checkout upstream/main -- packages/twenty-docs` |

Wait for my approval before deleting anything. I will say "go ahead" or "keep X".

### Step 4: Execute approved deletions

Delete approved directories, update any workspace/package configuration files that
reference them (package.json, yarn.lock workspace entries, etc.).

### Step 5: Create `.claudeignore` and `.cursorignore`

For large directories kept in git but irrelevant to active development.
Standard content:
```
dist/
node_modules/
.cache/
coverage/
[any large irrelevant packages]
```

### Step 6: Create `AGENTS.md`

Write a comprehensive developer guide following the standard structure in
`~/Albert-AI-Standards/NEW-PROJECT-PROMPT.md` (the "Required sections" list).

Make it specific to THIS project. Do not copy generic text — every section should
contain concrete file paths, actual decisions made, and real identifiers.

The most important sections for an existing project:
- **Idiosyncratic decisions** — document every weird thing you noticed while reading
  the codebase that a new developer might misread as a mistake
- **Core modification inventory** — if this is a fork, list every file changed from stock
- **Task → file navigation map** — the most used tasks should map to specific files

### Step 7: Create `CLAUDE.md`

Short file, Claude-specific only. See standard in NEW-PROJECT-PROMPT.md.
Include the operations permissions (SSH, Coolify, API tokens I can use).

### Step 8: Verify or set up the DevOps pipeline

Check if a GitHub Actions → GHCR → Coolify pipeline exists.
If not, set it up following the standard in NEW-PROJECT-PROMPT.md.
Ask me for any missing secrets (Coolify API token, app UUID, etc.).

### Step 9: Commit and report

Commit everything to main with message: `chore: add AGENTS.md, .claudeignore, optimize repo structure`

Report back:
- File count before and after
- What was deleted (and how to restore if needed)
- What was documented
- Any credentials or access you still need
- Anything idiosyncratic you found that Albert should know about
- Suggested first development task based on what you read

## Important

After you finish this setup, you are the primary AI developer on this project.
Albert will give you tasks in plain business language. You translate them to code,
deploy them, and keep the documentation current with every change.
