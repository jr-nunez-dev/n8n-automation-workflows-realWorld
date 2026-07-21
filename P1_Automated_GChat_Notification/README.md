# P1 Automated GChat Notification

An n8n workflow that watches Jira for P1 (highest priority) incident tickets and automatically posts a formatted status card to a Google Chat space every time one is opened or moves through its lifecycle (waiting for support, work in progress, resolved, or cancelled).

---

## Summary

This workflow is triggered directly by Jira whenever a "[System] Incident" ticket from a specific reporter is created or has its status changed. It filters the event down to only P1-severity tickets (by matching "P1 Alert" in the summary), de-duplicates near-simultaneous Jira webhook events so the same change isn't reported twice, then determines whether the event represents a brand-new ticket or a status update. A new ticket gets a "🚨 ticket created" card; a status update gets routed to the matching card (⏳ Waiting for support, ⚙️ Work in progress, ✅ Resolved, or ⚪ Cancelled) — each posted straight into a Google Chat space with a button to open the ticket in Jira. Any internal error is itself reported into the same Chat space as an alert card.

---

## What problem does it solve?

P1 incidents are the highest-severity issues a support/NOC team handles, and every minute of delayed visibility matters. Left to Jira alone:

- Stakeholders have to actively check Jira or wait for someone to manually announce updates in chat
- Status changes (a P1 moving to "Work in progress," being resolved, etc.) are easy to miss unless someone is actively watching the ticket
- Jira webhooks can fire more than once for the same underlying change, risking duplicate or noisy notifications if posted naively
- Manually posting a Chat update for every P1 lifecycle change is a distraction the on-call/assignee shouldn't have to manage during an active incident

This workflow makes P1 visibility automatic and real-time, without adding any manual reporting burden on whoever is working the ticket.

---

## Benefit to the company

- **Real-time incident visibility** — everyone in the Chat space sees a P1 ticket the moment it's created and every time its status meaningfully changes, without needing to poll Jira.
- **No duplicate noise** — a de-duplication step ensures that even if Jira fires the same event multiple times, only one Chat message goes out per unique ticket/status combination.
- **Faster stakeholder awareness** — leadership, other support staff, and dependent teams get immediate, formatted updates (with a direct "Open Ticket" link) instead of having to ask "what's the status of that P1?"
- **Consistent, professional formatting** — every update uses the same card layout (icon, title, assignee, timestamps, open-ticket button), regardless of who's working the ticket.
- **Self-monitoring** — if something inside the workflow itself errors out, that failure is posted to the same Chat space as an alert, so broken notifications don't fail silently.
- **Noise reduction for non-P1 tickets** — only tickets explicitly flagged "P1 Alert" trigger a notification, so the Chat space isn't flooded with lower-priority ticket churn.

---

## Who will use it?

- **NOC / IT Support team and on-call engineers** — get immediate visibility into new and updating P1 incidents without having to watch Jira directly.
- **Team leads / management** — stay informed of P1 status in real time via the Chat space, useful for escalation decisions.
- **Whoever is assigned or resolving the P1 ticket** — their status changes are automatically broadcast, removing the need to separately post manual updates.
- **Anyone in the monitored Google Chat space** — passively kept up to date on the organization's most severe incidents.

This runs unattended, triggered directly by Jira webhook events, as part of the NOC automation suite (tagged `nocAutomations`).

---

## How the flow works (high-level)

```
Jira Trigger (issue created/updated, specific reporter, "[System] Incident")
        │
        ▼
Dup Manipulator (wait 3s — lets near-simultaneous Jira webhook events settle)
        │
        ▼
Extract info (normalize fields; determine CREATED vs UPDATED; build throttle/dedupe ID)
        │
        ▼
Remove Duplicates (skip if this ticket_id + status was already processed)
        │
        ▼
   P1 Only!  ── No ──► Not P1 alert (do nothing)
   (summary matches "P1 Alert"?)
        │ Yes                    └─ Error ──► Error Message (post error card to Chat)
        ▼
  Open or Update?  ── Yes (new ticket) ──► Open ticket  (🚨 "P1 ticket created" card)
        │ No (status update)                    └─ Error ──► Error Message
        ▼
   switchUpdates (route by current status)
   ├─ "Waiting for support" ──► Waiting for customer  (⏳ card)
   ├─ "Work in progress"    ──► Work in progress       (⚙️ card)
   ├─ "Resolved"            ──► Resolved               (✅ card)
   ├─ "Cancel"              ──► Cancel                 (⚪ card)
   └─ (any other status)    ──► Error/Slowness (do nothing — no card built for this yet)
```

All notification-sending nodes POST directly to the same Google Chat webhook URL, each with its own card layout.

---

## Node-by-node breakdown

### 1. `Jira Trigger`
- **Type:** Jira Trigger, listening for `jira:issue_created` and `jira:issue_updated` events.
- **Filter (JQL):** project `CICT`, from a specific reporter account, issue type `[System] Incident`, and only where the status just changed or the ticket was created within the last minute.
- **Role:** The single entry point — every matching create or status-changing update starts a run.

### 2. `Dup Manipulator` — Wait node (3 seconds)
- **Function:** Pauses execution for 3 seconds before continuing.
- **Role:** Gives a short buffer so that near-simultaneous duplicate Jira webhook deliveries for the same change land within the same rolling window the dedupe logic uses, reducing the chance of double-processing.

### 3. `Extract info` — Code node
- **Function:** Normalizes the incoming Jira payload (handles both the trigger's raw shape and a nested `issue` shape), pulling out `ticket_id`, `summary`, `assignee` (defaulting to "👥 Unassigned"), `status`, formatted `time_created`/`time_updated`, and a `throttle_id` built from ticket + status + a rolling 3-second time window. It also compares the ticket's created vs. updated timestamps — if they're within 2 seconds of each other, it flags the event as `webhook_action: "CREATED"`; otherwise `"UPDATED"`.
- **Role:** Turns the raw Jira event into clean, predictable fields for everything downstream, and determines whether this is a brand-new ticket or a status update.

### 4. `Remove Duplicates` — Remove Duplicates node
- **Function:** Skips any item whose `ticket_id`-`status` combination has already been seen in a previous execution (tracked per-workflow, with a history of up to 10,000 entries).
- **Role:** Prevents the same ticket/status transition from generating more than one Chat notification, even if Jira sends the underlying webhook event more than once.

### 5. `P1 Only!` — IF node
- **Function:** Checks whether the ticket's summary matches the pattern "P1 Alert."
- **True:** proceeds to `Open or Update?`.
- **False:** routes to `Not P1 alert` and the run ends quietly.
- **Error output:** routes to `Error Message` (this node is configured to continue on error rather than fail the whole execution).
- **Role:** Filters out any non-P1 incident so only the highest-severity tickets generate notifications.

### 6. `Open or Update?` — IF node
- **Function:** Checks whether `webhook_action` equals `"CREATED"`.
- **True:** proceeds to `Open ticket` (a brand-new P1).
- **False:** proceeds to `switchUpdates` (a status update on an existing P1).
- **Error output:** routes to `Error Message`.
- **Role:** Splits "new ticket" notifications from "status changed" notifications, since they use different card layouts.

### 7. `Open ticket` — HTTP Request node
- **Function:** POSTs a Google Chat card ("🚨 P1 ticket has been created!") including the ticket ID, current status, summary, assignee, creation time, and an "Open Ticket" button linking to the Jira issue.
- **Role:** The notification sent the moment a new P1 is opened.

### 8. `switchUpdates` — Switch node (5 outputs)
- **Function:** Routes based on the ticket's current `status` value:
  1. `"Waiting for support"` → `Waiting for customer`
  2. `"Work in progress"` → `Work in progress`
  3. `"Resolved"` → `Resolved`
  4. `"Cancel"` → `Cancel`
  5. Fallback (any other non-empty status) → `Error/Slowness`
- **Role:** Directs each status-update event to the correctly formatted Chat card.

### 9. `Waiting for customer` — HTTP Request node
- **Function:** POSTs a "⏳ ticket updated to Waiting for Customer" card with assignee, created/updated timestamps, and an Open Ticket button.
- **Note:** this card fires when the Jira status is literally `"Waiting for support"` — the card's wording ("Waiting for Customer") doesn't exactly match the Jira status string it's triggered by; worth confirming that's intentional.

### 10. `Work in progress` — HTTP Request node
- **Function:** POSTs a "⚙️ ticket updated to Work in Progress" card, same field layout as above.

### 11. `Resolved` — HTTP Request node
- **Function:** POSTs a "✅ ticket has been Resolved" card, labeling the assignee as "Resolver." This is a terminal node with no further connections.

### 12. `Cancel` — HTTP Request node
- **Function:** POSTs a "⚪ ticket has been Canceled" card, same field layout as the others.

### 13. `Error/Slowness` — No-Op node
- **Function:** Does nothing; it's the fallback destination in `switchUpdates` for any status that isn't one of the four explicitly handled ("Waiting for support," "Work in progress," "Resolved," "Cancel").
- **Role:** A placeholder — statuses like an "Error" or "Slowness" state (implied by the node's name) currently produce no notification.

### 14. `Not P1 alert` — No-Op node
- **Function:** Does nothing; dead end for any incident ticket that isn't flagged "P1 Alert" in its summary.

### 15. `Error Message` — HTTP Request node
- **Function:** POSTs a plain-text error alert to the same Google Chat space, naming which node failed (`P1 Only!` or `Open or Update?`) and the underlying error message, whenever either of those IF nodes throws an error.
- **Role:** Ensures workflow failures are visible in Chat rather than failing silently.

---

## Sub-workflows

No `Execute Workflow` nodes are used — this is a single, self-contained workflow. Logically it's built from a shared intake pipeline (trigger → dedupe → P1 filter) that then branches into a **new-ticket path** and a **status-update path** (itself fanned out across four specific statuses plus a catch-all), with a parallel **error-reporting path** attached to the two filtering IF nodes.

---

## Configuration reference

| Item | Value |
|---|---|
| Trigger | Jira Trigger — `jira:issue_created`, `jira:issue_updated` |
| JQL filter | Project `CICT`, specific reporter, issue type `[System] Incident`, status changed or created within the last minute |
| P1 filter | Summary matches regex "P1 Alert" |
| Dedupe key | `ticket_id` + `status`, scoped to the workflow, history of 10,000 entries |
| Created vs. Updated detection | Created/updated timestamps within 2 seconds apart = "CREATED" |
| Status → card mapping | Waiting for support → Waiting-for-customer card · Work in progress → WIP card · Resolved → Resolved card · Cancel → Cancelled card · anything else → no card (Error/Slowness no-op) |
| Notification channel | Google Chat space, via webhook `HTTP Request` (one shared URL across all card-sending nodes) |
| Workflow timezone | Asia/Taipei |
| n8n tag | `nocAutomations` |

---

## Notes / things to double check before publishing

- **Secret in plain text:** every card-sending node (`Open ticket`, `Waiting for customer`, `Work in progress`, `Resolved`, `Cancel`, `Error Message`) POSTs to a Google Chat webhook URL with an embedded API key and token. Before this goes to GitHub, that URL should be redacted or moved into a credential/environment variable — as-is, anyone with the repo could post into that Chat space.
- **Status-name mismatch:** the "Waiting for customer" card is triggered by the Jira status `"Waiting for support"` — the wording doesn't match; confirm this is intentional or update one to match the other for clarity.
- **Unhandled statuses:** any P1 status other than "Waiting for support," "Work in progress," "Resolved," or "Cancel" currently falls into the `Error/Slowness` no-op and produces no Chat notification — if the Jira workflow has additional statuses (e.g., an actual "Error" or "Slowness" state), this branch would need a real card built out.
