🔥📈🤖 AI Agent  for n8n Creators Leaderboard - Find Popular Workflows

https://n8nworkflows.xyz/workflows/-------ai-agent--for-n8n-creators-leaderboard---find-popular-workflows-2940


# 🔥📈🤖 AI Agent  for n8n Creators Leaderboard - Find Popular Workflows

### 1. Workflow Overview

The **AI Agent for n8n Creators Leaderboard - Find Popular Workflows** workflow is designed to analyze and present detailed statistics about n8n workflow creators and their workflows within the n8n community. It automates data retrieval from GitHub-hosted JSON files, processes and merges creator and workflow statistics, filters results based on a specified username, and generates a comprehensive Markdown report using an AI agent. The report highlights popular workflows, community trends, and top contributors, and is saved locally with a timestamp for easy access.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception**: Receives user input via chat or external workflow trigger, extracting the target creator's username.
- **1.2 Global Variables Setup**: Defines base URLs and filenames for data retrieval, and captures the username parameter.
- **1.3 Data Retrieval**: Fetches creators and workflows aggregated statistics JSON files from GitHub.
- **1.4 Data Parsing and Preparation**: Parses JSON responses, splits arrays into individual items, sorts and limits datasets.
- **1.5 Data Merging and Filtering**: Merges creators and workflows data by username, then filters for the specified creator.
- **1.6 Aggregation and Report Generation**: Aggregates filtered data, invokes the AI agent to generate a detailed Markdown report.
- **1.7 Output Handling**: Converts the AI-generated report into a file and saves it locally with a timestamp.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block captures the username input to specify which creator's stats to analyze. It supports two entry points: chat message trigger and external workflow execution.

- **Nodes Involved:**  
  - When chat message received  
  - When Executed by Another Workflow

- **Node Details:**  
  - **When chat message received**  
    - Type: Chat Trigger (LangChain)  
    - Role: Listens for chat messages to start the workflow.  
    - Configuration: Uses webhook with no extra options; expects chat input containing username.  
    - Inputs: External chat messages.  
    - Outputs: Passes chat input to AI agent node.  
    - Edge Cases: Missing or malformed username in chat input may cause downstream filtering to fail.

  - **When Executed by Another Workflow**  
    - Type: Execute Workflow Trigger  
    - Role: Allows this workflow to be triggered programmatically by another workflow.  
    - Configuration: Uses a JSON example input with a "username" field.  
    - Inputs: JSON input from other workflows.  
    - Outputs: Passes input to Global Variables node.  
    - Edge Cases: Missing or invalid username in input JSON.

---

#### 1.2 Global Variables Setup

- **Overview:**  
  Sets global variables including GitHub base URL, filenames for creators and workflows stats JSON, current date, and the username to filter by.

- **Nodes Involved:**  
  - Global Variables

- **Node Details:**  
  - Type: Set  
  - Role: Defines reusable variables for URLs, filenames, and parameters.  
  - Configuration:  
    - `path`: GitHub raw content base URL.  
    - `workflows-filename`: Filename for workflows stats JSON.  
    - `creators-filename`: Filename for creators stats JSON.  
    - `chart-filename`: Filename for chart data (unused in main flow).  
    - `datetime`: Current date in `yyyy-MM-dd` format.  
    - `username`: Extracted from input JSON query (e.g., from chat or external trigger).  
  - Inputs: From trigger nodes.  
  - Outputs: Feeds into HTTP Request nodes for data retrieval.  
  - Edge Cases: Incorrect or missing username leads to empty filter results.

---

#### 1.3 Data Retrieval

- **Overview:**  
  Fetches JSON files containing aggregated statistics for creators and workflows from GitHub.

- **Nodes Involved:**  
  - stats_aggregate_creators  
  - stats_aggregate_workflows

- **Node Details:**  
  - Type: HTTP Request  
  - Role: Downloads JSON data files for creators and workflows.  
  - Configuration:  
    - URL dynamically constructed from `path` + filename variables.  
    - No authentication or special headers.  
  - Inputs: From Global Variables node.  
  - Outputs: JSON responses containing arrays of data.  
  - Edge Cases: Network errors, 404 if files missing, or malformed JSON responses.

---

#### 1.4 Data Parsing and Preparation

- **Overview:**  
  Parses the retrieved JSON data arrays, splits them into individual items, sorts by key metrics, and limits the number of items for performance.

- **Nodes Involved:**  
  - Parse Creators Data  
  - Parse Workflow Data  
  - Split Out Creators  
  - Split Out Workflows  
  - Sort By Top Weekly Creator Inserts  
  - Sort By Top Weekly Workflow Inserts  
  - Take Top 25 Creators  
  - Take Top 300 Workflows

- **Node Details:**  
  - **Parse Creators Data & Parse Workflow Data**  
    - Type: Set  
    - Role: Extracts the `data` array from HTTP response JSON for further processing.  
    - Configuration: Assigns `data` property from JSON.  
    - Inputs: From HTTP Request nodes.  
    - Outputs: Passes arrays to Split Out nodes.

  - **Split Out Creators & Split Out Workflows**  
    - Type: Split Out  
    - Role: Splits arrays into individual items for sorting and filtering.  
    - Inputs: From Parse Data nodes.  
    - Outputs: Individual creator or workflow items.

  - **Sort By Top Weekly Creator Inserts & Sort By Top Weekly Workflow Inserts**  
    - Type: Sort  
    - Role: Sorts creators by `sum_unique_weekly_inserters` and workflows by `unique_weekly_inserters` descending.  
    - Inputs: From Split Out nodes.  
    - Outputs: Sorted lists.

  - **Take Top 25 Creators & Take Top 300 Workflows**  
    - Type: Limit  
    - Role: Limits the number of items to top performers for efficiency.  
    - Inputs: From Sort nodes.  
    - Outputs: Limited datasets for merging.

- **Edge Cases:**  
  - Empty or missing data arrays cause no output.  
  - Sorting on missing fields may cause errors or unexpected order.

---

#### 1.5 Data Merging and Filtering

- **Overview:**  
  Merges creators and workflows data by matching usernames, then filters the merged data to focus on the specified creator.

- **Nodes Involved:**  
  - Creators Data  
  - Workflows Data  
  - Merge Creators & Workflows  
  - Filter By Creator Username

- **Node Details:**  
  - **Creators Data & Workflows Data**  
    - Type: Set  
    - Role: Maps and renames relevant fields from individual creator and workflow items to a consistent schema.  
    - Inputs: From Take Top nodes.  
    - Outputs: Prepared data for merging.

  - **Merge Creators & Workflows**  
    - Type: Merge  
    - Role: Combines creators and workflows data by enriching creators with matching workflow stats using the `username` field.  
    - Configuration: Mode "combine", join mode "enrichInput1", matching on `username`.  
    - Inputs: Creators Data (input1), Workflows Data (input2).  
    - Outputs: Merged enriched dataset.

  - **Filter By Creator Username**  
    - Type: Filter  
    - Role: Filters merged data to only include entries where `username` matches the input username variable.  
    - Configuration: Strict equality, case sensitive, comparing `$json.username` to the global variable username.  
    - Inputs: From Merge node.  
    - Outputs: Filtered data for aggregation.

- **Edge Cases:**  
  - No matching username results in empty output.  
  - Case sensitivity may cause missed matches if input username casing differs.

---

#### 1.6 Aggregation and Report Generation

- **Overview:**  
  Aggregates filtered data into a single item and uses an AI agent to generate a detailed Markdown report about the creator and their workflows.

- **Nodes Involved:**  
  - Aggregate  
  - gpt-4o-mini (AI Language Model)  
  - n8n Creator Stats Agent  
  - Workflow Tool  
  - Window Buffer Memory  
  - Summary Report

- **Node Details:**  
  - **Aggregate**  
    - Type: Aggregate  
    - Role: Combines all filtered items into one aggregated JSON object for AI processing.  
    - Inputs: From Filter node.  
    - Outputs: Single aggregated item.

  - **gpt-4o-mini**  
    - Type: LangChain OpenAI Chat Model  
    - Role: Provides the AI language model for generating the report.  
    - Configuration: Model set to "gpt-4o-mini", temperature 0.1 for low randomness.  
    - Credentials: OpenAI API credentials required.  
    - Inputs: Receives chat input from trigger node.  
    - Outputs: Passes AI-generated text to agent node.

  - **n8n Creator Stats Agent**  
    - Type: LangChain Agent  
    - Role: Orchestrates AI prompt and tool usage to generate a comprehensive Markdown report.  
    - Configuration:  
      - System message instructs detailed report structure including summary, tables, community analysis, and error handling.  
      - Uses the "n8n_creator_stats" tool (Workflow Tool node) to fetch detailed stats if needed.  
    - Inputs: Receives chat input and memory buffer.  
    - Outputs: Produces Markdown report text.

  - **Workflow Tool**  
    - Type: LangChain Tool Workflow  
    - Role: Allows the AI agent to call this workflow recursively to fetch detailed creator stats.  
    - Configuration: Defines input schema requiring a username.  
    - Inputs: Called by the AI agent as a tool.  
    - Outputs: Returns detailed stats for AI use.

  - **Window Buffer Memory**  
    - Type: LangChain Memory Buffer  
    - Role: Maintains chat history context for the AI agent.  
    - Inputs: Connected to AI agent.  
    - Outputs: Provides memory context.

  - **Summary Report**  
    - Type: Set  
    - Role: Captures the AI agent's Markdown output into a property named `output`.  
    - Inputs: From AI agent.  
    - Outputs: Passes text to file conversion.

- **Edge Cases:**  
  - AI API errors or timeouts.  
  - Missing or incomplete data causing incomplete reports.  
  - Recursive tool calls may fail if credentials or workflow IDs are incorrect.

---

#### 1.7 Output Handling

- **Overview:**  
  Converts the AI-generated Markdown text into a file and saves it locally with a timestamped filename.

- **Nodes Involved:**  
  - creator-summary  
  - Save creator-summary.md

- **Node Details:**  
  - **creator-summary**  
    - Type: Convert To File  
    - Role: Converts Markdown text from `output` property into a text file.  
    - Configuration: Filename prefix "creator-summary".  
    - Inputs: From Summary Report node.  
    - Outputs: Binary file data.

  - **Save creator-summary.md**  
    - Type: Read/Write File  
    - Role: Writes the Markdown file to local disk.  
    - Configuration:  
      - File path includes user directory and timestamp in filename.  
      - Append mode enabled (though typically writing new file).  
    - Inputs: Binary file data from previous node.  
    - Outputs: File saved confirmation.

- **Edge Cases:**  
  - File system permission errors.  
  - Invalid file path on different OS.  
  - Disk space issues.

---

### 3. Summary Table

| Node Name                   | Node Type                          | Functional Role                          | Input Node(s)                        | Output Node(s)                      | Sticky Note                                                                                      |
|-----------------------------|----------------------------------|----------------------------------------|------------------------------------|-----------------------------------|------------------------------------------------------------------------------------------------|
| When chat message received  | LangChain Chat Trigger           | Entry point via chat input              | -                                  | n8n Creator Stats Agent            | ## Local or Cloud LLM                                                                          |
| When Executed by Another Workflow | Execute Workflow Trigger          | Entry point via external workflow call | -                                  | Global Variables                  |                                                                                                |
| Global Variables            | Set                              | Defines URLs, filenames, username       | When Executed by Another Workflow   | stats_aggregate_creators, stats_aggregate_workflows | ## Global Workflow Variables                                                                   |
| stats_aggregate_creators    | HTTP Request                    | Fetch creators stats JSON from GitHub  | Global Variables                   | Parse Creators Data                | ## GET n8n Stats from GitHub repo                                                             |
| stats_aggregate_workflows   | HTTP Request                    | Fetch workflows stats JSON from GitHub | Global Variables                   | Parse Workflow Data               | ## GET n8n Stats from GitHub repo                                                             |
| Parse Creators Data         | Set                              | Extract creators data array             | stats_aggregate_creators           | Split Out Creators                | ## n8n Creators Stats                                                                         |
| Parse Workflow Data         | Set                              | Extract workflows data array            | stats_aggregate_workflows          | Split Out Workflows               | ## n8n Workflow Stats                                                                         |
| Split Out Creators          | Split Out                       | Split creators array into items         | Parse Creators Data                | Sort By Top Weekly Creator Inserts | ## n8n Creators Stats                                                                         |
| Split Out Workflows         | Split Out                       | Split workflows array into items        | Parse Workflow Data                | Sort By Top Weekly Workflow Inserts | ## n8n Workflow Stats                                                                         |
| Sort By Top Weekly Creator Inserts | Sort                             | Sort creators by weekly inserters       | Split Out Creators                | Take Top 25 Creators              | ## n8n Creators Stats                                                                         |
| Sort By Top Weekly Workflow Inserts | Sort                             | Sort workflows by weekly inserters      | Split Out Workflows               | Take Top 300 Workflows            | ## n8n Workflow Stats                                                                         |
| Take Top 25 Creators        | Limit                            | Limit to top 25 creators                 | Sort By Top Weekly Creator Inserts | Creators Data                    | ## n8n Creators Stats                                                                         |
| Take Top 300 Workflows      | Limit                            | Limit to top 300 workflows               | Sort By Top Weekly Workflow Inserts | Workflows Data                  | ## n8n Workflow Stats                                                                         |
| Creators Data               | Set                              | Map creator fields for merging          | Take Top 25 Creators              | Merge Creators & Workflows        | ## n8n Creators Stats                                                                         |
| Workflows Data              | Set                              | Map workflow fields for merging         | Take Top 300 Workflows            | Merge Creators & Workflows        | ## n8n Workflow Stats                                                                         |
| Merge Creators & Workflows | Merge                            | Merge creators and workflows by username | Creators Data, Workflows Data     | Filter By Creator Username        | ## Filter by n8n Creator Username                                                            |
| Filter By Creator Username  | Filter                           | Filter merged data by input username    | Merge Creators & Workflows        | Aggregate                        | ## Filter by n8n Creator Username                                                            |
| Aggregate                  | Aggregate                        | Aggregate filtered data into one item   | Filter By Creator Username        | Workflow Response                |                                                                                                |
| Workflow Response          | Set                              | Prepare response property                | Aggregate                        | (No direct output)               |                                                                                                |
| gpt-4o-mini                | LangChain OpenAI Chat Model      | AI language model for report generation | When chat message received        | n8n Creator Stats Agent          | ## Tool Call for n8n Creators Stats                                                          |
| n8n Creator Stats Agent    | LangChain Agent                  | AI agent generating Markdown report     | gpt-4o-mini, Workflow Tool, Window Buffer Memory | Summary Report, creator-summary |                                                                                                |
| Workflow Tool              | LangChain Tool Workflow          | Tool for AI agent to fetch creator stats | n8n Creator Stats Agent          | n8n Creator Stats Agent          | ## Tool Call for n8n Creators Stats                                                          |
| Window Buffer Memory       | LangChain Memory Buffer          | Maintains chat history for AI agent     | -                                | n8n Creator Stats Agent          | ## Chat History Memory                                                                       |
| Summary Report             | Set                              | Store AI-generated Markdown output      | n8n Creator Stats Agent          | creator-summary                 | ## Summary Report Response                                                                   |
| creator-summary            | Convert To File                  | Convert Markdown text to file binary    | Summary Report                   | Save creator-summary.md          | ## Save n8n Creator Report Locally                                                           |
| Save creator-summary.md    | Read/Write File                 | Save Markdown report locally             | creator-summary                  | -                               | ## Save n8n Creator Report Locally                                                           |
| Ollama Chat Model          | LangChain Ollama Chat Model      | Disabled alternative AI model            | -                                | -                               | ## Local or Cloud LLM                                                                        |
| Sticky Notes (multiple)    | Sticky Note                     | Documentation and guidance notes         | -                                | -                               | Various notes with links and explanations (see section 5)                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes**  
   - Add a **Chat Trigger** node named `When chat message received`. Configure it with a webhook to receive chat messages.  
   - Add an **Execute Workflow Trigger** node named `When Executed by Another Workflow`. Set a JSON example input with a `query.username` field.

2. **Set Global Variables**  
   - Add a **Set** node named `Global Variables`. Define variables:  
     - `path`: `"https://raw.githubusercontent.com/teds-tech-talks/n8n-community-leaderboard/refs/heads/main/"`  
     - `workflows-filename`: `"stats_aggregate_workflows"`  
     - `creators-filename`: `"stats_aggregate_creators"`  
     - `chart-filename`: `"stats_aggregate_chart"` (optional)  
     - `datetime`: Expression `{{$now.format('yyyy-MM-dd')}}`  
     - `username`: Expression `{{$json.query.username}}` (from input)

3. **Fetch Data from GitHub**  
   - Add **HTTP Request** node `stats_aggregate_creators`:  
     - URL: `={{ $json.path }}{{ $json['creators-filename'] }}.json`  
     - Method: GET  
   - Add **HTTP Request** node `stats_aggregate_workflows`:  
     - URL: `={{ $json.path }}{{ $json['workflows-filename'] }}.json`  
     - Method: GET  
   - Connect `Global Variables` node output to both HTTP Request nodes.

4. **Parse JSON Data Arrays**  
   - Add **Set** node `Parse Creators Data`: assign `data` to `={{ $json.data }}` from `stats_aggregate_creators` output.  
   - Add **Set** node `Parse Workflow Data`: assign `data` to `={{ $json.data }}` from `stats_aggregate_workflows` output.

5. **Split Arrays into Items**  
   - Add **Split Out** node `Split Out Creators` on field `data` connected to `Parse Creators Data`.  
   - Add **Split Out** node `Split Out Workflows` on field `data` connected to `Parse Workflow Data`.

6. **Sort and Limit Data**  
   - Add **Sort** node `Sort By Top Weekly Creator Inserts`: sort descending by `sum_unique_weekly_inserters` connected to `Split Out Creators`.  
   - Add **Sort** node `Sort By Top Weekly Workflow Inserts`: sort descending by `unique_weekly_inserters` connected to `Split Out Workflows`.  
   - Add **Limit** node `Take Top 25 Creators` connected to `Sort By Top Weekly Creator Inserts`.  
   - Add **Limit** node `Take Top 300 Workflows` connected to `Sort By Top Weekly Workflow Inserts`.

7. **Map Fields for Merging**  
   - Add **Set** node `Creators Data` connected to `Take Top 25 Creators`. Map fields:  
     - `name`, `username`, `bio`, `sum_unique_weekly_inserters`, `sum_unique_monthly_inserters`, `sum_unique_inserters` from JSON.  
   - Add **Set** node `Workflows Data` connected to `Take Top 300 Workflows`. Map fields:  
     - `template_url`, `wf_detais.name`, `wf_detais.createdAt`, `wf_detais.description`, `name`, `username`, `unique_weekly_inserters`, `unique_monthly_inserters`, `unique_weekly_visitors`, `unique_monthly_visitors`, `user.avatar`.

8. **Merge Creators and Workflows Data**  
   - Add **Merge** node `Merge Creators & Workflows` with mode "combine", join mode "enrichInput1", matching on `username`.  
   - Connect `Creators Data` as input 1 and `Workflows Data` as input 2.

9. **Filter by Username**  
   - Add **Filter** node `Filter By Creator Username`. Configure condition:  
     - `$json.username` equals `={{ $('Global Variables').item.json.username }}` (case sensitive).  
   - Connect output of `Merge Creators & Workflows` to this filter.

10. **Aggregate Filtered Data**  
    - Add **Aggregate** node `Aggregate` with default settings to combine all filtered items into one.  
    - Connect output of `Filter By Creator Username` to this node.

11. **Prepare Workflow Response**  
    - Add **Set** node `Workflow Response`. Assign property `response` to `={{ $json.data }}` from `Aggregate`.  
    - Connect output of `Aggregate` to this node.

12. **Configure AI Model and Agent**  
    - Add **LangChain OpenAI Chat Model** node `gpt-4o-mini`:  
      - Model: `gpt-4o-mini`  
      - Temperature: 0.1  
      - Credentials: OpenAI API credentials configured.  
    - Add **LangChain Agent** node `n8n Creator Stats Agent`:  
      - Text input: `={{ $json.chatInput }}` from chat trigger.  
      - System message: Detailed prompt instructing Markdown report generation (summary, tables, community analysis, error handling).  
      - Tools: Add `Workflow Tool` node as a tool for fetching creator stats.  
    - Add **LangChain Tool Workflow** node `Workflow Tool`:  
      - Name: `n8n_creator_stats`  
      - Workflow ID: current workflow ID  
      - Input schema: requires `username` string.  
    - Add **LangChain Memory Buffer** node `Window Buffer Memory` connected to AI agent for chat history.

13. **Connect AI Nodes**  
    - Connect `When chat message received` to `gpt-4o-mini`.  
    - Connect `gpt-4o-mini` to `n8n Creator Stats Agent`.  
    - Connect `Window Buffer Memory` to `n8n Creator Stats Agent` memory input.  
    - Connect `n8n Creator Stats Agent` output to `Summary Report` node.

14. **Prepare and Save Markdown Report**  
    - Add **Set** node `Summary Report` to assign AI output text to property `output`.  
    - Add **Convert To File** node `creator-summary`:  
      - Operation: toText  
      - Source property: `output`  
      - Filename: `creator-summary`  
    - Add **Read/Write File** node `Save creator-summary.md`:  
      - Operation: write  
      - File path: e.g., `C:\Users\joe\Downloads\{{ $binary.data.fileName }}-{{ $now.format('yyyy-MM-dd-hh-mm-ss') }}.md`  
      - Append: true (optional)  
    - Connect `Summary Report` → `creator-summary` → `Save creator-summary.md`.

15. **Connect Data Retrieval to Parsing**  
    - Connect `Global Variables` → `stats_aggregate_creators` → `Parse Creators Data` → `Split Out Creators` → `Sort By Top Weekly Creator Inserts` → `Take Top 25 Creators` → `Creators Data`.  
    - Connect `Global Variables` → `stats_aggregate_workflows` → `Parse Workflow Data` → `Split Out Workflows` → `Sort By Top Weekly Workflow Inserts` → `Take Top 300 Workflows` → `Workflows Data`.

16. **Connect Data Merging and Filtering**  
    - Connect `Creators Data` and `Workflows Data` to `Merge Creators & Workflows`.  
    - Connect `Merge Creators & Workflows` → `Filter By Creator Username` → `Aggregate` → `Workflow Response`.

17. **Connect Workflow Response to AI Agent**  
    - Connect `Workflow Response` output to AI agent input as needed (via tool or direct input).

18. **Activate Workflow**  
    - Ensure all credentials (OpenAI API) are configured.  
    - Activate the workflow.  
    - Test by sending chat message: `show me stats for username joe`.

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                                                                          |
|-------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| AI Agent for n8n Creator Leaderboard Stats                                                     | https://github.com/teds-tech-talks/n8n-community-leaderboard                                            |
| Daily n8n Leaderboard Stats and Dashboard                                                      | https://teds-tech-talks.github.io/n8n-community-leaderboard/                                           |
| Quick Start Guide for the n8n Creators Leaderboard Workflow                                    | Embedded in Sticky Note14 within the workflow                                                           |
| Benefits and Use Cases of the n8n Creators Leaderboard Workflow                                | Embedded in Sticky Note15 within the workflow                                                           |
| Workflow Overview and Data Flow Explanation                                                   | Embedded in Sticky Note12 within the workflow                                                           |
| Local or Cloud LLM Options (Ollama model disabled by default)                                  | Sticky Note9 and Sticky Note2                                                                            |
| Save n8n Creator Report Locally (optional for local installs)                                 | Sticky Note3                                                                                            |
| Summary Report Response Node Explanation                                                      | Sticky Note4                                                                                            |
| Tool Call for n8n Creators Stats (AI tool integration)                                        | Sticky Note1                                                                                            |
| Chat History Memory for AI Agent                                                             | Sticky Note13                                                                                           |

---

This documentation provides a complete, structured reference for understanding, reproducing, and maintaining the **AI Agent for n8n Creators Leaderboard** workflow, enabling advanced users and AI agents to work effectively with it.