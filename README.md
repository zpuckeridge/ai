# Agent skills

This repository holds **Cursor agent skills**: markdown instructions the agent loads when your message matches the skill’s purpose. Skills stay **off the default context** until they’re relevant, so long workflows do not clutter every chat.

---

## Install

### CLI (recommended)

Install skills with the **[skills](https://github.com/vercel-labs/skills)** CLI (`skills` on npm—the open agent skills ecosystem). It adds skills from a GitHub repo into Cursor (and other agents, depending on the tool).

```bash
bunx skills add zpuckeridge/ai
```

That pulls this repository and registers its skill folders under `~/.cursor/skills/`. After install, type **`/rc`** in Agent chat — it should appear in the slash menu.

Use `bunx skills --help` for other commands (e.g. list, remove). You can also use `npx skills` if you are not using Bun.

**Already installed?** Re-run the same command to pick up new skills (`rc`). Then reload Cursor (or start a new Agent chat) so `/rc` shows in the picker.

**Retire `/review-and-check`:** that old slash command lived as a Cursor **Command**, not in this repo. After installing `rc`:

1. Cursor → **Customize → Commands** (or `~/.cursor/commands/review-and-check.md`)
2. Delete **`review-and-check`**
3. Keep **`/rc`** only

A stub skill named `review-and-check` remains so an old `/review-and-check` still runs the same loop as `/rc` until you delete the command.

### Manual install

If you prefer not to use the CLI:

1. **Clone or copy** this repo, or add it as a submodule where the team should pin versions.

2. **Point Cursor at each skill** by copying or symlinking so each skill is `<skills-root>/<name>/SKILL.md`.

   | Location | Scope |
   | --- | --- |
   | `~/.cursor/skills/<name>/` | All projects on your machine (personal) |
   | `.cursor/skills/<name>/` | Single project only (commit if the team should share it) |

3. **Folder layout** (skill id = folder name, file must be `SKILL.md`):

   ```text
   .cursor/skills/
     rc/
       SKILL.md
     ship/
       SKILL.md
     review/
       SKILL.md
     ralph/
       SKILL.md
   ```

4. **Symlink example** (from this repo’s root, project-scoped):

   ```bash
   mkdir -p .cursor/skills
   ln -sf "$(pwd)/skills/rc" .cursor/skills/rc
   ln -sf "$(pwd)/skills/ship" .cursor/skills/ship
   ln -sf "$(pwd)/skills/review" .cursor/skills/review
   ln -sf "$(pwd)/skills/ralph" .cursor/skills/ralph
   ```

### Verify

In Cursor Agent chat, type `/` and look for **rc**. You can also open a `SKILL.md` and confirm the YAML frontmatter `name` and `description` fields are present.

---

## Quick reference

| Skill | One-line purpose | Typical triggers |
| --- | --- | --- |
| [**rc**](#rc) | Bugbot + security review + fix subagents + `bun run check` (React Doctor 100%, Ultracite clean) | **`/rc`** (replaces `/review-and-check`) |
| [**ship**](#ship) | Review changes, run checks, semantic commit, push | “ship”, “verify” |
| [**review**](#review) | Senior-style review of a diff (severity-ordered findings) | Paste diff, “review this PR”, “pre-merge review” |
| [**ralph**](#ralph) | Loop: GitHub `agent` issues → implement → validate → PR → repeat | “run RALPH”, “RALPH for 3 iterations”, “continue RALPH” |

---

## rc

**File:** [`skills/rc/SKILL.md`](skills/rc/SKILL.md)

**What it does**

1. **Bugbot** — one `bugbot` subagent on branch changes (by default).
2. **Security review** — one `security-review` subagent on the current tree.
3. **Fix subagents** — `generalPurpose` tasks to resolve every listed finding (batched/parallel when safe).
4. **`bun run check`** — repeat until React Doctor is **100%** and Ultracite reports **no errors and no warnings**.

**When to use it**

- Pre-merge quality loop. Type **`/rc`** in Agent chat.

**How to invoke**

- **`/rc`** only. `/review-and-check` is retired (stub still forwards to this skill).

**Requirements / notes**

- Target repo must have `bun run check` (React Doctor + Ultracite).
- Not done until all three pass: exit 0, React Doctor 100%, Ultracite zero errors **and** zero warnings.
- Do not commit or open a PR unless asked.

---

## ship

**File:** [`skills/ship/SKILL.md`](skills/ship/SKILL.md)

**What it does**

- **Step 0 — Code review:** Inspect working changes (`git status` / diff). Fix safe issues inline; stop and ask on critical product/security/architecture calls.
- **Checks:** Runs whatever exists in the repo—format, lint, typecheck, optional Convex dry-run (when `CONVEX_DEPLOYMENT` is set), build, tests, `bun audit` when a lockfile exists. Supports **Turbo monorepos** (scoped by changed paths under `apps/`) and **single-package** repos.
- **ship:** After checks pass → semantic commit → `git push`.
- **verify:** Same as ship but **stops before** commit/push.

**When to use it**

- You want a disciplined “review + quality gates + release” path without a separate script hiding failures.

**How to invoke**

- Say **ship** or **verify** explicitly. Casual “just push” or “deploy” is **not** this workflow unless you clearly mean the same pipeline.

**Requirements / notes**

- Uses **bun** when the repo uses Bun; otherwise the repo’s package manager.
- Convex dry-run: only when Convex is present **and** `CONVEX_DEPLOYMENT` is set; otherwise the skill expects a **loud warning**, not a silent skip.
- Merge into `main` only when you asked to merge from a branch; if unclear, the agent should ask.
- This is **instructions for the agent**, not a `ship` shell script—steps run in chat so you can see and fix failures.

---

## review

**File:** [`skills/review/SKILL.md`](skills/review/SKILL.md)

**What it does**

- Summarises the change (intent + scope, 2–4 bullets).
- Reviews for correctness, regressions, security (auth, exposure, injection, secrets), concurrency, error handling, and test gaps.
- Lists findings **Critical → High → Medium → Low** with a fixed rubric (see the skill file).
- Does **not** rewrite code unless you ask for patches or edits.

**When to use it**

- PR review, second opinion, risk check before merge, or auditing a pasted patch.

**How to invoke**

- Paste a diff, name a branch/range, or ask for a review of specific files/commits.

**Requirements / notes**

- Large diffs: agent prioritises risky hunks and states what was not fully covered.

---

## ralph

**File:** [`skills/ralph/SKILL.md`](skills/ralph/SKILL.md)

**What it does**

- **Loop:** List open GitHub issues with label **`agent`** → assign to yourself → implement → validate → open PR on branch `ralph/issue-<n>` → comment on issue → repeat until iteration limit or no more work.
- **Validation** in the skill references `bun run ship --check`—**adapt** that line if your repo uses a different check script.
- When idle, can merge open RALPH PRs (`ralph/issue-*`) so work is not blocked; see the skill for full rules.
- **Does not** stop to ask “continue?”—keeps going until a termination condition is met.

**When to use it**

- You want batched, issue-driven automation with PRs and GitHub as the task queue.

**How to invoke**

- Phrases like: “run RALPH”, “RALPH loop”, “RALPH for N iterations”, “RALPH until done”, “continue RALPH”, “next task”, “automate GitHub issues”.

**Requirements / notes**

- **[GitHub CLI](https://cli.github.com/)** (`gh`) installed and authenticated (`gh auth status`).
- Run from the **repository root** so `gh` scopes issues to the current repo.
- Issues must use the **`agent`** label (and optionally your own conventions for assignees).
- Your repo **does not** need to match every command in the skill verbatim—treat it as a template and edit `SKILL.md` for your stack (e.g. replace `bun run ship --check` with `npm run test && npm run lint`).

**Suggested habit:** Run your real CI/checks before relying on automated PRs; fixing failures early beats “fix CI” follow-up commits.

---

## Skill vs Cursor rule

| | **Skill** | **Rule** |
| --- | --- | --- |
| **Loads when** | Your message matches the skill’s topic | Always, or when glob/file patterns match (if configured) |
| **Best for** | Long or rare workflows (ship, RALPH, heavy review checklists) | Always-on standards, short reminders (“when user says ship, use the Ship skill”) |
| **Tradeoff** | Stays out of the way until relevant | Always-on rules can add noise if they embed huge procedures |

A one-line rule pointing at “say ship → follow Ship skill” is fine; pasting the entire ship checklist as an always-apply rule is usually too heavy—use the skill instead.

---

## Repo layout

```text
skills/
  rc/SKILL.md
  review-and-check/SKILL.md   # retired stub → rc
  ship/SKILL.md
  review/SKILL.md
  ralph/SKILL.md
README.md
```

There is no `package.json` here—this repo is **only** skill definitions. Dependent projects keep their own tooling.
