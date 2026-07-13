---
name: project-ticket-summary
description: Summarize ticket movement on a GitHub Project over a lookback window — what closed, what moved status, what got opened — with a one-line gist of the actual work behind each closed ticket instead of just its title.
compatibility: claude
license: MIT
allowed_tools:
  - github
metadata:
  author: joystream
  version: "1.0"
  category: developer-tools
---

# Purpose

Produce a concise summary of ticket activity on a GitHub Project. This is the
"ticket changes" half of a team news digest: it answers *"what did we finish and
what's newly on the board?"* — e.g. "closed 5 tickets yesterday, here's the gist
of each." It only reads and summarizes — it returns a digest and posts nowhere, so
the agent that runs it chooses the destination. Reusable on its own or alongside
`repo-activity-summary`.

# Inputs

- **project** (string, required): the GitHub Project to read. For JoyStream this is
  the org project named **"development project"** (resolve its project number/URL).
- **repositories** (list of `owner/repo`, optional): scope the project's items to
  these repos when the project spans several. Default for JoyStream:
  `joystream-ai/joystream`, `joystream-ai/llm-wiki`.
- **lookback_hours** (integer, default `24`): only include items whose status
  changed within the window.

# Reasoning Flow

1. Compute the cutoff: `since = now - lookback_hours` (UTC).
2. Resolve the project and read its items via `github` (Projects v2 GraphQL).
3. Bucket items whose relevant timestamp is `>= since`:
   - **Closed / Done** — moved to a terminal status or the underlying issue was closed.
   - **Moved** — status field changed (e.g. Todo → In Progress) but not terminal.
   - **Opened / Added** — newly added to the board in the window.
4. **For each closed ticket, write a one-line gist of the work**, not just the title.
   Pull the gist from the linked PR(s), the closing commit, or the issue's final
   comments — describe what was actually done and why it mattered. Cite the ticket
   number and any linked PR.
5. Lead with a headline count ("Closed 5 tickets, moved 3, opened 2") then the details.
6. Return markdown plus a structured object.

**Trigger-aware behavior:** on `trigger.type == "manual"`, return the draft for review;
on `cron` / `agent_call`, return the finished digest directly.

# Output

```
**Development Project — last 24h**
Closed 5 · Moved 3 · Opened 2

Closed
- #528 Cron unique-index bug — scoped the index to cron rows so duplicate bindings
  can't crash trigger creation.
- #517 Agent creation in one go — Lanes A/B/D plus BFF and session-nav fixes landed.
- …

Moved / In progress
- #405 MCP discovery — now In Review after Phase 1 authoring shipped.

Newly opened
- #531 Digest agent for team news.
```

Plus a structured object:
- `counts` (`closed`, `moved`, `opened`)
- `closed[]` (each: number, title, gist, refs[])
- `moved[]` (each: number, title, from_status, to_status)
- `opened[]` (each: number, title)

# Constraints

- **Every closed ticket gets a gist of the real work**, sourced from its PR/commit/
  comments — never fabricated. If no source explains the work, say
  "closed (no linked PR / description)" rather than guessing.
- Do not restate ticket titles as the whole summary; the title plus a substantive
  gist is the minimum.
- Keep the closed section to the tickets that actually closed in the window; do not
  pull in older done items.
- If the project spans repos beyond the requested scope, exclude out-of-scope items
  and note that you scoped the view.

# Edge Cases

## No ticket movement in the window
Return: `**Development Project** — no ticket changes in the last {lookback_hours}h.`

## Ticket closed without a linked PR
Include it with the gist marked `closed (no linked PR)`; do not omit it and do not invent work.

## Project not found / not accessible
Return an explicit failure line naming the project and the reason, so the composing
digest can flag that the ticket half is missing rather than appear empty.

# Examples

## Busy sprint day
**Input:** `project: "development project"`, `lookback_hours: 24`.
**Output:** Headline "Closed 5 · Moved 3 · Opened 2", each closed ticket with a
one-line gist sourced from its merged PR, followed by moved and opened lists.
