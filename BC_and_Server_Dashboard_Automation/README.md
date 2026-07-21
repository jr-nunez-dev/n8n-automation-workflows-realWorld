# BC and Server Dashboard Automation

An n8n workflow that turns raw ticket logs (Business Center / network devices and server / PRTG devices) into a live status dashboard, and automatically posts daily, weekly, and monthly summary reports to Google Chat.

---

## Summary

On a schedule, this workflow pulls ticket rows from two Google Sheets logs (one for Business Center / network devices, one for PRTG / server devices), merges them, and runs the combined data through a calculation engine that produces daily, weekly, monthly, and all-time totals (tickets monitored, resolved, and in progress, broken down by device type). Those numbers are written back into a "Status" tab in the same spreadsheet, turning it into a live NOC dashboard. The workflow then always sends a **daily** summary card to a Google Chat space, and additionally sends a **weekly** summary if it happens to run on a Saturday, and a **monthly** summary if it happens to run on the last day of the month.

---

## What problem does it solve?

NOC/IT teams typically log every server, network, and Business Center ticket in a spreadsheet as they work, but that raw log doesn't tell anyone, at a glance:

- How many issues are open right now vs. resolved
- How today, this week, or this month compares to normal
- What the split is between server-related and network-related issues

Without automation, someone has to manually re-count rows, build pivot tables, and separately message the team with a status update — a repetitive task that's easy to forget or get wrong, especially across daily/weekly/monthly cadences at once.

This workflow removes that manual reporting step entirely: the dashboard sheet updates itself, and the team update posts itself, on a defined schedule.

---

## Benefit to the company

- **Always-current dashboard** — the "Status" sheet reflects live daily/weekly/monthly/global numbers without anyone touching it.
- **Automatic stakeholder updates** — a Google Chat card is posted daily, and expanded automatically on Saturdays (weekly) and month-end (monthly), so leadership and the team see progress without having to ask.
- **Consistent, unbiased metrics** — the same counting rules (e.g., excluding cancelled tickets, per-device-type status mapping) are applied every time, removing manual counting errors and inconsistency between reporters.
- **Time savings** — eliminates the recurring manual work of tallying tickets and writing status update messages.
- **Better visibility into workload split** — separates server vs. network ticket volume and resolution status, which helps with resourcing and identifying which area needs more attention.
- **Historical consistency** — because it always reads the "current month" sheet tabs (named dynamically by month/year), the same logic works month after month without reconfiguration.

---

## Who will use it?

- **NOC / IT Support team** — as the source of truth dashboard for their daily workload (server + network + Business Center tickets).
- **Team leads / managers** — as the audience of the automatic Google Chat updates, to track team performance without pulling reports manually.
- **Anyone with access to the Google Chat space or the "NOC Monitoring Automation Logs" spreadsheet** — gets passive visibility into infrastructure health and ticket trends.

This runs unattended in the background as part of the NOC automation suite (tagged `nocAutomations` in n8n).

---

## How the flow works (high-level)

```
                         ┌──► Server Devices  (reads PRTG Critical Alerts sheet) ──┐
Schedule Trigger ────────┤                                                         ├──► Merge ──► Manage Data ──► Edit Fields ──► Update Dashboard
                         └──► Network Devices (reads Business Centers sheet) ──────┘                                                    │
                                                                                                    ┌───────────────────────────────────┼───────────────────────────────────┐
                                                                                                    ▼                                    ▼                                   ▼
                                                                                             Is Saturday?                       Is Last Day of Month?                     Wait (1)
                                                                                             Yes │   No                          Yes │   No                                │
                                                                                                 ▼    └──► No Operation              ▼    └──► No Operation                 ▼
                                                                                             Weekly (build chat card)          Monthly (build chat card)                 Daily (build chat card)
                                                                                                 │                                    │                                     │
                                                                                                 ▼                                    ▼                                     ▼
                                                                                          Weekly Update (post to Chat)         Monthly Update (post to Chat)         Daily Update (post to Chat)
```

Every run does the same data aggregation once, writes it to the dashboard, and then **always** sends the daily Chat update — while the weekly and monthly Chat updates only fire on the specific days that condition is true (Saturday, and the calendar's last day of the month, respectively).

There are no external sub-workflows called (no `Execute Workflow` node) — everything happens inside this single workflow. It does, however, have three distinct logical "sub-flows" branching off `Update Dashboard`: a daily-report branch, a weekly-report branch, and a monthly-report branch, described below.

---

## Node-by-node breakdown

### 1. `Schedule Trigger`
- **Type:** Schedule Trigger
- **Function:** Kicks off the workflow on a recurring interval (interval configuration is left at n8n's default rule).
- **Role:** The single entry point; fans out to both Google Sheets read nodes.

### 2. `Server Devices` — Google Sheets (read)
- **Function:** Reads the current month's tab from the sheet named `PRTG Critical Alerts {Month Year}` (e.g., "PRTG Critical Alerts July 2026") inside the **NOC Monitoring Automation Logs** spreadsheet.
- **Note:** despite the node name "Server Devices," it's actually pulling from the *PRTG* alerts log — worth double-checking this naming still matches your intent, since PRTG Critical Alerts is also logged by a separate related workflow (PRTG Critical Alert Automation Logs).
- **Role:** One of two data sources feeding into `Merge`.

### 3. `Network Devices` — Google Sheets (read)
- **Function:** Reads the current month's tab from the sheet named `Business Centers {Month Year}` in the same spreadsheet.
- **Role:** The second data source feeding into `Merge`.

### 4. `Merge`
- **Type:** Merge node (two inputs, index 0 = Server Devices, index 1 = Network Devices)
- **Function:** Combines both sets of ticket rows into a single data stream.
- **Role:** Passes the unified ticket list into the calculation engine.

### 5. `Manage Data` — Code node
- **Function:** The core calculation engine. For each ticket row it:
  - Skips any row without a `Date`, and skips any ticket whose `Status` is "cancelled."
  - Classifies the row as **server** or **network** based on the `Device Type` field.
  - Maps status text to **Resolved** ("resolved"/"closed") or **In Progress**, using a different set of valid in-progress status strings depending on whether it's a server or network ticket (e.g., network: "in progress," "endorse to enterprise support," "monitoring"; server: "work in progress," "waiting for support").
  - Buckets each ticket into **daily**, **weekly**, **monthly**, and **all-time (global)** totals based on its date, computing separate resolved/in-progress/monitored counts for server and network within each time window.
  - Outputs a flat list of `{ Label, Value }` rows (e.g., "Weekly Servers Resolved" → count), including section-header rows (e.g., "📊 GLOBAL MONITORING STATS") and separator rows, ready to be written straight into a spreadsheet.
- **Role:** Transforms raw ticket rows into the exact rows the dashboard sheet needs.

### 6. `Edit Fields` — Set node
- **Function:** Passes through only the `Label` and `Value` fields from `Manage Data`'s output.
- **Role:** Ensures a clean, predictable two-column shape before writing to Google Sheets.

### 7. `Update Dashboard` — Google Sheets (update)
- **Function:** Writes each `{Label, Value}` row into the **"Status"** tab of the NOC Monitoring Automation Logs spreadsheet, matching existing rows by the `Label` column (so it updates in place rather than duplicating rows).
- **Role:** This is where the "live dashboard" actually gets refreshed. From here the workflow branches into three parallel paths.

### 8. `Is Saturday?` — IF node
- **Function:** Checks whether today's day-of-week equals 6 (Saturday).
- **True:** proceeds to `Weekly` (build and send the weekly report).
- **False:** routes to `No Operation, do nothing` and that branch ends.

### 9. `Is Last Day of Month?` — IF node
- **Function:** Checks whether today's date matches the last calendar day of the current month.
- **True:** proceeds to `Monthly` (build and send the monthly report).
- **False:** routes to `No Operation, do nothing1` and that branch ends.

### 10. `Wait`
- **Type:** Wait node (1 unit delay)
- **Function:** Briefly pauses before continuing to `Daily`, likely to ensure the Google Sheets write in `Update Dashboard` has fully settled before the daily numbers are read back out for the chat card.
- **Role:** Feeds the always-runs daily reporting branch.

### 11. `Daily` — Code node
- **Function:** Reads the values written earlier (Daily Total Tickets, Daily Servers/Networks Resolved, Daily Servers/Networks In Progress) and builds a formatted Google Chat card (`cardsV2` JSON) titled "NOC AUTOMATED DASHBOARD" with a daily operational summary.
- **Role:** Feeds `Daily Update`.

### 12. `Weekly` — Code node
- **Function:** Same pattern as `Daily`, but pulls the weekly totals (Weekly Total Tickets, Weekly Total Resolved, Weekly Total In Progress) and builds a "7-Day Operational Summary" chat card.
- **Role:** Feeds `Weekly Update`. Only reached on Saturdays.

### 13. `Monthly` — Code node
- **Function:** Same pattern again, using the monthly totals (Monthly Total Tickets, Monthly Servers Monitored, Monthly Networks Monitored) to build a "30-Day Volume Insights" chat card.
- **Role:** Feeds `Monthly Update`. Only reached on the last day of the month.

### 14. `Daily Update` / `Weekly Update` / `Monthly Update` — HTTP Request nodes
- **Function:** Each POSTs its respective chat card JSON to a Google Chat webhook URL, posting the formatted card message into a designated Chat space.
- **Role:** The final delivery step for each of the three report cadences.

### 15. `No Operation, do nothing`
- **Function:** No-op; dead end for the "not Saturday" branch of `Is Saturday?`.

### 16. `No Operation, do nothing1`
- **Function:** No-op; dead end for the "not last day of month" branch of `Is Last Day of Month?`.

---

## Sub-workflows

No `Execute Workflow` nodes are used — this is a single, self-contained workflow. Its internal structure is best understood as **one shared data pipeline** (Schedule Trigger → Server/Network Devices → Merge → Manage Data → Edit Fields → Update Dashboard) that fans out into **three independent reporting branches** (Daily — always runs, Weekly — Saturday only, Monthly — last day of month only), each of which builds and sends its own Google Chat card.

---

## Configuration reference

| Item | Value |
|---|---|
| Trigger | Schedule Trigger (interval as configured in n8n) |
| Source spreadsheet | NOC Monitoring Automation Logs |
| Source sheet — "Server Devices" node | `PRTG Critical Alerts {Month Year}` |
| Source sheet — "Network Devices" node | `Business Centers {Month Year}` |
| Dashboard write target | "Status" tab, matched by `Label` column |
| Excluded ticket status | "cancelled" |
| Resolved statuses | "resolved," "closed" |
| In-progress statuses (network) | "in progress," "endorse to enterprise support," "monitoring" |
| In-progress statuses (server) | "work in progress," "waiting for support" |
| Weekly report trigger | Runs only when day-of-week is Saturday |
| Monthly report trigger | Runs only on the last calendar day of the month |
| Reporting destination | Google Chat space (via webhook `HTTP Request`) |
| Timezone | Asia/Singapore |
| n8n tag | `nocAutomations` |

---

## Notes / things to double check before publishing

- **Naming vs. source data:** the `Server Devices` node actually reads the *PRTG Critical Alerts* sheet, and `Network Devices` reads the *Business Centers* sheet — worth confirming this pairing is intentional, since the names don't match the sheet contents 1:1.
- **Secret in plain text:** the `Daily Update` / `Weekly Update` / `Monthly Update` nodes contain a live Google Chat webhook URL with an embedded API key and token. Since this workflow is planned for GitHub, that URL should be redacted or moved to a credential/environment variable before publishing — as-is, it would let anyone post messages into that Chat space.
- The three report nodes (`Daily`, `Weekly`, `Monthly`) each re-derive their numbers by re-reading the `Label`/`Value` rows produced earlier in the same execution, rather than querying the sheet again — so they depend on `Manage Data`'s output still being available in that run.
