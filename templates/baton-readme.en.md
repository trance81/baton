# Baton — task state that survives sessions

## Four invariants

1. **`.baton/<slug>.md` is the single durable source of truth** for task
   state. Neither the sidecar nor the transcript outranks it.
2. **Don't record every turn** — only the state the next session needs.
3. **The sidecar isn't truth, it's a local recovery breadcrumb** — for when
   a session ended abnormally.
4. **Past batons stay in git forever, but only enter context when needed.**

## Folder layout

```
.baton/
├─ README.md        this file — the rules              (git)
├─ <slug>.md        one baton per task/feature         (git)
├─ done/            archived finished batons (optional)(git)
├─ local/           machine-local notes (optional)     (git-ignored)
└─ .session/        runtime records the hook writes    (git-ignored)
```

Finished work isn't deleted. The accumulated files become a searchable
index of the project's tasks — you find things by content,
not by scrolling a `resume` list that stopped being legible long ago.

## Rules

1. **When a task will span multiple sessions**, create `<slug>.md`. Naming
   the file after the related branch keeps the mapping visible. Don't
   create one for work that finishes in a single sitting (typo fixes, etc.).
2. **As the task progresses, update that same file.** Don't create a new
   one. Don't update every turn — only at meaningful boundaries. Don't
   record intermediate trial-and-error (git history already has it).
3. **When the task is done, flip it to `status: passed` and keep the
   file.** Don't delete it.{{CHANGELOG_LINE}}
4. **Keep what you read at session start narrow** — see below.

## At session start

```
1. Check this README
2. Read only the running / waiting batons at the top level of .baton/
3. Search done/ and passed only when relevant to the current request
4. Consult local/ only when needed — work must proceed without it
```

Loading every past baton into context each session means baton pollutes
its own context. **Keep it in git forever; load it into context on demand.**

## File format

```markdown
---
status: running        # required. running | waiting | passed
updated: 2026-08-19T14:32:10Z   # required. When this doc was last meaningfully updated (ISO)
host: desk-office       # optional. Last machine that touched this
branch: feature/xxx     # optional. git only
assignee: hong           # optional. Only when several people share this folder
session: <session-id>   # optional. A shortcut for finding the sidecar, nothing more —
                         # a model doesn't always know its own session id, so this
                         # can go stale. Recovery must not depend on it.
---

# Title

## Current state
...

## Next steps
...

## Why this direction
(optional) Discarded options + one-line reason each. Don't pad into prose.

## Done when
(optional) Verifiable criteria only.

## Blocked on
...
```

**Only `status` and `updated` are required.** The rest are nice to have.
Don't add new fields — a person understanding the file in two seconds is
the whole point of this format.

`updated` means **when this document was last meaningfully updated**. It is
not a per-turn stamp.

## What to write, and what not to

`.baton/*.md` **goes into git.** Your team sees it; on a public repo,
anyone does.

**Write** reusable information:

```
Access: ssh prod-web-01   (alias defined in ~/.ssh/config)
Key: 1Password item "prod-server-ssh"
Deploy: run scripts/deploy.sh, then reload nginx
```

**Don't write** the credentials themselves:

```
password: ...
AWS_SECRET_ACCESS_KEY=...
-----BEGIN PRIVATE KEY-----
```

Never write the value of a password, API key, token, or private key.
**Write where it lives and what it's called.** That's enough for the next
session to work out how to connect.

### `local/` — machine-local notes

Config values that only exist on this machine, and personal material that
can't be shared with the team, go in `.baton/local/`. Same file format, but
never committed.

- **It's reference material.** Work must proceed without it. It isn't part
  of the session-start checklist.
- **It can disappear at any time** — mid-task or after. Don't build a
  procedure that depends on something being there.
- Since it isn't committed, **it doesn't follow you to another machine.**
  These values differ per machine anyway; that's the point.
- This still isn't a place to collect credentials. Your local secret
  manager, `.env`, and `~/.ssh/` already do that job.

## The sidecar (`.session/`)

A runtime record written by the hook. Not git-tracked, not hand-edited.

```json
{ "observedAt": "...", "event": "Stop", "transcriptPath": "...", "lastMessage": "..." }
```

`observedAt` is **when the hook last observed this session** — not a claim
that the baton is current. If the turn ended on an API error, `event` reads
`StopFailure` and an `error` comes with it.

**The sidecar does not prove the baton is fresh.** It preserves the last
observed session state, so you have a breadcrumb when a baton looks
doubtful or a session ended abnormally.

How to use it:

```
1. Read the running / waiting batons
2. If a baton looks doubtful, take the sidecar in .session/ with the most
   recent observedAt (a baton's `session` field points straight at one, but
   it may be stale)
3. If the last response differs meaningfully from the baton's contents,
   check transcriptPath and correct the baton
```

`observedAt` being newer than `updated` is **normal during active work** —
the body is only written at boundaries. It's worth a look once that session
has ended and enough time has passed since the last turn (say, 5 minutes)
with the body still unchanged. Even then it's a cue for a human to judge,
not a mechanical verdict.

**Some endings can't be observed at all.** A user interrupt (Ctrl+C/ESC), a
forced kill, or a power cut means the hook never runs. Baton doesn't try to
catch those — the source of truth is `.baton/*.md` regardless, and the
sidecar is a best-effort breadcrumb.

## When this folder grows

**Don't delete anything** — this folder is the project's task index.

- If `passed` files crowd the top level, move them into `.baton/done/`.
  They drop out of the session-start skim while staying alive and
  searchable.
- If there's reusable knowledge in one (setup procedures, decisions and
  their reasoning), *copy* it into the project's knowledge base or
  `CLAUDE.md`/`AGENTS.md`. This adds to the baton, it doesn't replace it —
  the knowledge base holds the curated "why we decided this," baton holds
  "how far this task got," per task. Neither is the chronological record —
  that's git log, since a baton body gets overwritten.
