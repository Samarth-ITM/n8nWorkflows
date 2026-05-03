Sync HubSpot, Pipedrive & Salesforce to Google Sheets with OpenAI Deduplication

https://n8nworkflows.xyz/workflows/sync-hubspot--pipedrive---salesforce-to-google-sheets-with-openai-deduplication-6598


# Sync HubSpot, Pipedrive & Salesforce to Google Sheets with OpenAI Deduplication

### 1. Workflow Overview

This workflow automates the synchronization and deduplication of contact data from three major CRMs—HubSpot, Pipedrive, and Salesforce—into Google Sheets using AI-enhanced data processing. It is designed for companies needing to consolidate multiple CRM systems into a single, clean master database with ongoing quality reporting and error management.

The workflow is composed of the following logical blocks:

- **1.1 Input Reception:** Triggering mechanisms to start the sync either on a daily schedule or manual webhook call.
- **1.2 Configuration Setup:** Centralized definition of credentials, sheet IDs, and operational parameters.
- **1.3 Data Fetch & Preprocessing:** Parallel fetching of CRM data, consolidation, and CSV preparation.
- **1.4 AI Deduplication & Quality Analysis:** Leveraging OpenAI and a pandas MCP server to deduplicate, score, and merge CRM records.
- **1.5 Data Writing & Reporting:** Writing the cleaned master data and quality reports back to Google Sheets and notifying via Slack.
- **1.6 Error Handling:** Capturing, logging, and notifying errors encountered during the workflow execution.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block initiates the workflow either automatically on a daily basis or manually via an HTTP POST webhook.

**Nodes Involved:**  
- Daily Sync Schedule  
- Manual Sync Webhook  
- Trigger Router

**Node Details:**

- **Daily Sync Schedule**  
  - *Type:* Schedule Trigger  
  - *Role:* Automatically triggers the sync daily at 02:00 AM (cron expression "0 2 * * *").  
  - *Config Highlights:* Uses cron to define schedule interval.  
  - *Input/Output:* No input; outputs trigger event to Trigger Router.  
  - *Potential Failures:* Cron misconfiguration, system clock issues.

- **Manual Sync Webhook**  
  - *Type:* Webhook  
  - *Role:* Allows manual triggering of the sync via HTTP POST to path `/crm-sync-manual`.  
  - *Config Highlights:* HTTP method POST, response mode set to last node output.  
  - *Input/Output:* Receives external HTTP requests; outputs trigger event to Trigger Router.  
  - *Potential Failures:* Webhook URL misconfiguration, authentication if required externally.

- **Trigger Router**  
  - *Type:* Merge (chooseBranch mode)  
  - *Role:* Routes the trigger from either the scheduled or manual start point into the next block.  
  - *Config Highlights:* Chooses one branch based on input source.  
  - *Input/Output:* Inputs from Daily Sync Schedule and Manual Sync Webhook; outputs to Configuration Center.  
  - *Potential Failures:* Branch routing errors if inputs are malformed.

---

#### 2.2 Configuration Setup

**Overview:**  
Defines all essential workflow parameters, API keys, sheet IDs, endpoints, and operational settings centrally in one node for easier management.

**Nodes Involved:**  
- Configuration Center

**Node Details:**

- **Configuration Center**  
  - *Type:* Set (variable assignment)  
  - *Role:* Stores all environment variables and configuration parameters like API keys, sheet IDs, quality thresholds, batch size, Slack webhook URL, MCP server endpoint, etc.  
  - *Config Highlights:* Uses expressions to generate a unique runId timestamp, placeholders for API keys and IDs, JSON arrays for deduplication keys and required fields.  
  - *Input/Output:* Input from Trigger Router; outputs config JSON to Parallel CRM Fetcher and other downstream nodes.  
  - *Potential Failures:* Missing or incorrect API key values, misconfigured sheet IDs, invalid JSON in arrays.  
  - *Version Requirements:* None special, but expression handling requires n8n version supporting JS expressions.

---

#### 2.3 Data Fetch & Preprocessing

**Overview:**  
Collects data from HubSpot, Pipedrive, and Salesforce (presumed external steps), merges all data, and prepares CSV-formatted content for further processing.

**Nodes Involved:**  
- Parallel CRM Fetcher

**Node Details:**

- **Parallel CRM Fetcher**  
  - *Type:* Code (JavaScript)  
  - *Role:* Combines CRM data arrays, maps them into a uniform structure, generates CSV content manually, and prepares metadata including record counts and error tracking.  
  - *Config Highlights:* Assumes input variables `hubspotData`, `pipedriveData`, `salesforceData`, and `errors` are available in the environment or previous nodes (likely from external API fetch nodes not shown). Creates CSV string manually without external libraries.  
  - *Input/Output:* Input from Configuration Center; outputs combined data object with CSV content and metadata to CRM Data Processing Agent.  
  - *Potential Failures:* Missing CRM data variables, malformed record fields, CSV encoding issues.  
  - *Edge Cases:* Empty CRM arrays, data with missing fields, large data volume handling.

---

#### 2.4 AI Deduplication & Quality Analysis

**Overview:**  
Processes the combined CRM data using a multi-step AI pipeline with LangChain agents and an MCP server to perform deduplication, quality scoring, and merging of records.

**Nodes Involved:**  
- CRM Data Processing Agent  
- OpenAI Chat Model  
- pandas-mcp-server

**Node Details:**

- **CRM Data Processing Agent**  
  - *Type:* LangChain Agent (AI processing)  
  - *Role:* Orchestrates AI logic for analyzing data quality, deduplicating records, merging duplicates, and generating cleansing reports.  
  - *Config Highlights:* Uses a detailed prompt instructing to use pandas MCP tools for data analysis and deduplication based on email, companyName, and phone fields. Returns structured results including quality scores and merge history.  
  - *Input/Output:* Input from Parallel CRM Fetcher; outputs deduplicated data to Master Database Writer.  
  - *Version Requirements:* Requires LangChain-compatible n8n version and configured AI credentials.  
  - *Edge Cases:* AI model rate limits, timeouts, incomplete data causing analysis failure.

- **OpenAI Chat Model**  
  - *Type:* LangChain OpenAI Chat Model  
  - *Role:* Provides GPT-4o-mini model for natural language processing tasks within the agent.  
  - *Config Highlights:* Uses OpenAI API credentials; model set to "gpt-4o-mini".  
  - *Input/Output:* Connected as AI language model for CRM Data Processing Agent.  
  - *Potential Failures:* API key invalid, rate limits, network issues.

- **pandas-mcp-server**  
  - *Type:* LangChain MCP Client Tool  
  - *Role:* External server endpoint to run pandas-based data processing for complex deduplication and quality analysis.  
  - *Config Highlights:* Server endpoint URL configured from environment variable; communicates over SSE.  
  - *Input/Output:* Connected as AI tool for CRM Data Processing Agent.  
  - *Edge Cases:* Server downtime, request timeouts, invalid data inputs.

---

#### 2.5 Data Writing & Reporting

**Overview:**  
Writes the cleaned and deduplicated CRM dataset to Google Sheets, generates a quality report, sends Slack notifications, and appends the report to a separate sheet.

**Nodes Involved:**  
- Master Database Writer  
- Quality Report Generator  
- Report Writer

**Node Details:**

- **Master Database Writer**  
  - *Type:* Google Sheets  
  - *Role:* Appends or updates deduplicated CRM records into the "Master_CRM_Data" sheet.  
  - *Config Highlights:* Uses auto-mapping mode, matches rows on the "email" column, uses service account credentials, writes with "USER_ENTERED" formatting. Sheet and document IDs are dynamically loaded from Configuration Center.  
  - *Input/Output:* Input from CRM Data Processing Agent; outputs to Quality Report Generator.  
  - *Potential Failures:* Google API authentication errors, missing sheet or incorrect ID, rate limits, malformed data.

- **Quality Report Generator**  
  - *Type:* Code (JavaScript)  
  - *Role:* Aggregates statistics from processed data such as record counts, duplicates, quality scores, and errors. Creates a Slack message payload and sends notification if configured. Prepares structured data for report writing.  
  - *Config Highlights:* Calculates various metrics, analyzes top quality issues, and conditionally sends Slack notifications via HTTP POST.  
  - *Input/Output:* Inputs from Master Database Writer and Parallel CRM Fetcher; outputs structured report data to Report Writer.  
  - *Potential Failures:* HTTP request failures to Slack, missing config variables, division by zero if no processed records.

- **Report Writer**  
  - *Type:* Google Sheets  
  - *Role:* Appends quality report data into the "Quality_Reports" sheet of the master spreadsheet.  
  - *Config Highlights:* Auto-maps input data, writes in append mode, uses same service account as Master Database Writer. Sheet name and document ID reference Configuration Center.  
  - *Input/Output:* Input from Quality Report Generator.  
  - *Potential Failures:* Same as Master Database Writer.

---

#### 2.6 Error Handling

**Overview:**  
Captures workflow errors, logs detailed information, sends Slack notifications, and optionally writes error logs to Google Sheets.

**Nodes Involved:**  
- Error Handler  
- Error Processor

**Node Details:**

- **Error Handler**  
  - *Type:* Error Trigger  
  - *Role:* Listens for any workflow errors and triggers error processing.  
  - *Config Highlights:* No parameters; triggers downstream Error Processor.  
  - *Input/Output:* Outputs error data to Error Processor.  
  - *Potential Failures:* None inherent; relies on n8n error system.

- **Error Processor**  
  - *Type:* Code (JavaScript)  
  - *Role:* Processes error metadata, logs details to console, sends Slack error notifications if configured, and writes errors to a Google Sheet if possible.  
  - *Config Highlights:* Collects error context including node name, message, stack trace, runId, and timestamps. Sends formatted Slack message asynchronously.  
  - *Input/Output:* Input from Error Handler; outputs error log for possible Google Sheets writing (not connected here).  
  - *Potential Failures:* Slack notification failures, missing config, error in error handling logic.

---

### 3. Summary Table

| Node Name                 | Node Type                 | Functional Role                         | Input Node(s)           | Output Node(s)           | Sticky Note                                                                                                                      |
|---------------------------|---------------------------|---------------------------------------|-------------------------|--------------------------|---------------------------------------------------------------------------------------------------------------------------------|
| Daily Sync Schedule        | Schedule Trigger          | Scheduled workflow trigger             | -                       | Trigger Router           |                                                                                                                                 |
| Manual Sync Webhook        | Webhook                   | Manual workflow trigger via HTTP POST | -                       | Trigger Router           |                                                                                                                                 |
| Trigger Router            | Merge (chooseBranch mode)  | Routes trigger from schedule or webhook | Daily Sync Schedule, Manual Sync Webhook | Configuration Center    |                                                                                                                                 |
| Configuration Center       | Set                       | Centralized config and credentials     | Trigger Router           | Parallel CRM Fetcher      | 🛠️ Multi-CRM Data Sync & AI Deduplication<br>Setup instructions and environment variables details                             |
| Parallel CRM Fetcher       | Code                      | Combines CRM data and prepares CSV    | Configuration Center     | CRM Data Processing Agent |                                                                                                                                 |
| CRM Data Processing Agent  | LangChain Agent           | AI-powered deduplication and scoring  | Parallel CRM Fetcher     | Master Database Writer    |                                                                                                                                 |
| OpenAI Chat Model          | LangChain OpenAI Chat     | Provides GPT model for AI agent        | -                       | CRM Data Processing Agent |                                                                                                                                 |
| pandas-mcp-server          | LangChain MCP Client Tool | External pandas server for data ops    | -                       | CRM Data Processing Agent | 🔧 Required MCP Servers<br>GitHub link and installation instructions for pandas-mcp-server                                       |
| Master Database Writer     | Google Sheets             | Writes deduplicated CRM data           | CRM Data Processing Agent | Quality Report Generator | 📊 Google Sheets Configuration<br>Master_CRM_Data sheet structure and service account permissions                               |
| Quality Report Generator   | Code                      | Aggregates quality metrics, sends Slack | Master Database Writer, Parallel CRM Fetcher | Report Writer           |                                                                                                                                 |
| Report Writer             | Google Sheets             | Writes quality reports to sheet        | Quality Report Generator | -                        | 📊 Google Sheets Configuration<br>Quality_Reports sheet structure and service account permissions                              |
| Error Handler             | Error Trigger             | Captures workflow errors               | -                       | Error Processor          | ## 🚨 Error Management System                                                                                                   |
| Error Processor           | Code                      | Logs, notifies, and records errors     | Error Handler            | -                        |                                                                                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**  
   - Add a **Schedule Trigger** node named "Daily Sync Schedule":  
     - Set cron expression to `0 2 * * *` to run daily at 2 AM.  
   - Add a **Webhook** node named "Manual Sync Webhook":  
     - Set HTTP method to POST.  
     - Set path to `crm-sync-manual`.  
     - Set response mode to "last node".

2. **Add a Merge Node for Trigger Routing:**  
   - Add a **Merge** node named "Trigger Router".  
   - Set mode to "chooseBranch", output "input1".  
   - Connect "Daily Sync Schedule" (main output) to first input of "Trigger Router".  
   - Connect "Manual Sync Webhook" (main output) to second input of "Trigger Router".

3. **Create Configuration Center Node:**  
   - Add a **Set** node named "Configuration Center".  
   - Configure variables:  
     - `runId` = current datetime formatted as "yyyy-MM-dd-HHmmss".  
     - `stagingSheetId`, `masterSheetId` = placeholders for Google Sheets IDs.  
     - `hubspotApiKey`, `pipedriveApiKey`, `salesforceAccessToken` = placeholders for CRM API keys.  
     - `salesforceInstance` = Salesforce instance domain or URL.  
     - `batchSize` = 100 (default).  
     - `qualityThreshold` = 0.7 (default).  
     - `deduplicationKeys` = JSON array `["email", "companyName", "phone"]`.  
     - `requiredFields` = JSON array `["email", "firstName", "lastName", "companyName"]`.  
     - `slackWebhookUrl` = placeholder Slack webhook URL.  
     - `mcpServerEndpoint` = `http://localhost:8000` or your MCP server URL.  
   - Connect "Trigger Router" output to this node.

4. **Add Parallel CRM Fetcher Node:**  
   - Add a **Code** node named "Parallel CRM Fetcher".  
   - Paste JavaScript code to combine CRM data arrays (`hubspotData`, `pipedriveData`, `salesforceData`), map fields, and generate CSV content as in the provided code.  
   - Input: from "Configuration Center".

5. **Add AI Processing Nodes:**  
   - Add **LangChain OpenAI Chat** node named "OpenAI Chat Model":  
     - Select OpenAI API credentials.  
     - Choose model "gpt-4o-mini".  
   - Add **LangChain MCP Client Tool** node named "pandas-mcp-server":  
     - Set SSE endpoint to MCP server URL (e.g., `http://localhost:8000`).  
   - Add **LangChain Agent** node named "CRM Data Processing Agent":  
     - Paste detailed prompt instructing AI to perform CRM deduplication and quality scoring.  
     - Connect "OpenAI Chat Model" as language model.  
     - Connect "pandas-mcp-server" as AI tool.  
     - Input from "Parallel CRM Fetcher".

6. **Add Google Sheets Writer for Master Data:**  
   - Add **Google Sheets** node named "Master Database Writer":  
     - Operation: Append or update.  
     - Sheet name: "Master_CRM_Data".  
     - Document ID: use expression to get `masterSheetId` from Configuration Center.  
     - Authentication: Google Service Account.  
     - Mapping mode: Auto map, matching on "email" column.  
     - Input from "CRM Data Processing Agent".

7. **Add Quality Report Generator:**  
   - Add **Code** node named "Quality Report Generator".  
   - Implement JavaScript code to compute sync statistics, generate Slack notification payload, and send Slack message using webhook URL from config.  
   - Input from "Master Database Writer" and "Parallel CRM Fetcher".

8. **Add Google Sheets Writer for Quality Reports:**  
   - Add **Google Sheets** node named "Report Writer":  
     - Operation: Append.  
     - Sheet name: "Quality_Reports".  
     - Document ID: from Configuration Center `masterSheetId`.  
     - Authentication: Google Service Account.  
     - Input from "Quality Report Generator".

9. **Add Error Handling:**  
   - Add **Error Trigger** node named "Error Handler".  
   - Add **Code** node named "Error Processor":  
     - Implement code to log error details, send Slack notifications, and return an error log object.  
   - Connect "Error Handler" output to "Error Processor".

10. **Connect Workflow Outputs:**  
    - Connect all nodes as per the described flow:  
      - "Trigger Router" → "Configuration Center" → "Parallel CRM Fetcher" → "CRM Data Processing Agent" → "Master Database Writer" → "Quality Report Generator" → "Report Writer".  
    - Error handling operates independently and triggers on errors.

11. **Credential Setup:**  
    - Create credentials for:  
      - Google Sheets (Service Account with Editor access to relevant sheets).  
      - OpenAI API (API key with access to GPT models).  
    - Store API keys and tokens securely in environment or n8n credentials.

12. **Google Sheets Preparation:**  
    - Create a Google Sheet with two tabs:  
      - "Master_CRM_Data" with columns: source, recordId, firstName, lastName, email, companyName, phone, stage, lastModified, dataQualityScore, isDuplicate.  
      - "Quality_Reports" with columns: runId, timestamp, totalRecords, duplicatesFound, averageQuality, topIssues, etc.  
    - Share sheets with the Google service account email.

13. **MCP Server Deployment (Optional):**  
    - Clone and deploy the MCP server from GitHub: https://github.com/marlonluo2018/pandas-mcp-server  
    - Configure endpoint URL in Configuration Center.

14. **Testing:**  
    - Trigger the workflow manually via webhook or wait for scheduled run.  
    - Validate data flows, Google Sheets updates, Slack notifications, and error handling.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                           | Context or Link                                                                                                                          |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------|
| 🛠️ Multi-CRM Data Sync & AI Deduplication: Setup environment variables, Google Sheets, MCP server, and customize deduplication settings. Includes manual and scheduled triggers.                                                                                                                       | Sticky Note at Configuration Center node                                                                                               |
| 📊 Google Sheets Configuration for "Master_CRM_Data": Expected columns and service account permissions for editing.                                                                                                                                                                                 | Sticky Note near Master Database Writer node                                                                                           |
| 📊 Google Sheets Configuration for "Quality_Reports": Expected columns and permissions for quality reporting.                                                                                                                                                                                      | Sticky Note near Report Writer node                                                                                                    |
| 🔧 Required MCP Servers: GitHub repo and installation instructions for pandas-mcp-server, used for advanced pandas operations for deduplication and cleansing. Alternative: implement logic directly in n8n Code nodes without MCP server.                                                         | https://github.com/marlonluo2018/pandas-mcp-server (Sticky Note near Trigger Router)                                                    |
| ## 🚨 Error Management System: Dedicated error trigger and processor nodes to capture, log, and notify errors during workflow execution, including Slack notifications.                                                                                                                              | Sticky Note near Error Handler node                                                                                                   |

---

**Disclaimer:**  
The provided text is exclusively derived from an n8n automated workflow. It complies strictly with content policies and contains no illegal or protected information. All data handled are lawful and publicly accessible.