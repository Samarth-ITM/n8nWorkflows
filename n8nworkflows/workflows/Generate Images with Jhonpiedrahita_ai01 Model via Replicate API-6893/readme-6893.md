Generate Images with Jhonpiedrahita_ai01 Model via Replicate API

https://n8nworkflows.xyz/workflows/generate-images-with-jhonpiedrahita_ai01-model-via-replicate-api-6893


# Generate Images with Jhonpiedrahita_ai01 Model via Replicate API

### 1. Workflow Overview

This workflow enables users to generate images (or other content) using the **jhonp4/jhonpiedrahita_ai01** AI model via the Replicate API. It is designed for interactive execution, where a user manually triggers the workflow and the system handles asynchronous prediction creation, status polling, and result processing.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Initialization:** Manual trigger and setting of the Replicate API key.
- **1.2 Prediction Creation:** Sending a prediction request to the Replicate API with user-configured parameters.
- **1.3 Prediction Status Polling:** Periodic checking of the prediction status until completion.
- **1.4 Result Processing:** Extracting and formatting the final prediction result for further use or output.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Initialization

- **Overview:**  
  This initial block waits for manual user input to start the workflow and then sets the API key necessary to authenticate requests to Replicate.

- **Nodes Involved:**  
  - On clicking 'execute'  
  - Set API Key  

- **Node Details:**

  - **On clicking 'execute'**  
    - Type: Manual Trigger  
    - Role: Entry point for manual execution  
    - Configuration: Default manual trigger with no parameters  
    - Inputs: None  
    - Outputs: Connects to Set API Key node  
    - Edge cases: User must execute manually; no automatic start.

  - **Set API Key**  
    - Type: Set node  
    - Role: Stores the Replicate API key as a workflow variable  
    - Configuration: Assigns a string named `replicate_api_key` with placeholder `YOUR_REPLICATE_API_KEY` (to be replaced by the user)  
    - Inputs: From manual trigger node  
    - Outputs: Connects to Create Prediction node  
    - Edge cases: Missing or invalid API key will cause authentication failures in subsequent HTTP requests.

---

#### 1.2 Prediction Creation

- **Overview:**  
  This block constructs and sends an HTTP POST request to the Replicate API to create a new prediction job with specific input parameters such as prompt, seed, width, height, and lora_scale.

- **Nodes Involved:**  
  - Create Prediction  
  - Extract Prediction ID  

- **Node Details:**

  - **Create Prediction**  
    - Type: HTTP Request  
    - Role: Initiates a prediction job in Replicate API  
    - Configuration:  
      - URL: `https://api.replicate.com/v1/predictions`  
      - Method: POST  
      - Headers: Authorization Bearer token using the stored API key, Content-Type application/json  
      - Body: JSON including model version id and input parameters (prompt, seed, width, height, lora_scale)  
      - Timeout: 60 seconds  
    - Input: Receives API key from Set API Key node  
    - Output: JSON response with prediction ID and status  
    - Edge cases: Network failures, invalid parameters, authorization errors, API rate limits.  
    - Expressions: Uses expression to set Authorization header as `Bearer {{replicate_api_key}}`.

  - **Extract Prediction ID**  
    - Type: Code  
    - Role: Parses the response to extract the prediction ID, initial status, and constructs a URL for status polling  
    - Configuration: JavaScript code extracting `id` and `status` from the HTTP response, returns `predictionId`, `status`, and `predictionUrl`  
    - Input: Output from Create Prediction  
    - Output: Object with prediction tracking details  
    - Edge cases: Unexpected response structure, undefined fields.

---

#### 1.3 Prediction Status Polling

- **Overview:**  
  This block periodically waits and checks the prediction status by querying the Replicate API until the status is `succeeded`.

- **Nodes Involved:**  
  - Wait  
  - Check Prediction Status  
  - Check If Complete  

- **Node Details:**

  - **Wait**  
    - Type: Wait node  
    - Role: Pauses the workflow for 2 seconds between status checks  
    - Configuration: Wait for 2 seconds  
    - Input: Receives from Extract Prediction ID or Check If Complete node if prediction not ready  
    - Output: Connects to Check Prediction Status node  
    - Edge cases: Excessive waiting time or failures in upstream nodes can stall workflow.

  - **Check Prediction Status**  
    - Type: HTTP Request  
    - Role: Requests current prediction status from Replicate API  
    - Configuration:  
      - URL: Dynamic, from `predictionUrl` extracted previously  
      - Method: GET (default)  
      - Headers: Authorization Bearer token  
    - Input: From Wait node  
    - Output: Connects to Check If Complete node  
    - Edge cases: Network issues, token expiration, API errors, malformed URLs.

  - **Check If Complete**  
    - Type: If node  
    - Role: Checks if the prediction status equals `succeeded`  
    - Configuration: Boolean condition comparing `$json.status` to "succeeded"  
    - Input: From Check Prediction Status  
    - Output:  
      - True branch: Connects to Process Result node  
      - False branch: Loops back to Wait node to continue polling  
    - Edge cases: If status never turns `succeeded`, workflow loops indefinitely; no timeout or failure branch implemented.

---

#### 1.4 Result Processing

- **Overview:**  
  Once the prediction is complete, this block extracts relevant result data and formats it for downstream consumption or storage.

- **Nodes Involved:**  
  - Process Result  

- **Node Details:**

  - **Process Result**  
    - Type: Code node  
    - Role: Extracts and formats prediction output details  
    - Configuration: JavaScript code that returns an object containing:  
      - status  
      - output (likely the generated image URL or content)  
      - metrics (performance or prediction stats)  
      - timestamps (created_at, completed_at)  
      - model identifier string  
      - other_url duplicating output field  
    - Input: From Check If Complete node (true branch)  
    - Output: Processed prediction result object  
    - Edge cases: Missing fields in response, unexpected data formats.

---

#### Additional Node

- **Sticky Note**  
  - Provides documentation and instructions about the workflow usage, setup steps, and model information.  
  - Content includes:  
    - Workflow title and model used  
    - Setup instructions (add API key, configure inputs, run)  
    - Model details (type, provider, required fields)  
  - Positioned visually near start nodes for user reference.

---

### 3. Summary Table

| Node Name              | Node Type        | Functional Role                      | Input Node(s)            | Output Node(s)           | Sticky Note                                                                                     |
|------------------------|------------------|------------------------------------|--------------------------|--------------------------|------------------------------------------------------------------------------------------------|
| On clicking 'execute'   | Manual Trigger   | Workflow entry point                | None                     | Set API Key              | ## Jhonp4 Jhonpiedrahita_ai01 AI Generator<br>This workflow uses the **jhonp4/jhonpiedrahita_ai01** model from Replicate to generate other content.<br>Setup steps included in sticky note. |
| Set API Key            | Set              | Store Replicate API key             | On clicking 'execute'    | Create Prediction        | ## Jhonp4 Jhonpiedrahita_ai01 AI Generator (see above)                                         |
| Create Prediction      | HTTP Request     | Create prediction on Replicate API | Set API Key              | Extract Prediction ID    | ## Jhonp4 Jhonpiedrahita_ai01 AI Generator (see above)                                         |
| Extract Prediction ID  | Code             | Extract prediction ID and URL      | Create Prediction        | Wait                     | ## Jhonp4 Jhonpiedrahita_ai01 AI Generator (see above)                                         |
| Wait                   | Wait             | Wait before polling prediction     | Extract Prediction ID, Check If Complete (false branch) | Check Prediction Status | ## Jhonp4 Jhonpiedrahita_ai01 AI Generator (see above)                                         |
| Check Prediction Status| HTTP Request     | Query prediction status            | Wait                     | Check If Complete        | ## Jhonp4 Jhonpiedrahita_ai01 AI Generator (see above)                                         |
| Check If Complete      | If               | Check if prediction succeeded      | Check Prediction Status  | Process Result (true), Wait (false) | ## Jhonp4 Jhonpiedrahita_ai01 AI Generator (see above)                                         |
| Process Result         | Code             | Format and output prediction result| Check If Complete (true) | None                     | ## Jhonp4 Jhonpiedrahita_ai01 AI Generator (see above)                                         |
| Sticky Note            | Sticky Note      | Documentation and instructions     | None                     | None                     | See node content above                                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node**  
   - Name: `On clicking 'execute'`  
   - Use default manual trigger settings.

2. **Create a Set Node to Store API Key**  
   - Name: `Set API Key`  
   - Add a string field named `replicate_api_key`  
   - Set its value to your Replicate API key (replace placeholder `YOUR_REPLICATE_API_KEY`)  
   - Connect `On clicking 'execute'` → `Set API Key`.

3. **Create an HTTP Request Node to Create Prediction**  
   - Name: `Create Prediction`  
   - Method: POST  
   - URL: `https://api.replicate.com/v1/predictions`  
   - Authentication: Generic HTTP Header with header:  
     - `Authorization: Bearer {{$node["Set API Key"].json["replicate_api_key"]}}`  
     - `Content-Type: application/json`  
   - Body Type: JSON  
   - Body Content:
     ```json
     {
       "version": "fcdcce61392c00f15318346038baa7c83af0a37e3e37a0318dd6aa8de33684b2",
       "input": {
         "prompt": "prompt value",
         "seed": 1,
         "width": 1,
         "height": 1,
         "lora_scale": 1
       }
     }
     ```
   - Timeout: 60 seconds  
   - Connect `Set API Key` → `Create Prediction`.

4. **Create a Code Node to Extract Prediction ID**  
   - Name: `Extract Prediction ID`  
   - Run Once For Each Item  
   - JavaScript code:
     ```js
     const prediction = $input.item.json;
     const predictionId = prediction.id;
     const initialStatus = prediction.status;

     return {
       predictionId: predictionId,
       status: initialStatus,
       predictionUrl: `https://api.replicate.com/v1/predictions/${predictionId}`
     };
     ```
   - Connect `Create Prediction` → `Extract Prediction ID`.

5. **Create a Wait Node**  
   - Name: `Wait`  
   - Wait Time: 2 seconds  
   - Connect `Extract Prediction ID` → `Wait`.

6. **Create an HTTP Request Node to Check Prediction Status**  
   - Name: `Check Prediction Status`  
   - Method: GET (default)  
   - URL: `={{ $json["predictionUrl"] }}` (use expression to get URL from previous node)  
   - Authentication: Generic HTTP Header with header:  
     - `Authorization: Bearer {{$node["Set API Key"].json["replicate_api_key"]}}`  
   - Connect `Wait` → `Check Prediction Status`.

7. **Create an If Node to Check Completion**  
   - Name: `Check If Complete`  
   - Condition: Boolean  
   - Expression: `$json["status"] === "succeeded"`  
   - Connect `Check Prediction Status` → `Check If Complete`.

8. **Connect If Node branches:**  
   - True branch → Create a Code Node `Process Result`  
   - False branch → Loop back to `Wait` node for polling.

9. **Create a Code Node to Process Result**  
   - Name: `Process Result`  
   - Run Once For Each Item  
   - JavaScript code:
     ```js
     const result = $input.item.json;

     return {
       status: result.status,
       output: result.output,
       metrics: result.metrics,
       created_at: result.created_at,
       completed_at: result.completed_at,
       model: 'jhonp4/jhonpiedrahita_ai01',
       other_url: result.output
     };
     ```
   - Connect `Check If Complete` (true branch) → `Process Result`.

10. **(Optional) Add a Sticky Note Node** near the start for user instructions:  
    - Content:
      ```
      ## Jhonp4 Jhonpiedrahita_ai01 AI Generator

      This workflow uses the **jhonp4/jhonpiedrahita_ai01** model from Replicate to generate other content.

      ### Setup
      1. Add your Replicate API key
      2. Configure the input parameters
      3. Run the workflow

      ### Model Details
      - Type: Other Generation
      - Provider: jhonp4
      - Required Fields: prompt
      ```

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                   | Context or Link                                                                                  |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| The workflow is designed for manual execution with an interactive trigger. To automate, consider adding trigger nodes or webhook integration.                                  | Workflow design consideration                                                                   |
| The Replicate model version ID `fcdcce61392c00f15318346038baa7c83af0a37e3e37a0318dd6aa8de33684b2` is specific to the `jhonp4/jhonpiedrahita_ai01` model; update if model changes. | Replicate API versioning                                                                         |
| API key must have sufficient permissions and quota on Replicate. Invalid keys or revoked tokens will cause authentication errors.                                             | Authentication prerequisite                                                                     |
| Polling interval is fixed at 2 seconds; consider adding a maximum retry or timeout to avoid infinite loops if prediction never completes.                                      | Workflow robustness recommendation                                                             |
| Replicate API documentation: https://replicate.com/docs                                                                                                                       | Official API reference                                                                           |
| The workflow currently does not handle prediction failure states other than "succeeded". Adding error handling for "failed" or "canceled" statuses is recommended.              | Error handling improvement                                                                      |

---

**Disclaimer:**  
The text provided is exclusively from an automated workflow built with n8n, a tool for integration and automation. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All manipulated data are legal and public.