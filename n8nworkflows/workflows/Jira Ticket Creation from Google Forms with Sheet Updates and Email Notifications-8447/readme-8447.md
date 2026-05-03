Jira Ticket Creation from Google Forms with Sheet Updates and Email Notifications

https://n8nworkflows.xyz/workflows/jira-ticket-creation-from-google-forms-with-sheet-updates-and-email-notifications-8447


# Jira Ticket Creation from Google Forms with Sheet Updates and Email Notifications

### 1. Workflow Overview

This workflow automates the creation of Jira tickets from Google Forms responses, updates the associated Google Sheet with ticket information, and sends email notifications to relevant parties. It is designed to streamline issue tracking by linking form submissions directly to Jira issues while maintaining synchronized records and providing timely notifications.

Logical blocks include:

- **1.1 Trigger and Input Normalization:** Detect new Google Sheets rows from form responses, clean and normalize data for Jira compatibility.  
- **1.2 Jira Issue Creation:** Use normalized data to create a Jira ticket via API, mapping priorities and formatting descriptions.  
- **1.3 Sheet Update and Notification:** Update the Google Sheet row with Jira ticket details and send notification emails via Gmail.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger and Input Normalization

**Overview:**  
This block initiates the workflow when a new response row is added to a Google Sheet linked to a Google Form. It normalizes and prepares data fields to match Jira’s expected input format, including priority mapping and constructing issue summaries and descriptions.

**Nodes Involved:**  
- Trigger when row added (Google Sheets Trigger)  
- Normalize fields (Code node)  
- Sticky Note (notes on trigger and normalization best practices)  
- Sticky Note1 (requirements)

**Node Details:**

- **Trigger when row added**  
  - Type: Google Sheets Trigger  
  - Role: Watches for new rows added to a specific Google Sheet (response sheet for Google Form).  
  - Configuration: Polls every minute for new rows; configured with Google Sheets OAuth2 credentials. Sheet name and document ID must be set to the form response sheet.  
  - Inputs: External event (new row).  
  - Outputs: JSON data representing the new row.  
  - Potential Failures: Authentication errors, API rate limits, sheet ID misconfiguration, empty or malformed rows.  

- **Normalize fields**  
  - Type: Code (JavaScript) node  
  - Role: Cleans and maps form fields to Jira-compatible data.  
  - Configuration:  
    - Maps priority strings (Highest, High, Medium, Low, Lowest) to Jira priority IDs (1-5). Defaults to Medium (3) if missing or unmatched.  
    - Extracts context from 'Context' field or defaults to "No context provided".  
    - Extracts email from multiple possible fields ('Adresse e-mail', 'Email'). Defaults to "No email provided".  
    - Builds a Jira issue summary from context.  
    - Constructs a detailed description including email, context, bug reproduction steps, and acceptance criteria (fallback to "N/A" if missing).  
    - Returns original trigger data extended with normalized properties: `priority`, `summary`, `description`.  
  - Inputs: JSON from Google Sheets trigger.  
  - Outputs: Normalized JSON for Jira ticket creation.  
  - Potential Failures: Missing expected fields, expression errors, inconsistent priority values.  
  - Version: Uses Code node v2.  

- **Sticky Note (Trigger and Normalize)**  
  - Notes that all data cleaning and normalization happens here to keep subsequent nodes simple and reliable.  
  - Advises handling optional fields and trimming whitespace.  

- **Sticky Note1 (Requirements)**  
  - Lists prerequisites: Google Form + response sheet, Jira Cloud project credentials (email + API token), Gmail OAuth2 credential.

---

#### 2.2 Jira Issue Creation

**Overview:**  
Creates a Jira issue using the normalized data from the previous block. Sets project, issue type, priority, summary, and description fields as per Jira API requirements.

**Nodes Involved:**  
- Cretate Jira Ticket (Jira node)  
- Sticky Note2 (notes about Jira ticket creation)

**Node Details:**

- **Cretate Jira Ticket**  
  - Type: Jira node (Jira Cloud API)  
  - Role: Creates a new Jira issue in the specified project.  
  - Configuration:  
    - Project ID set to a specific project (e.g., "10002").  
    - Issue Type: Story (ID "10014").  
    - Summary: Uses normalized summary from previous node.  
    - Priority: Uses normalized priority ID.  
    - Description: Uses constructed description string.  
  - Credentials: Jira Software Cloud API credentials configured for authentication.  
  - Inputs: Normalized JSON from code node.  
  - Outputs: Jira ticket details, including ticket key and URL.  
  - Potential Failures: Authentication failure, permission issues, API rate limits, invalid project or issue type ID, malformed input data.  

- **Sticky Note2 (Create Jira ticket)**  
  - Explains key parameters and outputs: project, type (Story), priority mapping, description format.  
  - Includes advice on mapping priority strings to Jira IDs.  

---

#### 2.3 Sheet Update and Notification

**Overview:**  
Updates the original Google Sheet row with Jira ticket metadata and sends notification emails to specified recipients, including ticket reference, URL, status, and requester information.

**Nodes Involved:**  
- Update the Google sheet with tickets information (Google Sheets node)  
- Notification email (Gmail node)  
- Sticky Note3 (notes about sheet update and email notification)

**Node Details:**

- **Update the Google sheet with tickets information**  
  - Type: Google Sheets node  
  - Role: Updates the same Google Sheet row that triggered the workflow with new Jira ticket info.  
  - Configuration:  
    - Operation: Update  
    - Sheet and document IDs must match the response sheet.  
    - Fields updated typically include:  
      - `jira_key` (ticket reference)  
      - `jira_url` (ticket link)  
      - `status` (set to "Created")  
      - `created_at` (timestamp of creation)  
    - Uses a matching key, e.g., "Horodateur" (timestamp column) to locate correct row.  
  - Credentials: Google Sheets OAuth2 credentials.  
  - Inputs: Jira ticket info from previous node.  
  - Outputs: Confirmation of sheet update.  
  - Potential Failures: Authentication errors, incorrect sheet or range references, concurrency issues, row matching failures.  

- **Notification email**  
  - Type: Gmail node  
  - Role: Sends an email notification about the newly created Jira ticket.  
  - Configuration:  
    - Subject: "You just received a new ticket"  
    - HTML body includes ticket reference, URL, title, priority (mapped back to readable form), status, and requester email extracted from multiple nodes to ensure fallback.  
    - Recipients: Configured in the Gmail credential or overridden in parameters (not explicitly shown, presumably defaults or dynamic).  
  - Credentials: Gmail OAuth2 credentials.  
  - Inputs: Data from Google Sheets update and Jira ticket creation nodes.  
  - Outputs: Email sent confirmation.  
  - Potential Failures: Authentication errors, email quota exceeded, invalid recipient addresses, connection timeouts.  

- **Sticky Note3 (Update sheet & send notification)**  
  - Details updated fields and matching mechanism using "Horodateur".  
  - Explains the email content and recipient roles.

---

### 3. Summary Table

| Node Name                               | Node Type              | Functional Role                                  | Input Node(s)               | Output Node(s)                          | Sticky Note                                                                                                  |
|----------------------------------------|------------------------|-------------------------------------------------|-----------------------------|----------------------------------------|--------------------------------------------------------------------------------------------------------------|
| Sticky Note1                           | Sticky Note            | Lists requirements                               |                             |                                        | • Google Form + response sheet • Jira Cloud project (API email + token) • Gmail credential                     |
| Trigger when row added                 | Google Sheets Trigger  | Trigger on new form response row                 |                             | Normalize fields                      |                                                                                                              |
| Normalize fields                      | Code                   | Normalize and map form data for Jira             | Trigger when row added       | Cretate Jira Ticket                   | Advises to clean all data here to avoid reprocessing later                                                  |
| Sticky Note                          | Sticky Note            | Notes about trigger and normalization best practice |                             |                                        | Advises trimming, handling optional fields                                                                 |
| Cretate Jira Ticket                   | Jira                   | Create Jira issue with mapped data                | Normalize fields            | Update the Google sheet with tickets information | Explains priority mapping and creation parameters                                                          |
| Sticky Note2                         | Sticky Note            | Notes on Jira ticket creation                      |                             |                                        |                                                                                                              |
| Update the Google sheet with tickets information | Google Sheets          | Update original sheet row with Jira ticket info   | Cretate Jira Ticket          | Notification email                   | Details updated columns and matching by "Horodateur"; explains notification content                          |
| Notification email                   | Gmail                  | Send email notification about new Jira ticket     | Update the Google sheet with tickets information |                                        |                                                                                                              |
| Sticky Note3                         | Sticky Note            | Notes on sheet update and email notification       |                             |                                        |                                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Sheets Trigger Node**  
   - Type: Google Sheets Trigger  
   - Configure with:  
     - Authentication: Google Sheets OAuth2 credentials  
     - Document ID: URL or ID of the Google Form response sheet  
     - Sheet Name: Name of the tab with form responses  
     - Event: "Row Added"  
     - Poll interval: Every 1 minute  

2. **Add Code Node "Normalize fields"**  
   - Type: Code (JavaScript) node (v2)  
   - Input: Connected from Google Sheets Trigger output  
   - Paste the following logic (conceptual summary):  
     - Map priority field text to Jira priority IDs (1-5)  
     - Extract context and email from multiple possible fields  
     - Construct summary from context  
     - Build description string including email, context, steps to reproduce, acceptance criteria  
     - Return a JSON object combining original data with normalized properties: `priority`, `summary`, `description`  

3. **Add Jira Node "Cretate Jira Ticket"**  
   - Type: Jira (Jira Software Cloud)  
   - Connect input from "Normalize fields" node  
   - Configure with:  
     - Credentials: Jira Cloud API (email + API token)  
     - Project: Select your Jira project (e.g., ID "10002")  
     - Issue Type: Select "Story" (e.g., ID "10014")  
     - Summary: Expression to use `{{$json.summary}}` from input  
     - Priority: Expression `{{$json.priority}}`  
     - Description: Expression `{{$json.description}}`  

4. **Add Google Sheets Node "Update the Google sheet with tickets information"**  
   - Type: Google Sheets  
   - Connect input from Jira node output  
   - Configure with:  
     - Credentials: Google Sheets OAuth2  
     - Document ID and Sheet Name: Same as trigger sheet  
     - Operation: Update  
     - Specify columns to update: `jira_key` (ticket key), `jira_url` (ticket URL), `status` ("Created"), `created_at` (timestamp)  
     - Set row matching key, typically using a timestamp column like "Horodateur" from trigger data to locate the correct row  

5. **Add Gmail Node "Notification email"**  
   - Type: Gmail  
   - Connect input from Google Sheets update node  
   - Configure with:  
     - Credentials: Gmail OAuth2  
     - Subject: "You just received a new ticket"  
     - Message body (HTML) includes:  
       - Ticket reference and URL (from Jira node output)  
       - Title and priority (mapped back to readable text)  
       - Status  
       - Requester email extracted from multiple fields for fallback robustness  
     - Recipients: As per your notification policy (default or set dynamically)  

6. **Add Sticky Notes** (Optional but recommended for documentation within the workflow)  
   - Add notes explaining prerequisites (Google Form, Jira credentials, Gmail credentials)  
   - Add notes about data normalization best practices and what is handled in each step  
   - Add notes about Jira ticket creation parameters and priority mapping  
   - Add notes about sheet update mechanism and email notification content  

7. **Connect the nodes in sequence:**  
   - Google Sheets Trigger → Normalize fields → Cretate Jira Ticket → Update the Google sheet → Notification email  

8. **Test the workflow:**  
   - Submit a new Google Form entry; verify the Jira ticket is created, the sheet is updated, and notification email is sent.  
   - Adjust mapping, credentials, and sheet ranges as necessary.

---

### 5. General Notes & Resources

| Note Content                                                                                                                    | Context or Link                                                                                      |
|---------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Use Jira Cloud API with email + API token for authentication. Credentials must have permission to create issues in chosen project. | Jira API Docs: https://developer.atlassian.com/cloud/jira/platform/rest/v3/                         |
| Google Sheets Trigger requires OAuth2 credentials with access to the form response sheet.                                        | Google Sheets API Docs: https://developers.google.com/sheets/api                                    |
| Gmail node uses OAuth2 credentials and must be authorized to send emails on behalf of the configured account.                   | Gmail API Docs: https://developers.google.com/gmail/api                                           |
| Priority mapping logic ensures Jira receives numeric priority IDs, avoiding invalid data errors.                                 | Jira Priority Mapping Best Practices                                                               |
| Matching sheet rows for update uses a timestamp ("Horodateur") column to avoid incorrect updates when multiple entries exist.   | Best practice for data synchronization between forms and sheets                                   |
| Email notification content includes safe fallback logic to handle missing or malformed data fields from form submissions.       | Recommended for robust user communication                                                           |
| Sticky Notes in the workflow provide inline documentation to help maintainers understand the logic and dependencies clearly.   | n8n Sticky Notes feature documentation: https://docs.n8n.io/nodes/n8n-nodes-base.stickyNote/       |

---

**Disclaimer:**  
The provided text is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.