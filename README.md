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
   ```

4. **Symlink example** (from this repo’s root, project-scoped):

   ```bash
   mkdir -p .cursor/skills
   ln -sf "$(pwd)/skills/rc" .cursor/skills/rc
   ```

### Verify

In Cursor Agent chat, type `/` and look for **rc**. You can also open a `SKILL.md` and confirm the YAML frontmatter `name` and `description` fields are present.

---

## Quick reference

| Skill | One-line purpose | Typical triggers |
| --- | --- | --- |
| [**rc**](#rc) | Parallel Bugbot + security review + `bun run check`, fix with up to 6 subagents, loop until clean, then commit and push | **`/rc`** (replaces `/review-and-check`) |

---

## rc

**File:** [`skills/rc/SKILL.md`](skills/rc/SKILL.md)

**What it does**

1. **Three parallel subagents:** Bugbot, security review and `bun run check`.
2. **One issue list:** every finding deduplicated, with severity, source and `file:line`.
3. **Fix subagents:** batched by file, **at most 6 at a time**, further waves as needed.
4. **Loop:** re-run all three checks until the gate passes.
5. **Commit and push:** grouped semantic commits, then push the current branch.

**When to use it**

- Pre-merge quality loop. Type **`/rc`** in Agent chat.

**How to invoke**

- **`/rc`** only. `/review-and-check` is retired (stub still forwards to this skill).

**Requirements / notes**

- Target repo must have `bun run check` (React Doctor + Ultracite).
- Hard gate: exit 0, React Doctor **100%**, Ultracite zero errors **and** zero warnings, no open review findings. Nothing is committed or pushed until it passes.
- If the same issue survives three fix cycles, `/rc` stops and reports failure without committing.

---

## Skill vs Cursor rule

| | **Skill** | **Rule** |
| --- | --- | --- |
| **Loads when** | Your message matches the skill’s topic | Always, or when glob/file patterns match (if configured) |
| **Best for** | Long or rare workflows (e.g. the `/rc` review and check loop) | Always-on standards, short reminders (“before merging, run `/rc`”) |
| **Tradeoff** | Stays out of the way until relevant | Always-on rules can add noise if they embed huge procedures |

A one-line rule pointing at “before merging → run `/rc`” is fine; pasting the entire `/rc` checklist as an always-apply rule is usually too heavy—use the skill instead.

---

## Repo layout

```text
skills/
  rc/SKILL.md
  review-and-check/SKILL.md   # retired stub → rc
commands/
  rc.md
  review-and-check.md       # retired stub → rc
README.md
```

There is no `package.json` here—this repo is **only** skill definitions. Dependent projects keep their own tooling.
