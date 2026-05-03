Generate Images from Text with IBM Granite Vision 3.3 2B AI Model

https://n8nworkflows.xyz/workflows/generate-images-from-text-with-ibm-granite-vision-3-3-2b-ai-model-7012


# Generate Images from Text with IBM Granite Vision 3.3 2B AI Model

### 1. Workflow Overview

This workflow, titled **"Ibm Granite Granite Vision 3.3 2b Image Generator"**, is designed to generate images from text using the IBM Granite Vision 3.3 2B AI Model hosted on Replicate. It targets users who want to programmatically create images through an AI model by invoking the Replicate API and handling asynchronous prediction polling until completion.

The workflow logic is divided into the following functional blocks:

- **1.1 Manual Trigger and API Key Setup:** User initiates the workflow manually and sets the required Replicate API key.
- **1.2 Prediction Creation:** Sends a POST request to Replicate API to start an image generation prediction with specified model parameters.
- **1.3 Prediction ID Extraction:** Extracts the prediction ID and the initial status from the API response to track the prediction progress.
- **1.4 Polling Prediction Status:** Waits for a short interval, then polls the API repeatedly to check if the prediction has completed.
- **1.5 Completion Check and Result Processing:** Evaluates if the prediction succeeded; if yes, processes and outputs the final image URL and metadata, else continues polling.

---

### 2. Block-by-Block Analysis

#### 2.1 Manual Trigger and API Key Setup

- **Overview:** This block allows the user to start the workflow manually and input their Replicate API key to authenticate subsequent API requests.
- **Nodes Involved:**  
  - On clicking 'execute' (Manual Trigger)  
  - Set API Key  

- **Node Details:**

  - **On clicking 'execute'**  
    - *Type & Role:* Manual Trigger node; initiates the workflow execution.  
    - *Configuration:* No parameters; standard manual trigger.  
    - *Inputs:* None (entry point).  
    - *Outputs:* Connected to "Set API Key".  
    - *Edge Cases:* None; user must manually trigger.  
    - *Version:* v1.

  - **Set API Key**  
    - *Type & Role:* Set node; assigns static API key value for authentication.  
    - *Configuration:* Stores a string field named `replicate_api_key` with placeholder `"YOUR_REPLICATE_API_KEY"`.  
    - *Expressions:* None; static assignment.  
    - *Inputs:* From manual trigger.  
    - *Outputs:* To "Create Prediction".  
    - *Edge Cases:* Missing or invalid API key will cause authentication errors downstream.  
    - *Version:* v3.3.

#### 2.2 Prediction Creation

- **Overview:** Sends a POST request to Replicate's prediction API endpoint to start the image generation process using the IBM Granite Vision model with specified parameters.
- **Nodes Involved:**  
  - Create Prediction  

- **Node Details:**

  - **Create Prediction**  
    - *Type & Role:* HTTP Request node; initiates prediction creation on Replicate API.  
    - *Configuration:*  
      - URL: `https://api.replicate.com/v1/predictions`  
      - Method: POST  
      - Timeout: 60 seconds  
      - JSON Body: Includes model version ID and input parameters for prediction (`seed`, `top_k`, `top_p`, `max_tokens`, `temperature`). Note, these parameters appear more typical for language generation rather than image generation, which may require adjustment.  
      - Authentication: HTTP Header with Bearer token from `replicate_api_key` set in the previous node.  
      - Headers: `Authorization` and `Content-Type: application/json`.  
    - *Expressions:* Authorization header dynamically constructed using `$node["Set API Key"].item.json.replicate_api_key`.  
    - *Inputs:* From "Set API Key".  
    - *Outputs:* To "Extract Prediction ID".  
    - *Edge Cases:*  
      - API key invalid or expired → 401 Unauthorized.  
      - Timeout or network issues → HTTP request failure.  
      - Invalid input parameters → 400 Bad Request.  
    - *Version:* v4.2.

#### 2.3 Prediction ID Extraction

- **Overview:** Extracts the prediction ID and status from the response, and constructs a polling URL to check prediction status in subsequent steps.
- **Nodes Involved:**  
  - Extract Prediction ID  

- **Node Details:**

  - **Extract Prediction ID**  
    - *Type & Role:* Code node; parses API response JSON.  
    - *Configuration:* Runs JavaScript code once per item to extract:  
      - `predictionId` from response `id`  
      - `initialStatus` from response `status`  
      - Constructs `predictionUrl` for polling: `https://api.replicate.com/v1/predictions/${predictionId}`  
    - *Inputs:* From "Create Prediction".  
    - *Outputs:* To "Wait".  
    - *Edge Cases:*  
      - Missing or malformed API response → code failure.  
      - Unexpected response structure → undefined fields.  
    - *Version:* v2.

#### 2.4 Polling Prediction Status

- **Overview:** Implements a polling mechanism that waits for a delay and then checks the prediction status repeatedly until it completes.
- **Nodes Involved:**  
  - Wait  
  - Check Prediction Status  
  - Check If Complete  

- **Node Details:**

  - **Wait**  
    - *Type & Role:* Wait node; pauses workflow execution.  
    - *Configuration:* Waits for 2 seconds before proceeding.  
    - *Inputs:* From "Extract Prediction ID" initially, then from "Check If Complete" in loop.  
    - *Outputs:* To "Check Prediction Status".  
    - *Edge Cases:* None significant; delay too short may cause rapid polling.  
    - *Version:* v1.

  - **Check Prediction Status**  
    - *Type & Role:* HTTP Request node; polls Replicate API to get current prediction status.  
    - *Configuration:*  
      - URL: dynamically set to `predictionUrl` from previous code node output.  
      - Method: GET (default).  
      - Authentication: Bearer token header from stored API key.  
    - *Inputs:* From "Wait".  
    - *Outputs:* To "Check If Complete".  
    - *Edge Cases:*  
      - Network or API errors causing failed requests.  
      - Prediction ID no longer valid.  
    - *Version:* v4.2.

  - **Check If Complete**  
    - *Type & Role:* If node; checks if prediction status is "succeeded".  
    - *Configuration:* Condition tests if `$json.status == "succeeded"`.  
    - *Inputs:* From "Check Prediction Status".  
    - *Outputs:*  
      - True branch to "Process Result".  
      - False branch loops back to "Wait" for continued polling.  
    - *Edge Cases:*  
      - Status other than "succeeded" or "failed" could cause indefinite polling.  
      - Should ideally handle failure or cancellation status (not implemented here).  
    - *Version:* v1.

#### 2.5 Completion and Result Processing

- **Overview:** Once prediction succeeds, this block processes the final response, extracting output image URL and metadata for downstream use.
- **Nodes Involved:**  
  - Process Result  

- **Node Details:**

  - **Process Result**  
    - *Type & Role:* Code node; formats and returns key information from the completed prediction response.  
    - *Configuration:*  
      - Extracts `status`, `output` (image URL), `metrics`, `created_at`, `completed_at`.  
      - Hardcodes model name as `'ibm-granite/granite-vision-3.3-2b'`.  
      - Returns an object containing these fields plus `image_url` (same as output).  
    - *Inputs:* From "Check If Complete" (true branch).  
    - *Outputs:* Terminal output with processed result data.  
    - *Edge Cases:*  
      - If output is missing or malformed, image URL may be invalid.  
    - *Version:* v2.

---

### 3. Summary Table

| Node Name               | Node Type         | Functional Role                           | Input Node(s)          | Output Node(s)          | Sticky Note                                                                                     |
|-------------------------|-------------------|-----------------------------------------|-----------------------|-------------------------|------------------------------------------------------------------------------------------------|
| On clicking 'execute'    | Manual Trigger    | Start workflow manually                  | None                  | Set API Key             |                                                                                                |
| Set API Key             | Set               | Store Replicate API key                   | On clicking 'execute' | Create Prediction       |                                                                                                |
| Create Prediction       | HTTP Request      | Initiate image generation prediction     | Set API Key           | Extract Prediction ID   |                                                                                                |
| Extract Prediction ID   | Code              | Extract prediction ID and status          | Create Prediction     | Wait                    |                                                                                                |
| Wait                    | Wait              | Pause before polling API                   | Extract Prediction ID, Check If Complete (false) | Check Prediction Status |                                                                                                |
| Check Prediction Status | HTTP Request      | Poll prediction status                      | Wait                  | Check If Complete       |                                                                                                |
| Check If Complete       | If                | Check if prediction succeeded               | Check Prediction Status | Process Result (true), Wait (false) |                                                                                                |
| Process Result          | Code              | Format and output final prediction data    | Check If Complete (true) | None                   |                                                                                                |
| Sticky Note             | Sticky Note       | Documentation note about the workflow      | None                  | None                    | ## Ibm Granite Granite Vision 3.3 2b Image Generator<br><br>This workflow uses the **ibm-granite/granite-vision-3.3-2b** model from Replicate to generate image content.<br><br>### Setup<br>1. Add your Replicate API key<br>2. Configure the input parameters<br>3. Run the workflow<br><br>### Model Details<br>- **Type**: Image Generation<br>- **Provider**: ibm-granite<br>- **Required Fields**: None |

---

### 4. Reproducing the Workflow from Scratch

1. **Add Manual Trigger Node**  
   - Add an **"Manual Trigger"** node named **"On clicking 'execute'"**. No configuration needed.

2. **Add Set Node for API Key**  
   - Add a **"Set"** node named **"Set API Key"**.  
   - Add a string field `replicate_api_key` with the placeholder value `"YOUR_REPLICATE_API_KEY"`.  
   - Connect the **Manual Trigger** output to this node.

3. **Add HTTP Request Node to Create Prediction**  
   - Add an **"HTTP Request"** node named **"Create Prediction"**.  
   - Set method to **POST**.  
   - URL: `https://api.replicate.com/v1/predictions`.  
   - Timeout: 60000 ms (60 seconds).  
   - In **Body Parameters**, select **JSON** and enter:  
     ```json
     {
       "version": "3339e8453ca94104383f6f085a511d7f26cca2d0cab2f6018986737b6cf7d391",
       "input": {
         "seed": 1,
         "top_k": 50,
         "top_p": 0.9,
         "max_tokens": 512,
         "temperature": 0.6
       }
     }
     ```  
   - Under **Authentication**, choose **Generic Credential Type** → **HTTP Header Auth**.  
   - Add headers:  
     - `Authorization`: expression `={{ 'Bearer ' + $node["Set API Key"].json["replicate_api_key"] }}`  
     - `Content-Type`: `application/json`  
   - Connect **Set API Key** output to this node.

4. **Add Code Node to Extract Prediction ID**  
   - Add a **Code** node named **"Extract Prediction ID"**.  
   - Use the following JavaScript code:  
     ```javascript
     const prediction = $input.item.json;
     const predictionId = prediction.id;
     const initialStatus = prediction.status;

     return {
       predictionId,
       status: initialStatus,
       predictionUrl: `https://api.replicate.com/v1/predictions/${predictionId}`
     };
     ```  
   - Set it to run once per item.  
   - Connect **Create Prediction** output to this node.

5. **Add Wait Node for Polling Delay**  
   - Add a **Wait** node named **"Wait"**.  
   - Configure to wait for **2 seconds**.  
   - Connect **Extract Prediction ID** output to this node.

6. **Add HTTP Request Node to Check Prediction Status**  
   - Add an **HTTP Request** node named **"Check Prediction Status"**.  
   - Set URL to the expression: `={{ $json.predictionUrl }}`  
   - Method: GET (default).  
   - Authentication: same as Create Prediction (Generic Credential Type → HTTP Header Auth).  
   - Header: `Authorization` with expression `={{ 'Bearer ' + $node["Set API Key"].json["replicate_api_key"] }}`.  
   - Connect **Wait** output to this node.

7. **Add If Node to Check Completion**  
   - Add an **If** node named **"Check If Complete"**.  
   - Condition: Boolean → check if `$json.status` equals `"succeeded"`.  
   - Connect **Check Prediction Status** output to this node.

8. **Add Code Node to Process Final Result**  
   - Add a **Code** node named **"Process Result"**.  
   - Use the following JavaScript code:  
     ```javascript
     const result = $input.item.json;

     return {
       status: result.status,
       output: result.output,
       metrics: result.metrics,
       created_at: result.created_at,
       completed_at: result.completed_at,
       model: 'ibm-granite/granite-vision-3.3-2b',
       image_url: result.output
     };
     ```  
   - Set to run once per item.  
   - Connect the **true** branch of **Check If Complete** to this node.

9. **Loop Back for Polling**  
   - Connect the **false** branch of **Check If Complete** back to the **Wait** node to continue polling.

10. **Add Sticky Note (Optional)**  
    - Add a **Sticky Note** node with content describing the workflow, setup instructions, and model details for user reference.

---

### 5. General Notes & Resources

| Note Content                                                                                           | Context or Link                                                                                     |
|------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| This workflow uses the **ibm-granite/granite-vision-3.3-2b** model from Replicate for image generation. | Workflow description and model info embedded in the Sticky Note node.                             |
| Setup requires a valid Replicate API key available at https://replicate.com/account/api-keys          | Users must create an account and generate their API key from Replicate’s website.                  |
| The input parameters in the JSON body may need adjustment to match image generation model expectations. | Some parameters resemble language model tokens and may not affect image generation or cause errors. |
| Polling mechanism uses a fixed 2-second delay; consider increasing to reduce API load or handling errors. | To avoid rate limiting or excessive requests, adjust wait time or add failure handling.           |
| The workflow currently lacks explicit error handling for prediction failure or cancellation statuses. | Adding error branches for status like `"failed"` or `"canceled"` is recommended for robustness.    |

---

**Disclaimer:** The provided text originates exclusively from an automated n8n workflow. All processing strictly complies with content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.