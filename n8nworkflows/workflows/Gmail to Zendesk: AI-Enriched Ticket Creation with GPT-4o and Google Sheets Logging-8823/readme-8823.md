Gmail to Zendesk: AI-Enriched Ticket Creation with GPT-4o and Google Sheets Logging

https://n8nworkflows.xyz/workflows/gmail-to-zendesk--ai-enriched-ticket-creation-with-gpt-4o-and-google-sheets-logging-8823


# Gmail to Zendesk: AI-Enriched Ticket Creation with GPT-4o and Google Sheets Logging

---

## 1. Workflow Overview

This n8n workflow automates the process of converting incoming Gmail messages into prioritized Zendesk tickets enriched by AI, and logs ticket metadata into Google Sheets for monitoring and audit purposes. It is designed for customer support and helpdesk teams who want to leverage AI (GPT-4o via Azure OpenAI) to enhance ticket triaging, particularly by detecting urgency and structuring task details automatically.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Watches a specific Gmail inbox label for new emails.
- **1.2 Data Normalization:** Extracts relevant email fields and prepares a clean, consistent dataset.
- **1.3 AI Processing & Prioritization:** Uses Azure OpenAI chat model and Langchain AI Agent to analyze email content and produce structured ticket metadata including priority.
- **1.4 Ticket Creation:** Creates a Zendesk ticket using the AI-enriched data.
- **1.5 Data Formatting for Logging:** Prepares a structured dataset for logging ticket details.
- **1.6 Logging:** Appends or updates a Google Sheets document with ticket information for tracking.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception

**Overview:**  
Monitors a Gmail account label for new messages and outputs raw email data for processing.

**Nodes Involved:**  
- Gmail Trigger  
- Sticky Note (Gmail Trigger)

**Node Details:**  

- **Gmail Trigger**  
  - *Type & Role:* Trigger node that polls Gmail for new emails under a specified label.  
  - *Configuration:* Polls every minute, targets Gmail label with ID `Label_7215267856143431312`.  
  - *Credentials:* Uses OAuth2 credentials for Gmail account named "jyothi".  
  - *Input/Output:* No input; outputs full Gmail message JSON including sender, subject, body, timestamps, and raw HTML.  
  - *Failure Modes:* Possible auth token expiration, Gmail API rate limits, missing label errors.  
  - *Sticky Note Content:* "Watches a specified Gmail inbox/label for new messages. Captures sender, subject, timestamp, thread ID, and raw body/HTML for downstream processing and urgency assessment."

---

### 2.2 Data Normalization

**Overview:**  
Cleans and standardizes incoming Gmail data into a uniform schema suitable for AI processing, extracting plain text, requester info, and urgency flags.

**Nodes Involved:**  
- Normalize Gmail Data (Code Node)  
- Sticky Note (Normalize Gmail Data)

**Node Details:**  

- **Normalize Gmail Data**  
  - *Type & Role:* Code node that extracts and normalizes sender email/name, subject, description (plain text), and urgency flag from raw Gmail data.  
  - *Configuration:*  
    - Extracts `requester_email` and `requester_name` from `from.value[0]` object.  
    - Uses `text` field if available; falls back to stripping HTML tags from `textAsHtml`.  
    - Detects urgency if "urgent" keyword appears in subject or description (case-insensitive).  
    - Adds metadata fields: source ("gmail"), timestamp, original Gmail message ID, and raw JSON for traceability.  
  - *Expressions:* Pure JavaScript code working on `$input.first().json`.  
  - *Input/Output:* Inputs raw Gmail data; outputs uniform JSON with fields: subject, description, priority (urgent/normal), requester info, etc.  
  - *Failure Modes:* Missing or malformed email fields, unexpected data structures, empty email body.  
  - *Sticky Note Content:* "Cleans and standardizes the incoming email: strips signatures/quotes, extracts plain text, detects language, and maps fields (from, to, cc, subject, body, attachments) into a consistent schema for AI parsing and prioritization."

---

### 2.3 AI Processing & Prioritization

**Overview:**  
Analyzes normalized email content with Azure OpenAI GPT-4o model via Langchain to extract structured task metadata including subject, description, and priority.

**Nodes Involved:**  
- Azure OpenAI Chat Model  
- Structured Output Parser  
- AI Agent for Task Prioritization (Langchain Agent)  
- Sticky Notes (Chat Model, Create Zendesk Ticket)

**Node Details:**  

- **Azure OpenAI Chat Model**  
  - *Type & Role:* Language model node using Azure OpenAI GPT-4o-mini to analyze email content.  
  - *Configuration:* Model set to "gpt-4o-mini" with default options.  
  - *Credentials:* Azure OpenAI API credentials named "Azure Open AI account".  
  - *Input:* Receives normalized email JSON as prompt.  
  - *Output:* Raw AI-generated text for task metadata extraction.  
  - *Failure Modes:* API rate limits, network errors, invalid API keys, unexpected model responses.

- **Structured Output Parser**  
  - *Type & Role:* Parses AI model output into a structured JSON format with fields: subject, description, priority.  
  - *Configuration:* Uses a JSON schema example defining expected fields and types.  
  - *Input:* AI model text output.  
  - *Output:* Structured JSON for task creation.  
  - *Failure Modes:* Parsing errors if AI output is malformed or does not match schema.

- **AI Agent for Task Prioritization**  
  - *Type & Role:* Langchain Agent that converts email fields into actionable ticket metadata.  
  - *Configuration:*  
    - Prompt instructs agent on inputs (email_subject, email_description) and outputs (task_subject, task_description, task_priority).  
    - Priority classification rules embedded in prompt (urgent if "today", "ASAP", etc.).  
    - Output parser enabled to ensure structured JSON output.  
  - *Input:* Normalized email data as well as AI model & parser outputs chained.  
  - *Output:* Structured ticket metadata including subject, description (bullet points), and priority ("urgent"/"normal").  
  - *Failure Modes:* Logic errors if prompt misunderstood, API failures, malformed output.  
  - *Sticky Note Content (Chat Model):* "Uses Azure OpenAI to analyze the normalized email and produce structured insights: issue summary, category, requested action, and an urgency score/class (Critical/High/Medium/Low) based on keywords, sender context, timestamps, and SLA hints."  
  - *Sticky Note Content (Create Zendesk Ticket):* "Creates a Zendesk ticket using the structured data: requester, subject, description, tags, and priority set from urgency classification (Critical → urgent, High → high, etc.). Adds category and SLA hints to custom fields."

---

### 2.4 Ticket Creation

**Overview:**  
Creates a Zendesk ticket using the structured AI metadata, tagging and prioritizing the ticket accordingly.

**Nodes Involved:**  
- Create Zendesk Ticket  
- Sticky Note (Create Zendesk Ticket)

**Node Details:**  

- **Create Zendesk Ticket**  
  - *Type & Role:* Zendesk API node creating a new ticket.  
  - *Configuration:*  
    - Uses AI agent output fields for `subject` and `description`.  
    - Sets tags from `priority` field returned by AI agent (e.g., urgent, normal).  
    - Uses OAuth2 credentials for Zendesk account named "vivek".  
  - *Input:* AI agent output JSON with ticket metadata.  
  - *Output:* Zendesk ticket JSON including ticket ID, URL, status, tags.  
  - *Failure Modes:* Auth failures, API limits, malformed data causing ticket creation failure.  
  - *Sticky Note Content:* See above.

---

### 2.5 Data Formatting for Logging

**Overview:**  
Prepares a clean and consistent data row combining original email info and Zendesk ticket details for logging.

**Nodes Involved:**  
- Format Sheet Data (Code Node)  
- Sticky Note (Format Sheet Data)

**Node Details:**  

- **Format Sheet Data**  
  - *Type & Role:* Code node preparing a structured dataset for Google Sheets logging.  
  - *Configuration:*  
    - Extracts Zendesk ticket ID, URL, subject, priority, status, creation timestamps.  
    - Combines with Gmail requester name/email, source channel, original message ID.  
    - Creates a short preview of the description (first 100 chars).  
    - Constructs the Zendesk agent ticket URL from API URL or fallback string.  
  - *Input:* Outputs from Zendesk ticket creation and normalized Gmail data.  
  - *Output:* Structured JSON compatible with Google Sheets column mapping.  
  - *Failure Modes:* Missing or malformed ticket data, inability to parse URLs.

---

### 2.6 Logging

**Overview:**  
Appends or updates a Google Sheets document with ticket information for operational tracking and reporting.

**Nodes Involved:**  
- Log to Google Sheets  
- Sticky Note (Log to Google Sheets)

**Node Details:**  

- **Log to Google Sheets**  
  - *Type & Role:* Google Sheets node appending or updating rows in a spreadsheet.  
  - *Configuration:*  
    - Spreadsheet ID: `1pz1aP0OIULFoEJCI6VWged3jq1W7870btV2QOC4vn1o`  
    - Sheet: `gid=0` (Sheet1)  
    - Columns mapped: Ticket ID, Ticket URL, Subject, Priority, Status, Tags, Created Timestamp, Zendesk Created At, Description Preview, etc.  
    - Operation: Append or update (matching on Ticket ID) to avoid duplicates.  
  - *Credentials:* OAuth2 credentials for Google Sheets account `automations@techdome.ai`.  
  - *Input:* Data from Format Sheet Data node.  
  - *Failure Modes:* Auth errors, quota limits, sheet not found, schema mismatches.  
  - *Sticky Note Content:* "Appends or updates the tracking sheet with each processed email and its urgency priority. Enables auditing, SLA monitoring, and trend analysis across tickets."

---

## 3. Summary Table

| Node Name                  | Node Type                         | Functional Role                          | Input Node(s)              | Output Node(s)            | Sticky Note                                                                                                    |
|----------------------------|----------------------------------|----------------------------------------|----------------------------|---------------------------|---------------------------------------------------------------------------------------------------------------|
| Gmail Trigger              | n8n-nodes-base.gmailTrigger       | Watches Gmail label for new emails     | -                          | Normalize Gmail Data       | Watches a specified Gmail inbox/label for new messages. Captures sender, subject, timestamp, thread ID, and raw body/HTML for downstream processing and urgency assessment. |
| Normalize Gmail Data       | n8n-nodes-base.code               | Normalizes and cleans email data       | Gmail Trigger              | AI Agent for Task Prioritization | Cleans and standardizes the incoming email: strips signatures/quotes, extracts plain text, detects language, and maps fields (from, to, cc, subject, body, attachments) into a consistent schema for AI parsing and prioritization. |
| AI Agent for Task Prioritization | @n8n/n8n-nodes-langchain.agent | Converts normalized email to structured ticket metadata with priority | Normalize Gmail Data, Azure OpenAI Chat Model, Structured Output Parser | Create Zendesk Ticket         | Uses Azure OpenAI to analyze the normalized email and produce structured insights: issue summary, category, requested action, and an urgency score/class (Critical/High/Medium/Low) based on keywords, sender context, timestamps, and SLA hints. Creates a Zendesk ticket using the structured data: requester, subject, description, tags, and priority set from urgency classification (Critical → urgent, High → high, etc.). Adds category and SLA hints to custom fields. |
| Azure OpenAI Chat Model    | @n8n/n8n-nodes-langchain.lmChatAzureOpenAi | AI language model for email analysis   | Normalize Gmail Data         | AI Agent for Task Prioritization | See AI Agent for Task Prioritization note                                                                 |
| Structured Output Parser   | @n8n/n8n-nodes-langchain.outputParserStructured | Parses AI output into structured JSON | Azure OpenAI Chat Model      | AI Agent for Task Prioritization | See AI Agent for Task Prioritization note                                                                 |
| Create Zendesk Ticket      | n8n-nodes-base.zendesk           | Creates Zendesk ticket with AI data    | AI Agent for Task Prioritization | Format Sheet Data         | Creates a Zendesk ticket using the structured data: requester, subject, description, tags, and priority set from urgency classification (Critical → urgent, High → high, etc.). Adds category and SLA hints to custom fields. |
| Format Sheet Data          | n8n-nodes-base.code               | Prepares data row for Google Sheets    | Create Zendesk Ticket        | Log to Google Sheets       | Prepares a clean row for Google Sheets: received_at, sender, subject, summary, category, priority_by_urgency, zendesk_ticket_id, status, and processing time. Normalizes values for consistent reporting. |
| Log to Google Sheets       | n8n-nodes-base.googleSheets       | Logs ticket data to Google Sheets       | Format Sheet Data           | -                         | Appends or updates the tracking sheet with each processed email and its urgency priority. Enables auditing, SLA monitoring, and trend analysis across tickets. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger Node**  
   - Type: Gmail Trigger  
   - Configure OAuth2 credentials for Gmail account.  
   - Set to poll every minute.  
   - Apply filter to watch label with ID `Label_7215267856143431312`.  
   - Position node as entry point.

2. **Add Code Node "Normalize Gmail Data"**  
   - Connect Gmail Trigger output to this node.  
   - Paste JavaScript code to extract and normalize sender email, name, subject, description (prefer plain text), urgency detection logic based on "urgent" keyword in subject/body.  
   - Output JSON schema includes source, subject, description, requester_email, requester_name, priority (urgent/normal), timestamp, original_id, and raw_data.

3. **Add Azure OpenAI Chat Model Node**  
   - Connect Normalize Gmail Data output to this node.  
   - Set model to "gpt-4o-mini".  
   - Configure Azure OpenAI API credentials.  
   - No special prompt needed here since Langchain Agent will use it.

4. **Add Structured Output Parser Node**  
   - Connect Azure OpenAI Chat Model output to this node.  
   - Define JSON schema example with fields: subject, description, priority (normal by default).  
   - This node parses raw AI output into structured JSON.

5. **Add AI Agent for Task Prioritization (Langchain Agent Node)**  
   - Connect Normalize Gmail Data output to main input.  
   - Connect Azure OpenAI Chat Model to AI language model input.  
   - Connect Structured Output Parser to AI output parser input.  
   - Paste prompt text instructing agent to convert email subject and description into task metadata including subject, bullet-pointed description, and priority classification (urgent/normal).  
   - Enable output parser for structured JSON output.

6. **Add Zendesk Node "Create Zendesk Ticket"**  
   - Connect AI Agent output to this node.  
   - Configure Zendesk OAuth2 credentials for target account.  
   - Map ticket subject and description from AI agent output.  
   - Set tags to priority value from AI agent output for urgency classification.

7. **Add Code Node "Format Sheet Data"**  
   - Connect Create Zendesk Ticket output to this node.  
   - Paste JavaScript code to prepare a data object combining Zendesk ticket info (ID, URL, status, priority) with original Gmail data (requester info, source, original ID), plus a description preview and timestamps.  
   - Construct Zendesk agent ticket URL from API URL if available, fallback to fixed subdomain URL.

8. **Add Google Sheets Node "Log to Google Sheets"**  
   - Connect Format Sheet Data output to this node.  
   - Configure Google Sheets OAuth2 credentials for account with access to spreadsheet ID `1pz1aP0OIULFoEJCI6VWged3jq1W7870btV2QOC4vn1o`.  
   - Set operation to append or update rows matching by "Ticket ID".  
   - Map columns: Ticket ID, Ticket URL, Subject, Priority, Status, Tags, Created Timestamp, Zendesk Created At, Description Preview, etc.  
   - Set sheet name to `gid=0` (Sheet1).

9. **Add Sticky Notes** (Optional but recommended for documentation)  
   - Add sticky notes describing the purpose of each block (Gmail Trigger, Normalize Gmail Data, Chat Model + Agent, Create Ticket, Format Sheet Data, Logging).

10. **Test Workflow**  
    - Activate and test by sending an email labeled appropriately in Gmail.  
    - Confirm ticket creation in Zendesk with correct metadata and priority.  
    - Check Google Sheets for logged ticket entry.

---

## 5. General Notes & Resources

| Note Content                                                                                                                              | Context or Link                                                                                                      |
|-------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------|
| This workflow leverages Azure OpenAI GPT-4o-mini via Langchain integration for advanced natural language understanding and task extraction. | Azure OpenAI documentation: https://learn.microsoft.com/en-us/azure/cognitive-services/openai/                      |
| Gmail label filtering ensures only relevant emails are processed, reducing noise and API usage.                                            | Gmail label management: https://support.google.com/mail/answer/118708?hl=en                                          |
| Google Sheets logging supports SLA monitoring, trend analysis, and audit trails for support teams.                                        | Google Sheets API guide: https://developers.google.com/sheets/api                                                        |
| Zendesk ticket creation uses OAuth2 authentication; ensure credentials have required API scopes for ticket management.                    | Zendesk API docs: https://developer.zendesk.com/api-reference/ticketing/tickets/tickets/                             |
| AI priority classification is based on keyword matching and prompt-engineered logic to detect urgency and deadlines.                      | Langchain prompt engineering best practices: https://python.langchain.com/en/latest/modules/prompts/examples.html    |
| The workflow includes robust fallback logic in URL construction for Zendesk tickets to handle possible API format changes or missing data. |                                                                                                                     |

---

### Disclaimer

The provided content is derived exclusively from an automated workflow created with n8n, an integration and automation tool. The processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All manipulated data is lawful and publicly accessible.

---