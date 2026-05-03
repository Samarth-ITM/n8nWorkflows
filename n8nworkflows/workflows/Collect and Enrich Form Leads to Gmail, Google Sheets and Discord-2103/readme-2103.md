Collect and Enrich Form Leads to Gmail, Google Sheets and Discord

https://n8nworkflows.xyz/workflows/collect-and-enrich-form-leads-to-gmail--google-sheets-and-discord-2103


# Collect and Enrich Form Leads to Gmail, Google Sheets and Discord

### 1. Workflow Overview

This workflow is designed to collect and enrich lead data submitted through a custom form created within n8n. It verifies the validity of each lead’s email address via Hunter.io, ensuring that only genuine leads progress further. Verified leads are then saved to multiple platforms—Gmail, Google Sheets, and Discord—for notification, archival, and team collaboration purposes. The workflow is modular and adaptable, allowing users to add or remove integrations as needed.

Logical blocks:

- **1.1 Input Reception:** Captures lead data from a web form created via n8n’s Form Trigger node.
- **1.2 Email Verification:** Uses Hunter.io’s email verification to validate the submitted email addresses.
- **1.3 Conditional Filtering:** Filters leads based on email validity.
- **1.4 Lead Distribution:** Sends validated lead data to Gmail (email notifications), Google Sheets (data storage), and Discord (team alerts).
- **1.5 No-Op Handling:** Handles invalid leads by stopping further processing.
- **1.6 User Guidance:** Sticky notes distributed throughout the workflow provide setup tips and best practices.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  Captures lead submissions from users through a customizable form interface hosted by n8n, collecting key fields such as name, email, and queries.

- **Nodes Involved:**  
  - n8n Form Trigger

- **Node Details:**  
  - **n8n Form Trigger**  
    - Type: Trigger node (formTrigger)  
    - Configuration:  
      - Path: `/form` (exposes webhook for form submissions)  
      - Form Title: "Form Title"  
      - Form Fields:  
        - Name (required text)  
        - Email (required text)  
        - "Let us know your queries" (textarea, optional)  
    - Input: HTTP POST request with form submission data  
    - Output: JSON object with form fields and metadata (`submittedAt`)  
    - Edge cases: Missing required fields will be rejected by the form UI; malformed JSON payloads would cause trigger failure.  
    - Version: 2  

---

#### 1.2 Email Verification

- **Overview:**  
  Validates the submitted email address using the Hunter.io API to reduce fake or invalid email leads.

- **Nodes Involved:**  
  - Hunter

- **Node Details:**  
  - **Hunter**  
    - Type: API integration node (hunter)  
    - Configuration:  
      - Operation: "emailVerifier"  
      - Email: Expression referencing form field `{{$json.Email}}`  
    - Input: JSON with email field from form trigger  
    - Output: Email verification result including status and confidence  
    - Edge cases:  
      - API rate limits or authentication errors may cause failure  
      - Invalid or malformed emails may return negative or uncertain status  
    - Version: 1  

---

#### 1.3 Conditional Filtering

- **Overview:**  
  Evaluates the Hunter.io verification results to decide whether the lead should proceed further or be discarded.

- **Nodes Involved:**  
  - If  
  - No Operation, do nothing  
  - Sticky Note3 (contextual guidance)  
  - Sticky Note4 (contextual guidance)

- **Node Details:**  
  - **If**  
    - Type: Conditional node (if)  
    - Configuration:  
      - Condition: Check if `{{$json.Email}}` equals empty string (note: likely a placeholder or misconfiguration—see edge cases)  
    - Input: Output from Hunter node  
    - Outputs:  
      - True branch: Leads with valid emails  
      - False branch: Leads with invalid or empty emails  
    - Edge cases:  
      - Condition seems incorrectly set to compare email to empty string, which might cause logic to fail or block valid emails unintentionally.  
      - Expression failures or unexpected data shape from Hunter could break condition evaluation.  
    - Version: 2  

  - **No Operation, do nothing**  
    - Type: No-op node  
    - Role: Endpoint for invalid leads to prevent workflow continuation  
    - Input: False branch from If node  
    - Output: None  
    - Edge cases: None  

  - **Sticky Note3**  
    - Content: "Use this only if you receive high volume of leads and you want to avoid fake leads with fake emails"  
    - Context: Near Hunter node to provide user advice about email verification necessity.

  - **Sticky Note4**  
    - Content: "Doesn't move forward if the email is not valid or if its fake email address"  
    - Context: Near If node to clarify filtering logic.

---

#### 1.4 Lead Distribution

- **Overview:**  
  For leads with verified emails, this block sends notifications and saves data to multiple platforms: Gmail for email notifications, Google Sheets for structured data storage, and Discord for team alerts.

- **Nodes Involved:**  
  - Gmail  
  - Google Sheets  
  - Discord  
  - Sticky Note (Gmail)  
  - Sticky Note2 (Google Sheets)  
  - Sticky Note1 (Discord)

- **Node Details:**  
  - **Gmail**  
    - Type: Email node (gmail)  
    - Configuration:  
      - Send To: `yourmail@gmail.com` (user must set their target email)  
      - Subject: "New lead from {{ $json.Name }}"  
      - Message Body: Multiline text with Name, Email, Query, Submitted On fields interpolated via expressions  
      - Email Type: Text  
    - Input: True branch output from If node (valid leads)  
    - Credentials: OAuth2 Gmail account  
    - Edge cases:  
      - Missing or incorrect "To" address causes email failure (highlighted in sticky note)  
      - Gmail API quota limits or auth token expiry could interrupt sending  
    - Version: 2.1  

  - **Google Sheets**  
    - Type: Spreadsheet node (googleSheets)  
    - Configuration:  
      - Operation: Update (matches existing rows or appends)  
      - Document ID: Google Sheet ID linked to leads list  
      - Sheet Name: Sheet1 (gid=0)  
      - Columns mapped: Name, Email, Query, Submitted On (timestamp)  
      - Matching Column: Name (to update existing entries)  
    - Input: True branch from If node  
    - Credentials: Google Sheets OAuth2  
    - Edge cases:  
      - Incorrect sheet ID or permissions cause failures  
      - Matching by "Name" only could cause unwanted overwrites if names repeat  
    - Version: 4.2  

  - **Discord**  
    - Type: Messaging node (discord)  
    - Configuration:  
      - Authentication: Webhook  
      - Webhook URL: Configured via credential  
      - Embeds: Rich message embedding lead details (color, title, author, description with Name, Email, Query, Submitted On)  
      - Content: Empty (using embed only)  
    - Input: True branch from If node  
    - Credentials: Discord Webhook API  
    - Edge cases:  
      - Invalid or revoked webhook URL causes failure  
      - Discord rate limits on webhook posts  
    - Version: 2  

  - **Sticky Note (near Gmail)**  
    - Content: "make sure to add To address so you can receive the notifications"  
    - Context: Reminder to set Gmail recipient address.

  - **Sticky Note2 (near Google Sheets)**  
    - Content: "Map the data to it's relevant fields/columns"  
    - Context: Guidance on proper field-to-column mapping.

  - **Sticky Note1 (near Discord)**  
    - Content: "Sometimes the email might not reach your inbox, but it rarely happens but if you receive a lot of leads it's better to setup discord webhook and receive updates that way so that your inbox doesn't get filled with all the leads"  
    - Context: Suggestion to use Discord webhook to avoid inbox overload.

---

### 3. Summary Table

| Node Name              | Node Type             | Functional Role                         | Input Node(s)         | Output Node(s)               | Sticky Note                                                                                      |
|------------------------|-----------------------|---------------------------------------|-----------------------|-----------------------------|------------------------------------------------------------------------------------------------|
| n8n Form Trigger       | formTrigger           | Capture lead submissions               | —                     | Hunter                      |                                                                                                |
| Hunter                 | hunter                | Verify email addresses                 | n8n Form Trigger       | If                          | "Use this only if you receive high volume of leads and you want to avoid fake leads with fake emails" |
| If                     | if                    | Filter leads by email validity        | Hunter                | Gmail, Google Sheets, Discord / No Operation, do nothing | "Doesn't move forward if the email is not valid or if its fake email address"                    |
| Gmail                  | gmail                 | Send lead notification email          | If (true branch)       | —                           | "make sure to add To address so you can receive the notifications"                              |
| Google Sheets          | googleSheets          | Save lead data in spreadsheet         | If (true branch)       | —                           | "Map the data to it's relevant fields/columns"                                                 |
| Discord                | discord               | Send lead notification to Discord     | If (true branch)       | —                           | "Sometimes the email might not reach your inbox, but it rarely happens but if you receive a lot of leads it's better to setup discord webhook and receive updates that way so that your inbox doesn't get filled with all the leads" |
| No Operation, do nothing | noOp                 | Stop processing invalid leads         | If (false branch)      | —                           |                                                                                                |
| Sticky Note            | stickyNote            | Guide Gmail "To" address setup        | —                     | —                           | "make sure to add To address so you can receive the notifications"                              |
| Sticky Note1           | stickyNote            | Suggest Discord for high volume leads | —                     | —                           | "Sometimes the email might not reach your inbox, but it rarely happens but if you receive a lot of leads it's better to setup discord webhook and receive updates that way so that your inbox doesn't get filled with all the leads" |
| Sticky Note2           | stickyNote            | Guide Google Sheets field mapping     | —                     | —                           | "Map the data to it's relevant fields/columns"                                                 |
| Sticky Note3           | stickyNote            | Advice on using Hunter for high volume leads | —               | —                           | "Use this only if you receive high volume of leads and you want to avoid fake leads with fake emails" |
| Sticky Note4           | stickyNote            | Clarify email validation filtering    | —                     | —                           | "Doesn't move forward if the email is not valid or if its fake email address"                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Form Trigger node**  
   - Add "n8n Form Trigger" node.  
   - Set path to `form`.  
   - Set form title to `"Form Title"`.  
   - Add required fields:  
     - Name (text, required)  
     - Email (text, required)  
     - Let us know your queries (textarea, optional)  
   - Save node.

2. **Add Hunter node for email verification**  
   - Add "Hunter" node.  
   - Select operation `emailVerifier`.  
   - Set email parameter to expression: `{{$json.Email}}`.  
   - Connect output of Form Trigger to Hunter.  
   - Configure Hunter credentials with your Hunter.io API key.

3. **Add If node for email validity check**  
   - Add "If" node.  
   - Define condition: **Note** the original condition is incorrectly checking if email equals empty string; correct this by checking Hunter's verification result instead. For example, use expression:  
     `{{$json['result'].status === 'valid'}}` (adjust according to Hunter's API response structure).  
   - Connect Hunter output to If node.

4. **Add Gmail node for email notification**  
   - Add "Gmail" node.  
   - Set "Send To" to your notification email address.  
   - Configure subject: `"New lead from {{ $json.Name }}"`.  
   - Configure message body with template:  
     ```
     Name:   {{ $json.Name }}

     Email:  {{ $json.Email }}

     Query:  {{ $json['Let us know your queries'] }}

     Submitted on:  {{ $json.submittedAt }}
     ```  
   - Connect If node's **true** output to Gmail node.  
   - Configure Gmail OAuth2 credentials.

5. **Add Google Sheets node to record leads**  
   - Add "Google Sheets" node.  
   - Set operation to `update`.  
   - Select your Google Sheet document and sheet (e.g., Sheet1).  
   - Map columns:  
     - Name → `{{$json.Name}}`  
     - Email → `{{$json.Email}}`  
     - Query → `{{$json['Let us know your queries']}}`  
     - Submitted On → `{{$json.submittedAt}}`  
   - Set matching column to "Name" (consider whether this suits your data).  
   - Connect If node's **true** output to Google Sheets node.  
   - Configure Google Sheets OAuth2 credentials.

6. **Add Discord node for team notifications**  
   - Add "Discord" node.  
   - Set authentication to Webhook and provide your Discord webhook URL via credentials.  
   - Configure embed message with:  
     - Title: `"New Lead from {{ $json.Name }}"`  
     - Description:  
       ```
       Name:   {{ $json.Name }}

       Email:  {{ $json.Email }}

       Query:  {{ $json['Let us know your queries'] }}

       Submitted on:  {{ $json.submittedAt }}
       ```  
   - Connect If node's **true** output to Discord node.

7. **Add No Operation node**  
   - Add "No Operation" node.  
   - Connect If node's **false** output (invalid emails) to this node to stop workflow progression.

8. **Add Sticky Notes (optional for guidance)**  
   - Add sticky notes near Gmail, Google Sheets, Discord, Hunter, and If nodes with the explanatory content provided in the sticky notes section above.

9. **Test workflow end-to-end**  
   - Deploy workflow.  
   - Submit test data via the exposed `/form` path.  
   - Verify that valid leads trigger email, spreadsheet update, and Discord notification.  
   - Confirm invalid emails do not progress past the filter.

---

### 5. General Notes & Resources

| Note Content                                                                                                               | Context or Link                                                                                                                                                       |
|----------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| This workflow uses Hunter.io for email verification; you can replace this with another service if preferred.               | Email Verification Block (Hunter node)                                                                                                                              |
| Gmail "To" address must be configured by the user to receive lead notifications.                                            | Gmail node sticky note                                                                                                                                                |
| Using Discord webhook notifications helps avoid inbox overload when receiving high volumes of leads.                       | Discord node sticky note: https://discord.com/developers/docs/resources/webhook                                                                                      |
| Mapping fields correctly in Google Sheets is critical for accurate data storage and updates.                               | Google Sheets node sticky note                                                                                                                                        |
| Consider revising the If node condition to properly evaluate Hunter.io's verification results instead of checking email empty string. | Potential bug in current workflow; must be corrected for email validation logic                                                                                        |
| Workflow inactive by default; activate when ready to deploy.                                                               | Workflow metadata                                                                                                                                                    |

---

This document provides a comprehensive explanation and reconstruction guide for the "Collect and Enrich Form Leads to Gmail, Google Sheets and Discord" workflow, enabling efficient usage, troubleshooting, and customization.