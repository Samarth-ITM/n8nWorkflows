Web Research Assistant: Automated Search & Scraping with Gemini AI and Spreadsheet Reports

https://n8nworkflows.xyz/workflows/web-research-assistant--automated-search---scraping-with-gemini-ai-and-spreadsheet-reports-8218


# Web Research Assistant: Automated Search & Scraping with Gemini AI and Spreadsheet Reports

### 1. Workflow Overview

This workflow, titled **"Web Research Assistant: Automated Search & Scraping with Gemini AI and Spreadsheet Reports"**, automates the process of gathering web data from multiple sources, analyzing it using AI, and generating structured reports in Google Sheets. It is targeted at users needing automated, multi-source web research with data extraction, aggregation, and presentation — such as market researchers, recruiters, or analysts.

The workflow is logically divided into these blocks:

- **1.1 Input Reception:** Captures user requests via chat messages.
- **1.2 AI Orchestration and Memory:** Uses Google Gemini AI to analyze requests, select tools, and maintain conversational context.
- **1.3 Data Collection Tools:** Interfaces with multiple scraping/search tools (Firecrawl, Brave, Apify) to retrieve raw data.
- **1.4 Report Generation:** Transforms collected data into structured Google Sheets reports with formatting.
- **1.5 Data Enrichment and Finalization:** Sends batch updates to Google Sheets API and prepares output data for downstream use or delivery.
- **1.6 Error Handling & Prerequisites:** Embedded considerations for retries, validation, and required API credentials.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This initial block listens for incoming chat messages that trigger the workflow.

**Nodes Involved:**  
- When chat message received  
- Sticky Note (Entry Point)

**Node Details:**

- **When chat message received**  
  - Type: Langchain Chat Trigger node  
  - Role: Listens for chat messages as input to start the workflow  
  - Configuration: Default webhook with no additional options; triggers on any received chat message  
  - Input: External chat message via webhook  
  - Output: Passes message to AI orchestrator  
  - Edge Cases: Failure if webhook is inaccessible or message format is invalid

- **Sticky Note (Entry Point)**  
  - Type: Sticky Note  
  - Role: Documentation for users indicating this is the entry point for natural language requests

---

#### 1.2 AI Orchestration and Memory

**Overview:**  
Processes the user input with Google Gemini AI, manages conversational memory, and orchestrates tool selection for research.

**Nodes Involved:**  
- Simple Memory  
- Google Gemini Chat Model  
- Gemini Research Orchestrator  
- Sticky Notes (Orchestration, Prerequisites, Data Flow, Error Handling)

**Node Details:**

- **Simple Memory**  
  - Type: Langchain Memory Buffer Window  
  - Role: Maintains contextual memory window of last 30 interactions to provide context to the AI agent  
  - Configuration: Context window length set to 30 messages  
  - Input: Chat messages from trigger node  
  - Output: Memory context passed to Gemini Research Orchestrator  
  - Edge Cases: Memory overflow or loss if context exceeds window length

- **Google Gemini Chat Model**  
  - Type: Langchain Google Gemini LLM Chat Model  
  - Role: Provides large language model capabilities via Google Gemini (PaLM API)  
  - Configuration: Uses Google Palm API credentials stored in n8n  
  - Input: Context from Simple Memory  
  - Output: Language model responses to orchestrator  
  - Edge Cases: API rate limits, invalid credentials, network timeouts

- **Gemini Research Orchestrator**  
  - Type: Langchain Agent  
  - Role: The core AI agent that interprets user queries, decides which scraping/search tools to use, and orchestrates calls to those tools  
  - Configuration: System message defines instructions to use Firecrawl, Brave, and Apify tools, emphasizing use of tool nodes before execution nodes  
  - Input: Chat message and memory context from Simple Memory and Google Gemini Chat Model  
  - Output: Structured commands for tool selection and execution  
  - Edge Cases: Misinterpretation of instructions, failure to select proper tools, malformed tool commands

- **Sticky Notes (Orchestration, Prerequisites, Data Flow, Error Handling)**  
  - Provide human-readable summaries of orchestration logic, required credentials (Google Gemini API key, MCP API keys, Google Sheets OAuth), data flow steps, and error handling strategies (timeouts, API limits, missing data validation).

---

#### 1.3 Data Collection Tools

**Overview:**  
Implements multi-source web scraping and search using three external MCP tools: Firecrawl, Brave, and Apify. Each tool has two nodes: one to list available tools and one to execute the selected tool with parameters.

**Nodes Involved:**  
- Firecrawl list  
- Firecrawl execute  
- Brave list  
- Brave execute  
- Apify list  
- Apify execute  
- Sticky Note (Data Collection Tools)

**Node Details:**

- **Firecrawl list & Firecrawl execute**  
  - Type: MCP Client Tool nodes  
  - Role: Firecrawl list retrieves available scraping tools; Firecrawl execute runs the chosen scraping task  
  - Configuration: Firecrawl credentials via MCP API  
  - Input: Tool selection commands from Gemini Research Orchestrator  
  - Output: Scraped data forwarded to report generation  
  - Edge Cases: MCP API timeouts, tool unavailability, invalid parameters

- **Brave list & Brave execute**  
  - Same structure and role as Firecrawl nodes, using Brave MCP API credentials

- **Apify list & Apify execute**  
  - Similar to above but note that Apify execute uses Firecrawl credentials (likely a configuration oversight or intentional depending on API access rights)  
  - Edge Case: Credential mismatch may cause failures; validate credentials carefully

- **Sticky Note (Data Collection Tools)**  
  - Describes automatic tool selection by agent and lists tools involved

---

#### 1.4 Report Generation

**Overview:**  
Organizes and formats the aggregated data into a comprehensive Google Spreadsheet report with multiple sheets for summary, categories, jobs, and statistics.

**Nodes Involved:**  
- Create Research Report  
- Format Research Data (Code)  
- Populate Research Report  
- Sticky Note (Report Generation)

**Node Details:**

- **Create Research Report**  
  - Type: Google Sheets node  
  - Role: Creates a new Google Spreadsheet to hold the research results  
  - Configuration: Uses Google Sheets OAuth credentials, creates a spreadsheet titled “scrapped”  
  - Input: Output from Gemini Research Orchestrator and data collection nodes  
  - Output: Spreadsheet metadata including spreadsheetId  
  - Edge Cases: API auth failures, quota limits

- **Format Research Data**  
  - Type: Code (JavaScript) node  
  - Role: Parses AI agent output markdown, extracts title, categories, jobs, and builds batch update requests and value ranges for the spreadsheet  
  - Configuration: Contains extensive custom parsing and Google Sheets batch request formatting logic, including cell formatting, sheet creation, and data insertion  
  - Input: AI output text, spreadsheetId from previous node  
  - Output: JSON with spreadsheetId, batch requests, valueRanges, and data stats for further processing  
  - Edge Cases: Parsing errors, malformed AI output, Google Sheets API constraints, empty data handling

- **Populate Research Report**  
  - Type: HTTP Request node  
  - Role: Executes Google Sheets batchUpdate API to apply formatting and insert data into the spreadsheet  
  - Configuration: Uses Google Sheets OAuth credentials, sends batchUpdate requests from Format Research Data  
  - Input: Requests and value ranges JSON  
  - Output: Confirmation of update success  
  - Edge Cases: API rate limiting, partial failures, authorization errors

- **Sticky Note (Report Generation)**  
  - Summarizes the report generation steps and nodes involved

---

#### 1.5 Data Enrichment and Finalization

**Overview:**  
Finalizes the report by sending any additional enrichment requests and sets up output data with error handling and metadata.

**Nodes Involved:**  
- Data Enrichment Request  
- Finalize Output Data  
- Sticky Note (Post-Processing)

**Node Details:**

- **Data Enrichment Request**  
  - Type: HTTP Request node  
  - Role: Sends a batchUpdate request to Google Sheets API to enrich or finalize the spreadsheet contents (likely overlaps with Populate Research Report node; may handle additional enrichment steps)  
  - Configuration: Uses spreadsheetId from previous nodes, sends JSON body with valueRanges  
  - Input: Output from Populate Research Report  
  - Output: Confirmation and status of enrichment  
  - Edge Cases: API failures or invalid enrichment data

- **Finalize Output Data**  
  - Type: Set node  
  - Role: Constructs a final JSON output including error messages, spreadsheet ID, and timestamp for downstream consumption or logging  
  - Configuration: Uses expressions to extract error info and spreadsheet metadata from previous nodes  
  - Input: Output and error data from prior nodes  
  - Output: Structured JSON summary of execution result  
  - Edge Cases: Missing data or errors causing incomplete output

- **Sticky Note (Post-Processing)**  
  - Describes this block’s role in data enrichment and final output preparation

---

#### 1.6 Error Handling & Prerequisites

**Overview:**  
Not explicit nodes but documented as sticky notes covering prerequisites and error handling strategies.

**Notes:**

- **Prerequisites:**  
  - Google Gemini API Key  
  - MCP API keys for Firecrawl, Brave, Apify  
  - Google Sheets OAuth credentials  
  - Self-hosted n8n instance recommended

- **Error Handling:**  
  - Automatic retry on MCP timeout  
  - Fallback modes for API rate limits  
  - Validation on missing or incomplete data

---

### 3. Summary Table

| Node Name                | Node Type                               | Functional Role                           | Input Node(s)                    | Output Node(s)               | Sticky Note                                                                                       |
|--------------------------|---------------------------------------|-----------------------------------------|---------------------------------|------------------------------|-------------------------------------------------------------------------------------------------|
| When chat message received| Langchain Chat Trigger                 | Entry point, receives chat message      | External webhook                | Gemini Research Orchestrator  | ## Entry point: The user sends a request in natural language.                                   |
| Simple Memory            | Langchain Memory Buffer Window          | Keeps chat context memory                | When chat message received       | Gemini Research Orchestrator  | ## Orchestration: When chat message received, Gemini Research Orchestrator, Simple Memory      |
| Google Gemini Chat Model | Langchain Google Gemini LLM Chat Model  | Provides AI language understanding       | Simple Memory                   | Gemini Research Orchestrator  | ## Orchestration                                                                               |
| Gemini Research Orchestrator | Langchain Agent                    | AI agent orchestrating tool usage         | When chat message received, Simple Memory, Google Gemini Chat Model | Create Research Report, Firecrawl list, Brave list, Apify list, Firecrawl execute, Brave execute, Apify execute | ## Orchestration                                                                           |
| Firecrawl list           | MCP Client Tool                        | Lists available Firecrawl tools          | Gemini Research Orchestrator     | Firecrawl execute             | ## Data Collection Tools: automatic tool selection including Firecrawl, Brave, Apify             |
| Firecrawl execute        | MCP Client Tool                        | Executes Firecrawl scraping tool          | Gemini Research Orchestrator     | Create Research Report        | ## Data Collection Tools                                                                       |
| Brave list               | MCP Client Tool                        | Lists available Brave tools               | Gemini Research Orchestrator     | Brave execute                | ## Data Collection Tools                                                                       |
| Brave execute            | MCP Client Tool                        | Executes Brave scraping tool              | Gemini Research Orchestrator     | Create Research Report        | ## Data Collection Tools                                                                       |
| Apify list               | MCP Client Tool                        | Lists available Apify tools               | Gemini Research Orchestrator     | Apify execute                | ## Data Collection Tools                                                                       |
| Apify execute            | MCP Client Tool                        | Executes Apify scraping tool              | Gemini Research Orchestrator     | Create Research Report        | ## Data Collection Tools                                                                       |
| Create Research Report   | Google Sheets node                    | Creates spreadsheet for report            | Gemini Research Orchestrator     | Format Research Data          | ## Report Generation: Create Research Report, Format Research Data, Populate Research Report   |
| Format Research Data     | Code (JavaScript)                     | Parses AI output, prepares batch requests | Create Research Report           | Populate Research Report      | ## Report Generation                                                                           |
| Populate Research Report | HTTP Request (Google Sheets API)     | Applies batch update requests to Sheets  | Format Research Data             | Data Enrichment Request       | ## Report Generation                                                                           |
| Data Enrichment Request  | HTTP Request (Google Sheets API)     | Additional batch update and enrichment    | Populate Research Report         | Finalize Output Data          | ## Post-Processing: Data Enrichment and Finalization                                           |
| Finalize Output Data     | Set node                             | Prepares final output JSON                 | Data Enrichment Request          | None                        | ## Post-Processing                                                                            |
| Sticky Note              | Sticky Note                         | Documentation/Comments                     | None                           | None                        | ## Entry point: The user sends a request in natural language.                                 |
| Sticky Note1             | Sticky Note                         | Prerequisites summary                      | None                           | None                        | ## ⚙️ PREREQUISITES: Google Gemini API Key, MCP API keys, Google Sheets credentials, n8n instance|
| Sticky Note2             | Sticky Note                         | Data flow summary                          | None                           | None                        | ## 📊 DATA FLOW: Request → AI analysis → Tool selection → Collection → Aggregation → Delivery  |
| Sticky Note3             | Sticky Note                         | Error handling summary                     | None                           | None                        | ## ⚠️ ERROR HANDLING: MCP timeout retry, API limits fallback, Missing data validation          |
| Sticky Note4             | Sticky Note                         | Orchestration summary                      | None                           | None                        | ## Orchestration: When chat message received, Gemini Research Orchestrator, Simple Memory      |
| Sticky Note5             | Sticky Note                         | Data Collection tools summary              | None                           | None                        | ## Data Collection Tools: Firecrawl, Brave, Apify                                            |
| Sticky Note6             | Sticky Note                         | Report Generation summary                   | None                           | None                        | ## Report Generation: Create Research Report, Format Research Data, Populate Research Report   |
| Sticky Note7             | Sticky Note                         | Post-processing summary                      | None                           | None                        | ## Post-Processing: Data enrichment and finalization for delivery                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node**  
   - Add **When chat message received** (Langchain Chat Trigger) node.  
   - Configure webhook to listen for incoming chat messages. No special parameters needed.

2. **Set Up Memory Context**  
   - Add **Simple Memory** (Langchain Memory Buffer Window) node.  
   - Set context window length to 30 messages.  
   - Connect output of chat trigger node to this memory node.

3. **Configure AI Language Model**  
   - Add **Google Gemini Chat Model** (Langchain LM Chat Google Gemini) node.  
   - Attach your Google Palm API credentials.  
   - Connect output of Simple Memory node to this node.

4. **Add AI Orchestrator Agent**  
   - Add **Gemini Research Orchestrator** (Langchain Agent) node.  
   - Set system message to instruct it to use Firecrawl, Brave, and Apify tools, with clear instructions to use list nodes before execute nodes.  
   - Connect outputs from When chat message received, Simple Memory, and Google Gemini Chat Model to this node using the appropriate inputs (main, ai_memory, ai_languageModel).  
   - Configure no additional credentials here; it delegates to tool nodes.

5. **Add Data Collection Tool Nodes**  
   - For each tool (Firecrawl, Brave, Apify), add two MCP Client Tool nodes: one for listing tools, one for executing tools.  
     - Firecrawl list & Firecrawl execute: Assign MCP Firecrawl credentials.  
     - Brave list & Brave execute: Assign MCP Brave credentials.  
     - Apify list & Apify execute: Assign MCP Apify credentials (note in original, Apify execute uses Firecrawl credentials; review accordingly).  
   - For execute nodes, configure parameters:  
     - Tool Name: Expression `{{$fromAI("tool", "the selected tool to use")}}`  
     - Operation: `executeTool`  
     - Tool Parameters: Expression `{{$fromAI('Tool_Parameters', '', 'json')}}`  
   - Connect outputs of Gemini Research Orchestrator (ai_tool) to all list and execute nodes accordingly.

6. **Create Google Sheets Report**  
   - Add **Create Research Report** (Google Sheets node).  
   - Assign Google Sheets OAuth2 credentials.  
   - Configure to create a new spreadsheet titled "scrapped".  
   - Connect main output of Gemini Research Orchestrator to this node.

7. **Add Code Node to Format Data**  
   - Add **Format Research Data** (Code node).  
   - Paste the provided JavaScript code that parses AI output, generates batch update requests for Google Sheets, formats cells, and prepares value ranges.  
   - Connect output of Create Research Report to this node.

8. **Populate the Spreadsheet**  
   - Add **Populate Research Report** (HTTP Request node).  
   - Configure for POST to `https://sheets.googleapis.com/v4/spreadsheets/{{ $json.spreadsheetId }}:batchUpdate`  
   - Use Google Sheets OAuth2 API credentials.  
   - Body: Send JSON with requests from Format Research Data node (`{{ JSON.stringify($json.requests) }}`).  
   - Connect output of Format Research Data to this node.

9. **Data Enrichment Request**  
   - Add **Data Enrichment Request** (HTTP Request node).  
   - Configure POST to `https://sheets.googleapis.com/v4/spreadsheets/{{ $json.spreadsheetId }}/values:batchUpdate`  
   - Send JSON body with valueInputOption and data from Format Research Data node.  
   - Assign Google Sheets OAuth2 credentials.  
   - Connect output of Populate Research Report to this node.

10. **Finalize Output Data**  
    - Add **Finalize Output Data** (Set node).  
    - Configure JSON output with error message, error code, spreadsheet ID, and timestamp extracted from previous nodes.  
    - Connect output of Data Enrichment Request to this node.

11. **Add Sticky Notes for Documentation**  
    - Add sticky notes at relevant places to describe:  
      - Entry point and input reception  
      - Orchestration and prerequisites  
      - Data collection tools explanation  
      - Report generation steps  
      - Post-processing and error handling summaries  

12. **Configure Connections**  
    - Connect nodes according to the flow described:  
      - When chat message received → Gemini Research Orchestrator  
      - Simple Memory and Google Gemini Chat Model → Gemini Research Orchestrator (ai_memory and ai_languageModel inputs)  
      - Gemini Research Orchestrator → Firecrawl list, Firecrawl execute, Brave list, Brave execute, Apify list, Apify execute (ai_tool inputs)  
      - Gemini Research Orchestrator → Create Research Report (main output)  
      - Create Research Report → Format Research Data → Populate Research Report → Data Enrichment Request → Finalize Output Data  

13. **Verify Credentials and API Access**  
    - Ensure all API keys and OAuth credentials are valid and have required scopes:  
      - Google Gemini API key (Google PaLM API)  
      - MCP API keys for Firecrawl, Brave, Apify  
      - Google Sheets OAuth2 with spreadsheet read/write permissions  

14. **Test Workflow**  
    - Trigger the workflow by sending a chat message to the webhook.  
    - Monitor execution for errors such as API limits, parsing errors, or credential failures.  
    - Adjust context window or system prompt as needed for improved AI orchestration.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                                                  |
|----------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Workflow requires a self-hosted n8n instance for full integration with MCP and Google APIs         | Sticky Note1 (Prerequisites)                                                                    |
| Data flow follows a 4-step process: Request → AI analysis → Tool selection → Collection → Aggregation → Delivery | Sticky Note2 (Data Flow)                                                                        |
| Error handling strategies include automatic retry on MCP timeouts, fallback modes for API limits, and data validation | Sticky Note3 (Error Handling)                                                                   |
| The AI agent is instructed explicitly to use the tool nodes (list nodes) before executing the scraping tasks | Gemini Research Orchestrator system message                                                     |
| Google Sheets API batchUpdate requests are used extensively for efficient bulk updates and formatting | Format Research Data (Code node)                                                                |
| The workflow is suitable for automating complex web research tasks involving diverse data sources and structured reporting | Workflow Overview                                                                                |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected material. All data handled is legal and publicly accessible.