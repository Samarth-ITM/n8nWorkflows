Content Summarizer via Webhook (ApyHub)

https://n8nworkflows.xyz/workflows/content-summarizer-via-webhook--apyhub--4603


# Content Summarizer via Webhook (ApyHub)

---
### 1. Workflow Overview

This workflow, titled **Content Summarizer via Webhook (ApyHub)**, provides an automated content summarization service triggered by incoming HTTP POST requests. It is designed for use cases where external applications or users send text content to be summarized, optionally specifying the desired summary length. The workflow interacts with the ApyHub summarization API to process the content asynchronously, polling for the job completion before returning the summarized text.

The workflow is logically divided into four blocks:

- **1.1 Input Reception:** Receives and validates incoming requests via a webhook.
- **1.2 Summarization Job Initialization:** Sends the content to ApyHub to start a summarization job.
- **1.3 Result Retrieval:** Polls the ApyHub API for job status and fetches the summary when ready.
- **1.4 Response Delivery:** Returns the summarized content to the original requestor.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block listens for incoming HTTP POST requests containing the text to summarize and optional parameters. It extracts required data for downstream processing.

- **Nodes Involved:**  
  - Receive Content Webhook  
  - Note for Webhook Trigger

- **Node Details:**

  - **Receive Content Webhook**  
    - Type: Webhook  
    - Role: Entry point that listens for POST requests at path `/summarize-content`.  
    - Configuration:  
      - HTTP Method: POST  
      - Response Mode: `responseNode` (response handled by a downstream node)  
      - Expects JSON body with:  
        - `content` (string, required): Text to summarize  
        - `summary_length` (string, optional): summary length indicator (e.g., 'short', 'medium', 'long')  
      - Expects header: `apy-token` (authentication for ApyHub API)  
    - Inputs: External HTTP request  
    - Outputs: JSON object containing body and headers forwarded to next node  
    - Edge Cases: Missing or malformed JSON body, missing `content`, missing or invalid `apy-token` in headers, unsupported HTTP method, authentication failures downstream  
    - Version: n8n webhook node version 2  

  - **Note for Webhook Trigger**  
    - Type: Sticky Note  
    - Purpose: Documentation attached to the webhook node describing expected input structure and authentication header requirements.  
    - Content: Explains input expectations and header requirement for 'apy-token'.

#### 1.2 Summarization Job Initialization

- **Overview:**  
  Sends a POST request to ApyHub’s summarization API to start processing the provided content asynchronously.

- **Nodes Involved:**  
  - Start Summarization Job  
  - Note for Start Summarization Job

- **Node Details:**

  - **Start Summarization Job**  
    - Type: HTTP Request  
    - Role: Initiates summarization by sending content and parameters to ApyHub API.  
    - Configuration:  
      - Method: POST  
      - URL: `https://api.apyhub.com/sharpapi/api/v1/content/summarize`  
      - Body (JSON):  
        ```json
        {
          "content": "{{ $json.body.content }}",
          "summary_length": "{{ $json.body.summary_length || 'medium' }}"
        }
        ```  
        This uses expressions to populate values from webhook JSON body; defaults `summary_length` to 'medium' if not provided.  
      - Headers (JSON):  
        ```json
        {
          "Content-Type": "application/json",
          "apy-token": "{{ $json.headers['apy-token'] }}"
        }
        ```  
        The `apy-token` is extracted from the incoming request headers for authentication.  
      - Sends both headers and JSON body  
    - Inputs: Output of webhook node containing body and headers  
    - Outputs: JSON response including a `job_id` to track summarization status  
    - Edge Cases: Invalid or missing `apy-token`, malformed content, API rate limits, API downtime, network errors, unexpected response format  
    - Version: n8n HTTP Request node version 4.2  

  - **Note for Start Summarization Job**  
    - Type: Sticky Note  
    - Purpose: Documents the POST request purpose, parameters sent, and token handling for this node.

#### 1.3 Result Retrieval

- **Overview:**  
  Polls ApyHub’s API to check the status of the summarization job using the `job_id`. Retrieves the summarized text when the job is complete.

- **Nodes Involved:**  
  - Get Summarization Result  
  - Note for Get Summarization Result

- **Node Details:**

  - **Get Summarization Result**  
    - Type: HTTP Request  
    - Role: Requests the job status and, when finished, obtains the summarized content.  
    - Configuration:  
      - Method: GET (default)  
      - URL:  
        ```  
        https://api.apyhub.com/sharpapi/api/v1/content/summarize/job/status/{{ $json.job_id }}  
        ```  
        Uses expression to dynamically insert the `job_id` from the previous node's output.  
      - Headers (JSON):  
        ```json
        {
          "Content-Type": "application/json",
          "apy-token": "{{ $('Receive Content Webhook').item.json.headers['apy-token'] }}"
        }
        ```  
        Extracts the authentication token again from the webhook headers node to maintain consistency.  
      - Sends headers  
    - Inputs: Output of "Start Summarization Job" node (which contains `job_id`)  
    - Outputs: JSON containing job status and summarized content once ready  
    - Edge Cases: Job may not be finished yet (requires polling or delay logic outside this workflow), invalid job ID, token issues, API errors, timeouts  
    - Version: n8n HTTP Request node version 4.2  

  - **Note for Get Summarization Result**  
    - Type: Sticky Note  
    - Purpose: Explains the polling mechanism and importance of waiting for job completion before fetching summary.

#### 1.4 Response Delivery

- **Overview:**  
  Sends the final summarized text back to the caller of the webhook, completing the request-response cycle.

- **Nodes Involved:**  
  - Respond with Summarized Content  
  - Note for Webhook Response

- **Node Details:**

  - **Respond with Summarized Content**  
    - Type: Respond to Webhook  
    - Role: Returns the processed summary data as HTTP response to the original webhook caller.  
    - Configuration:  
      - Response data: Sends all incoming items (i.e., the output from "Get Summarization Result" node) as the response payload  
      - No additional transformations or headers configured  
    - Inputs: Output from "Get Summarization Result" node  
    - Outputs: HTTP response to external client  
    - Edge Cases: If the summarization result is missing or incomplete, the response may be empty or erroneous; ensure prior nodes handle errors and retries appropriately  
    - Version: n8n Respond to Webhook node version 1.2  

  - **Note for Webhook Response**  
    - Type: Sticky Note  
    - Purpose: Suggests inserting further nodes here if post-processing or storage of summarized content is required before responding.

---

### 3. Summary Table

| Node Name                  | Node Type              | Functional Role                         | Input Node(s)                | Output Node(s)                 | Sticky Note                                                                                                       |
|----------------------------|------------------------|---------------------------------------|-----------------------------|-------------------------------|------------------------------------------------------------------------------------------------------------------|
| Receive Content Webhook     | Webhook                | Entry point; receives POST request    | —                           | Start Summarization Job        | This node listens for incoming POST requests. It expects a JSON body with a 'content' property (the text to summarize) and a 'summary_length' (optional, e.g., 'short', 'medium', 'long'). Your 'apy-token' must be provided in the request headers. |
| Note for Webhook Trigger    | Sticky Note            | Documentation for webhook input       | —                           | —                             | See above                                                                                                        |
| Start Summarization Job     | HTTP Request           | Starts summarization job on ApyHub    | Receive Content Webhook      | Get Summarization Result       | This node sends a POST request to ApyHub's summarization API. It passes the 'content' from the webhook body and your 'apy-token' from the headers. It initiates the summarization job and returns a 'job_id'. |
| Note for Start Summarization Job | Sticky Note        | Documentation for start job node      | —                           | —                             | See above                                                                                                        |
| Get Summarization Result    | HTTP Request           | Polls for summary job completion      | Start Summarization Job      | Respond with Summarized Content | This node polls ApyHub's API using the 'job_id' to check the status of the summarization. Once the status is 'finished', it retrieves the summarized content. This ensures you get the result only when processing is complete. |
| Note for Get Summarization Result | Sticky Note        | Documentation for result retrieval    | —                           | —                             | See above                                                                                                        |
| Respond with Summarized Content | Respond to Webhook  | Sends summary back as webhook response | Get Summarization Result     | —                             | This node sends the final summarized text back to the original caller of the webhook. You can insert other nodes before this to store, share, or further process the summarized content. |
| Note for Webhook Response   | Sticky Note            | Documentation for webhook response    | —                           | —                             | See above                                                                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node: "Receive Content Webhook"**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `summarize-content`  
   - Response Mode: `responseNode` (respond via downstream node)  
   - No authentication configured here (assumes client provides `apy-token` header)  

2. **Add Sticky Note: "Note for Webhook Trigger"**  
   - Attach near webhook node  
   - Content: Describe expected JSON body (`content` and optional `summary_length`) and header `apy-token` requirement  

3. **Create HTTP Request Node: "Start Summarization Job"**  
   - Method: POST  
   - URL: `https://api.apyhub.com/sharpapi/api/v1/content/summarize`  
   - Body Content-Type: JSON  
   - Body JSON:  
     ```json
     {
       "content": "{{ $json.body.content }}",
       "summary_length": "{{ $json.body.summary_length || 'medium' }}"
     }
     ```  
   - Headers JSON:  
     ```json
     {
       "Content-Type": "application/json",
       "apy-token": "{{ $json.headers['apy-token'] }}"
     }
     ```  
   - Connect input from "Receive Content Webhook" node  

4. **Add Sticky Note: "Note for Start Summarization Job"**  
   - Attach near "Start Summarization Job" node  
   - Content: Explain job initiation and token usage  

5. **Create HTTP Request Node: "Get Summarization Result"**  
   - Method: GET (default)  
   - URL:  
     ```
     https://api.apyhub.com/sharpapi/api/v1/content/summarize/job/status/{{ $json.job_id }}
     ```  
   - Headers JSON:  
     ```json
     {
       "Content-Type": "application/json",
       "apy-token": "{{ $('Receive Content Webhook').item.json.headers['apy-token'] }}"
     }
     ```  
   - Connect input from "Start Summarization Job" node  

6. **Add Sticky Note: "Note for Get Summarization Result"**  
   - Attach near "Get Summarization Result" node  
   - Content: Describe polling and waiting for job completion  

7. **Create Respond to Webhook Node: "Respond with Summarized Content"**  
   - Respond with: All incoming items (output of previous node)  
   - Connect input from "Get Summarization Result" node  

8. **Add Sticky Note: "Note for Webhook Response"**  
   - Attach near "Respond with Summarized Content" node  
   - Content: Indicate this node sends final summary back; suggest optional post-processing  

9. **Verify Connections:**  
   - Connect `Receive Content Webhook` → `Start Summarization Job` → `Get Summarization Result` → `Respond with Summarized Content`  

10. **Credentials:**  
    - None explicitly required inside the workflow; the `apy-token` is passed dynamically from incoming request headers.  
    - Ensure clients send valid `apy-token` in request headers for authentication with ApyHub API.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                 |
|---------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------|
| The workflow expects the client to provide the `apy-token` in the HTTP header for authentication with ApyHub. | Authentication mechanism for API calls                         |
| You can extend the workflow by adding nodes before the response node to store or process summaries further.   | Suggested extension point after "Get Summarization Result"    |
| Sticky notes provide inline documentation useful for maintenance and onboarding new users.                     | Workflow documentation practice                                |
| More information about ApyHub API can be found on their official site (not linked here due to scope).          | External API reference                                         |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.