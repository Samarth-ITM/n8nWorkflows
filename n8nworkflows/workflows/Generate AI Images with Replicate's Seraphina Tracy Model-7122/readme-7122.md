Generate AI Images with Replicate's Seraphina Tracy Model

https://n8nworkflows.xyz/workflows/generate-ai-images-with-replicate-s-seraphina-tracy-model-7122


# Generate AI Images with Replicate's Seraphina Tracy Model

### 1. Workflow Overview

This workflow, titled **Seraphina Design Tracy AI Generator**, automates the generation of AI images using the **seraphina-design/tracy** model from the Replicate platform. It is designed to take a textual prompt as input, send it to the Replicate API to trigger an image generation prediction, poll for the prediction status until completion, and then process and output the final generated image data.

The workflow is logically divided into the following blocks:

- **1.1 Trigger and Initialization:** Starts the workflow manually and sets up the API key required for authentication with Replicate.
- **1.2 Prediction Creation:** Sends a POST request to Replicate to create an image generation prediction with specified parameters.
- **1.3 Prediction Polling:** Extracts the prediction ID from the response, then repeatedly checks the prediction status with short delays until the generation is complete.
- **1.4 Result Processing:** Once the prediction is successful, processes and formats the output data for downstream use or storage.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger and Initialization

**Overview:**  
This block initiates the workflow manually and prepares the authentication credentials needed for API interaction with Replicate.

**Nodes Involved:**  
- On clicking 'execute'  
- Set API Key

**Node Details:**

- **On clicking 'execute'**  
  - Type: Manual Trigger  
  - Role: Starts the workflow execution when the user triggers it manually in n8n.  
  - Configuration: Default manual trigger, no parameters.  
  - Inputs: None  
  - Outputs: Connected to "Set API Key"  
  - Edge Cases: None; manual trigger requires user action.

- **Set API Key**  
  - Type: Set  
  - Role: Assigns the Replicate API key as a string variable to be used in subsequent HTTP requests.  
  - Configuration: Sets variable `replicate_api_key` with placeholder `"YOUR_REPLICATE_API_KEY"` which must be replaced by the user.  
  - Inputs: From manual trigger  
  - Outputs: Provides API key to "Create Prediction" node  
  - Edge Cases: Missing or incorrect API key will cause authentication failures in API calls.

#### 1.2 Prediction Creation

**Overview:**  
This block sends a request to the Replicate API to create a new image generation prediction using the specified model version and input parameters.

**Nodes Involved:**  
- Create Prediction  
- Extract Prediction ID

**Node Details:**

- **Create Prediction**  
  - Type: HTTP Request  
  - Role: Calls Replicate's `/v1/predictions` endpoint with POST method to start the image generation.  
  - Configuration:  
    - URL: `https://api.replicate.com/v1/predictions`  
    - Authentication: HTTP header with Bearer token using `replicate_api_key`  
    - Headers: `Authorization: Bearer <API_KEY>`, `Content-Type: application/json`  
    - JSON Body:  
      ```json
      {
        "version": "45424887eb6a8a9a55d375bfd65ed7de44864212a2d4923f5c6cb7afeb06fd8f",
        "input": {
          "prompt": "prompt value",
          "seed": 1,
          "width": 1,
          "height": 1,
          "lora_scale": 1
        }
      }
      ```
    - Notes: The prompt and other input parameters are currently hardcoded and should be replaced or parameterized for dynamic use.  
  - Inputs: From "Set API Key"  
  - Outputs: JSON response with prediction info forwarded to "Extract Prediction ID"  
  - Edge Cases:  
    - API errors due to invalid parameters or authentication failures  
    - Timeout after 60 seconds (configured timeout)

- **Extract Prediction ID**  
  - Type: Code (JavaScript)  
  - Role: Extracts the `id` and `status` from the prediction creation response and formats the URL to poll the prediction status.  
  - Configuration: Runs once per item; JS code extracts `predictionId`, initial `status`, and constructs `predictionUrl`.  
  - Key variables:  
    - `predictionId = $input.item.json.id`  
    - `status = $input.item.json.status`  
    - `predictionUrl = https://api.replicate.com/v1/predictions/${predictionId}`  
  - Inputs: From "Create Prediction"  
  - Outputs: Provides these values to "Wait" node  
  - Edge Cases: Missing or malformed response JSON causing extraction errors.

#### 1.3 Prediction Polling

**Overview:**  
This block waits briefly, then repeatedly polls the Replicate API to check if the prediction has completed, looping until the status is `succeeded`.

**Nodes Involved:**  
- Wait  
- Check Prediction Status  
- Check If Complete

**Node Details:**

- **Wait**  
  - Type: Wait  
  - Role: Pauses the workflow for 2 seconds between polling attempts to avoid excessive API calls.  
  - Configuration: 2 seconds delay  
  - Inputs: From "Extract Prediction ID" and also from "Check If Complete" when incomplete  
  - Outputs: To "Check Prediction Status"  
  - Edge Cases: None significant; delay may be adjusted to balance responsiveness and API rate limits.

- **Check Prediction Status**  
  - Type: HTTP Request  
  - Role: Performs a GET request on the prediction URL to retrieve current prediction status.  
  - Configuration:  
    - URL: dynamically set to `predictionUrl` from previous node  
    - Authentication: Bearer token header using `replicate_api_key`  
    - No timeout override  
  - Inputs: From "Wait"  
  - Outputs: JSON response forwarded to "Check If Complete"  
  - Edge Cases: API errors, network timeouts, invalid prediction ID leading to 404s.

- **Check If Complete**  
  - Type: IF  
  - Role: Checks if the prediction status equals `"succeeded"` to decide whether to continue polling or proceed to processing.  
  - Configuration: Condition: `$json.status == "succeeded"`  
  - Inputs: From "Check Prediction Status"  
  - Outputs:  
    - True branch to "Process Result"  
    - False branch loops back to "Wait" for another polling attempt  
  - Edge Cases:  
    - Status values other than `succeeded` or `failed` handled implicitly; no explicit failure branch, so infinite loop possible if prediction fails without changing status.

#### 1.4 Result Processing

**Overview:**  
This final block processes the completed prediction output, extracting relevant result fields and preparing the data for further use or export.

**Nodes Involved:**  
- Process Result

**Node Details:**

- **Process Result**  
  - Type: Code (JavaScript)  
  - Role: Extracts and formats the prediction result JSON fields such as output image URLs, status, timestamps, and metrics.  
  - Configuration:  
    - Runs once per item  
    - Returns an object with:  
      - `status`  
      - `output` (image URLs or data)  
      - `metrics` (prediction metadata)  
      - `created_at` and `completed_at` timestamps  
      - Fixed `model` name `"seraphina-design/tracy"`  
      - `other_url` duplicating output for convenience  
  - Inputs: From "Check If Complete" (True branch)  
  - Outputs: Final structured data for downstream consumption  
  - Edge Cases: Missing fields if prediction output is incomplete or malformed.

---

### 3. Summary Table

| Node Name             | Node Type          | Functional Role                          | Input Node(s)           | Output Node(s)           | Sticky Note                                                                                         |
|-----------------------|--------------------|----------------------------------------|------------------------|--------------------------|---------------------------------------------------------------------------------------------------|
| On clicking 'execute'  | Manual Trigger     | Starts workflow manually                 | None                   | Set API Key              |                                                                                                   |
| Set API Key           | Set                | Stores Replicate API key for auth        | On clicking 'execute'  | Create Prediction         |                                                                                                   |
| Create Prediction     | HTTP Request       | Sends POST to Replicate to create image prediction | Set API Key             | Extract Prediction ID     |                                                                                                   |
| Extract Prediction ID | Code (JavaScript)  | Extracts prediction ID and status         | Create Prediction       | Wait                     |                                                                                                   |
| Wait                  | Wait               | Delays 2 seconds between polling          | Extract Prediction ID, Check If Complete (false) | Check Prediction Status |                                                                                                   |
| Check Prediction Status | HTTP Request       | Checks current status of prediction        | Wait                    | Check If Complete         |                                                                                                   |
| Check If Complete     | IF                 | Branches workflow based on prediction status | Check Prediction Status | Process Result (true), Wait (false) |                                                                                                   |
| Process Result        | Code (JavaScript)  | Extracts and formats final prediction data | Check If Complete (true) | None                     |                                                                                                   |
| Sticky Note           | Sticky Note        | Provides setup instructions and model info | None                   | None                     | **Seraphina Design Tracy AI Generator**<br>This workflow uses the **seraphina-design/tracy** model from Replicate to generate content.<br>1. Add your Replicate API key<br>2. Configure input parameters<br>3. Run the workflow<br>Model details:<br>- Type: Other Generation<br>- Provider: seraphina-design<br>- Required Fields: prompt |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Name: `On clicking 'execute'`  
   - Purpose: Start workflow manually.

2. **Create Set Node:**  
   - Name: `Set API Key`  
   - Add a string field named `replicate_api_key` with value `"YOUR_REPLICATE_API_KEY"` (replace with actual key).  
   - Connect trigger output to this node.

3. **Create HTTP Request Node:**  
   - Name: `Create Prediction`  
   - Method: POST  
   - URL: `https://api.replicate.com/v1/predictions`  
   - Authentication: Generic HTTP Header Auth  
   - Header Parameters:  
     - `Authorization`: `Bearer {{$node["Set API Key"].json["replicate_api_key"]}}`  
     - `Content-Type`: `application/json`  
   - Body Content Type: JSON  
   - JSON Body:  
     ```json
     {
       "version": "45424887eb6a8a9a55d375bfd65ed7de44864212a2d4923f5c6cb7afeb06fd8f",
       "input": {
         "prompt": "prompt value",
         "seed": 1,
         "width": 1,
         "height": 1,
         "lora_scale": 1
       }
     }
     ```  
   - Connect output of Set API Key to this node.

4. **Create Code Node:**  
   - Name: `Extract Prediction ID`  
   - Mode: Run Once For Each Item  
   - JavaScript Code:  
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
   - Connect output of Create Prediction to this node.

5. **Create Wait Node:**  
   - Name: `Wait`  
   - Wait for 2 seconds.  
   - Connect output of Extract Prediction ID to this node.

6. **Create HTTP Request Node:**  
   - Name: `Check Prediction Status`  
   - Method: GET  
   - URL: Use expression `{{$json["predictionUrl"]}}`  
   - Authentication: Generic HTTP Header Auth  
   - Header Parameters:  
     - `Authorization`: `Bearer {{$node["Set API Key"].json["replicate_api_key"]}}`  
   - Connect output of Wait node to this node.

7. **Create IF Node:**  
   - Name: `Check If Complete`  
   - Condition: Boolean equals  
     - Value 1: `{{$json["status"]}}`  
     - Value 2: `"succeeded"`  
   - Connect output of Check Prediction Status to this node.

8. **Create Code Node:**  
   - Name: `Process Result`  
   - Mode: Run Once For Each Item  
   - JavaScript Code:  
     ```javascript
     const result = $input.item.json;

     return {
       status: result.status,
       output: result.output,
       metrics: result.metrics,
       created_at: result.created_at,
       completed_at: result.completed_at,
       model: 'seraphina-design/tracy',
       other_url: result.output
     };
     ```  
   - Connect the True output of the IF node to this node.

9. **Connect False output of IF node back to Wait node** (loop for polling).

10. **Add Sticky Note:**  
    - Content:  
      ```
      ## Seraphina Design Tracy AI Generator

      This workflow uses the **seraphina-design/tracy** model from Replicate to generate other content.

      ### Setup
      1. Add your Replicate API key
      2. Configure the input parameters
      3. Run the workflow

      ### Model Details
      - Type: Other Generation
      - Provider: seraphina-design
      - Required Fields: prompt
      ```  
    - Place near the start of the workflow for user reference.

---

### 5. General Notes & Resources

| Note Content                                                                 | Context or Link                                                 |
|------------------------------------------------------------------------------|----------------------------------------------------------------|
| Replace `"YOUR_REPLICATE_API_KEY"` with a valid Replicate API key before use. | Replicate API authentication requirement                        |
| Input parameters such as `prompt`, `seed`, `width`, `height`, and `lora_scale` are currently hardcoded and should be parameterized for dynamic input scenarios. | Workflow customization                                           |
| The polling interval is set to 2 seconds; adjust as needed based on API rate limits and expected prediction times. | Performance and API rate limiting considerations                |
| Model version ID used: `45424887eb6a8a9a55d375bfd65ed7de44864212a2d4923f5c6cb7afeb06fd8f` corresponds to the Seraphina Tracy model on Replicate. | Model version identification                                    |
| No explicit error handling is implemented for prediction failure or API errors; consider adding branches for `failed` status or HTTP error codes. | Potential workflow improvement for robustness                   |