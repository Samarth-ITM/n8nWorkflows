🌐 Firecrawl Website Content Extractor

https://n8nworkflows.xyz/workflows/---firecrawl-website-content-extractor-4764


# 🌐 Firecrawl Website Content Extractor

### 1. Workflow Overview

The **🌐 Firecrawl Website Content Extractor** workflow automates the process of extracting content from a web resource using HTTP requests and timed waits, handling asynchronous response delays and conditional logic for retrying or processing results. It is designed for scenarios where an initial extraction request triggers content generation or retrieval that is not immediately available, requiring polling for completion before final data handling.

The workflow’s logic is structured into the following functional blocks:

- **1.1 Input Trigger:** Manual initiation of the workflow.
- **1.2 Initial Extraction Request:** Sending an HTTP request to start content extraction.
- **1.3 Wait and Polling:** Waiting periods to allow the extraction process to complete asynchronously.
- **1.4 Retrieval of Results:** Sending HTTP requests to fetch the extraction results.
- **1.5 Conditional Logic:** Checking if the results are ready or if further waiting/polling is needed.
- **1.6 Final Data Handling:** Setting or editing the extracted fields for downstream use or output.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Trigger

- **Overview:** This block initiates the workflow manually, allowing users to trigger the extraction process on demand.
- **Nodes Involved:** `When clicking ‘Test workflow’`
- **Node Details:**

  - **Node Name:** When clicking ‘Test workflow’
  - **Type:** Manual Trigger node
  - **Role:** Entry point for manual execution.
  - **Configuration:** Default manual trigger with no parameters; activated via n8n UI.
  - **Input/Output:** No input; outputs to the `Extract` node.
  - **Version Requirements:** None specific.
  - **Edge Cases:** None inherent; user must trigger manually.

#### 2.2 Initial Extraction Request

- **Overview:** This block sends an HTTP request to start the content extraction process on the target website or API.
- **Nodes Involved:** `Extract`
- **Node Details:**

  - **Node Name:** Extract
  - **Type:** HTTP Request node
  - **Role:** Initiates extraction by sending a request to the external service.
  - **Configuration:** URL, method, headers, and body parameters are expected to be set depending on the target extraction API (not specified in JSON).
  - **Key Expressions/Variables:** None explicitly provided; likely configured with the extraction URL and parameters.
  - **Input:** Receives trigger from manual node.
  - **Output:** Passes to `30 Secs` wait node.
  - **Edge Cases:** HTTP errors (timeouts, 4xx, 5xx), invalid URLs, or network failures.

#### 2.3 Wait and Polling

- **Overview:** This block manages wait times between requests to allow asynchronous extraction processes to complete before polling for results.
- **Nodes Involved:** `30 Secs`, `10 Seconds`
- **Node Details:**

  - **Node Name:** 30 Secs
  - **Type:** Wait node
  - **Role:** Pauses workflow for 30 seconds after the initial extraction request.
  - **Configuration:** Fixed 30-second wait.
  - **Input:** From `Extract`.
  - **Output:** To `Get Results`.
  - **Edge Cases:** None significant; could delay workflow unnecessarily if extraction completes earlier.

  - **Node Name:** 10 Seconds
  - **Type:** Wait node
  - **Role:** Additional wait when results are not ready; short 10-second pause before retry.
  - **Configuration:** Fixed 10-second wait.
  - **Input:** Conditional branch from `If` node when retry is needed.
  - **Output:** To `Get Results`.
  - **Edge Cases:** Same as 30 Secs; wait may accumulate if repeated.

#### 2.4 Retrieval of Results

- **Overview:** Sends HTTP requests to fetch the results of the extraction after waiting periods.
- **Nodes Involved:** `Get Results`
- **Node Details:**

  - **Node Name:** Get Results
  - **Type:** HTTP Request node
  - **Role:** Polls the extraction service for completed content.
  - **Configuration:** Configured with URL and parameters to retrieve extraction results (details not provided).
  - **Input:** From both wait nodes (`30 Secs` and `10 Seconds`).
  - **Output:** To `If` node for result evaluation.
  - **Edge Cases:** HTTP errors, incomplete responses, empty payloads, or malformed data.

#### 2.5 Conditional Logic

- **Overview:** Evaluates whether the extraction results are ready and directs workflow accordingly.
- **Nodes Involved:** `If`
- **Node Details:**

  - **Node Name:** If
  - **Type:** If condition node
  - **Role:** Checks if results are complete or if further waiting and polling are needed.
  - **Configuration:** Condition logic not explicitly provided but typically might check for HTTP status, presence of data, or a flag in the response.
  - **Input:** From `Get Results`.
  - **Output:** Two branches:
    - True branch: to `Edit Fields` (processing results)
    - False branch: to `10 Seconds` (retry polling)
  - **Version Requirements:** Supports `continueErrorOutput` to avoid workflow stoppage on errors.
  - **Edge Cases:** Expression failures if response data is malformed or missing; potential infinite loops if exit condition not met.

#### 2.6 Final Data Handling

- **Overview:** Processes or formats the final extracted data for output or further use.
- **Nodes Involved:** `Edit Fields`
- **Node Details:**

  - **Node Name:** Edit Fields
  - **Type:** Set node
  - **Role:** Modifies or sets specific fields in the data structure before output.
  - **Configuration:** Parameters not detailed; likely sets variables or transforms extracted content.
  - **Input:** From `If` node’s true branch.
  - **Output:** None (end of workflow).
  - **Edge Cases:** Expression errors if fields referenced do not exist.

---

### 3. Summary Table

| Node Name                | Node Type          | Functional Role               | Input Node(s)               | Output Node(s)           | Sticky Note                          |
|--------------------------|--------------------|------------------------------|-----------------------------|--------------------------|------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger     | Workflow initiation          | —                           | Extract                  |                                    |
| Extract                  | HTTP Request       | Start extraction request      | When clicking ‘Test workflow’ | 30 Secs                  |                                    |
| 30 Secs                  | Wait               | Initial wait after request    | Extract                     | Get Results              |                                    |
| Get Results              | HTTP Request       | Polling for extraction results| 30 Secs, 10 Seconds          | If                       |                                    |
| If                       | If                 | Check if results are ready    | Get Results                 | Edit Fields, 10 Seconds  |                                    |
| 10 Seconds               | Wait               | Retry wait before re-polling  | If (false branch)           | Get Results              |                                    |
| Edit Fields              | Set                | Final data processing         | If (true branch)            | —                        |                                    |
| Sticky Note              | Sticky Note        | Visual annotation             | —                           | —                        | (Empty content)                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**
   - Name: `When clicking ‘Test workflow’`
   - Type: Manual Trigger
   - No additional parameters.

2. **Create HTTP Request Node for Initial Extraction:**
   - Name: `Extract`
   - Type: HTTP Request
   - Configure URL and method to initiate content extraction on target website/API.
   - Connect output of `When clicking ‘Test workflow’` to input of `Extract`.

3. **Create Wait Node for Initial Delay:**
   - Name: `30 Secs`
   - Type: Wait
   - Set wait duration to 30 seconds.
   - Connect output of `Extract` to input of `30 Secs`.

4. **Create HTTP Request Node to Get Results:**
   - Name: `Get Results`
   - Type: HTTP Request
   - Configure URL and method to poll retrieval of extraction results.
   - Connect output of `30 Secs` to input of `Get Results`.

5. **Create If Node to Check Results:**
   - Name: `If`
   - Type: If Condition
   - Configure condition to evaluate if extraction results are complete or valid.
     - For example, check if response contains expected data or status code.
   - Connect output of `Get Results` to input of `If`.

6. **Create Wait Node for Retry Delay:**
   - Name: `10 Seconds`
   - Type: Wait
   - Set wait duration to 10 seconds.
   - Connect `If` node’s false output branch to `10 Seconds`.

7. **Connect Retry Wait to Get Results:**
   - Connect output of `10 Seconds` back to input of `Get Results` to poll again after wait.

8. **Create Set Node for Final Data Handling:**
   - Name: `Edit Fields`
   - Type: Set
   - Configure to set or modify fields based on the final extracted data.
   - Connect `If` node’s true output branch to `Edit Fields`.

9. **No Credentials required explicitly unless HTTP requests require authentication:**
   - If the target API requires authentication, configure credentials accordingly (e.g., API key, OAuth2).

10. **Verify Workflow Operation:**
    - Manually trigger `When clicking ‘Test workflow’`.
    - Monitor progress through waits and polling until results are processed.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| This workflow demonstrates a polling pattern with waits to handle asynchronous HTTP extraction processes. | General workflow design |
| Use of `continueErrorOutput` on the If node allows the workflow to continue gracefully upon errors, useful for unstable APIs. | If node configuration |
| n8n version compatibility: HTTP Request nodes use version 4.2, which supports advanced HTTP features; Wait nodes use version 1.1. | Version compatibility |
| The empty sticky note present in the workflow does not contain comments or documentation. | Visual annotation |

---

**Disclaimer:** The provided content is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled are legal and public.