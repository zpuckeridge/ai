---
name: rc
description: >-
  Bugbot, security review, fix every finding via subagents, then bun run
  check until React Doctor is 100% and Ultracite is clean. Use for /rc or
  pre-merge cleanup. Replaces /review-and-check.
disable-model-invocation: true
---

# rc (review + check)

Run on the **active workspace repo root**. Do not compute the diff yourself before launching review subagents.

**Diff default:** `branch changes` (merge-base with the repo default branch — committed, staged, and unstaged). Use `uncommitted changes` only when the user asks to review dirty/local work only.

**Completion bar:** The workflow is **not done** until `bun run check` exits 0 **and** its output shows **React Doctor at 100%** and **Ultracite with no errors and no warnings**. Partial pass is failure.

Do not commit or open a PR unless the user asked.

## 1. Bugbot

Launch **exactly one** `bugbot` subagent (`subagent_type: "bugbot"`, `description: "Bugbot"`, `run_in_background: false` unless the user asked otherwise).

Prompt:

```
Full Repository Path: <absolute workspace path>
Diff: branch changes | uncommitted changes
Base Branch: <optional>
Change Description: <optional>
Custom Instructions: <optional>
```

Default `Diff` to **`branch changes`**. Do not compute or paste the diff yourself; the subagent resolves it.

After it returns, **fix every finding** using [fix subagents](#3-fix-subagents). Then summarize with a compact markdown table (Severity, Location `file:line`, Finding), highest severity first, and note which rows you fixed.

If the subagent fails, retry **once**. If it still fails, tell the user and continue to security review.

## 2. Security review

Launch **exactly one** `security-review` subagent (`subagent_type: "security-review"`, `description: "Security Review"`, `run_in_background: false` unless the user asked otherwise).

Prompt:

```
Full Repository Path: <absolute workspace path>
Diff: branch changes | uncommitted changes
Custom Instructions: <optional>
```

After it returns, **fix every finding** via [fix subagents](#3-fix-subagents). Summarize the same way. Continue even if Bugbot already edited files; review and fix against the **current tree**.

If the subagent fails, retry **once**. If it still fails, note that security review did not complete, then still run the check loop.

## 3. Fix subagents

After Bugbot or Security Review lists issues, **do not only fix them in the parent agent**. Spin up subagents to implement fixes.

### When to use subagents

- **Always** when there is more than one distinct finding, or any finding touches more than one file.
- **Always** when findings span different areas (e.g. auth + UI + API).
- For a **single trivial one-line fix** in one file, the parent may fix inline; still re-verify with the same gates as step 4.

### How to launch

Use the `Task` tool with `subagent_type: "generalPurpose"` (use `explore` only for investigation that must precede a fix).

- **Batching:** Group findings by file or subsystem. One subagent per batch is fine; run **in parallel** when batches do not touch the same files.
- **Prompt shape** (include all of this):

  ```
  Full Repository Path: <absolute path>
  Task: Fix the following review findings. Apply minimal, correct diffs. Match repo conventions. Do not commit unless explicitly asked.

  Findings:
  - [Severity] path:line — description
  ...

  When done: list what you changed (file + one line each). If a finding is a false positive, say why and what you verified instead of changing code.
  ```

- Wait for each fix subagent to finish (`run_in_background: false` unless the user asked otherwise).
- **Parent verifies:** skim diffs, then proceed. If a finding remains unfixed, launch another fix subagent or fix it in the parent — do not advance to "pass" with open review items.

## 4. `bun run check` (required pass)

From the repo root:

```bash
bun run check
```

If `check` is missing from `package.json`, say so and stop this step. If `bun` is missing, say so and do not invent a substitute unless the user names one.

### Pass criteria (all required)

1. Process **exit code 0**.
2. **React Doctor:** score **100%** (no failing checks).
3. **Ultracite:** **zero errors and zero warnings** in the check output.

Warnings count as failure even if the process exits 0. Do not treat "mostly green" as done.

### Fix loop

1. Run `bun run check` and capture **full stdout/stderr**.
2. If any pass criterion fails, launch one or more **fix subagents** with the complete log and explicit instruction to satisfy React Doctor 100% and Ultracite clean.
3. Re-run `bun run check`.
4. **Repeat until all pass criteria are met**.

If the **same** failure persists unchanged after **three** full fix-and-recheck cycles, stop looping: report the remaining errors verbatim and state that the workflow did **not** pass. Do not claim success.

## 5. Close

One short recap:

- **Bugbot:** count fixed / remaining (if any could not be fixed).
- **Security:** count fixed / remaining.
- **`bun run check`:** explicitly state React Doctor %, Ultracite errors/warnings, and exit code — **pass** only if all gates in step 4 passed.

If review findings were fixed but check introduced new issues, that is expected — the check loop must still end green before you close.
