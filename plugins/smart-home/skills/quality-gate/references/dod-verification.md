# quality-gate — verification and Jira update templates

Auth as everywhere: `$JIRA_EMAIL:$JIRA_API_TOKEN` from `<workspace-root>/.env` (Jira),
`gh` CLI (GitHub). Never print the token.

## Read the DoD with checkbox states

```bash
curl -s -u "$JIRA_EMAIL:$JIRA_API_TOKEN" \
  "https://magikabdul.atlassian.net/rest/api/3/issue/HAS-<n>?fields=summary,status,parent,description"
```

DoD = the `taskList` node in `fields.description.content`; each `taskItem` has
`attrs.localId` and `attrs.state` (`TODO`/`DONE`) and text in `content[0].text`.

## Evidence commands (read-only)

```bash
# merged PR from the convention branch
gh pr list -R smart-home-automation-system/<repo> --state merged \
  --search "head:feature/HAS-<n>" --json number,title,mergedAt,mergeCommit

# CI on main after the merge
gh run list -R smart-home-automation-system/<repo> --branch main --limit 5 \
  --json name,conclusion,headSha,createdAt

# release + artifacts
gh release list -R smart-home-automation-system/<repo> -L 3
docker manifest inspect magikabdul/<repo>:<version> >/dev/null && echo "image exists"
gh api /orgs/smart-home-automation-system/packages/maven/<group.artifact>/versions --jq '.[].name'
```

For `cholewa-commons`/`cholewa-security` substitute the personal account
(`-R magikabdul/<repo>`, packages under `/users/magikabdul/packages/...`).

## Tick verified checkboxes

GET the full description ADF, set `attrs.state = "DONE"` on the verified `taskItem`s
(match by text, not position), PUT back the whole description:

```bash
curl -s -o /dev/null -w "%{http_code}" -u "$JIRA_EMAIL:$JIRA_API_TOKEN" \
  -H "Content-Type: application/json" \
  -X PUT "https://magikabdul.atlassian.net/rest/api/3/issue/HAS-<n>" \
  -d @updated-description.json   # {"fields": {"description": <full modified ADF>}}
```

Modify only checkbox states — do not touch other content. Verify with a follow-up GET.

## Gate report comment

```bash
curl -s -u "$JIRA_EMAIL:$JIRA_API_TOKEN" -H "Content-Type: application/json" \
  -X POST "https://magikabdul.atlassian.net/rest/api/3/issue/HAS-<n>/comment" \
  -d @comment.json
```

`comment.json` body is ADF: heading "Quality gate report" + a bulletList with one line
per DoD item (`DONE — evidence`, `N/A — reason`, `MISSING — what remains`), ending with
the PASSED/FAILED verdict paragraph.
