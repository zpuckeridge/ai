---
name: rc
description: >-
  Bugbot, security review, fix subagents, then bun run check until React Doctor
  100% and Ultracite clean. Pre-merge quality loop. Replaces /review-and-check.
disable-model-invocation: true
---

# rc (review + check)

Read and follow the **rc** skill if present (`~/.cursor/skills/rc/SKILL.md` or `.cursor/skills/rc/SKILL.md`). If the skill is missing, follow this file end to end.

Run on the **active workspace repo root**. Do not compute the diff yourself before launching review subagents.

**Diff default:** `branch changes`. Use `uncommitted changes` only when the user asks for dirty/local review only.

**Not done** until `bun run check` exits 0 **and** React Doctor is **100%** **and** Ultracite has **zero errors and zero warnings**. Partial pass is failure.

Do not commit or open a PR unless the user asked.

## 1. Bugbot

Launch **exactly one** `bugbot` subagent (`subagent_type: "bugbot"`, `description: "Bugbot"`, `run_in_background: false` unless asked otherwise).

Prompt:

```
Full Repository Path: <absolute workspace path>
Diff: branch changes | uncommitted changes
Base Branch: <optional>
Change Description: <optional>
Custom Instructions: <optional>
```

Fix every finding via §3. Table: Severity, Location `file:line`, Finding. Retry Bugbot once on failure, then continue.

## 2. Security review

Launch **exactly one** `security-review` subagent (`subagent_type: "security-review"`, `description: "Security Review"`, `run_in_background: false` unless asked otherwise).

Prompt:

```
Full Repository Path: <absolute workspace path>
Diff: branch changes | uncommitted changes
Custom Instructions: <optional>
```

Fix every finding via §3 against the **current tree**. Retry once on failure, then still run the check loop.

## 3. Fix subagents

Do not only fix in the parent. Use `Task` + `subagent_type: "generalPurpose"`. Parallelise batches that do not share files. Parent may inline a single one-line one-file fix.

Prompt:

```
Full Repository Path: <absolute path>
Task: Fix the following review findings. Apply minimal, correct diffs. Match repo conventions. Do not commit unless explicitly asked.

Findings:
- [Severity] path:line — description

When done: list what you changed. If false positive, say why.
```

Do not mark pass with open review items.

## 4. bun run check

```bash
bun run check
```

Pass only if: exit 0, React Doctor 100%, Ultracite zero errors **and** zero warnings. Warnings = fail.

Loop: capture full log → fix subagents → re-run until green. Same failure after three cycles → stop, report verbatim, do not claim success.

If `check` or `bun` is missing, say so and stop this step.

## 5. Close

Bugbot fixed/remaining. Security fixed/remaining. Check: React Doctor %, Ultracite errors/warnings, exit code. Pass only if §4 passed.

Use `$ARGUMENTS` for extra instructions (diff scope, skip areas).
