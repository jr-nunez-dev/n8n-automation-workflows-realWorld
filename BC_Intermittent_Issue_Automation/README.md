# BC Intermittent Issue Automation

An n8n workflow that watches two support inboxes for emails reporting an "intermittent" issue and automatically opens a Jira incident ticket — no manual triage required.

---

## Summary

This workflow monitors two Gmail accounts (labeled inboxes) for incoming emails. When a new, non-reply email arrives with the word **"intermittent"** in the subject line, the workflow pulls the subject and body, and creates a Jira ticket in the **IT Support** project under the **"[Incident] System Downtime"** issue type. Emails that don't mention "intermittent," or that are replies within an existing thread, are ignored — preventing noise and duplicate tickets.

---

## What problem does it solve?

Intermittent issues (flaky connections, sporadic downtime, systems that "sometimes" fail) are commonly reported over email rather than through a formal ticketing channel, especially by non-technical staff or external partners. Left in an inbox, these reports:

- Get missed or buried, especially outside business hours
- Depend on someone manually reading, triaging, and re-typing the report into Jira
- Introduce delay between "issue reported" and "issue tracked," which matters for intermittent problems that are hard to reproduce later

This workflow removes the manual hand-off between "email arrives" and "ticket exists," so intermittent issues are captured and tracked the moment they're reported.

---

## Benefit to the company

- **Faster response time** — tickets are created within a minute of the email arriving (the trigger polls every minute), instead of waiting for someone to check the inbox.
- **No dropped reports** — nothing sent to the monitored label is missed, even overnight or on weekends.
- **Reduced manual work** — support staff no longer need to copy/paste email content into Jira by hand.
- **Consistent ticket data** — every auto-created ticket follows the same project, issue type, summary, and description format, making reporting and trend analysis on intermittent issues easier.
- **No duplicate tickets** — reply emails on an existing thread are filtered out, so a ticket isn't re-created every time someone replies to the same report.
- **Audit trail** — every intermittent-issue email becomes a permanent Jira record instead of living only in someone's inbox.

---

## Who will use it?

- **IT Support / NOC team** — the ticket lands directly in their Jira queue (IT Support project) for triage and resolution, without them needing to monitor the inbox.
- **End users / other teams (e.g., "johnrex" and "jam" inboxes)** — whoever sends or forwards issue reports to either monitored, labeled inbox benefits indirectly: their report is guaranteed to become a tracked ticket.
- **Team leads / managers** — benefit from having intermittent issues consistently logged in Jira, useful for tracking recurring problems over time.

This is intended to run unattended in the background as part of the NOC automation suite (tagged `nocAutomations` in n8n).

---

## How the flow works (high-level)

```
[Gmail: johnrex] ─┐
                   ├──► intermittent? ──Yes──► fresh email? ──Yes──► editData ──► createIssue (Jira)
[Gmail: jam]     ──┘         │                      │
                              No                     No (it's a reply)
                              │                      │
                              ▼                      ▼
                        notIntermittent           reply
                          (do nothing)          (do nothing)
```

Two Gmail inboxes feed into a single decision path. An email must pass **two checks** — "does the subject mention 'intermittent'?" and "is this a fresh email, not a reply?" — before a Jira ticket is created. Failing either check ends the run with no action taken.

There are no sub-workflows in this automation — it is a single, self-contained flow with two entry points (triggers) that converge into one shared processing path.

---

## Node-by-node breakdown

### 1. `johnrex` — Gmail Trigger
- **Type:** Gmail Trigger, polling every minute
- **Function:** Watches one Gmail inbox (credential: *JohnrexOAuth2 API*) for new mail carrying the label `Label_8663571737587485244`. Only the single most recent matching email is fetched per poll (`maxResults: 1`).
- **Role in flow:** One of two entry points into the workflow.

### 2. `jam` — Gmail Trigger
- **Type:** Gmail Trigger, polling every minute
- **Function:** Watches a second Gmail inbox (credential: *Jam-OAuth*) for new mail carrying the label `Label_2272028879666386065`. Same polling behavior as `johnrex`.
- **Role in flow:** The second entry point; both triggers feed into the same downstream logic.

### 3. `intermittent?` — IF node
- **Function:** Checks whether the email's `subject` field matches the regex `intermittent` (case-insensitive).
- **True branch:** subject mentions "intermittent" → continues to `fresh email?`
- **False branch:** subject does not mention "intermittent" → routes to `notIntermittent` and the run ends.
- **Purpose:** Filters out all emails that aren't reporting an intermittent issue, so only relevant reports proceed toward ticket creation.

### 4. `fresh email?` — IF node
- **Function:** Checks whether the email is a new/original message rather than a reply, by testing that the `in-reply-to` header does **not** exist (compared against `threadId`).
- **True branch:** no `in-reply-to` header → this is a fresh report → continues to `editData`.
- **False branch:** `in-reply-to` header exists → this is a reply on an existing thread → routes to `reply` and the run ends.
- **Purpose:** Prevents duplicate Jira tickets from being created every time someone replies within the same email thread (e.g., a back-and-forth conversation about the same issue).

### 5. `editData` — Set node
- **Function:** Maps the raw email fields into two clean, explicitly-named fields for use downstream:
  - `subject` ← `$json.subject`
  - `description` ← `$json.text` (the email body)
- **Purpose:** Normalizes the data going into Jira so the next node has predictable field names, regardless of which Gmail trigger (`johnrex` or `jam`) the email came from.

### 6. `createIssue` — Jira node
- **Function:** Creates a new Jira issue using:
  - **Project:** IT Support (id `10151`)
  - **Issue type:** [Incident] System Downtime (id `10629`)
  - **Summary:** the email subject
  - **Description:** the email body
  - **Credential:** Jira SW Cloud account Sandbox
- **Purpose:** This is the end goal of the workflow — the point where an inbox report officially becomes a tracked incident ticket.

### 7. `notIntermittent` — No-Op node
- **Function:** Does nothing; simply terminates the branch.
- **Purpose:** Acts as a labeled dead-end for emails that don't mention "intermittent," making the workflow's branching intent readable at a glance in the n8n canvas.

### 8. `reply` — No-Op node
- **Function:** Does nothing; simply terminates the branch.
- **Purpose:** Labeled dead-end for reply emails on an already-reported thread, so no duplicate ticket is created.

---

## Sub-workflows

This automation does **not** call any external sub-workflows (no `Execute Workflow` nodes are used). All logic — trigger, filtering, data shaping, and ticket creation — is contained within this single workflow.

---

## Configuration reference

| Item | Value |
|---|---|
| Trigger poll interval | Every minute (both Gmail triggers) |
| Gmail label — johnrex inbox | `Label_8663571737587485244` |
| Gmail label — jam inbox | `Label_2272028879666386065` |
| Subject filter | Regex match on "intermittent" (case-insensitive) |
| Jira project | IT Support (`10151`) |
| Jira issue type | [Incident] System Downtime (`10629`) |
| n8n tag | `nocAutomations` |

---

## Notes / possible future improvements

- Currently only the **latest** email per poll is processed (`maxResults: 1`); a burst of multiple intermittent-issue emails within the same minute on the same inbox could result in only the most recent one being picked up on that poll cycle.
- The `notIntermittent` and `reply` branches are no-ops; they could be extended later (e.g., auto-reply to the sender, or logging for visibility) without changing the core ticket-creation logic.
