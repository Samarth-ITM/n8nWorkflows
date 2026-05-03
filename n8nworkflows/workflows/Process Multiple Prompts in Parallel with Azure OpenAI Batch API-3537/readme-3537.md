Process Multiple Prompts in Parallel with Azure OpenAI Batch API

https://n8nworkflows.xyz/workflows/process-multiple-prompts-in-parallel-with-azure-openai-batch-api-3537


# Process Multiple Prompts in Parallel with Azure OpenAI Batch API

### 1. Workflow Overview

This workflow automates the process of sending multiple prompts to the Azure OpenAI Batch API in parallel, enabling efficient batch processing of large volumes of text data. It is designed for developers and data scientists who need to process multiple requests asynchronously, optimizing time and resource usage.

The workflow is logically divided into the following blocks:

- **1.1 Input Preparation and Defaults Setup**: Accepts input requests, sets default parameters, and prepares data structures.
- **1.2 Batch Request Construction**: Converts individual prompts into batch-compatible JSONL format and packages them into a file.
- **1.3 File Upload and Batch Job Creation**: Uploads the batch file to Azure OpenAI, creates a batch job, and manages job status polling.
- **1.4 Batch Job Output Retrieval and Parsing**: Retrieves the processed batch output file, parses JSONL results, and splits responses.
- **1.5 Results Filtering and Output**: Filters and separates individual prompt results for further use or return.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Preparation and Defaults Setup

**Overview:**  
This block initializes the workflow by receiving input requests, setting default API parameters, and preparing chat memory data for batch processing.

**Nodes Involved:**  
- When Executed by Another Workflow  
- Setup defaults  
- Set defaults if not set already  
- One query example  
- Truncate Chat Memory  
- Fill Chat Memory with example data  
- Load Chat Memory Data  
- Append custom_id for single query example  
- Append custom_id for chat memory example  
- Build batch 'request' object for single query  
- Build batch 'request' object from Chat Memory and execution data  
- Join two example requests into array  
- Construct 'requests' array  
- Set desired 'api-version'  

**Node Details:**

- **When Executed by Another Workflow**  
  - *Type:* Execute Workflow Trigger  
  - *Role:* Entry point for external workflow execution with input requests.  
  - *Config:* No parameters; expects input JSON array with `api-version` and `requests`.  
  - *Connections:* Outputs to "Setup defaults".  
  - *Edge Cases:* Missing or malformed input may cause failures downstream.

- **Setup defaults**  
  - *Type:* Set  
  - *Role:* Sets default values such as `az_openai_endpoint`.  
  - *Config:* Contains endpoint URL and other default parameters.  
  - *Connections:* Outputs to "Set defaults if not set already".  
  - *Edge Cases:* Missing or incorrect endpoint causes HTTP request failures.

- **Set defaults if not set already**  
  - *Type:* Code  
  - *Role:* Ensures required parameters like `api-version` are set if absent.  
  - *Config:* JavaScript code setting defaults conditionally.  
  - *Connections:* Outputs to "JSON requests to JSONL".  
  - *Edge Cases:* Logic errors in code could cause missing parameters.

- **One query example**  
  - *Type:* Set  
  - *Role:* Provides a sample single prompt request for testing.  
  - *Config:* Contains example prompt messages.  
  - *Connections:* Outputs to "Append custom_id for single query example" and "Truncate Chat Memory".  
  - *Edge Cases:* Example data may not reflect real input structure.

- **Truncate Chat Memory**  
  - *Type:* Langchain Memory Manager  
  - *Role:* Clears chat memory to ensure clean state before example data insertion.  
  - *Config:* No parameters; resets memory buffer.  
  - *Connections:* Outputs to "Fill Chat Memory with example data".  
  - *Edge Cases:* Failure to clear memory may cause stale data issues.

- **Fill Chat Memory with example data**  
  - *Type:* Langchain Memory Manager  
  - *Role:* Loads example chat conversation data into memory.  
  - *Config:* Predefined example messages.  
  - *Connections:* Outputs to "Load Chat Memory Data".  
  - *Edge Cases:* Memory load failure could disrupt batch request construction.

- **Load Chat Memory Data**  
  - *Type:* Langchain Memory Manager  
  - *Role:* Retrieves chat memory content for batch request building.  
  - *Config:* No parameters.  
  - *Connections:* Outputs to "Append custom_id for chat memory example".  
  - *Edge Cases:* Empty or corrupted memory data.

- **Append custom_id for single query example**  
  - *Type:* Set  
  - *Role:* Adds a unique `custom_id` to the single query request.  
  - *Config:* Sets `custom_id` field with a string identifier.  
  - *Connections:* Outputs to "Build batch 'request' object for single query".  
  - *Edge Cases:* Missing or duplicate IDs may cause result mapping issues.

- **Append custom_id for chat memory example**  
  - *Type:* Set  
  - *Role:* Adds a unique `custom_id` to chat memory-based requests.  
  - *Config:* Sets `custom_id` field similarly.  
  - *Connections:* Outputs to "Build batch 'request' object from Chat Memory and execution data".  
  - *Edge Cases:* Same as above.

- **Build batch 'request' object for single query**  
  - *Type:* Code  
  - *Role:* Constructs a batch-compatible request object from single query data.  
  - *Config:* JavaScript code formats the request with `custom_id` and `params`.  
  - *Connections:* Outputs to "Delete original properties".  
  - *Edge Cases:* Code errors or malformed data.

- **Build batch 'request' object from Chat Memory and execution data**  
  - *Type:* Code  
  - *Role:* Builds batch request objects from chat memory and execution context.  
  - *Config:* JavaScript code combining memory data and parameters.  
  - *Connections:* Outputs to "Join two example requests into array".  
  - *Edge Cases:* Data inconsistency between memory and execution data.

- **Join two example requests into array**  
  - *Type:* Merge  
  - *Role:* Combines single query and chat memory requests into a single array.  
  - *Config:* Merge mode: Append.  
  - *Connections:* Outputs to "Construct 'requests' array".  
  - *Edge Cases:* Duplicate or missing entries.

- **Construct 'requests' array**  
  - *Type:* Aggregate  
  - *Role:* Aggregates all requests into a single array object for batch processing.  
  - *Config:* Aggregation on input items.  
  - *Connections:* Outputs to "Set desired 'api-version'".  
  - *Edge Cases:* Empty aggregation results.

- **Set desired 'api-version'**  
  - *Type:* Set  
  - *Role:* Sets the OpenAI API version to use (default: "2025-03-01-preview").  
  - *Config:* Static value for `api-version`.  
  - *Connections:* Outputs to "Execute Workflow 'Process Multiple Prompts in Parallel with Azure OpenAI Batch API'".  
  - *Edge Cases:* Unsupported API version may cause request failures.

---

#### 1.2 Batch Request Construction

**Overview:**  
Transforms the array of requests into the JSONL format required by the Azure OpenAI Batch API, converts the JSONL string into a file, and prepares it for upload.

**Nodes Involved:**  
- Execute Workflow 'Process Multiple Prompts in Parallel with Azure OpenAI Batch API' (sub-workflow)  
- JSON requests to JSONL  
- Remove original requests  
- Convert requests jsonl to File  
- Change file name and mimetype  
- Merge File and API Version  

**Node Details:**

- **JSON requests to JSONL**  
  - *Type:* Code  
  - *Role:* Converts JSON array of requests into JSONL string (newline-delimited JSON).  
  - *Config:* JavaScript code iterates requests and serializes each line.  
  - *Connections:* Outputs to "Remove original requests" and "Convert requests jsonl to File".  
  - *Edge Cases:* Malformed JSON or empty input.

- **Remove original requests**  
  - *Type:* Set  
  - *Role:* Removes the original `requests` property from data, retaining only necessary fields like `api-version`.  
  - *Config:* Clears `requests` property.  
  - *Connections:* Outputs to "Merge File and API Version".  
  - *Edge Cases:* Accidental removal of needed data.

- **Convert requests jsonl to File**  
  - *Type:* Convert To File  
  - *Role:* Converts the JSONL string into a file object with a timestamped filename ending in `.jsonl`.  
  - *Config:* Filename pattern `%current-datetime%.jsonl`.  
  - *Connections:* Outputs to "Change file name and mimetype".  
  - *Edge Cases:* File conversion failures.

- **Change file name and mimetype**  
  - *Type:* Code  
  - *Role:* Adjusts the file metadata to ensure correct mimetype and filename before upload.  
  - *Config:* JavaScript code modifies file properties.  
  - *Connections:* Outputs to "Merge File and API Version".  
  - *Edge Cases:* Incorrect mimetype may cause upload rejection.

- **Merge File and API Version**  
  - *Type:* Merge  
  - *Role:* Combines the file object and API version data into a single payload for upload.  
  - *Config:* Merge mode: Append.  
  - *Connections:* Outputs to "Upload batch file".  
  - *Edge Cases:* Data mismatch or missing properties.

---

#### 1.3 File Upload and Batch Job Creation

**Overview:**  
Uploads the batch JSONL file to Azure OpenAI, creates a batch job to process the prompts, and polls for upload and job completion status.

**Nodes Involved:**  
- Upload batch file  
- If upload processed  
- File Upload Poll Interval  
- Track file upload status  
- Create batch job  
- If ended processing  
- Batch Status Poll Interval  
- Track batch job progress  

**Node Details:**

- **Upload batch file**  
  - *Type:* HTTP Request  
  - *Role:* Sends POST request to `/openai/files` endpoint to upload the batch file.  
  - *Config:* Uses Azure OpenAI credentials, multipart/form-data with file.  
  - *Connections:* Outputs to "If upload processed" (true branch) and "File Upload Poll Interval" (false branch).  
  - *Edge Cases:* Authentication errors, file size limits, network timeouts.

- **If upload processed**  
  - *Type:* If  
  - *Role:* Checks if file upload has completed successfully.  
  - *Config:* Condition based on HTTP response status or upload state.  
  - *Connections:* True branch to "Create batch job", false branch to "File Upload Poll Interval".  
  - *Edge Cases:* False negatives causing unnecessary polling.

- **File Upload Poll Interval**  
  - *Type:* Wait  
  - *Role:* Waits a configured interval before retrying upload status check.  
  - *Config:* Default wait time (e.g., seconds).  
  - *Connections:* Outputs to "Track file upload status".  
  - *Edge Cases:* Long wait times may delay processing.

- **Track file upload status**  
  - *Type:* HTTP Request  
  - *Role:* Sends GET request to `/openai/files/{file_id}` to check upload status.  
  - *Config:* Uses file ID from upload response, Azure OpenAI credentials.  
  - *Connections:* Outputs to "If upload processed".  
  - *Edge Cases:* API rate limits, invalid file ID.

- **Create batch job**  
  - *Type:* HTTP Request  
  - *Role:* Sends POST request to `/openai/batches` to start batch processing job.  
  - *Config:* Includes uploaded file ID and parameters, uses Azure OpenAI credentials.  
  - *Connections:* Outputs to "If ended processing".  
  - *Edge Cases:* Job creation failure, invalid parameters.

- **If ended processing**  
  - *Type:* If  
  - *Role:* Checks if batch job has completed processing.  
  - *Config:* Condition based on job status field (e.g., "succeeded", "failed", "running").  
  - *Connections:* True branch to "Retrieve batch job output file", false branch to "Batch Status Poll Interval".  
  - *Edge Cases:* Job stuck in pending or error states.

- **Batch Status Poll Interval**  
  - *Type:* Wait  
  - *Role:* Waits before polling batch job status again.  
  - *Config:* Default wait time.  
  - *Connections:* Outputs to "Track batch job progress".  
  - *Edge Cases:* Excessive polling may hit API limits.

- **Track batch job progress**  
  - *Type:* HTTP Request  
  - *Role:* Sends GET request to `/openai/batches/{batch_id}` to check job status.  
  - *Config:* Uses batch job ID, Azure OpenAI credentials.  
  - *Connections:* Outputs to "If ended processing".  
  - *Edge Cases:* API errors, job cancellation.

---

#### 1.4 Batch Job Output Retrieval and Parsing

**Overview:**  
Once the batch job completes, this block retrieves the output file, parses the JSONL response, and splits it into individual results.

**Nodes Involved:**  
- Retrieve batch job output file  
- Parse response  
- Split Out Parsed Results  

**Node Details:**

- **Retrieve batch job output file**  
  - *Type:* HTTP Request  
  - *Role:* Downloads the batch job output file content from `/openai/files/{output_file_id}/content`.  
  - *Config:* Uses output file ID, Azure OpenAI credentials.  
  - *Connections:* Outputs to "Parse response".  
  - *Edge Cases:* File not found, network errors.

- **Parse response**  
  - *Type:* Code  
  - *Role:* Parses JSONL response string into JSON objects array.  
  - *Config:* JavaScript code splitting by newline and parsing each line.  
  - *Connections:* Outputs to "Split Out Parsed Results".  
  - *Edge Cases:* Malformed JSONL lines.

- **Split Out Parsed Results**  
  - *Type:* Split Out  
  - *Role:* Splits array of parsed results into individual items for downstream processing.  
  - *Config:* Default split mode.  
  - *Connections:* Outputs to filters for individual prompt results.  
  - *Edge Cases:* Empty or missing results array.

---

#### 1.5 Results Filtering and Output

**Overview:**  
Filters the parsed batch results to separate individual prompt responses by their `custom_id` and prepares them for output or further processing.

**Nodes Involved:**  
- Filter First Prompt Results  
- Filter Second Prompt Results  
- First Prompt Result  
- Second Prompt Result  

**Node Details:**

- **Filter First Prompt Results**  
  - *Type:* Filter  
  - *Role:* Filters parsed results to isolate the response with `custom_id` matching the first prompt.  
  - *Config:* Condition on `custom_id == "first-prompt-in-my-batch"`.  
  - *Connections:* Outputs to "First Prompt Result".  
  - *Edge Cases:* Missing or duplicated IDs.

- **Filter Second Prompt Results**  
  - *Type:* Filter  
  - *Role:* Filters parsed results for the second prompt's `custom_id`.  
  - *Config:* Condition on `custom_id == "second-prompt-in-my-batch"`.  
  - *Connections:* Outputs to "Second Prompt Result".  
  - *Edge Cases:* Same as above.

- **First Prompt Result**  
  - *Type:* Execution Data  
  - *Role:* Holds the filtered first prompt response for output or further use.  
  - *Config:* No parameters.  
  - *Connections:* Terminal node for first prompt.  
  - *Edge Cases:* Empty data if filtering fails.

- **Second Prompt Result**  
  - *Type:* Execution Data  
  - *Role:* Holds the filtered second prompt response.  
  - *Config:* No parameters.  
  - *Connections:* Terminal node for second prompt.  
  - *Edge Cases:* Same as above.

---

### 3. Summary Table

| Node Name                                  | Node Type                              | Functional Role                                      | Input Node(s)                             | Output Node(s)                             | Sticky Note                                                                                  |
|--------------------------------------------|--------------------------------------|-----------------------------------------------------|------------------------------------------|--------------------------------------------|----------------------------------------------------------------------------------------------|
| When Executed by Another Workflow           | Execute Workflow Trigger              | Entry point for external execution                   | -                                        | Setup defaults                             |                                                                                              |
| Setup defaults                              | Set                                  | Sets default parameters (e.g., endpoint)             | When Executed by Another Workflow         | Set defaults if not set already             |                                                                                              |
| Set defaults if not set already             | Code                                 | Ensures required parameters are set                   | Setup defaults                            | JSON requests to JSONL                      |                                                                                              |
| One query example                           | Set                                  | Provides example single prompt                         | Run example                              | Append custom_id for single query example, Truncate Chat Memory |                                                                                              |
| Truncate Chat Memory                        | Langchain Memory Manager              | Clears chat memory for clean state                     | One query example                        | Fill Chat Memory with example data          | ensure clean state                                                                           |
| Fill Chat Memory with example data          | Langchain Memory Manager              | Loads example chat data into memory                     | Truncate Chat Memory                     | Load Chat Memory Data                        |                                                                                              |
| Load Chat Memory Data                       | Langchain Memory Manager              | Retrieves chat memory content                           | Fill Chat Memory with example data       | Append custom_id for chat memory example     |                                                                                              |
| Append custom_id for single query example   | Set                                  | Adds unique custom_id to single query                   | One query example                        | Build batch 'request' object for single query | custom_id                                                                                   |
| Append custom_id for chat memory example    | Set                                  | Adds unique custom_id to chat memory requests           | Load Chat Memory Data                    | Build batch 'request' object from Chat Memory and execution data | custom_id                                                                                   |
| Build batch 'request' object for single query | Code                                 | Constructs batch request object from single query       | Append custom_id for single query example | Delete original properties                   |                                                                                              |
| Build batch 'request' object from Chat Memory and execution data | Code                                 | Constructs batch request object from chat memory data    | Append custom_id for chat memory example | Join two example requests into array          |                                                                                              |
| Join two example requests into array        | Merge                                | Combines single and chat memory requests into array     | Build batch 'request' object from Chat Memory and execution data, Delete original properties | Construct 'requests' array                   |                                                                                              |
| Construct 'requests' array                   | Aggregate                           | Aggregates requests into a single array                  | Join two example requests into array      | Set desired 'api-version'                    |                                                                                              |
| Set desired 'api-version'                    | Set                                  | Sets OpenAI API version                                 | Construct 'requests' array                | Execute Workflow 'Process Multiple Prompts in Parallel with Azure OpenAI Batch API' | 2025-03-01-preview                                                                           |
| Execute Workflow 'Process Multiple Prompts in Parallel with Azure OpenAI Batch API' | Execute Workflow                    | Sub-workflow for batch processing                       | Set desired 'api-version'                 | Filter First Prompt Results, Filter Second Prompt Results | See above                                                                                   |
| JSON requests to JSONL                       | Code                                 | Converts JSON array to JSONL string                      | Set defaults if not set already           | Remove original requests, Convert requests jsonl to File |                                                                                              |
| Remove original requests                     | Set                                  | Removes original requests property                       | JSON requests to JSONL                    | Merge File and API Version                   | Keep 'api-version'                                                                           |
| Convert requests jsonl to File               | Convert To File                      | Converts JSONL string to file object                     | JSON requests to JSONL                    | Change file name and mimetype                | %current-datetime%.jsonl                                                                     |
| Change file name and mimetype                | Code                                 | Adjusts file metadata before upload                      | Convert requests jsonl to File            | Merge File and API Version                   |                                                                                              |
| Merge File and API Version                    | Merge                                | Combines file and API version data                       | Remove original requests, Change file name and mimetype | Upload batch file                            |                                                                                              |
| Upload batch file                            | HTTP Request                        | Uploads batch file to Azure OpenAI                       | Merge File and API Version                | If upload processed, File Upload Poll Interval | POST /openai/files                                                                          |
| If upload processed                         | If                                   | Checks if file upload is complete                        | Upload batch file                        | Create batch job (true), File Upload Poll Interval (false) |                                                                                              |
| File Upload Poll Interval                    | Wait                                 | Waits before polling upload status                       | If upload processed                      | Track file upload status                     |                                                                                              |
| Track file upload status                      | HTTP Request                        | Polls upload status via GET /openai/files/file_id       | File Upload Poll Interval                 | If upload processed                          | GET /openai/files/file_id                                                                    |
| Create batch job                            | HTTP Request                        | Creates batch processing job                             | If upload processed                      | If ended processing                          | POST /openai/batches                                                                        |
| If ended processing                         | If                                   | Checks if batch job has completed                        | Create batch job, Track batch job progress | Retrieve batch job output file (true), Batch Status Poll Interval (false) |                                                                                              |
| Batch Status Poll Interval                    | Wait                                 | Waits before polling batch job status                    | If ended processing                      | Track batch job progress                     |                                                                                              |
| Track batch job progress                      | HTTP Request                        | Polls batch job status via GET /openai/batches/batch_id | Batch Status Poll Interval                | If ended processing                          | GET /openai/batches/batch_id                                                                |
| Retrieve batch job output file                | HTTP Request                        | Downloads batch job output file content                   | If ended processing                      | Parse response                              | GET /openai/files/output_file_id/content                                                    |
| Parse response                              | Code                                 | Parses JSONL output into JSON array                       | Retrieve batch job output file            | Split Out Parsed Results                      | JSONL separated by newlines                                                                 |
| Split Out Parsed Results                     | Split Out                           | Splits parsed results array into individual items         | Parse response                          | Filter First Prompt Results, Filter Second Prompt Results |                                                                                              |
| Filter First Prompt Results                   | Filter                              | Filters results for first prompt by custom_id             | Split Out Parsed Results                  | First Prompt Result                          |                                                                                              |
| Filter Second Prompt Results                  | Filter                              | Filters results for second prompt by custom_id            | Split Out Parsed Results                  | Second Prompt Result                         |                                                                                              |
| First Prompt Result                          | Execution Data                     | Holds first prompt filtered result                        | Filter First Prompt Results               | -                                          |                                                                                              |
| Second Prompt Result                         | Execution Data                     | Holds second prompt filtered result                       | Filter Second Prompt Results              | -                                          |                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Entry Trigger Node**  
   - Add **Execute Workflow Trigger** node named "When Executed by Another Workflow".  
   - No parameters; this node receives input requests.

2. **Setup Defaults**  
   - Add **Set** node "Setup defaults".  
   - Configure to set `az_openai_endpoint` with your Azure OpenAI endpoint URL.  
   - Connect "When Executed by Another Workflow" → "Setup defaults".

3. **Set Defaults if Not Set Already**  
   - Add **Code** node "Set defaults if not set already".  
   - Write JavaScript to check and set default `api-version` if missing (e.g., "2025-03-01-preview").  
   - Connect "Setup defaults" → "Set defaults if not set already".

4. **Prepare Example Input (Optional for Testing)**  
   - Add **Manual Trigger** node "Run example".  
   - Connect to **Set** node "One query example" with sample prompt data.  
   - Connect "One query example" → "Append custom_id for single query example" (Set node to add `custom_id`).  
   - Add **Langchain Memory Manager** nodes:  
     - "Truncate Chat Memory" to clear memory.  
     - "Fill Chat Memory with example data" to load example chat data.  
     - "Load Chat Memory Data" to retrieve memory.  
   - Connect "Run example" → "One query example" and "Truncate Chat Memory".  
   - Connect "Truncate Chat Memory" → "Fill Chat Memory with example data" → "Load Chat Memory Data" → "Append custom_id for chat memory example".

5. **Build Batch Request Objects**  
   - Add **Code** node "Build batch 'request' object for single query" to format single query with `custom_id` and `params`.  
   - Add **Code** node "Build batch 'request' object from Chat Memory and execution data" for chat memory data.  
   - Connect "Append custom_id for single query example" → "Build batch 'request' object for single query".  
   - Connect "Append custom_id for chat memory example" → "Build batch 'request' object from Chat Memory and execution data".

6. **Join Requests into Array**  
   - Add **Merge** node "Join two example requests into array" (append mode).  
   - Connect both batch request objects to this merge node.

7. **Aggregate Requests**  
   - Add **Aggregate** node "Construct 'requests' array" to combine requests into one array.  
   - Connect "Join two example requests into array" → "Construct 'requests' array".

8. **Set API Version**  
   - Add **Set** node "Set desired 'api-version'" with value "2025-03-01-preview".  
   - Connect "Construct 'requests' array" → "Set desired 'api-version'".

9. **Convert JSON to JSONL**  
   - Add **Code** node "JSON requests to JSONL" to serialize requests array into newline-delimited JSON.  
   - Connect "Set defaults if not set already" → "JSON requests to JSONL".

10. **Remove Original Requests**  
    - Add **Set** node "Remove original requests" to clear `requests` property but keep `api-version`.  
    - Connect "JSON requests to JSONL" → "Remove original requests".

11. **Convert JSONL to File**  
    - Add **Convert To File** node "Convert requests jsonl to File".  
    - Configure filename pattern `%current-datetime%.jsonl`.  
    - Connect "JSON requests to JSONL" → "Convert requests jsonl to File".

12. **Change File Name and Mimetype**  
    - Add **Code** node "Change file name and mimetype" to set correct mimetype for `.jsonl` file.  
    - Connect "Convert requests jsonl to File" → "Change file name and mimetype".

13. **Merge File and API Version**  
    - Add **Merge** node "Merge File and API Version" (append mode).  
    - Connect "Remove original requests" and "Change file name and mimetype" to this node.

14. **Upload Batch File**  
    - Add **HTTP Request** node "Upload batch file".  
    - Configure:  
      - Method: POST  
      - URL: `/openai/files` (full Azure OpenAI endpoint URL)  
      - Authentication: Azure OpenAI credentials  
      - Body: multipart/form-data with file from merge node  
    - Connect "Merge File and API Version" → "Upload batch file".

15. **Check Upload Status**  
    - Add **If** node "If upload processed" to check upload completion from response.  
    - Connect "Upload batch file" → "If upload processed".

16. **File Upload Poll Interval**  
    - Add **Wait** node "File Upload Poll Interval" with suitable delay (e.g., 5 seconds).  
    - Connect "If upload processed" false branch → "File Upload Poll Interval".

17. **Track File Upload Status**  
    - Add **HTTP Request** node "Track file upload status".  
    - Configure GET `/openai/files/{file_id}` with file ID from upload response.  
    - Connect "File Upload Poll Interval" → "Track file upload status".  
    - Connect "Track file upload status" → "If upload processed".

18. **Create Batch Job**  
    - Add **HTTP Request** node "Create batch job".  
    - Configure POST `/openai/batches` with uploaded file ID and parameters.  
    - Connect "If upload processed" true branch → "Create batch job".

19. **Check Batch Job Completion**  
    - Add **If** node "If ended processing" to check batch job status.  
    - Connect "Create batch job" → "If ended processing".

20. **Batch Status Poll Interval**  
    - Add **Wait** node "Batch Status Poll Interval" with delay (e.g., 10 seconds).  
    - Connect "If ended processing" false branch → "Batch Status Poll Interval".

21. **Track Batch Job Progress**  
    - Add **HTTP Request** node "Track batch job progress".  
    - Configure GET `/openai/batches/{batch_id}` with batch job ID.  
    - Connect "Batch Status Poll Interval" → "Track batch job progress".  
    - Connect "Track batch job progress" → "If ended processing".

22. **Retrieve Batch Job Output File**  
    - Add **HTTP Request** node "Retrieve batch job output file".  
    - Configure GET `/openai/files/{output_file_id}/content`.  
    - Connect "If ended processing" true branch → "Retrieve batch job output file".

23. **Parse Response**  
    - Add **Code** node "Parse response".  
    - JavaScript code to split JSONL string by newline and parse each line into JSON.  
    - Connect "Retrieve batch job output file" → "Parse response".

24. **Split Parsed Results**  
    - Add **Split Out** node "Split Out Parsed Results".  
    - Connect "Parse response" → "Split Out Parsed Results".

25. **Filter Results by custom_id**  
    - Add **Filter** nodes "Filter First Prompt Results" and "Filter Second Prompt Results".  
    - Configure filters to match `custom_id` for each prompt.  
    - Connect "Split Out Parsed Results" → both filter nodes.

26. **Output Filtered Results**  
    - Add **Execution Data** nodes "First Prompt Result" and "Second Prompt Result" to hold filtered data.  
    - Connect filters to respective execution data nodes.

27. **Credentials Setup**  
    - Configure Azure OpenAI credentials in n8n with appropriate API keys and endpoint URLs.  
    - Ensure HTTP Request nodes use these credentials.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                        | Context or Link                                                                                      |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Azure OpenAI Batch API supports asynchronous processing with cost and time efficiency advantages over sequential requests.                                                                                                                                       | Workflow description                                                                                |
| Job cancellation is possible at any time; charges apply for completed work only.                                                                                                                                                                                   | Additional Notes                                                                                   |
| Data residency ensures data at rest stays in designated Azure geography; inferencing may occur in any Azure OpenAI location.                                                                                                                                       | Additional Notes                                                                                   |
| Exponential backoff strategy recommended for large batch jobs hitting token limits; some regions support queuing multiple batch jobs.                                                                                                                            | Additional Notes                                                                                   |
| Example input and output JSON provided in workflow description for reference.                                                                                                                                                                                       | Workflow description                                                                                |
| For detailed API documentation, refer to Azure OpenAI Batch API official docs: https://learn.microsoft.com/en-us/azure/cognitive-services/openai/reference#batch-requests                                                                                         | External resource                                                                                  |
| This workflow uses Langchain memory nodes for chat memory management; ensure Langchain nodes are installed and configured in your n8n environment.                                                                                                                | Implementation detail                                                                              |
| Polling intervals for upload and batch job status can be adjusted in Wait nodes to optimize performance and API rate limits.                                                                                                                                       | Implementation detail                                                                              |

---

This document provides a detailed, structured reference for understanding, reproducing, and modifying the "Process Multiple Prompts in Parallel with Azure OpenAI Batch API" workflow in n8n. It covers all nodes, their roles, configurations, and potential failure points to ensure robust integration and customization.