Smart Email Triage with Gmail, GPT-4, Slack and Google Sheets

https://n8nworkflows.xyz/workflows/smart-email-triage-with-gmail--gpt-4--slack-and-google-sheets-9680


# Smart Email Triage with Gmail, GPT-4, Slack and Google Sheets

### 1. Workflow Overview

This workflow, titled **"Smart Email Triage with Gmail, GPT-4, Slack and Google Sheets"**, automates the processing, analysis, and routing of incoming emails from Gmail using AI-powered triage. It is designed to intelligently analyze emails, determine urgency and required actions, and notify appropriate teams via Slack, as well as log email metadata and AI insights to Google Sheets for record-keeping.

**Target Use Cases:**
- Automating email triage for teams handling high volumes of incoming messages.
- Prioritizing emails that require urgent replies or attachment reviews.
- Centralizing email insights and actions for audit and analytics.
- Using AI (GPT-4) to extract meaningful summaries and action points from email content.

**Logical Blocks:**

- **1.1 Input Reception: Gmail Trigger & Extraction**  
  Detect new emails and extract relevant data including sender, subject, body, attachments, and links.

- **1.2 Attachment Handling**  
  Determine if email has attachments and either process them for metadata or skip processing accordingly.

- **1.3 AI Processing**  
  Use GPT-4 via Azure OpenAI to analyze email content and attachments, generating structured insights and action recommendations.

- **1.4 AI Response Parsing**  
  Parse and validate the AI-generated JSON response ensuring consistent output and fallback handling.

- **1.5 Routing & Notifications**  
  Route emails based on AI action flags to different Slack channels: urgent reply, attachments review, or general information.

- **1.6 Logging**  
  Format and append processed email data and AI insights to a Google Sheets spreadsheet for tracking.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception: Gmail Trigger & Extraction

- **Overview:**  
  This block listens for new emails arriving in the Gmail inbox and extracts comprehensive data including sender, subject, body text/HTML, attachments, and links for downstream processing.

- **Nodes Involved:**  
  - Gmail Trigger  
  - Extract Email Data

- **Node Details:**

  - **Gmail Trigger**  
    - Type: Trigger node monitoring Gmail inbox.  
    - Configuration: Polls every minute for new emails using OAuth2 credentials.  
    - Input: None (trigger).  
    - Output: Raw email data including full payload, headers, attachments, thread info.  
    - Failure Cases: OAuth token expiration, Gmail API rate limits or downtime.  
    - Sticky Note: Explains polling and authentication setup.

  - **Extract Email Data**  
    - Type: Code node (JavaScript).  
    - Configuration: Custom JS extracts sender, subject, plain text and HTML bodies, attachments, message and thread IDs, received date, and extracts all HTTP(S) links from text and HTML bodies.  
    - Attachment detection is enhanced by scanning payload parts, headers, and keywords in email body text.  
    - Outputs a clean, structured JSON with extracted fields plus a truncated "bodyForAnalysis" limited to 3000 characters for AI consumption.  
    - Inputs: Raw email JSON from Gmail Trigger.  
    - Outputs: Structured email data JSON.  
    - Edge Cases: Missing fields defaulted; attachment detection may underestimate complex multipart emails; regex link extraction could miss or falsely identify links.  
    - Sticky Note: Details extraction logic and attachment detection.

#### 1.2 Attachment Handling

- **Overview:**  
  Checks if attachments exist, routes accordingly to either process attachments or skip processing. Processes attachments for metadata and partial content extraction where possible.

- **Nodes Involved:**  
  - Check Attachments (If node)  
  - Process Attachments (Code)  
  - Skip Attachments (Code)

- **Node Details:**

  - **Check Attachments**  
    - Type: If node.  
    - Configuration: Boolean check if `hasAttachments` is true.  
    - Routes to Process Attachments if true, else Skip Attachments.  
    - Inputs: Extracted email data.  
    - Edge Cases: Attachment detection relies on previous code accuracy; false negatives possible.  
    - Sticky Note: Describes routing decision based on attachment presence.

  - **Process Attachments**  
    - Type: Code node.  
    - Configuration: Parses attachments array to identify attachment types (text, PDF, Word), extracts metadata (filename, size, content type), and appends textual indicators for attachments that can be partially analyzed (e.g., text files). Combines this info with email body for AI input, truncated to 4000 characters.  
    - Inputs: Email data including attachments.  
    - Outputs: Enhanced email data including attachment metadata and combined content string.  
    - Edge Cases: Does not perform actual text extraction for PDFs or Word docs (requires external service). Attachment metadata might be incomplete.  
    - Sticky Note: Explains attachment analysis and content preparation.

  - **Skip Attachments**  
    - Type: Code node.  
    - Configuration: Sets processed attachments array to empty and uses only email body content for AI analysis. Flags attachment processing complete.  
    - Inputs: Email data without attachments.  
    - Outputs: Clean data structure consistent with attachment-processed flow.  
    - Sticky Note: Describes bypassing attachment processing.

#### 1.3 AI Processing

- **Overview:**  
  The core AI engine analyzes the email content and any processed attachment text to produce a structured summary, call to action, and insights.

- **Nodes Involved:**  
  - AI Agent  
  - Azure OpenAI Chat Model1  
  - Simple Memory1  
  - Structured Output Parser

- **Node Details:**

  - **AI Agent**  
    - Type: LangChain agent node.  
    - Configuration: Receives concatenated email subject, body text, HTML, and links as input text. Uses a system prompt instructing it to analyze email content and respond with structured JSON fields: summary, callToAction (one of "Reply Needed", "Review Attachment", "For Your Information"), insights, links, and attachments list. Output parser enabled for JSON validation.  
    - Inputs: Combined email content with attachments processed.  
    - Outputs: AI analysis output JSON.  
    - Edge Cases: AI may return malformed JSON, partial responses, or unexpected content. Requires fallback parsing.  
    - Sticky Note: Describes AI agent role and prompt details.

  - **Azure OpenAI Chat Model1**  
    - Type: LangChain Azure OpenAI GPT-4o model node.  
    - Configuration: Uses Azure OpenAI credentials, runs GPT-4o model for chat completion.  
    - Inputs: From Simple Memory and AI Agent.  
    - Outputs: AI model response.  
    - Edge Cases: API timeouts, rate limits, authentication errors.  
    - Sticky Note: Notes usage of GPT-4o.

  - **Simple Memory1**  
    - Type: LangChain memory buffer window.  
    - Configuration: Maintains a 7-message sliding window context with a custom session key `"json_review"` to preserve conversation state for better AI consistency.  
    - Inputs: None directly; connected as memory input to AI Agent.  
    - Sticky Note: Explains memory for context retention.

  - **Structured Output Parser**  
    - Type: LangChain output parser node.  
    - Configuration: Enforces structured JSON schema with required fields and validation of callToAction options.  
    - Inputs: AI model output.  
    - Outputs: Parsed and validated structured JSON.  
    - Sticky Note: Specifies JSON schema and parsing enforcement.

#### 1.4 AI Response Parsing

- **Overview:**  
  This block ensures the AI response is correctly parsed and normalized, handling multiple possible response formats and errors gracefully.

- **Nodes Involved:**  
  - Parse AI Response (Code)

- **Node Details:**

  - **Parse AI Response**  
    - Type: Code node.  
    - Configuration: Attempts to parse the AI response from various possible fields (`output`, `message.content`, `text`, or direct JSON fields). It extracts summary, callToAction, and insights, validating callToAction against allowed values. If parsing fails, sets fallback values indicating manual review required.  
    - Inputs: AI Agent output.  
    - Outputs: Email data enriched with parsed AI analysis and processing flags.  
    - Edge Cases: Parsing failures, invalid JSON, unexpected response formats.  
    - Sticky Note: Details robust parsing and fallback logic.

#### 1.5 Routing & Notifications

- **Overview:**  
  Routes emails based on AI-determined action to appropriate Slack channels: urgent reply needed, attachment review required, or general informational.

- **Nodes Involved:**  
  - Route Urgent (If)  
  - Route Attachments (If)  
  - Slack Urgent  
  - Slack Attachments  
  - Slack General

- **Node Details:**

  - **Route Urgent**  
    - Type: If node.  
    - Configuration: Checks if AI callToAction equals "Reply Needed".  
    - Routes to Slack Urgent if true, else to Route Attachments node.  
    - Sticky Note: Describes first-level priority routing.

  - **Route Attachments**  
    - Type: If node.  
    - Configuration: Checks if AI callToAction equals "Review Attachment".  
    - Routes to Slack Attachments if true, else Slack General.  
    - Sticky Note: Describes second-level routing decision.

  - **Slack Urgent**  
    - Type: Slack node.  
    - Configuration: Posts formatted message to Slack channel "#reply-needed" with urgent emoji and detailed email AI summary, sender, subject, date, and action required. Uses Slack OAuth credentials.  
    - Inputs: Routed email data with AI analysis.  
    - Outputs: Slack message response.  
    - Sticky Note: Notes urgent notification details.

  - **Slack Attachments**  
    - Type: Slack node.  
    - Configuration: Posts formatted message to Slack channel "#review-needed" with attachment emoji, email details, AI summary, insights, and extracted links. Uses Slack OAuth credentials.  
    - Inputs: Routed email with attachments needing review.  
    - Outputs: Slack message response.  
    - Sticky Note: Notes attachment review notification.

  - **Slack General**  
    - Type: Slack node.  
    - Configuration: Posts formatted message to Slack channel "#general-information" for informational emails with no urgent or attachment review action.  
    - Inputs: Routed email data.  
    - Outputs: Slack message response.  
    - Sticky Note: Notes informational notification.

#### 1.6 Logging

- **Overview:**  
  Extracts key metadata from Slack messages and logs essential email and AI analysis data into a Google Sheets document for audit and tracking.

- **Nodes Involved:**  
  - Code (Format Data for Sheets)  
  - Log to Google Sheets

- **Node Details:**

  - **Code (Format Data for Sheets)**  
    - Type: Code node.  
    - Configuration: Parses Slack message text to extract sender, subject, AI summary, action required, deduces Slack channel name, and flags attachment presence. Truncates summary to 200 chars, standardizes data fields for Google Sheets.  
    - Inputs: Slack message response JSON.  
    - Outputs: Clean JSON for sheet insertion.  
    - Sticky Note: Describes data formatting for sheets.

  - **Log to Google Sheets**  
    - Type: Google Sheets node.  
    - Configuration: Appends or updates a row in a specified Google Sheet with columns: timestamp, sender, subject, AI summary, action required, slack channel, and attachment flag. Uses OAuth2 credentials for access.  
    - Inputs: Formatted data JSON from Code node.  
    - Outputs: Google Sheets API response.  
    - Sticky Note: Notes logging and audit trail purpose.

---

### 3. Summary Table

| Node Name               | Node Type                      | Functional Role                             | Input Node(s)                 | Output Node(s)              | Sticky Note                                                                                  |
|-------------------------|--------------------------------|---------------------------------------------|------------------------------|-----------------------------|----------------------------------------------------------------------------------------------|
| Gmail Trigger           | n8n-nodes-base.gmailTrigger    | Trigger on new Gmail emails                  | None                         | Extract Email Data           | Monitors Gmail inbox, polls every minute, uses OAuth2 authentication                         |
| Extract Email Data      | n8n-nodes-base.code            | Extracts structured email data               | Gmail Trigger                | Check Attachments            | Extracts sender, subject, body, links, advanced attachment detection                         |
| Check Attachments       | n8n-nodes-base.if              | Decide if attachments present                 | Extract Email Data           | Process Attachments, Skip Attachments | Routes emails based on attachment presence                                                  |
| Process Attachments     | n8n-nodes-base.code            | Extracts attachment metadata & content       | Check Attachments (true)     | AI Agent                    | Identifies attachment types, combines with email body for AI analysis                        |
| Skip Attachments        | n8n-nodes-base.code            | Skips attachment processing                   | Check Attachments (false)    | AI Agent                    | Bypasses attachment processing, uses email body only                                        |
| AI Agent                | @n8n/n8n-nodes-langchain.agent| Runs AI analysis on email content             | Process Attachments, Skip Attachments | Parse AI Response            | AI email assistant analyzing content, structured JSON output                                |
| Azure OpenAI Chat Model1| @n8n/n8n-nodes-langchain.lmChatAzureOpenAi | GPT-4o AI model provider                      | Simple Memory1               | AI Agent                    | Uses Azure OpenAI GPT-4o model                                                             |
| Simple Memory1          | @n8n/n8n-nodes-langchain.memoryBufferWindow | Maintains AI conversation memory             | None                        | AI Agent                    | 7-message sliding window memory with custom session key                                    |
| Structured Output Parser| @n8n/n8n-nodes-langchain.outputParserStructured | Validates AI structured output                | Azure OpenAI Chat Model1     | AI Agent                    | Enforces consistent JSON output format                                                     |
| Parse AI Response       | n8n-nodes-base.code            | Parses and validates AI response              | AI Agent                    | Route Urgent                | Handles multiple AI response formats, fallback on parsing errors                           |
| Route Urgent            | n8n-nodes-base.if              | Routes urgent emails needing reply            | Parse AI Response            | Slack Urgent, Route Attachments | Routes emails with 'Reply Needed' action                                                  |
| Slack Urgent            | n8n-nodes-base.slack           | Sends urgent email notification to Slack     | Route Urgent                | Code                       | Posts urgent emails to #reply-needed Slack channel                                         |
| Route Attachments       | n8n-nodes-base.if              | Routes emails needing attachment review      | Route Urgent                | Slack Attachments, Slack General | Routes emails with 'Review Attachment' action                                             |
| Slack Attachments       | n8n-nodes-base.slack           | Sends attachment review notifications to Slack | Route Attachments           | Code                       | Posts attachment emails to #review-needed Slack channel                                    |
| Slack General           | n8n-nodes-base.slack           | Sends informational email notifications to Slack | Route Attachments           | Code                       | Posts general info emails to #general-information Slack channel                            |
| Code                    | n8n-nodes-base.code            | Formats Slack message data for Sheets logging | Slack Urgent, Slack Attachments, Slack General | Log to Google Sheets        | Extracts key info from Slack message, formats for spreadsheet                             |
| Log to Google Sheets    | n8n-nodes-base.googleSheets    | Logs email metadata and AI insights to Sheets | Code                       | None                       | Appends/updates email data in Google Sheets using OAuth2                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger Node:**  
   - Type: Gmail Trigger  
   - Configure OAuth2 credentials for Gmail.  
   - Set polling to every 1 minute.  
   - Output: Raw email data.

2. **Create Code Node "Extract Email Data":**  
   - Use JavaScript to extract sender, subject, plain text, HTML body, attachments, messageId, receivedDate, threadId.  
   - Extract all HTTP/HTTPS links from text and HTML.  
   - Implement advanced attachment detection: check attachments array, payload parts, headers, and keywords in text.  
   - Prepare a truncated "bodyForAnalysis" string (max 3000 chars).  
   - Connect input from Gmail Trigger.

3. **Create If Node "Check Attachments":**  
   - Condition: `$json.hasAttachments == true`  
   - Connect input from "Extract Email Data".  
   - True branch to "Process Attachments", False branch to "Skip Attachments".

4. **Create Code Node "Process Attachments":**  
   - Parse attachments array: collect filename, contentType, size.  
   - Identify type (text, PDF, Word). For text files, note "Text file content available". For others, note extraction requirement.  
   - Combine attachment text indicators with email bodyForAnalysis.  
   - Truncate combined content to 4000 chars.  
   - Connect input from "Check Attachments" (true branch).

5. **Create Code Node "Skip Attachments":**  
   - Set processedAttachments to empty array.  
   - Use email bodyForAnalysis only.  
   - Flag attachmentProcessingComplete true.  
   - Connect input from "Check Attachments" (false branch).

6. **Set up LangChain Memory Node "Simple Memory1":**  
   - Type: Memory Buffer Window  
   - Session Key: `"json_review"` (string)  
   - Context Window Length: 7 messages.

7. **Create LangChain Azure OpenAI Chat Model Node "Azure OpenAI Chat Model1":**  
   - Use Azure OpenAI GPT-4o model.  
   - Configure Azure OpenAI API credentials.

8. **Create LangChain Structured Output Parser Node "Structured Output Parser":**  
   - Define JSON schema example with fields: summary (string), callToAction (enum: Reply Needed, Review Attachment, For Your Information), insights (string), links (array of strings), attachments (array of strings).  
   - Connect input from Azure OpenAI Chat Model.

9. **Create LangChain Agent Node "AI Agent":**  
   - Input text constructed from subject, bodyForAnalysis, bodyHtml, and links.  
   - System message instructs to analyze email and return JSON with summary, callToAction, insights, links, attachments.  
   - Enable output parser.  
   - Connect ai_memory input from Simple Memory1, ai_languageModel from Azure OpenAI Chat Model1, and ai_outputParser from Structured Output Parser.  
   - Connect main input from "Process Attachments" and "Skip Attachments" nodes.

10. **Create Code Node "Parse AI Response":**  
    - Parse AI response safely from multiple possible fields (`output`, `message.content`, `text`, or direct JSON).  
    - Extract summary, callToAction, insights.  
    - Validate callToAction against allowed values.  
    - On error, set fallback data indicating manual review needed.  
    - Connect input from AI Agent.

11. **Create If Node "Route Urgent":**  
    - Condition: `$json.aiAnalysis.callToAction == "Reply Needed"`  
    - True branch: Slack Urgent node  
    - False branch: Route Attachments node  
    - Connect input from Parse AI Response.

12. **Create If Node "Route Attachments":**  
    - Condition: `$json.aiAnalysis.callToAction == "Review Attachment"`  
    - True branch: Slack Attachments node  
    - False branch: Slack General node  
    - Connect input from Route Urgent (false branch).

13. **Create Slack Node "Slack Urgent":**  
    - Configure Slack OAuth credentials.  
    - Select channel "#reply-needed".  
    - Message template includes urgent emoji, sender, subject, received date, AI summary, action required, insights.  
    - Connect input from Route Urgent (true branch).

14. **Create Slack Node "Slack Attachments":**  
    - Configure Slack OAuth credentials.  
    - Select channel "#review-needed".  
    - Message template includes attachment emoji, sender, subject, received date, AI summary, action required, insights, extracted links.  
    - Connect input from Route Attachments (true branch).

15. **Create Slack Node "Slack General":**  
    - Configure Slack OAuth credentials.  
    - Select channel "#general-information".  
    - Message template includes sender, subject, received date, AI summary, action required, insights.  
    - Connect input from Route Attachments (false branch).

16. **Create Code Node "Format Data for Sheets":**  
    - Parse Slack message text for sender, subject, AI summary, action required.  
    - Determine Slack channel name based on message content.  
    - Flag if attachments present.  
    - Limit summary length to 200 chars.  
    - Connect input from Slack Urgent, Slack Attachments, Slack General.

17. **Create Google Sheets Node "Log to Google Sheets":**  
    - Configure OAuth2 credentials for Google Sheets.  
    - Set document ID and sheet name.  
    - Append or update rows with columns: timestamp, sender, subject, ai_summary, action_required, slack_channel, has_attachments.  
    - Connect input from Code node "Format Data for Sheets".

---

### 5. General Notes & Resources

| Note Content                                                                                  | Context or Link                                                                                  |
|-----------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Workflow uses Azure OpenAI GPT-4o model for AI email analysis.                                | Requires Azure OpenAI account and proper API setup.                                            |
| Slack notifications are sent to dedicated channels: #reply-needed, #review-needed, #general-information. | Slack OAuth2 credentials needed; channel IDs configured per workspace.                          |
| Google Sheets logging provides audit trail and analytics capability for processed emails.     | Google Sheets OAuth2 API credentials needed; spreadsheet must exist with expected columns.      |
| Attachment text extraction for PDFs and Word documents is noted but requires additional services. | No actual text extraction for complex file types implemented.                                  |
| AI output parser enforces strict JSON schema for downstream reliability.                      | Ensures consistent structured AI responses for routing logic.                                  |
| The workflow polls Gmail every minute; consider API rate limits and quotas in heavy use cases.| Gmail API usage policies should be reviewed to avoid throttling.                               |
| See LangChain and n8n documentation for advanced AI agent and memory node configuration details.| https://docs.n8n.io/integrations/builtin/nodes/n8n-nodes-langchain/                            |

---

**Disclaimer:** The workflow content originates exclusively from an automated n8n workflow and complies with all applicable content policies. No illegal or offensive data is involved.