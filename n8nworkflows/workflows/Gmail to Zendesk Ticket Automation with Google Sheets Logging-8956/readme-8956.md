Gmail to Zendesk Ticket Automation with Google Sheets Logging

https://n8nworkflows.xyz/workflows/gmail-to-zendesk-ticket-automation-with-google-sheets-logging-8956


# Gmail to Zendesk Ticket Automation with Google Sheets Logging

### 1. Workflow Overview

This workflow automates the creation of Zendesk support tickets from incoming Gmail messages filtered by a specific label and logs the created ticket information into a Google Sheets spreadsheet. It is designed for support teams or customer service automation where email requests must be tracked as tickets and recorded for reporting or auditing.

The workflow consists of four main logical blocks:

- **1.1 Input Reception (Gmail Trigger)**: Watches a specified Gmail label for new incoming emails, triggering the workflow per new message.
- **1.2 Data Normalization**: Transforms the raw Gmail payload into a standardized format suitable for ticket creation.
- **1.3 Zendesk Ticket Creation**: Uses normalized data to create a new ticket in Zendesk, capturing relevant ticket metadata.
- **1.4 Logging to Google Sheets**: Formats the combined data from Gmail and Zendesk and appends or updates a row in a Google Sheets spreadsheet for record keeping.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception (Gmail Trigger)

**Overview:**  
This block detects new incoming emails in a Gmail account, filtered by a specific Gmail label, and outputs the raw email data for processing.

**Nodes Involved:**  
- Gmail Trigger

**Node Details:**

- **Node Name:** Gmail Trigger  
- **Type:** Gmail Trigger (n8n-nodes-base.gmailTrigger)  
- **Technical Role:** Event trigger node that listens for new emails matching filter criteria.  
- **Configuration:**  
  - Polls every minute for new emails.  
  - Filters by Gmail label ID `"Label_7215267856143431312"` (user-configured label to scope emails).  
  - Outputs full email data including headers, subject, body (plain and HTML), sender info, recipients, timestamps, and attachment metadata.  
- **Expressions/Variables:** None externally configured; outputs raw Gmail data in JSON.  
- **Input Connections:** None (trigger node).  
- **Output Connections:** Connects to "Normalize Gmail Data" node.  
- **Version Requirements:** Uses Gmail OAuth2 credentials; ensure OAuth token is valid and has Gmail read access.  
- **Potential Failures:**  
  - Authentication errors if OAuth expires or is revoked.  
  - API quota limits or Gmail service downtime.  
  - Label ID misconfiguration causing no emails detected.  
- **Sticky Note:**  
  > Listens for new emails from the configured Gmail account/label and emits each message once with headers, subject, body (plain/HTML), sender, recipients, timestamp, and attachment metadata for downstream processing.

---

#### 2.2 Data Normalization

**Overview:**  
Transforms raw Gmail email data into a consistent, simplified JSON schema to ensure downstream nodes receive standardized fields such as requester email, subject, description, priority, and identifiers.

**Nodes Involved:**  
- Normalize Gmail Data

**Node Details:**

- **Node Name:** Normalize Gmail Data  
- **Type:** Code Node (n8n-nodes-base.code)  
- **Technical Role:** Data transformation using JavaScript to parse and normalize Gmail message structure.  
- **Configuration Highlights:**  
  - Extracts the sender's email and name from the "from" field.  
  - Prefers plain text body; if missing, strips HTML tags from HTML body.  
  - Detects urgency by checking if the subject or body contains the word "urgent" (case-insensitive).  
  - Sets normalized fields: `source`, `subject`, `description`, `requester_email`, `requester_name`, `priority` (urgent/normal), `timestamp`, and `original_id`.  
  - Preserves raw Gmail JSON data for reference.  
- **Expressions/Variables:** Uses `$input.first().json` to access incoming raw Gmail data.  
- **Input Connections:** From "Gmail Trigger" node.  
- **Output Connections:** To "Create Zendesk Ticket" node.  
- **Version Requirements:** None specific; requires JavaScript enabled and basic node execution support.  
- **Potential Failures:**  
  - Unexpected Gmail JSON structure changes causing missing fields.  
  - HTML parsing errors if body is malformed.  
- **Sticky Note:**  
  > Transforms the raw Gmail payload into a consistent schema (e.g., requesterEmail, subject, bodyText/bodyHtml, threadId, messageId, labels, attachments[]) to ensure reliable field mapping across subsequent nodes.

---

#### 2.3 Zendesk Ticket Creation

**Overview:**  
Creates a new Zendesk support ticket using the normalized email data, including subject, description, and priority tags, and outputs the ticket metadata for logging.

**Nodes Involved:**  
- Create Zendesk Ticket

**Node Details:**

- **Node Name:** Create Zendesk Ticket  
- **Type:** Zendesk Node (n8n-nodes-base.zendesk)  
- **Technical Role:** API interaction node to create a ticket in Zendesk.  
- **Configuration Highlights:**  
  - Sets ticket description from normalized `description` field.  
  - Sets ticket subject from normalized `subject` field.  
  - Tags include the priority value (e.g., `urgent` or `normal`) derived from email content analysis.  
  - Uses configured Zendesk API credentials with appropriate API key or OAuth token.  
- **Expressions/Variables:** Uses expressions like `={{ $json.description }}`, `={{ $json.subject }}`, and `={{ $json.priority }}` to dynamically set fields.  
- **Input Connections:** From "Normalize Gmail Data" node.  
- **Output Connections:** To "Format Sheet Data" node.  
- **Version Requirements:** Zendesk API credentials must be valid and have ticket creation permissions.  
- **Potential Failures:**  
  - Authentication failures (expired or invalid Zendesk credentials).  
  - API rate limits or downtime.  
  - Invalid or missing required fields causing ticket creation to fail.  
- **Sticky Note:**  
  > Creates a Zendesk ticket using normalized fields: requester (email), subject, description (plain text or HTML), optional priority/tags/group/assignee. Outputs ticketId, ticketUrl, status, and requesterId for logging.

---

#### 2.4 Logging to Google Sheets

**Overview:**  
Combines Gmail and Zendesk ticket data into a structured row and appends or updates this data in a Google Sheets spreadsheet to maintain a log of processed tickets.

**Nodes Involved:**  
- Format Sheet Data  
- Log to Google Sheets

**Node Details:**

- **Node Name:** Format Sheet Data  
- **Type:** Code Node (n8n-nodes-base.code)  
- **Technical Role:** Constructs a clean, structured object for Google Sheets logging.  
- **Configuration Highlights:**  
  - Extracts ticket ID and URL from Zendesk output.  
  - Constructs a clickable agent ticket URL using the Zendesk subdomain parsed from the API URL or a fallback hardcoded URL.  
  - Includes ticket metadata such as subject, priority, status, creation timestamps, description preview (first 100 chars), and tags.  
  - Includes requester information and original Gmail data fields for completeness.  
- **Expressions/Variables:** Uses `$input.first().json` and `$node["Create Zendesk Ticket"].json` to combine inputs.  
- **Input Connections:** From "Create Zendesk Ticket" node.  
- **Output Connections:** To "Log to Google Sheets" node.  
- **Version Requirements:** Requires JavaScript runtime with ES6 support.  
- **Potential Failures:**  
  - Errors if Zendesk ticket data is incomplete or missing URL/id fields.  
  - Date formatting issues.  

- **Node Name:** Log to Google Sheets  
- **Type:** Google Sheets Node (n8n-nodes-base.googleSheets)  
- **Technical Role:** Writes data rows to a specified Google Sheets document and sheet, appending or updating records keyed by Ticket ID.  
- **Configuration Highlights:**  
  - Document ID: `1pz1aP0OIULFoEJCI6VWged3jq1W7870btV2QOC4vn1o` (Google Sheets document).  
  - Sheet name identified by GID `0`.  
  - Columns mapped explicitly to ticket fields such as Ticket ID, Ticket URL, Subject, Priority, Status, timestamps, tags, and description preview.  
  - Uses "appendOrUpdate" operation keyed on "Ticket ID" to avoid duplicate entries.  
  - Uses Google Sheets OAuth2 credentials with write access.  
- **Expressions/Variables:** Uses expressions like `={{ $json.ticket_id }}`, `={{ $json.subject }}`, etc., to map data to columns.  
- **Input Connections:** From "Format Sheet Data" node.  
- **Output Connections:** None (terminal node).  
- **Version Requirements:** Valid Google Sheets OAuth2 credentials; document must be shared with the service account or user.  
- **Potential Failures:**  
  - Permission errors if Sheets document access is revoked or credentials invalid.  
  - API quota limits or Google Sheets service issues.  
  - Column schema mismatch if the sheet structure changes.  
- **Sticky Note:**  
  > Writes the row to the target spreadsheet—appending new records or updating existing ones using a unique key (e.g., messageId or ticketId). Confirms success and surfaces any write conflicts or missing columns.

- **Sticky Note on Format Sheet Data:**  
  > Builds a tidy row object combining Gmail and Zendesk outputs (e.g., timestamp, requesterEmail, subject, ticketId, ticketUrl, status, threadId, labels, workflowRunId). Ensures column order and default values match the Sheet schema.

---

### 3. Summary Table

| Node Name           | Node Type               | Functional Role                       | Input Node(s)         | Output Node(s)         | Sticky Note                                                                                                             |
|---------------------|-------------------------|------------------------------------|-----------------------|------------------------|-------------------------------------------------------------------------------------------------------------------------|
| Gmail Trigger       | Gmail Trigger           | Watches Gmail label for new emails | None                  | Normalize Gmail Data    | Listens for new emails from the configured Gmail account/label and emits each message once with headers, subject, body (plain/HTML), sender, recipients, timestamp, and attachment metadata for downstream processing. |
| Normalize Gmail Data | Code                    | Normalize raw Gmail data to schema | Gmail Trigger         | Create Zendesk Ticket   | Transforms the raw Gmail payload into a consistent schema (e.g., requesterEmail, subject, bodyText/bodyHtml, threadId, messageId, labels, attachments[]) to ensure reliable field mapping across subsequent nodes.         |
| Create Zendesk Ticket| Zendesk API             | Create Zendesk ticket from email   | Normalize Gmail Data  | Format Sheet Data       | Creates a Zendesk ticket using normalized fields: requester (email), subject, description (plain text or HTML), optional priority/tags/group/assignee. Outputs ticketId, ticketUrl, status, and requesterId for logging.    |
| Format Sheet Data    | Code                    | Format combined data for Sheets    | Create Zendesk Ticket | Log to Google Sheets    | Builds a tidy row object combining Gmail and Zendesk outputs (e.g., timestamp, requesterEmail, subject, ticketId, ticketUrl, status, threadId, labels, workflowRunId). Ensures column order and default values match the Sheet schema. |
| Log to Google Sheets | Google Sheets API        | Append or update ticket record     | Format Sheet Data     | None                   | Writes the row to the target spreadsheet—appending new records or updating existing ones using a unique key (e.g., messageId or ticketId). Confirms success and surfaces any write conflicts or missing columns.           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "Gmail Trigger" Node:**  
   - Type: Gmail Trigger  
   - Set credentials with a valid Gmail OAuth2 connection.  
   - Under Filters, add a label filter with the Gmail label ID you want to monitor.  
   - Set poll interval to every minute.  
   - No input connections; this node triggers the workflow.

2. **Create "Normalize Gmail Data" Node:**  
   - Type: Code Node  
   - Connect input from "Gmail Trigger".  
   - Paste the provided JavaScript code to extract and normalize fields: requester email, name, subject, description (prefer plain text, fallback to stripped HTML), priority detection for "urgent", timestamp, and original message ID.  
   - No special credentials needed.  
   - This node outputs normalized JSON for the next step.

3. **Create "Create Zendesk Ticket" Node:**  
   - Type: Zendesk Node  
   - Connect input from "Normalize Gmail Data".  
   - Configure Zendesk API credentials with permissions to create tickets.  
   - Set operation to create a new ticket.  
   - Map fields:  
     - Description: `={{ $json.description }}`  
     - Subject: `={{ $json.subject }}`  
     - Tags: `={{ $json.priority }}` (uses priority as tag)  
   - This node outputs created ticket metadata.

4. **Create "Format Sheet Data" Node:**  
   - Type: Code Node  
   - Connect input from "Create Zendesk Ticket".  
   - Paste JavaScript that:  
     - Combines incoming Gmail normalized data and Zendesk ticket data.  
     - Parses Zendesk URL to extract subdomain and build an agent ticket URL.  
     - Constructs a structured object with fields: ticket_id, ticket_url, api_url, subject, requester_name/email, source_channel, original_id, priority, status, created_timestamp, zendesk_created_at, description_preview (first 100 chars), tags.  
   - No credentials needed.

5. **Create "Log to Google Sheets" Node:**  
   - Type: Google Sheets Node  
   - Connect input from "Format Sheet Data".  
   - Configure Google Sheets OAuth2 credentials with access to the target spreadsheet.  
   - Set operation to "Append or Update".  
   - Specify the spreadsheet document ID and sheet name (by gid or name).  
   - Define the columns and map each field from the incoming JSON accordingly:  
     - Ticket ID, Ticket URL, Subject, Priority, Status, Created Timestamp, Zendesk Created At, Description Preview, Tags, etc.  
   - Set "Ticket ID" as the unique matching key to avoid duplicates.  
   - No output connections (terminal node).

---

### 5. General Notes & Resources

| Note Content                                                                                                             | Context or Link                                                                                                         |
|--------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------|
| The workflow relies on specific Gmail labels to filter incoming emails; ensure label IDs are correctly configured.       | Gmail label settings                                                                                                    |
| Zendesk ticket URLs are dynamically constructed by parsing the API response URL; adjust subdomain fallback accordingly.| The fallback URL uses `softwarecompany-66332` as a placeholder subdomain; replace with your actual Zendesk subdomain.   |
| Google Sheets document must be shared with the OAuth user/service account to allow write operations.                     | Google Sheets permissions                                                                                               |
| Priority detection is based on the presence of the word "urgent" in subject or body; this can be expanded or customized. | Priority detection logic in "Normalize Gmail Data" node                                                                 |
| See official n8n documentation for Gmail Trigger, Zendesk, and Google Sheets nodes for advanced credential and error handling setup. | https://docs.n8n.io/integrations/builtin/nodes/n8n-nodes-base.gmailTrigger/ <br> https://docs.n8n.io/integrations/builtin/nodes/n8n-nodes-base.zendesk/ <br> https://docs.n8n.io/integrations/builtin/nodes/n8n-nodes-base.googleSheets/ |

---

**Disclaimer:** The provided workflow originates exclusively from an automated n8n workflow. It strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.