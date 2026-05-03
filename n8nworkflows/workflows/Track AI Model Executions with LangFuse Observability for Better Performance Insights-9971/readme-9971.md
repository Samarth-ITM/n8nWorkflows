Track AI Model Executions with LangFuse Observability for Better Performance Insights

https://n8nworkflows.xyz/workflows/track-ai-model-executions-with-langfuse-observability-for-better-performance-insights-9971


# Track AI Model Executions with LangFuse Observability for Better Performance Insights

### 1. Workflow Overview

This workflow is designed to track and report AI model execution details from n8n workflow runs into Langfuse, a platform for observability and performance insights of AI models. It captures execution metadata, token usage, latencies, prompts, and completions, then sends structured event data to Langfuse’s ingestion API.

**Target Use Cases:**  
- Monitoring performance metrics and token consumption of AI models used in n8n workflows.  
- Aggregating execution data by AI model and workflow node to gain insights and detect anomalies.  
- Feeding detailed execution traces into Langfuse for enhanced observability and analysis.

**Logical Blocks:**  
- **1.1 Input Reception & Deduplication:** Receives execution ID from a triggering workflow and prevents duplicate processing.  
- **1.2 Execution Data Retrieval & Delay:** Waits for full execution data availability and fetches complete execution details from n8n.  
- **1.3 Execution Data Structuring:** Parses and normalizes raw execution data to extract relevant AI model runs, token usage, latencies, and prompt/completion texts.  
- **1.4 Iteration Over Runs:** Splits structured data into individual runs for processing.  
- **1.5 Langfuse JSON Preparation:** Formats each run into Langfuse-compatible trace and generation event objects.  
- **1.6 Data Ingestion:** Sends the prepared batch of events to the Langfuse ingestion API with authentication and includes a wait for rate limiting or sequencing.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Deduplication

**Overview:**  
Receives an `execution_id` parameter from another workflow and ensures each execution is processed only once.

**Nodes Involved:**  
- When Executed by Another Workflow  
- Remove Duplicates

**Node Details:**  
- **When Executed by Another Workflow**  
  - Type: Execute Workflow Trigger  
  - Role: Entry point triggered by another workflow, accepting `execution_id` as input.  
  - Configuration: Expects input parameter `execution_id`.  
  - Input Connections: None (trigger node).  
  - Output Connections: Connects to Remove Duplicates.  
  - Edge Cases: Missing or malformed `execution_id` would cause downstream failure or empty results.

- **Remove Duplicates**  
  - Type: Remove Duplicates  
  - Role: Filters out repeated executions by `execution_id` to avoid reprocessing same execution.  
  - Configuration: Defaults, operates on the incoming data stream keyed by execution_id.  
  - Input Connections: From "When Executed by Another Workflow".  
  - Output Connections: Connects to "Wait to get an execution data".  
  - Edge Cases: If duplicate detection fails, could lead to redundant API calls and duplicated events.

---

#### 1.2 Execution Data Retrieval & Delay

**Overview:**  
Waits a fixed amount of time to allow n8n execution metrics to finalize, then retrieves full execution details including token usage and node run data.

**Nodes Involved:**  
- Wait to get an execution data  
- n8n (Execution Get)

**Node Details:**  
- **Wait to get an execution data**  
  - Type: Wait  
  - Role: Delays processing by 80 seconds to ensure execution data completeness.  
  - Configuration: Fixed wait of 80 seconds.  
  - Input Connections: From "Remove Duplicates".  
  - Output Connections: Connects to "n8n".  
  - Edge Cases: Insufficient wait time may cause incomplete data retrieval.

- **n8n**  
  - Type: n8n API Node (Execution Get)  
  - Role: Fetches detailed execution metadata and run data from n8n API by `execution_id`.  
  - Configuration: Uses n8n API credentials; operation “get” on executions; executionId from previous node’s `execution_id`.  
  - Input Connections: From "Wait to get an execution data".  
  - Output Connections: Connects to "Code: structure execution data".  
  - Failure Types: Auth errors, invalid execution IDs, API timeouts.

---

#### 1.3 Execution Data Structuring

**Overview:**  
Transforms the raw execution data into a normalized, enriched format that aggregates per-model token usage, latencies, and extracts prompt/completion texts for each AI run.

**Nodes Involved:**  
- Code: structure execution data

**Node Details:**  
- **Code: structure execution data**  
  - Type: Code (JavaScript)  
  - Role: Parses the execution JSON, extracts AI model runs (focusing on OpenAI Chat Model nodes), links runs to prompt nodes, computes latency and token aggregates, and prepares summary objects.  
  - Configuration: Custom extensive JS code with safe getters, flattening functions, heuristic matching of prompt nodes, and aggregation logic.  
  - Key Variables: `perModelRuns` (flat run records), `perModel` (aggregated model stats), `totals` (global tokens), `perNodeLatency`, and `summary` metadata.  
  - Input Connections: From "n8n" node output.  
  - Output Connections: Connects to "Split Out".  
  - Edge Cases: Unexpected data structure in n8n execution JSON, missing token info, or absent prompt nodes could cause incomplete or inaccurate summaries.  
  - Notes: Heuristic prompt linking may fail if naming conventions differ.

---

#### 1.4 Iteration Over Runs

**Overview:**  
Splits the aggregated execution data per run for individual processing and iterates through each run.

**Nodes Involved:**  
- Split Out  
- Loop Over Items

**Node Details:**  
- **Split Out**  
  - Type: Split Out  
  - Role: Extracts the `perModelRuns` array from the structured data to split into individual items for processing.  
  - Configuration: Splits on field `perModelRuns`, includes selected fields such as workflow info and token totals.  
  - Input Connections: From "Code: structure execution data".  
  - Output Connections: Connects to "Loop Over Items".  
  - Edge Cases: Empty or missing `perModelRuns` results in no iterations.

- **Loop Over Items**  
  - Type: Split In Batches  
  - Role: Processes each run item individually, supporting batch processing if needed.  
  - Configuration: Defaults, no batch size specified (processes all).  
  - Input Connections: From "Split Out".  
  - Output Connections: On branch 2 (second output), connects to "Code: prepare JSON for LF".  
  - Edge Cases: Large data volumes may require batching or cause timeouts.

---

#### 1.5 Langfuse JSON Preparation

**Overview:**  
Prepares the JSON payload for the Langfuse ingestion API by creating trace and generation event objects per run, including stable IDs, timestamps, token usage, input/output texts, and metadata.

**Nodes Involved:**  
- Code: prepare JSON for LF  
- Wait1

**Node Details:**  
- **Code: prepare JSON for LF**  
  - Type: Code (JavaScript)  
  - Role: Builds Langfuse event batch array with two event types: `trace-create` and `generation-create`. Sets stable IDs using execution and workflow IDs, computes event timestamps based on latency, constructs prompt links, and includes usage metadata.  
  - Key Variables: `batch` array, `traceId`, `sessionId`, `promptLink`, `safeLatency`, etc.  
  - Input Connections: From "Loop Over Items" (second output).  
  - Output Connections: Connects to "Wait1".  
  - Edge Cases: Missing or malformed run data may cause invalid batch payloads.

- **Wait1**  
  - Type: Wait  
  - Role: Optional short delay (3 seconds) before sending to Langfuse API, possibly for rate limiting or sequencing.  
  - Configuration: Fixed wait of 3 seconds.  
  - Input Connections: From "Code: prepare JSON for LF".  
  - Output Connections: Connects to "HTTP Request".  
  - Edge Cases: None significant.

---

#### 1.6 Data Ingestion

**Overview:**  
Sends the prepared batch of trace and generation events to Langfuse’s public ingestion API using Basic Authentication.

**Nodes Involved:**  
- HTTP Request

**Node Details:**  
- **HTTP Request**  
  - Type: HTTP Request  
  - Role: Posts JSON batch to Langfuse ingestion endpoint.  
  - Configuration:  
    - URL: `https://cloud.langfuse.com/api/public/ingestion`  
    - Method: POST  
    - Body: JSON, sends `batch` array from previous code node.  
    - Authentication: Generic HTTP Basic Auth, credentials expected to be Langfuse public and secret keys.  
    - Headers: Content-Type: application/json.  
  - Input Connections: From "Wait1".  
  - Output Connections: Loops back to "Loop Over Items" (first output), allowing batch iteration.  
  - Edge Cases: Network errors, auth failures, API rate limits; requires valid Langfuse API keys.

---

### 3. Summary Table

| Node Name                    | Node Type                     | Functional Role                              | Input Node(s)                 | Output Node(s)                | Sticky Note                                                                                                                                                |
|------------------------------|-------------------------------|----------------------------------------------|------------------------------|------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------|
| When Executed by Another Workflow | Execute Workflow Trigger      | Entry trigger receiving execution_id input   | None                         | Remove Duplicates             | This template is to demonstrate how to trace the observations per execution ID in Langfuse via ingestion API. See details on endpoint and auth in sticky note. |
| Remove Duplicates             | Remove Duplicates              | Filters duplicate execution IDs               | When Executed by Another Workflow | Wait to get an execution data |                                                                                                                                                            |
| Wait to get an execution data| Wait                          | Delays to ensure execution data availability | Remove Duplicates             | n8n                          |                                                                                                                                                            |
| n8n                          | n8n API Node (Execution Get)  | Retrieves full execution data from n8n API   | Wait to get an execution data | Code: structure execution data |                                                                                                                                                            |
| Code: structure execution data| Code (JavaScript)             | Parses and aggregates execution data          | n8n                          | Split Out                    | Code node parses execution data, extracts OpenAI runs, aggregates tokens/latency, links prompts, and prepares summaries.                                   |
| Split Out                    | Split Out                     | Splits aggregated execution data into runs   | Code: structure execution data | Loop Over Items              |                                                                                                                                                            |
| Loop Over Items              | Split In Batches              | Iterates over individual model runs           | Split Out                    | Code: prepare JSON for LF (branch 2), HTTP Request (branch 1) |                                                                                                                                                            |
| Code: prepare JSON for LF    | Code (JavaScript)             | Builds Langfuse trace and generation events   | Loop Over Items              | Wait1                        |                                                                                                                                                            |
| Wait1                        | Wait                          | Short delay before sending to Langfuse API   | Code: prepare JSON for LF    | HTTP Request                 |                                                                                                                                                            |
| HTTP Request                 | HTTP Request                  | Sends batch event data to Langfuse ingestion API | Wait1                        | Loop Over Items (branch 1)   | Endpoint: https://cloud.langfuse.com/api/public/ingestion, Auth: Basic Auth with Langfuse API keys.                                                        |
| Sticky Note                 | Sticky Note                   | Documentation notes                           | None                         | None                         | About this template, usage instructions, Langfuse endpoint and auth details, customization tips.                                                          |
| Sticky Note1                | Sticky Note                   | Documentation on the Code: structure execution data node | None                         | None                         | Details on the function and logic of the “Code: structure execution data” node.                                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node:**  
   - Add **Execute Workflow Trigger** node, name it “When Executed by Another Workflow”.  
   - Configure to accept an input parameter named `execution_id`.

2. **Add Remove Duplicates:**  
   - Add **Remove Duplicates** node, connect input from “When Executed by Another Workflow”.  
   - Use default settings to filter based on incoming execution IDs.

3. **Add Wait Node (Wait to get execution data):**  
   - Add **Wait** node, name it “Wait to get an execution data”.  
   - Set fixed wait to 80 seconds.  
   - Connect input from “Remove Duplicates”.

4. **Add n8n API Execution Get Node:**  
   - Add **n8n** node, operation “Execution Get”.  
   - Configure credential with valid n8n API credentials.  
   - Set execution ID parameter to `{{$node["When Executed by Another Workflow"].json.execution_id}}`.  
   - Connect input from “Wait to get an execution data”.

5. **Add Code Node (structure execution data):**  
   - Add **Code** node, name it “Code: structure execution data”.  
   - Paste the provided JavaScript code that parses execution data, extracts AI runs, aggregates tokens and latencies, links prompts, and prepares a structured summary.  
   - Connect input from “n8n” node.

6. **Add Split Out Node:**  
   - Add **Split Out** node, configure it to split the `perModelRuns` field from the code output.  
   - Include fields: workflowId, workflowName, executionId, startedAt, stoppedAt, executionMs, executionSec, totals_promptTokens, totals_completionTokens, totals_totalTokens.  
   - Connect input from “Code: structure execution data”.

7. **Add Loop Over Items Node:**  
   - Add **Split In Batches** node, name it “Loop Over Items”.  
   - Connect input from “Split Out” node.

8. **Add Code Node (prepare JSON for Langfuse):**  
   - Add **Code** node, name it “Code: prepare JSON for LF”.  
   - Paste the provided JS code that formats trace-create and generation-create events for Langfuse ingestion.  
   - Connect to the second output (branch 2) of “Loop Over Items”.

9. **Add Wait Node (Wait1):**  
   - Add **Wait** node, name it “Wait1”.  
   - Set fixed wait to 3 seconds.  
   - Connect input from “Code: prepare JSON for LF”.

10. **Add HTTP Request Node:**  
    - Add **HTTP Request** node.  
    - Configure with URL: `https://cloud.langfuse.com/api/public/ingestion`.  
    - Method: POST.  
    - Set Body Content to JSON, sending the `batch` array from previous node.  
    - Add header `Content-Type: application/json`.  
    - Set Authentication to Generic Credential Type with HTTP Basic Auth: username = Langfuse public key, password = Langfuse secret key.  
    - Connect input from “Wait1”.  
    - Connect output back to the first output (branch 1) of “Loop Over Items” to continue batch processing.

11. **Set Credential:**  
    - Create or configure a credential in n8n for the Langfuse API using HTTP Basic Auth with your Langfuse public and secret keys.

12. **Final Configuration:**  
    - Ensure the workflow is triggered by the external workflow passing the `execution_id`.  
    - Test with actual execution IDs from your n8n runs to verify data flow and Langfuse ingestion.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                  | Context or Link                                                                                      |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| This template demonstrates tracing AI model executions in Langfuse via ingestion API with stable IDs and session grouping.                                                                                                   | Sticky Note at workflow start                                                                        |
| Langfuse ingestion endpoint: https://cloud.langfuse.com/api/public/ingestion                                                                                                                                                  | Sticky Note content                                                                                  |
| Authentication is HTTP Basic Auth using Langfuse public and secret API keys.                                                                                                                                                   | Sticky Note content                                                                                  |
| Wait time of 80 seconds is used to ensure execution data including token usage is fully available before fetching.                                                                                                            | Sticky Note content                                                                                  |
| The “Code: structure execution data” node implements heuristic logic to link AI runs to prompt nodes and aggregates usage metrics per model and per node for detailed insights.                                              | Sticky Note1 content                                                                                 |
| Customization suggestions: add span-create events with parentObservationId for nested traces, add score-create events for feedback, modify sessionId strategy as needed.                                                     | Sticky Note content                                                                                  |
| Langfuse project prompt link base URL is hardcoded and should be updated to your own project if reused.                                                                                                                      | In "Code: prepare JSON for LF" node code                                                            |
| For more info on Langfuse and observability concepts, visit Langfuse official docs (not included here).                                                                                                                      | External resource (not linked explicitly in workflow)                                              |

---

**Disclaimer:**  
The provided content is extracted exclusively from an automated n8n workflow. It respects all applicable content policies and contains no illegal or protected elements. All processed data is lawful and public.