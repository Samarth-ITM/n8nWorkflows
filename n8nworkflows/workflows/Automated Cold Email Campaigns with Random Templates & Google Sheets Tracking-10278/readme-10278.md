Automated Cold Email Campaigns with Random Templates & Google Sheets Tracking

https://n8nworkflows.xyz/workflows/automated-cold-email-campaigns-with-random-templates---google-sheets-tracking-10278


# Automated Cold Email Campaigns with Random Templates & Google Sheets Tracking

### 1. Workflow Overview

This workflow automates cold email campaigns by sending personalized emails to leads listed in a Google Sheet, using random email templates also stored in Google Sheets, and tracks the sending status back in the leads sheet.  
It targets marketing or sales teams aiming to conduct large-scale outreach with dynamic personalization and automated status updates, minimizing manual effort.

The workflow is organized into the following logical blocks:

- **1.1 Input Reception**  
  Manual trigger to start the process.

- **1.2 Leads Data Retrieval and Filtering**  
  Fetch leads from a Google Sheet and filter those who have not yet been sent emails.

- **1.3 Loop Control**  
  Process leads one by one.

- **1.4 Templates Retrieval and Random Selection**  
  Fetch email templates from a Google Sheet and pick one at random.

- **1.5 Personalization and Email Preparation**  
  Replace placeholders with lead-specific data, format the email body as HTML.

- **1.6 Email Sending**  
  Send personalized emails using SMTP credentials.

- **1.7 Status Logging and Loop Wait**  
  Update Google Sheet with sending status and wait a configurable time before sending the next email.

- **1.8 Error Handling and Guidance**  
  Several sticky notes guide setup, common issues, and recommended practices.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Starts the workflow execution manually when testing or running the campaign.

- **Nodes Involved:**  
  - `When clicking ‘Test workflow’`

- **Node Details:**  
  - Type: Manual Trigger  
  - Configuration: Default manual trigger, no parameters  
  - Inputs: None  
  - Outputs: Connects to `Google Sheets` (Leads retrieval)  
  - Edge cases: None  
  - Notes: Used to initiate the campaign manually for testing or live runs.

---

#### 2.2 Leads Data Retrieval and Filtering

- **Overview:**  
  Retrieves all leads from the designated Google Sheet and filters out those who have already been sent emails (status "SENT").

- **Nodes Involved:**  
  - `Google Sheets` (Leads)  
  - `If`

- **Node Details:**  
  - `Google Sheets` (Leads)  
    - Type: Google Sheets  
    - Role: Reads the Leads sheet from a Google Sheets document.  
    - Configuration: Reads from sheet "Sheet1" (ID: 436315764) in the document with ID `1EP82H05QzVy4JPHVvz06pO09p-V7M8_6-6O1jRAigN0`.  
    - Credentials: Google Sheets OAuth2 account.  
    - Inputs: Trigger from manual trigger node  
    - Outputs: To `If` node  
    - Edge cases: Authentication errors, empty sheet, API limits.

  - `If`  
    - Type: Conditional node  
    - Role: Filters out leads where "Send Status" equals "SENT". Only leads without "SENT" status proceed.  
    - Configuration: Condition checks if `Send Status` ≠ "SENT" (case sensitive, strict type validation).  
    - Inputs: From `Google Sheets`  
    - Outputs: True branch to `Loop Over Items` (process leads), false branch ignored.  
    - Edge cases: Missing or misspelled "Send Status" field, inconsistent data casing.

---

#### 2.3 Loop Control

- **Overview:**  
  Processes each lead individually by splitting the input into batches of one item to handle personalized sending.

- **Nodes Involved:**  
  - `Loop Over Items`

- **Node Details:**  
  - Type: SplitInBatches  
  - Role: Splits incoming lead list into batches (default batch size = 1) for sequential processing.  
  - Configuration: Default options, processes one lead at a time.  
  - Inputs: From `If` node (filtered leads)  
  - Outputs:  
    - True branch (empty) — no output  
    - False branch to `Google Sheets1` (Templates retrieval) and `Merge`  
  - Edge cases: Infinite loops if batch size or connections are misconfigured.

---

#### 2.4 Templates Retrieval and Random Selection

- **Overview:**  
  Retrieves email templates from Google Sheets and selects one randomly for each lead.

- **Nodes Involved:**  
  - `Google Sheets1` (Templates)  
  - `Code`  
  - `Merge`

- **Node Details:**  
  - `Google Sheets1` (Templates)  
    - Type: Google Sheets  
    - Role: Reads available email templates from a separate sheet "Sheet2" (ID: 1056380010) in the same document.  
    - Configuration: Reads all templates with fields "Subject" and "Body".  
    - Credentials: Same Google Sheets OAuth2 account.  
    - Inputs: From `Loop Over Items` node (false branch)  
    - Outputs: To `Code` node  
    - Edge cases: Empty templates, authentication issues.

  - `Code`  
    - Type: Code (JavaScript)  
    - Role: Picks one random template from input items, converts line breaks in the body to HTML `<br>` tags, returns subject and HTML body.  
    - Key expressions:  
      - Picks random index: `Math.floor(Math.random() * templates.length)`  
      - Converts body: `template.Body.replace(/\n/g, '<br>')`  
    - Inputs: Templates from `Google Sheets1`  
    - Outputs: Object with `subject` and `body` fields  
    - Edge cases: No templates found — throws error.

  - `Merge`  
    - Type: Merge node  
    - Role: Combines the single lead data (from `Loop Over Items` batch) with the randomly selected template data by position (index-based).  
    - Configuration: Combine mode, combines by position.  
    - Inputs: Lead data and template data  
    - Outputs: To `Edit Fields` node  
    - Edge cases: Mismatched input array lengths.

---

#### 2.5 Personalization and Email Preparation

- **Overview:**  
  Personalizes the selected template by replacing `[Name]` placeholders with the lead’s actual name or a fallback ("there"), preparing the final subject and body.

- **Nodes Involved:**  
  - `Edit Fields`

- **Node Details:**  
  - Type: Set node  
  - Role: Performs string replacements on `subject` and `body` fields, substituting `[Name]` with lead's `Name` field or `"there"` if empty.  
  - Configuration:  
    - subject: `={{ $json.subject.replace("[Name]", $json.Name || "there") }}`  
    - body: `={{ $json.body.replace("[Name]", $json.Name || "there") }}`  
    - Includes all other fields unchanged.  
  - Inputs: From `Merge` node  
  - Outputs: To `Send email` and `Merge1` nodes  
  - Edge cases: Missing `Name` field, template without `[Name]`.

---

#### 2.6 Email Sending

- **Overview:**  
  Sends the personalized email using SMTP credentials.

- **Nodes Involved:**  
  - `Send email`

- **Node Details:**  
  - Type: Email Send  
  - Role: Sends email with configured SMTP server, using personalized subject and HTML body.  
  - Configuration:  
    - From email: `anir@agramprojects.com` (must be set by user)  
    - To email: Lead’s `Email` field  
    - Subject: Personalized subject from `Edit Fields` node  
    - HTML body: Personalized email body  
    - SMTP credentials required  
    - Disables attribution footer (`appendAttribution: false`)  
  - Inputs: From `Edit Fields` (main output)  
  - Outputs: To `Wait` node (main output) and `Merge1` (second output)  
  - Edge cases: SMTP authentication failures, invalid email addresses, network issues, spam filtering.

---

#### 2.7 Status Logging and Loop Wait

- **Overview:**  
  Logs the sending status back to the Google Sheet and waits before processing the next lead to avoid spam flags.

- **Nodes Involved:**  
  - `Merge1`  
  - `Google Sheets6` (Log Status)  
  - `Wait`

- **Node Details:**  
  - `Merge1`  
    - Type: Merge node  
    - Role: Combines data from `Edit Fields` and `Send email` outputs by position to synchronize lead and send status data.  
    - Configuration: Combine mode, combine by position.  
    - Inputs: From `Edit Fields` and `Send email` (second output)  
    - Outputs: To `Google Sheets6`  
    - Edge cases: Data synchronization issues.

  - `Google Sheets6` (Log Status)  
    - Type: Google Sheets  
    - Role: Appends or updates lead's row in Leads sheet with current time, email, and send status (e.g., "SENT").  
    - Configuration:  
      - Sheet: "Sheet1" (Leads sheet)  
      - Matching by "Email" column  
      - Columns updated: Time (localized to "Africa/Casablanca" timezone, 24h format), Email, Send Status (from send email output labelIds[0])  
    - Credentials: Google Sheets OAuth2  
    - Inputs: From `Merge1`  
    - Outputs: None  
    - Edge cases: API quota limits, mismatched emails, update conflicts.

  - `Wait`  
    - Type: Wait node  
    - Role: Pauses workflow after sending each email to avoid spam detection and respect sending limits.  
    - Configuration: Default is instant, user must configure delay (recommended 30-120 seconds).  
    - Inputs: From `Send email` (main output)  
    - Outputs: Loops back to `Loop Over Items` node to process next lead  
    - Edge cases: No delay set may cause spam issues; infinite loop if misconfigured.

---

#### 2.8 Error Handling and Guidance (Sticky Notes)

- **Overview:**  
  Sticky notes provide structured setup instructions, troubleshooting tips, and important warnings.

- **Nodes Involved:**  
  - `Sticky Note` (Get Leads Data From Google Sheet)  
  - `Sticky Note1` (Get Email Templates From Google Sheet)  
  - `Sticky Note2` (Log The Status)  
  - `Sticky Note3` (Setup Checklist)  
  - `Sticky Note4` (Common First-Time Issues)  
  - `Sticky Note5` (What to Test Before Running Campaign)  
  - `Sticky Note6` (Step-by-Step Setup)  
  - `Sticky Note7` (Gmail SMTP Does Not Work!)

- **Node Details:**  
  - These nodes contain detailed instructions and tips for users to properly configure the workflow, avoid common errors such as SMTP failures or duplicate sends, and guidance on recommended SMTP providers.  
  - They do not affect workflow execution and have no inputs/outputs.

---

### 3. Summary Table

| Node Name                   | Node Type                   | Functional Role                           | Input Node(s)                 | Output Node(s)                | Sticky Note                                                                                                  |
|-----------------------------|-----------------------------|-----------------------------------------|------------------------------|------------------------------|--------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’| Manual Trigger              | Start workflow execution manually       | None                         | Google Sheets (Leads)         |                                                                                                              |
| Google Sheets               | Google Sheets               | Retrieve leads data                      | When clicking ‘Test workflow’| If                           | Get Leads Data From Google Sheet                                                                             |
| If                         | If                         | Filter leads not yet sent                | Google Sheets                | Loop Over Items               |                                                                                                              |
| Loop Over Items            | SplitInBatches             | Process leads one-by-one                  | If                           | Google Sheets1, Merge         |                                                                                                              |
| Google Sheets1             | Google Sheets               | Retrieve email templates                 | Loop Over Items              | Code                         | Get Email Templates From Google Sheet                                                                        |
| Code                       | Code                       | Pick random template & format body HTML | Google Sheets1               | Merge                        |                                                                                                              |
| Merge                      | Merge                      | Combine lead data and template           | Loop Over Items, Code        | Edit Fields                  |                                                                                                              |
| Edit Fields                | Set                        | Personalize subject and body             | Merge                        | Merge1, Send email            |                                                                                                              |
| Send email                 | Email Send                 | Send personalized email                   | Edit Fields                  | Wait, Merge1                 |                                                                                                              |
| Wait                       | Wait                       | Wait before next email                    | Send email                  | Loop Over Items              |                                                                                                              |
| Merge1                     | Merge                      | Combine email send data and lead info    | Edit Fields, Send email      | Google Sheets6               |                                                                                                              |
| Google Sheets6             | Google Sheets               | Log send status in leads sheet            | Merge1                       | None                         | Log The Status                                                                                                |
| Sticky Note                | Sticky Note                 | Setup guidance                           | None                         | None                         | Get Leads Data From Google Sheet                                                                             |
| Sticky Note1               | Sticky Note                 | Setup guidance                           | None                         | None                         | Get Email Templates From Google Sheet                                                                        |
| Sticky Note2               | Sticky Note                 | Setup guidance                           | None                         | None                         | Log The Status                                                                                                |
| Sticky Note3               | Sticky Note                 | Setup checklist                          | None                         | None                         | ⚙️ SETUP CHECKLIST (multi-step setup guidance)                                                               |
| Sticky Note4               | Sticky Note                 | Troubleshooting tips                     | None                         | None                         | Common First-Time Issues                                                                                       |
| Sticky Note5               | Sticky Note                 | Testing tips                            | None                         | None                         | What to Test Before Running Campaign                                                                          |
| Sticky Note6               | Sticky Note                 | Step-by-step setup instructions         | None                         | None                         | Step-by-Step Setup                                                                                            |
| Sticky Note7               | Sticky Note                 | Warning about Gmail SMTP limitations     | None                         | None                         | 🚫 GMAIL SMTP DOES NOT WORK! Recommended SMTP providers listed                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: `When clicking ‘Test workflow’`  
   - Type: Manual Trigger  
   - No parameters.

2. **Create Google Sheets Node for Leads**  
   - Name: `Google Sheets`  
   - Type: Google Sheets (v4.6)  
   - Credentials: Google Sheets OAuth2 connected account  
   - Operation: Read rows from sheet "Sheet1" of document ID `1EP82H05QzVy4JPHVvz06pO09p-V7M8_6-6O1jRAigN0` (your leads sheet)  
   - Connect output of manual trigger to this node.

3. **Create If Node for Filtering Sent Leads**  
   - Name: `If`  
   - Type: If (v2.2)  
   - Condition: `Send Status` field string not equals `"SENT"` (case-sensitive, strict)  
   - Connect `Google Sheets` output to `If`.

4. **Create SplitInBatches Node**  
   - Name: `Loop Over Items`  
   - Type: SplitInBatches (v3)  
   - Default batch size (1)  
   - Connect `If` true output to `Loop Over Items`.

5. **Create Google Sheets Node for Email Templates**  
   - Name: `Google Sheets1`  
   - Type: Google Sheets (v4.6)  
   - Credentials: Same Google Sheets OAuth2 account  
   - Operation: Read rows from sheet "Sheet2" of same document (your templates sheet)  
   - Connect `Loop Over Items` “false” output to this node.

6. **Create Code Node to Pick Random Template**  
   - Name: `Code`  
   - Type: Code (JavaScript) (v2)  
   - JavaScript:  
     ```javascript
     const templates = items.map(item => item.json);
     if (!templates || templates.length === 0) {
       throw new Error('No templates found');
     }
     const index = Math.floor(Math.random() * templates.length);
     const template = templates[index];
     const bodyHtml = template.Body.replace(/\n/g, '<br>');
     return [{ json: { subject: template.Subject, body: bodyHtml } }];
     ```  
   - Connect `Google Sheets1` output to `Code`.

7. **Create Merge Node to Combine Lead and Template**  
   - Name: `Merge`  
   - Type: Merge (v3.1)  
   - Mode: Combine by position  
   - Connect `Loop Over Items` (main output) and `Code` to `Merge`.

8. **Create Set Node for Personalization**  
   - Name: `Edit Fields`  
   - Type: Set (v3.4)  
   - Assignments:  
     - `subject`: `={{ $json.subject.replace("[Name]", $json.Name || "there") }}`  
     - `body`: `={{ $json.body.replace("[Name]", $json.Name || "there") }}`  
   - Include other fields unchanged.  
   - Connect `Merge` output to `Edit Fields`.

9. **Create Email Send Node**  
   - Name: `Send email`  
   - Type: Email Send (v2.1)  
   - Credentials: SMTP account (must be transactional email provider, see notes)  
   - From Email: Set your sending email address (e.g., `anir@agramprojects.com`)  
   - To Email: `={{ $json.Email }}`  
   - Subject: `={{ $json.subject }}`  
   - HTML Body: `={{ $json.body }}`  
   - Options: Disable attribution footer  
   - Connect `Edit Fields` output to this node.

10. **Create Wait Node for Delay**  
    - Name: `Wait`  
    - Type: Wait (v1.1)  
    - Configure delay (recommended 30-120 seconds) to avoid spam flags  
    - Connect `Send email` main output to `Wait`.

11. **Connect Wait Node to Loop Over Items**  
    - Connect `Wait` output back to `Loop Over Items` to process next lead.

12. **Create Merge Node to Combine Email Send Output**  
    - Name: `Merge1`  
    - Type: Merge (v3.1)  
    - Mode: Combine by position  
    - Connect `Edit Fields` output and `Send email` second output to `Merge1`.

13. **Create Google Sheets Node to Log Send Status**  
    - Name: `Google Sheets6`  
    - Type: Google Sheets (v4.6)  
    - Credentials: Google Sheets OAuth2 account  
    - Operation: Append or update rows in "Sheet1" (Leads sheet)  
    - Matching Columns: `Email`  
    - Columns to update:  
      - `Time` = current time localized to "Africa/Casablanca" timezone, 24h format  
      - `Email` = lead email  
      - `Send Status` = from email send output labelIds[0] (usually "SENT")  
    - Connect `Merge1` output to this node.

14. **Add Sticky Notes**  
    - Create all sticky notes as per content in the original workflow to provide setup instructions, common issues, and SMTP recommendations.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | Context or Link                                                                                         |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| ⚙️ SETUP CHECKLIST: Create leads and templates Google Sheets with specified columns; connect Google Sheets OAuth2 and SMTP credentials; set "From Email"; configure wait time; test with few leads first.                                                                                                                                                                                                                                                                                                                                                                                                | Setup guidance in Sticky Note3                                                                          |
| Common Issues: No emails sending → check SMTP; duplicate sends → check filter; name replacement issues; sheet update failures; spam filtering; loop configurations.                                                                                                                                                                                                                                                                                                                                                                                       | Sticky Note4                                                                                            |
| Testing Advice: Send 2-3 test emails; check personalization; confirm sheet updates; verify no duplicate sends; test empty Name fallback; check email formatting.                                                                                                                                                                                                                                                                                                                                                                                            | Sticky Note5                                                                                            |
| Gmail SMTP does not work for automated sending even with app passwords; recommended providers include SendGrid, Mailgun, Brevo, Amazon SES, Resend.com, Postmark, or custom domain SMTP.                                                                                                                                                                                                                                                                                                                                                                              | Sticky Note7                                                                                            |
| Step-by-step setup instructions including how to create sheets, fill columns, connect credentials, and test the workflow.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | Sticky Note6                                                                                            |
| Workflow respects rate limits and anti-spam by including a configurable wait node between emails.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               | General best practice                                                                                  |
| Replace `[Name]` placeholder carefully; fallback is "there" if lead name is missing or empty.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | Personalization logic in `Edit Fields` node                                                           |

---

This document fully describes the workflow structure, node configurations, dependencies, and operational details to replicate or modify the cold email campaign automation with Google Sheets tracking in n8n.