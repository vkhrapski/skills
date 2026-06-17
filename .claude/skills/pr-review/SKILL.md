---
   description: Peforms code review for pull requests associated with provided JIRA tickets.
---


## Step 1 — Get the Jira ticket

- If a URL was passed as an argument, use it. Otherwise ask the user to paste the Jira ticket link.
- Extract requirements from the ticket description.
- Extract the ticket key (e.g. `SFDC-1234`) from the URL.

---

## Step 2 — Find the associated GitHub PRs

Search for open PRs whose branch name or title contains the ticket key:

```bash
gh pr list --state open --search "<TICKET_KEY>" --json number,title,url,headRefName,author
```

Present the list to the user and confirm which PR(s) to review. If only one is found, proceed automatically.

---

## Step 3 — Fetch PR details and diff

For each PR to review, run:

```bash
gh pr view <PR_NUMBER> --json number,title,url,body,author,headRefName,baseRefName,additions,deletions,changedFiles,reviews,reviewRequests,assignees
gh pr diff <PR_NUMBER>
gh pr view <PR_NUMBER> --json files --jq '.files[].path'
```

Also fetch inline review comments already on the PR:

```bash
gh api repos/{owner}/{repo}/pulls/<PR_NUMBER>/comments
```

---

## Step 4 — Review the code

Carefully review the diff. Evaluate:

- **Correctness**: meeting acceptance criteria from the ticket, logic errors, missing null checks, off-by-one errors, wrong conditions
- **Security**: SOQL injection, exposed credentials, insecure apex patterns
- **Best practices**: Salesforce governor limits, bulkification, test coverage patterns, metadata naming conventions
- **Style / maintainability**: dead code, misleading names, overly complex logic

For each issue found, classify its criticality:

| Level | Meaning |
|-------|---------|
| **CRITICAL** | Must fix — bug, data loss risk, security vulnerability |
| **MAJOR** | Should fix — incorrect behaviour in edge cases, governor limit risk |
| **MINOR** | Nice to fix — style, readability, naming |
| **NIT** | Optional — very small preference |

---

## Step 5 — Present the issue report

Output a structured report like this:

```
## PR Review: <PR title> (#<number>)

### Issues Found

| # | Criticality | File | Description |
|---|-------------|------|-------------|
| 1 | CRITICAL | force-app/.../MyClass.cls | SOQL inside loop — governor limit risk |
| 2 | MAJOR    | force-app/.../MyClass.cls:45 | Null check missing before calling .size() |
| 3 | MINOR    | force-app/.../MyTest.cls | Test method name does not describe scenario |
| 4 | NIT      | force-app/.../MyClass.cls:12 | Unused variable `tempList` |

### Recommendation: REQUEST CHANGES  ← (or APPROVE if no CRITICAL/MAJOR issues)
```

Then ask the user:

> "Which issues should I include as PR comments? Enter issue numbers separated by commas, or 'all', or 'none'. Then confirm: approve or request changes?"

Wait for the user's response.

---

## Step 6 — Execute the decision

### Identify the previous assignee

Fetch the Jira ticket to find the current assignee (who submitted the PR — this is the person to re-assign to after review):

```
mcp__rovo__getJiraIssue  →  issue.fields.assignee
```

Save: `previous_assignee_account_id` and `previous_assignee_display_name`.

### If user chooses APPROVE

1. **Approve the PR(s)**:
   ```bash
   gh pr review <PR_NUMBER> --approve --body "LGTM"
   ```
   If the user selected specific issues to comment on, add them as inline comments first (see below), then approve.

2. **Leave a Jira comment** using `mcp__rovo__addCommentToJiraIssue` with ADF body (Jira REST API v3 requires ADF for mentions — plain `[~accountid:...]` wiki markup renders as literal text):
   ```json
   {
     "type": "doc",
     "version": 1,
     "content": [
       {
         "type": "paragraph",
         "content": [
           {
             "type": "mention",
             "attrs": {
               "id": "<previous_assignee_account_id>",
               "text": "@<previous_assignee_display_name>"
             }
           },
           {
             "type": "text",
             "text": " Pull request is approved."
           }
         ]
       }
     ]
   }
   ```

3. **Re-assign the Jira ticket** to the previous assignee (use `mcp__rovo__editJiraIssue`, field `assignee: { accountId: "<id>" }`).

---

### If user chooses REQUEST CHANGES

1. **Post selected issues as inline PR comments** for each selected issue:
   - If a file + line number is known, use the GitHub API to post an inline review comment:
     ```bash
     gh api repos/{owner}/{repo}/pulls/<PR_NUMBER>/reviews \
       --method POST \
       --field body="<overall comment>" \
       --field event="REQUEST_CHANGES" \
       --field 'comments[][path]=<file>' \
       --field 'comments[][line]=<line>' \
       --field 'comments[][body]=<issue description>'
     ```
   - Bundle all selected issues into a single review submission with event `REQUEST_CHANGES`.
   - If no specific line is known for an issue, post it as a top-level review comment body instead.

2. **Leave a Jira comment** using `mcp__rovo__addCommentToJiraIssue` with ADF body:
   ```json
   {
     "type": "doc",
     "version": 1,
     "content": [
       {
         "type": "paragraph",
         "content": [
           {
             "type": "mention",
             "attrs": {
               "id": "<previous_assignee_account_id>",
               "text": "@<previous_assignee_display_name>"
             }
           },
           {
             "type": "text",
             "text": " Please review comments in the pull request."
           }
         ]
       }
     ]
   }
   ```

3. **Re-assign the Jira ticket** to the previous assignee (use `mcp__rovo__editJiraIssue`, field `assignee: { accountId: "<id>" }`).

---

## Notes

- Always use the `gh` CLI for GitHub operations (PRs, reviews, comments). The repo remote is inferred from `git remote -v`.
- Always use `mcp__rovo__*` tools for Jira operations.
- Jira comments MUST use ADF format (JSON) for mentions. Never use wiki markup `[~accountid:...]` — it renders as plain text in Jira REST API v3.
- If multiple PRs are reviewed, apply Step 6 actions to each PR separately, but post only one Jira comment and one re-assignment.
- If the Jira ticket has no assignee, skip the re-assignment step and omit the mention from the comment; notify the user.