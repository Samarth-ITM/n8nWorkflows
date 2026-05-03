Monitor Zendesk Churn Risk Signals, Log in Sheets, and Alert Team in Slack

https://n8nworkflows.xyz/workflows/monitor-zendesk-churn-risk-signals--log-in-sheets--and-alert-team-in-slack-8746


# Monitor Zendesk Churn Risk Signals, Log in Sheets, and Alert Team in Slack

### 1. Workflow Overview

This workflow automates the monitoring of customer churn risk signals by analyzing Zendesk support tickets, logging relevant high-risk tickets into a Google Sheet, and sending real-time alerts to a Slack channel for the Customer Success (CS) team. It targets customer support teams that want to proactively identify dissatisfied customers and intervene quickly to reduce churn.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Ticket Retrieval:** Periodically fetches all Zendesk tickets using a schedule trigger.
- **1.2 Data Formatting and Enrichment:** Processes raw Zendesk ticket data to produce a clean, structured, and enriched dataset including computed fields like ticket age and priority level.
- **1.3 Churn Risk Filtering:** Filters tickets to identify those with negative satisfaction scores indicating potential churn risk.
- **1.4 Logging and Alerting:** Logs the filtered churn risk tickets into a Google Sheet and sends formatted Slack alerts to the CS team.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Ticket Retrieval

- **Overview:**  
  Triggers the workflow daily at 8:00 PM to fetch all tickets from Zendesk for that day, ensuring continuous monitoring without manual intervention.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Fetch Zendesk Tickets

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Initiates the workflow execution on a fixed schedule.  
    - Configuration:  
      - Cron expression: `0 0 20 * * 1-5` (at 8:00 PM Monday through Friday)  
      - Timezone: Local timezone setting (not explicitly shown but recommended)  
    - Inputs: None (trigger node)  
    - Outputs: Starts the workflow to retrieve tickets  
    - Potential Failures: Cron misconfiguration, timezone mismatch, workflow not activated  
    - Version: 1.2

  - **Fetch Zendesk Tickets**  
    - Type: Zendesk Node  
    - Role: Retrieves all tickets from the Zendesk account.  
    - Configuration:  
      - Operation: `getAll` (fetches all tickets without filters)  
      - Credentials: Zendesk API credentials for authorized access  
    - Inputs: Trigger from Schedule Trigger  
    - Outputs: Raw Zendesk ticket data array  
    - Potential Failures: API authentication errors, rate limiting, network issues, large data volume causing timeouts  
    - Version: 1

- **Sticky Note:**  
  Explains that this block fetches all tickets efficiently, with suggestions to add filters by ticket type, date, organization, or status for optimization.

---

#### 1.2 Data Formatting and Enrichment

- **Overview:**  
  Transforms raw Zendesk ticket data into a clean, structured JSON format enriched with computed fields such as ticket age in hours, priority levels, urgency flags, and channel categorization. This prepares the data for effective filtering and downstream processing.

- **Nodes Involved:**  
  - Format Ticket Data (Code Node)

- **Node Details:**

  - **Format Ticket Data**  
    - Type: Code Node (JavaScript)  
    - Role: Custom processing of each ticket item to:  
      - Normalize fields (e.g., subject, description defaults)  
      - Extract nested data (e.g., satisfaction score)  
      - Compute priority levels (low=1, normal=2, high=3, urgent=4)  
      - Calculate ticket age in hours from creation date  
      - Determine if ticket needs attention based on status and urgency or age  
      - Categorize channel types (web, API, unknown)  
    - Key Expressions/Logic:  
      - Uses JS functions inside the node to map and compute fields  
      - Handles null/undefined values gracefully  
    - Inputs: Raw ticket array from Zendesk node  
    - Outputs: Enriched ticket array with additional properties for filtering  
    - Potential Failures: JS runtime errors if input data structure changes, date parsing errors, unexpected null values  
    - Version: 2

- **Sticky Note:**  
  Details enhancements like urgency flags and null handling, highlighting that output is clean JSON ready for filtering.

---

#### 1.3 Churn Risk Filtering

- **Overview:**  
  Filters the formatted tickets to identify those with a negative satisfaction score labeled as "bad"—indicating potential churn risk.

- **Nodes Involved:**  
  - Check Negative Feedback (If Node)

- **Node Details:**

  - **Check Negative Feedback**  
    - Type: If Node (Conditional)  
    - Role: Evaluates each ticket’s `satisfaction_score` field to check if it equals "bad".  
    - Configuration:  
      - Condition: String equality check of `$json.satisfaction_score == "bad"`  
      - Combine Operation: `any` (passes if condition true for any input item)  
    - Inputs: Formatted ticket data  
    - Outputs: Routes tickets with bad ratings to logging/alerting; others are filtered out  
    - Potential Failures: Expression errors if `satisfaction_score` missing, requires consistent field naming  
    - Version: 1

- **Sticky Note:**  
  Suggests extending this filter with more complex criteria like multiple bad ratings, ticket age, priority, or VIP status to improve churn risk detection.

---

#### 1.4 Logging and Alerting

- **Overview:**  
  Logs all churn-risk tickets to a Google Sheet for tracking and analytics. Simultaneously sends a detailed Slack alert message to the Customer Success team’s channel for immediate action.

- **Nodes Involved:**  
  - Log Churn Risk to Sheet (Google Sheets Node)  
  - Send Slack Alert (Slack Node)

- **Node Details:**

  - **Log Churn Risk to Sheet**  
    - Type: Google Sheets Node  
    - Role: Appends each churn-risk ticket as a new row in a specified Google Sheet.  
    - Configuration:  
      - Operation: Append row  
      - Sheet: The main sheet with ID `11ojVmCr69OzkW8nGhkeXm9c3g8nNzqz5G4DFSfJvofQ` (Sheet1)  
      - Columns: Automatically mapped from ticket JSON fields such as `ticket_id`, `subject`, `description`, `status`, `priority`, `satisfaction_score`, `ticket_url`, etc.  
      - Credentials: OAuth2 for Google Sheets API with appropriate permissions  
    - Inputs: Filtered tickets from If node  
    - Outputs: Passes data along to Slack alert node  
    - Potential Failures: OAuth token expiry, Google Sheets API limits, schema mismatch if columns change  
    - Version: 4

  - **Send Slack Alert**  
    - Type: Slack Node  
    - Role: Sends a formatted alert message to a dedicated Slack channel.  
    - Configuration:  
      - Channel: `zendesk-churn-alerts` (Channel ID `C09FM9N8UEA`)  
      - Message Text: Includes ticket ID, satisfaction rating, subject, description, and clear action instructions  
      - Credentials: Slack API OAuth token with chat:write permissions  
      - Options: No workflow link included in message  
    - Inputs: Logged ticket data  
    - Outputs: None (end of workflow)  
    - Potential Failures: Slack API rate limits, invalid channel ID, permission errors, malformed message formatting  
    - Version: 2

- **Sticky Notes:**  
  - Tracking & Analytics: Highlights the importance of logging for reports, trend analysis, and customer health scoring.  
  - Instant Team Alerts: Emphasizes message content and suggests customization for mentions, escalation, and UX improvements.

---

### 3. Summary Table

| Node Name              | Node Type         | Functional Role                    | Input Node(s)          | Output Node(s)               | Sticky Note                                                                                     |
|------------------------|-------------------|----------------------------------|------------------------|-----------------------------|------------------------------------------------------------------------------------------------|
| Schedule Trigger       | Schedule Trigger  | Initiate daily workflow execution | None                   | Fetch Zendesk Tickets       | Automated Schedule Trigger: Daily at 8:00 PM. Cron: 0 20 * * * (Mon-Fri).                     |
| Fetch Zendesk Tickets  | Zendesk Node      | Retrieve all Zendesk tickets      | Schedule Trigger       | Format Ticket Data          | Fetches all tickets; can be customized with filters for performance.                           |
| Format Ticket Data     | Code Node         | Format and enrich ticket data     | Fetch Zendesk Tickets  | Check Negative Feedback     | Data processing: adds priority, age, urgency flags, channel type, and null handling.          |
| Check Negative Feedback| If Node           | Filter tickets with bad ratings   | Format Ticket Data     | Log Churn Risk to Sheet, Send Slack Alert | Churn risk filter: satisfaction_score = "bad". Suggests extending criteria.                    |
| Log Churn Risk to Sheet| Google Sheets Node| Log churn-risk tickets to Sheets  | Check Negative Feedback| Send Slack Alert            | Tracking & Analytics: Logs detailed ticket info for analysis and reporting.                   |
| Send Slack Alert       | Slack Node        | Send alert message to CS team     | Log Churn Risk to Sheet| None                       | Instant Team Alerts: Sends detailed Slack message; customizable for mentions and escalation.  |
| StickyNote3            | Sticky Note       | Info for Fetch Zendesk Tickets    | None                   | None                       | Describes ticket retrieval logic and customization options.                                  |
| StickyNote4            | Sticky Note       | Info for Format Ticket Data       | None                   | None                       | Describes data processing enhancements and output format.                                    |
| StickyNote5            | Sticky Note       | Info for Check Negative Feedback  | None                   | None                       | Details churn risk filtering logic and extension ideas.                                      |
| StickyNote6            | Sticky Note       | Info for Log Churn Risk to Sheet  | None                   | None                       | Explains logging benefits and captured data.                                                 |
| StickyNote7            | Sticky Note       | Info for Send Slack Alert         | None                   | None                       | Explains alert message content and customization tips.                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Configure:  
     - Set Cron expression to `0 0 20 * * 1-5` (8:00 PM Monday to Friday)  
     - Confirm timezone matches your local timezone or desired zone  
   - No credentials required

2. **Create Zendesk Node ("Fetch Zendesk Tickets")**  
   - Type: Zendesk  
   - Operation: `getAll` tickets  
   - Connect input from Schedule Trigger node output  
   - Set up Zendesk API credentials with read access to tickets

3. **Create Code Node ("Format Ticket Data")**  
   - Type: Code  
   - Connect input from Zendesk node output  
   - Paste provided JS code that:  
     - Normalizes fields (subject, description, etc.)  
     - Extracts nested data (satisfaction score)  
     - Computes priority levels and ticket age  
     - Flags urgency and attention needed  
     - Categorizes channel type  
   - Test with sample Zendesk data to confirm output structure

4. **Create If Node ("Check Negative Feedback")**  
   - Type: If  
   - Connect input from Code node output  
   - Condition: Check if `{{$json["satisfaction_score"]}} == "bad"`  
   - If true, continue; if false, stop workflow for that item

5. **Create Google Sheets Node ("Log Churn Risk to Sheet")**  
   - Type: Google Sheets  
   - Operation: Append row  
   - Connect input from If node's true output  
   - Configure:  
     - Document ID: your Google Sheet ID (create beforehand)  
     - Sheet name: e.g., "Sheet1" or specific tab  
     - Columns: Map all necessary ticket fields (ticket_id, subject, description, status, priority, satisfaction_score, ticket_url, etc.)  
   - Add OAuth2 credentials with Sheets API write access

6. **Create Slack Node ("Send Slack Alert")**  
   - Type: Slack  
   - Connect input from Google Sheets node output  
   - Configure:  
     - Channel: Select or input Slack channel ID for alerts (e.g., `zendesk-churn-alerts`)  
     - Message text: Format alert with ticket details using expressions, e.g.:  
       ```
       🚨 *Churn Risk Alert*
       **Ticket ID:** {{ $json.ticket_id }}
       **Rating:** {{ $json.satisfaction_score }}
       **Subject:** {{ $json.subject }}
       **Description:** {{ $json.description }}

       **Action Required:** Please reach out to this customer immediately to address their concerns.
       ```  
   - Add Slack API OAuth credentials with `chat:write` permission

7. **Connect Nodes in Workflow Order:**  
   Schedule Trigger → Fetch Zendesk Tickets → Format Ticket Data → Check Negative Feedback → (True) → Log Churn Risk to Sheet → Send Slack Alert

8. **Workflow Activation and Testing:**  
   - Activate the workflow  
   - Run manual tests or wait for scheduled trigger  
   - Monitor execution logs for errors (API limits, auth failures)  
   - Adjust filters or mappings as needed

---

### 5. General Notes & Resources

| Note Content                                                                                                                                       | Context or Link                                                  |
|----------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------|
| The workflow is designed to run daily on weekdays at 8:00 PM, but schedule can be customized as needed.                                            | Schedule Trigger node cron configuration                         |
| Google Sheet ID and Slack channel must be set up with correct permissions before workflow activation to avoid API errors.                         | Google Sheets and Slack API setup                                |
| Consider extending churn risk criteria beyond just “bad” satisfaction scores to improve detection accuracy.                                       | Sticky Note on Churn Risk Filter                                 |
| Slack alerts can be enhanced using Slack Block Kit formatting or mentions for better team engagement and escalation.                              | Sticky Note on Instant Team Alerts                               |
| The Code node relies on consistent Zendesk ticket JSON structure; monitor Zendesk API changes that may affect it.                                  | Format Ticket Data node details                                  |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. The process strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.