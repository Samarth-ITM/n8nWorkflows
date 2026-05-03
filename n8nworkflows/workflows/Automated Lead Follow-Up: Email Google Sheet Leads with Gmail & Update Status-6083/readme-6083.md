Automated Lead Follow-Up: Email Google Sheet Leads with Gmail & Update Status

https://n8nworkflows.xyz/workflows/automated-lead-follow-up--email-google-sheet-leads-with-gmail---update-status-6083


# Automated Lead Follow-Up: Email Google Sheet Leads with Gmail & Update Status

### 1. Workflow Overview

This workflow automates the process of following up with new leads recorded in a Google Sheet. It periodically fetches lead data, filters for leads marked as "New," sends personalized emails to those leads via Gmail, and then updates their status in the sheet to "Contacted" to avoid duplicate outreach.

Logical blocks:

- **1.1 Scheduled Trigger**: Initiates the workflow on a recurring schedule.
- **1.2 Lead Data Retrieval**: Pulls lead records from a Google Sheet.
- **1.3 Lead Filtering**: Identifies leads that have status "New."
- **1.4 Batch Processing**: Processes leads individually or in manageable batches to respect Gmail rate limits.
- **1.5 Email Dispatch**: Sends personalized emails to each new lead.
- **1.6 Status Update**: Updates the Google Sheet to mark leads as contacted after emailing.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:**  
  This block triggers the workflow every 60 minutes, ensuring periodic execution without manual intervention. It serves as the entry point.

- **Nodes Involved:**  
  - Trigger: Run Every Day 🕒

- **Node Details:**

  - **Trigger: Run Every Day 🕒**  
    - Type and Role: Schedule Trigger node — initiates the workflow on a timer.  
    - Configuration: Set to trigger every 60 minutes (configurable to daily or weekly as needed).  
    - Expressions / Variables: None.  
    - Input: None (trigger node).  
    - Output: Leads to the next node that fetches data from Google Sheets.  
    - Version: 1.2.  
    - Edge Cases / Failures: Time sync errors unlikely; misconfiguration could cause unintended trigger intervals.

---

#### 1.2 Lead Data Retrieval

- **Overview:**  
  Fetches all lead entries from a specified Google Sheet document and sheet tab. This is the source of raw lead data including names, emails, and statuses.

- **Nodes Involved:**  
  - Fetch Leads from Google Sheet 📄

- **Node Details:**

  - **Fetch Leads from Google Sheet 📄**  
    - Type and Role: Google Sheets node — reads rows from a spreadsheet.  
    - Configuration: Document ID and Sheet name are configured via credential-linked list selection (dynamic from credentials).  
    - Expressions: None indicated; expects the sheet to have columns like Name, Email, Status.  
    - Input: From the schedule trigger node.  
    - Output: Emits an array of lead objects with their data.  
    - Version: 4.5.  
    - Edge Cases:  
      - Authentication failures with Google API.  
      - Sheet or document ID missing or invalid.  
      - Empty or malformed rows.  
      - Rate limits on Google Sheets API.  

---

#### 1.3 Lead Filtering

- **Overview:**  
  Filters retrieved leads to only those whose "Status" field equals "New." Only these leads proceed to email outreach.

- **Nodes Involved:**  
  - Filter Only New Leads 🔍

- **Node Details:**

  - **Filter Only New Leads 🔍**  
    - Type and Role: If node — conditional filter.  
    - Configuration: Checks if `Status` property equals "New" using an expression `={{$json["Status"]}} == "New"`.  
    - Input: From Google Sheets node.  
    - Output: Two outputs – first is for leads matching "New," second for all others (ignored).  
    - Version: 1.  
    - Edge Cases:  
      - Missing "Status" field in JSON may cause expression errors.  
      - Case sensitivity or status variants ("new" vs "New").  
      - Empty or null statuses.  

---

#### 1.4 Batch Processing

- **Overview:**  
  Processes leads in batches (default one-by-one) to avoid Gmail sending rate limits and to handle large lists gracefully.

- **Nodes Involved:**  
  - Batch Process Leads 🔁

- **Node Details:**

  - **Batch Process Leads 🔁**  
    - Type and Role: SplitInBatches node — splits incoming items into smaller groups.  
    - Configuration: Default settings, likely batch size 1 (not explicitly set).  
    - Input: From filter node's "true" output (new leads).  
    - Output: Leads output one batch at a time to the next node.  
    - Version: 3.  
    - Edge Cases:  
      - Batch size too large could hit Gmail rate limits.  
      - Empty batches if no leads.  

---

#### 1.5 Email Dispatch

- **Overview:**  
  Sends personalized emails to each lead using Gmail. Email content can include dynamic placeholders like {{Name}} and {{Email}}.

- **Nodes Involved:**  
  - Send Email to Lead ✉️

- **Node Details:**

  - **Send Email to Lead ✉️**  
    - Type and Role: Gmail node — sends emails via Gmail API.  
    - Configuration: Uses OAuth2 credentials linked to Gmail account. Email body and subject support templating with lead fields.  
    - Input: From batch processing node (one lead per execution).  
    - Output: Emits success/failure info for the email sent.  
    - Version: 2.1.  
    - Edge Cases:  
      - Gmail API quota exceeded or rate limits.  
      - Authentication errors with Gmail OAuth2.  
      - Invalid email addresses or malformed content.  
      - Network timeouts.  

---

#### 1.6 Status Update

- **Overview:**  
  Updates the original Google Sheet entry for each emailed lead to set their "Status" to "Contacted," preventing repeated emails.

- **Nodes Involved:**  
  - Mark Lead as Contacted ✅

- **Node Details:**

  - **Mark Lead as Contacted ✅**  
    - Type and Role: Google Sheets node — updates a row in the spreadsheet.  
    - Configuration: Uses same document and sheet as data retrieval, updates status column to "Contacted."  
    - Input: From successful email node execution.  
    - Output: Returns updated row information.  
    - Version: 4.5.  
    - Edge Cases:  
      - Row identification errors (must correctly identify lead row to update).  
      - API errors or rate limits.  
      - Sheet structure changes causing update failures.  

---

### 3. Summary Table

| Node Name                   | Node Type               | Functional Role               | Input Node(s)                 | Output Node(s)                  | Sticky Note                                                                                          |
|-----------------------------|-------------------------|------------------------------|------------------------------|-------------------------------|----------------------------------------------------------------------------------------------------|
| Trigger: Run Every Day 🕒    | Schedule Trigger        | Initiates workflow on timer  | None                         | Fetch Leads from Google Sheet 📄| 🔄 This node triggers the workflow every 60 minutes. You can change it to daily or weekly as needed. |
| Fetch Leads from Google Sheet 📄 | Google Sheets          | Reads leads from sheet       | Trigger: Run Every Day 🕒     | Filter Only New Leads 🔍        | 📥 This node reads data from your Google Sheet — it should contain leads with name, email, and status. |
| Filter Only New Leads 🔍     | If Node                 | Filters leads with Status=New| Fetch Leads from Google Sheet 📄 | Batch Process Leads 🔁          | ✅ This checks if the lead's status is 'New'. Only these leads will be emailed. Others are skipped.  |
| Batch Process Leads 🔁       | SplitInBatches           | Processes leads in batches   | Filter Only New Leads 🔍       | Send Email to Lead ✉️           | ⚙️ This helps process leads one-by-one or in small chunks. Useful for rate-limited services like Gmail. |
| Send Email to Lead ✉️        | Gmail                   | Sends personalized email     | Batch Process Leads 🔁         | Mark Lead as Contacted ✅       | 📨 This node sends personalized emails using Gmail. You can use {{Name}}, {{Email}} in your content. |
| Mark Lead as Contacted ✅    | Google Sheets           | Updates lead status to contacted| Send Email to Lead ✉️          | Batch Process Leads 🔁          | 📝 This updates the same Google Sheet to mark the lead as 'Contacted' so it won’t be processed again. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Set to trigger every 60 minutes (or adjust to daily/weekly as preferred).  
   - This will be the workflow’s entry point.

2. **Add a Google Sheets node** named "Fetch Leads from Google Sheet 📄"  
   - Configure Google Sheets credentials with access to your spreadsheet.  
   - Set the Document ID and Sheet Name to point to your leads sheet.  
   - Set operation to "Read Rows" to fetch all leads.  
   - Connect output of Schedule Trigger to this node.

3. **Add an If node** named "Filter Only New Leads 🔍"  
   - Configure condition: String equals where `{{$json["Status"]}} == "New"`.  
   - Connect output of Google Sheets node to this If node.

4. **Add a SplitInBatches node** named "Batch Process Leads 🔁"  
   - Default batch size (1 or adjust as needed).  
   - Connect the first output (true) of the If node to this node.  
   - This will allow processing leads one at a time.

5. **Add a Gmail node** named "Send Email to Lead ✉️"  
   - Configure Gmail OAuth2 credentials for sending email.  
   - Set "To" field to `{{$json["Email"]}}`.  
   - Compose subject and body using expressions/variables like `{{$json["Name"]}}`.  
   - Connect output of SplitInBatches node to this Gmail node.

6. **Add another Google Sheets node** named "Mark Lead as Contacted ✅"  
   - Use same credentials, Document ID, and Sheet Name as the fetch node.  
   - Configure this node to update the lead’s "Status" column to "Contacted".  
   - Use data from the email node to identify the correct row (e.g., row number or unique ID).  
   - Connect output of Gmail node to this node.

7. **Connect the output of "Mark Lead as Contacted ✅" node back to "Batch Process Leads 🔁"**  
   - This allows batch node to continue processing next lead until all are done.

8. **Test the workflow** with sample data to ensure email sending and status updating work correctly.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                                                     |
|----------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| The workflow includes scheduling, Google Sheets integration, conditional filtering, and Gmail API usage. | Useful for automated lead follow-up campaigns in sales or marketing teams.                        |
| Gmail node requires OAuth2 credentials with send mail permission configured in Google Cloud Console. | https://developers.google.com/gmail/api/quickstart/nodejs                                         |
| Google Sheets API quota limits may affect large lead lists; consider batch sizes accordingly.       | https://developers.google.com/sheets/api/limits                                                    |
| For personalization, ensure your Google Sheet columns exactly match the variable names used in expressions (e.g., Name, Email, Status). |                                                                                                   |
| Sticky notes included in the workflow provide helpful reminders about node roles and configuration. |                                                                                                   |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled are legal and public.