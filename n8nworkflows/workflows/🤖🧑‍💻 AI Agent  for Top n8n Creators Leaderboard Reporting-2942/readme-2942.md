🤖🧑‍💻 AI Agent  for Top n8n Creators Leaderboard Reporting

https://n8nworkflows.xyz/workflows/--------ai-agent--for-top-n8n-creators-leaderboard-reporting-2942


# 🤖🧑‍💻 AI Agent  for Top n8n Creators Leaderboard Reporting

### 1. Workflow Overview

This workflow automates the aggregation, analysis, and reporting of statistics related to the n8n community’s top creators and workflows. It is designed to fetch data from a GitHub repository, process and rank creators and workflows by key engagement metrics, generate detailed Markdown reports using AI language models, and distribute these reports via multiple channels such as Google Drive, email, and Telegram. The workflow runs on a schedule to provide daily updated insights.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception & Initialization**: Triggering the workflow and setting global variables.
- **1.2 Data Retrieval**: Fetching JSON data files for creators and workflows from GitHub.
- **1.3 Data Parsing & Splitting**: Parsing raw JSON data and splitting it into individual creator and workflow records.
- **1.4 Data Processing & Sorting**: Sorting creators and workflows by weekly insertion metrics and limiting to top performers.
- **1.5 Data Enrichment & Aggregation**: Merging creator and workflow data, aggregating combined data.
- **1.6 AI Report Generation**: Using GPT-4o-mini and Google Gemini LLMs to generate detailed Markdown reports and top workflows lists.
- **1.7 Report Conversion & Distribution**: Converting Markdown to HTML, saving reports locally and to Google Drive, and sending reports via Gmail and Telegram.
- **1.8 Sub-Workflow & Tool Integration**: Integration of a LangChain tool workflow for on-demand creator stats queries.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Initialization

- **Overview**: This block triggers the workflow on a schedule or via external workflow execution and sets global variables used throughout the workflow.
- **Nodes Involved**: Schedule Trigger, When Executed by Another Workflow, Global Variables
- **Node Details**:
  - **Schedule Trigger**
    - Type: Schedule Trigger
    - Role: Initiates the workflow daily at 22:00 hours.
    - Config: Runs once daily at 22:00.
    - Inputs: None
    - Outputs: Triggers the AI Agent node.
    - Edge Cases: Missed trigger if n8n instance is down.
  - **When Executed by Another Workflow**
    - Type: Execute Workflow Trigger
    - Role: Allows this workflow to be triggered by another workflow with JSON input.
    - Config: Example JSON input with a username query.
    - Inputs: External workflow trigger.
    - Outputs: Starts Global Variables node.
    - Edge Cases: Invalid or missing JSON input.
  - **Global Variables**
    - Type: Set
    - Role: Defines global variables such as GitHub raw URL path, filenames for creators and workflows JSON, and current datetime.
    - Config: 
      - `path`: GitHub raw content URL.
      - `workflows-filename`: "stats_aggregate_workflows"
      - `creators-filename`: "stats_aggregate_creators"
      - `datetime`: Current date formatted as yyyy-MM-dd.
    - Inputs: From When Executed by Another Workflow or Schedule Trigger.
    - Outputs: Triggers HTTP Request nodes for data retrieval.
    - Edge Cases: Incorrect URL or variable values.

#### 1.2 Data Retrieval

- **Overview**: Fetches creators and workflows JSON data files from the GitHub repository using HTTP requests.
- **Nodes Involved**: stats_aggregate_creators, stats_aggregate_workflows
- **Node Details**:
  - **stats_aggregate_creators**
    - Type: HTTP Request
    - Role: Downloads creators statistics JSON file dynamically using global variables.
    - Config: URL constructed as `{{ $json.path }}{{ $json['creators-filename'] }}.json`.
    - Inputs: Global Variables node output.
    - Outputs: Raw JSON data for creators.
    - Edge Cases: HTTP errors (404, 500), network timeouts.
  - **stats_aggregate_workflows**
    - Type: HTTP Request
    - Role: Downloads workflows statistics JSON file similarly.
    - Config: URL constructed as `{{ $json.path }}{{ $json['workflows-filename'] }}.json`.
    - Inputs: Global Variables node output.
    - Outputs: Raw JSON data for workflows.
    - Edge Cases: Same as creators HTTP request.

#### 1.3 Data Parsing & Splitting

- **Overview**: Parses the downloaded JSON data and splits the arrays into individual creator and workflow records for further processing.
- **Nodes Involved**: Parse Creators Data, Parse Workflow Data, Split Out Creators, Split Out Workflows
- **Node Details**:
  - **Parse Creators Data**
    - Type: Set
    - Role: Extracts the `data` array from creators JSON.
    - Config: Sets `data` field to `{{$json.data}}`.
    - Inputs: stats_aggregate_creators output.
    - Outputs: Passes to Split Out Creators.
    - Edge Cases: Missing or malformed `data` field.
  - **Parse Workflow Data**
    - Type: Set
    - Role: Extracts the `data` array from workflows JSON.
    - Config: Sets `data` field to `{{$json.data}}`.
    - Inputs: stats_aggregate_workflows output.
    - Outputs: Passes to Split Out Workflows.
    - Edge Cases: Same as creators.
  - **Split Out Creators**
    - Type: Split Out
    - Role: Splits the creators `data` array into individual items.
    - Config: Field to split out is `data`.
    - Inputs: Parse Creators Data output.
    - Outputs: Individual creator records to sorting node.
    - Edge Cases: Empty arrays.
  - **Split Out Workflows**
    - Type: Split Out
    - Role: Splits the workflows `data` array similarly.
    - Config: Field to split out is `data`.
    - Inputs: Parse Workflow Data output.
    - Outputs: Individual workflow records to sorting node.
    - Edge Cases: Empty arrays.

#### 1.4 Data Processing & Sorting

- **Overview**: Sorts creators and workflows by weekly insertion metrics and limits the results to top performers.
- **Nodes Involved**: Sort By Top Weekly Creator Inserts, Sort By Top Weekly Workflow Inserts, Take Top 10 Creators, Take Top 50 Workflows, Creators Data, Workflows Data
- **Node Details**:
  - **Sort By Top Weekly Creator Inserts**
    - Type: Sort
    - Role: Sorts creators descending by `sum_unique_weekly_inserters`.
    - Inputs: Split Out Creators output.
    - Outputs: To Take Top 10 Creators.
    - Edge Cases: Missing or zero values.
  - **Sort By Top Weekly Workflow Inserts**
    - Type: Sort
    - Role: Sorts workflows descending by `unique_weekly_inserters`.
    - Inputs: Split Out Workflows output.
    - Outputs: To Take Top 50 Workflows.
    - Edge Cases: Missing or zero values.
  - **Take Top 10 Creators**
    - Type: Limit
    - Role: Limits sorted creators to top 10.
    - Inputs: Sort By Top Weekly Creator Inserts output.
    - Outputs: To Creators Data.
    - Edge Cases: Less than 10 creators available.
  - **Take Top 50 Workflows**
    - Type: Limit
    - Role: Limits sorted workflows to top 50.
    - Inputs: Sort By Top Weekly Workflow Inserts output.
    - Outputs: To Workflows Data.
    - Edge Cases: Less than 50 workflows available.
  - **Creators Data**
    - Type: Set
    - Role: Maps and extracts relevant fields from each creator record for reporting.
    - Config: Extracts name, username, bio, and insertion metrics.
    - Inputs: Take Top 10 Creators output.
    - Outputs: To Merge Creators & Workflows.
    - Edge Cases: Missing fields.
  - **Workflows Data**
    - Type: Set
    - Role: Maps and extracts relevant fields from each workflow record, including nested user info.
    - Config: Extracts workflow name, creation date, description, user info, and visitor/inserter metrics.
    - Inputs: Take Top 50 Workflows output.
    - Outputs: To Merge Creators & Workflows.
    - Edge Cases: Missing nested fields or inconsistent data.

#### 1.5 Data Enrichment & Aggregation

- **Overview**: Merges creator and workflow data on username, aggregates combined data for final report preparation.
- **Nodes Involved**: Merge Creators & Workflows, Aggregate, Workflow Response
- **Node Details**:
  - **Merge Creators & Workflows**
    - Type: Merge
    - Role: Combines creators and workflows data streams by matching `username`.
    - Config: Mode: Combine, Join Mode: Enrich Input 1, Match Field: `username`.
    - Inputs: Creators Data and Workflows Data outputs.
    - Outputs: To Aggregate.
    - Edge Cases: Usernames not matching, missing data enrichment.
  - **Aggregate**
    - Type: Aggregate
    - Role: Aggregates all merged data items into a single data structure.
    - Config: Aggregate all item data.
    - Inputs: Merge Creators & Workflows output.
    - Outputs: To Workflow Response.
    - Edge Cases: Large data causing performance issues.
  - **Workflow Response**
    - Type: Set
    - Role: Prepares the aggregated data as a string response for downstream nodes.
    - Config: Sets `response` field to aggregated data.
    - Inputs: Aggregate output.
    - Outputs: To AI Agent node.
    - Edge Cases: Data formatting errors.

#### 1.6 AI Report Generation

- **Overview**: Uses AI language models (GPT-4o-mini and Google Gemini) and LangChain agent to generate comprehensive Markdown reports and top workflows lists.
- **Nodes Involved**: n8n Creators Stats Agent, gpt-4o-mini, Workflow Tool, Google Gemini Chat Model, Create Top 10 Workflows List
- **Node Details**:
  - **n8n Creators Stats Agent**
    - Type: LangChain Agent
    - Role: Generates a detailed Markdown report summarizing creators and workflows using provided tools.
    - Config: System message instructs to create a comprehensive Markdown report with detailed summaries, tables, community analysis, and additional insights.
    - Inputs: Triggered by Schedule Trigger or Workflow Response.
    - Outputs: Markdown report in `output` field.
    - Credentials: OpenAI API key.
    - Edge Cases: API rate limits, incomplete data, AI generation errors.
  - **gpt-4o-mini**
    - Type: LangChain OpenAI Chat Model
    - Role: Provides language model capabilities to the agent.
    - Config: Model set to "gpt-4o-mini" with temperature 0.1.
    - Credentials: OpenAI API.
    - Inputs: From agent node.
    - Outputs: To agent node.
    - Edge Cases: API errors, authentication failures.
  - **Workflow Tool**
    - Type: LangChain Tool Workflow
    - Role: Defines a callable tool within the agent to fetch detailed n8n creator stats.
    - Config: Tool named "n8n_creator_stats" linked to this workflow ID, with input schema requiring a username.
    - Inputs: Agent node calls this tool as needed.
    - Outputs: Data returned to agent.
    - Edge Cases: Tool invocation errors, missing parameters.
  - **Google Gemini Chat Model**
    - Type: LangChain Google Gemini Chat Model
    - Role: Generates a list of top 10 workflows with hyperlinks from the report.
    - Config: Model "models/gemini-2.0-flash-exp" with temperature 0.2.
    - Credentials: Google PaLM API.
    - Inputs: Receives report output from agent.
    - Outputs: To Create Top 10 Workflows List.
    - Edge Cases: API quota, network errors.
  - **Create Top 10 Workflows List**
    - Type: LangChain Chain LLM
    - Role: Creates a concise list of top 10 workflows by weekly insertions with hyperlinks.
    - Config: Prompt instructs to output only the list without preamble.
    - Inputs: Google Gemini Chat Model output.
    - Outputs: To Markdown conversion and distribution nodes.
    - Edge Cases: Prompt failures, incomplete data.

#### 1.7 Report Conversion & Distribution

- **Overview**: Converts Markdown reports to HTML, saves reports locally and to Google Drive, and sends reports via Gmail and Telegram.
- **Nodes Involved**: creator-summary, Save creator-summary.md, Google Drive, Convert Markdown to HTML, Gmail Creators & Workflows Report, Convert Top 10 Markdown to HTML, Gmail Top 10 Workflows List, Telegram Top 10 Workflows List
- **Node Details**:
  - **creator-summary**
    - Type: Convert To File
    - Role: Converts Markdown report text into a file object with filename "creators-report".
    - Inputs: n8n Creators Stats Agent output.
    - Outputs: To Save creator-summary.md.
    - Edge Cases: File conversion errors.
  - **Save creator-summary.md**
    - Type: Read/Write File
    - Role: Saves the Markdown report locally with timestamped filename.
    - Config: Writes to local path `C:\Users\joe\Downloads\` with dynamic filename including timestamp.
    - Inputs: creator-summary output (binary file).
    - Outputs: None.
    - Edge Cases: File system permission errors.
  - **Google Drive**
    - Type: Google Drive
    - Role: Uploads the Markdown report as a text file to Google Drive root folder.
    - Config: Filename includes current datetime.
    - Inputs: n8n Creators Stats Agent output.
    - Credentials: Google Drive OAuth2.
    - Outputs: None.
    - Edge Cases: Authentication failures, quota limits.
  - **Convert Markdown to HTML**
    - Type: Markdown
    - Role: Converts the full Markdown report to HTML for email formatting.
    - Inputs: n8n Creators Stats Agent output.
    - Outputs: Gmail Creators & Workflows Report.
    - Edge Cases: Conversion errors.
  - **Gmail Creators & Workflows Report**
    - Type: Gmail
    - Role: Sends the full creators and workflows report via email.
    - Config: Sends to "joe@example.com" with subject "n8n Creator Stats".
    - Credentials: Gmail OAuth2.
    - Inputs: Convert Markdown to HTML output.
    - Edge Cases: Email sending failures.
  - **Convert Top 10 Markdown to HTML**
    - Type: Markdown
    - Role: Converts the top 10 workflows list Markdown to HTML.
    - Inputs: Create Top 10 Workflows List output.
    - Outputs: Gmail Top 10 Workflows List.
    - Edge Cases: Conversion errors.
  - **Gmail Top 10 Workflows List**
    - Type: Gmail
    - Role: Sends the top 10 workflows list via email.
    - Config: Sends to "joe@example.com" with subject "n8n Top 10 Workflows".
    - Credentials: Gmail OAuth2.
    - Inputs: Convert Top 10 Markdown to HTML output.
    - Edge Cases: Email sending failures.
  - **Telegram Top 10 Workflows List**
    - Type: Telegram
    - Role: Sends the top 10 workflows list as a Telegram message.
    - Config: Uses environment variable `TELEGRAM_CHAT_ID` for chat target, HTML parse mode.
    - Credentials: Telegram API.
    - Inputs: Create Top 10 Workflows List output.
    - Edge Cases: Chat ID missing, API errors.

#### 1.8 Sub-Workflow & Tool Integration

- **Overview**: Defines a LangChain tool workflow that can be called by the AI agent to fetch detailed creator stats on demand.
- **Nodes Involved**: Workflow Tool
- **Node Details**:
  - **Workflow Tool**
    - Type: LangChain Tool Workflow
    - Role: Exposes this workflow as a callable tool named "n8n_creator_stats" with input schema requiring a username.
    - Config: Uses current workflow ID, input JSON schema example provided.
    - Inputs: Called internally by AI agent.
    - Outputs: Returns creator stats data.
    - Edge Cases: Invocation errors, missing username parameter.

---

### 3. Summary Table

| Node Name                      | Node Type                          | Functional Role                                  | Input Node(s)                      | Output Node(s)                          | Sticky Note                                                                                               |
|--------------------------------|----------------------------------|-------------------------------------------------|----------------------------------|---------------------------------------|-----------------------------------------------------------------------------------------------------------|
| Schedule Trigger               | Schedule Trigger                  | Triggers workflow daily at 22:00                | None                             | n8n Creators Stats Agent               |                                                                                                           |
| When Executed by Another Workflow | Execute Workflow Trigger          | Allows external workflow trigger with JSON input| None                             | Global Variables                      |                                                                                                           |
| Global Variables              | Set                              | Defines global variables for URLs and datetime  | When Executed by Another Workflow| stats_aggregate_creators, stats_aggregate_workflows | Sticky Note5: Global Workflow Variables                                                                    |
| stats_aggregate_creators       | HTTP Request                     | Fetches creators JSON data from GitHub           | Global Variables                 | Parse Creators Data                   | Sticky Note11: GET n8n Stats from GitHub repo                                                             |
| stats_aggregate_workflows      | HTTP Request                     | Fetches workflows JSON data from GitHub          | Global Variables                 | Parse Workflow Data                  | Sticky Note11                                                                                              |
| Parse Creators Data            | Set                              | Extracts creators data array                      | stats_aggregate_creators        | Split Out Creators                   | Sticky Note7: n8n Creators Stats                                                                           |
| Parse Workflow Data            | Set                              | Extracts workflows data array                     | stats_aggregate_workflows       | Split Out Workflows                  | Sticky Note8: n8n Workflow Stats                                                                            |
| Split Out Creators            | Split Out                        | Splits creators array into individual records    | Parse Creators Data             | Sort By Top Weekly Creator Inserts   | Sticky Note7                                                                                              |
| Split Out Workflows           | Split Out                        | Splits workflows array into individual records   | Parse Workflow Data             | Sort By Top Weekly Workflow Inserts  | Sticky Note8                                                                                              |
| Sort By Top Weekly Creator Inserts | Sort                             | Sorts creators descending by weekly inserters    | Split Out Creators              | Take Top 10 Creators                 | Sticky Note7                                                                                              |
| Sort By Top Weekly Workflow Inserts | Sort                             | Sorts workflows descending by weekly inserters   | Split Out Workflows             | Take Top 50 Workflows                | Sticky Note8                                                                                              |
| Take Top 10 Creators          | Limit                            | Limits to top 10 creators                         | Sort By Top Weekly Creator Inserts | Creators Data                      | Sticky Note7                                                                                              |
| Take Top 50 Workflows         | Limit                            | Limits to top 50 workflows                        | Sort By Top Weekly Workflow Inserts | Workflows Data                    | Sticky Note8                                                                                              |
| Creators Data                 | Set                              | Maps creator fields for reporting                 | Take Top 10 Creators            | Merge Creators & Workflows          | Sticky Note7                                                                                              |
| Workflows Data                | Set                              | Maps workflow fields for reporting                | Take Top 50 Workflows           | Merge Creators & Workflows          | Sticky Note8                                                                                              |
| Merge Creators & Workflows    | Merge                            | Combines creators and workflows by username      | Creators Data, Workflows Data   | Aggregate                          |                                                                                                           |
| Aggregate                    | Aggregate                        | Aggregates merged data into one dataset           | Merge Creators & Workflows      | Workflow Response                  |                                                                                                           |
| Workflow Response            | Set                              | Prepares aggregated data as string response       | Aggregate                      | n8n Creators Stats Agent            |                                                                                                           |
| n8n Creators Stats Agent     | LangChain Agent                 | Generates detailed Markdown report using AI       | Schedule Trigger, Workflow Response | creator-summary, Google Drive, Convert Markdown to HTML, Create Top 10 Workflows List | Sticky Note: AI Agent for n8n Creator Leaderboard Stats                                                    |
| gpt-4o-mini                  | LangChain OpenAI Chat Model     | Provides GPT-4o-mini LLM for agent                | n8n Creators Stats Agent        | n8n Creators Stats Agent            | Sticky Note9: OpenAI LLM                                                                                   |
| Workflow Tool                | LangChain Tool Workflow          | Tool for fetching detailed creator stats          | n8n Creators Stats Agent        | n8n Creators Stats Agent            | Sticky Note1: Tool Call for n8n Creators Stats                                                            |
| Google Gemini Chat Model     | LangChain Google Gemini Model    | Generates top 10 workflows list                    | n8n Creators Stats Agent        | Create Top 10 Workflows List        | Sticky Note15: Google Gemini LLM                                                                           |
| Create Top 10 Workflows List | LangChain Chain LLM              | Creates list of top 10 workflows with hyperlinks  | Google Gemini Chat Model        | Convert Top 10 Markdown to HTML, Telegram Top 10 Workflows List | Sticky Note14: Create Top 10 Workflows List                                                                |
| creator-summary              | Convert To File                 | Converts Markdown report text to file              | n8n Creators Stats Agent        | Save creator-summary.md             | Sticky Note3: Save n8n Creators & Workflows Report Locally                                                |
| Save creator-summary.md      | Read/Write File                 | Saves Markdown report locally with timestamp       | creator-summary                | None                              | Sticky Note3                                                                                              |
| Google Drive                | Google Drive                    | Uploads Markdown report to Google Drive             | n8n Creators Stats Agent        | None                              | Sticky Note9: Save n8n Creator & Workflows Report to Google Drive                                         |
| Convert Markdown to HTML     | Markdown                       | Converts full Markdown report to HTML               | n8n Creators Stats Agent        | Gmail Creators & Workflows Report   |                                                                                                           |
| Gmail Creators & Workflows Report | Gmail                          | Sends full creators and workflows report via email | Convert Markdown to HTML        | None                              | Sticky Note12: Email n8n Creators & Workflows Report                                                      |
| Convert Top 10 Markdown to HTML | Markdown                       | Converts top 10 workflows list Markdown to HTML    | Create Top 10 Workflows List    | Gmail Top 10 Workflows List         |                                                                                                           |
| Gmail Top 10 Workflows List | Gmail                          | Sends top 10 workflows list via email               | Convert Top 10 Markdown to HTML | None                              | Sticky Note16: Email Top 10 Workflows List                                                                |
| Telegram Top 10 Workflows List | Telegram                       | Sends top 10 workflows list via Telegram            | Create Top 10 Workflows List    | None                              | Sticky Note13: Telegram (Optional)                                                                         |
| Sticky Note (various)        | Sticky Note                    | Provides documentation, links, and instructions    | None                           | None                              | Multiple sticky notes provide context and links for nodes as detailed in the workflow description above. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**
   - Type: Schedule Trigger
   - Configure to run daily at 22:00 hours.
   - Connect output to the "n8n Creators Stats Agent" node.

2. **Create Execute Workflow Trigger Node**
   - Type: Execute Workflow Trigger
   - Configure with example JSON input schema for username query.
   - Connect output to "Global Variables" node.

3. **Create Global Variables Node**
   - Type: Set
   - Define variables:
     - `path`: "https://raw.githubusercontent.com/teds-tech-talks/n8n-community-leaderboard/refs/heads/main/"
     - `workflows-filename`: "stats_aggregate_workflows"
     - `creators-filename`: "stats_aggregate_creators"
     - `datetime`: Expression `{{$now.format('yyyy-MM-dd')}}`
   - Connect outputs to "stats_aggregate_creators" and "stats_aggregate_workflows".

4. **Create HTTP Request Nodes for Data Retrieval**
   - **stats_aggregate_creators**
     - Type: HTTP Request
     - URL: Expression `{{$json.path}}{{$json['creators-filename']}}.json`
     - Connect output to "Parse Creators Data".
   - **stats_aggregate_workflows**
     - Type: HTTP Request
     - URL: Expression `{{$json.path}}{{$json['workflows-filename']}}.json`
     - Connect output to "Parse Workflow Data".

5. **Create Set Nodes to Parse Data Arrays**
   - **Parse Creators Data**
     - Type: Set
     - Set field `data` to `{{$json.data}}`
     - Connect output to "Split Out Creators".
   - **Parse Workflow Data**
     - Type: Set
     - Set field `data` to `{{$json.data}}`
     - Connect output to "Split Out Workflows".

6. **Create Split Out Nodes**
   - **Split Out Creators**
     - Type: Split Out
     - Field to split: `data`
     - Connect output to "Sort By Top Weekly Creator Inserts".
   - **Split Out Workflows**
     - Type: Split Out
     - Field to split: `data`
     - Connect output to "Sort By Top Weekly Workflow Inserts".

7. **Create Sort Nodes**
   - **Sort By Top Weekly Creator Inserts**
     - Type: Sort
     - Sort descending by `sum_unique_weekly_inserters`
     - Connect output to "Take Top 10 Creators".
   - **Sort By Top Weekly Workflow Inserts**
     - Type: Sort
     - Sort descending by `unique_weekly_inserters`
     - Connect output to "Take Top 50 Workflows".

8. **Create Limit Nodes**
   - **Take Top 10 Creators**
     - Type: Limit
     - Max items: 10
     - Connect output to "Creators Data".
   - **Take Top 50 Workflows**
     - Type: Limit
     - Max items: 50
     - Connect output to "Workflows Data".

9. **Create Set Nodes to Map Data Fields**
   - **Creators Data**
     - Type: Set
     - Map fields: name, username, bio, sum_unique_weekly_inserters, sum_unique_monthly_inserters, sum_unique_inserters
     - Connect output to "Merge Creators & Workflows".
   - **Workflows Data**
     - Type: Set
     - Map fields: template_url, wf_detais.name, wf_detais.createdAt, wf_detais.description, user.name, user.username, unique_weekly_inserters, unique_monthly_inserters, unique_weekly_visitors, unique_monthly_visitors, user.avatar
     - Connect output to "Merge Creators & Workflows".

10. **Create Merge Node**
    - Type: Merge
    - Mode: Combine
    - Join Mode: Enrich Input 1
    - Match Field: `username`
    - Inputs: Creators Data and Workflows Data
    - Connect output to "Aggregate".

11. **Create Aggregate Node**
    - Type: Aggregate
    - Aggregate all item data
    - Connect output to "Workflow Response".

12. **Create Set Node for Workflow Response**
    - Type: Set
    - Set field `response` to `{{$json.data}}`
    - Connect output to "n8n Creators Stats Agent".

13. **Create LangChain Agent Node**
    - Type: LangChain Agent
    - Text: "Prepare a report about the n8n creators"
    - System Message: Detailed instructions for Markdown report generation including sections and formatting.
    - Prompt Type: Define
    - Credentials: OpenAI API key configured
    - Connect outputs to:
      - Convert To File (creator-summary)
      - Google Drive
      - Convert Markdown to HTML
      - Create Top 10 Workflows List

14. **Create Convert To File Node**
    - Type: Convert To File
    - Operation: toText
    - File Name: "creators-report"
    - Source Property: `output`
    - Connect output to "Save creator-summary.md".

15. **Create Read/Write File Node**
    - Type: Read/Write File
    - Operation: write
    - File Name: Local path with dynamic timestamp, e.g., `C:\Users\joe\Downloads\{{ $binary.data.fileName }}-{{ $now.format('yyyy-MM-dd-hh-mm-ss') }}.md`
    - Connect output: None.

16. **Create Google Drive Node**
    - Type: Google Drive
    - Operation: createFromText
    - Name: "n8n Creator Stats Report - {{ $now.format('yyyy-MM-dd:hh:mm:ss') }}"
    - Drive ID: "My Drive"
    - Folder ID: "root"
    - Content: `{{$json.output}}`
    - Credentials: Google Drive OAuth2
    - Connect output: None.

17. **Create Markdown Node to Convert Report to HTML**
    - Type: Markdown
    - Mode: markdownToHtml
    - Markdown: `{{$json.output}}`
    - Connect output to "Gmail Creators & Workflows Report".

18. **Create Gmail Node for Full Report**
    - Type: Gmail
    - Send To: "joe@example.com"
    - Subject: "n8n Creator Stats"
    - Message: `{{$json.data}}`
    - Credentials: Gmail OAuth2
    - Connect output: None.

19. **Create Google Gemini Chat Model Node**
    - Type: LangChain Google Gemini Chat Model
    - Model Name: "models/gemini-2.0-flash-exp"
    - Temperature: 0.2
    - Credentials: Google PaLM API
    - Connect output to "Create Top 10 Workflows List".

20. **Create LangChain Chain LLM Node**
    - Type: Chain LLM
    - Text: Prompt to create a list with hyperlinks of top 10 workflows by weekly insertions from report.
    - Connect output to:
      - Convert Top 10 Markdown to HTML
      - Telegram Top 10 Workflows List

21. **Create Markdown Node to Convert Top 10 List to HTML**
    - Type: Markdown
    - Mode: markdownToHtml
    - Markdown: `{{$json.text}}`
    - Connect output to "Gmail Top 10 Workflows List".

22. **Create Gmail Node for Top 10 Workflows List**
    - Type: Gmail
    - Send To: "joe@example.com"
    - Subject: "n8n Top 10 Workflows"
    - Message: `{{$json.data}}`
    - Credentials: Gmail OAuth2
    - Connect output: None.

23. **Create Telegram Node for Top 10 Workflows List**
    - Type: Telegram
    - Chat ID: `{{$env.TELEGRAM_CHAT_ID}}`
    - Text: Formatted message with top 10 workflows list and timestamp.
    - Parse Mode: HTML
    - Credentials: Telegram API
    - Connect output: None.

24. **Create LangChain Tool Workflow Node**
    - Type: LangChain Tool Workflow
    - Name: "n8n_creator_stats"
    - Workflow ID: Current workflow ID
    - Description: "Call this tool to get n8n Creator Stats."
    - JSON Schema Example: `{ "username": "n8n creator username" }`
    - Specify Input Schema: true
    - Connect input from "n8n Creators Stats Agent" ai_tool input.

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                                                                                      |
|----------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| AI Agent for n8n Creator Leaderboard Stats                                                                          | https://github.com/teds-tech-talks/n8n-community-leaderboard                                        |
| Tool Call for n8n Creators Stats                                                                                     | https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolworkflow/ |
| OpenAI LLM API Key Setup                                                                                             | https://platform.openai.com/api-keys                                                                |
| Daily n8n Leaderboard Stats & Project Homepage                                                                       | https://github.com/teds-tech-talks/n8n-community-leaderboard                                        |
| n8n Leaderboard Web Interface                                                                                         | https://teds-tech-talks.github.io/n8n-community-leaderboard/                                        |
| GitHub HTTP Request Node Documentation                                                                                | https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.httprequest/                      |
| Google Drive Node Documentation                                                                                       | https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googledrive/                       |
| Gmail Node Documentation                                                                                              | https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.gmail/                             |
| Telegram Node Documentation (Optional)                                                                                | https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.telegram/                          |
| Google Gemini LLM API Key Setup                                                                                       | https://aistudio.google.com/apikey                                                                  |
| Workflow Importance Summary: Automates community insights, recognizes contributors, saves time, and fosters collaboration | See Sticky Note17 content in workflow                                                              |

---

This documentation provides a detailed, structured reference for understanding, reproducing, and maintaining the "AI Agent for Top n8n Creators Leaderboard Reporting" workflow. It covers all nodes, their configurations, data flow, and integration points with external AI and cloud services.