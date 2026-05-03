Process Customer Feedback with OpenAI, PDF Reports, Gmail & Slack Notifications

https://n8nworkflows.xyz/workflows/process-customer-feedback-with-openai--pdf-reports--gmail---slack-notifications-9489


# Process Customer Feedback with OpenAI, PDF Reports, Gmail & Slack Notifications

### 1. Workflow Overview

This workflow automates the processing of customer feedback by receiving submissions via webhook, cleaning and validating the data, performing AI-driven sentiment analysis and summarization, generating a PDF report, emailing the report to the customer, logging the data to Google Sheets, and notifying the internal team via Slack. It is designed for organizations seeking to collect structured customer feedback, enrich it with AI insights, and efficiently communicate results to both customers and internal teams.

Logical blocks and their roles:

- **1.1 Input Reception:** Webhook receives raw feedback submissions.
- **1.2 Data Cleaning & Validation:** Normalizes input data and validates email addresses.
- **1.3 AI Processing:** Uses OpenAI GPT models to analyze feedback for sentiment, highlights, suggestions, and summary.
- **1.4 AI Response Parsing:** Parses and validates the AI JSON output, maps sentiment to color codes, and merges with original data.
- **1.5 HTML Report Generation:** Builds a styled HTML report incorporating all feedback and AI analysis.
- **1.6 PDF Generation:** Converts the HTML report into a PDF using an external API.
- **1.7 PDF Metadata Processing:** Merges PDF metadata with feedback data, preparing for downstream use.
- **1.8 Email Validation & Sending:** Checks email validity and sends personalized reports to valid email addresses.
- **1.9 Data Logging:** Logs the complete feedback data and AI insights into a Google Sheets spreadsheet.
- **1.10 Team Notification:** Sends a detailed Slack message to notify the team of the new feedback.
- **1.11 Response to Submitter:** Sends a final webhook response confirming processing success.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Input Reception

- **Overview:** Receives customer feedback submissions as JSON via a webhook POST request.
- **Nodes Involved:** 
  - Webhook - Receive Feedback1
  - Sticky Note - Trigger1 (documentation note)

- **Node Details:**

  - **Webhook - Receive Feedback1**
    - Type: Webhook (HTTP POST Trigger)
    - Configured path: `/feedback-submission`
    - HTTP method: POST
    - Response mode: Uses downstream response node to reply
    - Input: Receives JSON payload with `name`, `email`, `rating`, `comments`, `suggestions`
    - Output: Raw submission JSON forwarded to cleaning node
    - Edge cases: Incoming data may be incomplete or malformed; webhook requires active workflow to generate permanent URL.

  - **Sticky Note - Trigger1**
    - Provides instructions on expected payload format and integration tips.

#### Block 1.2: Data Cleaning & Validation

- **Overview:** Cleans and normalizes incoming data, setting defaults for missing fields and validating email format.
- **Nodes Involved:** 
  - Clean & Normalize Data1 (Code)
  - Sticky Note - Clean Data1 (documentation)

- **Node Details:**

  - **Clean & Normalize Data1**
    - Type: Code (JavaScript)
    - Key logic:
      - Sets `name` to "Anonymous" if empty
      - Emails are trimmed, lowercased, and validated against regex; invalid emails cleared and flagged
      - Defaults rating to 0 if missing
      - Comments/suggestions default to placeholder text
      - Adds timestamp and unique `submissionId` (`FB-<timestamp>-<random>`)
      - Adds boolean flag `hasValidEmail`
    - Inputs: Raw webhook JSON
    - Outputs: Cleaned, consistent JSON object for AI analysis
    - Edge cases: Invalid or missing email handled gracefully; malformed data can cause unexpected blanks.

  - **Sticky Note - Clean Data1**
    - Documents the data normalization and cleaning rules.

#### Block 1.3: AI Processing

- **Overview:** Sends the cleaned feedback to OpenAI GPT (GPT-3.5-turbo or GPT-4) to analyze sentiment, highlights, suggestions, and a summary.
- **Nodes Involved:** 
  - Generate AI Summary1 (OpenAI node)
  - Sticky Note - AI Analysis1 (documentation)

- **Node Details:**

  - **Generate AI Summary1**
    - Type: OpenAI (LangChain wrapper)
    - Model: Configurable; default GPT-3.5-turbo recommended for cost-effectiveness
    - Messages: 
      - User prompt includes customer name, rating, comments, suggestions
      - System prompt instructs assistant to respond *only* with valid JSON containing keys `sentiment`, `highlights`, `suggestions`, `summary`
    - Credentials: OpenAI API key configured in n8n
    - Input: Cleaned feedback JSON
    - Output: AI JSON response string in `message.content`
    - Edge cases: OpenAI API failures, rate limits, or malformed responses; requires robust parsing downstream.

  - **Sticky Note - AI Analysis1**
    - Describes expected AI input and output JSON structure and usage.

#### Block 1.4: AI Response Parsing

- **Overview:** Parses the AI JSON response, validates required fields, maps sentiment to color codes, merges with cleaned feedback.
- **Nodes Involved:** 
  - Parse AI Response1 (Code)
  - Sticky Note - Parse Response1 (documentation)

- **Node Details:**

  - **Parse AI Response1**
    - Type: Code (JavaScript)
    - Logic:
      - Extracts `message.content` from AI response
      - Removes markdown code blocks if present
      - Parses JSON, validating presence of `sentiment`, `highlights`, `suggestions`, and `summary`
      - On parse failure, uses fallback heuristics to infer sentiment and fills missing data with fallback text
      - Maps sentiment to hex color codes (green, orange, red)
      - Ensures highlights and suggestions are arrays
      - Merges AI analysis with original feedback data
    - Input: AI response node output, plus cleaned feedback node data via reference
    - Output: Combined enriched feedback object
    - Edge cases: Malformed or incomplete AI response; fallback ensures workflow continuity.

  - **Sticky Note - Parse Response1**
    - Explains parsing steps, validation rules, and color mapping.

#### Block 1.5: HTML Report Generation

- **Overview:** Constructs a comprehensive, styled HTML report summarizing the feedback and AI analysis.
- **Nodes Involved:** 
  - Build HTML Report1 (Code)
  - Sticky Note - HTML Generation1 (documentation)

- **Node Details:**

  - **Build HTML Report1**
    - Type: Code (JavaScript)
    - Generates:
      - Header with gradient background and title
      - User info: name, email, submission ID
      - Visual star rating and sentiment badge with color coding
      - AI summary and key highlights as bullet lists
      - Original comments and suggestions in styled boxes
      - AI recommended actions as bullet list if any exist
      - Footer with report generation timestamp and thank-you note
    - Input: Parsed AI + feedback data
    - Output: Full HTML string stored in field `htmlContent`
    - Edge cases: Missing fields handled gracefully; HTML escapes assumed safe due to controlled input.

  - **Sticky Note - HTML Generation1**
    - Details report components and styling features.

#### Block 1.6: PDF Generation

- **Overview:** Converts HTML report into a downloadable PDF file via an external HTML to PDF API.
- **Nodes Involved:** 
  - HTML to PDF1 (HTMLCSSToPDF node)
  - Sticky Note - PDF Generation1 (documentation)

- **Node Details:**

  - **HTML to PDF1**
    - Type: HTML to PDF API integration
    - Input: `htmlContent` from previous node
    - Credentials: Requires API key for https://pdfmunk.com or similar
    - Output: JSON with `pdf_url`, `file_size_bytes`, `file_deletion_date`, and success flag
    - Edge cases: API failures, invalid HTML content, network issues.

  - **Sticky Note - PDF Generation1**
    - Shows example API response and explains output fields.

#### Block 1.7: PDF Metadata Processing

- **Overview:** Combines PDF metadata with feedback data, adds flags and constructs filename for downstream use.
- **Nodes Involved:** 
  - Process PDF Response1 (Code)
  - Sticky Note - Process PDF1 (documentation)

- **Node Details:**

  - **Process PDF Response1**
    - Type: Code (JavaScript)
    - Merges:
      - PDF URL, size, expiration date from API response
      - Generates filename: `Feedback-Report-[SubmissionID].pdf`
      - Adds `pdfReady: true` flag
      - Retains all previous feedback + AI data fields
    - Input: Output of HTML to PDF node and previous HTML report data
    - Output: Enriched JSON object for next steps
    - Edge cases: Missing PDF data or malformed API response.

  - **Sticky Note - Process PDF1**
    - Describes merging logic and output fields.

#### Block 1.8: Email Validation and Sending

- **Overview:** Checks if email is valid, sends personalized report email with PDF link if valid.
- **Nodes Involved:** 
  - Check Valid Email1 (If)
  - Email User Report1 (Gmail node)
  - Sticky Note - Email Validation1 (documentation)
  - Sticky Note (Step 9 Email User Report)

- **Node Details:**

  - **Check Valid Email1**
    - Type: If node (boolean condition)
    - Condition: `hasValidEmail` equals true
    - Routes valid email submissions to email node
    - Otherwise, skips emailing but continues logging and notifications
    - Edge cases: Invalid or missing emails bypass email sending

  - **Email User Report1**
    - Type: Gmail OAuth2 node
    - Sends rich HTML email to customer:
      - Thank-you message with customer name
      - Feedback rating and sentiment with color-coded highlight
      - Link to download PDF report
      - Summary of report contents
      - Submission metadata (ID and timestamp)
    - Credentials: Gmail OAuth2 configured in n8n
    - Input: Feedback + AI + PDF data
    - Output: Email sent confirmation
    - Edge cases: Gmail API limits, OAuth token expiration.

  - **Sticky Note - Email Validation1**
    - Explains boolean check and email sending branch.

  - **Sticky Note (Step 9 Email User Report)**
    - Short note describing email node purpose.

#### Block 1.9: Data Logging

- **Overview:** Logs all feedback and AI analysis data into a Google Sheets spreadsheet for tracking and future reference.
- **Nodes Involved:** 
  - Log Feedback Data1 (Google Sheets node)
  - Sticky Note - Log Data1 (documentation)

- **Node Details:**

  - **Log Feedback Data1**
    - Type: Google Sheets API node
    - Operation: Append or update row by matching on `Submission ID`
    - Columns logged:
      - Submission ID, Timestamp, Name, Email, Rating, Sentiment, Comments, Suggestions, AI Summary, PDF URL, PDF Expiration, Email Sent flag
    - Credentials: Google Sheets OAuth2 with access to the target spreadsheet and sheet
    - Input: Data from email sending node (valid email branch)
    - Output: Confirmation of row written
    - Edge cases: Sheet access permission errors, rate limits.

  - **Sticky Note - Log Data1**
    - Lists all columns and their meaning.

#### Block 1.10: Team Notification

- **Overview:** Sends a detailed notification message to a Slack channel to alert the team of new feedback, summarizing key info and linking to the report.
- **Nodes Involved:** 
  - Notify Team1 (Slack node)
  - Sticky Note - Team Alert1 (documentation)

- **Node Details:**

  - **Notify Team1**
    - Type: Slack API node
    - Sends formatted message to configured Slack channel
    - Message includes:
      - Header emoji and title
      - Customer name, email
      - Star rating and sentiment
      - AI summary, key highlights, and recommended actions
      - PDF report URL
      - Submission ID and confirmation of logging
    - Credentials: Slack OAuth2 with `chat:write`, `channels:read` scopes
    - Input: Data from Google Sheets logging node
    - Output: Message sent confirmation
    - Edge cases: Slack API limits, invalid channel ID, OAuth token expiration.

  - **Sticky Note - Team Alert1**
    - Details the message content and alternative notification options.

#### Block 1.11: Response to Submitter

- **Overview:** Sends a final JSON response back to the webhook caller, confirming successful processing and providing summary info.
- **Nodes Involved:** 
  - Send Success Response1 (Respond to Webhook node)
  - Sticky Note - Final Response1 (documentation)

- **Node Details:**

  - **Send Success Response1**
    - Type: Respond to Webhook node
    - HTTP 200 JSON response
    - Contains success flag, thank-you message, and data including submission ID, name, email, rating, sentiment, email sent status, report URL, expiration
    - Inputs: Feedback data from Google Sheets logging and email validation branches
    - Edge cases: Should only execute if all prior steps succeeded; else webhook caller may receive no or error response.

  - **Sticky Note - Final Response1**
    - Describes response JSON structure and use cases for integration.

---

### 3. Summary Table

| Node Name                  | Node Type                 | Functional Role                    | Input Node(s)                  | Output Node(s)                   | Sticky Note                                                                                         |
|----------------------------|---------------------------|----------------------------------|-------------------------------|---------------------------------|---------------------------------------------------------------------------------------------------|
| Sticky Note - Credentials Setup1 | Sticky Note              | Credentials & Setup Instructions |                               |                                 | Lists required credentials for OpenAI, Gmail OAuth2, Google Sheets OAuth2, Slack, HTML to PDF API, VerifiEmail API |
| Sticky Note - Trigger1       | Sticky Note              | Input format & webhook usage     |                               |                                 | Provides expected JSON input format and webhook integration details                                |
| Webhook - Receive Feedback1  | Webhook                  | Receive feedback submissions     |                               | Clean & Normalize Data1          |                                                                                                   |
| Sticky Note - Clean Data1    | Sticky Note              | Data cleaning instructions       |                               |                                 | Explains data normalization rules                                                                |
| Clean & Normalize Data1      | Code                     | Clean and normalize input data   | Webhook - Receive Feedback1    | Generate AI Summary1             |                                                                                                   |
| Sticky Note - AI Analysis1   | Sticky Note              | AI analysis description          |                               |                                 | Details AI input/output expectations                                                             |
| Generate AI Summary1         | OpenAI (LangChain)       | Generate AI analysis JSON        | Clean & Normalize Data1         | Parse AI Response1               |                                                                                                   |
| Sticky Note - Parse Response1| Sticky Note              | AI response parsing instructions |                               |                                 | Explains parsing, validation, color mapping logic                                                 |
| Parse AI Response1           | Code                     | Parse & validate AI JSON         | Generate AI Summary1            | Build HTML Report1               |                                                                                                   |
| Sticky Note - HTML Generation1| Sticky Note             | HTML report generation details   |                               |                                 | Explains HTML report structure                                                                   |
| Build HTML Report1           | Code                     | Generate HTML report string      | Parse AI Response1              | HTML to PDF1                    |                                                                                                   |
| Sticky Note - PDF Generation1| Sticky Note              | PDF generation API response info |                               |                                 | Shows example PDF API response                                                                   |
| HTML to PDF1                | HTMLCSS to PDF API       | Convert HTML to PDF              | Build HTML Report1              | Process PDF Response1            |                                                                                                   |
| Sticky Note - Process PDF1   | Sticky Note              | Merge PDF metadata with data     |                               |                                 | Describes PDF metadata merging                                                                   |
| Process PDF Response1        | Code                     | Merge PDF metadata               | HTML to PDF1                   | Check Valid Email1               |                                                                                                   |
| Sticky Note - Email Validation1| Sticky Note            | Email validation & routing       |                               |                                 | Explains email validation and routing logic                                                      |
| Check Valid Email1           | If                       | Conditional branch on email validity | Process PDF Response1          | Email User Report1               |                                                                                                   |
| Sticky Note (Step 9 Email User Report) | Sticky Note         | Email user report description    |                               |                                 | Describes sending email to user                                                                   |
| Email User Report1           | Gmail                    | Email personalized report        | Check Valid Email1              | Log Feedback Data1               |                                                                                                   |
| Sticky Note - Log Data1      | Sticky Note              | Google Sheets logging details    |                               |                                 | Lists columns logged to Google Sheets                                                            |
| Log Feedback Data1           | Google Sheets            | Log feedback and AI data         | Email User Report1              | Notify Team1                    |                                                                                                   |
| Sticky Note - Team Alert1    | Sticky Note              | Slack notification message       |                               |                                 | Explains Slack message content                                                                    |
| Notify Team1                | Slack                    | Notify team of new feedback      | Log Feedback Data1              | Send Success Response1           |                                                                                                   |
| Sticky Note - Final Response1| Sticky Note              | Webhook response details         |                               |                                 | Describes final JSON response structure                                                          |
| Send Success Response1       | Respond to Webhook       | Send final webhook response      | Notify Team1                   |                                 |                                                                                                   |
| Sticky Note - Testing Guide1 | Sticky Note              | Testing instructions             |                               |                                 | Provides step-by-step testing guide and sample payload                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node:**
   - Type: Webhook
   - Name: "Webhook - Receive Feedback1"
   - HTTP Method: POST
   - Path: `feedback-submission`
   - Response Mode: Use downstream node response
   - Connect this node first.

2. **Add Code Node to Clean & Normalize Data:**
   - Type: Code
   - Name: "Clean & Normalize Data1"
   - Input: Output of webhook node
   - Paste provided JavaScript code that:
     - Sets default values for missing fields
     - Normalizes email and validates format
     - Adds timestamp and unique submissionId
   - Connect webhook node → this node.

3. **Add OpenAI Node to Generate AI Summary:**
   - Type: OpenAI (LangChain)
   - Name: "Generate AI Summary1"
   - Credentials: Configure OpenAI API credentials (use GPT-3.5-turbo or GPT-4)
   - Model: Select model id (default GPT-3.5-turbo)
   - Messages:
     - User message with template including name, rating, comments, suggestions
     - System message instructing to return valid JSON only with sentiment, highlights, suggestions, summary
   - Connect Clean & Normalize Data1 → Generate AI Summary1.

4. **Add Code Node to Parse AI Response:**
   - Type: Code
   - Name: "Parse AI Response1"
   - Input: Output AI node
   - Paste parsing code that:
     - Extracts and cleans JSON from AI response
     - Validates required fields
     - Maps sentiment to colors
     - Merges with original data from Clean & Normalize Data1 node
   - Connect Generate AI Summary1 → Parse AI Response1.

5. **Add Code Node to Build HTML Report:**
   - Type: Code
   - Name: "Build HTML Report1"
   - Input: Parsed AI data
   - Paste provided HTML generation JavaScript code to create styled HTML report string in `htmlContent` field
   - Connect Parse AI Response1 → Build HTML Report1.

6. **Add HTML to PDF Node:**
   - Type: HTMLCSS to PDF API
   - Name: "HTML to PDF1"
   - Credentials: Configure API key for PDFMunk or similar HTML to PDF service
   - Parameter: Use `htmlContent` from previous node as input
   - Connect Build HTML Report1 → HTML to PDF1.

7. **Add Code Node to Process PDF Response:**
   - Type: Code
   - Name: "Process PDF Response1"
   - Input: Output of HTML to PDF node and Build HTML Report1 node (use $node references)
   - Paste code to merge PDF metadata (URL, size, expiration) and generate PDF filename, plus flag `pdfReady: true`
   - Connect HTML to PDF1 → Process PDF Response1.

8. **Add If Node to Check Valid Email:**
   - Type: If
   - Name: "Check Valid Email1"
   - Condition: Field `hasValidEmail` equals true
   - Connect Process PDF Response1 → Check Valid Email1.

9. **Add Gmail Node to Email User Report:**
   - Type: Gmail (OAuth2)
   - Name: "Email User Report1"
   - Credentials: Configure Gmail OAuth2 credentials
   - To: Use expression to send to `email` field
   - Subject: "Your Feedback Summary Report – Thank You for Sharing!"
   - Message: Paste provided rich HTML template using feedback and AI data placeholders
   - Connect "true" branch of Check Valid Email1 → Email User Report1.

10. **Add Google Sheets Node to Log Feedback Data:**
    - Type: Google Sheets (OAuth2)
    - Name: "Log Feedback Data1"
    - Credentials: Configure Google Sheets OAuth2 credentials
    - Operation: Append or update by `Submission ID`
    - Document ID: Your Google Sheet ID with sheet named "Feedback Log"
    - Map columns: Submission ID, Timestamp, Name, Email, Rating, Sentiment, Comments, Suggestions, AI Summary, PDF URL, PDF Expiration, Email Sent (Yes/No)
    - Connect Email User Report1 → Log Feedback Data1.

11. **Add Slack Node to Notify Team:**
    - Type: Slack (API)
    - Name: "Notify Team1"
    - Credentials: Configure Slack API OAuth2 with scopes `chat:write, channels:read`
    - Channel: Set channel ID for notifications
    - Message: Paste provided Slack message template with feedback and AI data placeholders
    - Connect Log Feedback Data1 → Notify Team1.

12. **Add Respond to Webhook Node for Final Response:**
    - Type: Respond to Webhook
    - Name: "Send Success Response1"
    - Response code: 200
    - Response body: JSON with success flag, message, and data fields (submissionId, name, email, rating, sentiment, emailSent, reportUrl, reportAvailableUntil)
    - Connect Notify Team1 → Send Success Response1.

13. **Connect "false" branch of Check Valid Email1:**
    - To Log Feedback Data1 directly (skipping email sending)
    - Ensures all feedback logged regardless of email validity.

14. **Add Sticky Notes:**
    - Add sticky notes at relevant places describing credential setup, input format, data cleaning, AI analysis, parsing response, HTML generation, PDF generation, email validation, logging, team notification, and final response. Use provided content from workflow for reference.

15. **Activate Workflow:**
    - Ensure credentials are configured for OpenAI, Gmail, Google Sheets, Slack, and HTML to PDF API.
    - Activate workflow to obtain permanent webhook URL.
    - Test by sending POST requests matching expected JSON format.

---

### 5. General Notes & Resources

| Note Content                                                                                                               | Context or Link                                      |
|----------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| Workflow requires API keys for OpenAI GPT models, Gmail OAuth2, Google Sheets OAuth2, Slack API, and HTML to PDF conversion. | Credentials Setup Sticky Note                        |
| Google Sheets spreadsheet named "Feedback Log" with specified columns must be created and shared with OAuth2 credentials.  | Google Sheets Setup                                 |
| Slack bot requires `chat:write` and `channels:read` scopes and must be added to the notification channel.                   | Slack Configuration                                 |
| HTML to PDF conversion uses https://pdfmunk.com API; signup and API key needed.                                              | PDF Generation API documentation                     |
| Email sent only if email address passes regex validation; otherwise, workflow logs data and notifies team without emailing. | Email Validation Logic                              |
| Final webhook response provides submission confirmation and report links for integration with external forms or apps.      | Webhook Response Node documentation                   |
| Testing Guide Sticky Note includes sample POST JSON for testing via Postman or curl.                                        | Testing instructions and sample data                 |

---

**Disclaimer:** The provided text is derived exclusively from an automated n8n workflow. It fully respects content policies and contains no illegal, offensive, or protected content. All processed data is legal and public.