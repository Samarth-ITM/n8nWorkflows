Text-Guided Image Editing with ByteDance Seededit 3.0 on Replicate

https://n8nworkflows.xyz/workflows/text-guided-image-editing-with-bytedance-seededit-3-0-on-replicate-6882


# Text-Guided Image Editing with ByteDance Seededit 3.0 on Replicate

### 1. Workflow Overview

This workflow automates text-guided image editing using the **bytedance/seededit-3.0** AI model hosted on Replicate. It is designed to generate or modify images based on a textual prompt and an input image URL, leveraging the Seededit 3.0 model’s capabilities for guided image synthesis.

The workflow is structured into the following logical blocks:

- **1.1 Trigger and Initialization:** Manual trigger and setting of API authentication credentials.
- **1.2 Prediction Creation:** Sending a prediction request to the Replicate API with user-defined input parameters.
- **1.3 Prediction Monitoring:** Polling the prediction status until completion.
- **1.4 Result Processing:** Extracting and formatting the final output from the completed prediction.

This modular structure allows clear separation of concerns: starting the workflow, initiating the AI image generation request, waiting for completion, and handling the output.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger and Initialization

**Overview:**  
This block initiates the workflow manually and sets the required API key for authenticating requests to the Replicate API.

**Nodes Involved:**  
- On clicking 'execute'  
- Set API Key  

**Node Details:**  

- **On clicking 'execute'**  
  - Type: Manual Trigger (n8n-nodes-base.manualTrigger)  
  - Configuration: No parameters; triggers workflow execution manually.  
  - Inputs: None  
  - Outputs: Connects to "Set API Key" node  
  - Edge Cases: None typical; user must manually trigger.

- **Set API Key**  
  - Type: Set (n8n-nodes-base.set)  
  - Configuration: Sets a string variable `replicate_api_key` with the user’s Replicate API key (placeholder `"YOUR_REPLICATE_API_KEY"`).  
  - Inputs: From manual trigger  
  - Outputs: Connects to "Create Prediction" node  
  - Edge Cases: Missing or invalid API key will cause authentication failures downstream.

---

#### 2.2 Prediction Creation

**Overview:**  
This block sends a POST request to Replicate’s prediction API to create a new image generation task with specified parameters including prompt, input image, seed, and guidance scale.

**Nodes Involved:**  
- Create Prediction  
- Extract Prediction ID  

**Node Details:**  

- **Create Prediction**  
  - Type: HTTP Request (n8n-nodes-base.httpRequest)  
  - Configuration:  
    - URL: `https://api.replicate.com/v1/predictions` (POST)  
    - Request body (JSON):  
      - `version`: Fixed model version ID for bytedance/seededit-3.0  
      - `input`: Object with keys:  
        - `prompt`: A placeholder string `"prompt value"`; user should replace with actual prompt text.  
        - `image`: URL string `"https://example.com/input.image"`; user should replace with actual image URL.  
        - `seed`: integer (1) for random seed control  
        - `guidance_scale`: float (5.5), controls adherence to prompt  
    - Headers: Authorization with Bearer token from `replicate_api_key`, Content-Type: application/json  
  - Inputs: From "Set API Key"  
  - Outputs: Connects to "Extract Prediction ID"  
  - Edge Cases:  
    - Network timeouts (configured timeout 60s)  
    - Invalid API key or model version leads to 401/404 errors  
    - Malformed input parameters cause 400 errors
  
- **Extract Prediction ID**  
  - Type: Code (n8n-nodes-base.code)  
  - Configuration:  
    - JavaScript extracts `id` and `status` from response JSON.  
    - Constructs `predictionUrl` for later polling:  
      `https://api.replicate.com/v1/predictions/{predictionId}`  
  - Inputs: From "Create Prediction"  
  - Outputs: Connects to "Wait" node  
  - Edge Cases:  
    - Unexpected response shape may cause code errors  
    - Missing ID or status breaks polling logic

---

#### 2.3 Prediction Monitoring

**Overview:**  
This block repeatedly polls the prediction endpoint until the prediction completes successfully.

**Nodes Involved:**  
- Wait  
- Check Prediction Status  
- Check If Complete  

**Node Details:**  

- **Wait**  
  - Type: Wait (n8n-nodes-base.wait)  
  - Configuration: Waits 2 seconds between polls to avoid rate limits.  
  - Inputs: From "Extract Prediction ID" or "Check If Complete" (in incomplete case)  
  - Outputs: Connects to "Check Prediction Status"  
  - Edge Cases: Very long wait times if prediction is slow; no max retries implemented.

- **Check Prediction Status**  
  - Type: HTTP Request (n8n-nodes-base.httpRequest)  
  - Configuration:  
    - URL: dynamic, uses `predictionUrl` from previous step  
    - Headers: Authorization with Bearer token  
    - GET request (default)  
  - Inputs: From "Wait"  
  - Outputs: Connects to "Check If Complete"  
  - Edge Cases: Network errors, API errors, or expired tokens can disrupt polling.

- **Check If Complete**  
  - Type: If (n8n-nodes-base.if)  
  - Configuration: Checks if `status` field equals `"succeeded"`.  
  - Inputs: From "Check Prediction Status"  
  - Outputs:  
    - True branch: Connects to "Process Result"  
    - False branch: Connects back to "Wait" node (polling loop)  
  - Edge Cases:  
    - If status is `"failed"` or other terminal states, loop may continue indefinitely without handling failure explicitly.

---

#### 2.4 Result Processing

**Overview:**  
Once the prediction completes, this block processes the output, extracts relevant data, and prepares a simplified JSON result with key fields.

**Nodes Involved:**  
- Process Result  

**Node Details:**  

- **Process Result**  
  - Type: Code (n8n-nodes-base.code)  
  - Configuration:  
    - Extracts `status`, `output`, `metrics`, `created_at`, `completed_at` from prediction JSON.  
    - Adds static field `model` set to `'bytedance/seededit-3.0'`.  
    - Sets `image_url` to the output value (typically URL(s) of generated image(s)).  
  - Inputs: From "Check If Complete" (true branch)  
  - Outputs: Final output of the workflow  
  - Edge Cases: If output is missing or malformed, downstream usage may fail.

---

#### 2.5 Informational Sticky Note

**Overview:**  
Provides user guidance, model description, and setup instructions.

**Nodes Involved:**  
- Sticky Note  

**Node Details:**  

- **Sticky Note**  
  - Type: Sticky Note (n8n-nodes-base.stickyNote)  
  - Configuration: Contains markdown content describing the workflow’s purpose, setup steps, and model details.  
  - Position: Top-left for visibility  
  - No inputs or outputs  
  - Content includes:  
    - Workflow name  
    - Setup instructions (add API key, configure inputs, run)  
    - Model details (type, provider, required fields)  
  - Edge Cases: None; purely informational.

---

### 3. Summary Table

| Node Name                | Node Type              | Functional Role                  | Input Node(s)              | Output Node(s)           | Sticky Note                                                                                          |
|--------------------------|------------------------|--------------------------------|----------------------------|--------------------------|----------------------------------------------------------------------------------------------------|
| On clicking 'execute'     | Manual Trigger         | Start workflow manually         | —                          | Set API Key               |                                                                                                    |
| Set API Key              | Set                    | Store Replicate API key         | On clicking 'execute'       | Create Prediction         |                                                                                                    |
| Create Prediction         | HTTP Request           | Send prediction creation request | Set API Key                 | Extract Prediction ID     |                                                                                                    |
| Extract Prediction ID     | Code                   | Extract prediction ID and initial status | Create Prediction           | Wait                     |                                                                                                    |
| Wait                     | Wait                   | Pause between status checks     | Extract Prediction ID, Check If Complete (false) | Check Prediction Status |                                                                                                    |
| Check Prediction Status   | HTTP Request           | Poll prediction status          | Wait                       | Check If Complete         |                                                                                                    |
| Check If Complete         | If                     | Determine if prediction succeeded | Check Prediction Status     | Process Result (true), Wait (false) |                                                                                                    |
| Process Result            | Code                   | Prepare final output summary    | Check If Complete (true)    | —                        |                                                                                                    |
| Sticky Note               | Sticky Note            | User guidance and workflow info | —                          | —                        | ## Bytedance Seededit 3.0 Image Generator<br> This workflow uses the **bytedance/seededit-3.0** model from Replicate.<br> ### Setup<br> 1. Add your Replicate API key<br> 2. Configure the input parameters<br> 3. Run the workflow<br> ### Model Details<br> - **Type**: Image Generation<br> - **Provider**: bytedance<br> - **Required Fields**: prompt, image |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Name: `On clicking 'execute'`  
   - No special configuration.

2. **Create Set Node for API Key**  
   - Type: Set  
   - Name: `Set API Key`  
   - Add a new string field named `replicate_api_key`  
   - Set value to your actual Replicate API key (replace `"YOUR_REPLICATE_API_KEY"`).  
   - Connect `On clicking 'execute'` → `Set API Key`.

3. **Create HTTP Request Node for Prediction Creation**  
   - Type: HTTP Request  
   - Name: `Create Prediction`  
   - Method: POST  
   - URL: `https://api.replicate.com/v1/predictions`  
   - Authentication: Generic Credential, HTTP Header Auth  
     - Header `Authorization`: `Bearer {{$node["Set API Key"].json["replicate_api_key"]}}`  
     - Header `Content-Type`: `application/json`  
   - Request Body Type: JSON  
   - JSON Body:  
     ```json
     {
       "version": "2c4be22ec2c834b160f6587130d5b9d5ba6d498a5203b80ab874f35d0ce73fa6",
       "input": {
         "prompt": "prompt value",
         "image": "https://example.com/input.image",
         "seed": 1,
         "guidance_scale": 5.5
       }
     }
     ```  
   - Set timeout to 60 seconds  
   - Connect `Set API Key` → `Create Prediction`.

4. **Create Code Node to Extract Prediction ID**  
   - Type: Code  
   - Name: `Extract Prediction ID`  
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
   - Connect `Create Prediction` → `Extract Prediction ID`.

5. **Create Wait Node**  
   - Type: Wait  
   - Name: `Wait`  
   - Wait Time: 2 seconds  
   - Connect `Extract Prediction ID` → `Wait`.

6. **Create HTTP Request Node to Check Prediction Status**  
   - Type: HTTP Request  
   - Name: `Check Prediction Status`  
   - Method: GET (default)  
   - URL: `{{$json["predictionUrl"]}}`  
   - Authentication: Generic Credential, HTTP Header Auth  
     - Header `Authorization`: `Bearer {{$node["Set API Key"].json["replicate_api_key"]}}`  
   - Connect `Wait` → `Check Prediction Status`.

7. **Create If Node to Check Completion**  
   - Type: If  
   - Name: `Check If Complete`  
   - Condition:  
     - Boolean: `{{$json["status"]}}` equals `succeeded`  
   - Connect `Check Prediction Status` → `Check If Complete`.

8. **Create Code Node to Process Result**  
   - Type: Code  
   - Name: `Process Result`  
   - Code:  
     ```javascript
     const result = $input.item.json;
     return {
       status: result.status,
       output: result.output,
       metrics: result.metrics,
       created_at: result.created_at,
       completed_at: result.completed_at,
       model: 'bytedance/seededit-3.0',
       image_url: result.output
     };
     ```  
   - Connect `Check If Complete` (true) → `Process Result`.

9. **Connect Polling Loop**  
   - Connect `Check If Complete` (false) → `Wait` to repeat polling.

10. **Optional: Add Sticky Note**  
    - Type: Sticky Note  
    - Position near start nodes for visibility  
    - Content:  
      ```
      ## Bytedance Seededit 3.0 Image Generator

      This workflow uses the **bytedance/seededit-3.0** model from Replicate to generate image content.

      ### Setup
      1. Add your Replicate API key
      2. Configure the input parameters
      3. Run the workflow

      ### Model Details
      - **Type**: Image Generation
      - **Provider**: bytedance
      - **Required Fields**: prompt, image
      ```

---

### 5. General Notes & Resources

| Note Content                                                                                                             | Context or Link                                                                                   |
|--------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| The model version ID `2c4be22ec2c834b160f6587130d5b9d5ba6d498a5203b80ab874f35d0ce73fa6` is specific to bytedance/seededit-3.0 | Replicate API documentation: https://replicate.com/docs/api-reference/predictions/create        |
| Use a valid image URL accessible by Replicate’s servers for the `image` parameter to avoid errors                         | Replace `"https://example.com/input.image"` with your actual image URL                           |
| Polling interval is set to 2 seconds; adjust if API rate limits or latency require                                       | Consider adding a max retry or failure handling in production use                               |
| Authentication uses Bearer token in HTTP header; ensure your API key is kept secure                                       | Store API keys securely; do not hardcode in shared workflows                                   |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.