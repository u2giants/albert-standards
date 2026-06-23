# AGENTS.md — guidance for AI sessions working in this repo

`albert-standards` holds cross-cutting standards + infra docs. **Start at
[`infrastructure/HANDOFF.md`](infrastructure/HANDOFF.md)** for current status and
[`infrastructure/DECISIONS.md`](infrastructure/DECISIONS.md) for the *why* behind decisions.

### Pushing to GitHub from the server — read this BEFORE you push (do not repeat these mistakes)

What to know (these bit a prior session):
- The local clones on the server are **stale and diverged** from GitHub, and the repos enforce a
  **main-only push hook** (`.githooks/post-checkout` blocks creating any non-`main` branch, so
  `git rebase` fails with *"Branch creation is not allowed"*).
- GitHub **rejects commits authored with the owner's private email** (`u2giants@gmail.com` → error
  GH007). Commit with the noreply address **`55610577+u2giants@users.noreply.github.com`**.

Future sessions should:
- **Don't trust the server's local clone.** Make a **fresh `git clone` of the latest `main`**, apply
  the change there, commit with the noreply email, push linearly. **Don't `rebase`** (the hook blocks
  it). To reconcile diverged local commits, `git reset --hard origin/main` then `git cherry-pick`
  onto `main` (cherry-pick doesn't trip the branch hook).
- **Sweep for secrets before every push** and gate the commit on a clean result:
  `grep -rnE 'ghp_[A-Za-z0-9]|AKIA[A-Z0-9]{16}|DO[0-9A-Z]{18}|-----BEGIN'` — a real DO Spaces
  access-key id was once committed via a stale `.env.template`.
- Agent-initiated **repo creation / bulk push of infra content is blocked** by the permission
  classifier. Commit into existing repos (`albert-standards`, `backrest-wiz`); don't create new ones.

### Where things live
- **Backup / backrest system** → `backrest-wiz` repo (`hetzner-producer/`), incl. its `HANDOFF.md`.
- **Host Ansible + CI build plan** → [`ansible/ANSIBLE-IMPLEMENTATION-PLAN.md`](ansible/ANSIBLE-IMPLEMENTATION-PLAN.md).
- **Live server reference + "do not touch" rules** → [`infrastructure/CLAUDE.md`](infrastructure/CLAUDE.md).
