Generate AI-Powered End-of-Day Reports from ClickUp & GoHighLevel to Multi-Channel

https://n8nworkflows.xyz/workflows/generate-ai-powered-end-of-day-reports-from-clickup---gohighlevel-to-multi-channel-9463


# Generate AI-Powered End-of-Day Reports from ClickUp & GoHighLevel to Multi-Channel

### 1. Workflow Overview

This workflow automates the generation and distribution of comprehensive End-of-Day (EOD) reports by integrating data from ClickUp (task management) and GoHighLevel (GHL) CRM platforms. It is designed for teams seeking automated daily summaries of task completions, sales pipeline progress, team performance, and next-day action items. The workflow runs Monday to Friday at 6:00 PM server time and outputs reports simultaneously to Slack, email, and Google Drive.

The workflow consists of the following logical blocks:

- **1.1 Schedule Trigger:** Initiates the workflow on a defined schedule (6 PM Mon-Fri).
- **1.2 Data Retrieval:** Fetches current day tasks from ClickUp and won opportunities from GHL.
- **1.3 Data Merging:** Combines ClickUp and GHL data into a single dataset.
- **1.4 Data Transformation:** Converts merged data into a unified format suitable for AI processing and calculates summary metrics.
- **1.5 AI Report Generation:** Uses Azure OpenAI GPT-4o model to analyze data and produce a structured EOD report in JSON.
- **1.6 Report Formatting:** Creates three formatted report variants tailored for Slack, email, and Google Drive.
- **1.7 Routing:** Routes formatted reports to respective delivery channels based on their type.
- **1.8 Distribution:** Sends the reports to Slack channel, email recipients, and uploads to Google Drive.

---

### 2. Block-by-Block Analysis

#### 1.1 Schedule Trigger

- **Overview:** Triggers the workflow execution every weekday at 6:00 PM.
- **Nodes Involved:**  
  - Schedule EOD Report Trigger  
  - Sticky Note1 (Documentation)
- **Node Details:**  
  - **Schedule EOD Report Trigger**  
    - Type: Schedule Trigger  
    - Configuration: Cron expression `0 18 * * 1-5` for 6 PM Mon-Fri  
    - Input: None  
    - Output: Triggers downstream nodes  
    - Edge cases: Cron misconfiguration or server timezone mismatch may cause unexpected trigger times  
    - Sticky Note1 provides setup instructions and cron details  

#### 1.2 Data Retrieval

- **Overview:** Fetches today’s tasks from ClickUp and “won” opportunities from GoHighLevel CRM.
- **Nodes Involved:**  
  - Fetch ClickUp Tasks for Today  
  - Fetch GHL Won Opportunities  
  - Sticky Note2 (ClickUp fetch instructions)  
  - Sticky Note3 (GHL fetch instructions)
- **Node Details:**  
  - **Fetch ClickUp Tasks for Today**  
    - Type: ClickUp node  
    - Configuration: OAuth2 authentication, filters set to retrieve tasks due today including subtasks, requires user to replace placeholder IDs (Team, Space, Folder, List)  
    - Input: Trigger  
    - Output: List of tasks JSON  
    - Edge cases: Missing or incorrect OAuth credentials; invalid IDs; API rate limits  
  - **Fetch GHL Won Opportunities**  
    - Type: GoHighLevel node  
    - Configuration: OAuth2 authentication, filters for status “won”, limit 100 items  
    - Input: Trigger  
    - Output: List of won opportunities JSON  
    - Edge cases: Auth errors; network timeouts; data volume exceeding limit  

#### 1.3 Data Merging

- **Overview:** Merges the two data sources (ClickUp tasks and GHL opportunities) into a combined dataset for unified processing.
- **Nodes Involved:**  
  - Merge All Data Sources  
  - Sticky Note4 (merge strategy)
- **Node Details:**  
  - **Merge All Data Sources**  
    - Type: Merge node  
    - Configuration: Append mode to combine all items from both inputs  
    - Input: Outputs of ClickUp and GHL fetch nodes  
    - Output: Single combined array of records  
    - Edge cases: If either input is empty, output will be partial; node expects both inputs connected  

#### 1.4 Data Transformation

- **Overview:** Transforms merged ClickUp and GHL data into a normalized structure mimicking the GHL opportunity format, extracts monetary values, maps statuses, and aggregates summary statistics.
- **Nodes Involved:**  
  - Transform and Structure Data (Code node)  
  - Sticky Note5 (transformation explanation)
- **Node Details:**  
  - **Transform and Structure Data**  
    - Type: Code node (JavaScript)  
    - Configuration: Custom JS to:  
      - Detect if item is a ClickUp task or GHL opportunity  
      - For ClickUp tasks, map fields to GHL format including monetary value extraction from custom fields  
      - Map ClickUp status types to GHL-equivalent statuses (e.g., closed → won, cancelled → abandoned)  
      - Aggregate counts by status, total monetary value, counts by source  
      - Return a single aggregated object with total items, timestamp, opportunities array, and summary metrics  
    - Input: Merged data  
    - Output: Aggregated JSON object for AI processing  
    - Edge cases: Missing or malformed custom fields; unexpected status types; empty inputs  
    - Version requirements: n8n v2+ recommended for code node execution  

#### 1.5 AI Report Generation

- **Overview:** Uses Azure OpenAI GPT-4o model with memory buffer to analyze aggregated data and generate a structured, insightful EOD report in JSON.
- **Nodes Involved:**  
  - Simple Memory (LangChain memory buffer)  
  - Azure OpenAI Chat Model  
  - Structured Output Parser (LangChain)  
  - AI Agent: Generate EOD Report (LangChain Agent)  
  - Sticky Note6 (AI generation details)
- **Node Details:**  
  - **Simple Memory**  
    - Type: LangChain memory buffer  
    - Configuration: 7-message context window, session key `"eod_report_generator"`  
    - Input: AI Agent node output context  
    - Output: Context for GPT model  
  - **Azure OpenAI Chat Model**  
    - Type: LangChain Azure OpenAI chat model  
    - Configuration: Model `gpt-4o`  
    - Input: Text prompt with aggregated data and system instructions  
    - Output: AI-generated text response  
  - **Structured Output Parser**  
    - Type: LangChain output parser for structured JSON  
    - Configuration: JSON schema example specifying report fields (`title`, `briefSummary`, `wins`, `keyMetrics`, `blockers`, `actionItems`)  
    - Input: AI response text  
    - Output: Parsed JSON object  
  - **AI Agent: Generate EOD Report**  
    - Type: LangChain Agent  
    - Configuration:  
      - Text prompt includes detailed analysis guidelines for ClickUp tasks and GHL opportunities  
      - System message sets expert executive assistant persona and output requirements  
      - Has output parser enabled  
    - Input: Transformed and aggregated data  
    - Output: Structured EOD report JSON  
    - Edge cases: API limits, malformed AI output, parsing errors, auth failures  

#### 1.6 Report Formatting

- **Overview:** Creates three tailored report formats for Slack (Markdown with emojis), email (styled HTML), and Google Drive (plain text file).
- **Nodes Involved:**  
  - Format Reports for Distribution (Code node)  
  - Sticky Note7 (formatting explanation)
- **Node Details:**  
  - **Format Reports for Distribution**  
    - Type: Code node (JavaScript)  
    - Configuration: Extracts AI report JSON fields, constructs three distinct JSON objects each including type metadata (`slack`, `email`, `google_drive`), content formatted suitably for target channel  
    - Slack: Markdown with emojis and sections  
    - Email: Subject and HTML body with styling and sections  
    - Google Drive: Plain text with ASCII art borders and full content  
    - Input: AI Agent output JSON  
    - Output: Array of three report objects for routing  
    - Edge cases: Missing fields in AI output, encoding issues  

#### 1.7 Routing

- **Overview:** Routes each formatted report to the appropriate channel node based on the `type` attribute.
- **Nodes Involved:**  
  - Route to Slack (IF node)  
  - Route to Email (IF node)  
  - Route to Google Drive (IF node)  
  - Sticky Note8 (routing explanation)
- **Node Details:**  
  - Each IF node checks if `$json.type` equals the respective channel string (`slack`, `email`, or `google_drive`)  
  - Routes data downstream accordingly  
  - Executes all three routes in parallel  
  - Input: Array of formatted report objects  
  - Output: Filtered single report object per route  
  - Edge cases: Unexpected or missing `type` field leads to dropped message  

#### 1.8 Distribution

- **Overview:** Sends the reports via Slack message, email, and uploads a file to Google Drive.
- **Nodes Involved:**  
  - Send Slack Message (Slack node)  
  - Send Email Report (Email Send node)  
  - Convert to File (Convert to File node)  
  - Upload Report to Google Drive (Google Drive node)  
  - Sticky Note9 (Slack setup)  
  - Sticky Note10 (Email setup)  
  - Sticky Note11 (Google Drive setup)
- **Node Details:**  
  - **Send Slack Message**  
    - Type: Slack node  
    - Configuration: OAuth2 credentials, channel ID must be replaced by user, message text uses Markdown with report fields and emojis  
    - Input: Routed Slack report  
    - Output: Slack message confirmation  
    - Edge cases: Invalid channel ID, auth failure, rate limits  
  - **Send Email Report**  
    - Type: Email Send node  
    - Configuration: SMTP credentials, from and to email placeholders must be replaced, uses professional HTML template with gradient header and responsive design  
    - Input: Routed email report  
    - Output: Email delivery confirmation  
    - Edge cases: SMTP auth failure, invalid email addresses, HTML rendering issues  
  - **Convert to File**  
    - Type: Convert To File node  
    - Configuration: Converts JSON report to JSON file format for upload  
    - Input: Routed Google Drive report  
    - Output: File binary data  
    - Edge cases: Conversion errors, data size limits  
  - **Upload Report to Google Drive**  
    - Type: Google Drive node  
    - Configuration: OAuth2 credentials, uploads to root or specified folder (recommended EOD Reports folder), filename formatted as `EOD_Report_YYYY-MM-DD.txt`  
    - Input: File from Convert to File node  
    - Output: Upload confirmation  
    - Edge cases: Auth failure, folder permission issues, naming conflicts  

---

### 3. Summary Table

| Node Name                  | Node Type                    | Functional Role                          | Input Node(s)                      | Output Node(s)                   | Sticky Note                                                                                               |
|----------------------------|------------------------------|----------------------------------------|----------------------------------|---------------------------------|----------------------------------------------------------------------------------------------------------|
| Schedule EOD Report Trigger | Schedule Trigger             | Initiates workflow on schedule         | None                             | Fetch GHL Won Opportunities, Fetch ClickUp Tasks for Today | Sticky Note1: Cron setup instructions for 6 PM Mon-Fri                                                     |
| Fetch ClickUp Tasks for Today | ClickUp Node               | Retrieves today's ClickUp tasks        | Schedule EOD Report Trigger      | Merge All Data Sources           | Sticky Note2: Setup & security notes for ClickUp task fetching                                           |
| Fetch GHL Won Opportunities | GoHighLevel Node             | Retrieves won GHL opportunities        | Schedule EOD Report Trigger      | Merge All Data Sources           | Sticky Note3: Setup & filter notes for GHL retrieval                                                    |
| Merge All Data Sources      | Merge Node                   | Combines ClickUp and GHL data          | Fetch ClickUp Tasks, Fetch GHL Won Opportunities | Transform and Structure Data    | Sticky Note4: Merge strategy explanation                                                                |
| Transform and Structure Data| Code Node                   | Normalizes data and aggregates summary | Merge All Data Sources           | AI Agent: Generate EOD Report    | Sticky Note5: Data transformation and aggregation details                                               |
| Simple Memory              | LangChain Memory Buffer      | Maintains conversation context for AI | AI Agent: Generate EOD Report    | Azure OpenAI Chat Model          | Sticky Note6: AI generation setup and model details                                                     |
| Azure OpenAI Chat Model    | LangChain Azure OpenAI       | Processes text prompt with GPT-4o      | Simple Memory                   | Structured Output Parser         | Sticky Note6                                                                                              |
| Structured Output Parser   | LangChain Output Parser      | Parses AI text output into JSON        | Azure OpenAI Chat Model          | AI Agent: Generate EOD Report    | Sticky Note6                                                                                              |
| AI Agent: Generate EOD Report | LangChain Agent           | Generates structured EOD report JSON   | Transform and Structure Data, Structured Output Parser, Azure OpenAI Chat Model, Simple Memory | Format Reports for Distribution | Sticky Note6                                                                                              |
| Format Reports for Distribution | Code Node              | Creates Slack, Email, and Drive report formats | AI Agent: Generate EOD Report    | Route to Slack, Route to Email, Route to Google Drive | Sticky Note7: Report formatting explanation                                                             |
| Route to Slack             | IF Node                      | Routes Slack-type reports               | Format Reports for Distribution | Send Slack Message              | Sticky Note8: Routing logic for Slack message                                                           |
| Route to Email             | IF Node                      | Routes email-type reports               | Format Reports for Distribution | Send Email Report               | Sticky Note8: Routing logic for Email                                                                   |
| Route to Google Drive      | IF Node                      | Routes Google Drive-type reports        | Format Reports for Distribution | Convert to File                 | Sticky Note8: Routing logic for Google Drive                                                            |
| Send Slack Message         | Slack Node                   | Sends report message to Slack channel  | Route to Slack                  | None                           | Sticky Note9: Slack channel ID setup instructions                                                       |
| Send Email Report          | Email Send Node              | Sends report email                      | Route to Email                  | None                           | Sticky Note10: SMTP credentials and email placeholders setup                                            |
| Convert to File            | Convert To File Node         | Converts report JSON to file format     | Route to Google Drive           | Upload Report to Google Drive   | Sticky Note11: Conversion for Google Drive upload                                                       |
| Upload Report to Google Drive | Google Drive Node          | Uploads report file to Google Drive    | Convert to File                 | None                           | Sticky Note11: Google Drive folder and filename setup                                                   |
| Sticky Note                | Sticky Note                  | Documentation                          | None                             | None                           | Sticky Note: General workflow overview and purpose                                                     |
| Sticky Note1               | Sticky Note                  | Schedule trigger instructions           | None                             | None                           | Cron expression and manual execution instructions                                                      |
| Sticky Note2               | Sticky Note                  | ClickUp fetch instructions              | None                             | None                           | ClickUp OAuth2 credential and ID setup notes                                                          |
| Sticky Note3               | Sticky Note                  | GHL fetch instructions                  | None                             | None                           | GoHighLevel OAuth2 credential and filter setup notes                                                  |
| Sticky Note4               | Sticky Note                  | Merge node explanation                   | None                             | None                           | Merge strategy overview                                                                                |
| Sticky Note5               | Sticky Note                  | Data transformation explanation          | None                             | None                           | Transform and aggregation logic description                                                           |
| Sticky Note6               | Sticky Note                  | AI generation details                    | None                             | None                           | Azure OpenAI GPT-4o model and LangChain agent setup                                                   |
| Sticky Note7               | Sticky Note                  | Report formatting explanation            | None                             | None                           | Formatting for Slack, email, and Google Drive                                                         |
| Sticky Note8               | Sticky Note                  | Routing logic explanation                | None                             | None                           | IF nodes routing reports based on type                                                                |
| Sticky Note9               | Sticky Note                  | Slack message sending setup              | None                             | None                           | Slack OAuth2 credential and channel ID replacement instructions                                      |
| Sticky Note10              | Sticky Note                  | Email sending setup                      | None                             | None                           | SMTP credentials and email placeholders setup                                                        |
| Sticky Note11              | Sticky Note                  | Google Drive upload setup                 | None                             | None                           | Google Drive OAuth2 credential and folder/filename setup                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node:**  
   - Add "Schedule Trigger" node  
   - Set Cron expression to `0 18 * * 1-5` (6 PM Mon-Fri)  
   - No credentials needed  

2. **Add ClickUp Node to Fetch Tasks:**  
   - Add "ClickUp" node  
   - Operation: `getAll` tasks  
   - Apply filters: subtasks included, due date equals today  
   - Provide OAuth2 credentials for ClickUp  
   - Replace placeholders with your Team ID, Space ID, Folder ID, List ID  

3. **Add GoHighLevel Node to Fetch Opportunities:**  
   - Add "GoHighLevel" node  
   - Resource: `opportunity`, Operation: `getAll`  
   - Filter for status = `won`  
   - Limit result to 100  
   - Provide OAuth2 credentials for GoHighLevel  

4. **Add Merge Node to Combine Data:**  
   - Add "Merge" node  
   - Mode: Append  
   - Connect ClickUp node output to input 1  
   - Connect GHL node output to input 2  

5. **Add Code Node for Data Transformation:**  
   - Add "Code" node  
   - Paste the provided JavaScript code that:  
     - Detects ClickUp vs GHL records  
     - Maps ClickUp tasks to GHL opportunity format  
     - Extracts monetary values  
     - Aggregates summary statistics  
   - Input: Connect from Merge node output  

6. **Add LangChain Simple Memory Node:**  
   - Add "Memory Buffer" node  
   - Set Session Key to `"eod_report_generator"`  
   - Context window length: 7 messages  

7. **Add Azure OpenAI Chat Model Node:**  
   - Add LangChain "Azure OpenAI Chat Model" node  
   - Set model to `gpt-4o`  
   - Provide Azure OpenAI API credentials  

8. **Add Structured Output Parser Node:**  
   - Add LangChain "Output Parser Structured" node  
   - Provide JSON schema example with keys: `title`, `briefSummary`, `wins`, `keyMetrics`, `blockers`, `actionItems`  

9. **Add LangChain Agent Node:**  
   - Add "Agent" node  
   - Configure prompt with detailed instructions analyzing the aggregated data and generating concise EOD report JSON  
   - Link inputs:  
     - Text input from Code node (transformed data)  
     - AI language model from Azure OpenAI Chat Model node  
     - AI output parser from Structured Output Parser node  
     - AI memory from Simple Memory node  

10. **Add Code Node to Format Reports:**  
    - Add "Code" node  
    - Paste JavaScript that:  
      - Extracts structured report fields  
      - Creates three objects with type `slack`, `email`, and `google_drive`  
      - Formats content appropriately for each channel  

11. **Add Three IF Nodes for Routing:**  
    - Add three "IF" nodes named `Route to Slack`, `Route to Email`, `Route to Google Drive`  
    - Configure each to test if `$json.type` equals respective string (`slack`, `email`, `google_drive`)  

12. **Add Slack Node:**  
    - Add "Slack" node  
    - Use OAuth2 credentials for Slack  
    - Replace channel ID with your target Slack channel  
    - Use message template with Markdown and emojis as in the original node  

13. **Add Email Send Node:**  
    - Add "Email Send" node  
    - Provide SMTP credentials (Gmail, Outlook, etc.)  
    - Set `fromEmail` and `toEmail` addresses appropriately  
    - Use the provided HTML template for the email body  
    - Subject line bound to report title  

14. **Add Convert To File Node:**  
    - Add "Convert To File" node  
    - Operation: Convert to JSON file  
    - Input from `Route to Google Drive` node  

15. **Add Google Drive Node:**  
    - Add "Google Drive" node  
    - Provide OAuth2 credentials  
    - Choose destination folder (recommend creating "EOD Reports" folder)  
    - Filename format: `EOD_Report_YYYY-MM-DD.txt`  
    - Input binary file from Convert To File node  

16. **Connect Nodes:**  
    - Schedule Trigger → Fetch ClickUp Tasks & Fetch GHL Opportunities (parallel)  
    - Both fetch nodes → Merge node  
    - Merge → Transform and Structure Data (Code)  
    - Transform → AI Agent node (with Azure OpenAI Chat Model, Memory, Parser connected appropriately)  
    - AI Agent → Format Reports for Distribution (Code)  
    - Format Reports → IF routing nodes (Slack, Email, Google Drive)  
    - Each IF node → respective channel nodes for sending/uploading  

17. **Test Workflow:**  
    - Manually execute or wait for scheduled run  
    - Verify data retrieval, AI generation, and delivery to Slack, Email, and Google Drive  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                           | Context or Link                                                                                  |
|--------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Workflow runs Monday through Friday at 6:00 PM server time. Adjust the cron expression in Schedule Trigger if needed.                                 | Sticky Note1                                                                                    |
| Replace all placeholder IDs (ClickUp Team ID, Space ID, Folder ID, List ID, Slack Channel ID, email addresses) before running in production.          | Sticky Notes2, 3, 9, 10                                                                        |
| Azure OpenAI GPT-4o is recommended for best analysis performance. Requires Azure OpenAI API credentials.                                              | Sticky Note6                                                                                   |
| Slack message uses markdown with emojis for better readability.                                                                                        | Sticky Note9                                                                                   |
| Email uses a professional, mobile-responsive HTML template with gradient header and color-coded sections for clarity.                                  | Sticky Note10                                                                                  |
| Google Drive uploads a plain text file with ASCII art borders including all report sections and a timestamp.                                           | Sticky Note11                                                                                  |
| Ensure OAuth2 credentials for all third-party services are properly set up and tested before deployment.                                               | Setup Requirements in Sticky Note                                                               |
| API rate limits or authentication failures are possible failure points in data fetching and AI nodes; include error handling as needed.              | General best practice                                                                           |
| The workflow uses LangChain nodes for AI integration, requiring n8n v2+ and LangChain plugin compatibility.                                           | Version-specific requirement                                                                   |
| For further customization, adjust data transformation code or AI prompt in the Agent node to fit specific business logic or report requirements.      | Customization tip                                                                              |

---

**Disclaimer:** The provided text and workflow originate exclusively from an automated n8n workflow. The content complies strictly with current content policies and contains no illegal or offensive material. All data processed is legal and public.