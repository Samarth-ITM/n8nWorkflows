Generate AI Avatars with Replicate's babysea/babyavatar Model

https://n8nworkflows.xyz/workflows/generate-ai-avatars-with-replicate-s-babysea-babyavatar-model-6877


# Generate AI Avatars with Replicate's babysea/babyavatar Model

### 1. Workflow Overview

This workflow, titled **"Babysea Babyavatar AI Generator"**, automates the generation of AI avatars using the **babysea/babyavatar** model hosted on Replicate. It is designed for users who want to programmatically create avatar images by submitting a prediction request to the Replicate API, then polling for the completion of this prediction, and finally processing the resulting output.

The workflow is logically divided into the following blocks:

- **1.1 Input Initialization**: Manual trigger and API key setup.
- **1.2 Prediction Request Creation**: Sending the initial prediction request to Replicate with model parameters.
- **1.3 Prediction ID Extraction and Polling Setup**: Extracting the prediction ID and preparing to poll for the prediction status.
- **1.4 Polling Loop**: Waiting and repeatedly checking the status of the prediction until completion.
- **1.5 Result Processing**: Handling the successful prediction output for further use or integration.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Initialization

- **Overview:** This block waits for manual execution to start the workflow and sets the Replicate API key as a workflow variable for authentication in subsequent requests.
- **Nodes Involved:**  
  - On clicking 'execute' (Manual Trigger)  
  - Set API Key (Set node)

- **Node Details:**

  - **On clicking 'execute'**  
    - Type: Manual Trigger  
    - Role: Entry point to manually start the workflow execution.  
    - Configuration: No parameters; triggers workflow on user interaction.  
    - Input: None  
    - Output: Triggers the next node, Set API Key  
    - Failure modes: None expected here since it's manual.

  - **Set API Key**  
    - Type: Set  
    - Role: Stores the Replicate API key as a variable for use in HTTP requests.  
    - Configuration: Assigns a string variable `replicate_api_key` with the placeholder `"YOUR_REPLICATE_API_KEY"` that must be replaced with the user's actual key.  
    - Input: Trigger from Manual Trigger  
    - Output: Passes data including the API key to Create Prediction node  
    - Failure modes: Missing or invalid API key will cause authentication failures downstream.

---

#### 1.2 Prediction Request Creation

- **Overview:** Sends a POST request to Replicate’s API to create a new prediction task for the babysea/babyavatar model, specifying generation parameters.
- **Nodes Involved:**  
  - Create Prediction (HTTP Request)

- **Node Details:**

  - **Create Prediction**  
    - Type: HTTP Request  
    - Role: Initiates the AI avatar generation by calling Replicate’s prediction API.  
    - Configuration:  
      - URL: `https://api.replicate.com/v1/predictions`  
      - Method: POST  
      - Authentication: HTTP Header with Bearer token (`replicate_api_key`)  
      - Headers: `Authorization: Bearer <api_key>`, `Content-Type: application/json`  
      - Body: JSON specifying model version and input parameters:  
        - `version`: fixed model version hash `"f879b89acc5b931e4d180de5ef0e89782a4937cd4ae1d59be9a502a01a7b8c8c"`  
        - `input`: seed=1, width=1024, height=1024, lora_scale=0.6, num_outputs=1  
      - Timeout: 60 seconds  
    - Input: Receives API key from Set API Key node  
    - Output: JSON response containing prediction metadata including a prediction ID  
    - Failure modes:  
      - Network timeouts  
      - Authentication errors (invalid token)  
      - API rate limits or quota exceeded  
      - Invalid request parameters

---

#### 1.3 Prediction ID Extraction and Polling Setup

- **Overview:** Extracts the unique prediction ID from the initial response to enable polling the prediction status.
- **Nodes Involved:**  
  - Extract Prediction ID (Code)

- **Node Details:**

  - **Extract Prediction ID**  
    - Type: Code (JavaScript)  
    - Role: Parses the HTTP response to extract the prediction ID and initial status, formats a polling URL.  
    - Configuration:  
      - Reads prediction ID and status from input JSON  
      - Constructs `predictionUrl` for subsequent polling calls  
    - Input: Response from Create Prediction node  
    - Output: JSON object with `predictionId`, `status`, and `predictionUrl`  
    - Failure modes:  
      - If API response is malformed or missing fields, code may throw errors or produce invalid URLs.

---

#### 1.4 Polling Loop

- **Overview:** Implements a polling mechanism with a wait period to repeatedly check if the prediction has completed.
- **Nodes Involved:**  
  - Wait (Wait node)  
  - Check Prediction Status (HTTP Request)  
  - Check If Complete (If node)

- **Node Details:**

  - **Wait**  
    - Type: Wait  
    - Role: Pauses workflow execution for 2 seconds before next status check to avoid excessive requests.  
    - Configuration: Wait for 2 seconds  
    - Input: From Extract Prediction ID (initial) or from Check If Complete (when prediction incomplete)  
    - Output: Triggers Check Prediction Status node  
    - Failure modes: Minimal; long-running workflows may be interrupted if limits exceeded.

  - **Check Prediction Status**  
    - Type: HTTP Request  
    - Role: Queries the prediction status URL to check current state of the avatar generation task.  
    - Configuration:  
      - URL dynamically set to `predictionUrl` from previous node  
      - Headers: Authentication with Replicate API key as Bearer token  
      - Method: GET (default)  
    - Input: From Wait node  
    - Output: JSON with current prediction status  
    - Failure modes:  
      - API errors or timeouts  
      - Authentication failures  
      - Unexpected response format

  - **Check If Complete**  
    - Type: If  
    - Role: Decision node to test if prediction status equals "succeeded".  
    - Configuration: Compares `status` field in JSON to `"succeeded"` boolean condition  
    - Input: From Check Prediction Status  
    - Output:  
      - True branch: Process Result node (prediction complete)  
      - False branch: Wait node (continue polling)  
    - Failure modes: None, but if prediction fails or errors, workflow may loop indefinitely or need additional failure handling.

---

#### 1.5 Result Processing

- **Overview:** Processes the successful prediction output to extract relevant data such as output URLs and metadata for further use.
- **Nodes Involved:**  
  - Process Result (Code)

- **Node Details:**

  - **Process Result**  
    - Type: Code (JavaScript)  
    - Role: Extracts and formats the final prediction output details for downstream consumption or storage.  
    - Configuration:  
      - Reads prediction JSON including status, output URLs, metrics, timestamps  
      - Packages these into a simplified JSON object with keys: `status`, `output`, `metrics`, `created_at`, `completed_at`, `model`, `other_url`  
      - Hardcodes model name as `"babysea/babyavatar"`  
    - Input: From Check If Complete node (true branch)  
    - Output: Structured JSON data representing the prediction result  
    - Failure modes: Malformed or incomplete API response data could cause issues.

---

#### Additional Node: Sticky Note

- **Sticky Note**  
  - Provides documentation inside the workflow editor describing the workflow purpose, setup instructions, and model details.  
  - Contains instructions about adding API key, configuring inputs, and running the workflow.  
  - No functional impact on execution.

---

### 3. Summary Table

| Node Name             | Node Type         | Functional Role                         | Input Node(s)            | Output Node(s)           | Sticky Note                                                                                                    |
|-----------------------|-------------------|---------------------------------------|--------------------------|--------------------------|---------------------------------------------------------------------------------------------------------------|
| On clicking 'execute'  | Manual Trigger    | Workflow start trigger                 | None                     | Set API Key              | ## Babysea Babyavatar AI Generator<br>This workflow uses the **babysea/babyavatar** model from Replicate...   |
| Set API Key           | Set               | Sets API key variable                  | On clicking 'execute'     | Create Prediction        | ## Babysea Babyavatar AI Generator<br>This workflow uses the **babysea/babyavatar** model from Replicate...   |
| Create Prediction     | HTTP Request      | Sends prediction creation request     | Set API Key              | Extract Prediction ID    |                                                                                                               |
| Extract Prediction ID | Code              | Extracts prediction ID and URL         | Create Prediction        | Wait                     |                                                                                                               |
| Wait                  | Wait              | Pauses before polling                  | Extract Prediction ID, Check If Complete (false branch) | Check Prediction Status |                                                                                                               |
| Check Prediction Status | HTTP Request    | Polls prediction status                | Wait                     | Check If Complete        |                                                                                                               |
| Check If Complete     | If                | Checks if prediction succeeded         | Check Prediction Status  | Process Result (true), Wait (false) |                                                                                                               |
| Process Result        | Code              | Processes completed prediction output | Check If Complete (true) | None                     |                                                                                                               |
| Sticky Note           | Sticky Note       | Workflow documentation                 | None                     | None                     | ## Babysea Babyavatar AI Generator<br>This workflow uses the **babysea/babyavatar** model from Replicate...   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: `On clicking 'execute'`  
   - Type: Manual Trigger  
   - No parameters needed.

2. **Create Set Node for API Key**  
   - Name: `Set API Key`  
   - Type: Set  
   - Add a string field called `replicate_api_key` with the value: `"YOUR_REPLICATE_API_KEY"` (replace with your actual Replicate API key).  
   - Connect `On clicking 'execute'` output to this node input.

3. **Create HTTP Request Node to Create Prediction**  
   - Name: `Create Prediction`  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.replicate.com/v1/predictions`  
   - Authentication: Generic HTTP Header Authentication  
     - Header Name: `Authorization`  
     - Header Value: `Bearer {{$node["Set API Key"].json["replicate_api_key"]}}` (use expression)  
   - Additional Header: `Content-Type: application/json`  
   - Body Content (raw JSON):  
     ```json
     {
       "version": "f879b89acc5b931e4d180de5ef0e89782a4937cd4ae1d59be9a502a01a7b8c8c",
       "input": {
         "seed": 1,
         "width": 1024,
         "height": 1024,
         "lora_scale": 0.6,
         "num_outputs": 1
       }
     }
     ```  
   - Timeout: 60 seconds  
   - Connect `Set API Key` output to this node input.

4. **Create Code Node to Extract Prediction ID**  
   - Name: `Extract Prediction ID`  
   - Type: Code  
   - Mode: Run Once for Each Item  
   - Code (JavaScript):  
     ```javascript
     const prediction = $input.item.json;
     const predictionId = prediction.id;
     const initialStatus = prediction.status;

     return {
       predictionId: predictionId,
       status: initialStatus,
       predictionUrl: `https://api.replicate.com/v1/predictions/${predictionId}`
     };
     ```  
   - Connect `Create Prediction` output to this node input.

5. **Create Wait Node**  
   - Name: `Wait`  
   - Type: Wait  
   - Parameters: Wait for 2 seconds  
   - Connect `Extract Prediction ID` output to this node input.

6. **Create HTTP Request Node to Check Prediction Status**  
   - Name: `Check Prediction Status`  
   - Type: HTTP Request  
   - Method: GET (default)  
   - URL: Use expression `{{$json["predictionUrl"]}}` to dynamically call the prediction status URL.  
   - Authentication: Generic HTTP Header Authentication  
     - Header Name: `Authorization`  
     - Header Value: `Bearer {{$node["Set API Key"].json["replicate_api_key"]}}`  
   - Connect `Wait` output to this node input.

7. **Create If Node to Check Completion**  
   - Name: `Check If Complete`  
   - Type: If  
   - Condition: Boolean  
     - Value 1: `{{$json["status"]}}`  
     - Operation: equal  
     - Value 2: `"succeeded"`  
   - Connect `Check Prediction Status` output to this node input.

8. **Create Code Node to Process Result**  
   - Name: `Process Result`  
   - Type: Code  
   - Mode: Run Once for Each Item  
   - Code:  
     ```javascript
     const result = $input.item.json;

     return {
       status: result.status,
       output: result.output,
       metrics: result.metrics,
       created_at: result.created_at,
       completed_at: result.completed_at,
       model: 'babysea/babyavatar',
       other_url: result.output
     };
     ```  
   - Connect `Check If Complete` true output to this node input.

9. **Connect `Check If Complete` false output back to `Wait` node**  
   - This creates the polling loop until prediction completes.

10. **Add Sticky Note (optional)**  
    - Place a Sticky Note with the workflow description and setup instructions for clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                               | Context or Link                                              |
|------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------|
| This workflow uses the **babysea/babyavatar** model from Replicate for AI avatar generation.                                                              | Workflow purpose description                                 |
| Setup requires user to insert a valid Replicate API key in the `Set API Key` node.                                                                          | Setup instructions                                           |
| The model version hash is fixed and should be kept as is for consistent results.                                                                           | Model configuration detail                                   |
| Polling interval is set to 2 seconds to balance between responsiveness and API rate limiting.                                                              | Polling strategy                                            |
| The workflow expects the prediction to eventually reach a `"succeeded"` status; no explicit failure handling or timeout exit is implemented.               | Potential improvement area                                   |
| Replicate API documentation: https://replicate.com/docs                                                                                                   | Official API reference                                       |

---

This documentation fully captures the workflow structure, node configurations, logic flow, and how to reproduce it manually in n8n. It also highlights potential failure points and operational considerations for users or automation agents.