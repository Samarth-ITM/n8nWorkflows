Manage WhatsApp property maintenance tickets with WATI and Google Sheets

https://n8nworkflows.xyz/workflows/manage-whatsapp-property-maintenance-tickets-with-wati-and-google-sheets-13705


# Manage WhatsApp property maintenance tickets with WATI and Google Sheets

This technical document provides a detailed analysis and reconstruction guide for the **Tenant Support System – Maintenance Request Tracker** workflow. This system automates the lifecycle of property maintenance tickets using WhatsApp (via WATI) as the communication interface and Google Sheets as the database.

---

### 1. Workflow Overview

The workflow manages property maintenance requests from inception to resolution. It serves three primary user groups: **Tenants** (requesting help), **Maintenance Teams** (performing work), and **Landlords** (monitoring performance).

**Functional Blocks:**
*   **1.1 Input & Command Routing:** Receives all incoming WhatsApp messages and routes them based on keyword commands (`update`, `rate`, `mystatus`, `dashboard`) or treats them as new ticket requests.
*   **1.2 Ticket Creation Path:** Automatically classifies the issue category (Plumbing, Electrical, etc.) and priority using keyword analysis, generates a unique ID, and logs the request.
*   **1.3 Status Management:** Allows maintenance teams to update ticket progress via a standardized WhatsApp command, which synchronizes with the database and notifies the tenant.
*   **1.4 Feedback & Reporting:** Captures tenant satisfaction ratings and generates a real-time management dashboard for the landlord.
*   **1.5 Proactive SLA Monitoring:** A scheduled process that runs daily to identify overdue tickets and alert stakeholders of service level agreement (SLA) breaches.

---

### 2. Block-by-Block Analysis

#### 2.1 Input & Command Routing
This block acts as the gateway for all human interaction.
*   **Nodes Involved:** `Wati Trigger`, `Master Router`.
*   **Node Details:**
    *   **Wati Trigger (v1):** Listens for the `messageReceived` event. It captures the sender's phone number (`waId`), name, and message text.
    *   **Master Router (Switch v3):** Evaluates the message text using string operations (`startsWith` or `equals`). 
        *   *Outputs:* "Status Update", "Rate Ticket", "My Tickets", "Dashboard", and a fallback "extra" for new ticket creation.

#### 2.2 Ticket Creation Path
Triggered when a message does not match any specific command keywords.
*   **Nodes Involved:** `Classify Issue & Create Ticket`, `Google Sheets – Create Ticket`, `WATI – Confirm to Tenant`, `WATI – Notify Maintenance Team`.
*   **Node Details:**
    *   **Classify Issue & Create Ticket (Code v2):** Runs JavaScript to scan the message for keywords (e.g., "leak" → Plumbing). It assigns a priority (High/Medium/Low) based on urgency keywords and calculates the `slaDueAt` timestamp. It generates a Ticket ID in the format `TKT-YYYYMMDD-XXXX`.
    *   **Google Sheets – Create Ticket (v4.5):** Appends a new row to the "Tickets" sheet with all metadata (Category, Priority, SLA deadline).
    *   **WATI Nodes:** Sends tailored messages. The tenant gets a confirmation with their ID; the specific maintenance team (mapped via category) receives the ticket details.

#### 2.3 Status Management
Triggered by the command: `update <ticket-id> <status> <note>`.
*   **Nodes Involved:** `Google Sheets – Read Tickets (Update)`, `Parse Status Update`, `Google Sheets – Update Ticket Status`, `WATI – Acknowledge Team`, `WATI – Notify Tenant of Update`.
*   **Node Details:**
    *   **Parse Status Update (Code v2):** Validates the command syntax and checks if the status is valid (`assigned`, `in-progress`, `resolved`, `closed`). It cross-references the input ID against the current sheet data.
    *   **Google Sheets – Update (v4.5):** Uses `ticketId` as the lookup key to update the `status`, `notes`, and `lastUpdated` columns.
    *   **Notifications:** If the status is "resolved", the logic automatically appends a rating prompt to the tenant's notification.

#### 2.4 Feedback & Queries
Handles tenant status checks and ratings, and landlord dashboard requests.
*   **Nodes Involved:** `Parse Rating`, `Google Sheets – Save Rating`, `Google Sheets – Read My Tickets`, `Build My Tickets View`, `Build Landlord Dashboard`.
*   **Node Details:**
    *   **Parse Rating (Code v2):** Processes `rate <id> <1-5>`. It converts the numeric rating into a star-emoji string for visual confirmation.
    *   **Build Landlord Dashboard (Code v2):** Aggregates data from all rows to calculate: Total open tickets, SLA breach count, Average rating, and a breakdown by category.

#### 2.5 SLA Monitoring (Scheduled)
A standalone logic path triggered by time rather than a message.
*   **Nodes Involved:** `Schedule Trigger – Daily SLA Check 8AM`, `Check SLA Breaches`, `Split Team Alerts`, `WATI – Send SLA Report to Landlord`.
*   **Node Details:**
    *   **Check SLA Breaches (Code v2):** Compares the current time against the `slaDueAt` column for all open tickets. 
    *   **Thresholds:** High (4h), Medium (24h), Low (72h).
    *   **Output:** Generates a summarized report for the landlord and individual alert objects for each delinquent maintenance team.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Wati Trigger | WATI Trigger | Entry Point | - | Master Router | Handles all tenant & maintenance replies |
| Master Router | Switch | Logic Routing | Wati Trigger | Code/Sheets nodes | - |
| Classify Issue & Create Ticket | Code | AI/Logic | Master Router | Google Sheets | Classify Issue Code uses keyword matching to auto-detect category |
| Google Sheets – Create Ticket | Google Sheets | Database | Classify Issue | WATI Nodes | Sheets Append creates ticket with unique ID |
| Parse Status Update | Code | Data Parsing | Sheets Read | Sheets Update | Maintenance team replies with update... Parse Update Code extracts fields |
| Google Sheets – Update Ticket Status | Google Sheets | Database | Parse Status Update | WATI Nodes | Sheets Update changes ticket status |
| Parse Rating | Code | Data Parsing | Master Router | Google Sheets | rating logged in Sheets |
| Build Landlord Dashboard | Code | Aggregation | Sheets Read | WATI | Landlord texts dashboard -> gets full property overview |
| Schedule Trigger | Schedule | Time Trigger | - | Sheets Read | Runs every morning at 8AM |
| Check SLA Breaches | Code | Monitoring | Sheets Read | WATI / Split | Flags any ticket older than the SLA threshold |
| Split Team Alerts | Code | Flow Control | Check SLA | WATI | Pings each maintenance team for their overdue tickets |

---

### 4. Reproducing the Workflow from Scratch

1.  **Database Setup:** Create a Google Sheet named "Tickets" with headers: `ticketId`, `phone`, `tenantName`, `issueText`, `category`, `priority`, `status`, `assignedTeam`, `teamPhone`, `createdAt`, `slaDueAt`, `rating`, `lastUpdated`, `notes`.
2.  **Input Trigger:** Add a **WATI Trigger** node. Set the event to `messageReceived`.
3.  **Command Routing:** 
    *   Add a **Switch** node (Master Router). 
    *   Set 4 rules for `text` starting with/equals to: "update", "rate", "mystatus", "dashboard". 
    *   Enable the "Fallback" output for new ticket creation.
4.  **Ticket Creation Path:**
    *   Connect the Fallback to a **Code** node. Use logic to map keywords (leak, pipe, water) to categories. Calculate SLA hours (4, 24, 72).
    *   Add a **Google Sheets (Append)** node. Map the output of the Code node to the sheet headers.
    *   Add two **WATI (Send Message)** nodes to notify the tenant and the respective team.
5.  **Update Path:**
    *   Connect the "update" output to a **Google Sheets (Get Many)** node to read the current status.
    *   Connect to a **Code** node to parse the update command and validate the ticket ID.
    *   Add a **Google Sheets (Update)** node. Set the "Lookup Column" to `ticketId`.
    *   Add **WATI** nodes to confirm the update to both parties.
6.  **SLA Monitor:**
    *   Add a **Schedule Trigger**. Set to "Every Day" at 08:00.
    *   Connect to **Google Sheets (Get Many)** to fetch all rows.
    *   Add a **Code** node to iterate through rows, check if `status` is not "closed" and `now > slaDueAt`.
    *   Use a **WATI** node to send the summary to the Landlord’s phone number.
7.  **Credentials:**
    *   Configure **Google Sheets OAuth2** with `https://www.googleapis.com/auth/spreadsheets` scope.
    *   Configure **WATI API** with your API Key and Endpoint URL.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Valid Statuses for Updates | `assigned`, `in-progress`, `resolved`, `closed` |
| SLA Thresholds | High: 4h, Medium: 24h, Low: 72h |
| Keywords for Classification | Located inside the "Classify Issue" Code node |
| Requirements | WATI account and Google Cloud Console Project for Sheets API |