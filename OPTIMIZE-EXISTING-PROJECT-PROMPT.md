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

---

## Why we optimize existing codebases — THE CORE PROBLEM

AI coding tools have a **context window** — a limit on how much code they can "see" at once.
Every file the AI reads costs tokens. When a repo has 18,000 files but we only work on 50 of
them, the AI wastes most of its budget reading irrelevant code. This makes it:
- **Slower** — more time reading, less time building
- **More expensive** — every token costs money
- **Dumber** — the AI's attention is spread across thousands of files that don't matter,
  so it understands the important files less deeply

**The goal of optimization is to make sure the AI only ingests the code that matters.**

There are two ways to shrink what the AI sees:

1. **Delete** files and directories we will never modify. If it's a fork, anything deleted
   can be restored with `git checkout upstream/main -- path/to/deleted/thing`. Never
   hesitate to delete. It is always reversible.

2. **Ignore** files and directories we need in git but don't need the AI to read.
   This is done via ignore files (`.claudeignore`, `.cursorignore`, `.copilotignore`).
   **This is the preferred method** because it reduces what the AI indexes without changing
   the repo contents. The AI simply won't read those files unless explicitly asked to.

**Always prefer `.claudeignore` / `.cursorignore` over deletion** when the files might be
needed for builds, tests, or upstream merges. Only delete when the files are truly dead weight
(vendor documentation sites, example apps, desktop companion apps we'll never use, etc.).

---

## The codebase map — AGENTS.md

Every project must have an `AGENTS.md` file at the repo root. This is the **codebase map**.

**The problem it solves:** When an AI gets a task like "add a new cron job", it doesn't know
which of the 500 files in the project to read. Without guidance, it will explore directories,
read wrong files, backtrack, and burn thousands of tokens just figuring out where things live.
That exploration costs real money and eats into the context window that should be spent on
the actual work.

**What a good codebase map does:** It gives the AI a **Task → File Navigation Map** so it can
go directly to the right files without exploring. For example:

```
### Adding a new cron job
Touch these files IN ORDER:
1. Create `src/modules/pop-creations/crons/jobs/[name].cron.job.ts`
2. Add command to `src/modules/pop-creations/crons/commands/pop-creations-cron.commands.ts`
3. Register both in `src/modules/pop-creations/pop-creations.module.ts`
Do not touch anything else.
```

With this, the AI reads 3 files instead of 50. That's a 90% reduction in wasted tokens for
every single task.

**The sections that reduce token waste the most:**

1. **Task → File Navigation Map** — "if you need to do X, touch ONLY these files." One entry
   per common task type. The more specific, the better. This is the single highest-ROI section.

2. **The Prime Directive** — "our code lives HERE. Everything else is off-limits unless you
   have a specific reason." This prevents the AI from reading or modifying third-party code.

3. **What to Ignore** — lists directories the AI should skip. Works alongside `.claudeignore`
   but as human-readable guidance for AI tools that don't support ignore files.

4. **Idiosyncratic Decisions** — documents things that look wrong but are intentional, so the
   next AI session doesn't waste time "fixing" something that isn't broken (or worse, breaking
   something that was carefully designed).

AGENTS.md must be **kept current with every change.** If you add a route, update the navigation
map. If you make a weird architectural choice, document it in the Idiosyncratic Decisions section
before the session ends. A stale AGENTS.md is worse than no AGENTS.md because it actively
misleads future sessions into reading wrong files and making wrong assumptions.

See `~/Albert-AI-Standards/NEW-PROJECT-PROMPT.md` for the full list of required sections.

---

## Branch strategy — CRITICAL RULE

**There are NO branches. Everything commits to `main`. Always.**

Do not create feature branches. Do not create pull requests between branches. Do not ask
"should I create a branch for this?" The answer is always no.

Every commit goes directly to `main`. Every push goes to `main`. The CI/CD pipeline triggers
on push to `main` and deploys automatically.

Why: Albert is not a developer. Branch management, merge conflicts between branches, and PR
review workflows add complexity with zero value in a solo-AI-developer workflow. The code
either works or it doesn't, and `main` is always the truth.

---

## Deployment pipeline — how code gets to production

```
You commit code and push to main
        ↓
GitHub Actions triggers automatically (on push to main)
        ↓
Builds a Docker image from the project's Dockerfile
        ↓
Pushes the image to GitHub Container Registry (GHCR):
  ghcr.io/u2giants/[project-name]:latest
  ghcr.io/u2giants/[project-name]:main
  ghcr.io/u2giants/[project-name]:sha-[commit-hash]
        ↓
GitHub Actions calls the Coolify API to trigger a redeploy
        ↓
Coolify pulls the new :latest image and restarts the containers
        ↓
Live at the production URL
```

**Key principles:**

- **Coolify is a consumer of pre-built images, not a builder.** It does not build from source.
  It does not run `npm install`. It pulls a finished Docker image from GHCR and runs it. Period.
- **GitHub is the source of truth.** No changes are ever made to running containers directly.
  If something is broken in production, fix the code, push to main, let the pipeline deploy.
- **Every deployment is traceable.** The `:sha-[commit]` tag means any running container can
  be traced back to the exact commit that produced it.
- **Rollback** = tell Coolify to pull a previous `:sha-[commit]` tag instead of `:latest`.
- **Never modify a running container.** Not via SSH, not via Coolify console, not via
  `docker exec`. Code change → commit → push → deploy. No exceptions.

---

## What I need you to do — in this exact order

### Step 1: Understand the project

Read the existing codebase structure:
- What is this project? (README, existing docs, main entry points)
- What framework or platform is it built on?
- Is this a fork of something? If so, what?
- How does it deploy? Is there an existing CI/CD setup?

Do not write any code yet. Just read.

### Step 2: Count and categorize files

The purpose of this step is to understand how big the codebase is and find what can be
trimmed to reduce AI context waste.

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

### Step 3: Propose what to ignore and what to delete

For each large directory, assess: "Is this part of what we actively build and deploy?"

Present a table like:
| Directory | Files | Action | Reason | Recoverable? |
|---|---|---|---|---|
| `packages/twenty-docs` | 2,634 | DELETE | Documentation website, not our app | `git checkout upstream/main -- packages/twenty-docs` |
| `packages/twenty-front/src/modules/workflow/` | 800 | IGNORE | We use it but never modify it | Remove from `.claudeignore` when needed |
| `packages/twenty-front/src/modules/pop-creations/` | 50 | KEEP | This is our code | — |

**Decision guide:**
- **DELETE** = files we will never need and that can be restored from upstream if we're wrong
- **IGNORE** = files that must exist in git (for builds, dependencies, upstream merges) but
  that the AI should not read during normal development
- **KEEP** = files the AI should always have access to (our custom code, config files, etc.)

Wait for my approval before deleting anything. I will say "go ahead" or "keep X".

### Step 4: Execute approved changes

1. Delete approved directories
2. Update any workspace/package configuration files that reference deleted packages
3. Create `.claudeignore` and `.cursorignore` with identical content:
   ```
   # Build artifacts and tooling
   dist/
   node_modules/
   .cache/
   coverage/

   # [Large directories we use but never modify — add project-specific entries]
   ```

### Step 5: Create `AGENTS.md` — the codebase map

Write a comprehensive developer guide following the standard structure in
`~/Albert-AI-Standards/NEW-PROJECT-PROMPT.md` (the "Required sections" list).

Make it specific to THIS project. Do not copy generic text — every section should
contain concrete file paths, actual decisions made, and real identifiers.

The most important sections for an existing project:
- **The Prime Directive** — where does our custom code live? What directories are off-limits?
- **Idiosyncratic decisions** — document every weird thing you noticed while reading
  the codebase that a new developer might misread as a mistake
- **Core modification inventory** — if this is a fork, list every file changed from stock
- **Task → file navigation map** — the most used tasks should map to specific files
- **Business context** — explain the business in enough detail that the AI understands
  *why* things are built the way they are, not just *what* they do

### Step 6: Create `CLAUDE.md`

Short file, Claude-specific only. See standard in NEW-PROJECT-PROMPT.md.
Include the operations permissions (SSH, Coolify, API tokens I can use).

### Step 7: Verify or set up the DevOps pipeline

Check if a GitHub Actions → GHCR → Coolify pipeline exists.
If not, set it up following the standard in NEW-PROJECT-PROMPT.md.
Ask me for any missing secrets (Coolify API token, app UUID, etc.).

**Remember:** everything commits to `main`. The workflow triggers on `push` to `main`.
No branch protection, no PR requirements, no merge queues.

### Step 8: Commit and report

Commit everything to main with message:
`chore: add AGENTS.md, .claudeignore, optimize repo structure`

Push to main.

Report back:
- File count before and after
- What was deleted (and how to restore if needed)
- What was added to `.claudeignore`
- What was documented in AGENTS.md
- Any credentials or access you still need
- Anything idiosyncratic you found that Albert should know about
- Suggested first development task based on what you read

---

## Ongoing documentation maintenance — NON-NEGOTIABLE

These rules apply to every session, not just the initial optimization:

1. **Update AGENTS.md with every significant change.** New route added → update navigation map.
   New service created → update decision tree. Bug fixed with unusual approach → add to
   Idiosyncratic Decisions. This is not optional. Do it before the session ends.

2. **Update `.claudeignore` when the repo structure changes.** New large vendor package added?
   Add it to the ignore list. Deleted a package that was in the ignore list? Remove the entry.

3. **Document every non-obvious decision at the time it is made.** Not later. Not "I'll
   remember." You won't, and the next AI session definitely won't.

4. **The new-developer test:** Before closing any session, ask: "If a brand new AI session
   opened this repo right now with no context, would AGENTS.md give it enough information to
   work productively without exploring the codebase from scratch?" If no, update it.

5. **Keep the Pending Work section current.** Complete items get marked done. New items get
   added. A stale pending section is actively harmful — it misleads future sessions into
   working on things that are already done.

---

## Important

After you finish this setup, you are the primary AI developer on this project.
Albert will give you tasks in plain business language. You translate them to code,
deploy them, and keep the documentation current with every change.
