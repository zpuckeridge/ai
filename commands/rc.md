---
name: rc
description: >-
  Bugbot, security review and bun run check as three parallel subagents, fix
  every issue with up to 6 subagents at a time, loop until React Doctor 100% and
  Ultracite clean, then commit and push. Replaces /review-and-check.
disable-model-invocation: true
---

# rc (review + check)

Read and follow the **rc** skill if present (`~/.cursor/skills/rc/SKILL.md` or `.cursor/skills/rc/SKILL.md`). If the skill is missing, follow this file end to end.

Use `$ARGUMENTS` for extra instructions (diff scope, skip areas).

Run on the **active workspace repo root**. Do not compute the diff yourself before launching review subagents.

**Diff default:** `branch changes` (merge-base with the repo default branch: committed, staged and unstaged). Use `uncommitted changes` only when the user asks to review dirty/local work only.

**Hard gate:** never finish, commit or push until **all** of these hold on the current tree:

1. `bun run check` exits **0**.
2. **React Doctor** score is **100%**.
3. **Ultracite** reports **zero errors and zero warnings**. Warnings are failures.
4. No open Bugbot or security finding remains (fixed, or shown to be a false positive with evidence).

Anything less is failure. Do not accept "mostly green".

## 1. Run all three checks in parallel

Launch **three separate subagents in the same turn** so they run in parallel (`run_in_background: false` unless the user asked otherwise):

| # | Subagent | `subagent_type` | `description` |
| --- | --- | --- | --- |
| A | Bugbot | `bugbot` | `Bugbot` |
| B | Security review | `security-review` | `Security Review` |
| C | Check | `generalPurpose` | `bun run check` |

**A. Bugbot prompt:**

```
Full Repository Path: <absolute workspace path>
Diff: branch changes | uncommitted changes
Base Branch: <optional>
Change Description: <optional>
Custom Instructions: <optional>
```

**B. Security review prompt:**

```
Full Repository Path: <absolute workspace path>
Diff: branch changes | uncommitted changes
Custom Instructions: <optional>
```

**C. Check prompt:**

```
Full Repository Path: <absolute workspace path>
Task: From the repo root run `bun run check` and capture full stdout/stderr. Do not edit any files.
Report: exit code, React Doctor score (%), Ultracite error count and warning count, and every individual error/warning/failing React Doctor check as `path:line — rule — message`. Include the raw log tail if parsing is uncertain.
```

If `bun` or a `check` script is missing, subagent C reports it and the workflow stops at the gate (do not invent a substitute unless the user names one). If Bugbot or security review fails, retry it **once**; if it still fails, say so and do not claim a clean pass.

## 2. List all issues

Merge the three results into **one deduplicated list** before fixing anything. Show it as a table, highest severity first:

| # | Severity | Source | Location `file:line` | Issue |
| --- | --- | --- | --- | --- |

- **Source:** `Bugbot`, `Security`, `React Doctor` or `Ultracite` (combine sources when the same issue is reported twice).
- Every Ultracite warning and every failing React Doctor check is its own row.

## 3. Fix with subagents (max 6 at a time)

Do not fix in the parent, except a single trivial one-line fix in one file.

- Group issues into batches by file or subsystem. **Two batches running together must never touch the same file.**
- Launch fix subagents with `subagent_type: "generalPurpose"`, **at most 6 in parallel**. If there are more batches, wait for that wave to finish, then launch the next wave of up to 6.
- Prompt shape:

  ```
  Full Repository Path: <absolute path>
  Task: Fix the following issues. Apply minimal, correct diffs. Match repo conventions. Do not commit or push.
  Gate: the result must keep React Doctor at 100% and Ultracite at zero errors and zero warnings.

  Issues:
  - [#] [Severity] [Source] path:line — description

  When done: list what you changed (file + one line each). If an issue is a false positive, say why and what you verified instead of changing code.
  ```

- Parent skims the diffs and marks each row fixed or false positive.

## 4. Re-check and loop

Re-run step 1 (all three subagents in parallel) against the current tree, then steps 2 and 3 for anything new or remaining. Repeat until the hard gate passes.

If the **same** issue survives **three** fix-and-recheck cycles unchanged, stop: report it verbatim and state that `/rc` did **not** pass. Do **not** commit or push in that case.

## 5. Commit and push

Only after the hard gate passes:

1. Review `git status` and `git diff`.
2. Commit in **sensible grouped semantic commits** (e.g. `fix(auth): …`, `fix(ui): …`, `chore(lint): …`), grouping by concern rather than one commit per file or one giant commit. Never commit secrets or unrelated files.
3. Push the current branch (`git push`, or `git push -u origin HEAD` if it has no upstream). Never force-push, and never push straight to the default branch unless the user is already working on it and asked for that.

## 6. Close

One short recap:

- **Issues:** total found, fixed, false positives (by source).
- **Gate:** React Doctor %, Ultracite errors/warnings, `bun run check` exit code. **Pass** only if all hold.
- **Commits:** list of commit subjects and the pushed branch.
