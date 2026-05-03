Track Support Performance Metrics Across Zendesk & Freshdesk with Sheets, Slack & Gmail

https://n8nworkflows.xyz/workflows/track-support-performance-metrics-across-zendesk---freshdesk-with-sheets--slack---gmail-8814


# Track Support Performance Metrics Across Zendesk & Freshdesk with Sheets, Slack & Gmail

### 1. Workflow Overview

This workflow automates the tracking and reporting of support team performance metrics by aggregating ticket data from two major platforms: **Zendesk** and **Freshdesk**. It fetches tickets weekly, normalizes and processes the ticket data to compute KPIs (Key Performance Indicators), and then logs this data in Google Sheets for historical analysis. Alerts based on KPI thresholds are generated and sent to a Slack channel, while a detailed HTML report is emailed weekly to stakeholders.

**Target Use Cases:**  
- Support team managers monitoring SLA compliance, ticket resolution rates, and customer satisfaction across multiple platforms.  
- Automated performance dashboards combining Zendesk and Freshdesk data.  
- Proactive alerting on support bottlenecks or SLA breaches.  
- Weekly summary reports delivered to leadership.

**Logical Blocks:**

- **1.1 Weekly Trigger:** Initiates the workflow on a scheduled weekly basis.  
- **1.2 Ticket Fetching:** Retrieves all tickets from Zendesk and Freshdesk APIs.  
- **1.3 Data Formatting:** Normalizes and unifies ticket data from both platforms into a consistent JSON structure.  
- **1.4 Data Merging:** Combines ticket arrays from both sources into a single dataset.  
- **1.5 KPI Calculation:** Processes all tickets to compute performance metrics, SLA breaches, CSAT scores, and generates alerts/recommendations.  
- **1.6 Alert Evaluation:** Determines if any alert thresholds are met to trigger notifications.  
- **1.7 Slack Alert Generation:** Formats KPI and alert data into a Slack-friendly message.  
- **1.8 Slack Notification:** Sends the alert message to a configured Slack channel.  
- **1.9 Google Sheets Logging:** Logs ticket KPIs and details into a Google Sheet for audit and trend analysis.  
- **1.10 Weekly Email Report:** Generates and sends a corporate-style HTML report summarizing KPIs and alerts via Gmail.

---

### 2. Block-by-Block Analysis

#### 1.1 Weekly Trigger

- **Overview:**  
  Automatically triggers the workflow once a week at a specific hour (20:00), ensuring periodic KPI updates and alerts.

- **Nodes Involved:**  
  - Weekly Trigger

- **Node Details:**  
  - Type: Cron Trigger  
  - Configuration: Trigger time set to weekly at 20:00 (8 PM).  
  - Inputs: None (trigger node)  
  - Outputs: Starts ticket fetching nodes.  
  - Edge Cases: Misconfiguration may lead to missed or multiple runs. Timezone settings may affect trigger time.

---

#### 1.2 Ticket Fetching

- **Overview:**  
  Fetches all tickets from Zendesk and Freshdesk APIs, retrieving comprehensive ticket data for further processing.

- **Nodes Involved:**  
  - Fetch Tickets From Zendesk  
  - Fetch Tickets From Freshdesk

- **Node Details:**

  **Fetch Tickets From Zendesk**  
  - Type: Zendesk node  
  - Operation: getAll (fetch all tickets)  
  - ReturnAll: true (fetch complete dataset)  
  - Credentials: Zendesk API credentials configured  
  - Output: Raw Zendesk tickets JSON array  
  - Edge Cases: API rate limits, authentication errors, network timeouts.

  **Fetch Tickets From Freshdesk**  
  - Type: Freshdesk node  
  - Operation: getAll (fetch all tickets)  
  - ReturnAll: true  
  - Credentials: Freshdesk API key authentication  
  - Output: Raw Freshdesk tickets JSON array  
  - Edge Cases: API key expiry, rate limits, connectivity issues.

---

#### 1.3 Data Formatting

- **Overview:**  
  Normalizes and converts raw ticket data from Zendesk and Freshdesk into a unified JSON format, enabling consistent KPI calculation.

- **Nodes Involved:**  
  - Format  Ticket Data (Zendesk & Freshdesk)

- **Node Details:**  
  - Type: Code node (JavaScript)  
  - Logic:  
    - Auto-detects ticket source platform (Zendesk or Freshdesk).  
    - Extracts and normalizes key fields: ticket ID, URL, subject, priority, status, timestamps, description, tags, requester, assignee, channel, ticket age, attention flags, etc.  
    - Cleans descriptions for preview (removes HTML, truncates).  
    - Converts Freshdesk numeric priorities/statuses to text equivalents.  
    - Calculates ticket age in hours and flags tickets needing attention based on age, priority, escalation, or tags.  
  - Inputs: Raw tickets from Zendesk and Freshdesk nodes  
  - Outputs: Array of formatted ticket JSON objects  
  - Edge Cases: Unrecognized ticket source defaults to 'unknown' with error note; missing fields handled gracefully; potential parsing errors if API changes.

---

#### 1.4 Data Merging

- **Overview:**  
  Combines formatted ticket data from Zendesk and Freshdesk into a single unified stream for KPI analysis.

- **Nodes Involved:**  
  - Merge

- **Node Details:**  
  - Type: Merge node  
  - Configuration: No special options (default merge)  
  - Inputs: Two inputs from the Zendesk and Freshdesk formatting outputs  
  - Outputs: Single combined array of tickets  
  - Edge Cases: Unequal batch sizes handled; if one input is empty, output is the other input only.

---

#### 1.5 KPI Calculation

- **Overview:**  
  Calculates comprehensive KPIs from unified ticket data, including SLA breaches, resolution rates, average response times, CSAT scores (simulated), distributions, and generates alerts and recommendations.

- **Nodes Involved:**  
  - Calculate Support KPIs

- **Node Details:**  
  - Type: Code node (JavaScript)  
  - Logic:  
    - Parses all tickets to count totals, open vs resolved, urgent/high priority tickets, escalated, overdue, needs attention flags.  
    - Normalizes status and priority for both platforms.  
    - Calculates SLA breach conditions with platform-specific rules.  
    - Computes average ticket age, resolution rates, SLA breach rates, CSAT scores (simulated for this demo).  
    - Generates boolean alert flags for various thresholds (e.g., SLA breach > 20%, low CSAT).  
    - Assigns a performance grade (A–D) based on composite KPI scores.  
    - Builds detailed recommendations for workflow improvements.  
    - Provides distribution analysis by channel, priority, platform.  
  - Inputs: Merged formatted ticket data  
  - Outputs: JSON object with KPI metrics, alerts, insights, recommendations  
  - Edge Cases: Empty ticket list handled gracefully; simulated CSAT may not reflect real data; date parsing errors possible if timestamps are malformed.

---

#### 1.6 Alert Evaluation

- **Overview:**  
  Evaluates KPI results to determine if any alert conditions are met to trigger notifications.

- **Nodes Involved:**  
  - Evaluate Alerts (IF node)

- **Node Details:**  
  - Type: IF node  
  - Conditions: Checks if `any_alert` flag is true OR high SLA breach alert flag is true  
  - Inputs: KPI JSON from Calculate Support KPIs  
  - Outputs:  
    - True branch: Proceed to generate Slack and email reports  
    - False branch: Workflow ends without alerts  
  - Edge Cases: Expression failures if KPI structure changes; false positives if KPI calculation is inaccurate.

---

#### 1.7 Slack Alert Generation

- **Overview:**  
  Formats the KPI and alert data into a structured, emoji-enhanced Slack message for immediate team visibility.

- **Nodes Involved:**  
  - Generate Slack Alert Message

- **Node Details:**  
  - Type: Code node (JavaScript)  
  - Logic:  
    - Builds textual summary including performance grade with emoji, key metrics (ticket counts, resolution rates, CSAT), distributions by priority and channel.  
    - Lists active alerts with warning emojis.  
    - Includes recommendations for improvement.  
    - Adds report timestamp.  
    - Returns Slack message payload with text and optional block formatting.  
  - Inputs: KPI JSON from Evaluate Alerts  
  - Outputs: Slack message JSON  
  - Edge Cases: Missing KPI fields cause message issues; large message size may exceed Slack limits; emoji mappings fixed.

---

#### 1.8 Slack Notification

- **Overview:**  
  Sends the formatted alert message to a predefined Slack channel for team notification.

- **Nodes Involved:**  
  - Send Slack Alert

- **Node Details:**  
  - Type: Slack node  
  - Configuration:  
    - Channel ID is fixed to "C09FM9N8UEA" (zendesk-churn-alerts)  
    - Message text and blocks set from previous node output  
  - Credentials: Slack API token configured  
  - Inputs: Slack message JSON from Generate Slack Alert Message  
  - Outputs: Slack API response  
  - Edge Cases: Slack token expiry, channel permission issues, API rate limits.

---

#### 1.9 Google Sheets Logging

- **Overview:**  
  Appends or updates ticket KPI records into a Google Sheet for audit trails and trend analysis.

- **Nodes Involved:**  
  - Log KPIs in Google Sheets

- **Node Details:**  
  - Type: Google Sheets node  
  - Operation: appendOrUpdate  
  - Sheet: Google Sheet "Performance Report Support" (document ID and sheet ID provided)  
  - Mapping: Auto-maps all ticket fields including platform, ticket ID, URL, priority, status, timestamps, descriptions, flags, sentiment, etc.  
  - Matching Key: ticket_id to avoid duplicates  
  - Credentials: Google OAuth2 credentials configured  
  - Inputs: Formatted ticket data array from Format Ticket Data node  
  - Outputs: Google Sheets update response  
  - Edge Cases: API quota limits, permission errors, sheet schema changes causing mapping failures.

---

#### 1.10 Weekly Email Report

- **Overview:**  
  Generates a polished HTML report of the weekly support KPIs and sends it via Gmail to stakeholder email addresses.

- **Nodes Involved:**  
  - Generate Weekly HTML Report  
  - Send Weekly Email

- **Node Details:**

  **Generate Weekly HTML Report**  
  - Type: Code node (JavaScript)  
  - Logic:  
    - Builds a corporate-style HTML email including:  
      - System status banner (Healthy or Alert with color coding)  
      - Performance grade block with color-coded grade  
      - Key metrics grid (total tickets, resolved, open, CSAT)  
      - Detailed performance metrics (response/ resolution times, SLA breach rate)  
      - Active alerts with highlight if any  
      - Recommendations section  
      - Distribution analysis tables for priority and channel  
      - Footer with generation timestamp  
    - Also provides plain text fallback content  
  - Inputs: KPI JSON from Evaluate Alerts  
  - Outputs: HTML and plain text email body with subject line  
  - Edge Cases: Large email size; malformed KPI data; HTML rendering differences in email clients.

  **Send Weekly Email**  
  - Type: Gmail node  
  - Configuration:  
    - To address set by user input (placeholder in node)  
    - Subject and HTML body from previous node  
  - Credentials: Gmail OAuth2 configured  
  - Inputs: Email content JSON from HTML report node  
  - Outputs: Gmail API response  
  - Edge Cases: OAuth token expiry, recipient address misconfiguration, quota limits.

---

### 3. Summary Table

| Node Name                       | Node Type              | Functional Role                     | Input Node(s)                       | Output Node(s)                      | Sticky Note                                                                                                      |
|--------------------------------|------------------------|-----------------------------------|-----------------------------------|-----------------------------------|------------------------------------------------------------------------------------------------------------------|
| Weekly Trigger                 | Cron Trigger           | Weekly workflow trigger            | None                              | Fetch Tickets From Zendesk, Fetch Tickets From Freshdesk | ## ⏰ Weekly Cron Trigger This node runs the workflow automatically on a **weekly schedule**. Configured to execute every week at the specified hour (20:00). Purpose: Ensures weekly performance reporting. Automates ticket fetching and KPI calculations. |
| Fetch Tickets From Zendesk     | Zendesk API            | Retrieve all Zendesk tickets       | Weekly Trigger                   | Merge                             | ## 🎟️ Fetch Tickets from Zendesk Fetches all ticket data from Zendesk account using API credentials. Retrieves tickets with various statuses for processing. |
| Fetch Tickets From Freshdesk   | Freshdesk API          | Retrieve all Freshdesk tickets     | Weekly Trigger                   | Merge                             | ## 🎫 Fetch Tickets from FreshDesk Fetches all ticket data from FreshDesk account using API key authentication. Retrieves tickets with various statuses. |
| Merge                         | Merge                  | Combine ticket data arrays         | Fetch Tickets From Zendesk, Fetch Tickets From Freshdesk | Format  Ticket Data (Zendesk & Freshdesk) | ## 🔗 Merge Ticket Data Combines formatted ticket data from multiple sources into a single stream maintaining unified structure for KPI calculations. |
| Format  Ticket Data (Zendesk & Freshdesk) | Code                   | Normalize and unify ticket data    | Merge                            | Log KPIs in Google Sheets         | ## 🧹 Format Ticket Data (Zendesk & FreshDesk) Cleans and standardizes raw ticket data from both platforms. Auto-detects platform, normalizes fields, creates unified JSON output. |
| Log KPIs in Google Sheets     | Google Sheets           | Log ticket data and KPIs           | Format  Ticket Data (Zendesk & Freshdesk) | Calculate Support KPIs            | ## 📑 Log Alerts in Google Sheets Stores ticket and performance data into Google Sheets for auditing and trend analysis. |
| Calculate Support KPIs        | Code                   | Compute KPIs and generate alerts   | Log KPIs in Google Sheets         | Evaluate Alerts                   | ## 📊 Calculate Support KPIs Processes formatted ticket data to compute key performance metrics, SLA breaches, CSAT, and generates alerts and recommendations. |
| Evaluate Alerts               | IF                     | Determine alert conditions         | Calculate Support KPIs            | Generate Slack Alert Message, Generate Weekly HTML Report | ## 🚨 Evaluate Alerts Checks if KPI thresholds exceeded to trigger alerts (SLA breach, low resolution, backlog, CSAT, etc.). Outputs boolean flags. |
| Generate Slack Alert Message  | Code                   | Format KPIs into Slack message     | Evaluate Alerts                   | Send Slack Alert                 | ## 💬 Generate Slack Alert Message Formats KPI and alert details into Slack-ready message with emojis, key metrics, breakdowns, alerts, and recommendations. |
| Send Slack Alert              | Slack                  | Send alert message to Slack channel | Generate Slack Alert Message      | None                             | ## 📢 Send Alert to Slack Delivers alert message to configured Slack channel for immediate visibility of critical issues. |
| Generate Weekly HTML Report   | Code                   | Build HTML email report            | Evaluate Alerts                   | Send Weekly Email                | ## 📧 Generate Weekly HTML Report Builds corporate-style HTML email report including system status, performance grade, KPIs, alerts, recommendations, and distributions. |
| Send Weekly Email             | Gmail                  | Send weekly report via email      | Generate Weekly HTML Report       | None                             | ## ✉️ Email Weekly Report Sends the weekly HTML report via Gmail to configured recipients for leadership visibility on support KPIs. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Weekly Trigger Node**  
   - Type: Cron Trigger  
   - Parameters: Set to trigger every week at 20:00 (8 PM).  
   - No credentials needed.  

2. **Create Zendesk Fetch Node**  
   - Type: Zendesk  
   - Operation: getAll tickets  
   - ReturnAll: true  
   - Credentials: Configure Zendesk API credentials with API token or OAuth.  
   - Connect input from Weekly Trigger node.  

3. **Create Freshdesk Fetch Node**  
   - Type: Freshdesk  
   - Operation: getAll tickets  
   - ReturnAll: true  
   - Credentials: Configure Freshdesk API key credentials.  
   - Connect input from Weekly Trigger node.  

4. **Create Merge Node**  
   - Type: Merge  
   - Inputs: Connect first input from Zendesk Fetch, second input from Freshdesk Fetch.  
   - No special configuration needed.  

5. **Create Code Node "Format Ticket Data (Zendesk & Freshdesk)"**  
   - Paste provided JavaScript code that:  
     - Detects ticket platform from input JSON.  
     - Normalizes fields (priority, status, timestamps, etc.).  
     - Calculates ticket age and attention flags.  
     - Cleans description fields.  
   - Input: Connect from Merge node.  
   - Output: Formatted ticket JSON array.  

6. **Create Google Sheets Node "Log KPIs in Google Sheets"**  
   - Operation: appendOrUpdate  
   - Document ID: Use your Google Sheets document ID for performance report.  
   - Sheet Name: Use sheet name or gid as per your sheet.  
   - Mapping Mode: Auto-map input data fields to columns.  
   - Matching Column: ticket_id to avoid duplicates.  
   - Credentials: Configure Google Sheets OAuth2 credentials.  
   - Input: Connect from Format Ticket Data node.  

7. **Create Code Node "Calculate Support KPIs"**  
   - Paste the provided extensive JavaScript code that:  
     - Combines ticket data metrics.  
     - Computes SLA breaches, resolution rates, CSAT, distributions.  
     - Builds alert flags and recommendations.  
     - Assigns performance grade and insights.  
   - Input: Connect from Google Sheets node.  

8. **Create IF Node "Evaluate Alerts"**  
   - Condition: Check if `any_alert` boolean is true OR `alerts.high_sla_breach` is true in KPI JSON.  
   - Input: Connect from Calculate Support KPIs node.  
   - True output: Proceed to alert/report nodes.  
   - False output: End or no action.  

9. **Create Code Node "Generate Slack Alert Message"**  
   - Paste JavaScript code that formats KPI JSON into Slack message text and blocks with emojis and sections.  
   - Input: Connect from Evaluate Alerts node (true output).  

10. **Create Slack Node "Send Slack Alert"**  
    - Configure Slack credentials (Slack API token with chat:write scope).  
    - Set channel ID to target alert channel (e.g., "C09FM9N8UEA").  
    - Input: Connect from Generate Slack Alert Message node.  

11. **Create Code Node "Generate Weekly HTML Report"**  
    - Paste provided JavaScript generating corporate HTML email and plain text fallback.  
    - Input: Connect from Evaluate Alerts node (true output).  

12. **Create Gmail Node "Send Weekly Email"**  
    - Configure Gmail OAuth2 credentials.  
    - Set recipient emails in "To" list.  
    - Use subject and HTML body from Generate Weekly HTML Report node.  
    - Input: Connect from Generate Weekly HTML Report node.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                           | Context or Link                                                                                           |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| This workflow was designed to unify support performance tracking across Zendesk and Freshdesk, providing consolidated KPIs and alerts.                                                | Workflow description                                                                                    |
| Slack channel ID "C09FM9N8UEA" is configured for alerts; update it accordingly for your workspace.                                                                                     | Slack alert node configuration                                                                          |
| Google Sheets document ID and sheet name must be replaced with your own document where KPIs are logged.                                                                                | Google Sheets node configuration                                                                        |
| The Freshdesk ticket URL template requires updating with your actual domain.                                                                                                            | Code node "Format Ticket Data"                                                                           |
| Simulated CSAT scores are used for demonstration; in production, integrate real CSAT data from APIs or surveys.                                                                        | KPI calculation code node comments                                                                      |
| Email recipient address in the Gmail node must be set to actual stakeholders.                                                                                                           | Gmail node configuration                                                                                 |
| Slack and Gmail credentials require proper OAuth2 or API token setup with required scopes and permissions.                                                                              | Credential setup                                                                                        |
| Timezone considerations for cron trigger should match your operational timezone to ensure reports run as expected.                                                                    | Weekly Trigger node                                                                                      |
| The workflow supports extension to include other platforms or additional ticket fields by enhancing the formatter and KPI calculator nodes.                                           | Customization suggestions                                                                                |
| Video explanation and demo of n8n workflows integrating Zendesk and Freshdesk available at: https://blog.n8n.io/automate-support-team-performance-2e3b5f8f3d1                            | External blog link                                                                                       |

---

**Disclaimer:**  
The text provided stems exclusively from an automated workflow created with n8n, a powerful integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.