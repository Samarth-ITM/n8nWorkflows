Automated SEO Watchlist: Continuous Audits with Decodo, Gemini & Google Sheets

https://n8nworkflows.xyz/workflows/automated-seo-watchlist--continuous-audits-with-decodo--gemini---google-sheets-10689


# Automated SEO Watchlist: Continuous Audits with Decodo, Gemini & Google Sheets

### 1. Workflow Overview

This workflow, named **"Automated SEO Watchlist: Continuous Audits with Decodo, Gemini & Google Sheets"**, automates the process of auditing a list of website URLs for SEO quality and issues. It is designed for marketing teams, SEO specialists, agencies, and website owners who want continuous, hands-free SEO monitoring and actionable insights.

The workflow operates on a scheduled basis (every 5 days) and performs the following logical blocks:

- **1.1 Input Reception**: Periodically triggers and loads a list of URLs from a Google Sheets document for auditing.
- **1.2 URL Processing Loop**: Iterates over each URL individually to ensure stable and orderly processing.
- **1.3 Data Acquisition (Decodo)**: Fetches live SEO-relevant data from each URL using Decodo’s website auditing capabilities.
- **1.4 AI-Powered SEO Analysis**: Uses Google Gemini AI via LangChain nodes to analyze the Decodo data and generate a structured SEO report.
- **1.5 Output Parsing and Formatting**: Parses and formats the AI’s JSON output into concise human-readable summaries.
- **1.6 Result Storage**: Appends each URL’s SEO report data as a new row in a Google Sheets results document for historical tracking.
- **1.7 Team Notification**: Sends a Telegram message to notify the team that the latest SEO audit results are available.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block triggers the workflow automatically every 5 days and retrieves a list of URLs from a Google Sheets document to audit.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Get row(s) in sheet

- **Node Details:**

  - **Schedule Trigger**  
    - Type: `Schedule Trigger`  
    - Role: Starts the workflow every 5 days automatically.  
    - Configuration: Interval set to trigger every 5 days.  
    - Inputs: None (trigger node)  
    - Outputs: Connected to "Get row(s) in sheet".  
    - Potential Failures: Scheduler misconfiguration or n8n instance downtime.

  - **Get row(s) in sheet**  
    - Type: `Google Sheets` node  
    - Role: Reads the list of URLs to audit from a specified sheet in Google Sheets.  
    - Configuration:  
      - Document ID and Sheet Name set to the source sheet containing URLs.  
      - Operation: Read rows (exact mode not specified but presumably all rows with URLs).  
    - Credentials: Uses Google Sheets OAuth2.  
    - Inputs: From Schedule Trigger  
    - Outputs: Connected to "Loop Over Items".  
    - Potential Failures: Authentication errors, sheet access permissions, empty or malformed sheet data.

---

#### 2.2 URL Processing Loop

- **Overview:**  
  Processes each URL individually to maintain workflow stability and orderly data handling.

- **Nodes Involved:**  
  - Loop Over Items (SplitInBatches)

- **Node Details:**

  - **Loop Over Items**  
    - Type: `SplitInBatches`  
    - Role: Iterates over each URL row one at a time, ensuring sequential processing.  
    - Configuration: Default batch size (1 by default) to handle URLs individually.  
    - Inputs: From "Get row(s) in sheet"  
    - Outputs: Two branches: one to "Notify Team" and one to "Decodo".  
    - Potential Failures: Large batch sizes could cause timeouts; empty input sets.

---

#### 2.3 Data Acquisition (Decodo)

- **Overview:**  
  For each URL, fetches live SEO data including title, content, tags, speed signals, and other relevant metadata using Decodo API.

- **Nodes Involved:**  
  - Decodo

- **Node Details:**

  - **Decodo**  
    - Type: `@decodo/n8n-nodes-decodo.decodo` (third-party API integration)  
    - Role: Pulls real-time SEO data from the URL for analysis.  
    - Configuration: URL parameter dynamically set from the current item’s JSON `url` field.  
    - Credentials: Uses Decodo API credentials.  
    - Inputs: From "Loop Over Items"  
    - Outputs: Connected to "SEO Analyzer".  
    - Potential Failures: API authentication failures, rate limits, invalid URL errors, Decodo service downtime.

---

#### 2.4 AI-Powered SEO Analysis

- **Overview:**  
  Analyzes the data fetched by Decodo using Google Gemini AI via LangChain nodes to produce a structured SEO report with scoring.

- **Nodes Involved:**  
  - SEO Analyzer  
  - Google Gemini Chat Model1  
  - Structured Output Parser

- **Node Details:**

  - **SEO Analyzer**  
    - Type: `@n8n/n8n-nodes-langchain.chainLlm`  
    - Role: Processes Decodo data with AI to generate SEO audit JSON report.  
    - Configuration:  
      - Input text: JSON stringified results from Decodo node.  
      - Prompt instructions include SEO expert role, detailed SEO checks (metadata, content, performance, indexing, links, schema), and scoring rules (weights for metadata, content, performance, etc.).  
      - Output parser: Enabled to expect structured JSON.  
    - Inputs: From "Decodo" node (results)  
    - Outputs: Connected to "Mapping Result".  
    - Potential Failures: AI model request failures, malformed JSON input or output, API quota limits.

  - **Google Gemini Chat Model1**  
    - Type: `@n8n/n8n-nodes-langchain.lmChatGoogleGemini`  
    - Role: Language model call for the AI chain in "SEO Analyzer".  
    - Configuration: Default options; uses Google PaLM API credentials.  
    - Inputs: Invoked internally by "SEO Analyzer".  
    - Outputs: Connected to "Structured Output Parser".  
    - Potential Failures: API authentication or quota issues, network timeouts.

  - **Structured Output Parser**  
    - Type: `@n8n/n8n-nodes-langchain.outputParserStructured`  
    - Role: Validates and auto-fixes AI JSON output to conform to a complex SEO report schema.  
    - Configuration:  
      - AutoFix enabled.  
      - JSON schema example includes fields like URL, fetched_at timestamp, status, page summary, checks, issues, recommendations, and metrics.  
    - Inputs: From "Google Gemini Chat Model1"  
    - Outputs: Back to "SEO Analyzer" node’s output parser.  
    - Potential Failures: Parsing errors if AI output is malformed beyond repair.

---

#### 2.5 Output Parsing and Formatting

- **Overview:**  
  Converts the AI-generated JSON report into concise, human-readable summaries for checks, issues, and recommendations, preparing data for storage.

- **Nodes Involved:**  
  - Mapping Result (Code)

- **Node Details:**

  - **Mapping Result**  
    - Type: `Code` (JavaScript)  
    - Role: Transforms the structured JSON output into formatted strings for easy reading and Google Sheets storage.  
    - Configuration:  
      - Extracts arrays of checks, issues, and recommendations from JSON.  
      - Maps them into concatenated strings with key details (e.g., id, status, severity, instructions).  
      - Returns an object with timestamp, URL, and formatted text blocks.  
    - Inputs: From "SEO Analyzer" node output.  
    - Outputs: Connected to "Store Result".  
    - Potential Failures: Null or undefined data in JSON fields, unexpected array formats.

---

#### 2.6 Result Storage

- **Overview:**  
  Appends the formatted SEO audit results as new rows in a Google Sheets document, maintaining a historical log of audits.

- **Nodes Involved:**  
  - Store Result (Google Sheets)

- **Node Details:**

  - **Store Result**  
    - Type: `Google Sheets`  
    - Role: Inserts a new row into the results sheet with the URL, audit date, checks, issues, and recommendations.  
    - Configuration:  
      - Document ID and target Sheet ID set for the results sheet.  
      - Operation: Append row.  
      - Columns mapped explicitly to formatted JSON data fields (url, date, checks, issues, recommendations).  
    - Credentials: Google Sheets OAuth2.  
    - Inputs: From "Mapping Result".  
    - Outputs: Connected back to "Loop Over Items" to continue processing next URL.  
    - Potential Failures: Google Sheets API limits, permission errors, malformed data.

---

#### 2.7 Team Notification

- **Overview:**  
  Sends a Telegram message to notify the SEO or marketing team when the batch of URLs has been processed and fresh results are available.

- **Nodes Involved:**  
  - Notify Team (Telegram)

- **Node Details:**

  - **Notify Team**  
    - Type: `Telegram` node  
    - Role: Sends a text message alerting the team that new SEO audit results are ready in Google Sheets.  
    - Configuration:  
      - Text template includes current date and a reminder to check the results sheet.  
      - Chat ID configured to notify the correct Telegram group or user.  
    - Credentials: Telegram API token.  
    - Inputs: From "Loop Over Items" (secondary output branch after each item processed).  
    - Outputs: None (terminal node).  
    - Potential Failures: Telegram API errors, incorrect chat ID, network failures.

---

### 3. Summary Table

| Node Name              | Node Type                                         | Functional Role                                  | Input Node(s)            | Output Node(s)          | Sticky Note                                                                                                                             |
|------------------------|--------------------------------------------------|-------------------------------------------------|--------------------------|-------------------------|-----------------------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger       | Schedule Trigger                                 | Triggers the workflow every 5 days               | None                     | Get row(s) in sheet      |                                                                                                                                         |
| Get row(s) in sheet    | Google Sheets                                    | Reads list of URLs from Google Sheets             | Schedule Trigger          | Loop Over Items          |                                                                                                                                         |
| Loop Over Items        | SplitInBatches                                  | Processes URLs one by one                          | Get row(s) in sheet       | Notify Team, Decodo      | ## ✅ Loop Over Items<br>Takes each URL one-by-one and processes them separately. This keeps the workflow stable and organized.          |
| Notify Team            | Telegram                                         | Sends Telegram notification when batch completes | Loop Over Items           | None                    | ## Notify Team (Telegram)<br>Sends a Telegram message telling the team the analysis is complete and results are in the sheet.            |
| Decodo                 | Decodo API Integration                           | Fetches live SEO data from each URL               | Loop Over Items           | SEO Analyzer             | ## ✅ Decodo<br>Fetches live data from the website (title, content, tags, speed signals, etc.) so it can be analyzed.                    |
| SEO Analyzer           | Langchain LLM Chain                             | AI SEO analysis to produce structured JSON report | Decodo                    | Mapping Result           | ## ✅ SEO Analyzer<br>Uses AI to review the website and create an SEO report based on scoring rules (content, metadata, links, speed).   |
| Google Gemini Chat Model1 | Langchain AI Language Model                      | Invoked by SEO Analyzer to run AI model           | SEO Analyzer (internal)   | Structured Output Parser |                                                                                                                                         |
| Structured Output Parser | Langchain Output Parser                          | Cleans and validates AI JSON output                | Google Gemini Chat Model1 | SEO Analyzer (output)    | ## ✅ Structured Output Parser<br>Makes sure the AI’s answer comes back as clean, valid JSON. This prevents messy or unexpected text.   |
| Mapping Result         | Code                                            | Formats AI JSON report into readable summaries    | SEO Analyzer              | Store Result             | ## Mapping Result (Code)<br>Turns the JSON report into short, readable summaries (checks, issues, recommendations) and prepares them for Sheets. |
| Store Result           | Google Sheets                                   | Appends audit results to Google Sheets             | Mapping Result            | Loop Over Items          | ## Store Result (Google Sheets)<br>Adds a new row to the results sheet with the report for that URL, so results are saved historically.  |
| Sticky Note            | Sticky Note                                     | Documentation notes                                | None                     | None                    | Covers Loop Over Items, Decodo, SEO Analyzer, Structured Output Parser blocks.                                                           |
| Sticky Note1           | Sticky Note                                     | Documentation notes                                | None                     | None                    | Covers Notify Team node.                                                                                                                 |
| Sticky Note2           | Sticky Note                                     | Documentation notes                                | None                     | None                    | Covers Store Result node.                                                                                                                |
| Sticky Note3           | Sticky Note                                     | Documentation notes                                | None                     | None                    | General workflow overview and explanation.                                                                                              |
| Sticky Note4           | Sticky Note                                     | Documentation notes                                | None                     | None                    | Covers Mapping Result node.                                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Configuration: Set to repeat every 5 days.  
   - No credentials needed.

2. **Create Google Sheets node ("Get row(s) in sheet")**  
   - Type: Google Sheets  
   - Operation: Read rows from a sheet containing URLs.  
   - Set Document ID and Sheet Name for the input URLs sheet.  
   - Attach Google Sheets OAuth2 credential.  
   - Connect Schedule Trigger output to this node.

3. **Create SplitInBatches node ("Loop Over Items")**  
   - Type: SplitInBatches  
   - Default batch size (1) to process URLs one at a time.  
   - Connect "Get row(s) in sheet" output to this node.

4. **Create Telegram node ("Notify Team")**  
   - Type: Telegram  
   - Configure Chat ID to the appropriate Telegram group or user.  
   - Configure message text:  
     ```
     ⚠️INFORMATION⚠️
     Date: {{ new Date() }}
     Kindly check the gsheets for the latest analysis result
     ```  
   - Attach Telegram API credentials.  
   - Connect one output branch of "Loop Over Items" to this node.

5. **Create Decodo node**  
   - Type: Decodo (third-party integration node)  
   - Set URL parameter to: `={{ $json.url }}` (dynamically from current batch item)  
   - Attach Decodo API credentials.  
   - Connect second output branch of "Loop Over Items" to this node.

6. **Create SEO Analyzer node**  
   - Type: Langchain Chain LLM  
   - Configure the input text as the JSON stringified Decodo output:  
     ```
     =Input
     {{ JSON.stringify($json.results[0]) }}
     ```  
   - Set prompt with detailed SEO instructions including checks and scoring weights.  
   - Enable Output Parser to expect structured JSON.  
   - Connect Decodo output to this node.

7. **Create Google Gemini Chat Model node ("Google Gemini Chat Model1")**  
   - Type: Langchain LM Chat Google Gemini  
   - Use default options.  
   - Attach Google PaLM API credentials.  
   - Connect this node to "SEO Analyzer" as AI language model.

8. **Create Structured Output Parser node**  
   - Type: Langchain Output Parser Structured  
   - Paste the complex JSON schema example for SEO report validation.  
   - Enable AutoFix option.  
   - Connect to "Google Gemini Chat Model1" output parser.

9. **Connect Structured Output Parser back to SEO Analyzer’s output parser**  
   - This completes the AI analysis and parsing chain.

10. **Create Code node ("Mapping Result")**  
    - Type: Code (JavaScript)  
    - Implement code to map JSON arrays (`checks`, `issues`, `recommendations`) into formatted multiline strings.  
    - Extract `fetched_at` as time and `final_url` as URL.  
    - Return an object with these formatted fields.  
    - Connect SEO Analyzer output to this node.

11. **Create Google Sheets node ("Store Result")**  
    - Type: Google Sheets  
    - Operation: Append row to results sheet.  
    - Configure Document ID and Sheet ID for results storage.  
    - Map columns: url, date, checks, issues, recommendations from code node output.  
    - Attach Google Sheets OAuth2 credentials.  
    - Connect "Mapping Result" output to this node.

12. **Connect "Store Result" output back to "Loop Over Items" to continue processing next URL**  
    - This creates a loop until all URLs processed.

---

### 5. General Notes & Resources

| Note Content                                                                                                                             | Context or Link                                                                                   |
|------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Automated SEO Watchlist: Continuous Audits Powered by Decodo. Automatically audits websites every 5 days, stores results in Google Sheets, and notifies the team via Telegram. Decodo provides live page data, AI analyzes SEO health, and results are summarized clearly. | See Sticky Note3 in workflow for detailed explanation.                                          |
| Loop Over Items node processes URLs one-by-one to maintain workflow stability and data integrity.                                         | Sticky Note covering Loop Over Items node.                                                      |
| Decodo node fetches live SEO data including page content, metadata, and speed signals for accurate AI analysis.                          | Sticky Note covering Decodo node.                                                               |
| SEO Analyzer uses AI prompts to perform extensive SEO checks and scoring based on best practices.                                         | Sticky Note covering SEO Analyzer node.                                                         |
| Structured Output Parser ensures AI output is valid JSON to prevent data corruption or parsing errors.                                    | Sticky Note covering Structured Output Parser node.                                             |
| Mapping Result converts complex JSON into human-readable text summaries to simplify results for storage and review.                      | Sticky Note covering Mapping Result node.                                                       |
| Store Result node appends audit data to Google Sheets for historical tracking and reporting.                                              | Sticky Note covering Store Result node.                                                         |
| Notify Team node sends Telegram alerts to keep the team informed when new audit results are available.                                    | Sticky Note covering Notify Team node.                                                          |

---

**Disclaimer:** The provided text is exclusively derived from an n8n automated workflow. It complies strictly with content policies and contains no illegal or offensive elements. All data processed is legal and public.