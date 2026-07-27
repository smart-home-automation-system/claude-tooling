# Jira Agile REST API — sprint management on board 3

Base: `https://magikabdul.atlassian.net/rest/agile/1.0`. Auth: HTTP Basic
`$JIRA_EMAIL:$JIRA_API_TOKEN` from `<workspace-root>/.env`, `Content-Type:
application/json` on writes. Write JSON payloads to temp files and use `-d @file.json`.

## Sprints of the board (name history for element naming)

```bash
curl -s -u "$JIRA_EMAIL:$JIRA_API_TOKEN" \
  "https://magikabdul.atlassian.net/rest/agile/1.0/board/3/sprint?state=active,closed,future&maxResults=50"
```

Active only: `?state=active`. Response: `values[]` with `id`, `name`, `state`,
`startDate`, `endDate`.

## Issues and Done count of a sprint

```bash
curl -s -u "$JIRA_EMAIL:$JIRA_API_TOKEN" --get \
  --data-urlencode 'jql=sprint = <sprintId>' \
  --data-urlencode 'fields=summary,status,parent' \
  --data-urlencode 'maxResults=100' \
  "https://magikabdul.atlassian.net/rest/api/3/search/jql"
```

Done = issues with `fields.status.statusCategory.key == "done"`. Count both ways
(Done / not Done) from one response; page with `nextPageToken` if more than 100.

## Backlog candidates (top-up planning)

```bash
curl -s -u "$JIRA_EMAIL:$JIRA_API_TOKEN" --get \
  --data-urlencode 'jql=project = HAS AND issuetype = Task AND statusCategory != Done AND sprint IS EMPTY ORDER BY parent, created' \
  --data-urlencode 'fields=summary,status,parent,labels' \
  "https://magikabdul.atlassian.net/rest/api/3/search/jql"
```

## Create a sprint (state starts as `future`)

```bash
curl -s -u "$JIRA_EMAIL:$JIRA_API_TOKEN" -H "Content-Type: application/json" \
  -X POST "https://magikabdul.atlassian.net/rest/agile/1.0/sprint" \
  -d '{"name": "2026-08 Hydrogen", "originBoardId": 3}'
```

Response contains the new sprint `id`.

## Move issues into a sprint (max 50 keys per call)

```bash
curl -s -u "$JIRA_EMAIL:$JIRA_API_TOKEN" -H "Content-Type: application/json" \
  -X POST "https://magikabdul.atlassian.net/rest/agile/1.0/sprint/<newSprintId>/issue" \
  -d '{"issues": ["HAS-113", "HAS-114"]}'
```

Moving an issue that sits in the active sprint removes it from there — do this BEFORE
closing the old sprint (issues left in a sprint when it closes fall back to the backlog).

## Close the old sprint

`PUT` is a full update here too — `{"state": "closed"}` alone fails with
`400 {"errors":{"name":"Sprint name is required"}}` and leaves the rotation half-done
(new sprint created and filled, old one still active). GET the sprint first and resend
its `name`, `startDate` and `endDate` alongside the new state:

```bash
OLD=$(curl -s -u "$JIRA_EMAIL:$JIRA_API_TOKEN" \
  "https://magikabdul.atlassian.net/rest/agile/1.0/sprint/<oldSprintId>")
NAME=$(echo "$OLD" | jq -r .name)
START=$(echo "$OLD" | jq -r .startDate)
END=$(echo "$OLD" | jq -r .endDate)

curl -s -u "$JIRA_EMAIL:$JIRA_API_TOKEN" -H "Content-Type: application/json" \
  -X PUT "https://magikabdul.atlassian.net/rest/agile/1.0/sprint/<oldSprintId>" \
  -d "{\"name\": \"$NAME\", \"state\": \"closed\", \"startDate\": \"$START\", \"endDate\": \"$END\"}"
```

If the close does fail mid-rotation, nothing is lost: re-run this step, then activate the
new sprint. The issues already moved stay in the new sprint.

## Activate the new sprint (dates required for future → active)

```bash
START=$(date -u +%Y-%m-%dT%H:%M:%S.000Z)
END=$(date -u -d "+1 month" +%Y-%m-%dT%H:%M:%S.000Z)
curl -s -u "$JIRA_EMAIL:$JIRA_API_TOKEN" -H "Content-Type: application/json" \
  -X PUT "https://magikabdul.atlassian.net/rest/agile/1.0/sprint/<newSprintId>" \
  -d "{\"name\": \"<sprint name>\", \"state\": \"active\", \"startDate\": \"$START\", \"endDate\": \"$END\"}"
```

PUT is a full update — omitting `name` fails with "Sprint name is required", so always
resend it (this applies to **every** sprint PUT, closing included). Only one sprint can be
active per board — close the old one first.

## Errors

- `400` on activate — usually missing dates or another sprint still active.
- `404` — wrong sprint id or the sprint belongs to another board.
- Verify every mutation with a follow-up GET (sprint state, issue counts).
