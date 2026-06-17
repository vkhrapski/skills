---
name: error-to-jira
description: "Analyzes error and failure emails from Salesforce and related tools, then generates a ready-to-paste Jira ticket description. Use this skill whenever the user pastes any kind of error or failure email — even if they don't explicitly ask for a Jira ticket. Triggers include Apex trigger exceptions, Flow runtime errors, EmailToCase failures, Batch job failures, Email Deliverability Alerts, and Gearset deployment or test failures. If the email contains a failure, an exception, or an alert, use this skill."
---
 
# Error Email → Jira Ticket Creator
 
Turns a pasted error or failure email into a copy-paste-ready Jira ticket description.
 
## Input
 
The user pastes raw email content. Supported email types:
 
| Sender | Subject pattern | Type |
|---|---|---|
| `ApexApplication` | "Developer script exception from [Org] : [Trigger]..." | Apex Trigger Exception |
| `FlowApplication` | "An error occurred with your [Flow Name] flow" | Flow Error |
| `System` | "Failed to process EmailToCase" | EmailToCase Failure |
| `System` / `Salesforce Platform` / human | "[BatchName] Batch process completed - Failed" or "Batch Process Completed (Job Id: ...)" | Batch Job Failure |
| `System` | "Emails Deliverability Alert" | Email Deliverability Alert |
| `Gearset` / `noreply@gearset.com` | "Tests failed on [run]" / "Deployment failed" | Gearset Failure |
 
The email may be forwarded by a human (e.g. a colleague forwarding a batch failure) — extract the original error content from the forward.
 
## Output Format
 
Create a `.md` file saved to `/mnt/user-data/outputs/` and present it using `present_files`. Name the file after the ticket title, e.g. `flow-error-check-or-wire-closed-won.md`.
 
The file content must follow this exact structure, ready to paste into Jira:
 
---
 
**[Title]**
 
Short, specific title. No brackets or prefixes. Describes what failed and where.
Examples:
- `Apex Trigger Exception — TaskTrigger AfterInsert on TripAdvisor`
- `Flow Error — Check or Wire Closed Won`
- `Batch Job Failure — B2B Price Override (Validation Error)`
- `EmailToCase Failure — Unable to Create Case from Inbound Email`
- `Email Deliverability Alert — TripAdvisor Org (00DA0000000IORf)`
**Context**
 
2–4 sentences. What happened, when, and in what environment. Reference the org name, job ID, flow name, trigger name, or run name if present. Explain why this matters (e.g. blocks a live process, affects customer data, causes case creation to fail, impacts revenue operations).
 
**Goal**
1–2 sentences. What we're trying to achieve — fix the failure, restore functionality, or understand the root cause before acting.
 
**Task**
 
Bulleted list of specific actions to investigate and resolve. Tailor to the error type (see rules below).
 
**Acceptance Criteria**
Bulleted list. Each item is a verifiable done condition:
- Failure identified and root cause documented
- Fix implemented and tested in sandbox
- Deployed to affected org without errors
- (Where applicable) Confirmed working on a real trigger/run after deployment
**Notes**
Any urgency, downstream impact, or constraints visible in the email (e.g. recurring failure, blocks deployment, affects live customer-facing process). If nothing notable, omit this section.
 
---
 
## Task Structure by Error Type
 
### Apex Trigger Exception
- Identify the trigger, DML event (AfterInsert / AfterUpdate / etc.), and the exception class/message
- Determine the record type or context that triggered the exception
- Check whether the exception is a null reference, governor limit, or data quality issue
- Reproduce in sandbox; fix the trigger logic or add a null guard
- Verify no related triggers or classes are also affected
### Flow Error
- Identify the flow name, the failing element, and the error message
- Identify the record (type, ID if available) and the action that triggered the flow
- Determine whether this is a data issue (missing required field, null value) or a logic/config issue
- Fix the flow element or add fault path handling; test in sandbox
- If recurring: consider adding a fault path email alert to catch future failures silently
### EmailToCase Failure
- Identify the routing address or email service that received the inbound email
- Identify the failure reason (null dereference, routing rule, permission, parsing issue)
- Check `EmailServiceHandler` or the relevant Apex class for the root cause
- Test with a real inbound email after fixing; confirm case is created and assigned correctly
### Batch Job Failure
- Identify the batch class name and Job ID (if present)
- Identify the failure type: Validation Error (record-level), SQL/SOQL Error (query issue), or governor limit
- If Validation Error: find the record ID in the email and determine which validation rule fired
- If SQL/SOQL Error: review the query in the batch class for the offending condition
- Re-run the batch in sandbox against representative data; confirm clean execution
- Check whether this is a recurring failure (same batch, same error pattern)
### Email Deliverability Alert
- Identify the org (by Org ID or name) and the date range of the report
- Note the bounce rate, spam rate, or specific deliverability metric that triggered the alert
- Identify the email type or template responsible (if mentioned)
- Review Salesforce Email Deliverability settings and sending domain/SPF/DKIM configuration
- Determine corrective action: list cleanup, template fix, or DNS configuration update
### Gearset Failure
- **Test failures**: identify the failing Apex class/method and assertion; check for data issue vs code regression; fix or skip with justification
- **Deployment failures**: identify the component(s) that failed and the error; resolve conflicts or dependency issues before re-deploying
- Reference the Gearset run name and target org if present in the email
---
 
## Behavior Rules
 
- **Don't invent details.** Only use what's in the email. If something is unclear (e.g. which record triggered the error), flag it in the Task section as something to investigate.
- **Extract from forwards.** If the email was forwarded by a colleague, pull the original error content — ignore the forwarding wrapper.
- **Be specific in the title.** Include the trigger name, flow name, batch name, or org name when present.
- **Write tasks as actions**, not observations. "Identify the failing assertion in OpportunityTriggerTest" not "There is a failing test".
- **Acceptance criteria must be checkable.** Avoid vague items like "issue is resolved" — specify what verification looks like.
- **Group related failures into one ticket.** If multiple errors share the same root cause, create one ticket. If they are distinct failures, create one ticket per failure and label them (Ticket 1 of N, etc.).
- **Recurring patterns matter.** If the email subject shows a count (e.g. "FlowApplication 43"), note in the Notes section that this is a recurring failure.