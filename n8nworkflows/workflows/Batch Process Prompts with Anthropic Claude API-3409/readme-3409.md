Batch Process Prompts with Anthropic Claude API

https://n8nworkflows.xyz/workflows/batch-process-prompts-with-anthropic-claude-api-3409


# Batch Process Prompts with Anthropic Claude API

### 1. Workflow Overview

This n8n workflow template is designed to efficiently batch process multiple text prompts using Anthropic's Claude language models via their Batch API endpoint (`/v1/messages/batches`). It enables sending many prompts in a single API call, polling for completion, retrieving results, and outputting each prompt's response as a separate item. The workflow is split into two main logical blocks:

- **1.1 Core Batch Processing Workflow:**  
  This block handles the core logic of receiving batch requests, submitting them to Anthropic’s Batch API, polling for completion, fetching results, parsing them, and splitting output items. It is designed to be called by other workflows via the "Execute Workflow" node.

- **1.2 Example Usage Branch:**  
  This separate branch demonstrates how to prepare input data (the batch request array and Anthropic API version) and trigger the core batch processing workflow. It includes examples of building requests from simple query strings and from n8n’s Langchain Chat Memory nodes, illustrating practical usage scenarios.

---

### 2. Block-by-Block Analysis

#### 2.1 Core Batch Processing Workflow

**Overview:**  
This block receives batch prompt requests, submits them to Anthropic’s batch endpoint, polls until processing is complete, retrieves the results file, parses the JSON Lines response, and outputs each individual prompt result as a separate item.

**Nodes Involved:**  
- When Executed by Another Workflow  
- Submit batch  
- If ended processing  
- Batch Status Poll Interval (Wait)  
- Check batch status  
- Get results  
- Parse response (Code)  
- Split Out Parsed Results  

**Node Details:**

- **When Executed by Another Workflow**  
  - Type: Execute Workflow Trigger  
  - Role: Entry point triggered by another workflow, expects inputs: `anthropic-version` (string) and `requests` (array of prompt objects)  
  - Inputs: External workflow call  
  - Outputs: Passes input data to "Submit batch"  
  - Edge cases: Missing or malformed inputs may cause errors downstream  

- **Submit batch**  
  - Type: HTTP Request  
  - Role: Sends POST request to `https://api.anthropic.com/v1/messages/batches` with the batch of prompts in JSON body  
  - Configuration:  
    - Method: POST  
    - Body: JSON, constructed dynamically from input `requests` array  
    - Headers: Includes `anthropic-version` header from input  
    - Authentication: Uses Anthropic API credential  
  - Inputs: From "When Executed by Another Workflow"  
  - Outputs: Batch job creation response (includes batch `id`)  
  - Edge cases: API auth errors, invalid request format, network timeouts  

- **If ended processing**  
  - Type: If node  
  - Role: Checks if batch job's `processing_status` is `"ended"`  
  - Inputs: From "Submit batch" (initially) and "Check batch status" (loop)  
  - Outputs:  
    - True branch: batch ended → proceed to "Get results"  
    - False branch: batch still processing → proceed to "Batch Status Poll Interval"  
  - Edge cases: Unexpected status values, missing `processing_status` field  

- **Batch Status Poll Interval**  
  - Type: Wait  
  - Role: Waits 10 seconds before next status check to avoid excessive polling  
  - Inputs: From "If ended processing" false branch  
  - Outputs: After wait, triggers "Check batch status"  
  - Edge cases: Long wait times may delay processing; too short may hit rate limits  

- **Check batch status**  
  - Type: HTTP Request  
  - Role: Sends GET request to `https://api.anthropic.com/v1/messages/batches/{batch_id}` to check current batch status  
  - Configuration:  
    - URL dynamically built from batch `id`  
    - Headers: `anthropic-version` from initial input  
    - Authentication: Anthropic API credential  
  - Inputs: From "Batch Status Poll Interval"  
  - Outputs: Batch status JSON with `processing_status` and possibly `results_url`  
  - Edge cases: API errors, batch not found, network issues  

- **Get results**  
  - Type: HTTP Request  
  - Role: Fetches the batch results file from `results_url` once processing is complete  
  - Configuration:  
    - URL: `results_url` from batch status response  
    - Headers: `anthropic-version`  
    - Authentication: Anthropic API credential  
  - Inputs: From "If ended processing" true branch  
  - Outputs: Raw JSON Lines response text in `data` field  
  - Edge cases: URL expired, network errors, malformed response  

- **Parse response**  
  - Type: Code (JavaScript)  
  - Role: Parses JSON Lines response text into an array of JSON objects under `parsed` field  
  - Logic:  
    - Splits response string by newline  
    - Filters out empty lines  
    - Parses each line as JSON  
  - Inputs: From "Get results"  
  - Outputs: Items with `parsed` array field  
  - Edge cases: Malformed JSON lines, empty response  

- **Split Out Parsed Results**  
  - Type: Split Out  
  - Role: Splits the `parsed` array into individual output items for downstream processing  
  - Inputs: From "Parse response"  
  - Outputs: One item per prompt result  
  - Edge cases: Empty array results in no output items  

---

#### 2.2 Example Usage Branch

**Overview:**  
This branch demonstrates how to prepare the input data (`anthropic-version` and `requests` array) and trigger the core batch processing workflow. It shows two methods for building prompt requests: from a simple query string and from Langchain Chat Memory nodes.

**Nodes Involved:**  
- Run example (Manual Trigger)  
- One query example (Set)  
- Truncate Chat Memory (Langchain MemoryManager)  
- Fill Chat Memory with example data (Langchain MemoryManager)  
- Load Chat Memory Data (Langchain MemoryManager)  
- Append execution data for chat memory example (Set)  
- Build batch 'request' object from Chat Memory and execution data (Code)  
- Append execution data for single query example (Set)  
- Build batch 'request' object for single query (Code)  
- Delete original properties (Set)  
- Join two example requests into array (Merge)  
- Construct 'requests' array (Aggregate)  
- Set desired 'anthropic-version' (Set)  
- Execute Workflow 'Process Multiple Prompts in Parallel with Anthropic Claude Batch API' (Execute Workflow)  
- Filter First Prompt Results (Filter)  
- Filter Second Prompt Results (Filter)  
- First Prompt Result (Execution Data)  
- Second Prompt Result (Execution Data)  

**Node Details:**

- **Run example**  
  - Type: Manual Trigger  
  - Role: Starts the example usage branch manually for testing/demo  

- **One query example**  
  - Type: Set  
  - Role: Defines a simple query string: `"Hey Claude, tell me a short fun fact about bees!"`  

- **Truncate Chat Memory**  
  - Type: Langchain MemoryManager (Delete mode)  
  - Role: Clears previous chat memory to ensure clean state for example  

- **Fill Chat Memory with example data**  
  - Type: Langchain MemoryManager (Insert mode)  
  - Role: Inserts example chat messages simulating a conversation with Claude  

- **Load Chat Memory Data**  
  - Type: Langchain MemoryManager (Load mode)  
  - Role: Loads the stored chat memory data for processing  

- **Append execution data for chat memory example**  
  - Type: Set  
  - Role: Adds metadata fields (`custom_id`, `model`, `max_tokens`) to chat memory data for batch request construction  

- **Build batch 'request' object from Chat Memory and execution data**  
  - Type: Code (JavaScript)  
  - Role: Transforms chat memory messages and metadata into Anthropic batch request objects  
  - Logic:  
    - Converts chat messages into Anthropic message format with roles (`user`, `assistant`)  
    - Extracts optional system message  
    - Constructs `params` object per prompt with model, max_tokens, and messages  
    - Assigns `custom_id` for tracking  

- **Append execution data for single query example**  
  - Type: Set  
  - Role: Adds metadata fields (`custom_id`, `model`, `max_tokens`) to the simple query string for batch request  

- **Build batch 'request' object for single query**  
  - Type: Code (JavaScript)  
  - Role: Converts the simple query string and metadata into Anthropic batch request object format  

- **Delete original properties**  
  - Type: Set  
  - Role: Keeps only `custom_id` and `params` fields to match Anthropic batch request schema  

- **Join two example requests into array**  
  - Type: Merge  
  - Role: Combines chat memory and single query batch request objects into one stream  

- **Construct 'requests' array**  
  - Type: Aggregate  
  - Role: Aggregates all input items into a single array under `requests` field for batch submission  

- **Set desired 'anthropic-version'**  
  - Type: Set  
  - Role: Adds the required `anthropic-version` header value (`2023-06-01`) to the payload  

- **Execute Workflow 'Process Multiple Prompts in Parallel with Anthropic Claude Batch API'**  
  - Type: Execute Workflow  
  - Role: Calls the core batch processing workflow with prepared inputs  
  - Configuration: Waits for sub-workflow to finish and returns results  

- **Filter First Prompt Results** and **Filter Second Prompt Results**  
  - Type: Filter  
  - Role: Separates results by `custom_id` to demonstrate handling individual prompt outputs  

- **First Prompt Result** and **Second Prompt Result**  
  - Type: Execution Data  
  - Role: Extracts and saves the assistant response text from each filtered result for demonstration  

---

### 3. Summary Table

| Node Name                                            | Node Type                         | Functional Role                                      | Input Node(s)                                    | Output Node(s)                                   | Sticky Note                                                                                                       |
|-----------------------------------------------------|----------------------------------|-----------------------------------------------------|-------------------------------------------------|-------------------------------------------------|-------------------------------------------------------------------------------------------------------------------|
| When Executed by Another Workflow                    | Execute Workflow Trigger          | Entry point for core batch processing                | External workflow call                           | Submit batch                                    |                                                                                                                   |
| Submit batch                                        | HTTP Request                     | Sends batch request to Anthropic API                 | When Executed by Another Workflow                | If ended processing                             | ## Submit batch request to Anthropic                                                                              |
| If ended processing                                 | If                              | Checks if batch processing is complete               | Submit batch, Check batch status                  | Get results (true), Batch Status Poll Interval (false) | ## Loop until processing status is 'ended'                                                                         |
| Batch Status Poll Interval                          | Wait                            | Waits 10 seconds between status checks                | If ended processing (false branch)                | Check batch status                              |                                                                                                                   |
| Check batch status                                 | HTTP Request                     | Polls Anthropic API for batch job status              | Batch Status Poll Interval                        | If ended processing                             |                                                                                                                   |
| Get results                                        | HTTP Request                     | Retrieves batch results file                           | If ended processing (true branch)                 | Parse response                                  | ### Retrieve Message Batch Results [User guide](https://docs.anthropic.com/en/docs/build-with-claude/batch-processing) |
| Parse response                                     | Code                            | Parses JSON Lines response into JSON array            | Get results                                      | Split Out Parsed Results                        | JSONL separated by newlines                                                                                        |
| Split Out Parsed Results                           | Split Out                       | Outputs each parsed result as separate item           | Parse response                                   |                                                 |                                                                                                                   |
| Run example                                        | Manual Trigger                  | Starts example usage branch                            |                                                 | One query example, Truncate Chat Memory         |                                                                                                                   |
| One query example                                 | Set                             | Defines a simple query string                          | Run example                                      | Append execution data for single query example |                                                                                                                   |
| Truncate Chat Memory                              | Langchain MemoryManager (Delete) | Clears chat memory for clean example state            | One query example                                | Fill Chat Memory with example data              | # Environment setup For Chat History Node                                                                          |
| Fill Chat Memory with example data                | Langchain MemoryManager (Insert) | Inserts example chat messages                          | Truncate Chat Memory                             | Load Chat Memory Data                           |                                                                                                                   |
| Load Chat Memory Data                             | Langchain MemoryManager (Load)   | Loads stored chat memory                               | Fill Chat Memory with example data               | Append execution data for chat memory example  |                                                                                                                   |
| Append execution data for chat memory example     | Set                             | Adds metadata for batch request from chat memory       | Load Chat Memory Data                            | Build batch 'request' object from Chat Memory and execution data | custom_id, model and max tokens                                                                                     |
| Build batch 'request' object from Chat Memory and execution data | Code                            | Converts chat memory data to Anthropic batch request format | Append execution data for chat memory example    | Join two example requests into array             |                                                                                                                   |
| Append execution data for single query example    | Set                             | Adds metadata for batch request from simple query      | One query example                               | Build batch 'request' object for single query  | custom_id, model and max tokens                                                                                     |
| Build batch 'request' object for single query     | Code                            | Converts simple query to Anthropic batch request format | Append execution data for single query example  | Delete original properties                      |                                                                                                                   |
| Delete original properties                        | Set                             | Keeps only required fields for batch request           | Build batch 'request' object for single query    | Join two example requests into array             |                                                                                                                   |
| Join two example requests into array              | Merge                           | Combines chat memory and simple query batch requests   | Build batch 'request' object from Chat Memory and execution data, Delete original properties | Construct 'requests' array                      |                                                                                                                   |
| Construct 'requests' array                         | Aggregate                      | Aggregates all requests into single array for submission | Join two example requests into array             | Set desired 'anthropic-version'                 |                                                                                                                   |
| Set desired 'anthropic-version'                    | Set                             | Sets API version header value                           | Construct 'requests' array                       | Execute Workflow 'Process Multiple Prompts in Parallel with Anthropic Claude Batch API' | 2023-06-01                                                                                                         |
| Execute Workflow 'Process Multiple Prompts in Parallel with Anthropic Claude Batch API' | Execute Workflow               | Calls core batch processing workflow                    | Set desired 'anthropic-version'                  | Filter First Prompt Results, Filter Second Prompt Results | See above                                                                                                          |
| Filter First Prompt Results                        | Filter                         | Filters results by first prompt's custom_id             | Execute Workflow 'Process Multiple Prompts...'  | First Prompt Result                             |                                                                                                                   |
| Filter Second Prompt Results                       | Filter                         | Filters results by second prompt's custom_id            | Execute Workflow 'Process Multiple Prompts...'  | Second Prompt Result                            |                                                                                                                   |
| First Prompt Result                                | Execution Data                 | Extracts assistant response text from first prompt result | Filter First Prompt Results                      |                                                 | # Example usage with Chat History Node (result)                                                                    |
| Second Prompt Result                               | Execution Data                 | Extracts assistant response text from second prompt result | Filter Second Prompt Results                     |                                                 | # Example usage with single query string (result)                                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Core Batch Processing Workflow:**

   1. Add an **Execute Workflow Trigger** node named `When Executed by Another Workflow`. Configure it to accept two inputs:  
      - `anthropic-version` (string)  
      - `requests` (array)

   2. Add an **HTTP Request** node named `Submit batch`:  
      - Method: POST  
      - URL: `https://api.anthropic.com/v1/messages/batches`  
      - Authentication: Anthropic API credential  
      - Headers: Add header `anthropic-version` with value from input (`={{ $json["anthropic-version"] }}`)  
      - Body: JSON, with property `requests` set to the stringified input array (`={ "requests": {{ JSON.stringify($json.requests) }} }`)  
      - Connect `When Executed by Another Workflow` → `Submit batch`

   3. Add an **If** node named `If ended processing`:  
      - Condition: Check if `processing_status` equals `"ended"`  
      - Connect `Submit batch` → `If ended processing`

   4. Add an **HTTP Request** node named `Get results`:  
      - Method: GET  
      - URL: dynamic from `results_url` field (`={{ $json.results_url }}`)  
      - Authentication: Anthropic API credential  
      - Header: `anthropic-version` from initial input  
      - Connect `If ended processing` (true branch) → `Get results`

   5. Add a **Wait** node named `Batch Status Poll Interval`:  
      - Wait time: 10 seconds (adjustable)  
      - Connect `If ended processing` (false branch) → `Batch Status Poll Interval`

   6. Add an **HTTP Request** node named `Check batch status`:  
      - Method: GET  
      - URL: `https://api.anthropic.com/v1/messages/batches/{{ $json.id }}` (dynamic batch id)  
      - Authentication: Anthropic API credential  
      - Header: `anthropic-version` from initial input  
      - Connect `Batch Status Poll Interval` → `Check batch status`  
      - Connect `Check batch status` → `If ended processing` (loop)

   7. Add a **Code** node named `Parse response`:  
      - JavaScript code to split the JSON Lines response text into an array of JSON objects stored in `parsed` field:  
        ```javascript
        for (const item of $input.all()) {
          if (item.json && item.json.data) {
            const jsonStrings = item.json.data.split('\n');
            const parsedData = jsonStrings.filter(str => str.trim() !== '').map(str => JSON.parse(str));
            item.json.parsed = parsedData;
          }
        }
        return $input.all();
        ```
      - Connect `Get results` → `Parse response`

   8. Add a **Split Out** node named `Split Out Parsed Results`:  
      - Field to split out: `parsed`  
      - Connect `Parse response` → `Split Out Parsed Results`

2. **Create the Example Usage Branch:**

   1. Add a **Manual Trigger** node named `Run example`

   2. Add a **Set** node named `One query example`:  
      - Set field `query` to a sample string, e.g., `"Hey Claude, tell me a short fun fact about bees!"`  
      - Connect `Run example` → `One query example`

   3. Add a **Langchain MemoryManager** node named `Truncate Chat Memory`:  
      - Mode: Delete all  
      - Connect `One query example` → `Truncate Chat Memory`

   4. Add a **Langchain MemoryManager** node named `Fill Chat Memory with example data`:  
      - Mode: Insert  
      - Insert example conversation messages simulating chat history  
      - Connect `Truncate Chat Memory` → `Fill Chat Memory with example data`

   5. Add a **Langchain MemoryManager** node named `Load Chat Memory Data`:  
      - Mode: Load  
      - Connect `Fill Chat Memory with example data` → `Load Chat Memory Data`

   6. Add a **Set** node named `Append execution data for chat memory example`:  
      - Add fields:  
        - `custom_id`: e.g., `"first-prompt-in-my-batch"`  
        - `model`: e.g., `"claude-3-5-haiku-20241022"`  
        - `max_tokens`: 100  
      - Connect `Load Chat Memory Data` → `Append execution data for chat memory example`

   7. Add a **Code** node named `Build batch 'request' object from Chat Memory and execution data`:  
      - JavaScript to transform chat memory messages into Anthropic batch request format with roles and system message handling  
      - Connect `Append execution data for chat memory example` → `Build batch 'request' object from Chat Memory and execution data`

   8. Add a **Set** node named `Append execution data for single query example`:  
      - Add fields:  
        - `custom_id`: e.g., `"second-prompt-in-my-batch"`  
        - `model`: same as above  
        - `max_tokens`: 100  
      - Connect `One query example` → `Append execution data for single query example`

   9. Add a **Code** node named `Build batch 'request' object for single query`:  
      - JavaScript to convert simple query string into Anthropic batch request format  
      - Connect `Append execution data for single query example` → `Build batch 'request' object for single query`

   10. Add a **Set** node named `Delete original properties`:  
       - Keep only `custom_id` and `params` fields to match API schema  
       - Connect `Build batch 'request' object for single query` → `Delete original properties`

   11. Add a **Merge** node named `Join two example requests into array`:  
       - Merge inputs from:  
         - `Build batch 'request' object from Chat Memory and execution data`  
         - `Delete original properties`  
       - Connect both → `Join two example requests into array`

   12. Add an **Aggregate** node named `Construct 'requests' array`:  
       - Aggregate all items into a single array field named `requests`  
       - Connect `Join two example requests into array` → `Construct 'requests' array`

   13. Add a **Set** node named `Set desired 'anthropic-version'`:  
       - Set field `anthropic-version` to `"2023-06-01"` (latest supported version)  
       - Include other fields  
       - Connect `Construct 'requests' array` → `Set desired 'anthropic-version'`

   14. Add an **Execute Workflow** node named `Execute Workflow 'Process Multiple Prompts in Parallel with Anthropic Claude Batch API'`:  
       - Select the core batch processing workflow by ID or name  
       - Map inputs:  
         - `requests` from current JSON  
         - `anthropic-version` from current JSON  
       - Enable "Wait for Sub-Workflow" to get results synchronously  
       - Connect `Set desired 'anthropic-version'` → `Execute Workflow`

   15. Add two **Filter** nodes named `Filter First Prompt Results` and `Filter Second Prompt Results`:  
       - Filter by `custom_id` matching each prompt’s ID  
       - Connect `Execute Workflow` → both Filter nodes

   16. Add two **Execution Data** nodes named `First Prompt Result` and `Second Prompt Result`:  
       - Extract assistant response text from filtered results  
       - Connect `Filter First Prompt Results` → `First Prompt Result`  
       - Connect `Filter Second Prompt Results` → `Second Prompt Result`

---

### 5. General Notes & Resources

| Note Content                                                                                                                           | Context or Link                                                                                         |
|----------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| This workflow automates sending batched prompts to Claude using the Anthropic API. Call this workflow with an array of `requests`.     | Sticky Note near entry point node                                                                      |
| Results are returned as an array of objects with `custom_id` and `result` fields, containing the assistant's message and usage details. | Sticky Note near output parsing nodes                                                                  |
| Polling interval for batch status check is set to 10 seconds by default; adjust carefully to balance responsiveness and rate limits.   | Workflow description and Sticky Note near Wait node                                                   |
| Anthropic Batch API supports up to 100,000 requests per batch and 256 MB payload size; ensure your n8n instance can handle large data. | Official Anthropic API documentation: https://docs.anthropic.com/en/api/creating-message-batches       |
| Always include the `anthropic-version` header in API requests; recommended version is `2023-06-01`.                                     | Anthropic API versioning docs: https://docs.anthropic.com/en/api/versioning                             |
| Example usage branch demonstrates integration with n8n Langchain Chat Memory nodes for advanced prompt construction.                   | n8n Langchain docs: https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.memorybufferwindow/ |
| Enhance error handling by adding nodes to check HTTP status codes or handle batch failures (`batch.status === 'failed'`).              | Suggested customization                                                                                 |
| Batch processing reduces latency and improves throughput compared to sequential prompt submission.                                      | Workflow description                                                                                     |

---

This structured documentation enables advanced users and automation agents to understand, reproduce, and customize the workflow for batch processing prompts with Anthropic Claude models efficiently.