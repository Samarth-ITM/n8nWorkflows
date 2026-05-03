Capture and Nurture Fraud-Proof Leads with AI Scoring, Sheets Tracking & Multi-Channel Alerts

https://n8nworkflows.xyz/workflows/capture-and-nurture-fraud-proof-leads-with-ai-scoring--sheets-tracking---multi-channel-alerts-8879


# Capture and Nurture Fraud-Proof Leads with AI Scoring, Sheets Tracking & Multi-Channel Alerts

### 1. Workflow Overview

This workflow automates the end-to-end process of capturing, validating, scoring, nurturing, and reporting leads with a strong focus on fraud prevention and quality assurance. It is designed primarily for B2B SaaS companies, sales teams, and ad agencies seeking to efficiently manage leads and improve conversion through AI-based scoring, multi-channel notifications, and engagement tracking.

The workflow comprises the following logical blocks:

- **1.1 Lead Capture and Validation**  
  Receives incoming lead data via a webhook, validates the email address using Verifi Email API, and enriches lead data with IP geolocation.

- **1.2 Lead Quality Scoring and Data Storage**  
  Combines lead and IP data, calculates a quality score based on business rules, and appends the enriched lead record to a Google Sheet for tracking.

- **1.3 Notification and Lead Filtering**  
  Checks if the lead score surpasses a configurable threshold, notifies the sales team via Slack if so, or otherwise discards the lead silently.

- **1.4 Lead Nurturing and Engagement Tracking**  
  Sends automated personalized welcome emails, tracks email opens, waits 24 hours to check for engagement, and sends follow-up emails if no response is detected. Engagement scores are calculated accordingly.

- **1.5 Reporting**  
  Runs weekly on a schedule to generate a sales performance report summarizing total leads, high-quality leads, and average engagement, then emails this report to the sales head.

- **1.6 Configuration and Utilities**  
  Centralized user-configurable parameters and multiple sticky notes provide detailed instructions and contextual information for maintenance and customization.

---

### 2. Block-by-Block Analysis

#### 1.1 Lead Capture and Validation

**Overview:**  
This block captures lead data from external sources via a webhook, validates the email addresses using a specialized API, and enriches leads with IP geolocation data to assist in fraud detection and lead profiling.

**Nodes Involved:**  
- Capture Leads (Webhook)  
- Set User Config (Set)  
- Validate Email (Verifi Email)  
- Filter Valid Emails (If)  
- IP Lookup (HTTP Request)  
- Merge (Merge)

**Node Details:**

- **Capture Leads**  
  - Type: Webhook  
  - Role: Entry point for receiving POST requests containing lead form data.  
  - Config: Path set to `/lead-capture`, HTTP method POST.  
  - Inputs: External HTTP POST.  
  - Outputs: Leads data JSON with IP.  
  - Edge cases: Missing or malformed payloads, webhook unavailability.

- **Set User Config**  
  - Type: Set  
  - Role: Centralizes configurable parameters such as lead score threshold, Slack channel, sales head email.  
  - Config: Defines `score_threshold` (70), `sales_channel` (#leads), `sales_head_email`.  
  - Inputs: Output from Capture Leads.  
  - Outputs: Passes data with added config fields.  
  - Edge cases: Misconfiguration leading to improper thresholds or notification failures.

- **Validate Email**  
  - Type: Verifi Email (Community Node)  
  - Role: Validates email addresses via Verifi Email API to prevent fraudulent or invalid leads.  
  - Config: Uses email from webhook data, requires API credential setup.  
  - Inputs: From Set User Config node.  
  - Outputs: Adds `valid` boolean flag to each lead.  
  - Version: Requires community node installation (`n8n-nodes-verifiemail`).  
  - Edge cases: API errors, rate limits, invalid email formats.

- **Filter Valid Emails**  
  - Type: If  
  - Role: Passes only leads with validated emails (`valid == true`) downstream.  
  - Config: Condition checks `$json.valid === true`.  
  - Inputs: From Validate Email.  
  - Outputs: Passes valid leads to IP Lookup, filters out invalids silently.  
  - Edge cases: Expression failures if `valid` is missing.

- **IP Lookup**  
  - Type: HTTP Request  
  - Role: Retrieves geolocation info for the IP address associated with the lead to assess risk and contextualize the lead.  
  - Config: Requests `https://ipapi.co/{{ip}}/json/` with IP from webhook JSON.  
  - Inputs: Filtered valid leads.  
  - Outputs: Appends IP geo-data for scoring.  
  - Edge cases: API downtime, IP missing or invalid.

- **Merge**  
  - Type: Merge  
  - Role: Combines IP Lookup and original webhook lead data into a single dataset for scoring.  
  - Inputs: IP Lookup and Capture Leads nodes output.  
  - Outputs: Merged lead data for scoring.  
  - Edge cases: Data misalignment if asynchronous or missing inputs.

---

#### 1.2 Lead Quality Scoring and Data Storage

**Overview:**  
This block calculates a lead quality score based on predefined business logic and appends valid, enriched lead data to a Google Sheet for record-keeping and further analysis.

**Nodes Involved:**  
- Lead Quality Score (Code)  
- Append row in sheet (Google Sheets)  
- High Score Check (If)

**Node Details:**

- **Lead Quality Score**  
  - Type: Code  
  - Role: Executes custom JavaScript to score leads based on email validity, region, and job role.  
  - Config:  
    - Scores +50 for valid email (assumed from validation).  
    - +30 if country is US or region is CA.  
    - +20 if job role matches decision makers like CEO, CTO, VP.  
    - Adds timestamp ISO string.  
  - Inputs: Merged data from Merge node.  
  - Outputs: Enriched lead JSON with `leadScore` and `timestamp`.  
  - Edge cases: Missing keys causing runtime errors, unexpected data formats.

- **Append row in sheet**  
  - Type: Google Sheets  
  - Role: Adds the scored lead record to a configured Google Sheet for persistent storage.  
  - Config:  
    - Maps lead fields (IP, Name, Email, Company, Industry, Job Role, Timestamp, Lead Score) to sheet columns.  
    - Uses OAuth2 credential for access.  
    - Targets a specific spreadsheet and sheet (ID: `15JAabnn5NBtiFom_EKkIMONKz-yiLZGvWz5y6YqNOwY`, sheet GID 0).  
  - Inputs: From Lead Quality Score node.  
  - Outputs: Passes lead data for filtering on score.  
  - Edge cases: Google Sheets API rate limits, credential expiration.

- **High Score Check**  
  - Type: If  
  - Role: Filters leads based on whether `Lead Score` exceeds a threshold (70 by default).  
  - Config: Condition `$json["Lead Score"] > 70`.  
  - Inputs: From Append row in sheet.  
  - Outputs:  
    - True branch triggers sales notifications.  
    - False branch leads to no operation (silently discard).  
  - Edge cases: Missing or non-numeric Lead Score causing false negatives.

---

#### 1.3 Notification and Lead Filtering

**Overview:**  
This block alerts the sales team on Slack about high-quality leads and prepares lead data for nurturing communications.

**Nodes Involved:**  
- Notify Sales Team (Slack)  
- No Operation, do nothing (NoOp)  
- Reformat for Email (Set)  
- Auto-Send Welcome Email (Gmail)  
- Track Email Opens (Set)  
- Delay 24 Hours (Wait)

**Node Details:**

- **Notify Sales Team**  
  - Type: Slack  
  - Role: Sends a message to a configured Slack channel notifying sales about a new high-quality lead.  
  - Config:  
    - Message includes Name, Email, Lead Score, IP, Timestamp.  
    - Posts to channel `#leads` (configurable).  
    - Uses Slack API credentials.  
  - Inputs: From High Score Check node (true branch).  
  - Outputs: Passes lead data to email reformatting.  
  - Edge cases: Slack API failures, channel permission issues.

- **No Operation, do nothing**  
  - Type: NoOp  
  - Role: Terminates flow for leads below threshold silently.  
  - Inputs: From High Score Check node (false branch).  
  - Outputs: None.

- **Reformat for Email**  
  - Type: Set  
  - Role: Extracts and restructures lead data from Slack notification text into separate fields for email sending (firstname, lastname, email, lead_score, ip).  
  - Config: Uses JavaScript expressions parsing Slack message text.  
  - Inputs: From Notify Sales Team.  
  - Outputs: Cleaned lead data for email sending.  
  - Edge cases: Parsing failures if Slack message format changes.

- **Auto-Send Welcome Email**  
  - Type: Gmail  
  - Role: Sends a personalized welcome email to the lead using extracted data.  
  - Config:  
    - Subject and message templates with dynamic placeholders.  
    - Uses Gmail OAuth2 credentials.  
  - Inputs: From Reformat for Email.  
  - Outputs: Triggers tracking of email opens.  
  - Edge cases: SMTP issues, invalid email addresses.

- **Track Email Opens**  
  - Type: Set  
  - Role: Initializes tracking variable `opened` to false to track if the email is opened.  
  - Inputs: From Auto-Send Welcome Email.  
  - Outputs: Leads into Delay node.  
  - Edge cases: None significant.

- **Delay 24 Hours**  
  - Type: Wait  
  - Role: Pauses execution for 24 hours before checking lead engagement.  
  - Inputs: From Track Email Opens.  
  - Outputs: Connects to engagement check.  
  - Edge cases: Workflow timeouts or interruptions.

---

#### 1.4 Lead Nurturing and Engagement Tracking

**Overview:**  
After initial contact, this block monitors lead engagement, sends automated follow-ups if needed, and calculates an engagement score.

**Nodes Involved:**  
- Check No Response (If)  
- Auto Follow-Up Email (Gmail)  
- Calculate Engagement Score (Code)

**Node Details:**

- **Check No Response**  
  - Type: If  
  - Role: Checks if the lead has opened the initial email (`opened == false`).  
  - Config: Condition checks `$json.opened === false`.  
  - Inputs: From Delay 24 Hours node.  
  - Outputs:  
    - False branch leads to no action.  
    - True branch triggers follow-up email.  
  - Edge cases: Missing or malformed `opened` flag.

- **Auto Follow-Up Email**  
  - Type: Gmail  
  - Role: Sends a polite follow-up email encouraging the lead to engage.  
  - Config:  
    - Uses dynamic placeholders for personalization.  
    - Uses Gmail OAuth2 credentials.  
  - Inputs: From Check No Response (true branch).  
  - Outputs: Passes leads into engagement scoring.  
  - Edge cases: Email sending failures.

- **Calculate Engagement Score**  
  - Type: Code  
  - Role: Assigns an engagement score based on email opens and potentially other metrics (expandable).  
  - Config: Adds 20 points if email opened; placeholder for future clicks/replies logic.  
  - Inputs: From Auto Follow-Up Email.  
  - Outputs: Leads enriched with `engagement_score`.  
  - Edge cases: Missing `opened` flag or unexpected data.

---

#### 1.5 Reporting

**Overview:**  
Generates a weekly report summarizing lead volume, quality, and engagement, then emails it to the sales head.

**Nodes Involved:**  
- Weekly Report Trigger (Schedule Trigger)  
- Generate Weekly Report (Code)  
- Send Report to Sales Head (Gmail)

**Node Details:**

- **Weekly Report Trigger**  
  - Type: Schedule Trigger  
  - Role: Initiates workflow every Monday at 00:00 (midnight).  
  - Config: Cron expression `0 0 * * 1`.  
  - Inputs: None (trigger).  
  - Outputs: Triggers report generation.  
  - Edge cases: Scheduler misconfiguration or downtime.

- **Generate Weekly Report**  
  - Type: Code  
  - Role: Aggregates lead data to compute total leads, high score leads, and average engagement score.  
  - Config: Uses input items to calculate summary metrics into a JSON report object.  
  - Inputs: Triggered by schedule, likely expects data feed or can be extended to query sheet.  
  - Outputs: JSON report.  
  - Edge cases: Empty datasets causing divide-by-zero.

- **Send Report to Sales Head**  
  - Type: Gmail  
  - Role: Emails the generated report to the configured sales head email address.  
  - Config:  
    - Dynamic subject with current date.  
    - Text body with summary statistics.  
    - Uses Gmail OAuth2 credentials.  
  - Inputs: From Generate Weekly Report.  
  - Outputs: None.  
  - Edge cases: Email delivery failures.

---

#### 1.6 Configuration and Utilities

**Overview:**  
This block includes sticky notes for documentation and instructions, and a no-op node to handle filtered conditions elegantly.

**Nodes Involved:**  
- Sticky Note (multiple)  
- No Operation, do nothing (NoOp)  

**Node Details:**

- **Sticky Notes**  
  - Type: Sticky Note  
  - Role: Provide detailed instructions on setup, configuration, and operational guidance.  
  - Content Highlights:  
    - Lead capture setup and webhook URL.  
    - Email validation node installation and credentials.  
    - Scoring logic customization.  
    - Slack notifications and Gmail email templates.  
    - Scheduling and report customization.  
  - Edge cases: None; purely informational.

- **No Operation, do nothing**  
  - Type: NoOp  
  - Role: Gracefully terminates the flow for leads that do not meet the high score threshold, preventing unnecessary downstream processing.  
  - Inputs: From High Score Check (false branch).  
  - Outputs: None.

---

### 3. Summary Table

| Node Name               | Node Type              | Functional Role                                 | Input Node(s)          | Output Node(s)                     | Sticky Note                                              |
|-------------------------|------------------------|------------------------------------------------|------------------------|----------------------------------|----------------------------------------------------------|
| Capture Leads           | Webhook                | Entry point to receive leads via HTTP POST     | External HTTP POST      | Merge, Set User Config            | See Sticky Note: Lead Capture and Filtering               |
| Set User Config         | Set                    | Holds user-configurable parameters              | Capture Leads          | Validate Email                   | See Sticky Note: Lead Capture and Filtering               |
| Validate Email          | Verifi Email           | Validates email addresses for legitimacy        | Set User Config        | Filter Valid Emails               | See Sticky Note: Verifi Email (Community node)            |
| Filter Valid Emails     | If                     | Filters leads with valid emails                  | Validate Email         | IP Lookup                       | See Sticky Note: Lead Capture and Filtering               |
| IP Lookup               | HTTP Request           | Retrieves geolocation data based on IP           | Filter Valid Emails    | Merge                          | See Sticky Note: Lead Capture and Filtering               |
| Merge                  | Merge                  | Combines IP data and webhook lead data           | IP Lookup, Capture Leads | Lead Quality Score              | See Sticky Note: Lead Capture and Filtering               |
| Lead Quality Score      | Code                   | Calculates lead quality score                     | Merge                  | Append row in sheet             | See Sticky Note: Scoring and Notification                  |
| Append row in sheet     | Google Sheets           | Stores lead data in Google Sheets                 | Lead Quality Score      | High Score Check               | See Sticky Note: Scoring and Notification                  |
| High Score Check        | If                     | Checks if lead score passes threshold             | Append row in sheet     | Notify Sales Team, No Operation | See Sticky Note: Scoring and Notification                  |
| Notify Sales Team       | Slack                  | Sends Slack notification for high-quality leads  | High Score Check (true) | Reformat for Email             | See Sticky Note: Scoring and Notification                  |
| No Operation, do nothing| NoOp                   | Ends flow silently for low-score leads            | High Score Check (false) | None                          |                                                          |
| Reformat for Email      | Set                    | Extracts and formats lead data for emails         | Notify Sales Team       | Auto-Send Welcome Email        | See Sticky Note: Nurturing                                 |
| Auto-Send Welcome Email | Gmail                  | Sends personalized welcome email to lead          | Reformat for Email      | Track Email Opens              | See Sticky Note: Nurturing                                 |
| Track Email Opens       | Set                    | Initializes email open tracking flag              | Auto-Send Welcome Email | Delay 24 Hours                | See Sticky Note: Nurturing                                 |
| Delay 24 Hours          | Wait                   | Waits 24 hours before checking engagement          | Track Email Opens       | Check No Response             | See Sticky Note: Delay                                     |
| Check No Response       | If                     | Checks if lead opened welcome email                | Delay 24 Hours          | Auto Follow-Up Email (true), None (false) | See Sticky Note: Nurturing                           |
| Auto Follow-Up Email    | Gmail                  | Sends follow-up email to unresponsive leads        | Check No Response (true) | Calculate Engagement Score    | See Sticky Note: Nurturing                                 |
| Calculate Engagement Score | Code                | Assigns engagement score based on email opens      | Auto Follow-Up Email    | None                         | See Sticky Note: Nurturing                                 |
| Weekly Report Trigger   | Schedule Trigger        | Triggers weekly report generation                   | None                   | Generate Weekly Report        | See Sticky Note: Reporting                                 |
| Generate Weekly Report  | Code                   | Creates summary report of leads and engagement     | Weekly Report Trigger   | Send Report to Sales Head      | See Sticky Note: Reporting                                 |
| Send Report to Sales Head | Gmail                 | Emails weekly report to sales head                  | Generate Weekly Report  | None                         | See Sticky Note: Reporting                                 |
| Sticky Note             | Sticky Note             | Documentation and instructions                      | None                   | None                         | Multiple notes scattered throughout the workflow          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node:**  
   - Type: Webhook  
   - Name: Capture Leads  
   - HTTP Method: POST  
   - Path: lead-capture  
   - This node receives incoming lead data.

2. **Create Set Node:**  
   - Type: Set  
   - Name: Set User Config  
   - Add variables:  
     - `score_threshold` (Number): 70  
     - `sales_channel` (String): "#leads"  
     - `sales_head_email` (String): "saleshead@acme.com"  
   - Connect output of Capture Leads to this node.

3. **Create Verifi Email Node:**  
   - Type: Verifi Email (community node)  
   - Name: Validate Email  
   - Configure email field with expression: `{{$json.body.email}}` from webhook data  
   - Setup Verifi Email API credentials with your API key  
   - Connect Set User Config node output here.

4. **Create If Node:**  
   - Type: If  
   - Name: Filter Valid Emails  
   - Condition: `$json.valid === true`  
   - Connect Validate Email node output here.

5. **Create HTTP Request Node:**  
   - Type: HTTP Request  
   - Name: IP Lookup  
   - Method: GET  
   - URL: `https://ipapi.co/{{$json.body.ip}}/json/`  
   - Connect Filter Valid Emails (true output) here.

6. **Create Merge Node:**  
   - Type: Merge  
   - Name: Merge  
   - Connect IP Lookup output to first input  
   - Connect Capture Leads output to second input  
   - Configure to merge inputs by position or data keys.

7. **Create Code Node:**  
   - Type: Code  
   - Name: Lead Quality Score  
   - Paste scoring JavaScript logic:  
     - Score +50 for validated email  
     - +30 if country_code US or region_code CA  
     - +20 if jobRole is CEO, CTO, or VP (case-insensitive)  
     - Add ISO timestamp  
   - Connect Merge output here.

8. **Create Google Sheets Node:**  
   - Type: Google Sheets  
   - Name: Append row in sheet  
   - Operation: Append  
   - Document ID and Sheet Name: Use your Google Sheet credentials and target sheet  
   - Map columns: IP, Name, Email, Company, Industry, Job Role, Timestamp, Lead Score  
   - Connect Lead Quality Score output here and set Google Sheets OAuth2 credentials.

9. **Create If Node:**  
   - Type: If  
   - Name: High Score Check  
   - Condition: `$json["Lead Score"] > 70`  
   - Connect Append row in sheet output here.

10. **Create Slack Node:**  
    - Type: Slack  
    - Name: Notify Sales Team  
    - Configure text message with lead details  
    - Channel: Use `#leads` or configured sales_channel  
    - Connect High Score Check (true output) here  
    - Set Slack API credentials.

11. **Create NoOp Node:**  
    - Type: NoOp  
    - Name: No Operation, do nothing  
    - Connect High Score Check (false output) here.

12. **Create Set Node:**  
    - Type: Set  
    - Name: Reformat for Email  
    - Extract email, firstname, lastname, lead_score, and ip from Slack message text using JavaScript expressions  
    - Connect Notify Sales Team output here.

13. **Create Gmail Node:**  
    - Type: Gmail  
    - Name: Auto-Send Welcome Email  
    - Configure recipient with extracted email field  
    - Subject and message with placeholders for firstname and lead_score  
    - Connect Reformat for Email output here  
    - Setup Gmail OAuth2 credentials.

14. **Create Set Node:**  
    - Type: Set  
    - Name: Track Email Opens  
    - Initialize `opened` flag to false  
    - Connect Auto-Send Welcome Email output here.

15. **Create Wait Node:**  
    - Type: Wait  
    - Name: Delay 24 Hours  
    - Duration: 24 hours  
    - Connect Track Email Opens output here.

16. **Create If Node:**  
    - Type: If  
    - Name: Check No Response  
    - Condition: `$json.opened === false`  
    - Connect Delay 24 Hours output here.

17. **Create Gmail Node:**  
    - Type: Gmail  
    - Name: Auto Follow-Up Email  
    - Send to extracted email  
    - Subject and message with firstname placeholder  
    - Connect Check No Response (true output) here  
    - Use Gmail OAuth2 credentials.

18. **Create Code Node:**  
    - Type: Code  
    - Name: Calculate Engagement Score  
    - Add 20 points if `opened` is true; placeholder for further engagement metrics  
    - Connect Auto Follow-Up Email output here.

19. **Create Schedule Trigger Node:**  
    - Type: Schedule Trigger  
    - Name: Weekly Report Trigger  
    - Cron expression: `0 0 * * 1` (every Monday at midnight)

20. **Create Code Node:**  
    - Type: Code  
    - Name: Generate Weekly Report  
    - Calculate total leads, number of high score leads (≥70), average engagement score  
    - Connect Weekly Report Trigger output here.

21. **Create Gmail Node:**  
    - Type: Gmail  
    - Name: Send Report to Sales Head  
    - Recipient: sales head email from config  
    - Subject: Weekly Sales Performance Report with current date  
    - Body: Summary stats from report  
    - Connect Generate Weekly Report output here  
    - Use Gmail OAuth2 credentials.

22. **Add Sticky Notes:**  
    - Add detailed sticky notes for each block with setup instructions, credential requirements, and customization tips as in original.

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                                                                                   |
|----------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This workflow automates lead capture, fraud-proofing with email validation, IP geolocation, quality scoring, Slack notifications, nurturing, engagement tracking, and weekly reporting. | Sticky Note4 (Main Overview)                                                                      |
| Lead capture webhook URL format: `https://[your-n8n-url]/webhook/lead-capture`.                                      | Sticky Note (Lead Capture and Filtering)                                                        |
| Verifi Email node is a community node requiring installation: `npm install n8n-nodes-verifi-email`.                   | Sticky Note6 (Verifi Email)                                                                      |
| Customize lead scoring logic in the Lead Quality Score function node to fit your business needs.                      | Sticky Note1 (Scoring and Notification)                                                         |
| Slack notifications require Slack API credentials and appropriate channel permissions.                                | Sticky Note1 (Scoring and Notification)                                                         |
| Gmail nodes require OAuth2 credentials; personalize email messages in Auto-Send Welcome Email and Auto Follow-Up Email nodes. | Sticky Note2 (Nurturing)                                                                         |
| Weekly report is triggered every Monday at 00:00; customize sales head email address in Set User Config node.         | Sticky Note3 and Sticky Note5 (Reporting and Delay)                                             |

---

**Disclaimer:**  
The text provided is exclusively derived from a workflow automated with n8n, an integration and automation tool. All content complies with applicable content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.