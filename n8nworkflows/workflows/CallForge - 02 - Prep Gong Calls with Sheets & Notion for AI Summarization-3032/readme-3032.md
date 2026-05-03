CallForge - 02 - Prep Gong Calls with Sheets & Notion for AI Summarization

https://n8nworkflows.xyz/workflows/callforge---02---prep-gong-calls-with-sheets---notion-for-ai-summarization-3032


# CallForge - 02 - Prep Gong Calls with Sheets & Notion for AI Summarization

### 1. Workflow Overview

**Purpose:**  
This workflow, titled **CallForge - 02 - Prep Gong Calls with Sheets & Notion for AI Summarization**, automates the extraction, enrichment, deduplication, and preparation of Gong.io sales call data for AI-driven summarization. It integrates Gong call data with competitor and integration insights from Notion and Google Sheets, cleans and structures transcripts, and passes them to an AI call processor workflow for generating structured summaries.

**Target Use Cases:**  
- Sales teams automating call analysis  
- Revenue operations teams optimizing call data workflows  
- Businesses leveraging Gong.io for sales call insights  

**Logical Blocks:**  

- **1.1 Input Reception & Initial Data Extraction**  
  Triggering the workflow and retrieving Gong call data, competitor data from Notion, and integration data from Google Sheets.

- **1.2 Data Aggregation & Deduplication**  
  Aggregating raw data, merging multiple data sources, and removing calls already processed by checking Notion.

- **1.3 Data Enrichment**  
  Preparing enriched call data by combining Gong calls with competitor and integration information.

- **1.4 Transcript Processing & AI Preparation**  
  Cleaning and formatting call transcripts for AI consumption, reducing prompt complexity.

- **1.5 AI Call Processing Invocation**  
  Sending the prepared transcripts to an AI-powered sub-workflow for structured call summary generation.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Initial Data Extraction

**Overview:**  
This block triggers the workflow and fetches raw data from Gong, Notion, and Google Sheets to gather call details, competitor insights, and integration information.

**Nodes Involved:**  
- Execute Workflow Trigger  
- Gong  
- Get list of Competitors (Notion)  
- Get Integrations (Google Sheets)  

**Node Details:**  

- **Execute Workflow Trigger**  
  - Type: Trigger node to start the workflow on demand or via external trigger.  
  - Configuration: No parameters; serves as the entry point.  
  - Inputs: None  
  - Outputs: Gong, Get list of Competitors, Get Integrations  
  - Edge Cases: Trigger failures or misconfiguration could prevent workflow start.

- **Gong**  
  - Type: Gong API node to fetch call data.  
  - Configuration: Retrieves all calls from the last 2 days (`fromDateTime` = now minus 2 days, `toDateTime` = now). Returns all results.  
  - Credentials: Uses Gong API credentials.  
  - Inputs: Trigger node output  
  - Outputs: Raw Gong call data  
  - Edge Cases: API rate limits, authentication errors, empty call data.

- **Get list of Competitors (Notion)**  
  - Type: Notion database node to fetch competitor data.  
  - Configuration: Reads from a specific Notion database containing competitor information.  
  - Credentials: Notion API credentials.  
  - Inputs: Trigger node output  
  - Outputs: Competitor data with detailed properties.  
  - Edge Cases: API failures, rate limits, database permission issues.

- **Get Integrations (Google Sheets)**  
  - Type: Google Sheets node to fetch integration data.  
  - Configuration: Reads a specific sheet and document ID containing integration details. Executes once per workflow run.  
  - Credentials: Google Sheets OAuth2 credentials.  
  - Inputs: Trigger node output  
  - Outputs: Integration data as rows from the sheet.  
  - Edge Cases: API quota limits, OAuth token expiration, sheet access errors.

---

#### 1.2 Data Aggregation & Deduplication

**Overview:**  
Aggregates the raw data from Gong, Notion, and Google Sheets, merges them into a unified object, and removes calls already processed by querying Notion for previous call summaries.

**Nodes Involved:**  
- Call Aggregator  
- Integration Aggregator  
- Comma Separate Integrations  
- Comma separate competitors  
- Merge 3 objects into one  
- Aggregate Call Data  
- Split Out Call Data and Competitors  
- Reduce down to 1 object  
- Get Previous Phone Calls (Notion)  
- Isolate Only Call IDs  
- Only Process New Calls (Compare Datasets)  

**Node Details:**  

- **Call Aggregator**  
  - Type: Aggregate node  
  - Configuration: Aggregates all Gong call data items into a single field `calls`.  
  - Inputs: Gong node output  
  - Outputs: Aggregated call data array  
  - Edge Cases: Empty input data.

- **Integration Aggregator**  
  - Type: Aggregate node  
  - Configuration: Aggregates integration data from Google Sheets under the field `Google Sheets`.  
  - Inputs: Get Integrations output  
  - Outputs: Aggregated integration data  
  - Edge Cases: Empty or malformed sheet data.

- **Comma Separate Integrations**  
  - Type: Set node  
  - Configuration: Converts the aggregated integration array into a comma-separated string under the variable `integrations`.  
  - Inputs: Integration Aggregator output  
  - Outputs: String of integrations  
  - Edge Cases: Empty arrays result in empty strings.

- **Comma separate competitors**  
  - Type: Set node  
  - Configuration: Extracts competitor names from Notion data using JMESPath and joins them into a comma-separated string `competitors`.  
  - Inputs: Get list of Competitors output  
  - Outputs: String of competitor names  
  - Edge Cases: Missing or malformed competitor data.

- **Merge 3 objects into one**  
  - Type: Merge node  
  - Configuration: Merges three inputs: aggregated Gong calls, comma-separated integrations, and comma-separated competitors into a single object.  
  - Inputs: Call Aggregator, Comma Separate Integrations, Comma separate competitors  
  - Outputs: Unified data object  
  - Edge Cases: Missing inputs cause incomplete merges.

- **Aggregate Call Data**  
  - Type: Aggregate node  
  - Configuration: Aggregates all call data into a field `calldata`.  
  - Inputs: Merge 3 objects into one output  
  - Outputs: Aggregated call data object  
  - Edge Cases: Empty inputs.

- **Split Out Call Data and Competitors**  
  - Type: SplitOut node  
  - Configuration: Splits the aggregated `calldata` into separate fields: calls, integrations, and competitors for easier processing.  
  - Inputs: Aggregate Call Data output  
  - Outputs: Split data fields  
  - Edge Cases: Missing or malformed calldata.

- **Reduce down to 1 object**  
  - Type: Aggregate node  
  - Configuration: Aggregates previous Notion call summaries into a single object.  
  - Inputs: Output from Merge 3 objects into one (second input)  
  - Outputs: Aggregated previous calls  
  - Edge Cases: Empty previous calls data.

- **Get Previous Phone Calls (Notion)**  
  - Type: Notion databasePage node  
  - Configuration: Retrieves all previously processed call summaries from a Notion database.  
  - Credentials: Notion API credentials  
  - Inputs: Reduce down to 1 object output  
  - Outputs: List of previous call pages  
  - Edge Cases: API failures, permission issues.

- **Isolate Only Call IDs**  
  - Type: Set node  
  - Configuration: Extracts only the Gong call IDs from previous call summaries, defaulting to "none" if missing.  
  - Inputs: Get Previous Phone Calls output  
  - Outputs: List of call IDs  
  - Edge Cases: Missing or malformed IDs.

- **Only Process New Calls (Compare Datasets)**  
  - Type: Compare Datasets node  
  - Configuration: Compares current Gong calls with previous call IDs to filter out duplicates, preferring current calls.  
  - Inputs: Split Out Call Data and Competitors output (calls), Isolate Only Call IDs output  
  - Outputs: Filtered new calls only  
  - Edge Cases: Matching logic failures, empty datasets.

---

#### 1.3 Data Enrichment

**Overview:**  
Loops through each new call to enrich it with integration and competitor data, preparing for transcript processing.

**Nodes Involved:**  
- Loop Over Calls  

**Node Details:**  

- **Loop Over Calls**  
  - Type: SplitInBatches node  
  - Configuration: Processes calls one by one (batch size = 1 by default) to handle complex enrichment logic per call.  
  - Inputs: Only Process New Calls output  
  - Outputs: Individual call data for enrichment and processing  
  - Edge Cases: Large call volumes may slow processing; batch size can be adjusted.

---

#### 1.4 Transcript Processing & AI Preparation

**Overview:**  
Processes each call transcript by cleaning and formatting it to reduce AI prompt complexity, improving AI summarization accuracy.

**Nodes Involved:**  
- Transcript Processor (Execute Workflow)  
- Receive all Transcripts (NoOp)  

**Node Details:**  

- **Transcript Processor**  
  - Type: Execute Workflow node  
  - Configuration: Calls a sub-workflow (ID: 7BAQDjnHQVYO1SWG) dedicated to cleaning and preparing call transcripts for AI.  
  - Inputs: Loop Over Calls output  
  - Outputs: Processed transcripts passed back to Loop Over Calls  
  - Edge Cases: Sub-workflow failures, data format mismatches.

- **Receive all Transcripts**  
  - Type: NoOp node  
  - Configuration: Acts as a collector for all processed transcripts after looping.  
  - Inputs: Loop Over Calls output (first output)  
  - Outputs: Passes transcripts to next processing step  
  - Edge Cases: None (pass-through node).

---

#### 1.5 AI Call Processing Invocation

**Overview:**  
Sends the cleaned and enriched transcripts to an AI-powered sub-workflow that generates structured JSON summaries of the calls.

**Nodes Involved:**  
- Process All Call Transcripts (Execute Workflow)  

**Node Details:**  

- **Process All Call Transcripts**  
  - Type: Execute Workflow node  
  - Configuration: Invokes an AI call processor sub-workflow (ID: cg4Eo7yZlhWkqHCB) that uses OpenAI to generate structured call summaries.  
  - Inputs: Receive all Transcripts output  
  - Outputs: Final AI-generated call summaries  
  - Edge Cases: AI API rate limits, prompt errors, sub-workflow failures.

---

### 3. Summary Table

| Node Name                   | Node Type               | Functional Role                                  | Input Node(s)                     | Output Node(s)                         | Sticky Note                                                                                              |
|-----------------------------|-------------------------|-------------------------------------------------|----------------------------------|---------------------------------------|--------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’| Manual Trigger          | Manual start for testing                         | None                             | Gong, Get Integrations, Get list of Competitors |                                                                                                        |
| Execute Workflow Trigger     | Execute Workflow Trigger| Main workflow trigger                            | None                             | Gong, Get list of Competitors, Get Integrations |                                                                                                        |
| Gong                        | Gong API node           | Fetch Gong calls from last 2 days                | Execute Workflow Trigger         | Call Aggregator                       |                                                                                                        |
| Call Aggregator             | Aggregate               | Aggregate Gong call data                          | Gong                            | Merge 3 objects into one              | ## Get Gong Calls and Supporting Data: Besides the phone calls, integration and competitor data is extracted to supplement the AI prompt with accurate data to compare against mispronunciations. |
| Get Integrations            | Google Sheets           | Fetch integration data                            | Execute Workflow Trigger         | Integration Aggregator                |                                                                                                        |
| Integration Aggregator      | Aggregate               | Aggregate integration data                        | Get Integrations                 | Comma Separate Integrations           |                                                                                                        |
| Comma Separate Integrations | Set                     | Convert integrations array to comma-separated string | Integration Aggregator           | Merge 3 objects into one              |                                                                                                        |
| Get list of Competitors     | Notion                  | Fetch competitor data                             | Execute Workflow Trigger         | Comma separate competitors            |                                                                                                        |
| Comma separate competitors  | Set                     | Convert competitor data to comma-separated string | Get list of Competitors          | Merge 3 objects into one              |                                                                                                        |
| Merge 3 objects into one    | Merge                   | Merge calls, integrations, competitors into one | Call Aggregator, Comma Separate Integrations, Comma separate competitors | Aggregate Call Data, Reduce down to 1 object |                                                                                                        |
| Aggregate Call Data         | Aggregate               | Aggregate merged call data                        | Merge 3 objects into one         | Split Out Call Data and Competitors   |                                                                                                        |
| Split Out Call Data and Competitors | SplitOut          | Split aggregated data into calls, integrations, competitors | Aggregate Call Data              | Only Process New Calls                 | ## Loop through all calls to get enrichment: Allows for easier processing due to complexity             |
| Reduce down to 1 object     | Aggregate               | Aggregate previous Notion call summaries         | Merge 3 objects into one         | Get Previous Phone Calls              |                                                                                                        |
| Get Previous Phone Calls    | Notion                  | Fetch previously processed calls from Notion     | Reduce down to 1 object          | Isolate Only Call IDs                 | ## Remove Duplicates from Queue: Checks notion for already processed calls and removes them from the processing queue ensuring data is not duplicated. |
| Isolate Only Call IDs       | Set                     | Extract Gong call IDs from previous calls        | Get Previous Phone Calls         | Only Process New Calls                |                                                                                                        |
| Only Process New Calls      | Compare Datasets         | Filter out calls already processed                | Split Out Call Data and Competitors, Isolate Only Call IDs | Loop Over Calls                      |                                                                                                        |
| Loop Over Calls             | SplitInBatches          | Process calls one by one for enrichment and processing | Only Process New Calls           | Receive all Transcripts, Transcript Processor |                                                                                                        |
| Transcript Processor        | Execute Workflow        | Clean and prepare transcripts for AI             | Loop Over Calls                 | Loop Over Calls                      | ## Generate Clean Transcript: Allows for reduced prompting in the OpenAI node.                         |
| Receive all Transcripts     | NoOp                    | Collect all processed transcripts                 | Loop Over Calls                 | Process All Call Transcripts          |                                                                                                        |
| Process All Call Transcripts| Execute Workflow        | AI-powered call summary generation                 | Receive all Transcripts          | None                                | ## Pass Call Transcripts to Call Processor: The OpenAI node handles this process and outputs in structured JSON. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node:**  
   - Add an **Execute Workflow Trigger** node as the main entry point.

2. **Fetch Gong Calls:**  
   - Add a **Gong** node connected to the trigger.  
   - Configure to fetch calls from now minus 2 days to now.  
   - Use Gong API credentials.

3. **Fetch Competitor Data from Notion:**  
   - Add a **Notion** node connected to the trigger.  
   - Set to read from the competitor database.  
   - Use Notion API credentials.

4. **Fetch Integration Data from Google Sheets:**  
   - Add a **Google Sheets** node connected to the trigger.  
   - Configure with the specific document ID and sheet name.  
   - Use Google Sheets OAuth2 credentials.  
   - Set to execute once per workflow run.

5. **Aggregate Gong Calls:**  
   - Add an **Aggregate** node connected to Gong output.  
   - Aggregate all call items into a field named `calls`.

6. **Aggregate Integration Data:**  
   - Add an **Aggregate** node connected to Google Sheets output.  
   - Aggregate integration rows under `Google Sheets`.

7. **Comma Separate Integrations:**  
   - Add a **Set** node connected to Integration Aggregator.  
   - Assign a string field `integrations` by joining the aggregated array.

8. **Comma Separate Competitors:**  
   - Add a **Set** node connected to Notion competitor node.  
   - Use JMESPath to extract competitor names and join them into a string `competitors`.

9. **Merge Calls, Integrations, Competitors:**  
   - Add a **Merge** node with 3 inputs: Call Aggregator, Comma Separate Integrations, Comma separate competitors.  
   - Merge into one object.

10. **Aggregate Merged Call Data:**  
    - Add an **Aggregate** node connected to the Merge node output.  
    - Aggregate all into `calldata`.

11. **Split Out Call Data and Competitors:**  
    - Add a **SplitOut** node connected to Aggregate Call Data.  
    - Split `calldata[0].calls`, `calldata[1].integrations`, and `calldata[2].competitors`.

12. **Aggregate Previous Calls:**  
    - Add an **Aggregate** node connected to the Merge node (second input).  
    - Aggregate previous call summaries.

13. **Get Previous Phone Calls from Notion:**  
    - Add a **Notion** node connected to the previous aggregate.  
    - Fetch all pages from the call summaries database.  
    - Use Notion API credentials.

14. **Isolate Only Call IDs:**  
    - Add a **Set** node connected to Get Previous Phone Calls.  
    - Extract `property_gong_call_id` or default to "none" as `Call ID`.

15. **Filter Only New Calls:**  
    - Add a **Compare Datasets** node connected to Split Out Call Data and Isolate Only Call IDs.  
    - Merge by Gong call ID to exclude duplicates.

16. **Loop Over Calls:**  
    - Add a **SplitInBatches** node connected to Only Process New Calls.  
    - Configure batch size as needed (default 1).

17. **Execute Transcript Processor Sub-Workflow:**  
    - Add an **Execute Workflow** node connected to Loop Over Calls.  
    - Set workflow ID to the transcript processing workflow.  
    - This sub-workflow cleans and prepares transcripts.

18. **Receive All Transcripts:**  
    - Add a **NoOp** node connected to Loop Over Calls (first output).  
    - Collect all processed transcripts.

19. **Execute AI Call Processor Sub-Workflow:**  
    - Add an **Execute Workflow** node connected to Receive All Transcripts.  
    - Set workflow ID to the AI call processor workflow.  
    - This workflow generates structured AI summaries.

20. **Credentials Setup:**  
    - Configure and link credentials for Gong API, Google Sheets OAuth2, and Notion API.  
    - Ensure OpenAI credentials are set up in the AI call processor sub-workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                   | Context or Link                                                                                         |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| ![Callforge](https://uploads.n8n.io/templates/callforgeshadow.png) CallForge allows you to extract important information for different departments from your Sales Gong Calls. This workflow preps calls and enriches AI prompts. | Branding image and project description                                                                |
| Workflow designed to reduce errors from AI-generated summaries by correcting mispronunciations using enriched competitor and integration data.                                                                                 | Workflow description                                                                                   |
| For more information on setting up Gong API credentials, Google Sheets OAuth2, and Notion API, refer to official n8n documentation and API provider guides.                                                                   | Credential setup instructions                                                                          |
| AI processing sub-workflows (Transcript Processor and Call Processor) must be implemented separately and linked via Execute Workflow nodes.                                                                                   | Sub-workflow dependency                                                                                |
| Workflow uses JMESPath expressions for data extraction and filtering. Familiarity with JMESPath syntax is recommended for customization.                                                                                       | Technical note                                                                                         |
| Video and blog resources for CallForge are available at the original template source (not included here).                                                                                                                      | External resources (refer to original template source)                                                |

---

This document provides a detailed, structured reference for understanding, reproducing, and modifying the **CallForge - 02 - Prep Gong Calls with Sheets & Notion for AI Summarization** workflow. It covers all nodes, their roles, configurations, and integration points to ensure smooth operation and extensibility.