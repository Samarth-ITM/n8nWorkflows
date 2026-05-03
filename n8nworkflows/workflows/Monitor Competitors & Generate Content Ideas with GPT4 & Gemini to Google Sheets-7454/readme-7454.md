Monitor Competitors & Generate Content Ideas with GPT4 & Gemini to Google Sheets

https://n8nworkflows.xyz/workflows/monitor-competitors---generate-content-ideas-with-gpt4---gemini-to-google-sheets-7454


# Monitor Competitors & Generate Content Ideas with GPT4 & Gemini to Google Sheets

### 1. Workflow Overview

This workflow automates weekly competitor monitoring and content ideation for elegant creators and women-led brands. It crawls competitor websites, summarizes their content using OpenAI GPT-4, generates new content ideas with Google’s Gemini model, and appends the results to a Google Sheet. Optionally, it emails a digest and sends a Telegram notification.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Trigger & Configuration Setup:** Initiates the workflow on a weekly schedule and loads user-configured parameters.
- **1.2 Competitor URL Preparation & Crawling:** Builds the list of competitor URLs and scrapes their content via Firecrawl API.
- **1.3 Content Normalization & AI Processing:** Normalizes scraped content, then processes it through OpenAI for summarization and Gemini for idea generation.
- **1.4 Data Merging & Google Sheets Append:** Merges AI outputs, compiles data rows, and appends them to Google Sheets.
- **1.5 Digest Creation & Distribution:** Builds an email digest, sends it via SMTP, and optionally notifies via Telegram.
- **1.6 Error Handling:** Catches errors from HTTP nodes and emails error notifications.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger & Configuration Setup

- **Overview:** This block triggers workflow execution weekly and sets all configurable parameters like emails, sheet IDs, AI models, and competitor URLs.
- **Nodes Involved:**  
  - Cron: Weekly (Sun 5 PM)  
  - Set: Configuration (edit me)  

- **Node Details:**

  - **Cron: Weekly (Sun 5 PM)**  
    - Type: Cron trigger  
    - Role: Triggers workflow weekly (default Sunday 5 PM)  
    - Config: Default cron schedule, no custom parameters  
    - Inputs: None  
    - Outputs: To `Set: Configuration (edit me)`  
    - Edge cases: Timezone differences may affect trigger time  

  - **Set: Configuration (edit me)**  
    - Type: Set node  
    - Role: Defines user-editable parameters such as owner email, recipient email, Telegram chat ID, Google Sheets ID/range, AI models, Firecrawl base URL, and competitor URLs list  
    - Config: Stores strings and booleans for email, Telegram, AI model names, URLs  
    - Inputs: From Cron node  
    - Outputs: To `Code: Build URL Items`  
    - Edge cases: Missing or invalid config values will cause downstream failures  

#### 1.2 Competitor URL Preparation & Crawling

- **Overview:** Converts multiline competitor URLs string into individual items and performs HTTP scraping of each URL using Firecrawl API.
- **Nodes Involved:**  
  - Code: Build URL Items  
  - HTTP: Firecrawl Scrape  

- **Node Details:**

  - **Code: Build URL Items**  
    - Type: Code  
    - Role: Parses the competitor URLs list string into an array of URL items for iteration  
    - Key Expressions: Accesses `$json.competitorUrls` from configuration; splits by line; creates JSON items per URL  
    - Inputs: From `Set: Configuration (edit me)`  
    - Outputs: To `HTTP: Firecrawl Scrape`  
    - Edge cases: Empty or malformed URL lines; must trim lines to avoid empty URLs  

  - **HTTP: Firecrawl Scrape**  
    - Type: HTTP Request  
    - Role: Calls Firecrawl API to scrape the web content of each competitor URL  
    - Config: POST to `{{$json.firecrawlBaseUrl}}/v1/scrape` with URL in the request body  
    - Credentials: Requires HTTP header auth with `Authorization: Bearer <FIRECRAWL_KEY>` credential attached externally  
    - Inputs: From `Code: Build URL Items` (multi-item)  
    - Outputs: To `Code: Normalize Content`  
    - Edge cases: HTTP errors (4xx, 5xx), timeouts, auth failures, empty scrape results  

#### 1.3 Content Normalization & AI Processing

- **Overview:** Normalizes the scraped content structure, then sends it to OpenAI for summarization and to Gemini for content idea generation. Extracts AI responses for further processing.
- **Nodes Involved:**  
  - Code: Normalize Content  
  - HTTP: OpenAI Summarize  
  - HTTP: Gemini Ideas  
  - Code: Extract OpenAI  
  - Code: Extract Gemini  

- **Node Details:**

  - **Code: Normalize Content**  
    - Type: Code  
    - Role: Cleans and standardizes Firecrawl scrape output for AI input, possibly extracting main text and metadata  
    - Inputs: From `HTTP: Firecrawl Scrape`  
    - Outputs: To both `HTTP: OpenAI Summarize` and `HTTP: Gemini Ideas` (parallel)  
    - Edge cases: Missing fields, malformed content, requires validation  

  - **HTTP: OpenAI Summarize**  
    - Type: HTTP Request  
    - Role: Sends normalized content to OpenAI GPT-4 model to generate a concise summary  
    - Config: POST to OpenAI chat completions endpoint with model `gpt-4o-mini` (default from config)  
    - Credentials: Header auth `Authorization: Bearer <OPENAI_API_KEY>` attached externally  
    - Inputs: From `Code: Normalize Content`  
    - Outputs: To `Code: Extract OpenAI`  
    - Edge cases: Rate limits, API errors, malformed prompts, timeout  

  - **HTTP: Gemini Ideas**  
    - Type: HTTP Request  
    - Role: Sends normalized content to Google Gemini model to generate new content ideas or titles  
    - Config: POST to Gemini content generation URL using model name from config (e.g., `gemini-1.5-flash`)  
    - Credentials: API key via HTTP node credentials (no keys in JSON)  
    - Inputs: From `Code: Normalize Content`  
    - Outputs: To `Code: Extract Gemini`  
    - Edge cases: API quota exceeded, invalid credentials, malformed request body  

  - **Code: Extract OpenAI**  
    - Type: Code  
    - Role: Parses and extracts the summary text from OpenAI response JSON  
    - Inputs: From `HTTP: OpenAI Summarize`  
    - Outputs: To `Merge: Summaries + Ideas` (input 0)  
    - Edge cases: Unexpected response format, missing fields  

  - **Code: Extract Gemini**  
    - Type: Code  
    - Role: Parses and extracts content ideas from Gemini response JSON  
    - Inputs: From `HTTP: Gemini Ideas`  
    - Outputs: To `Merge: Summaries + Ideas` (input 1)  
    - Edge cases: Unexpected response format, missing fields  

#### 1.4 Data Merging & Google Sheets Append

- **Overview:** Combines the OpenAI summaries and Gemini ideas per URL, compiles all relevant fields into spreadsheet rows, appends to Google Sheets.
- **Nodes Involved:**  
  - Merge: Summaries + Ideas  
  - Code: Compile Row  
  - Google Sheets: Append  

- **Node Details:**

  - **Merge: Summaries + Ideas**  
    - Type: Merge  
    - Role: Combines two streams (OpenAI summary and Gemini ideas) by position (index) so each item pair corresponds to one URL  
    - Inputs: From `Code: Extract OpenAI` (0) and `Code: Extract Gemini` (1)  
    - Outputs: To `Code: Compile Row`  
    - Edge cases: Mismatched item counts, missing data  

  - **Code: Compile Row**  
    - Type: Code  
    - Role: Formats and compiles date, URL, title, summary, and ideas into a single row array for Google Sheets insertion  
    - Inputs: From `Merge: Summaries + Ideas`  
    - Outputs: To `Google Sheets: Append`  
    - Edge cases: Data type mismatches, empty fields  

  - **Google Sheets: Append**  
    - Type: Google Sheets node  
    - Role: Appends compiled rows to specified Google Sheets spreadsheet and range  
    - Config: Uses OAuth credential; configured with `sheetsSpreadsheetId` and `sheetsRange` from config node  
    - Inputs: From `Code: Compile Row`  
    - Outputs: To `Code: Build Digest`  
    - Edge cases: OAuth token expiration, quota limits, invalid sheet ID or range  

#### 1.5 Digest Creation & Distribution

- **Overview:** Builds a digest email body from appended data, sends the email, and optionally sends a Telegram notification.
- **Nodes Involved:**  
  - Code: Build Digest  
  - Email: Send Digest  
  - IF: Telegram Enabled?  
  - Telegram: Notify  

- **Node Details:**

  - **Code: Build Digest**  
    - Type: Code  
    - Role: Constructs an email-friendly digest text summarizing competitor posts from Google Sheets appended data  
    - Inputs: From `Google Sheets: Append`  
    - Outputs: To `Email: Send Digest`  
    - Edge cases: Empty appended data, formatting issues  

  - **Email: Send Digest**  
    - Type: Email Send  
    - Role: Sends the digest email to configured recipient(s) with subject "Elegant Weekly Competitor Inspo Digest"  
    - Config: Uses SMTP credential externally attached; email body uses variables for owner email, recipient email, and built digest text  
    - Inputs: From `Code: Build Digest`  
    - Outputs: To `IF: Telegram Enabled?`  
    - Edge cases: SMTP auth failure, invalid email addresses  

  - **IF: Telegram Enabled?**  
    - Type: If  
    - Role: Conditional node checking boolean flag `sendTelegram` from config  
    - Inputs: From `Email: Send Digest`  
    - Outputs: To `Telegram: Notify` if true; no output if false  

  - **Telegram: Notify**  
    - Type: Telegram node  
    - Role: Sends a notification message to the Telegram chat ID confirming digest sent and item count  
    - Config: Uses Telegram bot token credential; chat ID from config  
    - Inputs: From `IF: Telegram Enabled?` (true branch)  
    - Outputs: None  
    - Edge cases: Invalid chat ID, bot token revoked  

#### 1.6 Error Handling

- **Overview:** Centralized error handling for HTTP nodes failures that formats error details and sends an error notification email.
- **Nodes Involved:**  
  - Catch: Error Handling  
  - Format: Error Payload  
  - Email: Error Notification  

- **Node Details:**

  - **Catch: Error Handling**  
    - Type: Set node with error trigger  
    - Role: Captures errors from HTTP nodes (`Firecrawl Scrape`, `OpenAI Summarize`, `Gemini Ideas`)  
    - Inputs: Error outputs from HTTP nodes  
    - Outputs: To `Format: Error Payload`  
    - Edge cases: Must be connected to all HTTP nodes that can fail  

  - **Format: Error Payload**  
    - Type: Code  
    - Role: Formats error message, node name, and details into email-friendly text  
    - Inputs: From `Catch: Error Handling`  
    - Outputs: To `Email: Error Notification`  

  - **Email: Error Notification**  
    - Type: Email Send  
    - Role: Sends an error report email to the configured owner email with subject indicating failure node  
    - Config: Uses SMTP credential  
    - Inputs: From `Format: Error Payload`  
    - Outputs: None  
    - Edge cases: SMTP failure or invalid email address  

---

### 3. Summary Table

| Node Name                  | Node Type            | Functional Role                          | Input Node(s)                   | Output Node(s)                   | Sticky Note                                                                                      |
|----------------------------|----------------------|----------------------------------------|--------------------------------|---------------------------------|-------------------------------------------------------------------------------------------------|
| README – Template Description | Sticky Note          | Workflow purpose and setup instructions| None                           | None                            | Elegant creators and women-led brands who want a graceful, low-lift way to watch competitors weekly. |
| Submission Checklist        | Sticky Note          | Submission requirements checklist      | None                           | None                            | Sticky notes present; no API keys hardcoded; markdown format; original work                      |
| Cron: Weekly (Sun 5 PM)     | Cron                 | Weekly trigger                         | None                           | Set: Configuration (edit me)    |                                                                                                 |
| Set: Configuration (edit me)| Set                  | Stores user parameters                  | Cron: Weekly                   | Code: Build URL Items           |                                                                                                 |
| Code: Build URL Items       | Code                 | Parses competitor URLs into individual items | Set: Configuration            | HTTP: Firecrawl Scrape          |                                                                                                 |
| HTTP: Firecrawl Scrape      | HTTP Request         | Scrapes competitor URLs using Firecrawl API | Code: Build URL Items          | Code: Normalize Content         | Attach Header Auth credential: Authorization: Bearer <FIRECRAWL_KEY>                            |
| Code: Normalize Content     | Code                 | Cleans and formats scraped content     | HTTP: Firecrawl Scrape          | HTTP: OpenAI Summarize, HTTP: Gemini Ideas |                                                                                                 |
| HTTP: OpenAI Summarize      | HTTP Request         | Generates summaries using OpenAI GPT-4 | Code: Normalize Content         | Code: Extract OpenAI            | Attach Header Auth credential: Authorization: Bearer <OPENAI_API_KEY>                           |
| HTTP: Gemini Ideas          | HTTP Request         | Generates content ideas using Gemini   | Code: Normalize Content         | Code: Extract Gemini            | Attach API key via credentials (no keys in JSON)                                               |
| Code: Extract OpenAI        | Code                 | Extracts summary text from OpenAI response | HTTP: OpenAI Summarize          | Merge: Summaries + Ideas        |                                                                                                 |
| Code: Extract Gemini        | Code                 | Extracts ideas from Gemini response     | HTTP: Gemini Ideas              | Merge: Summaries + Ideas        |                                                                                                 |
| Merge: Summaries + Ideas    | Merge                | Combines summaries and ideas by position | Code: Extract OpenAI, Code: Extract Gemini | Code: Compile Row           |                                                                                                 |
| Code: Compile Row           | Code                 | Formats data row for Sheets append      | Merge: Summaries + Ideas        | Google Sheets: Append           |                                                                                                 |
| Google Sheets: Append       | Google Sheets        | Appends compiled rows to Google Sheets  | Code: Compile Row               | Code: Build Digest              | Google Sheets OAuth credential required                                                        |
| Code: Build Digest          | Code                 | Builds email digest text                  | Google Sheets: Append           | Email: Send Digest              |                                                                                                 |
| Email: Send Digest          | Email Send           | Sends the weekly digest email             | Code: Build Digest              | IF: Telegram Enabled?           | SMTP credential required                                                                       |
| IF: Telegram Enabled?       | If                   | Checks if Telegram notifications enabled | Email: Send Digest             | Telegram: Notify                |                                                                                                 |
| Telegram: Notify            | Telegram             | Sends Telegram notification               | IF: Telegram Enabled?          | None                           | Telegram Bot token + chat ID credentials required                                              |
| Catch: Error Handling       | Set (Error Trigger)  | Catches errors from HTTP nodes            | HTTP: Firecrawl Scrape, HTTP: OpenAI Summarize, HTTP: Gemini Ideas | Format: Error Payload        | Receives error outputs; forwards to error email                                                |
| Format: Error Payload       | Code                 | Formats error details for email           | Catch: Error Handling          | Email: Error Notification       |                                                                                                 |
| Email: Error Notification   | Email Send           | Sends error notification email            | Format: Error Payload          | None                           | SMTP credential required                                                                       |
| Credentials – Attach After Import | Sticky Note          | Lists all credentials to attach after import | None                           | None                            | See detailed credential attachment instructions                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Cron node**  
   - Name: `Cron: Weekly (Sun 5 PM)`  
   - Set to trigger weekly (default Sunday 5 PM).  

2. **Add a Set node**  
   - Name: `Set: Configuration (edit me)`  
   - Add string parameters:  
     - `ownerEmail` (e.g., owner@example.com)  
     - `emailTo` (e.g., owner@example.com)  
     - `telegramChatId` (optional)  
     - `sheetsSpreadsheetId` (Google Sheets ID)  
     - `sheetsRange` (e.g., `Digest!A:F`)  
     - `geminiModel` (default `gemini-1.5-flash`)  
     - `openaiModel` (default `gpt-4o-mini`)  
     - `firecrawlBaseUrl` (default `https://api.firecrawl.dev`)  
     - `competitorUrls` (multiline string of URLs, starting with "- ")  
   - Add boolean parameter `sendTelegram` (default false).  
   - Connect `Cron: Weekly` → `Set: Configuration`.  

3. **Add a Code node**  
   - Name: `Code: Build URL Items`  
   - Implement JS code to split `competitorUrls` string into array items, trimming spaces and ignoring empty lines.  
   - Connect `Set: Configuration` → `Code: Build URL Items`.  

4. **Add HTTP Request node**  
   - Name: `HTTP: Firecrawl Scrape`  
   - Method: POST  
   - URL: `={{$json.firecrawlBaseUrl}}/v1/scrape`  
   - Attach HTTP header credential with `Authorization: Bearer <FIRECRAWL_KEY>`  
   - Send JSON body containing the URL from incoming item.  
   - Connect `Code: Build URL Items` → `HTTP: Firecrawl Scrape`.  

5. **Add a Code node**  
   - Name: `Code: Normalize Content`  
   - Implement JS code to extract and clean content from Firecrawl API response, preparing for AI input.  
   - Connect `HTTP: Firecrawl Scrape` → `Code: Normalize Content`.  

6. **Add two parallel HTTP Request nodes:**  
   - **HTTP: OpenAI Summarize**  
     - POST to `https://api.openai.com/v1/chat/completions`  
     - Use model from config `openaiModel`  
     - Attach Header auth credential: `Authorization: Bearer <OPENAI_API_KEY>`  
     - Send prompt with normalized content requesting summary.  
     - Connect `Code: Normalize Content` → `HTTP: OpenAI Summarize`.  

   - **HTTP: Gemini Ideas**  
     - POST to `https://generativelanguage.googleapis.com/v1beta/models/{{$json.geminiModel}}:generateContent`  
     - Attach API key credential (no keys in JSON)  
     - Send prompt requesting content ideas based on normalized content.  
     - Connect `Code: Normalize Content` → `HTTP: Gemini Ideas`.  

7. **Add two Code nodes to extract AI responses:**  
   - **Code: Extract OpenAI**  
     - Extract summary text from OpenAI response JSON.  
     - Connect `HTTP: OpenAI Summarize` → `Code: Extract OpenAI`.  

   - **Code: Extract Gemini**  
     - Extract content ideas from Gemini response JSON.  
     - Connect `HTTP: Gemini Ideas` → `Code: Extract Gemini`.  

8. **Add a Merge node**  
   - Name: `Merge: Summaries + Ideas`  
   - Mode: mergeByPosition  
   - Connect `Code: Extract OpenAI` to input 0 and `Code: Extract Gemini` to input 1.  

9. **Add a Code node**  
   - Name: `Code: Compile Row`  
   - Combine date, URL, title, summary, and ideas into an array matching Google Sheets columns.  
   - Connect `Merge: Summaries + Ideas` → `Code: Compile Row`.  

10. **Add Google Sheets node**  
    - Name: `Google Sheets: Append`  
    - Operation: Append  
    - Configure with OAuth credential and use `sheetsSpreadsheetId` and `sheetsRange` from config.  
    - Connect `Code: Compile Row` → `Google Sheets: Append`.  

11. **Add a Code node**  
    - Name: `Code: Build Digest`  
    - Build email body string summarizing all appended entries.  
    - Connect `Google Sheets: Append` → `Code: Build Digest`.  

12. **Add Email Send node**  
    - Name: `Email: Send Digest`  
    - Use SMTP credentials.  
    - To: `emailTo` from config  
    - From: `ownerEmail` from config  
    - Subject: "Elegant Weekly Competitor Inspo Digest"  
    - Body: includes digest from previous node  
    - Connect `Code: Build Digest` → `Email: Send Digest`.  

13. **Add If node**  
    - Name: `IF: Telegram Enabled?`  
    - Condition: boolean `sendTelegram` is true  
    - Connect `Email: Send Digest` → `IF: Telegram Enabled?`.  

14. **Add Telegram node**  
    - Name: `Telegram: Notify`  
    - Use Telegram bot token credentials and `telegramChatId` from config.  
    - Message: "✅ Elegant Competitor Digest sent ({{$json.count}} items)."  
    - Connect `IF: Telegram Enabled?` (true output) → `Telegram: Notify`.  

15. **Error Handling Setup:**  
    - Add a Set node, name it `Catch: Error Handling`, enable error trigger mode.  
    - Connect error outputs from HTTP nodes (`Firecrawl Scrape`, `OpenAI Summarize`, `Gemini Ideas`) to this node.  

16. **Add a Code node**  
    - Name: `Format: Error Payload`  
    - Format error information for email notification.  
    - Connect `Catch: Error Handling` → `Format: Error Payload`.  

17. **Add Email Send node**  
    - Name: `Email: Error Notification`  
    - Use SMTP credentials.  
    - To: `ownerEmail` from config  
    - Subject: includes failing node name  
    - Body: includes error details  
    - Connect `Format: Error Payload` → `Email: Error Notification`.  

18. **Add Sticky Notes for documentation:**  
    - Add one describing workflow purpose and setup instructions.  
    - Add one listing credentials to attach after import.  
    - Add one submission checklist.  

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                       |
|---------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| Elegant creators and women-led brands are the primary audience for this workflow.                              | README sticky note                                                                                   |
| Firecrawl API is used for web content scraping; requires HTTP header Authorization with Bearer token.         | Credentials sticky note                                                                              |
| OpenAI and Gemini APIs require credentials attached externally; keys must not be hardcoded in nodes.           | Credentials sticky note                                                                              |
| Google Sheets OAuth credential needed for appending rows.                                                     | Credentials sticky note                                                                              |
| SMTP credentials required for sending emails (digest and error notifications).                                | Credentials sticky note                                                                              |
| Telegram bot token and chat ID are optional for notifications; enable via boolean flag in configuration.      | Credentials sticky note and workflow configuration node                                             |
| Test workflow by setting Cron node to run every minute before reverting to weekly to ensure proper setup.     | README sticky note                                                                                   |
| Submission requires no hardcoded API keys and the use of markdown with H2 headings in notes and documentation. | Submission checklist sticky note                                                                    |