# Jira Agile REST API — sprint management on board 3

Base: `https://magikabdul.atlassian.net/rest/agile/1.0`. Every call goes through
`bash organization-repository/claude/scripts/jira-api.sh <path below /rest/> [curl args]`, run from the workspace
root — the script loads `JIRA_EMAIL` / `JIRA_API_TOKEN` from `<workspace-root>/.env` and
never prints them (setup: `organization-repository/claude/SETUP.md`, section 4a). Never
source the `.env` yourself. `Content-Type: application/json` on writes; write JSON payloads
to temp files and use `-d @file.json`.

## Sprints of the board (name history for element naming)

```bash
bash organization-repository/claude/scripts/jira-api.sh 'agile/1.0/board/3/sprint?state=active,closed,future&maxResults=50'
```

Active only: `?state=active`. Response: `values[]` with `id`, `name`, `state`,
`startDate`, `endDate`.

## Issues and Done count of a sprint

```bash
bash organization-repository/claude/scripts/jira-api.sh api/3/search/jql --get \
  --data-urlencode 'jql=sprint = <sprintId>' \
  --data-urlencode 'fields=summary,status,parent' \
  --data-urlencode 'maxResults=100'
```

Done = issues with `fields.status.statusCategory.key == "done"`. Count both ways
(Done / not Done) from one response; page with `nextPageToken` if more than 100.

## Backlog candidates (top-up planning)

```bash
bash organization-repository/claude/scripts/jira-api.sh api/3/search/jql --get \
  --data-urlencode 'jql=project = HAS AND issuetype = Task AND statusCategory != Done AND sprint IS EMPTY ORDER BY parent, created' \
  --data-urlencode 'fields=summary,status,parent,labels'
```

## Create a sprint (state starts as `future`)

```bash
bash organization-repository/claude/scripts/jira-api.sh agile/1.0/sprint -X POST -H "Content-Type: application/json" \
  -d '{"name": "2026-08 Hydrogen", "originBoardId": 3}'
```

Response contains the new sprint `id`.

## Move issues into a sprint (max 50 keys per call)

```bash
bash organization-repository/claude/scripts/jira-api.sh agile/1.0/sprint/<newSprintId>/issue -X POST \
  -H "Content-Type: application/json" -d '{"issues": ["HAS-113", "HAS-114"]}'
```

Moving an issue that sits in the active sprint removes it from there — do this BEFORE
closing the old sprint (issues left in a sprint when it closes fall back to the backlog).

## Close the old sprint

`PUT` is a full update here too — `{"state": "closed"}` alone fails with
`400 {"errors":{"name":"Sprint name is required"}}` and leaves the rotation half-done
(new sprint created and filled, old one still active). GET the sprint first and resend
its `name`, `startDate` and `endDate` alongside the new state:

```bash
OLD=$(bash organization-repository/claude/scripts/jira-api.sh agile/1.0/sprint/<oldSprintId>)
NAME=$(echo "$OLD" | jq -r .name)
START=$(echo "$OLD" | jq -r .startDate)
END=$(echo "$OLD" | jq -r .endDate)

bash organization-repository/claude/scripts/jira-api.sh agile/1.0/sprint/<oldSprintId> -X PUT \
  -H "Content-Type: application/json" \
  -d "{\"name\": \"$NAME\", \"state\": \"closed\", \"startDate\": \"$START\", \"endDate\": \"$END\"}"
```

If the close does fail mid-rotation, nothing is lost: re-run this step, then activate the
new sprint. The issues already moved stay in the new sprint.

## Activate the new sprint (dates required for future → active)

```bash
START=$(date -u +%Y-%m-%dT%H:%M:%S.000Z)
END=$(date -u -d "+1 month" +%Y-%m-%dT%H:%M:%S.000Z)
bash organization-repository/claude/scripts/jira-api.sh agile/1.0/sprint/<newSprintId> -X PUT \
  -H "Content-Type: application/json" \
  -d "{\"name\": \"<sprint name>\", \"state\": \"active\", \"startDate\": \"$START\", \"endDate\": \"$END\"}"
```

PUT is a full update — omitting `name` fails with "Sprint name is required", so always
resend it (this applies to **every** sprint PUT, closing included). Only one sprint can be
active per board — close the old one first.

## Errors

- The script exits non-zero on any HTTP error and prints the body Jira answered with.
- `400` on activate — usually missing dates or another sprint still active.
- `401` — the token is missing, wrong or expired in `<workspace-root>/.env`; tell the user.
- `404` — wrong sprint id or the sprint belongs to another board.
- Verify every mutation with a follow-up GET (sprint state, issue counts).
