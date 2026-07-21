# KeyCloak Logs via Jira Ticket Automation

An n8n workflow that listens for "GoFiber High Login Count Alert" Jira tickets, parses out every flagged Keycloak account mentioned in the ticket (whether it's one account or several), and logs each one as its own tracked row in a Google Sheets log.

---

## Summary

This workflow is triggered directly by Jira whenever a "Service Request" ticket matching the "GoFiber High Login Count Alert" summary is created or updated. It first checks whether the ticket is flagging a single account or multiple accounts (based on a count mentioned in the ticket summary). Single-account tickets are parsed directly; multi-account tickets have their description split into per-account blocks and parsed one at a time. Each flagged account (email, account number, user ID, login count, assignee, reporter, status) is then written as its own row into the "UAM - Keycloak Tickets" tab of the shared NOC log spreadsheet, matched on a unique ID so repeat updates to the same ticket/account update the existing row instead of duplicating it.

---

## What problem does it solve?

Keycloak login-count alerts arrive as Jira tickets, but a single ticket can flag **one account or a whole batch of accounts at once**, with the details embedded as unstructured text inside the ticket description (sometimes separated into multiple account blocks by divider lines). Manually tracking these tickets is difficult because:

- A single ticket may reference several distinct flagged accounts, each needing its own tracking row — easy to under-count or miss entries if done by hand
- The account details (email, account number, user ID, login count) live inside free-text ticket descriptions, not structured fields
- Tickets get updated over time (e.g., status changes), and without an update-aware key, re-logging would create duplicate rows instead of updating the existing ones
- Security/account-monitoring teams need this data in a structured, filterable log rather than scattered across individual Jira tickets

This workflow automatically extracts every flagged account from every matching ticket — regardless of how many are bundled into one ticket — and keeps a clean, de-duplicated log of them.

---

## Benefit to the company

- **Nothing gets missed** — even tickets that bundle multiple flagged accounts into one report have every single account extracted and logged individually.
- **Structured security/account log** — turns free-text Jira descriptions into a consistent, filterable spreadsheet record (email, account number, user ID, login count, status) useful for security review or audit trails.
- **No duplicate rows** — each ticket+account combination gets a unique `baselineId`, so ticket updates refresh the existing row instead of creating clutter.
- **Faster triage** — security/support staff can see all flagged Keycloak accounts and their statuses in one place instead of opening each Jira ticket individually.
- **Reduces manual transcription errors** — account numbers, user IDs, and login counts are extracted by regex instead of being manually copied out of ticket text.

---

## Who will use it?

- **NOC / IT Security / UAM (User Access Management) team** — as the primary consumers of the "UAM - Keycloak Tickets" log for reviewing and tracking flagged accounts.
- **Whoever creates or updates the "GoFiber High Login Count Alert" Jira tickets** — their reports are automatically captured without any manual data entry.
- **Security auditors / compliance reviewers** — benefit from having a structured historical log of flagged accounts rather than having to dig through Jira.

This runs unattended, triggered directly by Jira webhook events, as part of the NOC automation suite (tagged `nocAutomations`).

---

## How the flow works (high-level)

```
jiraTrigger (Service Request tickets matching "GoFiber High Login Count Alert")
        │
        ▼
Is Multi-Account Ticket?  (parses "<N> Accounts Flagged" from the summary, N >= 2 ?)
   ┌───────┴────────┐
  Yes                No
   ▼                 ▼
multipleAccountFlagged        1AccoountFlagged
(split description into      (parse the single
 per-account blocks,          account directly)
 extract each account)
   │                            │
   ▼                            ▼
splitData                  setDataForSingle
(one item per                  │
 flagged account)               │
   │                            │
   ▼                            │
setDataForMultiple               │
   │                            │
   └──────────────┬─────────────┘
                   ▼
                saveData
     (append/update row per account in
      "UAM - Keycloak Tickets" sheet,
      matched by baselineId)
```

Both paths converge on the same `saveData` step — the workflow always ends up writing one row per flagged account, whether it came from a single-account ticket or was split out of a multi-account one.

---

## Node-by-node breakdown

### 1. `jiraTrigger`
- **Type:** Jira Trigger, listening for `jira:issue_created` and `jira:issue_updated` events.
- **Filter (JQL):** project `CICT`, issue type "Service Request," summary containing "GoFiber High Login Count Alert."
- **Role:** The single entry point — every create or update to a matching ticket starts a new run.

### 2. `Is Multi-Account Ticket?` — IF node
- **Function:** Extracts a number from a "`<N> Accounts Flagged`" pattern in the ticket summary (defaulting to 1 if no such pattern is found) and checks whether it's ≥ 2.
- **True:** routes to `multipleAccountFlagged` (multi-account path).
- **False:** routes to `1AccoountFlagged` (single-account path).
- **Role:** Decides which parsing strategy the ticket needs.

### 3. `1AccoountFlagged` — Code node (single-account path)
- **Function:** Parses the ticket directly (handling both possible payload shapes) to pull out the ticket key, summary, assignee, reporter, and status, then regex-extracts the flagged account's email, account number, user ID, and login count from the description text. Builds a unique `baselineId` from the ticket key + user ID (or a fallback index).
- **Role:** Produces one structured record for the single flagged account.

### 4. `setDataForSingle` — Set node
- **Function:** Maps the parsed single-account fields into the exact column names used by the log sheet (`System Ticket`, `Subject`, `E-mail Address:`, `Account Number:`, `User ID:`, `Login Count:`, `Assignee`, `Reporter`, `Status`, `baselineId`).
- **Role:** Final shaping step before `saveData` for the single-account path.

### 5. `multipleAccountFlagged` — Code node (multi-account path)
- **Function:** Runs once per item. Cleans the ticket description into plain text, then splits it into blocks wherever a divider line of 10+ dashes appears (each block representing one flagged account). For every block that actually contains a User ID or Account Number, it regex-extracts that account's email, account number, user ID, and login count, and builds a unique `baselineId` per account (ticket key + user ID, or an index-based fallback). Collects all extracted accounts into a single `accountList` array.
- **Role:** Turns one multi-account ticket description into a list of individual account records.

### 6. `splitData` — Split Out node
- **Function:** Splits the `accountList` array produced by `multipleAccountFlagged` into separate individual items, one per flagged account.
- **Role:** Converts the batch into individually processable items so each account gets its own row downstream.

### 7. `setDataForMultiple` — Set node
- **Function:** Same field mapping as `setDataForSingle` (System Ticket, Subject, E-mail Address, Account Number, User ID, Login Count, Assignee, Reporter, Status, baselineId), applied to each split-out account item.
- **Role:** Final shaping step before `saveData` for the multi-account path.

### 8. `saveData` — Google Sheets node
- **Function:** Appends or updates a row in the **"UAM - Keycloak Tickets"** tab of the NOC Monitoring Automation Logs spreadsheet, matching existing rows by the `baselineId` column — so if the same ticket/account combination is seen again (e.g., on a status update), the existing row is updated in place rather than duplicated.
- **Role:** The end goal of both paths — this is what keeps the Keycloak flagged-account log current.

---

## Sub-workflows

No `Execute Workflow` nodes are used — this is a single, self-contained workflow. It splits into two independent logical paths right after the trigger: a **single-account path** (`1AccoountFlagged` → `setDataForSingle`) and a **multi-account path** (`multipleAccountFlagged` → `splitData` → `setDataForMultiple`), which both converge on the same `saveData` step.

---

## Configuration reference

| Item | Value |
|---|---|
| Trigger | Jira Trigger — `jira:issue_created`, `jira:issue_updated` |
| JQL filter | Project `CICT`, issue type "Service Request," summary contains "GoFiber High Login Count Alert" |
| Multi-account threshold | Summary count pattern `"<N> Accounts Flagged"`, N ≥ 2 |
| Target spreadsheet | NOC Monitoring Automation Logs |
| Target sheet tab | UAM - Keycloak Tickets |
| Match/dedupe key | `baselineId` (ticket key + user ID, or a fallback index) |
| Multi-account block delimiter | 10 or more consecutive dashes in the description |
| Fields extracted per account | Email, Account Number, User ID, Login Count |
| n8n tag | `nocAutomations` |

---

## Notes / things worth knowing

- The `1AccoountFlagged` node name has a typo ("Accoount") — purely cosmetic, doesn't affect functionality, but worth knowing if you search for it in the canvas.
- Both parsing branches (`1AccoountFlagged` and `multipleAccountFlagged`) contain near-duplicate extraction logic; if the underlying ticket format ever changes, both would need to be updated in sync.
- The multi-account split relies on the ticket description containing dashed divider lines (10+ dashes) between account blocks — tickets formatted differently could fail to split correctly and might need the regex/delimiter logic adjusted.
