Generate Images with Settyan Flash AI via Replicate API

https://n8nworkflows.xyz/workflows/generate-images-with-settyan-flash-ai-via-replicate-api-7105


# Generate Images with Settyan Flash AI via Replicate API

### 1. Workflow Overview

This workflow, titled **Settyan Flash V2.0.0 Beta.9 AI Generator**, is designed to generate images or other content using the **settyan/flash-v2.0.0-beta.9** AI model hosted on the Replicate platform. It automates the process of sending generation requests to the Replicate API, polling for completion, and processing the results once available.

The workflow is structured into the following logical blocks:

- **1.1 Input Initialization:** Manual trigger and setting the API key.
- **1.2 Prediction Creation:** Sending the generation request to Replicate with configured inputs.
- **1.3 Prediction Monitoring:** Polling the API until the prediction status is completed.
- **1.4 Result Processing:** Extracting and formatting the output data from the completed prediction.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Initialization

- **Overview:**  
  This block initiates the workflow execution manually and sets the required Replicate API key for authentication.

- **Nodes Involved:**  
  - On clicking 'execute' (Manual Trigger)  
  - Set API Key

- **Node Details:**

  - **On clicking 'execute'**  
    - Type: Manual Trigger  
    - Role: Entry point to start the workflow manually  
    - Configuration: Default manual trigger with no parameters  
    - Inputs: None  
    - Outputs: Connects to "Set API Key"  
    - Edge Cases: None (manual user action required)

  - **Set API Key**  
    - Type: Set  
    - Role: Defines the Replicate API key as a workflow variable for authentication  
    - Configuration: Assigns the string `"YOUR_REPLICATE_API_KEY"` to `replicate_api_key`  
    - Inputs: From Manual Trigger  
    - Outputs: Connects to "Create Prediction"  
    - Edge Cases: Failure if the API key is missing or invalid; must be replaced with a valid key before execution

---

#### 1.2 Prediction Creation

- **Overview:**  
  This block composes and sends a POST request to the Replicate API to create a new prediction job using the specified AI model and input parameters.

- **Nodes Involved:**  
  - Create Prediction  
  - Extract Prediction ID

- **Node Details:**

  - **Create Prediction**  
    - Type: HTTP Request  
    - Role: Sends POST request to `https://api.replicate.com/v1/predictions` to start image generation  
    - Configuration:  
      - Method: POST  
      - Authentication: HTTP Header Auth using the API key from "Set API Key" node  
      - Headers: Authorization Bearer token and Content-Type application/json  
      - Body: JSON containing:  
        - version: fixed model version ID `3053b0f8f3cff9282f18eb7782feb7b8d8b2e8dffb256d316cab1225a2cf3dad`  
        - input object with prompt, seed, width, height, lora_scale (example values, prompt is placeholder `"prompt value"`)  
      - Timeout: 60 seconds  
    - Inputs: From "Set API Key"  
    - Outputs: Connects to "Extract Prediction ID"  
    - Edge Cases:  
      - Network timeouts or API errors (HTTP 4xx/5xx)  
      - Invalid or missing input parameters causing API rejection  
      - Authentication errors if API key is invalid

  - **Extract Prediction ID**  
    - Type: Code  
    - Role: Parses the API response to extract prediction ID, status, and constructs a URL for polling  
    - Configuration: JavaScript code runs once per item, extracting:  
      - `predictionId` from response JSON's `id`  
      - `status` from response JSON's `status`  
      - Constructs `predictionUrl` for checking status  
    - Inputs: From "Create Prediction"  
    - Outputs: Connects to "Wait" node for polling delay  
    - Edge Cases: Failure if response JSON is malformed or missing expected fields

---

#### 1.3 Prediction Monitoring

- **Overview:**  
  This block waits a short interval, then polls the Replicate API repeatedly until the prediction status is `'succeeded'`.

- **Nodes Involved:**  
  - Wait  
  - Check Prediction Status  
  - Check If Complete

- **Node Details:**

  - **Wait**  
    - Type: Wait  
    - Role: Pauses workflow execution for 2 seconds between status polling attempts to avoid excessive API calls  
    - Configuration: Wait for 2 seconds  
    - Inputs: From "Extract Prediction ID" initially, and from "Check If Complete" if prediction is not finished  
    - Outputs: Connects to "Check Prediction Status"  
    - Edge Cases: None significant; excessive waiting may slow workflow

  - **Check Prediction Status**  
    - Type: HTTP Request  
    - Role: Performs GET request to the stored `predictionUrl` to fetch current status  
    - Configuration:  
      - URL: dynamic from previous node's `predictionUrl`  
      - Headers: Authorization Bearer token with API key  
      - Authentication: HTTP Header Auth  
    - Inputs: From "Wait"  
    - Outputs: Connects to "Check If Complete"  
    - Edge Cases:  
      - Network or API errors  
      - Unexpected status values  
      - Authentication failure

  - **Check If Complete**  
    - Type: If  
    - Role: Branches workflow based on whether prediction status equals `'succeeded'`  
    - Configuration: Boolean condition comparing `$json.status` to string `"succeeded"`  
    - Inputs: From "Check Prediction Status"  
    - Outputs:  
      - True branch: Connects to "Process Result"  
      - False branch: Connects back to "Wait" to continue polling  
    - Edge Cases:  
      - Status values like `'failed'` or `'canceled'` are not explicitly handled and may cause indefinite polling  
      - Could be improved with additional condition handling for failure states

---

#### 1.4 Result Processing

- **Overview:**  
  After successful prediction completion, this block extracts and formats useful output data for downstream use or display.

- **Nodes Involved:**  
  - Process Result

- **Node Details:**

  - **Process Result**  
    - Type: Code  
    - Role: Runs JavaScript to extract relevant result fields and output a clean JSON object  
    - Configuration:  
      - Extracts `status`, `output`, `metrics`, `created_at`, `completed_at` from API response  
      - Adds static `model` name string `'settyan/flash-v2.0.0-beta.9'`  
      - Adds `other_url` field duplicating `output` URL  
    - Inputs: From "Check If Complete" (true branch)  
    - Outputs: Final processed JSON object  
    - Edge Cases: None critical; assumes all fields exist in successful response

---

### 3. Summary Table

| Node Name              | Node Type          | Functional Role                                | Input Node(s)              | Output Node(s)               | Sticky Note                                                                                         |
|------------------------|--------------------|-----------------------------------------------|----------------------------|-----------------------------|---------------------------------------------------------------------------------------------------|
| On clicking 'execute'   | Manual Trigger     | Entry point to start the workflow manually    | -                          | Set API Key                 |                                                                                                   |
| Set API Key            | Set                | Stores Replicate API key for authentication   | On clicking 'execute'       | Create Prediction           |                                                                                                   |
| Create Prediction       | HTTP Request       | Sends request to Replicate API to create job  | Set API Key                | Extract Prediction ID       |                                                                                                   |
| Extract Prediction ID   | Code               | Extracts prediction ID and status from response| Create Prediction          | Wait                       |                                                                                                   |
| Wait                   | Wait               | Waits 2 seconds between polling requests       | Extract Prediction ID, Check If Complete (False branch) | Check Prediction Status      |                                                                                                   |
| Check Prediction Status | HTTP Request       | Polls Replicate API for current prediction status | Wait                     | Check If Complete           |                                                                                                   |
| Check If Complete       | If                 | Determines if prediction is completed          | Check Prediction Status     | Process Result (True), Wait (False) |                                                                                                   |
| Process Result          | Code               | Processes and formats final prediction output  | Check If Complete (True)    | -                           |                                                                                                   |
| Sticky Note            | Sticky Note        | Provides workflow overview and setup instructions | -                          | -                           | ## Settyan Flash V2.0.0 Beta.9 AI Generator\n\nThis workflow uses the **settyan/flash-v2.0.0-beta.9** model from Replicate to generate other content.\n\n### Setup\n1. Add your Replicate API key\n2. Configure the input parameters\n3. Run the workflow\n\n### Model Details\n- **Type**: Other Generation\n- **Provider**: settyan\n- **Required Fields**: prompt |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a **Manual Trigger** node named `On clicking 'execute'`. No special parameters needed.

2. **Create Set API Key Node**  
   - Add a **Set** node named `Set API Key`.  
   - Add a string field `replicate_api_key` with value `YOUR_REPLICATE_API_KEY` (replace this with your actual Replicate API key).  
   - Connect `On clicking 'execute'` → `Set API Key`.

3. **Create Create Prediction Node**  
   - Add an **HTTP Request** node named `Create Prediction`.  
   - Set method to `POST`.  
   - URL: `https://api.replicate.com/v1/predictions`.  
   - Under Authentication, select `HTTP Header Auth`.  
   - Add header `Authorization` with value `Bearer {{$node["Set API Key"].json["replicate_api_key"]}}`.  
   - Add header `Content-Type` with value `application/json`.  
   - Under Body Parameters, select `JSON` and enter:  
     ```json
     {
       "version": "3053b0f8f3cff9282f18eb7782feb7b8d8b2e8dffb256d316cab1225a2cf3dad",
       "input": {
         "prompt": "prompt value",
         "seed": 1,
         "width": 1,
         "height": 1,
         "lora_scale": 1
       }
     }
     ```  
   - Set timeout to 60000 ms.  
   - Connect `Set API Key` → `Create Prediction`.

4. **Create Extract Prediction ID Node**  
   - Add a **Code** node named `Extract Prediction ID`.  
   - Set mode to run once per item.  
   - Paste the following JavaScript:  
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
   - Connect `Create Prediction` → `Extract Prediction ID`.

5. **Create Wait Node**  
   - Add a **Wait** node named `Wait`.  
   - Set to wait for 2 seconds.  
   - Connect `Extract Prediction ID` → `Wait`.

6. **Create Check Prediction Status Node**  
   - Add an **HTTP Request** node named `Check Prediction Status`.  
   - Set method to `GET`.  
   - URL: `{{$json["predictionUrl"]}}`.  
   - Authentication: HTTP Header Auth.  
   - Header `Authorization`: `Bearer {{$node["Set API Key"].json["replicate_api_key"]}}`.  
   - Connect `Wait` → `Check Prediction Status`.

7. **Create Check If Complete Node**  
   - Add an **If** node named `Check If Complete`.  
   - Set condition to Boolean:  
     - Value 1: `{{$json["status"]}}`  
     - Operation: Equal  
     - Value 2: `succeeded`  
   - Connect `Check Prediction Status` → `Check If Complete`.

8. **Create Process Result Node**  
   - Add a **Code** node named `Process Result`.  
   - Set to run once per item.  
   - Paste the following JavaScript:  
     ```javascript
     const result = $input.item.json;

     return {
       status: result.status,
       output: result.output,
       metrics: result.metrics,
       created_at: result.created_at,
       completed_at: result.completed_at,
       model: 'settyan/flash-v2.0.0-beta.9',
       other_url: result.output
     };
     ```  
   - Connect `Check If Complete` (true branch) → `Process Result`.

9. **Create Loop Back to Wait for Polling**  
   - Connect `Check If Complete` (false branch) → `Wait`.

10. **Optional: Add Sticky Note**  
    - Add a **Sticky Note** with the overview, setup instructions, and model details as per the content in the original workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                   | Context or Link                                                |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------|
| This workflow uses the **settyan/flash-v2.0.0-beta.9** model from Replicate to generate content. The model requires a prompt and supports parameters like seed, width, height, and lora_scale. Replace the placeholder prompt and parameters in the "Create Prediction" node with your desired inputs.                                                                                                     | Workflow description and model documentation                   |
| The Replicate API key must be obtained from your Replicate account and kept secure. Without a valid key, the workflow will fail authentication.                                                                                                                                                                                                                                               | https://replicate.com/docs/api                                  |
| Polling is done every 2 seconds; this interval can be adjusted in the "Wait" node to balance responsiveness and API rate limits. Consider adding error handling for prediction failures (e.g., status `'failed'`) for production robustness.                                                                                                                                                     | API polling strategy                                           |
| Model version ID `3053b0f8f3cff9282f18eb7782feb7b8d8b2e8dffb256d316cab1225a2cf3dad` is fixed; update this if you want to use newer versions of the model.                                                                                                                                                                                                                                      | Version control of AI models                                  |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.