Generate Weekly Document Digests from Google Docs with GPT-4 and Email Delivery

https://n8nworkflows.xyz/workflows/generate-weekly-document-digests-from-google-docs-with-gpt-4-and-email-delivery-8039


# Generate Weekly Document Digests from Google Docs with GPT-4 and Email Delivery

### 1. Workflow Overview

This workflow automates the generation and email delivery of a weekly summary digest of updates from selected Google Docs documents. It is designed for teams needing consolidated weekly updates from multiple documents such as project statuses, meeting notes, and team updates. The workflow fetches specified Google Docs, checks if they were updated in the past week, extracts and aggregates their content, generates a professional summary using GPT-4, and emails the digest to configured recipients every Monday at 9 AM UTC.

Logical blocks include:

- **1.1 Setup & Trigger:** Defines the schedule and setup instructions.
- **1.2 Document Preparation:** Lists monitored Google Docs and determines the date range.
- **1.3 Document Retrieval:** Fetches content and metadata for each document.
- **1.4 Document Processing:** Extracts text, checks update status, and prepares structured data.
- **1.5 Aggregation and AI Summarization:** Aggregates updated documents’ content and generates a summary with GPT-4.
- **1.6 Email Preparation & Delivery:** Prepares the email content and sends it to the recipients.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Setup & Trigger

**Overview:**  
This block contains user instructions to configure the workflow and the time-based trigger that initiates the workflow every Monday at 9 AM UTC.

**Nodes Involved:**  
- Setup Instructions (Sticky Note)  
- Weekly Monday 9AM Trigger (Cron)

**Node Details:**

- **Setup Instructions (StickyNote):**  
  - Type: Sticky Note  
  - Role: Provides detailed configuration steps for Google Docs, OpenAI API keys, email recipients, and scheduling.  
  - Content includes API credential setup, document ID configuration, and schedule details.  
  - No inputs or outputs.  
  - Edge cases: None, informational only.

- **Weekly Monday 9AM Trigger (Cron):**  
  - Type: Cron Trigger  
  - Role: Starts the workflow every Monday at 09:00 UTC.  
  - Configuration: Cron expression “0 9 * * 1” (minute 0, hour 9, any day, any month, Monday).  
  - Outputs trigger signal; no inputs.  
  - Version-specific: Compatible with n8n Cron node version 1.  
  - Edge cases: Workflow will not trigger outside specified time; cron expression must be adjusted for different time zones if necessary.

---

#### 1.2 Document Preparation

**Overview:**  
Prepares a list of Google Docs to monitor and calculates the date range representing the last week for update checks.

**Nodes Involved:**  
- Prepare Docs List (Code)

**Node Details:**

- **Prepare Docs List (Code):**  
  - Type: Code (JavaScript)  
  - Role: Defines an array of Google Docs with IDs, names, and categories to monitor. Calculates the date range from now to one week ago.  
  - Key variables: `docsToMonitor` (hardcoded docs list), `lastWeek` and `now` (date range).  
  - Outputs each doc as separate JSON item with its metadata for parallel processing.  
  - Inputs: Trigger signal from Cron node.  
  - Outputs: JSON items for each doc to be fetched next.  
  - Edge cases: Missing or incorrect document IDs will lead to API errors downstream; date calculation assumes UTC timezone.

---

#### 1.3 Document Retrieval

**Overview:**  
Fetches the full content and metadata for each document identified in the previous block.

**Nodes Involved:**  
- Get Doc Content (Google Docs)  
- Get Doc Metadata (Google Drive)

**Node Details:**

- **Get Doc Content (Google Docs):**  
  - Type: Google Docs node  
  - Role: Retrieves the entire content of the document by file ID.  
  - Configuration: Uses OAuth2 authentication with Google credentials, fileId taken dynamically from input JSON (`{{$json.id}}`).  
  - Inputs: From Prepare Docs List node, one document per execution.  
  - Outputs: Full Google Docs API response with document body content.  
  - Edge cases: API quota limits, invalid or inaccessible document ID, OAuth token expiration.

- **Get Doc Metadata (Google Drive):**  
  - Type: Google Drive node  
  - Role: Retrieves metadata fields (`modifiedTime`, `lastModifyingUser`, `version`) for the document.  
  - Configuration: OAuth2, dynamic fileId from input JSON (`{{$json.id}}`), fields limited to relevant metadata to optimize performance.  
  - Inputs: Same as Get Doc Content, parallel execution.  
  - Outputs: Metadata JSON for use in update checks.  
  - Edge cases: API errors, permissions issues, missing metadata fields.

---

#### 1.4 Document Processing

**Overview:**  
Combines content and metadata to verify if the document was updated in the last week, extracts and cleans text content, and prepares a structured summary for each document.

**Nodes Involved:**  
- Process Doc Data (Code)

**Node Details:**

- **Process Doc Data (Code):**  
  - Type: Code (JavaScript)  
  - Role:  
    - Receives content and metadata from prior nodes.  
    - Checks if the document was modified in the past week based on `modifiedTime`.  
    - Extracts plain text from nested Google Docs structure, concatenating text runs.  
    - Cleans whitespace and trims content.  
    - Calculates word count and prepares a short preview.  
    - Outputs structured JSON with all key info including `was_updated` flag.  
  - Inputs: Content from "Get Doc Content" and metadata from "Get Doc Metadata", plus original doc info.  
  - Outputs: Detailed doc object if updated; returns `null` (filters out) if not updated.  
  - Edge cases:  
    - Documents with no content or malformed structure may yield empty text.  
    - If no update in last week, the document is skipped (returns null which downstream nodes must handle).  
    - Expression failures if input JSON is missing expected fields.

---

#### 1.5 Aggregation and AI Summarization

**Overview:**  
Aggregates all updated documents, prepares a combined text prompt, and sends it to OpenAI GPT-4 to generate a professional weekly summary email.

**Nodes Involved:**  
- Aggregate Updated Docs (Code)  
- Generate AI Summary (OpenAI)

**Node Details:**

- **Aggregate Updated Docs (Code):**  
  - Type: Code  
  - Role:  
    - Receives all processed docs.  
    - Filters to include only those updated this week.  
    - If none updated, returns a flag indicating no updates and a placeholder message.  
    - Otherwise, creates a combined textual content string formatted with document details and content for AI input.  
    - Prepares metadata summary with doc names, categories, last modifier, and date.  
  - Inputs: From Process Doc Data node (multiple docs).  
  - Outputs: Aggregated JSON including combined content for AI, list of updated docs, and date range.  
  - Edge cases: Empty updated docs list leads to skipping AI call.  
  - Logging included for debugging.

- **Generate AI Summary (OpenAI):**  
  - Type: OpenAI (Chat Completion) node  
  - Role: Sends the aggregated content to GPT-4 model to create a concise, professional weekly document update summary formatted as a business email.  
  - Configuration:  
    - Model: GPT-4  
    - System message defines assistant role and output style.  
    - User message contains combined document content and instructions for summary structure (Executive Summary, Key Updates, Changes, Action Items, Next Week Focus).  
    - Parameters: max_tokens=1500, temperature=0.3 for focused output.  
  - Inputs: Aggregated content from prior node.  
  - Outputs: AI-generated text summary in response.  
  - Edge cases:  
    - API rate limits or failures.  
    - Large input size could exceed token limits.  
    - Incomplete or malformed input may cause unfocused summaries.

---

#### 1.6 Email Preparation & Delivery

**Overview:**  
Prepares the final email content (text and HTML) using the AI summary and document metadata, and sends the email to configured recipients via Gmail.

**Nodes Involved:**  
- Prepare Email Content (Code)  
- Send Summary Email (Gmail)

**Node Details:**

- **Prepare Email Content (Code):**  
  - Type: Code  
  - Role:  
    - Checks if updates exist; if none, prepares a no-update email with a simple message.  
    - Otherwise, extracts AI summary text and builds a professional email body and HTML version with sections, formatted lists, and styling.  
    - Sets email subject line with date range.  
    - Includes footer note about automatic generation and instructions for modification.  
  - Inputs: JSON from Aggregate Updated Docs and AI summary text from Generate AI Summary.  
  - Outputs: JSON containing `send_email` flag, `subject`, `body`, and `html_body`.  
  - Edge cases: If AI summary missing or empty, email content may be incomplete.  
  - Logs summary info for audit.

- **Send Summary Email (Gmail):**  
  - Type: Gmail node  
  - Role: Sends the prepared email to specified recipients.  
  - Configuration:  
    - Recipients: `team@yourcompany.com,manager@yourcompany.com` (comma-separated).  
    - Subject and message body taken from previous node output.  
    - Sends both plain text and HTML versions of the email.  
    - Uses Gmail OAuth2 credentials.  
  - Inputs: Email content JSON from Prepare Email Content node.  
  - Outputs: Email send status.  
  - Edge cases:  
    - Authentication failures (expired or revoked OAuth).  
    - Invalid recipient emails.  
    - Gmail API rate limits or quota issues.

---

### 3. Summary Table

| Node Name               | Node Type           | Functional Role                            | Input Node(s)           | Output Node(s)             | Sticky Note                                        |
|-------------------------|---------------------|------------------------------------------|-------------------------|----------------------------|---------------------------------------------------|
| Setup Instructions      | Sticky Note         | Provides setup and configuration guidance| None                    | None                       | 📄 **SETUP REQUIRED:** Google Docs IDs, OAuth2, OpenAI key, email recipients, schedule |
| Weekly Monday 9AM Trigger| Cron                | Triggers workflow every Monday at 9 AM UTC| None                    | Prepare Docs List          |                                                   |
| Prepare Docs List       | Code                | Defines monitored docs and date range    | Weekly Monday 9AM Trigger| Get Doc Content, Get Doc Metadata |                                                   |
| Get Doc Content         | Google Docs         | Retrieves full content of each document  | Prepare Docs List        | Process Doc Data           |                                                   |
| Get Doc Metadata        | Google Drive        | Retrieves metadata for each document     | Prepare Docs List        | Process Doc Data           |                                                   |
| Process Doc Data        | Code                | Extracts and processes doc content and metadata; filters updated docs| Get Doc Content, Get Doc Metadata | Aggregate Updated Docs     |                                                   |
| Aggregate Updated Docs  | Code                | Aggregates updated docs and prepares AI input| Process Doc Data         | Generate AI Summary        |                                                   |
| Generate AI Summary     | OpenAI (Chat)       | Generates professional weekly summary using GPT-4| Aggregate Updated Docs  | Prepare Email Content      |                                                   |
| Prepare Email Content   | Code                | Constructs email subject/body from AI summary and metadata| Generate AI Summary     | Send Summary Email         |                                                   |
| Send Summary Email      | Gmail               | Sends the summary email to recipients    | Prepare Email Content    | None                      |                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Sticky Note node named "Setup Instructions":**  
   - Enter the provided setup content including Google Docs IDs, OAuth2 setup, OpenAI key instructions, email recipient configuration, and schedule info.

2. **Create a Cron node named "Weekly Monday 9AM Trigger":**  
   - Set to trigger weekly every Monday at 9 AM UTC using cron expression `0 9 * * 1`.

3. **Create a Code node named "Prepare Docs List":**  
   - Configure JavaScript code to:  
     - Define an array of objects for Google Docs to monitor, each with `id`, `name`, and `category`.  
     - Calculate date range covering the last 7 days.  
     - Return each document as an individual JSON item for parallel processing.

4. **Create a Google Docs node named "Get Doc Content":**  
   - Set operation to "Get".  
   - Use OAuth2 authentication with valid Google credentials.  
   - Set `fileId` to `={{ $json.id }}` to dynamically fetch each doc in the list.

5. **Create a Google Drive node named "Get Doc Metadata":**  
   - Set operation to "Get".  
   - Use OAuth2 authentication with same Google credentials.  
   - Set `fileId` to `={{ $json.id }}`.  
   - Request fields: `modifiedTime,lastModifyingUser,version`.

6. **Connect "Prepare Docs List" outputs to both "Get Doc Content" and "Get Doc Metadata" nodes in parallel.**

7. **Create a Code node named "Process Doc Data":**  
   - Configure code to:  
     - Retrieve the document content and metadata from previous nodes.  
     - Check if the document was modified in the last week.  
     - Extract and clean plain text from Google Docs JSON structure.  
     - Calculate word count and prepare a preview.  
     - Output structured JSON only if the document was updated; otherwise return null to skip.

8. **Connect outputs of "Get Doc Content" and "Get Doc Metadata" to "Process Doc Data".**

9. **Create a Code node named "Aggregate Updated Docs":**  
   - Configure code to:  
     - Collect all processed docs, filter for updated ones.  
     - If no updated docs, return a message indicating no updates.  
     - Otherwise, build a combined text string summarizing all updated docs for AI.  
     - Include metadata summary and date range.

10. **Connect "Process Doc Data" output to "Aggregate Updated Docs".**

11. **Create an OpenAI node named "Generate AI Summary":**  
    - Set resource to "Chat" and operation "Create".  
    - Use GPT-4 model.  
    - Provide a system prompt defining the assistant’s role as a professional executive assistant.  
    - Pass user message with combined documents content and instructions for formatting the summary into sections.  
    - Set parameters: max_tokens = 1500, temperature = 0.3.  
    - Configure OpenAI API credentials.

12. **Connect "Aggregate Updated Docs" output to "Generate AI Summary".**

13. **Create a Code node named "Prepare Email Content":**  
    - Configure code to:  
      - Check if there are updates; if none, prepare a simple no-update email.  
      - Otherwise, extract AI summary text and build a professional email body in both text and HTML formats.  
      - Include document update list and footer note.  
      - Output JSON with send_email flag, subject, body, and html_body.

14. **Connect "Generate AI Summary" output to "Prepare Email Content".**

15. **Create a Gmail node named "Send Summary Email":**  
    - Configure operation as "Send".  
    - Set recipients (e.g., `team@yourcompany.com,manager@yourcompany.com`).  
    - Set subject, plain text message, and HTML message dynamically from previous node output.  
    - Use OAuth2 credentials for Gmail API.

16. **Connect "Prepare Email Content" output to "Send Summary Email".**

17. **Ensure all credentials (Google OAuth2 for Docs and Drive, OpenAI API key, Gmail OAuth2) are configured and tested before running.**

18. **Activate the workflow and verify it triggers on schedule, processes docs, generates summaries, and sends emails as expected.**

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                                                      |
|----------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------|
| The workflow uses GPT-4 model from OpenAI for improved summarization quality.                                         | Requires valid OpenAI API key with GPT-4 access                    |
| Google OAuth2 credentials must have Drive and Docs API access enabled; ensure API quota and permissions are sufficient.| Google Cloud Console API setup                                     |
| Email recipients and templates can be customized in the “Prepare Email Content” node.                               | Modify JavaScript code for personalized email formatting           |
| The workflow runs on UTC timezone; adjust cron expression if a different time zone is needed.                        | n8n Cron node documentation                                        |
| For detailed Google Docs API structure, see: https://developers.google.com/docs/api/reference/rest/v1/documents      | Useful for customizing text extraction                             |
| This workflow can be extended by adding more documents in the “Prepare Docs List” node or by modifying the summarization logic.|                                                                  |

---

**Disclaimer:** The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All processed data is legal and publicly accessible.