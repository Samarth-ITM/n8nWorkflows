Collect Conference Feedback with Forms and Log to Excel OneDrive with Outlook Notifications

https://n8nworkflows.xyz/workflows/collect-conference-feedback-with-forms-and-log-to-excel-onedrive-with-outlook-notifications-4389


# Collect Conference Feedback with Forms and Log to Excel OneDrive with Outlook Notifications

### 1. Workflow Overview

This workflow automates the collection, processing, and archiving of conference feedback submitted via a custom online form. It targets event organizers seeking to gather attendee feedback efficiently, log responses into a centralized Excel spreadsheet stored on OneDrive, and notify support staff via Outlook email of new submissions.

The workflow’s logic is divided into these functional blocks:

- **1.1 Input Reception:** Captures form submissions from attendees providing feedback on the conference.
- **1.2 Data Parsing:** Extracts and normalizes submitted form data into structured variables.
- **1.3 Document Search & Validation:** Searches OneDrive for the feedback Excel file and checks its existence.
- **1.4 Data Preparation for Excel:** Maps parsed data into a format suitable for appending as a new row in Excel.
- **1.5 Append to Excel:** Adds the new feedback record to the identified Excel workbook worksheet.
- **1.6 Notification:** Sends an email alert to the support team with key feedback details.
- **1.7 Workflow Termination:** Ends the workflow cleanly after processing.

The workflow ensures reliable data capture, centralized record keeping, and timely notifications for conference feedback management.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Captures user feedback via a web form accessible under the `/feedback` path. This node triggers the workflow each time a submission occurs.

- **Nodes Involved:**  
  - On form submission

- **Node Details:**  
  - **Type:** Form Trigger  
  - **Role:** Entry point for feedback data capture  
  - **Configuration:**  
    - Webhook path: `feedback`  
    - Form title: "Conference Feedback Form"  
    - Fields: Full Name, Email, Company Name, Job Title, Referral Source (dropdown), Overall Experience Rating (dropdown), Preferred Sessions (textarea), Content Relevancy (dropdown), Networking Opportunities (dropdown), Improvement Suggestions (textarea), Future Suggestions (textarea), Attendance Intent (dropdown), Additional Comments (textarea), Open to Contact (dropdown)  
    - Required fields marked appropriately (e.g., Full Name, Email, Referral Source, Ratings)  
    - Form description includes a thank-you note with emoji  
  - **Input/Output:** No input; outputs form submission JSON payload  
  - **Edge Cases:**  
    - Missing required fields prevented by form validation  
    - Malformed email inputs handled by form validation  
    - Timezone awareness enabled to ensure submittedAt timestamps use workflow timezone  

#### 2.2 Data Parsing

- **Overview:**  
  Maps raw form submission JSON fields into normalized workflow variables for consistent downstream use.

- **Nodes Involved:**  
  - Parse Data

- **Node Details:**  
  - **Type:** Set node  
  - **Role:** Extracts and renames JSON fields for clarity  
  - **Configuration:** Assigns each form field value to a corresponding named variable such as `full_name`, `email`, `company_name`, `rating`, `submitted_at`, etc.  
  - **Expressions:** Uses syntax like `={{ $json['Full Name'] }}` to reference incoming data  
  - **Input:** JSON from form trigger  
  - **Output:** Structured JSON with clean variable names  
  - **Edge Cases:**  
    - Missing optional fields will result in empty strings, which downstream nodes should handle gracefully  

#### 2.3 Document Search & Validation

- **Overview:**  
  Searches OneDrive for the Excel workbook named `test-n8n-feedback-form-data.xlsx` to find the file where feedback will be appended.

- **Nodes Involved:**  
  - Sample File  
  - Search Document  
  - Code  
  - If Document Exists

- **Node Details:**  
  - **Sample File**  
    - Type: Convert To File (used here as a placeholder or to pass file name)  
    - Role: Provides the file name for search  
    - Output passes filename to `Search Document`  
  - **Search Document**  
    - Type: Microsoft OneDrive node (search operation)  
    - Role: Queries OneDrive for the file by name  
    - Credentials: Microsoft Drive OAuth2 connected  
    - Outputs metadata including file ID if found  
  - **Code**  
    - Type: Code node (JavaScript)  
    - Role: Extracts file ID from search results or returns null if not found  
    - Logic: Checks if `id` exists in input JSON, returns `id` or null  
  - **If Document Exists**  
    - Type: If node  
    - Role: Conditional branch based on presence of file ID  
    - Checks if `id` is defined and non-empty  
  - **Edge Cases:**  
    - File not found in OneDrive leads to workflow branch that skips appending data but still sends notification  
    - Permissions errors on OneDrive API calls can cause failure  
    - Multiple files with the same name may cause ambiguous search results (not handled explicitly)  

#### 2.4 Data Preparation for Excel

- **Overview:**  
  Prepares the parsed form data in a format matching the Excel worksheet columns, ensuring proper field mapping before appending.

- **Nodes Involved:**  
  - Build Sheet Data

- **Node Details:**  
  - **Type:** Set node  
  - **Role:** Maps each variable from `Parse Data` to a corresponding field for Excel append  
  - **Configuration:** Assignments for all relevant fields including `full_name`, `email`, `company_name`, `job_title`, `referral_source`, `rating`, `preferred_sessions`, and timestamps  
  - **Input:** Output of `If Document Exists` (true branch)  
  - **Output:** JSON formatted to match Excel columns  
  - **Edge Cases:**  
    - Missing or empty fields propagate as empty strings, which Excel accepts  
    - Data types assumed as strings, no explicit type conversion beyond that  

#### 2.5 Append to Excel

- **Overview:**  
  Adds the new feedback record as a row to the identified Excel worksheet on OneDrive.

- **Nodes Involved:**  
  - Append Data

- **Node Details:**  
  - **Type:** Microsoft Excel node (append operation)  
  - **Role:** Inserts new row data into worksheet with ID from `Code` node  
  - **Parameters:**  
    - Workbook identified by ID passed from `Code` node  
    - Worksheet set to default internal ID `{00000000-0001-0000-0000-000000000000}` (likely Sheet1)  
    - Data mode set to autoMap with rawData enabled  
  - **Credentials:** Microsoft Excel OAuth2 account configured  
  - **Input:** Prepared sheet data from `Build Sheet Data`  
  - **Output:** Confirmation or result of append operation  
  - **Edge Cases:**  
    - Invalid file or worksheet ID results in failure  
    - API rate limits or network issues may cause temporary failures  
    - Excel file locked or open by another user may block append  

#### 2.6 Notification

- **Overview:**  
  Sends an Outlook email to notify support staff of new feedback submission details.

- **Nodes Involved:**  
  - Notify Support

- **Node Details:**  
  - **Type:** Microsoft Outlook node (send email)  
  - **Role:** Sends notification email with key feedback data  
  - **Parameters:**  
    - Subject: "New Feedback Submission Received"  
    - Body: Includes submission timestamp, name, email, rating, and additional comments using expressions referencing `Parse Data`  
    - Recipient: `akhilgadiraju@gmail.com` (customizable)  
  - **Credentials:** Outlook OAuth2 account configured  
  - **Input:** Output from `Append Data` or `If Document Exists` false branch (ensures notification regardless of append success)  
  - **Edge Cases:**  
    - Email sending failure due to auth or network issues  
    - Missing fields in message body handled by empty string interpolation  

#### 2.7 Workflow Termination

- **Overview:**  
  Cleanly ends the workflow after all processing steps.

- **Nodes Involved:**  
  - End Workflow

- **Node Details:**  
  - **Type:** NoOp node (no operation)  
  - **Role:** Marks logical end point of the workflow  
  - **Input:** Output from `Notify Support` node  
  - **Output:** None  
  - **Edge Cases:** None  

---

### 3. Summary Table

| Node Name          | Node Type                | Functional Role                  | Input Node(s)        | Output Node(s)       | Sticky Note                                                                                  |
|--------------------|--------------------------|---------------------------------|----------------------|----------------------|----------------------------------------------------------------------------------------------|
| On form submission  | Form Trigger             | Capture form submissions        | None                 | Parse Data            |                                                                                              |
| Parse Data         | Set                      | Map and normalize form data     | On form submission   | Sample File           |                                                                                              |
| Sample File        | Convert To File           | Provide sample Excel filename   | Parse Data           | Search Document       | 📄 Sample Document Notice: Upload the document to OneDrive before proceeding.                |
| Search Document    | Microsoft OneDrive        | Search for Excel file on OneDrive | Sample File          | Code                  | 📁 Update File Name: Update file name as needed before use.                                  |
| Code               | Code                     | Extract file ID from search     | Search Document      | If Document Exists    | 📁 Update File Name: Update file name as needed before use.                                  |
| If Document Exists | If                       | Check if Excel file found       | Code                 | Build Sheet Data, Notify Support |                                                                                              |
| Build Sheet Data   | Set                      | Prepare data for Excel append   | If Document Exists   | Append Data           |                                                                                              |
| Append Data        | Microsoft Excel           | Append feedback to Excel sheet  | Build Sheet Data     | Notify Support        |                                                                                              |
| Notify Support     | Microsoft Outlook         | Send feedback notification email | Append Data, If Document Exists (false branch) | End Workflow          | ✉️ Customize Email Settings: Update subject and body as needed.                             |
| End Workflow       | NoOp                     | End workflow cleanly            | Notify Support       | None                  |                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add “On form submission” node:**
   - Type: Form Trigger  
   - Configure webhook path: `feedback`  
   - Set form title: "Conference Feedback Form"  
   - Add form fields exactly as in the original (including types, labels, placeholders, dropdown options, and required flags)  
   - Enable "Use Workflow Timezone"  
   - Save this node as the workflow entry point.

3. **Add “Parse Data” node:**
   - Type: Set  
   - Connect input from “On form submission”  
   - Add assignments mapping each form field from `$json[...]` to a named variable (e.g., `full_name`, `email`, etc.)  
   - Include the `submitted_at` field mapped from `$json.submittedAt`.

4. **Add “Sample File” node:**
   - Type: Convert To File  
   - Operation: `xlsx`  
   - FileName: `test-n8n-feedback-form-data.xlsx` (change as needed)  
   - Connect from “Parse Data”.

5. **Add “Search Document” node:**
   - Type: Microsoft OneDrive  
   - Operation: Search  
   - Query: Use expression referencing the filename from “Sample File” node  
   - Configure Microsoft OneDrive OAuth2 credentials  
   - Connect from “Sample File”.

6. **Add “Code” node:**
   - Type: Code (JavaScript)  
   - Paste the provided JS code snippet that extracts `id` from search results or returns null  
   - Connect from “Search Document”.

7. **Add “If Document Exists” node:**
   - Type: If  
   - Condition: Check if `id` exists and is not empty (`{{$json.id}} exists`)  
   - Connect from “Code”.

8. **Add “Build Sheet Data” node:**
   - Type: Set  
   - Connect from the true branch of “If Document Exists”  
   - Map all parsed variables from “Parse Data” into new variables matching Excel columns.

9. **Add “Append Data” node:**
   - Type: Microsoft Excel  
   - Operation: Append  
   - Resource: Worksheet  
   - Workbook: Set mode to ID, value from `{{$node["Code"].json.id}}`  
   - Worksheet: Use internal ID `{00000000-0001-0000-0000-000000000000}` or sheet name “Sheet1”  
   - Data Mode: AutoMap with rawData enabled  
   - Configure Microsoft Excel OAuth2 credentials  
   - Connect from “Build Sheet Data”.

10. **Add “Notify Support” node:**
    - Type: Microsoft Outlook  
    - Subject: "New Feedback Submission Received"  
    - Body: Use expressions to include submission date, name, email, rating, and comments from “Parse Data”  
    - Recipient: Set appropriate email (e.g., `akhilgadiraju@gmail.com`)  
    - Configure Microsoft Outlook OAuth2 credentials  
    - Connect from “Append Data” node (true branch) and also from the false branch of “If Document Exists” to ensure notifications always send.

11. **Add “End Workflow” node:**
    - Type: NoOp  
    - Connect from “Notify Support”.

12. **Add Sticky Notes (optional but recommended):**
    - Near “Sample File” and “Search Document” nodes: Note to upload and update Excel file name.  
    - Near “Notify Support” node: Note to customize email content and subject lines.

13. **Test the workflow end-to-end:**
    - Submit test data via the form at `/feedback` webhook  
    - Confirm data is appended to Excel file on OneDrive  
    - Confirm notification email is received.

---

### 5. General Notes & Resources

| Note Content                                                                                                              | Context or Link                                                                                     |
|---------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| This workflow uses Microsoft OAuth2 credentials for OneDrive and Outlook. Ensure valid credentials are set up accordingly. | Credential setup required for Microsoft Excel, OneDrive, and Outlook nodes.                        |
| The Excel file must be uploaded to OneDrive and named appropriately before running the workflow.                          | See sticky note near “Sample File” and “Search Document” nodes for instructions.                   |
| Email notification content can be customized in the “Notify Support” node body and subject parameters.                    | Sticky note near “Notify Support” node provides guidance.                                         |
| The form trigger supports multiple required and optional fields with validation to ensure data quality.                   | Form configuration in “On form submission” node includes placeholders and required flags.          |
| For troubleshooting, check API permissions, OneDrive file availability, and network connectivity.                         | Common failure points include OneDrive search failures, Excel append errors, and email send errors.|

---

**Disclaimer:** The text provided is exclusively from an automated workflow created with n8n, a workflow automation tool. The processing strictly respects applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.