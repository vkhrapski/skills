---
name: create-jira-ticket
description: >-
  Turn any input — a freeform request, a pasted Slack/email thread, a Salesforce
  platform notification, or an error/failure/exception email — into a well-formed
  Jira ticket and create it in the  Jira (default project SFDC). Use
  this skill whenever the user wants to file, open, raise, or log a Jira
  ticket/issue, including phrasings like "make a ticket for this", "ticket this
  up", "open a Story for...", "we need a Jira for...", or when they paste a Slack
  or email conversation to capture as work. ALSO use it when the user pastes a
  Salesforce error or failure email (Apex trigger exception, Flow runtime error,
  EmailToCase failure, batch job failure, email deliverability alert, Gearset
  deployment/test failure) or a Salesforce platform notification (API version
  retirement, feature deprecation, scheduled maintenance) — these should be
  analyzed and turned into a Bug or impact-analysis ticket. Use it even
  if the user doesn't say the word "Jira", as long as they clearly want a tracked
  work item created from what they pasted.
---

## Workflow

1. **Read the input and extract the work item.** Figure out what is actually being
   asked for. From a Slack/email thread this is the hard part — strip out the
   back-and-forth, side comments, and pleasantries, and identify the real ask, who
   wants it, and why. If the thread contains a decision or conclusion, that's usually
   the ticket; the discussion is context.

2. **Choose the issue type.** Match intent to type rather than defaulting blindly:
   - Something is broken / not behaving as expected, or an error/exception/failure
     email → **Bug**
   - A new capability, feature, or user-facing change → **Story**
   - A chore, investigation, config change, general to-do, or impact-analysis work
     prompted by a platform notification → **Task**

   If you genuinely can't tell, prefer **Task** and mention your reasoning when you
   confirm.

3. **Draft the fields** (see Ticket format below).

4. **Confirm before creating.** Show the user the full drafted ticket — project,
   issue type, summary, labels, and the rendered description — and ask them to
   confirm or adjust. Creating a ticket is an outward-facing action that notifies a
   team and can't be cleanly undone, so always get the green light first. If the user
   already said something like "just create it" / "no need to confirm", skip straight
   to creation.

5. **Create it** with `mcp__rovo__createJiraIssue` (see Creating the ticket below).

6. **Return the link.** Give the user the issue key and a clickable browse URL.


## Creating the ticket

Call `mcp__rovo__createJiraIssue` with:

- `projectKey`: `SFDC` (or whatever the user specified)
- `issueTypeName`: e.g. `Story`, `Task`, `Bug`
- `summary`: the title
- `description`: the Markdown body
- `contentFormat`: `"markdown"`
- `additional_fields`: for labels, `{"labels": ["SF_M5"]}`

After it returns, surface the result like:

> Created **SFDC-1234** — [EmailToCase routing drops cases](https://company-name.atlassian.net/browse/SFDC-1234)

If the call fails (permissions, bad field, project not found), report the error
plainly and show the drafted ticket so the user's work isn't lost — don't silently
retry against a different project.

## Ticket Description Format

```markdown
## Background
Why this is needed and where it came from — the problem, the trigger, or the
request. If the input was a Slack/email thread, summarize the relevant context here
and note the source (e.g. "From #salesforce-support thread, 2026-06-18").

## Context
What needs to happen, in concrete terms. Specifics the input gave you: affected
components, objects, flows, users, expected behavior. For a Bug, include steps to
reproduce and expected vs. actual.

## Goal
Describe the end goal of the ticket.

## Task
- [ ] Bullet checklist of steps to complete the task

## Acceptance Criteria
- [ ] Bullet checklist of what "done" looks like
- [ ] One criterion per line, each independently verifiable
```

Fill every section. If the input genuinely doesn't supply something (e.g. no repro
steps), write what you can infer and flag the gap rather than leaving it blank or
inventing details — e.g. "Repro steps not provided in source thread."


## Examples

**Example 1 — freeform request**

Input: "We should add a validation rule so reps can't close an Opportunity without a
close reason. Keeps biting us in reporting."

Draft:
- Type: **Story** (new behavior/feature)
- Summary: `Require Close Reason before an Opportunity can be closed`
- Description follows the template — Background explains the reporting pain, Details
  names the Opportunity object and the close-reason field, Acceptance Criteria lists
  the validation firing on close and an error message shown to the rep.

**Example 2 — pasted Slack thread**

Input: a thread where support says cases aren't getting
assigned, an engineer confirms the routing rule skips that inbox, and they agree it
needs fixing.

Draft:
- Type: **Bug** (something is broken)
- Summary: `Case routing skips inbound email`
- The Background cites the thread; Details captures the engineer's root-cause note and
  expected vs. actual routing; Acceptance Criteria is "cases from that address route
  to the correct queue."

**Example 3 — Apex trigger exception email**

Input: an alert email "Apex Trigger Exception — TaskTrigger AfterInsert" with a
`System.NullPointerException` and a stack trace.

Draft:
- Type: **Bug**
- Summary: `NullPointerException in TaskTrigger AfterInsert handler`
- Background quotes the exception type/message and names TaskTrigger; Details gives the
  impact analysis — Task inserts are failing for affected users — and, after grepping
  the repo, points at the likely null dereference (e.g. `TaskTrigger.cls` /
  `TaskTriggerHandler.cls:NN`); Acceptance Criteria is "Task inserts no longer throw
  and the handler null-checks the offending field."

**Example 4 — Salesforce platform notification**

Input: a Salesforce email announcing that API version 21.0–30.0 will be retired and
integrations on those versions will stop working after a given date.

Draft:
- Type: **Task** (action needed, nothing broken yet)
- Summary: `Assess and remediate impact of legacy API version retirement`
- Background quotes the retirement notice and the deadline; Details lists the impact
  analysis work — find integrations/metadata still on the retiring versions and plan
  upgrades; Acceptance Criteria is "all affected components identified and upgraded to
  a supported API version before the deadline."