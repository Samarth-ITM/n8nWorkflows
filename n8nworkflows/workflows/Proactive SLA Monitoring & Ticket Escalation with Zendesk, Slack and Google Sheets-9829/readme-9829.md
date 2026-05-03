Proactive SLA Monitoring & Ticket Escalation with Zendesk, Slack and Google Sheets

https://n8nworkflows.xyz/workflows/proactive-sla-monitoring---ticket-escalation-with-zendesk--slack-and-google-sheets-9829


# Proactive SLA Monitoring & Ticket Escalation with Zendesk, Slack and Google Sheets

---

### 1. Workflow Overview

This workflow automates proactive SLA (Service Level Agreement) monitoring and ticket escalation for a Zendesk-based customer support environment. It runs hourly to fetch all open Zendesk tickets, calculates SLA time remaining for each ticket, and triggers alerts and updates based on SLA consumption thresholds. The workflow’s purpose is to ensure timely responses to support tickets, prevent SLA breaches, and keep the support team informed through Slack notifications and compliance logging.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Trigger & Data Fetching:** Automatically triggers the workflow every hour and retrieves all tickets from Zendesk.
- **1.2 Ticket Filtering:** Filters tickets to only those currently open and requiring SLA monitoring.
- **1.3 SLA Calculation:** Calculates the percentage of SLA time elapsed and time remaining for each open ticket.
- **1.4 Priority Escalation & Warning Preparation:** Prepares escalation payloads and applies threshold logic for warning (75%) and critical escalation (90%) of SLA consumption.
- **1.5 Zendesk Ticket Updates:** Updates Zendesk tickets with new priorities and notes based on SLA thresholds.
- **1.6 Alerts & Notifications:** Sends Slack alerts for warnings and escalations to inform support staff.
- **1.7 Compliance Logging:** Logs ticket SLA compliance data to Google Sheets for audit and reporting.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger & Data Fetching

- **Overview:** This block initiates the workflow automatically every hour and fetches all Zendesk tickets.
- **Nodes Involved:**  
  - Trigger Every Hour  
  - Fetch Open Tickets from Zendesk  
  - 📥 Zendesk Fetch (Sticky Note)

- **Node Details:**  
  - **Trigger Every Hour**  
    - Type: Cron Trigger  
    - Configuration: Default hourly trigger (no custom cron expression specified)  
    - Connections: Output to "Fetch Open Tickets from Zendesk"  
    - Potential Failures: Cron misconfiguration unlikely but possible; workflow may be inactive and miss triggers.  
  - **Fetch Open Tickets from Zendesk**  
    - Type: Zendesk node — “getAll” operation  
    - Configuration: Retrieves all tickets (`returnAll: true`) from Zendesk API, including fields like id, status, created_at, sla_due, priority.  
    - Credentials: Requires Zendesk API credentials with read access.  
    - Connections: Output to "Filter: Open Tickets Only"  
    - Potential Failures: API authentication errors, rate limits, network timeouts, missing ticket fields.  
  - **📥 Zendesk Fetch (Sticky Note)**  
    - Technical Role: Documentation  
    - Content: Explains this node fetches all tickets with key fields required for SLA monitoring.

---

#### 1.2 Ticket Filtering

- **Overview:** Filters the fetched tickets to only those with status "open" for active SLA monitoring.
- **Nodes Involved:**  
  - Filter: Open Tickets Only  
  - 🔍 Filter Open (Sticky Note)  
  - Notify: No Open Tickets  
  - ✅ Clear Alert (Sticky Note)

- **Node Details:**  
  - **Filter: Open Tickets Only**  
    - Type: If node (conditional filter)  
    - Configuration: Checks if ticket `status` equals "open" (case-sensitive strict string match).  
    - Input: Tickets from Zendesk fetch  
    - Outputs:  
      - True: Tickets with status "open" → SLA calculation  
      - False: No open tickets → Notify via Slack  
    - Potential Failures: Expression errors if `status` field missing; case sensitivity could cause issues if status varies.  
  - **Notify: No Open Tickets**  
    - Type: Slack node (message send)  
    - Configuration: Posts a confirmation message "✅ No open tickets at this time" to Slack channel “general-information”  
    - Credentials: Slack API with posting permissions  
    - Input: From False output of filter node  
    - Potential Failures: Slack API auth errors, channel permissions, network issues.  
  - **Sticky Notes:** Provide explanation about filtering logic and alert when no open tickets.

---

#### 1.3 SLA Calculation

- **Overview:** For each open ticket, calculates SLA time remaining and percentage of SLA elapsed.
- **Nodes Involved:**  
  - Calculate SLA Time Remaining (Function)  
  - ⏱️ SLA Calculation (Sticky Note)

- **Node Details:**  
  - **Calculate SLA Time Remaining**  
    - Type: Function node  
    - Configuration: Custom JavaScript calculates:  
      - Total SLA duration = sla_due - created_at (milliseconds)  
      - Time remaining = sla_due - current time  
      - Percent elapsed = (1 - timeRemaining / slaTotal) × 100, capped between 0 and 100  
      - Adds `timeRemainingMinutes` and `percentElapsed` fields to ticket JSON  
    - Input: Open tickets from filter node  
    - Output: Tickets enriched with SLA calculation fields  
    - Potential Failures: Date parsing errors if `sla_due` or `created_at` malformed; edge cases if SLA dates missing or in the past/future.  
  - **Sticky Note:** Documents the SLA calculation logic and outputs.

---

#### 1.4 Priority Escalation & Warning Preparation

- **Overview:** Prepares escalation payload and evaluates SLA thresholds (75% warning, 90% escalation).
- **Nodes Involved:**  
  - Prepare Escalation Payload (Set)  
  - 🚨 Escalation (Sticky Note)  
  - If ≥ 90% (Escalate)1 (If)  
  - 🔴 90% Threshold (Sticky Note)  
  - If ≥ 75% (Warn)1 (If)  
  - 🟡 75% Threshold (Sticky Note)

- **Node Details:**  
  - **Prepare Escalation Payload**  
    - Type: Set node  
    - Configuration: Sets `priority` to "High" and adds note: "Auto-prioritised due to SLA nearing breach"  
    - Input: SLA-calculated tickets  
    - Output: Tickets with escalation-ready payload  
  - **If ≥ 90% (Escalate)1**  
    - Type: If node  
    - Configuration: Checks if `percentElapsed` ≥ 90  
    - Output (true): Tickets requiring escalation  
  - **If ≥ 75% (Warn)1**  
    - Type: If node  
    - Configuration: Checks if `percentElapsed` ≥ 75  
    - Output (true): Tickets requiring warning alert  
  - **Sticky Notes:** Explain SLA threshold meanings and respective actions.

---

#### 1.5 Zendesk Ticket Updates

- **Overview:** Updates Zendesk tickets based on SLA thresholds to change priority and add notes.
- **Nodes Involved:**  
  - Update Zendesk: Warning Priority  
  - 🔄 Zendesk Update (Sticky Note)  
  - Update Zendesk: Escalate (90%+)  
  - 🚨 Zendesk Escalate (Sticky Note)  

- **Node Details:**  
  - **Update Zendesk: Warning Priority**  
    - Type: Zendesk node — Update operation  
    - Configuration: Updates ticket priority to "High" and adds an internal note for tickets ≥ 75% SLA elapsed. Uses ticket ID to target.  
    - Input: From prepare escalation payload node (for tickets ≥ 75%)  
  - **Update Zendesk: Escalate (90%+)**  
    - Type: Zendesk node — Update operation  
    - Configuration: Similar update but triggered for tickets ≥ 90% SLA elapsed, ensuring priority is set to "High" and adding breach warning note.  
    - Input: From If ≥ 90% node  
  - **Sticky Notes:** Detail the update purpose and routing logic for these nodes.

---

#### 1.6 Alerts & Notifications

- **Overview:** Sends Slack alerts to notify the support team of SLA warnings.
- **Nodes Involved:**  
  - Alert Slack: SLA Warning  
  - ⚠️ Slack Alert (Sticky Note)

- **Node Details:**  
  - **Alert Slack: SLA Warning**  
    - Type: Slack node (message send)  
    - Configuration: Sends a message with ticket ID, time remaining, SLA elapsed percentage, and priority to "general-information" Slack channel.  
    - Input: From If ≥ 75% node (true output)  
    - Potential Failures: Slack API auth or permissions issues, message formatting errors.  
  - **Sticky Note:** Describes the alert content and purpose for team awareness.

---

#### 1.7 Compliance Logging

- **Overview:** Logs SLA compliance metrics for each ticket into a Google Sheets spreadsheet for auditing and analysis.
- **Nodes Involved:**  
  - Prepare Compliance Log1 (Function)  
  - Log to Google Sheets  
  - 📊 Compliance Log (Sticky Note)  
  - 📈 Sheets Logging (Sticky Note)

- **Node Details:**  
  - **Prepare Compliance Log1**  
    - Type: Function node  
    - Configuration: Generates simplified compliance log with ticket ID, percent elapsed, time remaining, and timestamp (ISO) for each ticket.  
  - **Log to Google Sheets**  
    - Type: Google Sheets node — Append operation  
    - Configuration: Appends data to a specific Google Sheets document's "Stripe Data" sheet (gid=0)  
    - Fields logged: `ticket_id`, `percent_elapsed`, `time_remaining_minutes`, `timestamp`  
    - Credentials: Google Sheets OAuth2 required with write permissions  
    - Potential Failures: OAuth token expiration, sheet access permissions, API limits  
  - **Sticky Notes:** Explain the audit and reporting purpose of logging.

---

### 3. Summary Table

| Node Name                      | Node Type              | Functional Role                 | Input Node(s)                 | Output Node(s)                         | Sticky Note                                                                                      |
|--------------------------------|------------------------|--------------------------------|------------------------------|---------------------------------------|------------------------------------------------------------------------------------------------|
| Trigger Every Hour              | Cron                   | Scheduled workflow trigger     | —                            | Fetch Open Tickets from Zendesk       | ⏰ WORKFLOW START - Runs every hour to start monitoring                                        |
| Fetch Open Tickets from Zendesk | Zendesk                | Data retrieval                 | Trigger Every Hour            | Filter: Open Tickets Only              | 📥 ZENDESK DATA RETRIEVAL - Fetches all tickets with required fields                           |
| Filter: Open Tickets Only       | If                     | Filters open tickets           | Fetch Open Tickets from Zendesk | Calculate SLA Time Remaining; Notify: No Open Tickets | 🔍 OPEN TICKET FILTER - Only open tickets proceed                                              |
| Notify: No Open Tickets         | Slack                  | Notification for no open tickets | Filter: Open Tickets Only (false) | —                                     | ✅ NO TICKETS ALERT - Confirms no open tickets                                                |
| Calculate SLA Time Remaining    | Function               | SLA elapsed/time calc          | Filter: Open Tickets Only (true) | Prepare Escalation Payload             | ⏱️ SLA CALCULATION - Calculates SLA % elapsed and time remaining                              |
| Prepare Escalation Payload      | Set                    | Escalation data preparation   | Calculate SLA Time Remaining | Update Zendesk: Warning Priority       | 🚨 ESCALATION - Prepares high priority escalation payload                                     |
| Update Zendesk: Warning Priority| Zendesk                | Update ticket at warning level | Prepare Escalation Payload    | If ≥ 90% (Escalate)1; If ≥ 75% (Warn)1; Prepare Compliance Log1 | 🔄 ZENDESK UPDATE (75%+) - Updates priority and adds internal note                            |
| If ≥ 90% (Escalate)1            | If                     | Escalation threshold check    | Update Zendesk: Warning Priority | Update Zendesk: Escalate (90%+)        | 🔴 CRITICAL THRESHOLD - Tickets at or above 90% SLA elapsed                                  |
| Update Zendesk: Escalate (90%+) | Zendesk                | Update ticket for escalation  | If ≥ 90% (Escalate)1          | —                                     | 🚨 ZENDESK UPDATE (90%+) - Sets priority high and adds breach note                            |
| If ≥ 75% (Warn)1                | If                     | Warning threshold check       | Update Zendesk: Warning Priority | Alert Slack: SLA Warning               | 🟡 WARNING THRESHOLD - Tickets at or above 75% SLA elapsed                                   |
| Alert Slack: SLA Warning        | Slack                  | Sends warning alert to Slack  | If ≥ 75% (Warn)1              | —                                     | ⚠️ SLACK WARNING ALERT - Notifies team of at-risk tickets                                    |
| Prepare Compliance Log1         | Function               | Prepare logging data          | Update Zendesk: Warning Priority | Log to Google Sheets                   | 📊 COMPLIANCE LOGGING - Prepares audit log data                                              |
| Log to Google Sheets            | Google Sheets           | Stores compliance logs        | Prepare Compliance Log1       | —                                     | 📈 GOOGLE SHEETS LOGGING - Stores SLA metrics for analysis                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Cron Trigger node ("Trigger Every Hour")**  
   - Type: Cron  
   - Set to trigger every hour (default settings)  
   - No credentials required

2. **Create a Zendesk node ("Fetch Open Tickets from Zendesk")**  
   - Type: Zendesk  
   - Operation: getAll  
   - Return All: true  
   - Credentials: Configure Zendesk API credentials with read permission  
   - Connect output from "Trigger Every Hour" node

3. **Create an IF node ("Filter: Open Tickets Only")**  
   - Type: If  
   - Condition: String equals  
   - Left Value: Expression `{{$json["status"]}}`  
   - Right Value: `"open"` (case-sensitive)  
   - Connect input from "Fetch Open Tickets from Zendesk"

4. **Create a Slack node ("Notify: No Open Tickets")**  
   - Type: Slack  
   - Message: "✅ No open tickets at this time"  
   - Channel: Select the Slack channel ID (e.g., "general-information")  
   - Credentials: Slack API OAuth with posting permissions  
   - Connect this to the **false** output of "Filter: Open Tickets Only"

5. **Create a Function node ("Calculate SLA Time Remaining")**  
   - Type: Function  
   - Paste the provided JavaScript code that calculates SLA elapsed % and time remaining in minutes.  
   - Input: Connect to **true** output of "Filter: Open Tickets Only"

6. **Create a Set node ("Prepare Escalation Payload")**  
   - Type: Set  
   - Add fields:  
     - priority (string): "High"  
     - note (string): "Auto-prioritised due to SLA nearing breach"  
   - Input: Connect from "Calculate SLA Time Remaining"

7. **Create Zendesk node ("Update Zendesk: Warning Priority")**  
   - Type: Zendesk  
   - Operation: Update  
   - Ticket ID: Expression `{{$json["id"]}}`  
   - Update Fields: Set priority and add note from previous Set node  
   - Credentials: Zendesk API credentials  
   - Connect from "Prepare Escalation Payload"

8. **Create two IF nodes for SLA thresholds:**  
   - "If ≥ 90% (Escalate)1":  
     - Condition: Number ≥ 90  
     - Value1: Expression `{{$json["percentElapsed"]}}`  
   - "If ≥ 75% (Warn)1":  
     - Condition: Number ≥ 75  
     - Value1: Expression `{{$json["percentElapsed"]}}`  
   - Connect both from "Update Zendesk: Warning Priority" node outputs (main output)

9. **Create Zendesk node ("Update Zendesk: Escalate (90%+)")**  
   - Type: Zendesk  
   - Operation: Update  
   - Ticket ID: Expression `{{$json["id"]}}`  
   - Update Fields: Set priority to High, add escalation note  
   - Credentials: Zendesk API credentials  
   - Connect from **true** output of "If ≥ 90% (Escalate)1"

10. **Create Slack node ("Alert Slack: SLA Warning")**  
    - Type: Slack  
    - Text: Expression `{{$json["message"]}}` or customize message to include ticket info  
    - Channel: Slack channel (e.g., "general-information")  
    - Credentials: Slack API OAuth  
    - Connect from **true** output of "If ≥ 75% (Warn)1"

11. **Create Function node ("Prepare Compliance Log1")**  
    - Type: Function  
    - Code: Return object with keys `id`, `percentElapsed`, `timeRemainingMinutes`, and current ISO timestamp.  
    - Connect from output of "Update Zendesk: Warning Priority"

12. **Create Google Sheets node ("Log to Google Sheets")**  
    - Type: Google Sheets  
    - Operation: Append  
    - Document ID: Google Sheets document ID  
    - Sheet Name: Sheet1 (gid=0)  
    - Columns to map: ticket_id, percent_elapsed, time_remaining_minutes, timestamp  
    - Credentials: Google Sheets OAuth2 with write access  
    - Connect from "Prepare Compliance Log1"

13. **Verify all connections:**  
    - "Trigger Every Hour" → "Fetch Open Tickets from Zendesk" → "Filter: Open Tickets Only"  
    - "Filter: Open Tickets Only" true → "Calculate SLA Time Remaining" → "Prepare Escalation Payload" → "Update Zendesk: Warning Priority"  
    - "Update Zendesk: Warning Priority" → "If ≥ 90% (Escalate)1" → "Update Zendesk: Escalate (90%+)"  
    - "Update Zendesk: Warning Priority" → "If ≥ 75% (Warn)1" → "Alert Slack: SLA Warning"  
    - "Update Zendesk: Warning Priority" → "Prepare Compliance Log1" → "Log to Google Sheets"  
    - "Filter: Open Tickets Only" false → "Notify: No Open Tickets"  

14. **Test the workflow:**  
    - Ensure Zendesk API credentials have sufficient scopes for reading and updating tickets.  
    - Slack OAuth must have permission to post messages in the target channel.  
    - Google Sheets OAuth must have write access to the specified spreadsheet.  
    - Validate date/time fields in Zendesk tickets are present and correctly formatted.  
    - Run the workflow manually to verify data flow and error handling.

---

### 5. General Notes & Resources

| Note Content                                                                                             | Context or Link                                                                                          |
|--------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| Workflow ensures early detection and proactive handling of SLA breaches in Zendesk tickets.             | Core workflow purpose                                                                                    |
| Slack notifications keep the support team aware of tickets approaching SLA limits for timely action.   | Operational alerting                                                                                     |
| Compliance logs stored in Google Sheets enable SLA trend analysis and audit reporting.                  | Reporting and audit resource                                                                              |
| Slack channel used: “general-information” — ensure correct channel ID and permissions are set.         | Slack alert destination                                                                                   |
| Google Sheets document used: “Stripe Data” (Sheet1) with columns: ticket_id, percent_elapsed, etc.     | Compliance logging destination                                                                            |
| Zendesk API credentials require read and write access to tickets and SLA fields.                        | Integration prerequisite                                                                                  |
| Sticky notes document node purposes and SLA logic to aid maintainability.                              | Workflow internal documentation                                                                           |

---

**Disclaimer:** The provided text is derived solely from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.

---