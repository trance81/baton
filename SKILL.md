---
name: baton-init
description: Installs the Baton cross-session task-state convention into a project — a `.baton/<slug>.md` file per task or feature (kept forever, never deleted, even once finished) that together form a readable index of the project's tasks, a pointer line in CLAUDE.md/AGENTS.md, a git-ignored `local/` lane for machine-local values, and an end-of-turn hook — ready-made snippets for Claude Code, Codex, and Gemini CLI — that writes a zero-token JSON breadcrumb (last turn time, transcript path, last response) after every turn for recovering context when a session ends abnormally. Use this whenever the user wants to "add baton to this project", "carry task state across sessions", "remember what I was doing when the session dies", finds the `resume` session list unusable once it piles up, or is setting up a new repo and asks for session-continuity/handoff tooling for an AI coding agent. Idempotent — safe to re-run on a project that already has some of the pieces.
---

# Baton init

Installs the Baton convention (`.baton/` folder + end-of-turn hook) into a target project so agent sessions can hand off in-progress, multi-session work to the next session without the user re-explaining context.

**The `.baton/` file convention is tool-agnostic** — it's plain markdown, and any agent capable of reading and writing files can be told (via its own entry file: `CLAUDE.md`, `AGENTS.md`, …) to follow the same rules. The hook script in `templates/` is tool-neutral as well: it reads an end-of-turn JSON payload on stdin and already accepts the field names several CLIs use.

What is *not* tool-neutral is the hook **registration**, because every tool has its own config file and event names. This skill ships ready-made snippets for three — Claude Code, Codex, and Gemini CLI (see step 5). For anything else, don't guess at the format: install the script, then tell the user which of that tool's lifecycle events to point at it, or skip the hook entirely. The sidecar is a convenience; the markdown files are the whole point and work without it.

## Why this exists

A long task spans several sessions. Between sessions the model has no memory, and a CLI's `resume` picker gets harder to use as sessions pile up — titles and timestamps don't say what actually happened. `.baton/<slug>.md` is a small markdown file the model itself keeps current (current state, next step, blockers) — read at the start of a session, updated at meaningful boundaries, and *kept, not deleted, once the task finishes*. The accumulated files become a readable index of the project's tasks: what got built, how a piece of infrastructure was set up, what a bug fix actually did — searchable by content instead of by hunting through an opaque `resume` list.

The hook adds a cheap breadcrumb. Because it's a shell command rather than a model call, it can't write prose, but it *can* record "the session was last seen here" for free — the time, the transcript path, and the model's last response. That's a recovery cue when a baton looks doubtful or a session ended abnormally, not proof that the baton is current. It's explicitly best-effort: a user interrupt (Ctrl+C/ESC), a forced kill, or a power cut means the hook never runs at all. The durable source of truth is the markdown file either way.

## Steps

Work in the target project's root (confirm with the user if ambiguous — don't assume the current working directory is the target if the conversation didn't establish that).

### 1. Check what's already there

Before writing anything, check:
- Does `.baton/` already exist? If so, treat this as a repair/upgrade, not a fresh install — don't overwrite existing task files, only fill in missing pieces (README, hook, pointer line).
- Which entry file does this project use? `CLAUDE.md` and `AGENTS.md` are the common ones — use whichever already exists (if both do, use both, or ask). If neither exists, ask the user which one they want created, based on the agent they actually use.
- Does the entry file already have a Baton pointer line? Skip step 3 if so.
- Does the tool's hook config already have end-of-turn hooks (for Claude Code: `Stop`/`StopFailure` in `.claude/settings.json`)? If it has *different* ones, merge — append to the existing arrays, don't replace them. If Baton's hook is already registered for both events, skip step 5.
- Is there a project changelog file (`CHANGELOG.md`, `pjt-docs/CHANGELOG.md`, etc.)? Note its path for step 2 — the README can optionally mention it as a place to also leave a one-line entry alongside a finished task file (finished files are never deleted, so this is a bonus cross-reference, not a replacement).

### 2. Create `.baton/README.md`

Two source files exist: `templates/baton-readme.md` (Korean, default) and `templates/baton-readme.en.md` (English). Default to the Korean one — pick the English one only if the target project's dominant working language is English (its `CLAUDE.md`/other docs are written in English). Read the chosen file and write it to `.baton/README.md`, replacing the single `{{CHANGELOG_LINE}}` placeholder:
- If a changelog convention exists (found in step 1), replace it with a short clause pointing at it (e.g. "체인지로그가 있으면 `pjt-docs/CHANGELOG.md`에 한 줄 남겨도 된다").
- If none exists, replace it with an empty string — don't invent a changelog convention the project doesn't have.

Finished tasks are kept, never deleted (`status: passed`) — this isn't a per-session scratch file, it's meant to be the readable index of the *project's tasks*, standing in for a `resume` list that gets unusable once sessions pile up. Don't delete a `passed` file for any reason, including after its content gets copied into a knowledge base (`pjt-docs/`, `CLAUDE.md`) — that copy is additive, not a replacement, per the template's "이 폴더가 커지면" section. If the folder gets large enough that skimming it at session start is slow, the answer is archiving `passed` files into a subfolder (e.g. `.baton/done/`), never deleting them.

The template's frontmatter example includes `assignee` alongside `branch` — both are git-only fields, cut them from the written README if the project has no git. Keep `assignee` only if more than one person is expected to write baton files in this project (ask the user if unclear — a solo repo doesn't need it).

Leave the template's "무엇을 적고 무엇을 안 적나" / "What to write, and what not to" section intact, including the `local/` subsection. Baton files are committed, so the rule that credential *values* never go in them — only where a secret lives and what it's called — is load-bearing, not boilerplate. Trimming it to save space is the one edit that turns this convention into a liability.

### 3. Add the pointer line to the entry file

Append (don't prepend or replace existing content) a line to `CLAUDE.md` (or `AGENTS.md`) matching this pattern, adapted to whatever heading/list style the file already uses:

```
- **IMPORTANT**: 세션 시작 시 `.baton/` 최상위의 `running`·`waiting` 배턴만 읽는다. 진행 중
  작업이 있으면 그 맥락에서 이어간다. `done/`과 지난 배턴은 지금 요청과 관련 있을 때만 찾아본다
  — 전부 읽으면 컨텍스트만 낭비한다. 운영 규칙은 그 폴더의 README.md.
```

Write it in Korean by default, matching whichever `baton-readme` variant was picked in step 2. Switch to English only if the entry file's existing content is clearly written in English — the *rule* matters, not this exact wording.

Keep the scope in this line rather than deferring it to `.baton/README.md`. The entry file is what loads on every session; the folder README is read one hop later, if at all. A pointer that just says "read `.baton/`" is an instruction to read the whole folder, and it's the one that's always in context — so the narrowing has to travel with it. Stating the cost ("전부 읽으면 컨텍스트만 낭비한다") is part of the fix: a rule with a reason attached survives better than a bare directive.

### 4. Install the hook script

Copy `templates/baton-stop-hook.mjs` to `<project root>/scripts/baton-stop-hook.mjs` (create `scripts/` if it doesn't exist; if the project already keeps hook scripts elsewhere, e.g. `.claude/hooks/`, use that location instead and adjust the command paths in step 5 to match). The script is copied as-is — it has no project-specific values baked in, it only needs `.baton/` to exist somewhere above its `cwd` at runtime.

### 5. Register the hook for both turn-ending events

Pick the snippet for the tool the project actually uses. All three register the *same* script — they differ only in which file they go in and what the tool calls its end-of-turn event.

| Tool | Config file | Snippet | Events |
|---|---|---|---|
| Claude Code | `.claude/settings.json` | `templates/settings-snippet.json` | `Stop`, `StopFailure` |
| Codex | `.codex/hooks.json` | `templates/hooks-codex.json` | `Stop`, `SessionEnd` |
| Gemini CLI | `.gemini/settings.json` | `templates/hooks-gemini.json` | `AfterAgent`, `SessionEnd` |

If the file doesn't exist, create it with the snippet's content. If it does, merge into the existing arrays for those events — read the current file first, don't overwrite unrelated hooks (`PreToolUse`, `SessionStart`, etc.). Fix the script paths if step 4 used a non-default location.

Register both events for whichever tool, not just the turn-end one. The turn-end event misses the case where the session dies rather than finishes, which is exactly when the breadcrumb is worth having. On Claude Code that second event is `StopFailure`, which fires when a turn ends on an API error and carries an error instead of a last response; on Codex and Gemini CLI it's `SessionEnd`.

Substitute the path placeholder before writing:
- Claude Code's snippet uses `${CLAUDE_PROJECT_DIR}`, which the tool expands itself — leave it as is.
- The Codex and Gemini snippets carry a literal `<PROJECT_ROOT>`. Replace it with the project's absolute path. Both tools take `command` as a single string, so the path stays double-quoted inside it — a project path containing a space breaks an unquoted one.

Claude Code's snippet uses **exec form** instead — `"command": "node"` plus an `args` array — which its hooks documentation recommends for any hook referencing a path placeholder. In shell form the substituted path gets re-parsed by a shell (PowerShell or Git Bash on Windows), so a space in the path breaks it. Exec form spawns the executable directly with the path as one verbatim argument. Keep that shape if you edit it.

For a tool not in the table, don't guess at its config format. Install the script (step 4), then tell the user which of its lifecycle events to point at `scripts/baton-stop-hook.mjs`, or skip the hook — the markdown files work without it.

### 6. Keep the machine-local lanes out of git

Add both of these to `.gitignore` if they aren't already covered:

```
.baton/.session/
.baton/local/
```

`.session/` holds runtime info the hook writes (timestamps, transcript paths, raw last message). `local/` is where a person or the model puts machine-local config values and personal material that shouldn't reach the team — same file format as a normal baton, but never committed. Neither belongs in git or merges sensibly across machines.

Don't create `.baton/local/` during install — it's created on first use, and an empty ignored directory just adds noise.

### 7. Report what happened

Tell the user, briefly:
- Which files were created vs. already present and left alone
- The entry file used (`CLAUDE.md` or `AGENTS.md`)
- Which tool's hook config was wired up (or that the hook was skipped, and why). If they also use another agent on this project, the `.baton/` file convention works there too — that agent just needs the pointer line in its own entry file — but its sidecar only runs once that tool's hook is pointed at the same script.

Don't run a smoke test by actually triggering the hook (there's no way to fire one outside a real turn) — just verify the files exist and the hook config parses.

## Reference

`templates/` holds the files this skill writes, each usable as-is or with the placeholders described above:
- `baton-readme.md` (Korean, default) / `baton-readme.en.md` (English) — the `.baton/README.md` content (both have one placeholder: `{{CHANGELOG_LINE}}`)
- `baton-stop-hook.mjs` — the hook script for both `Stop` and `StopFailure`, no placeholders
- `settings-snippet.json` — Claude Code's `hooks.Stop` / `hooks.StopFailure` entries, to merge into `.claude/settings.json`
- `hooks-codex.json` — Codex's `Stop` / `SessionEnd` entries, to merge into `.codex/hooks.json` (replace `<PROJECT_ROOT>`)
- `hooks-gemini.json` — Gemini CLI's `AfterAgent` / `SessionEnd` entries, to merge into `.gemini/settings.json` (replace `<PROJECT_ROOT>`)
