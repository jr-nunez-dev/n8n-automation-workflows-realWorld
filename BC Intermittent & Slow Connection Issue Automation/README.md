# BC Intermittent & Slow Connection Issue Automation

An n8n workflow that watches a shared support inbox for network-monitoring alert emails, automatically classifies them as either an **Intermittent Connection** issue or a **Slow Connection** issue, files a Jira incident ticket, and logs a structured, dashboard-ready record of the incident to Google Sheets.

---

## 1. Summary

This workflow removes the human step between "a network monitoring alert lands in an inbox" and "a Jira ticket exists for it." Two Gmail labels feed the same pipeline: one for emails already tagged as network *intermittent connection* alerts, and one for *slow connection* alerts. Each fresh (non-reply) email is turned into a Jira issue, and once the issue is created, the raw email text is parsed for the diagnosis, affected network device, location, and downtime timestamp, then appended as a row to a Google Sheet used for reporting/dashboards (see the related **BC and Server Dashboard Automation** workflow).

---

## 2. Problem It Solves

- **Manual inbox monitoring:** Without this workflow, someone on the support/network team has to keep an eye on a shared inbox for alert emails and manually decide whether they represent a new incident or a duplicate/reply.
- **Manual, inconsistent ticket creation:** Network issue tickets were being created by hand, which is slow and leads to inconsistent summaries/descriptions and missing fields (device, downtime, business impact).
- **No structured incident history:** There was no consistent, structured log of network incidents (device, diagnosis, downtime start, status) that could be rolled up into a dashboard or trend report.
- **Reply-email noise:** Reply/thread emails on an already-open ticket could otherwise trigger duplicate ticket creation if not filtered out.

## 3. Benefit to the Company

- **Faster response time** — tickets are opened within a minute of the alert email arriving (Gmail triggers poll every minute), instead of waiting for a person to notice and log it.
- **Consistency** — every ticket has the same fields populated the same way (summary, description, device, diagnosis, downtime), regardless of who or what generated the source email.
- **No missed or duplicate incidents** — the "fresh email" check ensures only the first email in a thread creates a ticket; reply/follow-up emails are safely disregarded.
- **Reporting-ready data** — every incident is automatically appended to a Google Sheet with normalized fields (Ticket ID, Date, Device Type, Network Device, Status, Business Impact, Diagnosis, Start of Downtime), which can feed dashboards, SLA reports, or trend analysis without extra manual data entry.
- **Reduced workload on IT/network staff** — frees the team from repetitive, low-value ticket-logging work so they can focus on actually resolving the incident.

## 4. Who Will Use It

- **IT/NOC (Network Operations Center) staff** — the primary Jira tickets are created for and worked by this team.
- **Support/Helpdesk team** — benefits from tickets already being logged and categorized by the time they review the queue.
- **IT Managers/Team Leads** — consume the Google Sheet log for status visibility, historical trends, and reporting (e.g., feeding it into the BC and Server Dashboard).
- No one interacts with this workflow directly during normal operation — it runs unattended, triggered purely by incoming email. Its "users" are effectively the downstream Jira board and the Google Sheet dashboard, and by extension the people who read those.

---

## 5. High-Level Flow

```
Gmail Trigger (intermittent label)  ──┐
Gmail Trigger (slow connection label)─┼──> intermittent? (classify by subject)
                                       │        ├── true  ──> freshIntermittentEmail?
                                       │        └── false ──> freshSlowConnectionEmail?
                                       │
                          freshIntermittentEmail? ── true ──> editDataForIntermittent ──> createIssueIntermittent ─┐
                                       └── false ──> disregardTicket                                               │
                                                                                                                    ├──> getData ──> parseData ──> enrichParseData ──> logData (Google Sheets)
                          freshSlowConnectionEmail? ── true ──> editDataForSlowConnection ──> createSlowConnectionIssue ─┘
                                       └── false ──> disregardTicket
```

The workflow has two mirrored "front-end" branches (one per issue type) that both converge on a **shared back-end sub-flow** which parses the ticket and logs it to Sheets.

---

## 6. Node-by-Node Breakdown

### A. Trigger Layer

| Node | Type | Function |
|---|---|---|
| **intermittent** | Gmail Trigger | Polls Gmail every minute for new emails carrying the "Intermittent" Gmail label. This label is presumably applied by an upstream filter/monitoring tool for intermittent network connectivity alerts. |
| **slowConnection** | Gmail Trigger | Polls Gmail every minute for new emails carrying the "Slow Connection" Gmail label, for slow-connection network alerts. |

Both triggers feed into the same next node, so from this point on the workflow treats emails from either source identically until it needs to re-classify them.

### B. Classification & Routing Sub-flow

| Node | Type | Function |
|---|---|---|
| **intermittent?** | IF | Regex-checks the email `subject` (case-insensitive) for the word "intermittent." **True** → routes to the Intermittent branch (`freshIntermittentEmail?`). **False** → routes to the Slow Connection branch (`freshSlowConnectionEmail?`). This re-classifies by subject content rather than trusting the Gmail label alone, so the two labels/triggers still funnel through one consistent classification rule. |
| **freshIntermittentEmail?** | IF | Checks whether the email's `in-reply-to` header is missing/does not equal the thread ID — i.e., whether this is the *first* email in the thread rather than a reply. **True** (fresh) → `editDataForIntermittent`. **False** (a reply) → `disregardTicket`. |
| **freshSlowConnectionEmail?** | IF | Same "is this a fresh, non-reply email" check for the Slow Connection branch. **True** → `editDataForSlowConnection`. **False** → `disregardTicket`. |
| **disregardTicket** | No-Op | Dead-end for reply/thread emails. No ticket is created and no further action is taken — this is what prevents duplicate tickets from follow-up emails on an already-open incident. |

### C. Intermittent Issue Sub-flow

| Node | Type | Function |
|---|---|---|
| **editDataForIntermittent** | Set | Maps the raw email into two clean fields: `subject` and `description` (from the email body text). |
| **createIssueIntermittent** | Jira – Create Issue | Creates a new Jira issue using `subject` as the issue summary and `description` as the issue description. |

### D. Slow Connection Issue Sub-flow

| Node | Type | Function |
|---|---|---|
| **editDataForSlowConnection** | Set | Maps the raw email into `Summary` and `Description` fields (mirrors `editDataForIntermittent`, just for the Slow Connection branch). |
| **createSlowConnectionIssue** | Jira – Create Issue | Creates a new Jira issue using `Summary`/`Description`. |

Both issue-creation nodes feed into the same shared node next (`getData`), which is where the two branches reconverge.

### E. Shared Post-Processing Sub-flow (Ticket Enrichment & Logging)

This is the "back office" sub-workflow that both branches share — it takes whichever Jira ticket was just created (from either branch) and turns it into a structured log row.

| Node | Type | Function |
|---|---|---|
| **getData** | Set | Pulls the *original raw* email subject/description back in (via `$('intermittent?')`, referencing the pre-classification node) into `Subject`/`Description` fields, while also passing through all other fields already on the item — critically, this preserves the Jira API response from whichever `createIssue…` node just ran (including the new ticket's `key`, e.g. `PROJ-123`). |
| **parseData** | Code (JavaScript) | Parses the raw `Subject`/`Description` text with regex to extract structured data: <br>• `summary` (subject with "Subject: " prefix stripped) <br>• `dateTime` (a date/time pattern found in the body, e.g. `7/20/2026 8:15:42 PM`) <br>• `diagnosis` (matches "Slow Connection" or "Intermittent" in the summary) <br>• `location` and `networkDevice` (split out of pipe-delimited (`\|`) summary segments, using parentheses like `(CICTBC_IMU_AS01)` to identify the device segment) <br>• `date` (date-only portion of `dateTime`) |
| **enrichParseData** | Set | Assembles the final log record: `Ticket ID` (the Jira key from `getData`), `Date`, `Device Type` (hardcoded `"Network"`), `Network Device`, `Description` (the parsed `summary`), `Status` (hardcoded `"Open"`), `Affected IT System` (hardcoded `"NETWORK"`), `Business Impact` (hardcoded `"UNPLANNED & BUSINESS AFFECTING"`), `Diagnosis`, and `Start of Downtime` (the parsed `dateTime`). |
| **logData** | Google Sheets – Append | Appends the enriched record as a new row to a Google Sheet, creating a running, structured incident log that can power dashboards/reports. |

---

## 7. Key Design Notes

- **Dual trigger, single classifier:** Even though there are two separate Gmail triggers (one per label), the workflow re-derives the issue type from the subject text via `intermittent?` rather than trusting the trigger source directly. This makes the classification logic resilient to a mislabeled email landing in the wrong inbox/label.
- **Reply suppression:** Both branches independently check for "is this the first email in the thread," which is what stops reply/notification-chain emails from spawning duplicate Jira tickets.
- **Reconvergence for shared logging:** Rather than duplicating the parse/log logic per branch, both Jira-creation paths route into one shared `getData → parseData → enrichParseData → logData` tail, keeping the Sheet-logging logic in one place.
- **Hardcoded fields:** `Device Type`, `Status`, and `Affected IT System`/`Business Impact` are currently hardcoded (`Network` / `Open` / `NETWORK` / `UNPLANNED & BUSINESS AFFECTING`) since this workflow is scoped specifically to network incidents — these would need to become dynamic if the workflow were generalized to other issue types.
- **Credentials used:** Gmail OAuth2 (reading labeled emails), Jira Cloud API (creating issues), Google Sheets OAuth2 (appending the log row).

---

## 8. Related Workflows

- **BC and Server Dashboard Automation** — likely the consumer of the Google Sheet this workflow writes to, rolling the logged incidents into a live dashboard and periodic summaries.
- **Business Center Logs Automation** — a related workflow that reacts to Jira ticket updates for Business Center downtime incidents.
