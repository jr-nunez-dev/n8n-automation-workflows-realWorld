# Business Center Logs Automation

An n8n workflow that listens to Jira for Business Center network-downtime incidents and automatically keeps a Google Sheets log of them up to date — logging new/updated incidents, and cleaning up the log when a ticket turns out to be a false alarm (cancelled).

---

## Summary

This workflow is triggered directly by Jira whenever a "[Incident] System Downtime" ticket for the Business Center classification is created or updated. It parses the ticket (subject, description, custom fields, and change history) to pull out the affected location, region-based support group, downtime start/end times, and total downtime duration, then writes that as a row into the correct month's "Business Centers" tab of the shared NOC log spreadsheet — creating the row if it's new, or updating it in place if the ticket already exists there. If the ticket is genuinely cancelled, the workflow instead finds and deletes any row that was already logged for it, so cancelled incidents don't linger in the log.

---

## What problem does it solve?

Business Center network downtime incidents are tracked in Jira, but a centralized spreadsheet log (used for reporting and dashboards — see the related [[bc-and-server-dashboard-automation|BC and Server Dashboard Automation]] workflow) needs the same data in a structured, analyzable format. Manually copying every incident from Jira into a spreadsheet, keeping it updated as the ticket status changes, and remembering to remove entries for tickets that get cancelled is:

- Slow and easy to forget, especially for tickets updated multiple times
- Error-prone — manually transcribed timestamps, durations, and locations are easy to get wrong
- Not real-time — the log falls out of sync with Jira until someone manually reconciles it
- A source of stale/incorrect data if cancelled tickets are never removed from the log

This workflow keeps the spreadsheet log and Jira in sync automatically, in real time, including cleanup of false alarms.

---

## Benefit to the company

- **Real-time, always-accurate log** — the moment a ticket is created or updated in Jira, the spreadsheet reflects it; no manual data entry.
- **Automatic downtime calculation** — total downtime duration is computed automatically (in Philippine time) from the extracted start time and the ticket's resolution field, removing manual math and the errors that come with it.
- **Clean data for reporting** — this log directly feeds downstream reporting/dashboard workflows, so keeping it accurate here means every report built on top of it is accurate too.
- **Self-cleaning** — cancelled tickets are automatically detected and their rows removed, so the log doesn't accumulate false incidents that would skew ticket counts and downtime statistics.
- **Consistent structure** — every incident is logged with the same set of fields (support group, affected domain/system, business impact, etc.), regardless of who created or updated the ticket, making the log reliable to build reports on.
- **Regional routing insight** — automatically tags each incident with the correct regional support group (Central/North/South/NCR/Visayas-Mindanao) based on the location codes in the ticket, useful for regional workload tracking.

---

## Who will use it?

- **NOC / IT Support team** — as the ultimate consumers of the accurate Business Center downtime log, which feeds dashboards and reports (see [[bc-and-server-dashboard-automation]]).
- **Whoever creates/updates the Jira tickets** (e.g., the reporter monitoring Business Center connectivity) — their ticket updates automatically propagate into the tracking log without any extra action on their part.
- **Regional support teams** — benefit from tickets being automatically tagged to the correct regional support group.
- **Anyone using the "NOC Monitoring Automation Logs" spreadsheet for reporting** — relies on this workflow keeping the "Business Centers" tab accurate.

This runs unattended, triggered directly by Jira webhook events, as part of the NOC automation suite (tagged `nocAutomations`).

---

## How the flow works (high-level)

```
Jira Trigger (issue created/updated, Business Center incidents only)
        │
        ▼
Code in JavaScript  (parse ticket: location, support group, timestamps, downtime, cancellation check)
        │
        ▼
     Switch
   ┌────┴─────────────────────┬───────────────────────────┐
   │ isTrueCancellation        │ status == "Cancelled"      │ (fallback) not cancelled
   ▼                           ▼                            ▼
  Limit                Prevents Cancelled to Append     Edit Fields
   │                       (do nothing)                     │
   ▼                                                         ▼
Get row(s) in sheet                                Append or update row in sheet
   │                                                  (writes into Business Centers
   ▼                                                   {Month Year} tab)
   If (row found?)
   ├─Yes─► Delete rows or columns from sheet
   └─No──► No Output Fallback (do nothing)
```

There are three outcomes per Jira event: a genuinely cancelled ticket has its logged row deleted (if one exists); a ticket whose plain status is "Cancelled" but that didn't trip the change-history check is safely ignored (belt-and-suspenders guard); and every other ticket is parsed and appended/updated in the log.

---

## Node-by-node breakdown

### 1. `Jira Trigger`
- **Type:** Jira Trigger, listening for `jira:issue_created` and `jira:issue_updated` events.
- **Filter (JQL):** only fires for issues in project `CICT`, of type `[Incident] System Downtime`, classified under "Business Center," and created by a specific reporter account.
- **Role:** The single entry point — every create or update to a matching Jira ticket starts a new run.

### 2. `Code in JavaScript`
- **Function:** The main parsing/extraction engine. For the incoming Jira issue payload it:
  - **Cancellation defense:** inspects the ticket's changelog for a status change *to* "CANCELLED," and also checks the ticket's current status directly, setting an `isTrueCancellation` flag if either is true (guards against race conditions between the changelog and current state).
  - **Location extraction:** regex-matches a region code + location pattern (`SLZ`, `NCR`, `VIZMIN`, `CNLZ`, `NLZ`) out of the ticket description.
  - **Downtime start extraction:** regex-matches a "...is currently Down as of [date/time]" pattern out of the description to get the outage start timestamp.
  - **Support group mapping:** maps the region code found in the summary/description to a named regional support group (e.g., `CNLZ` → "IT Technical Central," `NCR` → "IT Technical NCR").
  - **Target sheet name:** builds the dynamic sheet tab name `Business Centers {Month} {Year}` based on the ticket's creation date, so incidents always land in the correct month's tab.
  - **Downtime duration:** using the extracted start time and a Jira custom field for the resolution/end time, calculates total downtime in hours/minutes, normalized to Asia/Manila time.
  - Outputs one clean object per ticket with all of the above plus ticket key, summary, assignee, and status.
- **Role:** Turns a raw, messy Jira payload into structured data the rest of the workflow can act on.

### 3. `Switch`
- **Function:** Three-way router based on the parsed data:
  1. **`isTrueCancellation` is true** → route to `Limit` (begin the "find and delete the logged row" path).
  2. **`status` field literally equals "Cancelled"** (fallback safety check) → route to `Prevents Cancelled to Append` (a dead end — do nothing).
  3. **Fallback / none of the above (not cancelled)** → route to `Edit Fields` (begin the "log this incident" path).
- **Role:** Decides whether this event should clean up an existing log row, be safely ignored, or be logged/updated.

### 4. `Limit`
- **Function:** Restricts processing to a single item, ensuring the downstream row lookup acts on exactly the ticket being processed.
- **Role:** Feeds into the row-lookup step of the deletion path.

### 5. `Get row(s) in sheet`
- **Function:** Looks up rows in the current month's "Business Centers" tab where the `Ticket ID` column matches this ticket's key.
- **Role:** Determines whether this now-cancelled ticket had already been logged.

### 6. `If`
- **Function:** Checks whether a matching row was actually found (`row_number` is not empty).
- **True:** proceeds to `Delete rows or columns from sheet`.
- **False:** routes to `No Output Fallback` — nothing to delete since the ticket was never logged in the first place.

### 7. `Delete rows or columns from sheet`
- **Function:** Deletes the previously logged row for this ticket from the "Business Centers" tab, using the row number found by `Get row(s) in sheet`.
- **Role:** Keeps the log free of incidents that turned out to be false alarms.

### 8. `No Output Fallback`
- **Function:** No-op; dead end when a cancelled ticket had no existing row to remove.

### 9. `Prevents Cancelled to Append`
- **Function:** No-op; dead end for the redundant "status literally equals Cancelled" safety branch, ensuring these never accidentally get appended to the log.

### 10. `Edit Fields`
- **Function:** Maps the parsed data into the exact column set used by the log sheet: `Ticket ID`, `Date`, `Device Type` (hardcoded "Network"), `Network Device` (extracted location), `Description` (summary), `Status`, `Support Group`, `Assignee`, `Start of Downtime` (from a Jira custom field, falling back to the regex-extracted timestamp), `Affected IT Domain` / `Affected IT System` (both hardcoded "NETWORK"), `Business Impact` (hardcoded "UNPLANNED & BUSINESS AFFECTING"), `Time Restored`, and `Total Downtime`.
- **Role:** Final shaping step before the row is written to the spreadsheet.

### 11. `Append or update row in sheet`
- **Function:** Writes the row into the dynamically-named `Business Centers {Month Year}` tab, matching on the `Ticket ID` column — so a new ticket gets a new row, and an update to an existing ticket updates that same row in place instead of duplicating it.
- **Role:** The end goal of the "log this incident" path — this is what keeps the shared NOC log spreadsheet current.

---

## Sub-workflows

No `Execute Workflow` nodes are used — this is a single, self-contained workflow. It does branch into two independent logical paths after the `Switch` node: a **cleanup path** (find and delete a row for a truly cancelled ticket) and a **log path** (shape and append/update a row for every other ticket).

---

## Configuration reference

| Item | Value |
|---|---|
| Trigger | Jira Trigger — `jira:issue_created`, `jira:issue_updated` |
| JQL filter | Project `CICT`, issue type `[Incident] System Downtime`, classification "Business Center," specific reporter |
| Target spreadsheet | NOC Monitoring Automation Logs |
| Target sheet tab | `Business Centers {Month Year}` (dynamic, based on ticket creation date) |
| Match/dedupe column | `Ticket ID` |
| Location codes recognized | SLZ, NCR, VIZMIN, CNLZ, NLZ |
| Support group mapping | CNLZ → IT Technical Central · NLZ → IT Technical North · SLZ → IT Technical South · NCR → IT Technical NCR · VIZMIN → IT Technical Visayas/Mindanao |
| Hardcoded fields | Device Type = "Network," Affected IT Domain/System = "NETWORK," Business Impact = "UNPLANNED & BUSINESS AFFECTING" |
| Downtime timezone | Asia/Manila |
| Workflow timezone | Asia/Taipei |
| n8n tag | `nocAutomations` |

---

## Notes / things worth knowing

- This workflow is what actually populates the `Business Centers {Month Year}` tab that [[bc-and-server-dashboard-automation|BC and Server Dashboard Automation]] later reads from — the two are directly linked in the overall NOC pipeline.
- The cancellation check is deliberately double-guarded (changelog-based `isTrueCancellation` flag, plus a separate literal status check) to avoid a race condition where a ticket's status has already changed by the time this workflow reads it.
- `Device Type`, `Affected IT Domain`, `Affected IT System`, and `Business Impact` are all hardcoded to Network-related values — this workflow assumes every ticket it processes is a Business Center network incident, consistent with its JQL filter.
