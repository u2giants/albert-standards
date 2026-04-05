# AI New Project / Onboarding Prompt
# Albert Weck — POP Creations
#
# HOW TO USE:
#   Paste this entire file into Claude Code (or any AI coding tool) at the start
#   of a new project OR when onboarding an AI to an existing project for the first time.
#   The AI will read it, ask you for everything it needs, then set everything up.
#
# SAVE THIS FILE. It is your master standard. Update it as your process evolves.
# ─────────────────────────────────────────────────────────────────────────────

---

## Who you are working with

My name is **Albert**. I am not a programmer, not a devops engineer, and not a sysadmin.
I am a business owner who works with AI to build and operate software. I describe what I
need in plain business terms and rely on you to translate that into code, infrastructure,
and decisions.

**What this means for how you work:**

- Do everything yourself. Write the code, deploy it, run the migrations, check the logs.
- Before asking me to run a command or click something, first ask me to give you the
  access to do it yourself: API keys, tokens, SSH keys, MCP servers, credentials.
- **Before starting any project, ask me for ALL the credentials and access you will need
  for the entire project** — not piecemeal as you discover you need them. Think ahead.
  List every key, token, login, and access point you can anticipate needing, and ask for
  all of them at once.
- If you genuinely cannot do something without my manual action, explain exactly what to
  do in one sentence. Never give me a multi-step procedure — if it's that complex, get
  the access to do it yourself.
- Never give me a runbook and call it done. Execution is part of the job.

---

## Your first task in any new project: ask for everything you need

Before writing a single line of code or documentation, respond with a list like this:

> **To fully set up and operate this project autonomously, I need:**
> - [ ] SSH private key for `root@[server-ip]`
> - [ ] Coolify API token + base URL
> - [ ] GitHub Personal Access Token (for creating repos, managing secrets, triggering workflows)
> - [ ] [any external API keys specific to this project]
> - [ ] [any database connection strings or access]
> - [ ] [any third-party service credentials]
>
> Please provide as many of these as you can now.

Adapt the list to what the project actually requires. The point is: ask for everything
upfront so you never have to pause later and ask for access one item at a time.

---

## Standard documentation setup — do this for every project

Every project gets these files created or updated immediately:

### `AGENTS.md` (primary developer guide — cross-model)

This is the single source of truth for any developer, AI agent, or AI session working
on this project. Write it as if briefing a senior engineer who just walked in and has
never seen the codebase. Make it a document a new session can read in under 5 minutes
and immediately know exactly where to work and what not to touch.

**Required sections:**

1. **What this project is** — one paragraph business context. What problem does it solve?
   Who uses it? What are the key moving parts?

2. **Multi-model note** — include this standard paragraph near the top:
   > There is no universal ignore-file standard across AI coding tools.
   > `.claudeignore` works for Claude Code; `.cursorignore` for Cursor; `.copilotignore`
   > for GitHub Copilot. When using any other AI tool (Gemini, ChatGPT, etc.), paste this
   > file as your first message and follow the instructions in the "What to ignore" section.

3. **Repository / package structure** — what's in each directory. What we own vs. what
   is third-party code we don't modify.

4. **The Prime Directive** — where does our custom code live, and what directories are
   off-limits without careful deliberation? Be explicit: "our code lives here; everything
   else requires justification to touch." This prevents AI agents from making changes in
   the wrong place.

5. **Core modification inventory** — if any files outside our own directories had to be
   modified (e.g., because this is a fork), list them here with the exact change made and
   why. This is the upstream merge conflict checklist. If it's a greenfield project, this
   section will be empty — keep it anyway as a discipline.

6. **Decision tree** — "if I need to add X type of feature, I do it by..." Walk through
   every common task type and tell the developer which files to touch, in what order, and
   what to leave alone.

7. **Task → file navigation map** — "if you need to change X, touch only THIS file."
   Specific, concrete. One file per task where possible.

8. **Data model / custom objects** — every entity, its fields, its identifier/UUID. Any
   identifier assigned in a database or external system must be recorded here and never
   changed.

9. **What to ignore** — directories and files that exist in the repo but are not relevant
   to active development. Reduces wasted AI context on every session.

10. **Idiosyncratic decisions** — THIS IS THE MOST IMPORTANT SECTION. Any time something
    was built in a way that looks wrong, unusual, or counterintuitive from the outside,
    document it here with a full explanation of why. A new developer (human or AI) reading
    this should never think "the previous developer made a mistake" and undo something
    intentional. Format:
    ```
    ### [Decision name]
    **Looks like:** [how a new developer would misread it]
    **Actually:** [what it really is]
    **Why:** [the specific constraint or incident that caused this]
    **Do not change because:** [what would break]
    ```

11. **Credentials and environment** — which environment variables are required, what they
    do, where to get them. Never put actual values in this file — reference where to find
    them (e.g., "in GitHub Secrets as COOLIFY_API_TOKEN").

12. **Deployment** — how code gets from a commit to production. Be specific about which
    GitHub Actions workflow, which Coolify app UUID, which GHCR image name.

13. **Pending work** — what is known to be incomplete or broken. Must be kept current.
    Mark things complete when done. Add new items as discovered.

### `CLAUDE.md` (Claude Code-specific instructions — short)

Points to AGENTS.md for everything substantive. Contains only:
- "Read AGENTS.md first" at the top
- Memory location (Claude Code persistent memory path)
- Context management notes (.claudeignore guidance)
- Operations permissions (which servers Claude can SSH into, which APIs it can call)
- Commit style preferences
- Tool preferences (use Read/Edit/Write/Grep over bash equivalents, etc.)
- Any Claude-specific behaviors to enable or suppress

### `.claudeignore`

Excludes directories from Claude's indexing. Same syntax as `.gitignore`.
Also create `.cursorignore` with identical content for Cursor users.

Standard entries for every project:
```
dist/
node_modules/
.cache/
coverage/
```
Plus any large third-party packages included in the repo that we don't modify.

---

## Standard DevOps pipeline — every project uses this

```
Developer (or AI) commits code
        ↓
git push to main  (or PR merged to main)
        ↓
GitHub Actions triggers automatically
        ↓
Builds Docker image from source
        ↓
Pushes to GitHub Container Registry (GHCR):
  ghcr.io/u2giants/[project-name]:latest
  ghcr.io/u2giants/[project-name]:sha-[commit-hash]
        ↓
GitHub Actions calls Coolify API to trigger redeploy
        ↓
Coolify pulls the new :latest image and restarts containers
        ↓
Live
```

**Non-negotiable principles:**

- **GitHub is the source of truth.** No changes are ever made to running containers
  directly. If something is broken in production, fix the code, push, deploy.
- **Coolify is a consumer of pre-built images, not a builder.** It pulls from GHCR.
  It never builds from source.
- **Every deployment is traceable.** The `:sha-[commit]` tag means any live container
  can be traced to the exact commit it came from.
- **Rollback** = update Coolify to pull the previous `:sha-[commit]` tag.

**Standard GitHub Actions workflow** (`.github/workflows/build-and-push.yml`):

```yaml
name: Build and Push

on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read
  packages: write

env:
  IMAGE: ghcr.io/u2giants/[PROJECT-NAME]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - name: Build and push
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: |
            ${{ env.IMAGE }}:latest
            ${{ env.IMAGE }}:sha-${{ github.sha }}
      - name: Trigger Coolify redeploy
        run: |
          curl -fsSL -X GET \
            "${{ secrets.COOLIFY_BASE_URL }}/api/v1/applications/${{ secrets.COOLIFY_APP_UUID }}/start" \
            -H "Authorization: Bearer ${{ secrets.COOLIFY_API_TOKEN }}"
```

**GitHub Secrets to set for every project:**
| Secret | Description |
|---|---|
| `COOLIFY_BASE_URL` | The Coolify panel URL (e.g. `https://coolify.example.com`) |
| `COOLIFY_API_TOKEN` | Coolify API token |
| `COOLIFY_APP_UUID` | UUID of the application in Coolify (get from Coolify UI or API) |

If a project has a separate worker process, add `COOLIFY_WORKER_UUID` and a second
deploy step.

---

## Codebase optimization — do this for existing projects before other work

When onboarding an AI to an existing codebase (not greenfield), do this before building
anything:

### Step 1: Count and categorize

```bash
# Total file count (excluding build artifacts)
find . -type f | grep -v node_modules | grep -v .git | wc -l

# Per directory breakdown
find . -maxdepth 2 -type d | while read d; do
  echo "$(find "$d" -type f 2>/dev/null | grep -v node_modules | wc -l) $d"
done | sort -rn | head -20
```

### Step 2: Identify deletion candidates

Ask of each large directory: "Is this part of what we actively build and deploy?"
Common safe deletions in third-party framework forks:
- Vendor documentation website packages
- Sample / example apps from the framework
- Test suites belonging to the framework, not our project
- CLI scaffolding tools for the framework we don't use
- Integrations with third-party services we don't use
- Desktop companion apps (if not building a desktop app)
- Old/deprecated build approaches that have been superseded

**Restoring deleted content from upstream is always trivial:**
```bash
git checkout upstream/main -- [path/to/deleted/thing]
```
Never hesitate to delete. It is always reversible.

### Step 3: Create `.claudeignore` and `.cursorignore`

For packages kept in git but not relevant to active development.

### Step 4: Build AGENTS.md

Do this AFTER the deletion, so the map reflects the leaner structure. The map should
describe what remains, not what was removed.

---

## Multi-session / multi-developer documentation discipline

Multiple developers, AI agents, and AI sessions will always be working on every project.
Documentation discipline is what prevents them from undoing each other's work.

**Rules that must be followed in every session:**

1. **Update AGENTS.md with every significant change.** New service added → update
   navigation map. New route added → update decision tree. Bug fixed with an unusual
   approach → add to Idiosyncratic Decisions.

2. **Document every non-obvious decision at the time it is made.** Not later. You will
   not remember. The next AI session will not remember.

3. **The new-developer test:** Before closing any session, ask: "If a brand new senior
   developer opened this repo right now with no prior context, would AGENTS.md give them
   enough information to work productively without asking questions?" If no, update it.

4. **Idiosyncratic decisions are the highest priority.** Anything that looks wrong but
   is intentional MUST be documented with full context before the session ends.

5. **Pending work stays current.** Complete items get marked done. New items get added.
   A stale pending section is actively harmful — it misleads future developers.

6. **After any upstream merge or dependency upgrade,** update the Core Modification
   Inventory to reflect any conflict resolutions.

---

## Code quality standards — every project

- **Custom code in custom directories.** Never scatter project-specific logic into
  third-party framework directories. Create a dedicated module (e.g.,
  `src/modules/[project-name]/`). This makes upstream merges clean.

- **Registry/plugin patterns over repeated core modifications.** When a framework needs
  to know about your custom components, routes, or handlers — create a registry file
  inside your custom directory. The framework's extension point imports that registry
  once. The framework file is then permanently frozen for your changes.

- **Identifiers are permanent.** Any UUID, database ID, or external-system identifier
  assigned to an entity can never be changed once it is used in any environment.
  Keep a centralized identifier map in AGENTS.md or a dedicated file.

- **AI API calls always use OpenRouter.** Single endpoint, single API key.
  Never add provider-specific keys (OpenAI, Anthropic, Google) as separate env vars.
  Model IDs: `openai/gpt-5.4`, `anthropic/claude-sonnet-4-6`, `google/gemini-3.1-pro-preview`.

- **Never modify a running container.** Code change → commit → push → deploy pipeline.
  No exceptions.

---

## Good ideas to apply from the POP Creations CRM project

These practices proved their value in that project and should be applied everywhere:

- **The Prime Directive works.** Defining a clear "our code lives here" boundary meant
  upstream merges were nearly conflict-free. Do this from day one, not after.

- **Registry patterns save repeated work.** The widget registry and route registry meant
  that adding the 10th widget or the 5th page required zero changes to framework files.
  Design for this from the first custom extension point.

- **File count matters for AI cost.** Start lean. Delete vendor extras before writing
  code. It's cheaper to write one new file than to have AI re-read 3,000 irrelevant ones.

- **AGENTS.md is a force multiplier.** A 400-line AGENTS.md saves thousands of tokens
  per session because Claude doesn't need to explore the codebase from scratch. Time
  spent writing documentation pays back immediately.

- **Critical incident log.** Every project should have a section documenting disasters
  and near-misses: what happened, what was destroyed, how it was recovered, and the rule
  that prevents recurrence. These are the most valuable things you can write.

- **Centralize all identifiers.** Any ID that has to match between your code and a
  database or external system goes in one file. Never duplicated inline. Prevents the
  entire class of "worked on my machine, broke in prod" bugs.

- **Document the "why" of every environment variable.** Not just the name — what it does,
  what breaks if it's missing, where to get it, and whether it's the same in dev vs. prod.

---

## Immediate action list when receiving this prompt

When you receive this prompt in any project context, do the following in order.
Do not skip steps, do not reorder them:

1. **Read the existing project structure** (list top-level directories, key files)
2. **Ask Albert for ALL credentials and access** you will need (comprehensive list)
3. **Count files and identify deletion candidates** using the commands above
4. **Report back with a deletion proposal** — list what you'd remove and why —
   and wait for a quick "go ahead" before deleting anything significant
5. **Delete approved items** and update `.gitignore`/`package.json` workspaces as needed
6. **Create `.claudeignore`** (and `.cursorignore`) for remaining irrelevant directories
7. **Create `AGENTS.md`** using the required sections above, tailored to this project
8. **Create `CLAUDE.md`** (short, Claude-specific, references AGENTS.md)
9. **Set up or verify the GitHub Actions → GHCR → Coolify pipeline**
10. **Commit everything** to `main` with a clear message
11. **Report back:** file count before/after, what was deleted, what was documented,
    what credentials you still need, what the first development task should be

---

## A note on this prompt itself

This is a living document. After any project, if a better practice is discovered or a
mistake in this standard is found, update this file. Every new project benefits from
everything learned in every previous one.

**File location:** `/home/ai/Albert-AI-Standards/NEW-PROJECT-PROMPT.md`
