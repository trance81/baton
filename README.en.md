<div align="center">

<img src="assets/logo-tagline.png" alt="Baton" width="180">

# Baton

**An AI coding agent skill so work survives a dead session**

One markdown file per task. It isn't deleted when the task ends.

[![License: 0BSD](https://img.shields.io/badge/License-0BSD-black.svg)](LICENSE)
[![Agent skill](https://img.shields.io/badge/AI%20agent-skill-d97757)](SKILL.md)
[![Zero deps](https://img.shields.io/badge/dependencies-0-brightgreen)](templates/baton-stop-hook.mjs)

[한국어](README.md) · English

</div>

---

## Install

```bash
npx skills add trance81/baton -g
```

Then one sentence to whichever agent you use:

```
add baton to this project
```

## What changes

A CLI session forgets everything once it's gone. There's a `resume` list, but
after a few days, titles and timestamps alone don't tell you which one was which.

<table>
<tr><th width="50%">Without baton</th><th width="50%">With baton</th></tr>
<tr valign="top"><td>

```
"let's finish the webhook"

→ skim the diff, infer
  how far it got

→ why it was built that
  way isn't in the code

→ explain it again
```

</td><td>

```
"let's finish the webhook"

→ reads .baton/payment-
  webhook.md (running)

→ "Signature verification is
   done; retry policy is next.
   We skipped the queue — a few
   hundred a day, so overkill."
```

</td></tr>
</table>

A diff tells you **what changed**. It doesn't tell you **what you were in the
middle of, or why you chose it**. Those two live in the baton.

Batons stay after the task ends, which makes this work months later:

```
"how do we deploy to staging again?"
   → search .baton/ → deploy-pipeline.md (passed)
   → the steps, why that order, what blocked you at the time
```

## Four invariants

```
1. .baton/<task>.md is the single durable source of truth for task state.
2. Don't record every turn — only the state the next session needs.
3. The sidecar is a local recovery breadcrumb, nothing more.
4. Past batons stay in git forever, but only enter context when needed.
```

## How you use it

**1 · Install** — the skill creates `.baton/README.md` (the operating rules),
adds a pointer line to the project's entry file (`CLAUDE.md`, `AGENTS.md`,
whichever it uses), registers an end-of-turn hook, and updates `.gitignore`.
It leaves existing pieces alone, so re-running it is safe.

**2 · Start** — a baton appears when work won't finish today. Typo fixes don't
get one.

```markdown
---
status: running
updated: 2026-08-19T09:05:00Z
branch: feature/payment-webhook
---

# Payment webhook

## Current state
Endpoint skeleton only. No signature verification yet.

## Next steps
Add signature verification.
```

Only `status` and `updated` are required.

**3 · Resume** — a new session reads only the `running`/`waiting` batons at the
top level of `.baton/` and continues from there. It doesn't load every past
baton; that would have baton polluting its own context.

**4 · Finish** — flip it to `status: passed` and keep it. This is what you
search months later when you're wondering how something was done.

## Folder layout

```
.baton/
├─ README.md        the rules                       git
├─ <slug>.md        one baton per task/feature      git
├─ done/            archive (optional)              git
├─ local/           machine-local notes (optional)  git-ignored
└─ .session/        runtime records from the hook   git-ignored
```

## What to write, and what not to

| | Write | Don't write |
|---|---|---|
| Access | `ssh prod-web-01` (alias in `~/.ssh/config`) | Passwords, private keys |
| Keys | 1Password item `"prod-server-ssh"` | `AWS_SECRET_ACCESS_KEY=...` |
| Steps | Run `scripts/deploy.sh`, reload nginx | Token strings |

> [!WARNING]
> `.baton/*.md` goes into git. Your team sees it; on a public repo, anyone does.
> **Record where it lives and what it's called.** That's enough for the next
> session to work out how to connect, and no value ever lands in git history.

Config values that only exist on this machine, and personal material you can't
share with the team, go in `local/`, which is never committed. Treat it as
**reference only** — it can be deleted at any time, so don't build a procedure
that depends on it.

<details>
<summary><b>How this differs from an agent's built-in memory</b></summary>

<br>

Most CLI agents ship some form of memory. They tend to share two traits: it's
**machine-local**, and what gets saved is left to the model's judgment. Claude
Code's [Auto memory](https://code.claude.com/docs/en/memory), for instance,
lives in `~/.claude/projects/<repo>/memory/`, and its docs say outright it isn't
"shared across machines or cloud environments."

| | Built-in memory | baton |
|---|---|---|
| Storage | Machine-local | `.baton/*.md` — git-committed |
| Team sharing | No | Yes, via git push |
| What it keeps | General learnings, the model's call | One task's confirmed state |
| Shape | Summary notes the model curates | A work document people read as-is |

They don't overlap — baton fills a gap built-in memory explicitly doesn't cover
(git-based sharing across machines and people).

</details>

<details>
<summary><b>What the sidecar does</b></summary>

<br>

At the end of every turn, the hook rewrites
`.baton/.session/<session-id>.json`. It never calls the model, so it costs zero
tokens.

```json
{
  "observedAt": "2026-08-19T17:54:00Z",
  "event": "Stop",
  "transcriptPath": "C:/Users/.../a1b2c3d4-....jsonl",
  "lastMessage": "Signature check done, next is retry backoff."
}
```

`observedAt` is **when the hook last observed this session**. It says nothing
about whether the baton is current. It's the **recovery breadcrumb** you open
when a baton looks doubtful or a session ended abnormally. `event` carries
whatever the hosting tool calls the event — on Claude Code, a turn that ends on
an API error reads `StopFailure` and brings an `error` with it.

</details>

<details>
<summary><b>Limitations worth knowing</b></summary>

<br>

- **Not every ending is observable.** A user interrupt (Ctrl+C/ESC), a forced
  kill, or a power cut means the hook never runs. Baton doesn't try to catch
  those. The source of truth is the markdown file regardless, and the sidecar is
  a best-effort breadcrumb.
- **The `.baton/` file convention isn't tied to a tool.** Any agent that reads
  and writes files can follow it once the rule is in the project's entry file
  (`CLAUDE.md`, `AGENTS.md`, …). The sidecar hook script is tool-neutral too — it
  works with any tool that pipes an end-of-turn JSON payload on stdin. Ready-made
  config snippets ship for **Claude Code, Codex, and Gemini CLI**; on anything else,
  point that tool's own hook config at the same script.
- Projects without git drop the `branch`/`assignee` fields. Batons themselves
  still work, but **there's no history layer at all** — overwrite a body and the
  previous text is gone, because the `git log` note below assumes git is filling
  that role. On such a project, it pays to keep the "why this direction" section
  fuller than you otherwise would.
- `local/` isn't committed, so **it doesn't follow you to another machine.**
  These values differ per machine anyway, but anything the team needs belongs in
  the git lane — as a reference, not a value.
- Baton doesn't hold a chronological record. A baton body is overwritten, so
  that job belongs to `git log`. That's deliberate: accumulating history inside
  the file grows what every later session has to read, and splits the source of
  truth in two.
- **The read scope is a convention, not an enforcement.** A hook is a shell
  command; it can't govern what the model reads. Invariant 4 holds because the
  entry file and `.baton/README.md` say so, and breaking it wastes context. A
  `PreToolUse` hook could hard-block reads under `done/`, but that would also
  block the "how did we do this last time" lookup — the thing to prevent and the
  thing to keep are the same action, so it stays unenforced. Instead `done/` is
  a subdirectory, which makes the default path the cheap one.

</details>

<details>
<summary><b>Other ways to install</b></summary>

<br>

This project only:

```bash
npx skills add trance81/baton
```

It unpacks into `.agents/skills/baton-init/` and symlinks from the skills
folder of each installed tool (`.claude/skills/`, …), so updating the one copy
reaches every linked tool.

Manually — copy it into the skills folder of whichever tool you use:

```bash
git clone https://github.com/trance81/baton.git
cp -r baton <skills-folder>/baton-init   # e.g. ~/.claude/skills/baton-init
```

</details>

---

<div align="center">

0BSD · `main` is always the current convention

</div>
