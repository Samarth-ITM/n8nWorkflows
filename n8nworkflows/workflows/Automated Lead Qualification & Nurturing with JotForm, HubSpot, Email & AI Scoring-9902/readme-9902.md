Automated Lead Qualification & Nurturing with JotForm, HubSpot, Email & AI Scoring

https://n8nworkflows.xyz/workflows/automated-lead-qualification---nurturing-with-jotform--hubspot--email---ai-scoring-9902


# Automated Lead Qualification & Nurturing with JotForm, HubSpot, Email & AI Scoring

### 1. Workflow Overview

This workflow automates lead qualification and nurturing using data submitted via JotForm. It processes incoming lead data, applies an AI-driven scoring system based on email domain, company size, budget, and timeline to categorize leads into tiers (Hot, Warm, Cold). Depending on the lead tier, it routes leads for appropriate follow-up: hot and warm leads trigger sales notifications and personalized emails, while cold leads notify marketing for nurturing. All leads are logged in HubSpot CRM and Google Sheets for tracking, and daily summaries are generated.

**Logical Blocks:**

- **1.1 Input Reception:** Receive and format incoming lead data from JotForm submissions.
- **1.2 AI Lead Scoring:** Apply scoring logic to evaluate lead quality and assign lead tiers.
- **1.3 Lead Routing:** Route leads based on tier to sales or marketing.
- **1.4 CRM and Data Logging:** Add leads to HubSpot CRM and log details in Google Sheets.
- **1.5 Notifications:** Notify sales via Slack for hot leads and marketing for warm/cold leads.
- **1.6 Personalized Email Generation & Sending:** Create and send tailored emails based on lead tier.
- **1.7 Daily Summary Creation:** Aggregate daily lead statistics for reporting.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** Captures lead submission data from JotForm webhook and formats the data for processing.
- **Nodes Involved:**
  - JotForm Trigger
  - Extract & Format Lead Data

- **Node Details:**

  - **JotForm Trigger**
    - Type: Trigger node for JotForm submissions.
    - Configuration: Listens to submissions via webhook with predefined webhook ID.
    - Inputs: None (trigger node).
    - Outputs: Raw lead submission data.
    - Failures: Webhook misconfiguration, connectivity issues.
    - Version-specific: n8n supports JotForm trigger in standard versions.
  
  - **Extract & Format Lead Data**
    - Type: Set node (data transformation).
    - Configuration: Combines and formats incoming JSON data into a structured format for scoring.
    - Key Usage: Extracts fields like email, company size, budget, timeline, etc.
    - Inputs: JotForm Trigger output.
    - Outputs: Cleaned and formatted lead data.
    - Failures: Missing expected fields, malformed data.
    - Version-specific: Uses version 3.3 for set node.

#### 1.2 AI Lead Scoring

- **Overview:** Computes a lead score and tier based on multiple criteria using custom JavaScript code.
- **Nodes Involved:**
  - AI Lead Scoring

- **Node Details:**

  - **AI Lead Scoring**
    - Type: Code node executing JavaScript.
    - Configuration: Custom scoring algorithm:
      - Email domain: distinguishes business vs personal emails (+25 points for business).
      - Company size: large (+30), medium (+20), small (+10).
      - Budget: high (+25), medium (+15), low (+5).
      - Timeline urgency: urgent (+20), medium (+10), long (+5).
    - Output: Adds `leadScore`, `leadTier` (Hot/Warm/Cold), `qualificationNotes`, and timestamp.
    - Inputs: Formatted lead data.
    - Outputs: Scored lead data.
    - Failures: JavaScript runtime errors, missing data keys.
    - Version-specific: Code node version 2.
  
#### 1.3 Lead Routing

- **Overview:** Routes leads based on the computed lead tier to appropriate follow-up actions.
- **Nodes Involved:**
  - Route by Lead Quality

- **Node Details:**

  - **Route by Lead Quality**
    - Type: If node (conditional routing).
    - Configuration: Checks if leadTier is “Hot” or “Warm” to route accordingly.
    - Inputs: AI Lead Scoring output.
    - Outputs:
      - True branch: Hot or Warm leads.
      - False branch: Cold leads.
    - Failures: Expression evaluation errors.
    - Version-specific: Version 2.

#### 1.4 CRM and Data Logging

- **Overview:** Adds all leads to HubSpot CRM and appends or updates lead data in Google Sheets.
- **Nodes Involved:**
  - Add to HubSpot CRM
  - Log to Google Sheets

- **Node Details:**

  - **Add to HubSpot CRM**
    - Type: HubSpot node.
    - Configuration: Creates a new contact record using lead data.
    - Inputs: Branches from Route by Lead Quality.
    - Outputs: Passes data downstream for summary creation.
    - Failures: HubSpot API errors, authentication failures.
    - Version-specific: Version 2.
    - Credentials: Requires HubSpot API credentials.

  - **Log to Google Sheets**
    - Type: Google Sheets node.
    - Configuration: Appends or updates a row in a specified Google Sheet with detailed lead info (name, email, phone, budget, source, status, tier, timestamp, score, company size).
    - Inputs: Route by Lead Quality output.
    - Outputs: None further in this branch.
    - Failures: Google Sheets API errors, permission issues.
    - Version-specific: Version 4.4.
    - Credentials: Requires Google Sheets OAuth2 credentials.
    - Notes: Spreadsheet Id and sheet name configured; placeholder “your-spreadsheet-id” to be replaced.

#### 1.5 Notifications

- **Overview:** Sends Slack notifications for hot leads to sales and for warm/cold leads to marketing.
- **Nodes Involved:**
  - Notify Sales Team (Hot Lead)
  - Notify Marketing (Warm/Cold)

- **Node Details:**

  - **Notify Sales Team (Hot Lead)**
    - Type: Slack node.
    - Configuration: Sends a formatted Slack message with lead details and urgent follow-up request.
    - Inputs: Hot/Warm leads branch from Route by Lead Quality (true branch specifically for hot leads).
    - Outputs: None further.
    - Failures: Slack webhook invalid, network issues.
    - Version-specific: Version 2.1.
    - Credentials: Slack webhook URL configuration.

  - **Notify Marketing (Warm/Cold)**
    - Type: Slack node.
    - Configuration: Sends a notification for new leads categorized as warm or cold to marketing.
    - Inputs: False branch from Route by Lead Quality.
    - Outputs: None further.
    - Failures: Slack webhook errors.
    - Version-specific: Version 2.1.
    - Credentials: Slack webhook URL configuration.

#### 1.6 Personalized Email Generation & Sending

- **Overview:** Generates personalized email content based on lead tier and sends the email.
- **Nodes Involved:**
  - Generate Personalized Email
  - Send Personalized Email

- **Node Details:**

  - **Generate Personalized Email**
    - Type: Code node.
    - Configuration: Uses JavaScript to create customized email subject and body:
      - Hot leads get a direct follow-up email proposing a demo and call.
      - Warm/Cold leads receive nurturing content with resources and calendar link.
    - Inputs: Route by Lead Quality output.
    - Outputs: Lead data enriched with `emailSubject` and `emailBody`.
    - Failures: JavaScript errors, missing data fields.
    - Version-specific: Version 2.

  - **Send Personalized Email**
    - Type: Email Send node.
    - Configuration: Sends email using SMTP with dynamic subject and recipient from node inputs.
    - Inputs: Email content generated prior.
    - Outputs: Triggers daily summary node after email sent.
    - Failures: SMTP connection/authentication issues, invalid email addresses.
    - Version-specific: Version 2.1.
    - Credentials: SMTP credentials required.
    - Parameters: Reply-to set as sales@yourcompany.com, from email same.

#### 1.7 Daily Summary Creation

- **Overview:** Compiles and sets a daily summary text of leads processed, split by tiers and average score.
- **Nodes Involved:**
  - Create Daily Summary

- **Node Details:**

  - **Create Daily Summary**
    - Type: Set node.
    - Configuration: Assigns a string summarizing total leads, counts per tier, and average lead score.
    - Inputs: Output from Add to HubSpot CRM and Send Personalized Email nodes.
    - Outputs: Final summary object (presumably for reporting or logging).
    - Failures: Missing count variables, timing issues.
    - Version-specific: Version 3.3.

#### Miscellaneous

- **Sticky Note**
  - Provides a comprehensive overview and setup instructions for the workflow.
  - Includes author credit and a LinkedIn link.

---

### 3. Summary Table

| Node Name                  | Node Type          | Functional Role                          | Input Node(s)                 | Output Node(s)                                 | Sticky Note                                                                                                         |
|----------------------------|--------------------|----------------------------------------|------------------------------|-----------------------------------------------|---------------------------------------------------------------------------------------------------------------------|
| JotForm Trigger            | JotForm Trigger    | Receive lead submission via webhook    | None                         | Extract & Format Lead Data                     | # 🎯 Lead Qualification & Nurturing System  Auto-captures JotForm leads, scores them by AI, and sends hot leads to sales, others to marketing. All leads go to HubSpot and Google Sheets, with Slack alerts and auto follow-up email. Scoring based on email, company size, budget, and timeline. Setup: Connect JotForm, HubSpot, Sheets, Slack, SMTP, map fields, adjust logic & templates. Results: Respond faster, convert more, eliminate manual entry. Built by Daniel Shashko [LinkedIn](https://www.linkedin.com/in/daniel-shashko/) |
| Extract & Format Lead Data | Set                | Format and prepare lead data           | JotForm Trigger              | AI Lead Scoring                               |                                                                                                                     |
| AI Lead Scoring            | Code               | Score leads and assign lead tier       | Extract & Format Lead Data   | Route by Lead Quality                         |                                                                                                                     |
| Route by Lead Quality      | If                 | Route leads by tier (Hot/Warm/Cold)    | AI Lead Scoring              | Add to HubSpot CRM, Log to Google Sheets, Notify Sales, Generate Personalized Email (Hot/Warm branch); Notify Marketing, Add to HubSpot CRM, Log to Google Sheets, Generate Personalized Email (Cold branch) |                                                                                                                     |
| Add to HubSpot CRM         | HubSpot             | Add lead to CRM                        | Route by Lead Quality        | Create Daily Summary (via Send Personalized Email) |                                                                                                                     |
| Log to Google Sheets       | Google Sheets       | Log lead data into spreadsheet         | Route by Lead Quality        | None                                          |                                                                                                                     |
| Notify Sales Team (Hot Lead)| Slack              | Notify sales team of hot leads          | Route by Lead Quality (Hot)  | None                                          |                                                                                                                     |
| Notify Marketing (Warm/Cold)| Slack              | Notify marketing team of warm/cold leads| Route by Lead Quality (Cold) | None                                          |                                                                                                                     |
| Generate Personalized Email| Code               | Create customized email content         | Route by Lead Quality        | Send Personalized Email                       |                                                                                                                     |
| Send Personalized Email    | Email Send          | Send personalized email to lead         | Generate Personalized Email  | Create Daily Summary                           |                                                                                                                     |
| Create Daily Summary       | Set                 | Assemble daily lead summary text        | Add to HubSpot CRM, Send Personalized Email | None                                         |                                                                                                                     |
| Sticky Note                | Sticky Note         | Documentation and overview              | None                        | None                                          | # 🎯 Lead Qualification & Nurturing System ... Built by Daniel Shashko [LinkedIn](https://www.linkedin.com/in/daniel-shashko/) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create JotForm Trigger Node**
   - Type: JotForm Trigger
   - Configuration: Set webhook ID and enable to listen for new form submissions.
   - No input nodes.
   - Credentials: Connect your JotForm account.

2. **Add “Extract & Format Lead Data” Set Node**
   - Type: Set
   - Connect input from JotForm Trigger.
   - Configure to extract and combine relevant fields (email, firstName, lastName, companySize, budget, timeline, phone, company).
   - Ensure fields are mapped consistently.

3. **Add “AI Lead Scoring” Code Node**
   - Type: Code
   - Connect input from Extract & Format Lead Data node.
   - Paste the JavaScript code that:
     - Scores leads by email domain, company size, budget, timeline.
     - Assigns leadTier as Hot, Warm, or Cold.
     - Adds qualificationNotes and timestamp.
   - Use Node version 2 or later.

4. **Add “Route by Lead Quality” If Node**
   - Type: If
   - Connect input from AI Lead Scoring.
   - Set condition to check if `leadTier` equals “Hot” or “Warm”.
   - True branch for Hot/Warm leads; False branch for Cold leads.

5. **Add “Add to HubSpot CRM” Node**
   - Type: HubSpot
   - Connect inputs from both True and False branches of the Route node.
   - Configure to create new contact with lead data.
   - Credentials: Configure HubSpot OAuth2 credentials.

6. **Add “Log to Google Sheets” Node**
   - Type: Google Sheets
   - Connect inputs from both Route node branches.
   - Configure to append or update sheet rows with lead details.
   - Set spreadsheet ID and sheet name.
   - Credentials: Google Sheets OAuth2 credentials required.

7. **Add “Notify Sales Team (Hot Lead)” Slack Node**
   - Type: Slack
   - Connect input from Route node True branch filtered for Hot leads.
   - Configure Slack webhook URL.
   - Message includes lead details and urgent follow-up note.

8. **Add “Notify Marketing (Warm/Cold)” Slack Node**
   - Type: Slack
   - Connect input from Route node False branch (Cold leads).
   - Configure Slack webhook URL.
   - Message includes lead details and nurture sequence note.

9. **Add “Generate Personalized Email” Code Node**
   - Type: Code
   - Connect inputs from Route node outputs.
   - Paste JavaScript to generate email subject and body based on leadTier.
   - Use Node version 2 or later.

10. **Add “Send Personalized Email” Email Send Node**
    - Type: Email Send
    - Connect input from Generate Personalized Email.
    - Configure SMTP credentials.
    - Dynamic subject and To email address based on previous node output.
    - Set From and Reply-To emails (e.g., sales@yourcompany.com).

11. **Add “Create Daily Summary” Set Node**
    - Type: Set
    - Connect input from Add to HubSpot CRM and Send Personalized Email nodes.
    - Configure to build a summary string with counts of hot, warm, cold leads and average score.

12. **Add Sticky Note for Documentation**
    - Type: Sticky Note
    - Add overview content explaining workflow purpose, scoring criteria, setup steps, and author credit.

**Additional Setup Notes:**
- Replace placeholders such as spreadsheet ID and calendar link with actual values.
- Ensure credentials for JotForm, HubSpot, Google Sheets, Slack, and SMTP are properly set and tested.
- Adjust scoring thresholds and email templates as needed.
- Test webhook connectivity and data field mappings carefully.

---

### 5. General Notes & Resources

| Note Content                                                                                                    | Context or Link                                                                                  |
|-----------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Workflow built by Daniel Shashko, LinkedIn: [https://www.linkedin.com/in/daniel-shashko/](https://www.linkedin.com/in/daniel-shashko/) | Author credit and professional profile link                                                    |
| Scoring logic weights email domain, company size, budget, and timeline to prioritize leads effectively.          | Important to customize scoring to business needs                                               |
| Ensure all API credentials (HubSpot, Google Sheets, Slack, SMTP) are configured with proper scopes and tokens. | Credential setup critical for seamless operation                                               |
| Slack notifications use incoming webhook URLs for channel-specific alerts.                                      | Create appropriate Slack webhooks per team                                                     |
| Google Sheets node requires spreadsheet ID and sheet name; update “your-spreadsheet-id” placeholder.            | Spreadsheet must have columns matching mapped fields                                           |
| Email templates include personalization and call-to-action for hot leads, and nurturing content for others.    | Modify templates to align with your sales/marketing messaging                                  |
| Daily summary node aggregates lead metrics for management reporting or dashboard integration.                   | Further integration possible with Slack or email summary delivery                              |

---

*Disclaimer: The provided text is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.*