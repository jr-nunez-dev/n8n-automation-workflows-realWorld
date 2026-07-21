# PRTG Critical Alert Automation Logs

An n8n workflow that listens to Jira for PRTG server-monitoring incident tickets, diagnoses what kind of issue was flagged (CPU, memory, disk, or HTTP downtime), calculates the outage duration, and logs it into a structured Google Sheets record.

---

## Summary

This workflow is triggered directly by Jira whenever a "[System] Incident" ticket from a specific reporter is created or updated. It fetches the full ticket (including its comment history), then runs it through a diagnosis engine that keyword-matches the ticket summary to classify the alert as High CPU Utilization, High Memory Utilization, Disk Utilization, or HTTP Downtime — silently dropping any ticket that doesn't match one of those categories, since this workflow is scoped to PRTG server alerts specifically. For matching tickets, it extracts the affected server/device, the PRTG sensor ID, the downtime start time, and scans the ticket's comments for a "restored" timestamp left by a human (explicitly ignoring automation-authored comments) to calculate total downtime. The result is written as a row into the correct month's "PRTG Critical Alerts" tab of the shared NOC log spreadsheet, updating the row in place if the same ticket is seen again.

---

## What problem does it solve?

PRTG (server/infrastructure monitoring) alerts flow into Jira as incident tickets, but the useful data — what kind of resource issue it was, which server, when it started, and how long it lasted — is buried in free-text summaries, descriptions, and comment threads rather than structured fields. Without automation:

- Someone has to manually read each ticket and classify what type of alert it was (CPU/memory/disk/HTTP)
- Downtime duration has to be manually calculated from a start timestamp and whenever a technician noted the issue was resolved in a comment
- The "[System] Incident" ticket type isn't exclusively used for server alerts, so unrelated tickets would need to be manually filtered out of any reporting
- Ticket updates (e.g., a restoration comment added later) require someone to go back and update the previously logged entry

This workflow automates the classification, extraction, and duration calculation end-to-end, and keeps the log entry updated as the ticket evolves.

---

## Benefit to the company

- **Automatic incident classification** — every relevant ticket is automatically tagged as a CPU, memory, disk, or HTTP downtime issue, useful for spotting patterns (e.g., recurring memory issues on a specific server).
- **Accurate downtime tracking** — total downtime is calculated automatically from the outage start time and the first genuinely human "restored" comment, removing manual math and the inconsistency of different people calculating it differently.
- **Noise filtering** — tickets that don't match a known PRTG alert pattern are automatically dropped, so the log stays focused on genuine server-monitoring incidents rather than every "[System] Incident" ticket.
- **Anti-automation-comment safeguard** — the restoration-time scan explicitly skips comments made by automation accounts, avoiding false "restored" timestamps being picked up from bot-generated comments.
- **Feeds the broader NOC reporting pipeline** — this log is the same "PRTG Critical Alerts" data source consumed by the [[bc-and-server-dashboard-automation|BC and Server Dashboard Automation]] workflow's server-side metrics, so keeping it accurate here makes every downstream dashboard and Chat report accurate too.
- **Self-updating** — a ticket that gets updated (e.g., a later restoration comment) automatically refreshes its existing row instead of creating a duplicate.

---

## Who will use it?

- **NOC / IT Support team monitoring server infrastructure** — as the primary consumers of the "PRTG Critical Alerts" log for tracking server incident history and downtime.
- **Whoever works PRTG-triggered "[System] Incident" tickets in Jira** — their ticket updates and restoration comments automatically flow into the structured log without extra effort.
- **Anyone relying on the NOC dashboard/reporting workflows** — since this log directly feeds the server-side numbers used in the [[bc-and-server-dashboard-automation]] dashboard and its Chat summaries.

This runs unattended, triggered directly by Jira webhook events, as part of the NOC automation suite (tagged `nocAutomations`).

---

## How the flow works (high-level)

```
Jira Trigger (issue created/updated, specific reporter, "[System] Incident", this month)
        │
        ▼
Get an issue  (fetch full ticket details, including comments)
        │
        ▼
Code in JavaScript
   ├─ classify diagnosis from summary keywords (CPU / Memory / Disk / HTTP)
   │      → no match? drop the ticket entirely, nothing further happens
   ├─ extract server/device name, PRTG sensor ID, downtime start time
   ├─ scan comments (skipping automation authors) for a "restored" timestamp
   ├─ calculate total downtime duration
   └─ build the target sheet name "PRTG Critical Alerts {Month Year}"
        │
        ▼
Edit Fields  (map extracted data to sheet columns, add fixed fields)
        │
        ▼
Append or update row in sheet
(writes into "PRTG Critical Alerts {Month Year}" tab, matched by Ticket ID)
```

---

## Node-by-node breakdown

### 1. `Jira Trigger`
- **Type:** Jira Trigger, listening for `jira:issue_created` and `jira:issue_updated` events.
- **Filter (JQL):** project `CICT`, from a specific reporter account, issue type `[System] Incident`, created since the start of the current month.
- **Role:** The single entry point — every matching create or update starts a run.

### 2. `Get an issue` — Jira node
- **Function:** Fetches the full ticket by its key (from either the trigger's raw or nested payload shape), pulling complete ticket data including its full comment history.
- **Role:** The Jira Trigger's webhook payload alone may not include everything needed (particularly comments), so this node retrieves the complete, current state of the ticket for parsing.

### 3. `Code in JavaScript`
- **Function:** The core diagnosis and extraction engine. For the fetched ticket it:
  - **Diagnosis classification:** keyword-matches the summary (case-insensitive) against known PRTG alert patterns — "meminfo"/"physical memory" → *High Memory Utilization*; "disk"/"disks" → *Disk Utilization*; "cpu"/"cpu load" → *High CPU Utilization*; "http"/"https"/"web api"/"api" → *HTTP Downtime*.
  - **Critical filter:** if none of those keywords match, the function returns an empty result, dropping the ticket entirely — this workflow only logs tickets it can positively classify as a PRTG server alert.
  - **Server/device extraction:** regex-matches a "The `<device>` usage..." pattern in the description; if that fails, falls back to the first line of the description (or a generic "Web Service/API"/"HTTP Endpoint" label for long or empty first lines).
  - **Sensor ID extraction:** regex-matches a PRTG sensor URL pattern (`sensor.htm?id=<number>`) out of the description.
  - **Downtime start extraction:** regex-matches an "as of `<timestamp>`" pattern in the description.
  - **"Anti-bot" restoration scan:** walks the ticket's comments from newest to oldest, skipping any comment authored by an account with "automation" in its display name, and looks for the first comment (i.e., most recent human comment) containing a date/time pattern — treating that as the restoration timestamp.
  - **Flexible date parsing:** a custom parser (`parseAnything`) handles `M/D/YYYY H:MM(:SS) AM/PM`-style strings robustly, including missing seconds or AM/PM markers.
  - **Downtime duration:** calculates the difference between the extracted start time and the restoration time, formatted as hours/minutes.
  - **Target sheet name:** builds the dynamic tab name `PRTG Critical Alerts {Month} {Year}` from the alert's date (or the current date, as a fallback).
  - Outputs one structured record per qualifying ticket.
- **Role:** Turns a raw, unstructured Jira ticket into a fully classified, dated, and duration-calculated incident record — or filters it out entirely if it's not a recognized PRTG alert type.

### 4. `Edit Fields` — Set node
- **Function:** Maps the parsed data into the exact column set used by the log sheet: `Ticket ID`, `Server Device`, `Description` (from the summary), `Date`, `Status`, `Start of Downtime`, `Sensor`, `Assignee`, `Diagnosis`, `Time Restored`, `Total Downtime`, and `Monitored By` (same as assignee) — plus fixed/hardcoded values `Affected IT Domain` = "Server and storage," `Affected IT System` = "Server and storage," `Business Impact` = "UNPLANNED & NON-BUSINESS AFFECTING," and `Device Type` = "Server."
- **Role:** Final shaping step before the row is written to the spreadsheet.

### 5. `Append or update row in sheet` — Google Sheets node
- **Function:** Writes the row into the dynamically-named `PRTG Critical Alerts {Month Year}` tab of the NOC Monitoring Automation Logs spreadsheet, matching on the `Ticket ID` column — so a new ticket becomes a new row, and an update to an existing ticket (e.g., a later restoration comment) updates that same row instead of duplicating it.
- **Role:** The end goal of the workflow — this is what keeps the PRTG incident log current.

---

## Sub-workflows

No `Execute Workflow` nodes are used — this is a single, self-contained, linear workflow (no branching): every ticket either gets fully processed and logged, or is silently dropped inside the `Code in JavaScript` node if it doesn't match a recognized PRTG alert pattern.

---

## Configuration reference

| Item | Value |
|---|---|
| Trigger | Jira Trigger — `jira:issue_created`, `jira:issue_updated` |
| JQL filter | Project `CICT`, specific reporter, issue type `[System] Incident`, created since start of current month |
| Diagnosis keywords | Memory: "meminfo," "physical memory" · Disk: "disk"/"disks" · CPU: "cpu"/"cpu load" · HTTP: "http," "https," "web api," "api" |
| Unmatched tickets | Dropped entirely — no row is written |
| Restoration-comment filter | Skips any comment whose author display name contains "automation" |
| Target spreadsheet | NOC Monitoring Automation Logs |
| Target sheet tab | `PRTG Critical Alerts {Month Year}` (dynamic, based on the alert date) |
| Match/dedupe column | `Ticket ID` |
| Hardcoded fields | Affected IT Domain/System = "Server and storage," Business Impact = "UNPLANNED & NON-BUSINESS AFFECTING," Device Type = "Server" |
| n8n tag | `nocAutomations` |

---

## Notes / things worth knowing

- This workflow is what populates the `PRTG Critical Alerts {Month Year}` tab that [[bc-and-server-dashboard-automation|BC and Server Dashboard Automation]]'s `Server Devices` node reads from — the earlier note about that node's naming turned out to check out: "Server Devices" reading the "PRTG Critical Alerts" sheet is correct, since this workflow always logs `Device Type = "Server"` into that sheet.
- The diagnosis classification and the restoration-comment scan are both keyword/pattern-based; a ticket summary or comment phrased unusually (e.g., not containing any of the recognized keywords, or a restoration note that doesn't include a full date/time) could fail to be classified or timed correctly.
- Because unmatched tickets are dropped with no output and no error, there's no visible record of *why* a given "[System] Incident" ticket wasn't logged here — worth keeping in mind if a PRTG-related ticket ever seems to be missing from the log.
