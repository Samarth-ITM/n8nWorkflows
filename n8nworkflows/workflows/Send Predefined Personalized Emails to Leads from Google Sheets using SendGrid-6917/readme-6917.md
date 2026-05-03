Send Predefined Personalized Emails to Leads from Google Sheets using SendGrid

https://n8nworkflows.xyz/workflows/send-predefined-personalized-emails-to-leads-from-google-sheets-using-sendgrid-6917


# Send Predefined Personalized Emails to Leads from Google Sheets using SendGrid

### 1. Workflow Overview

This workflow automates sending **predefined personalized emails** to leads stored in a Google Sheets document, using SendGrid as the email delivery service. It targets use cases where businesses want to engage multiple leads with consistent, templated email content, but personalized dynamically using lead-specific data from the sheet.

The workflow is logically divided into these blocks:

- **1.1 Input Reception:** Manual trigger to start the workflow.
- **1.2 Lead Data Retrieval:** Fetch lead records from a Google Sheet and limit the number processed per run.
- **1.3 Email Template Processing:** Retrieve predefined email templates from another Google Sheet tab, pick one randomly, and prepare it for personalization.
- **1.4 Personalization and Email Sending:** For each lead, replace template placeholders with actual lead data, then send personalized emails via SendGrid.
- **1.5 Throttling and Batch Control:** Introduce a wait step to control email send pacing and batch splitting to handle multiple leads efficiently.
- **1.6 Merging and Finalization:** Aggregate data streams for coherent processing and output.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Starts the workflow manually by user interaction, enabling controlled execution.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’

- **Node Details:**  
  - **Type:** Manual Trigger  
  - **Configuration:** Default manual trigger; no parameters.  
  - **Input/Output:** No input; outputs trigger event to next node.  
  - **Potential Failures:** None typical, manual start required to run workflow.  

---

#### 2.2 Lead Data Retrieval

- **Overview:**  
  Retrieves lead data from the Google Sheet “Leads” tab. Then limits the number of leads processed per run to 20 for performance management.

- **Nodes Involved:**  
  - Get row(s) in sheet  
  - Limit

- **Node Details:**  

  - **Get row(s) in sheet**  
    - Type: Google Sheets  
    - Role: Fetch leads data from the "Leads" tab of a specific Google Sheets document.  
    - Configuration:  
      - Document ID: from provided Google Sheet ID  
      - Sheet Name: dynamically set to “Leads” tab (empty in JSON, but expected to be set)  
      - Credentials: Google Sheets OAuth2 API  
    - Input: Trigger from manual start node  
    - Output: List of lead records in JSON format  
    - Failures: Authentication errors, empty sheet, API rate limits  

  - **Limit**  
    - Type: Limit  
    - Role: Restricts the number of lead rows processed to 20 per workflow execution.  
    - Configuration: maxItems = 20  
    - Input: Output from Google Sheets  
    - Output: Up to 20 lead items  
    - Failures: None expected  

---

#### 2.3 Email Template Processing

- **Overview:**  
  Retrieves all predefined email templates from the “Email Template” tab in the same Google Sheet, picks one randomly per batch for sending, and converts the email body to HTML format.

- **Nodes Involved:**  
  - Parse Email Template from DB  
  - Pick a random template

- **Node Details:**  

  - **Parse Email Template from DB**  
    - Type: Google Sheets  
    - Role: Fetch predefined email templates from the “Email Template” tab.  
    - Configuration:  
      - Document ID: same Google Sheet ID as leads  
      - Sheet Name: “Email Template”  
      - Credentials: Google Sheets OAuth2 API  
    - Input: Triggered after batch split  
    - Output: All email templates as JSON  
    - Failures: Auth errors, empty or missing template sheet  

  - **Pick a random template**  
    - Type: Code (JavaScript)  
    - Role: Randomly selects one template from fetched templates and converts the email body text into HTML by replacing line breaks with `<br>`.  
    - Key Code Logic:  
      - Throws error if no templates found  
      - Selects random index within templates array  
      - Transforms `Body` field newline characters to `<br>` for HTML formatting  
      - Outputs object with `subject` and `body` in HTML  
    - Input: Templates JSON array  
    - Output: One randomly chosen template with subject and HTML body  
    - Failures: No templates available, code execution errors  

---

#### 2.4 Personalization and Email Sending

- **Overview:**  
  For each lead, replaces placeholders in the email subject and body with lead-specific values, then sends the personalized email via SendGrid.

- **Nodes Involved:**  
  - Fix Variables  
  - Send an email

- **Node Details:**  

  - **Fix Variables**  
    - Type: Set  
    - Role: Performs string replacements on the email template’s subject and body fields, substituting placeholders such as `[BusinessName]`, `[Category]`, `[Location]`, and `[OtherCategories]` with corresponding lead data or default fallback values. Also sets recipient email and lead ID.  
    - Configuration:  
      - Uses expressions with `.replaceAll()` on `$json.subject` and `$json.body`  
      - Assigns new fields: `body`, `subject`, `email`, and `ID`  
    - Input: Merged lead and template data  
    - Output: Fully personalized email data for sending  
    - Failures: Missing lead fields causing undefined replacements (mitigated by defaults), expression evaluation errors  

  - **Send an email**  
    - Type: SendGrid  
    - Role: Sends the personalized email using SendGrid’s API.  
    - Configuration:  
      - Subject: from personalized subject field  
      - To Email: lead’s email address  
      - Content Type: text/html  
      - Content Value: personalized email body HTML  
      - ReplyToEmail: blank (can be configured)  
      - On Error: continue workflow execution despite errors  
      - Credentials: SendGrid API (assumed preconfigured)  
    - Input: Personalized email data  
    - Output: SendGrid API response  
    - Failures: Authentication errors, rate limits, invalid email addresses, network timeouts  

---

#### 2.5 Throttling and Batch Control

- **Overview:**  
  Controls sending speed to avoid API rate limits or spamming by introducing a 1-second wait between each email. Splits lead data into batches for efficient processing.

- **Nodes Involved:**  
  - Loop Over Items  
  - Wait  
  - Merge

- **Node Details:**  

  - **Loop Over Items**  
    - Type: SplitInBatches  
    - Role: Processes leads one by one (batch size defaults to 1) to allow per-item handling and throttling.  
    - Configuration: Default batch size; no additional options set  
    - Inputs: Limited leads and templates (via second output connection)  
    - Outputs: Each lead as individual batch item  
    - Failures: Large datasets may cause slow processing, batch size should be adjusted as needed  

  - **Wait**  
    - Type: Wait  
    - Role: Pauses workflow for 1 second after sending each email to manage send rate.  
    - Configuration: amount = 1 second  
    - Input: After email send node  
    - Output: Triggers next batch iteration  
    - Failures: Minimal risk, possible delays in workflow execution  

  - **Merge**  
    - Type: Merge (combineByPosition)  
    - Role: Combines the template data stream and the individual lead data stream so that personalization can occur with both data sources aligned.  
    - Configuration: Mode “combine” by position to pair corresponding items  
    - Input: Two streams — one from batch split (leads), one from template processing  
    - Output: Merged combined data  
    - Failures: Mismatched array lengths causing no pairs, empty inputs  

---

#### 2.6 Documentation and Notes

- **Overview:**  
  Provides contextual information, setup instructions, and suggestions for workflow extension, embedded inside sticky notes.

- **Nodes Involved:**  
  - Sticky Note  
  - Sticky Note1

- **Node Details:**  

  - **Sticky Note**  
    - Type: Sticky Note  
    - Content:  
      - Explains the workflow purpose: scalable, reliable email sending with predefined templates via SendGrid.  
      - Setup requirements: SendGrid API setup (including domain and sender identity), Google Sheets with two tabs ("Leads" and "Email Template").  
      - Provides a link to a sample Google Sheets database:  
        https://docs.google.com/spreadsheets/d/1UGarQNCplIfKKPSInxZlIC72oosZ45ul5jAQjYfpWrs/edit?usp=sharing  
      - Contact email for enquiries: buzanalytics@gmail.com  
    - Position: Top-left of canvas for visibility  

  - **Sticky Note1**  
    - Type: Sticky Note  
    - Content:  
      - Suggestions for extending the workflow into a full email automation system, including: webhook tracking, reply updates, AI-based email routing/classification, and automated follow-ups.  
    - Position: Right side, below email sending logic  

---

### 3. Summary Table

| Node Name                  | Node Type            | Functional Role                                  | Input Node(s)           | Output Node(s)           | Sticky Note                                                                                                             |
|----------------------------|----------------------|-------------------------------------------------|-------------------------|--------------------------|-------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger      | Starts the workflow manually                     | None                    | Get row(s) in sheet      |                                                                                                                         |
| Get row(s) in sheet        | Google Sheets        | Retrieve lead data from Google Sheets “Leads” tab | When clicking ‘Execute workflow’ | Limit                    |                                                                                                                         |
| Limit                      | Limit                | Limit processing to 20 leads per run             | Get row(s) in sheet     | Loop Over Items          |                                                                                                                         |
| Loop Over Items            | SplitInBatches       | Process each lead individually in batches        | Limit                   | Parse Email Template from DB (2nd output), no output (1st output) |                                                                                                                         |
| Parse Email Template from DB | Google Sheets        | Retrieve email templates from Google Sheets “Email Template” tab | Loop Over Items (2nd output) | Pick a random template   |                                                                                                                         |
| Pick a random template     | Code (JavaScript)    | Select a random email template and convert body to HTML | Parse Email Template from DB | Merge                    |                                                                                                                         |
| Merge                      | Merge                | Combine lead data and chosen template data       | Loop Over Items (1st output), Pick a random template | Fix Variables            |                                                                                                                         |
| Fix Variables              | Set                  | Replace placeholders with lead-specific values   | Merge                   | Send an email            |                                                                                                                         |
| Send an email              | SendGrid             | Send personalized email via SendGrid             | Fix Variables           | Wait                     |                                                                                                                         |
| Wait                      | Wait                 | Pause 1 second to throttle email sending         | Send an email           | Loop Over Items          |                                                                                                                         |
| Sticky Note                | Sticky Note          | Provides workflow overview, setup instructions   | None                    | None                     | 📧 Email Workflow with SendGrid & Predefined Email Templates. Setup requires SendGrid API and Google Sheets with Leads & Email Template tabs. Sample sheet: https://docs.google.com/spreadsheets/d/1UGarQNCplIfKKPSInxZlIC72oosZ45ul5jAQjYfpWrs/edit?usp=sharing |
| Sticky Note1               | Sticky Note          | Suggestions for workflow extensions               | None                    | None                     | 🧩 Extend into full email system with webhooks, reply updates, AI routing, and follow-ups.                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add “Manual Trigger” node:**  
   - Name: When clicking ‘Execute workflow’  
   - Leave default parameters  
   - This node starts the workflow manually.

3. **Add “Google Sheets” node to get leads:**  
   - Name: Get row(s) in sheet  
   - Operation: Read rows  
   - Document ID: Set to your Google Sheet’s ID containing leads & templates  
   - Sheet Name: Set to the tab name with leads (e.g., “Leads”)  
   - Authentication: Use Google Sheets OAuth2 credentials configured with access to the document  
   - Connect “When clicking ‘Execute workflow’” main output to this node.

4. **Add “Limit” node:**  
   - Name: Limit  
   - Set maxItems to 20 to limit leads processed per run  
   - Connect “Get row(s) in sheet” main output to this node.

5. **Add “SplitInBatches” node:**  
   - Name: Loop Over Items  
   - Leave default batch size (1) for processing leads one by one  
   - Connect “Limit” main output to this node’s main input.

6. **Add second “Google Sheets” node to get email templates:**  
   - Name: Parse Email Template from DB  
   - Operation: Read rows  
   - Document ID: Same Google Sheet ID as above  
   - Sheet Name: Set to the tab with email templates (e.g., “Email Template”)  
   - Credentials: Same Google Sheets OAuth2  
   - Connect second output of “Loop Over Items” node (batch output) to this node’s input.

7. **Add “Code” node:**  
   - Name: Pick a random template  
   - Language: JavaScript  
   - Code snippet:
     ```javascript
     const templates = items.map(item => item.json);
     if (!templates || templates.length === 0) {
       throw new Error('No templates found');
     }
     const index = Math.floor(Math.random() * templates.length);
     const template = templates[index];
     const bodyHtml = template.Body.replace(/\n/g, '<br>');
     return [{
       json: {
         subject: template.Subject,
         body: bodyHtml
       }
     }];
     ```
   - Connect “Parse Email Template from DB” output to this node.

8. **Add “Merge” node:**  
   - Name: Merge  
   - Mode: Combine  
   - Combine By: Position  
   - Connect first output of “Loop Over Items” (lead data) and output of “Pick a random template” to this node’s two inputs respectively.

9. **Add “Set” node:**  
   - Name: Fix Variables  
   - Add fields with expressions to replace placeholders in subject and body:  
     - `body` = `{{$json.body.replaceAll("[BusinessName]", $json["Business Name"]?.trim() || "your restaurant").replaceAll("[Category]", $json["Category"]?.trim() || "restaurant").replaceAll("[Location]", $json["Location"]?.trim() || "your area").replaceAll("[OtherCategories]", $json["Other Categories"]?.trim() || "similar restaurants")}}`  
     - `subject` = `{{$json.subject.replaceAll("[BusinessName]", $json["Business Name"]?.trim() || "your restaurant").replaceAll("[Category]", $json["Category"]?.trim() || "restaurant").replaceAll("[Location]", $json["Location"]?.trim() || "your area").replaceAll("[OtherCategories]", $json["Other Categories"]?.trim() || "similar restaurants")}}`  
     - `email` = `{{$json.Email}}`  
     - `ID` = `{{$json.ID}}`  
   - Connect “Merge” output to this node.

10. **Add “SendGrid” node:**  
    - Name: Send an email  
    - Operation: Send Mail  
    - Subject: `={{ $json.subject }}`  
    - To Email: `={{ $json.email }}`  
    - Content Type: `text/html`  
    - Content Value: `={{ $json.body }}`  
    - Additional fields: ReplyToEmail can be left blank or set as needed  
    - Credentials: Configure with your SendGrid API key (ensure domain & sender identity are verified)  
    - Set “On Error” behavior to “Continue” to skip errors and proceed  
    - Connect “Fix Variables” output to this node.

11. **Add “Wait” node:**  
    - Name: Wait  
    - Amount: 1 second  
    - Connect “Send an email” output to this node.

12. **Connect “Wait” node output back to “Loop Over Items” node:**  
    - This completes the batch processing loop to process next lead after wait.

13. **Optionally add “Sticky Note” nodes:**  
    - Add workflow description and setup instructions for maintainers and users.  
    - Include the sample Google Sheets link and contact info.  
    - Add another sticky note with suggestions for workflow extension.

14. **Test the workflow:**  
    - Run the manual trigger.  
    - Verify emails are sent with personalized content.  
    - Monitor for errors and adjust batch size or wait time as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                                                        | Context or Link                                                                                                         |
|-------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------|
| Sample Google Sheets database used for demo is available here: https://docs.google.com/spreadsheets/d/1UGarQNCplIfKKPSInxZlIC72oosZ45ul5jAQjYfpWrs/edit?usp=sharing | Predefined Leads and Email Template tabs setup                                                                        |
| SendGrid setup requires authenticating your domain and setting up sender identity to avoid delivery issues                          | SendGrid official docs: https://sendgrid.com/docs/ui/account-and-settings/how-to-set-up-domain-authentication/          |
| Workflow can be extended with webhook event tracking, reply processing, AI classification, and automated follow-ups for a full email system | See Sticky Note1 content inside the workflow                                                                            |
| Contact for enquiries and support: buzanalytics@gmail.com                                                                           | Workflow author contact                                                                                                |

---

This detailed documentation enables advanced users and automation agents to fully understand, reproduce, and modify the workflow for scalable, personalized email sending using Google Sheets and SendGrid.