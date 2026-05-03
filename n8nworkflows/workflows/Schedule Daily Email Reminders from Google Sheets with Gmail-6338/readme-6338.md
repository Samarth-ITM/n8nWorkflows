Schedule Daily Email Reminders from Google Sheets with Gmail

https://n8nworkflows.xyz/workflows/schedule-daily-email-reminders-from-google-sheets-with-gmail-6338


# Schedule Daily Email Reminders from Google Sheets with Gmail

### 1. Workflow Overview

This workflow automates the process of sending daily email reminders based on data stored in a Google Sheet. It is designed to query rows from a specified Google Sheet, filter those rows to identify which entries have a "Due Date" matching the current date, and then send a customized email to the corresponding recipients via Gmail.

**Target Use Cases:**  
- Daily task or event reminders  
- Automated notifications for deadlines or appointments  
- Email campaigns triggered by schedule stored in Sheets

**Logical Blocks:**

- **1.1 Manual Trigger:** Initiates the workflow manually or can be adapted for scheduled execution.
- **1.2 Data Retrieval:** Fetches rows from a specific Google Sheet.
- **1.3 Filtering Logic:** Filters rows where the "Due Date" equals today’s date.
- **1.4 Email Dispatch:** Sends customized emails to the filtered recipients using Gmail.

---

### 2. Block-by-Block Analysis

#### 1.1 Manual Trigger

- **Overview:**  
  This block starts the workflow manually when the user clicks “Execute workflow” in n8n’s UI. It allows on-demand execution for testing or manual runs.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’

- **Node Details:**  
  - **Node Name:** When clicking ‘Execute workflow’  
  - **Type:** Manual Trigger  
  - **Configuration:** No parameters; default manual trigger node.  
  - **Expressions:** None  
  - **Input:** None (start node)  
  - **Output:** Passes control to the Google Sheets node  
  - **Version Requirements:** Available in all n8n versions supporting manual triggers.  
  - **Potential Failures:** None expected; manual execution only.  
  - **Sub-workflow:** None

#### 1.2 Data Retrieval

- **Overview:**  
  Retrieves all rows from a specified Google Sheet document and sheet tab. This data represents all reminder entries.

- **Nodes Involved:**  
  - Get row(s) in sheet

- **Node Details:**  
  - **Node Name:** Get row(s) in sheet  
  - **Type:** Google Sheets node (version 4.6)  
  - **Configuration:**  
    - **Document ID:** User must specify Google Sheet document ID.  
    - **Sheet Name:** User must specify the tab name within the sheet.  
    - **Options:** Default, no filters applied at this stage (fetches all rows).  
  - **Expressions:** None  
  - **Input:** Receives trigger from manual node  
  - **Output:** Emits array of rows as JSON objects representing each row’s data fields, including "Due Date", "Email", "Body", "Subject"  
  - **Credentials:** Uses Google Sheets OAuth2 credentials with team account access  
  - **Version Requirements:** Uses Google Sheets node version 4.6 or higher for compatibility with current API features  
  - **Potential Failures:**  
    - Authentication errors if credentials expire or revoke access  
    - Invalid document ID or sheet name causing "not found" errors  
    - API rate limits from Google  
  - **Sub-workflow:** None

#### 1.3 Filtering Logic

- **Overview:**  
  Filters the sheet rows to select only those where the "Due Date" field exactly matches today’s date, formatted as ‘yyyy-MM-dd’.

- **Nodes Involved:**  
  - If

- **Node Details:**  
  - **Node Name:** If  
  - **Type:** If node (version 2.2)  
  - **Configuration:**  
    - Condition uses strict string equality between the field `Due Date` from each row and the current date formatted as `yyyy-MM-dd`.  
    - Case sensitive and strict type validation applied.  
  - **Expressions:**  
    - Left value: `{{$json['Due Date']}}`  
    - Right value: `{{$now.format('yyyy-MM-dd')}}`  
  - **Input:** Receives array of rows from Google Sheets node  
  - **Output:**  
    - True branch: rows matching today’s date  
    - False branch: rows that do not match  
  - **Version Requirements:** Requires n8n version supporting expression syntax `$now.format()` and If node v2.2+  
  - **Potential Failures:**  
    - Missing or malformed "Due Date" fields causing expression errors  
    - Date format mismatches leading to false negatives  
  - **Sub-workflow:** None

#### 1.4 Email Dispatch

- **Overview:**  
  Sends an email through Gmail to each filtered recipient, with subject and body dynamically populated from the sheet data.

- **Nodes Involved:**  
  - Send a message

- **Node Details:**  
  - **Node Name:** Send a message  
  - **Type:** Gmail node (version 2.1)  
  - **Configuration:**  
    - **Send To:** Uses `{{$json.Email}}` from the filtered row.  
    - **Subject:** Uses `{{$json.Subject}}`.  
    - **Message Body:** Uses `{{$json.Body}}`.  
    - **Options:** Default send options; no CC/BCC or attachments configured.  
  - **Expressions:** All dynamic fields are expressions evaluated per item.  
  - **Input:** Receives only rows passing the If node’s true condition.  
  - **Output:** Sends email and outputs success/failure status.  
  - **Credentials:** Uses Gmail OAuth2 credentials tied to a notification account.  
  - **Version Requirements:** Gmail node 2.1 or above recommended for OAuth2 support  
  - **Potential Failures:**  
    - Authentication errors if OAuth tokens expire or lack scopes  
    - Invalid email addresses causing send failures  
    - Gmail API rate limits or quota exceeded  
  - **Sub-workflow:** None

---

### 3. Summary Table

| Node Name                  | Node Type           | Functional Role               | Input Node(s)               | Output Node(s)        | Sticky Note                                           |
|----------------------------|---------------------|------------------------------|-----------------------------|-----------------------|-------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger     | Initiates workflow execution | None                        | Get row(s) in sheet    |                                                       |
| Get row(s) in sheet        | Google Sheets       | Retrieves all rows from sheet | When clicking ‘Execute workflow’ | If                    |                                                       |
| If                         | If                  | Filters rows with Due Date = today | Get row(s) in sheet          | Send a message         |                                                       |
| Send a message             | Gmail               | Sends email reminders         | If (true branch)            | None                  |                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Add a node of type “Manual Trigger.”  
   - Leave default configuration (no parameters).  
   - Name it: `When clicking ‘Execute workflow’`.

2. **Create Google Sheets Node:**  
   - Add a node of type “Google Sheets” (version 4.6+).  
   - Name it: `Get row(s) in sheet`.  
   - Set **Document ID** to your Google Sheets file ID (obtainable from sheet URL).  
   - Set **Sheet Name** to the exact tab name where data resides.  
   - Use default options to fetch all rows.  
   - Configure Google Sheets OAuth2 credentials authorized to access the sheet.  
   - Connect output of the Manual Trigger node to this node.

3. **Create If Node:**  
   - Add a node of type “If” (version 2.2+).  
   - Name it: `If`.  
   - Configure condition:  
     - Condition type: String equals  
     - Left Value: `{{$json["Due Date"]}}`  
     - Right Value: `{{$now.format("yyyy-MM-dd")}}`  
     - Enable strict type validation and case sensitivity.  
   - Connect output of the Google Sheets node to this node.

4. **Create Gmail Node:**  
   - Add a node of type “Gmail” (version 2.1+).  
   - Name it: `Send a message`.  
   - Configure parameters:  
     - **Send To:** `{{$json.Email}}`  
     - **Subject:** `{{$json.Subject}}`  
     - **Message:** `{{$json.Body}}`  
   - Configure Gmail OAuth2 credentials with appropriate scopes to send emails.  
   - Connect the **true** output branch of the If node to this Gmail node.

5. **Activate Workflow (Optional):**  
   - Test workflow manually using the manual trigger.  
   - Optionally set up a cron or time trigger to automate daily execution.

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                                    |
|-----------------------------------------------------------------------------------------------|---------------------------------------------------|
| The workflow depends on the date format in the "Due Date" column being exactly `yyyy-MM-dd`.  | Important for filtering logic to work correctly.  |
| Ensure Google Sheets OAuth2 credentials have “read” scopes for the target spreadsheet.         | Google Sheets API documentation                    |
| Gmail OAuth2 credentials must have “send” mail scopes enabled to avoid permission errors.      | Gmail API OAuth2 setup documentation                |
| For automation, replace the manual trigger with a Cron node to schedule daily runs.           | n8n Cron node documentation                         |

---

**Disclaimer:** The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.