---
name: jira-sprint
description: Manage the monthly sprint on the Jira HAS board (magikabdul.atlassian.net, board 3) — check sprint status, close the sprint once at least 7 tasks are Done, roll unfinished tasks into a new one-month sprint named after the next chemical element, and top the new sprint up to at least 10 tasks together with the user. Use whenever the user wants to check, close, rotate or start a sprint, or asks about sprint progress ("sprawdź sprint", "zamknij sprint", "utwórz nowy sprint", "co ze sprintem", "sprint status").
---

# jira-sprint

Solo-developer sprint cadence: one sprint at a time, one month long, rotated when enough
work is Done. This skill verifies, proposes, and — only after explicit user confirmation —
rotates. It never assigns people, changes issue statuses, or touches sprint scope beyond
what is agreed in the conversation.

Jira site: `https://magikabdul.atlassian.net`, project `HAS`, board `3`.
Credentials: same `<workspace-root>/.env` as jira-backlog (`JIRA_EMAIL`,
`JIRA_API_TOKEN`); never print the token; verify auth with `myself` first.
Endpoints and curl templates: `references/agile-api.md`. Element order for names:
`references/elements.md`.

## The rules (org decisions)

- Sprint length: **1 month** from the day it starts.
- A sprint is **eligible to close when ≥ 7 issues are Done** (status category Done).
- On rotation, every non-Done issue of the old sprint **moves to the new sprint**.
- The new sprint needs **≥ 10 issues**; when short, the top-up is agreed with the user.
- Sprint names: `YYYY-MM <Element>` — chemical elements in atomic-number order
  (`2026-08 Hydrogen`, `2026-09 Helium`, …). Find the last element used by scanning
  existing sprint names on the board (state=active,closed,future) and take the next one;
  start at Hydrogen if none match. The `YYYY-MM` prefix is the month the sprint starts.

## Workflow

**1. Status.** Fetch the active sprint of board 3. Count its issues by status category
(Done vs the rest) and always report: sprint name, dates, Done count vs the ≥7 threshold,
remaining issues with keys and statuses.

**2. No active sprint?** Say so, then plan the first/new sprint content with the user:
fetch backlog issues grouped by epic and propose a sensible ≥10 selection (respect
dependency order — e.g. library tasks before their consumers). After the user agrees to
the content and the name/dates, create the sprint, move the issues in, activate it.

**3. Active sprint, Done < 7.** Not eligible: report how many are missing and stop.
Exception: the user explicitly asks to close anyway — their call, proceed as in step 4
after restating the shortfall.

**4. Active sprint, Done ≥ 7 — propose the rotation.** Present, and wait for explicit
confirmation before touching anything:
- old sprint: name, Done count (this is the velocity note), issues that will be carried over
- new sprint: proposed name (next element), start = today, end = +1 month
- resulting size after carry-over; if < 10, the top-up proposal (step 5) is part of the
  same plan

**5. Top-up below 10.** Fetch backlog candidates (epics with work already in progress
first, then dependency-orderly picks). Propose; the user decides what goes in. Never
top up silently.

**6. Execute (only after confirmation).** Order matters:
1. create the new sprint (state `future`, with `originBoardId: 3`),
2. move all carried-over + agreed top-up issues into it (batches of ≤ 50),
3. close the old sprint,
4. activate the new one (start/end dates required).

**7. Report.** Closed sprint summary (name, Done count), new sprint (name, dates, issue
table with statuses), and verify by re-fetching the active sprint — it must be the new
one, with the expected issue count.
