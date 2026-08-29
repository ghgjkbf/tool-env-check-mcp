---
name: "tool-location-confirm"
description: "Ask-before-search tool locator. Invoke BEFORE any disk-wide search when a task needs an external program/tool: quick PATH check, then ask the user if installed and where; if missing offer abort/download/alternative. Triggers: find a tool, locate a program, where is it installed, missing-dependency check. Implicit scenarios: publishing/committing/storing to a repository, opening a file with a specific program, exporting/converting formats — any task that depends on a local program or repository must also trigger this skill."
---

# Tool Location Confirm: Ask Before You Search

> **Language:** [简体中文](SKILL.md) · English

When a task requires an external program or tool, this skill enforces "ask before search": run a second-level quick check first, then ask the user to confirm. A disk-wide scan must never be the first step.

## Hard Rules

1. When a task needs external program X, **a full-disk recursive search is forbidden** as the first step
2. The first step is only a second-level quick check (see toolbox below), usually done within seconds
3. Quick check misses → you MUST ask the user first using the question templates; do not scan the disk on your own
4. Valid paths confirmed by the user are cached in session state; do not ask again within the same task

## Workflow Overview

```
Task needs program X
  ├─ ① Second-level quick check (PATH / Get-Command / registry spot check)
  │     └─ ✓ Found → verify, call it, continue the task
  ├─ ② Not found → ask the user: "Is it installed? Where?"
  │     └─ ✓ User gives path P → verify it exists and is executable → use it
  └─ ③ User answers "not installed" → present options A/B/C
        └─ Alternative still unavailable → back to A/B/C (max 2 loops)
```

## Question Templates

"The task needs **{X}**. Is it already installed? What is the installation path?"

"This task involves **{repository/project/file}**. Which directory is it in?"

## Implicit Trigger Scenarios (lesson logged 2026-08-27)

Tool-location needs often **do not appear directly in the task description** — they hide behind the task verb. The following scenarios must trigger this skill even without words like "find a tool":

| Task example | Implicit thing to locate |
|---------|-------------|
| "Publish/commit/store to a GitHub repository" | Local repository location + git or GitHub Desktop |
| "Open/process this file with XX software" | Where that software is installed |
| "Export as XX format" / "compress/transcode" | The handler program (ffmpeg, magick, etc.) |
| "Package/build/release" | The build toolchain |

Real lesson: for the task "help me store this in a GitHub repository", this skill was not triggered, the agent blindly probed the wrong directories, and the repository's actual location only surfaced after the user said so. Rule: **as soon as a task depends on a local program or repository, quick-check first, then ask for the location — never assume, never guess.**

## Handling the Answers (after asking)

- **Path P given** → verify the path exists and is executable; on failure, report the specific reason and ask once more (a path typo shouldn't kill the task)
- **Not sure / forgot** → escalate the quick check: registry uninstall list + `Get-StartApps` + spot checks of common install directories; still nothing → fall through to the three options
- **Not installed** → present options A/B/C on the spot

## The Three Options (required when the user answers "not installed")

- **A · Abort the task**: output what was completed and why it stopped; wrap up cleanly, leave no half-finished work
- **B · Help me download and install**: follow the safety boundaries; agree on the source and version with the user before executing
- **C · Use an alternative**: ask the user for alternative Y, then re-run steps ① and ② for Y

## Loop Limits

- The alternative is still unavailable → back to the three options, max **2 rounds**
- Beyond the limit → list the blocking reasons explicitly and wait for the user's final decision; no further self-directed attempts

## Windows Quick-Check Toolbox

| Target type | Means |
|---------|------|
| CLI tools | `where.exe X`, `Get-Command X` |
| GUI programs | Registry Uninstall keys, `Get-StartApps` |
| Installed list | `winget list` (when available) |

## Memory Hooks

- **In-session**: cache confirmed valid paths immediately; do not ask again within this task
- **Cross-session**: write to `tool_locations.md` per your memory-store conventions and sync the MEMORY.md index
- **On reuse**: for the same program next time, skip the question and keep only a one-shot second-level verification; re-ask if the directory is gone

## Safety Boundaries (override all other rules)

- Download priority: winget > choco > scoop > official direct link; verify hashes/signatures
- Operations requiring admin rights must be announced to the user first
- No silent installs; no PATH changes without consent
- Expanding the search scope is allowed only when the user explicitly says "go find it" (whitelisted directories only)

## Batch Asking

When a task depends on 2 or more missing tools, merge them into **one** question listing the full inventory with per-item status — do not interrupt the user repeatedly.

## Boundaries with Other Skills

- Searching the web for ready-made solutions/libraries → use `search-first`
- This skill only handles **local program location** and user confirmation
- When in conflict with efficiency rules such as `ponytail`, safety rules outrank efficiency rules

## Acceptance Tests

1. Task requires ffmpeg but it is not installed → should trigger the question, not a disk-wide scan
2. User answers "I don't know" → should escalate the quick check instead of jumping straight to the three options
3. Same tool needed again in a later session → should use the stored path directly (with the second-level verification)
