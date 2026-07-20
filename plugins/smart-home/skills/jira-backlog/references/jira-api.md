# Jira Cloud REST API v3 — templates for the HAS project

Base URL: `https://magikabdul.atlassian.net`. Auth: HTTP Basic with
`$JIRA_EMAIL:$JIRA_API_TOKEN` (load from `<workspace-root>/.env`). Always pass
`-H "Content-Type: application/json"` on writes. Never echo the token; prefer
`curl -s -u "$JIRA_EMAIL:$JIRA_API_TOKEN"` with variables sourced from the env file.

## Auth check

```bash
curl -s -u "$JIRA_EMAIL:$JIRA_API_TOKEN" \
  "https://magikabdul.atlassian.net/rest/api/3/myself" | head -c 300
```

Expect JSON with `accountId`. 401 → wrong/expired token; tell the user to regenerate at
id.atlassian.com → Security → API tokens.

## List open epics

```bash
curl -s -u "$JIRA_EMAIL:$JIRA_API_TOKEN" --get \
  --data-urlencode 'jql=project = HAS AND issuetype = Epic AND statusCategory != Done ORDER BY created DESC' \
  --data-urlencode 'fields=summary,status' \
  "https://magikabdul.atlassian.net/rest/api/3/search/jql"
```

(`/rest/api/3/search/jql` is the current endpoint; if it 404s on an older site, fall back
to `/rest/api/3/search` with the same params.)

## Create an epic

```bash
curl -s -u "$JIRA_EMAIL:$JIRA_API_TOKEN" -H "Content-Type: application/json" \
  -X POST "https://magikabdul.atlassian.net/rest/api/3/issue" \
  -d @epic.json
```

`epic.json`:

```json
{
  "fields": {
    "project": { "key": "HAS" },
    "issuetype": { "name": "Epic" },
    "summary": "Epic title",
    "description": { "type": "doc", "version": 1, "content": [
      { "type": "paragraph", "content": [ { "type": "text", "text": "Epic goal in one paragraph." } ] }
    ] }
  }
}
```

## Create a task (with epic parent, labels, ADF description + DoD checkboxes)

```json
{
  "fields": {
    "project": { "key": "HAS" },
    "issuetype": { "name": "Task" },
    "parent": { "key": "HAS-<epicNumber>" },
    "summary": "repo-name: imperative task title",
    "labels": ["frontend"],
    "description": {
      "type": "doc",
      "version": 1,
      "content": [
        { "type": "heading", "attrs": { "level": 3 },
          "content": [ { "type": "text", "text": "Context" } ] },
        { "type": "paragraph",
          "content": [ { "type": "text", "text": "Why this task exists." } ] },
        { "type": "heading", "attrs": { "level": 3 },
          "content": [ { "type": "text", "text": "Implementation details" } ] },
        { "type": "bulletList", "content": [
          { "type": "listItem", "content": [ { "type": "paragraph",
            "content": [ { "type": "text", "text": "Detail with " },
                         { "type": "text", "text": "code", "marks": [ { "type": "code" } ] } ] } ] }
        ] },
        { "type": "heading", "attrs": { "level": 3 },
          "content": [ { "type": "text", "text": "Definition of Done" } ] },
        { "type": "taskList", "attrs": { "localId": "dod" }, "content": [
          { "type": "taskItem", "attrs": { "localId": "dod-1", "state": "TODO" },
            "content": [ { "type": "text", "text": "First DoD item" } ] },
          { "type": "taskItem", "attrs": { "localId": "dod-2", "state": "TODO" },
            "content": [ { "type": "text", "text": "Second DoD item" } ] }
        ] }
      ]
    }
  }
}
```

Notes:
- `parent.key` assigns the epic (works in team-managed and company-managed projects).
- Omit `labels` for backend tasks; `frontend` label marks tasks Claude implements.
- Each `taskItem` needs a unique `localId` within the document; `state` is `TODO`.
- Response: `{"id":"...","key":"HAS-123","self":"..."}`. Browse URL:
  `https://magikabdul.atlassian.net/browse/HAS-123`.
- Write each payload to a temp file (scratchpad) and pass with `-d @file.json` — inline
  `-d '...'` with nested quotes breaks easily.

## Verify a created issue

```bash
curl -s -u "$JIRA_EMAIL:$JIRA_API_TOKEN" \
  "https://magikabdul.atlassian.net/rest/api/3/issue/HAS-123?fields=summary,parent,labels" \
  | head -c 500
```

## Error handling

- `400` — check the JSON (most often: unknown field for the project type, bad ADF).
  The response body names the offending field.
- `401/403` — credentials/permissions; re-run the auth check.
- `404` on `parent` — epic key typo or epic in another project.
