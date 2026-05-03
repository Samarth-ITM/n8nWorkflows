Generate Images with Lorealcantara AI Model via Replicate API

https://n8nworkflows.xyz/workflows/generate-images-with-lorealcantara-ai-model-via-replicate-api-7096


# Generate Images with Lorealcantara AI Model via Replicate API

### 1. Workflow Overview

This workflow named **Ligua033 Lorealcantara AI Generator** automates the process of generating images (or other generated content) by invoking the **ligua033/lorealcantara** AI model hosted on the Replicate API platform. The workflow is designed for users who want to programmatically send a prompt to the model and retrieve the generated output asynchronously, handling API polling and result extraction.

The logical structure of the workflow can be divided into the following blocks:

- **1.1 Manual Trigger & API Key Setup:** Initiates the workflow manually and assigns the Replicate API key.
- **1.2 Prediction Request Creation:** Sends a prediction creation request to the Replicate API with user-defined input parameters.
- **1.3 Prediction Monitoring & Polling:** Extracts the prediction ID, then polls the API repeatedly until the prediction is complete.
- **1.4 Result Processing:** Processes the final prediction result, extracting relevant information for output or further use.

---

### 2. Block-by-Block Analysis

#### 2.1 Manual Trigger & API Key Setup

- **Overview:** This block starts the workflow on user command and sets the API key required for authenticating requests to the Replicate API.
- **Nodes Involved:**  
  - On clicking 'execute'  
  - Set API Key  

- **Node Details:**

  - **On clicking 'execute'**  
    - Type: Manual Trigger  
    - Role: Starts the workflow manually; no input required.  
    - Configuration: Default manual trigger with no parameters.  
    - Inputs: None  
    - Outputs: Initiates "Set API Key" node.  
    - Edge cases: None typical; user must manually trigger to start.  

  - **Set API Key**  
    - Type: Set  
    - Role: Sets a workflow variable `replicate_api_key` used for authentication.  
    - Configuration: Assigns a string variable `replicate_api_key` with the placeholder value `"YOUR_REPLICATE_API_KEY"`.  
    - Inputs: Output of manual trigger node.  
    - Outputs: Passes the API key to the HTTP request node.  
    - Edge cases: User must replace the placeholder with a valid Replicate API key to avoid authentication errors.

#### 2.2 Prediction Request Creation

- **Overview:** Sends a POST request to the Replicate API to create a new prediction job using the specified AI model and input parameters.
- **Nodes Involved:**  
  - Create Prediction  

- **Node Details:**

  - **Create Prediction**  
    - Type: HTTP Request  
    - Role: Calls Replicate API endpoint to create a prediction.  
    - Configuration:  
      - URL: `https://api.replicate.com/v1/predictions`  
      - Method: POST  
      - Timeout: 60 seconds  
      - Body (JSON):  
        - Model version: `"21c8ba35556f26425991a971a1419e264fccc020cd32170d54924d1958f2e397"`  
        - Inputs: prompt (string), seed (int), width (int), height (int), lora_scale (float) — currently hardcoded but can be parameterized.  
      - Headers: Authorization with Bearer token using API key from "Set API Key" node, Content-Type as application/json.  
    - Inputs: Receives API key from "Set API Key" node.  
    - Outputs: JSON response containing prediction ID and initial status.  
    - Edge cases:  
      - API key invalid or missing → HTTP 401 Unauthorized.  
      - Timeout or network errors.  
      - Invalid input parameters → API validation errors.  
    - Version requirements: HTTP Request node v4.2 or higher recommended for advanced authentication handling.

#### 2.3 Prediction Monitoring & Polling

- **Overview:** Extracts the prediction ID and initial status from the creation response, then repeatedly polls the prediction status endpoint until the prediction is marked as "succeeded."
- **Nodes Involved:**  
  - Extract Prediction ID  
  - Wait  
  - Check Prediction Status  
  - Check If Complete  

- **Node Details:**

  - **Extract Prediction ID**  
    - Type: Code  
    - Role: Parses the prediction creation response to retrieve the prediction ID, initial status, and constructs the URL for status checking.  
    - Configuration: JavaScript code extracting `id`, `status` from input and returning an object with those plus a URL string for polling.  
    - Inputs: Output from "Create Prediction" node.  
    - Outputs: Object with `predictionId`, `status`, and `predictionUrl`.  
    - Edge cases: If response structure changes, parsing may fail.

  - **Wait**  
    - Type: Wait  
    - Role: Introduces delay (2 seconds) between polling attempts to avoid rate limits or excessive API calls.  
    - Configuration: Wait for 2 seconds.  
    - Inputs: Runs after extracting prediction ID or after a failed completion check.  
    - Outputs: Triggers "Check Prediction Status".  
    - Edge cases: Excessive wait or too short wait may impact performance or API limits.

  - **Check Prediction Status**  
    - Type: HTTP Request  
    - Role: Sends GET request to the prediction status endpoint to check the current status of the prediction.  
    - Configuration:  
      - URL: Dynamic, from `predictionUrl` extracted previously.  
      - Method: GET (default)  
      - Headers: Authorization Bearer token with API key.  
    - Inputs: From "Wait" node.  
    - Outputs: JSON with updated status and output if ready.  
    - Edge cases: API errors, network failure, invalid URL, expired prediction ID.

  - **Check If Complete**  
    - Type: If  
    - Role: Condition node that checks if the prediction status equals "succeeded".  
    - Configuration: Boolean condition comparing `$json.status` to `"succeeded"`.  
    - Inputs: From "Check Prediction Status".  
    - Outputs:  
      - True branch: Prediction complete → "Process Result".  
      - False branch: Prediction not complete → loops back to "Wait".  
    - Edge cases: Other statuses like "failed" or "canceled" not explicitly handled; may cause indefinite polling.

#### 2.4 Result Processing

- **Overview:** Processes the completed prediction result, extracting output URLs and metadata for further use or export.
- **Nodes Involved:**  
  - Process Result  

- **Node Details:**

  - **Process Result**  
    - Type: Code  
    - Role: Extracts and formats key details from the final prediction response, including status, output URLs, metrics, timestamps, and model info.  
    - Configuration: JavaScript code returning an object with relevant fields: `status`, `output`, `metrics`, `created_at`, `completed_at`, `model`, and `other_url` (same as output).  
    - Inputs: Final successful prediction JSON from "Check If Complete" true branch.  
    - Outputs: Structured data usable for downstream systems or storage.  
    - Edge cases: If output is missing or malformed, data extraction may fail.

---

### 3. Summary Table

| Node Name             | Node Type          | Functional Role                           | Input Node(s)             | Output Node(s)             | Sticky Note                                                                                                         |
|-----------------------|--------------------|-----------------------------------------|---------------------------|----------------------------|---------------------------------------------------------------------------------------------------------------------|
| On clicking 'execute'  | Manual Trigger     | Initiates workflow manually              | —                         | Set API Key                |                                                                                                                     |
| Set API Key           | Set                | Sets Replicate API key variable          | On clicking 'execute'      | Create Prediction          |                                                                                                                     |
| Create Prediction     | HTTP Request       | Creates a prediction on Replicate API    | Set API Key               | Extract Prediction ID      |                                                                                                                     |
| Extract Prediction ID | Code               | Extracts prediction ID and status        | Create Prediction          | Wait                      |                                                                                                                     |
| Wait                  | Wait               | Delays polling to avoid API overload     | Extract Prediction ID / Check If Complete (false branch) | Check Prediction Status |                                                                                                                     |
| Check Prediction Status | HTTP Request     | Queries prediction status from API       | Wait                      | Check If Complete          |                                                                                                                     |
| Check If Complete     | If                  | Checks if prediction finished successfully | Check Prediction Status    | Process Result (true) / Wait (false) |                                                                                                                     |
| Process Result        | Code               | Extracts and formats prediction results  | Check If Complete (true)   | —                          |                                                                                                                     |
| Sticky Note           | Sticky Note        | Provides workflow and model explanation  | —                         | —                          | ## Ligua033 Lorealcantara AI Generator<br>This workflow uses the **ligua033/lorealcantara** model from Replicate.<br>Setup:<br>1. Add your Replicate API key<br>2. Configure input parameters<br>3. Run the workflow<br>Model Details:<br>- **Type**: Other Generation<br>- **Provider**: ligua033<br>- **Required Fields**: prompt |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Manual Trigger node:**  
   - Name: "On clicking 'execute'"  
   - Purpose: To manually start the workflow.

3. **Add a Set node:**  
   - Name: "Set API Key"  
   - Connect input from "On clicking 'execute'".  
   - Add a field:  
     - Name: `replicate_api_key`  
     - Type: String  
     - Value: `"YOUR_REPLICATE_API_KEY"` (replace with actual API key)  

4. **Add an HTTP Request node:**  
   - Name: "Create Prediction"  
   - Connect input from "Set API Key".  
   - Configure:  
     - HTTP Method: POST  
     - URL: `https://api.replicate.com/v1/predictions`  
     - Authentication: None (use Generic Credential via HTTP Header)  
     - Headers:  
       - `Authorization`: `Bearer {{$node["Set API Key"].json["replicate_api_key"]}}`  
       - `Content-Type`: `application/json`  
     - Body Content Type: JSON  
     - Request Body:  
       ```json
       {
         "version": "21c8ba35556f26425991a971a1419e264fccc020cd32170d54924d1958f2e397",
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

5. **Add a Code node:**  
   - Name: "Extract Prediction ID"  
   - Connect input from "Create Prediction".  
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

6. **Add a Wait node:**  
   - Name: "Wait"  
   - Connect input from "Extract Prediction ID" and also from "Check If Complete" false branch.  
   - Set to wait for 2 seconds.

7. **Add an HTTP Request node:**  
   - Name: "Check Prediction Status"  
   - Connect input from "Wait".  
   - Configure:  
     - HTTP Method: GET  
     - URL: `{{$json["predictionUrl"]}}` (dynamic from previous node)  
     - Headers:  
       - `Authorization`: `Bearer {{$node["Set API Key"].json["replicate_api_key"]}}`  
     - No body required.

8. **Add an If node:**  
   - Name: "Check If Complete"  
   - Connect input from "Check Prediction Status".  
   - Condition:  
     - Boolean check if `$json.status` equals `"succeeded"`  
   - True branch connects to "Process Result".  
   - False branch loops back to "Wait".

9. **Add a Code node:**  
   - Name: "Process Result"  
   - Connect input from "Check If Complete" true branch.  
   - Code (JavaScript):  
     ```javascript
     const result = $input.item.json;
     
     return {
       status: result.status,
       output: result.output,
       metrics: result.metrics,
       created_at: result.created_at,
       completed_at: result.completed_at,
       model: 'ligua033/lorealcantara',
       other_url: result.output
     };
     ```

10. **Optionally, add a Sticky Note node:**  
    - Content:  
      ```
      ## Ligua033 Lorealcantara AI Generator

      This workflow uses the **ligua033/lorealcantara** model from Replicate to generate other content.

      ### Setup
      1. Add your Replicate API key
      2. Configure the input parameters
      3. Run the workflow

      ### Model Details
      - **Type**: Other Generation
      - **Provider**: ligua033
      - **Required Fields**: prompt
      ```

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                   | Context or Link                                  |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------|
| The workflow uses the Replicate API, which requires an API key obtainable from https://replicate.com/account/api-tokens                                           | Replicate API Key Setup                          |
| The model version ID `"21c8ba35556f26425991a971a1419e264fccc020cd32170d54924d1958f2e397"` must be accurate and correspond to the current model version on Replicate | Model Version Reference                          |
| Polling interval is set at 2 seconds to balance responsiveness and API rate limits                                                                               | Polling Delay Configuration                       |
| No explicit error handling for prediction failure or cancellation status is implemented; consider extending the workflow to handle these states               | Potential Enhancement                             |
| Input parameters for the model (prompt, seed, width, height, lora_scale) are currently hardcoded; these can be parameterized to allow dynamic input             | Workflow Customization                            |

---

**Disclaimer:** The text provided is derived exclusively from an automated workflow created with n8n, a tool for integration and automation. This process strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.