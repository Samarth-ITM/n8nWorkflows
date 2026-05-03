Track Excel 365 changes and approvals with Telegram and Google Sheets logging

https://n8nworkflows.xyz/workflows/track-excel-365-changes-and-approvals-with-telegram-and-google-sheets-logging-13716


# Track Excel 365 changes and approvals with Telegram and Google Sheets logging

This document provides a technical breakdown and implementation guide for the **Track Excel 365 changes and approvals** workflow.

### 1. Workflow Overview
The workflow is an automated monitoring system designed to track row-level changes in a Microsoft Excel 365 spreadsheet. It polls the spreadsheet every minute, compares the current data against a stored snapshot (using n8n static data), and identifies new, updated, or deleted entries. 

The logic is divided into four functional blocks:
1.  **Data Acquisition & Configuration:** Triggers the process, fetches Excel rows, and defines environment variables.
2.  **State Comparison Engine:** Compares the current state with the previous run to detect specific field changes.
3.  **Notification & Approval Routing:** Filters changes that require manual approval and sends interactive Telegram alerts.
4.  **Audit Logging:** If enabled, records every field-level change into both Google Sheets and a secondary Excel audit sheet.

---

### 2. Block-by-Block Analysis

#### 2.1 Input & Data Normalization
This block initializes the workflow and ensures data consistency before the comparison logic begins.
*   **Nodes Involved:** `Run every 1 minute`, `Get rows`, `Normalize Data`, `Environment Config`.
*   **Node Details:**
    *   **Run every 1 minute (Schedule Trigger):** Fires the workflow on a fixed interval.
    *   **Get rows (Microsoft Excel):** Fetches all data from a specific Table/Worksheet. *Requirement: A unique ID column must exist.*
    *   **Normalize Data (Code):** Ensures the unique ID is a trimmed string to prevent comparison errors caused by whitespace or formatting.
    *   **Environment Config (Code):** A centralized configuration node. 
        *   *Variables:* `idField` (the primary key), `ignoreFields` (e.g., timestamps to ignore), `enableAuditLog` (boolean), and `firstRunSilent` (prevents mass alerts on first setup).

#### 2.2 Change Detection Engine
The core logic that identifies exactly what changed between the last execution and the current one.
*   **Nodes Involved:** `Compare previous and current snapshot`.
*   **Node Details:**
    *   **Compare previous and current snapshot (Code):** 
        *   **Role:** Uses `$getWorkflowStaticData('global')` to persist state. 
        *   **Logic:** Maps the new data by ID. Checks for missing IDs (DELETED), new IDs (NEW), and compares field values for existing IDs (UPDATED).
        *   **Output:** Returns a JSON object containing a boolean `hasChanges` and an array of specific `changes` (old vs. new values).

#### 2.3 Notification & Approval Logic
Handles the communication layer via Telegram, distinguishing between general updates and specific items requiring approval.
*   **Nodes Involved:** `Check if changes exist`, `Transform changes for notification`, `Send notification`, `Filter rows with waiting approval`, `Notification Approval`.
*   **Node Details:**
    *   **Check if changes exist (If):** Gatekeeper that stops execution if no differences are found.
    *   **Transform changes for notification (Code):** Converts the technical change array into a human-readable Telegram message string.
    *   **Filter rows with waiting approval (Code):** Scans the `UPDATED` rows specifically for a change where the "Status" field becomes "Waiting Approval".
    *   **Notification Approval (Telegram):** Sends an interactive message with "Approve" and "Reject" buttons that trigger external webhooks.

#### 2.4 Audit Logging
Maintains a historical record of all modifications for compliance.
*   **Nodes Involved:** `Check if audit logging is enabled`, `Build audit log rows`, `Append Log to Excel`, `Append Log to Sheet`.
*   **Node Details:**
    *   **Check if audit logging is enabled (Switch):** Reads the configuration from Block 2.1 to decide if logging should proceed.
    *   **Build audit log rows (Code):** Flattens the change data into a row-based format: `Timestamp`, `ChangeType`, `RowID`, `Field`, `OldValue`, `NewValue`.
    *   **Append Log to Excel / Sheet:** Writes the flattened rows to the respective cloud storage providers.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Run every 1 minute | Schedule Trigger | Workflow Initiation | None | Get rows | Step 1 |
| Get rows | Microsoft Excel | Fetch Source Data | Run every 1 minute | Normalize Data | Step 1 |
| Normalize Data | Code | Data Sanitization | Get rows | Environment Config | Step 1 |
| Environment Config | Code | Central Settings | Normalize Data | Compare snapshot | Step 1 |
| Compare snapshot | Code | Logic Engine | Environment Config | Check changes | Step 2 |
| Check changes | If | Flow Control | Compare snapshot | Notification / Filter / Audit | Step 2 |
| Transform changes | Code | Notification Formatting | Check changes | Send notification | Step 2 |
| Send notification | Telegram | General Alerting | Transform changes | None | Step 2 |
| Filter approval | Code | Approval Extraction | Check changes | Notification Approval | Step 3 Approval |
| Notification Approval | Telegram | Interactive Alerts | Filter approval | None | Step 3 Approval |
| Check audit enabled | Switch | Logging Logic | Check changes | Build audit rows | Step 4 – Audit Log |
| Build audit rows | Code | Data Transformation | Check audit enabled | Append Excel / Sheet | Step 4 – Audit Log |
| Append Log to Excel | Microsoft Excel | Data Archiving | Build audit rows | None | Step 4 – Audit Log |
| Append Log to Sheet | Google Sheets | Data Archiving | Build audit rows | None | Step 4 – Audit Log |

---

### 4. Reproducing the Workflow from Scratch

1.  **Trigger Setup:** Create a `Schedule Trigger` node set to an interval of 1 minute.
2.  **Excel Connection:** Add a `Microsoft Excel` node. Configure it to `Get Rows` from your target workbook. Ensure the sheet has a column that acts as a unique ID.
3.  **Sanitization:** Add a `Code` node to trim the ID column of the incoming items using `item.json.ID.trim()`.
4.  **Configuration:** Add a `Code` node returning a `CONFIG` object. Include keys for `idField`, `ignoreFields` (array), and `enableAuditLog` (boolean).
5.  **State Engine:** Add a `Code` node. Use `$getWorkflowStaticData('global')` to retrieve `previousData`. Iterate through current items to find differences against `previousData`. Update the static data at the end of the script with the current state.
6.  **Branching:** Add an `If` node to check if the length of the detected changes is greater than 0.
7.  **General Notifications:** 
    *   Connect a `Code` node to the `True` path to loop through changes and build a string message.
    *   Connect a `Telegram` node to send this message to a specific Chat ID.
8.  **Approval Logic:**
    *   Connect a `Code` node to the `True` path. Filter items where a field (e.g., `Status`) has changed to `Waiting Approval`.
    *   Connect a `Telegram` node using the `Inline Keyboard` parameter. Set the URL of the buttons to your approval webhooks, passing the `ID` in the query parameters.
9.  **Logging Logic:**
    *   Add a `Switch` node to check the `enableAuditLog` variable from step 4.
    *   Add a `Code` node to transform the nested change object into a flat array of objects (one per field changed).
    *   Connect `Google Sheets` and `Microsoft Excel` nodes configured to `Append` data to your audit logs.
10. **Credentials:** Set up OAuth2 for Microsoft 365 and Google, and provide a Bot Token for Telegram.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **How it works:** Reads Excel, stores snapshot, compares state, detects NEW/UPDATED/DELETED, filters approvals, logs changes. | High-level Logic |
| **Setup steps:** Configure MS Excel credentials, ensure unique ID column exists, update Environment Config, (Optional) Config Google Sheets. | Initial Setup |
| **Production Ready:** Unlike simple polling, this tracks changes at the field level and avoids duplicate alerts. | Performance Note |
| **First Run Behavior:** If `firstRunSilent` is true, the workflow initializes storage without sending alerts. | Initialization |