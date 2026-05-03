Generate Images from Text Prompts with Replicate's Flux Schnell AI Model

https://n8nworkflows.xyz/workflows/generate-images-from-text-prompts-with-replicate-s-flux-schnell-ai-model-7120


# Generate Images from Text Prompts with Replicate's Flux Schnell AI Model

### 1. Workflow Overview

This workflow, titled **Prunaai Flux Schnell Image Generator**, is designed to generate images from text prompts using the Replicate platform’s **prunaai/flux-schnell** AI model. It is intended for users who want to automate image creation based on descriptive text inputs, leveraging an external AI service.

The workflow is logically divided into the following blocks:

- **1.1 Manual Trigger & API Key Setup:** Initiates the workflow manually and sets the Replicate API key for authentication.
- **1.2 Create Prediction Request:** Sends a request to the Replicate API to start an image generation prediction based on a prompt.
- **1.3 Prediction ID Extraction & Polling:** Extracts the prediction ID, waits for the prediction to process, and polls the API to check the prediction status.
- **1.4 Completion Check & Result Processing:** Determines if the prediction has succeeded, processes the image URL output, and makes it available for downstream use.

This workflow handles asynchronous prediction processing by polling the Replicate API until the image generation completes.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Manual Trigger & API Key Setup

- **Overview:**  
  This block starts the workflow upon manual execution and sets the required API key to authenticate requests to Replicate.

- **Nodes Involved:**  
  - On clicking 'execute' (Manual Trigger)  
  - Set API Key (Set node)

- **Node Details:**

  - **On clicking 'execute'**  
    - Type: Manual Trigger  
    - Role: Starts workflow execution manually.  
    - Configuration: No parameters needed, awaits user execution.  
    - Input: None  
    - Output: Triggers "Set API Key" node.  
    - Edge Cases: User must manually trigger; no automatic execution.

  - **Set API Key**  
    - Type: Set  
    - Role: Stores the Replicate API key as a workflow variable for authentication.  
    - Configuration: Assigns a string variable `replicate_api_key` with a placeholder value `"YOUR_REPLICATE_API_KEY"`.  
    - Input: Triggered from Manual Trigger node.  
    - Output: Feeds the API key to the next HTTP request node.  
    - Edge Cases: Missing or invalid API key will cause authentication failures downstream.

#### Block 1.2: Create Prediction Request

- **Overview:**  
  Sends a POST request to Replicate’s API to create an image generation prediction based on the provided prompt and parameters.

- **Nodes Involved:**  
  - Create Prediction (HTTP Request)

- **Node Details:**

  - **Create Prediction**  
    - Type: HTTP Request  
    - Role: Initiates a new prediction on the Replicate platform.  
    - Configuration:  
      - URL: `https://api.replicate.com/v1/predictions`  
      - Method: POST  
      - Body (JSON):  
        ```json
        {
          "version": "8954d56c83d4db1abf8e701e86c5c9faf231e76a7469821428746972ed75fddb",
          "input": {
            "prompt": "prompt value",
            "seed": 1,
            "num_outputs": 1,
            "output_quality": 80,
            "num_inference_steps": 4
          }
        }
        ```  
      - Authentication: Uses an HTTP header with `Authorization: Bearer <replicate_api_key>`  
      - Timeout: 60 seconds  
    - Expressions:  
      - Authorization header dynamically uses the API key from "Set API Key" node:  
        `={{ 'Bearer ' + $('Set API Key').item.json.replicate_api_key }}`  
    - Input: Receives API key from previous node.  
    - Output: Response JSON containing prediction details (including ID and status).  
    - Edge Cases:  
      - Network timeouts or API rate limits.  
      - Invalid prompt or malformed request body.  
      - Authentication failures if API key is missing or invalid.

#### Block 1.3: Prediction ID Extraction & Polling

- **Overview:**  
  Extracts the prediction ID from the creation response, then waits and polls the Replicate API repeatedly until the prediction is complete.

- **Nodes Involved:**  
  - Extract Prediction ID (Code)  
  - Wait  
  - Check Prediction Status (HTTP Request)  
  - Check If Complete (If)

- **Node Details:**

  - **Extract Prediction ID**  
    - Type: Code (JavaScript)  
    - Role: Retrieves the prediction ID and initial status from the API response to use for subsequent status checks.  
    - Code Summary:  
      Extracts `id` and `status` from input JSON, constructs a URL to poll prediction status, returns these values.  
    - Input: Response from "Create Prediction" node.  
    - Output: Object containing `predictionId`, `status`, and `predictionUrl`.  
    - Edge Cases:  
      - Missing or malformed prediction ID in the API response.  
      - Unexpected response structure.

  - **Wait**  
    - Type: Wait  
    - Role: Pauses execution for 2 seconds between status checks to avoid excessive API calls.  
    - Configuration: Wait for 2 seconds.  
    - Input: From "Extract Prediction ID" or "Check If Complete" node (if not complete).  
    - Output: Triggers "Check Prediction Status" node after delay.  
    - Edge Cases: Fixed wait time may not be optimal for very fast or slow predictions.

  - **Check Prediction Status**  
    - Type: HTTP Request  
    - Role: Queries the Replicate API to get the current status of the prediction.  
    - Configuration:  
      - URL: Dynamic, from `predictionUrl` field (e.g., `https://api.replicate.com/v1/predictions/<id>`)  
      - Method: GET (default)  
      - Authentication: HTTP header with Bearer token same as "Create Prediction"  
    - Input: Wait node output.  
    - Output: Prediction status JSON.  
    - Edge Cases:  
      - Network failures or timeouts.  
      - API rate limits.  
      - Prediction ID invalid or expired.

  - **Check If Complete**  
    - Type: If  
    - Role: Checks if the prediction status is `"succeeded"` to branch flow.  
    - Condition: Boolean comparison between `status` field and `"succeeded"`.  
    - Input: From "Check Prediction Status" node.  
    - Outputs:  
      - True branch: Proceed to process result.  
      - False branch: Loop back to "Wait" node for another status check.  
    - Edge Cases:  
      - Status values other than `"succeeded"` or `"failed"`.  
      - Potential infinite loop if prediction never completes or fails silently.

#### Block 1.4: Completion Check & Result Processing

- **Overview:**  
  Processes the final prediction results after successful image generation, extracting relevant metadata and output image URL.

- **Nodes Involved:**  
  - Process Result (Code)

- **Node Details:**

  - **Process Result**  
    - Type: Code (JavaScript)  
    - Role: Extracts and formats the final prediction output for downstream use or display.  
    - Code Summary:  
      Extracts fields such as `status`, `output` (image URL), `metrics`, `created_at`, `completed_at`, and sets a model identifier string. Returns these as output JSON.  
    - Input: Prediction JSON with status `"succeeded"`.  
    - Output: Simplified JSON including `image_url` for easy access.  
    - Edge Cases:  
      - Missing or empty output array.  
      - Unexpected API response format.

---

### 3. Summary Table

| Node Name              | Node Type          | Functional Role                         | Input Node(s)                 | Output Node(s)                 | Sticky Note                                                                                              |
|------------------------|--------------------|---------------------------------------|------------------------------|-------------------------------|---------------------------------------------------------------------------------------------------------|
| On clicking 'execute'   | Manual Trigger     | Starts workflow manually               | —                            | Set API Key                   |                                                                                                         |
| Set API Key            | Set                | Stores Replicate API key               | On clicking 'execute'          | Create Prediction             |                                                                                                         |
| Create Prediction      | HTTP Request       | Sends prediction request to Replicate | Set API Key                   | Extract Prediction ID         |                                                                                                         |
| Extract Prediction ID  | Code               | Extracts prediction ID and status      | Create Prediction             | Wait                         |                                                                                                         |
| Wait                   | Wait               | Delays before polling prediction status| Extract Prediction ID / Check If Complete (false branch) | Check Prediction Status       |                                                                                                         |
| Check Prediction Status| HTTP Request       | Polls Replicate API for prediction status| Wait                         | Check If Complete             |                                                                                                         |
| Check If Complete      | If                 | Checks if prediction succeeded         | Check Prediction Status       | Process Result (true), Wait (false) |                                                                                                         |
| Process Result         | Code               | Processes final prediction output      | Check If Complete (true)      | —                             |                                                                                                         |
| Sticky Note            | Sticky Note        | Workflow description and setup notes   | —                            | —                             | ## Prunaai Flux Schnell Image Generator\n\nThis workflow uses the **prunaai/flux-schnell** model from Replicate to generate image content.\n\n### Setup\n1. Add your Replicate API key\n2. Configure the input parameters\n3. Run the workflow\n\n### Model Details\n- **Type**: Image Generation\n- **Provider**: prunaai\n- **Required Fields**: prompt |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a **Manual Trigger** node named `"On clicking 'execute'"`. No special parameters needed.

2. **Add Set Node for API Key**  
   - Add a **Set** node named `"Set API Key"`.  
   - Add a string field called `replicate_api_key`.  
   - Set its value to your Replicate API key (e.g., `"YOUR_REPLICATE_API_KEY"`).  
   - Connect `"On clicking 'execute'"` → `"Set API Key"`.

3. **Create HTTP Request Node to Create Prediction**  
   - Add an **HTTP Request** node named `"Create Prediction"`.  
   - Set method: `POST`.  
   - Set URL: `https://api.replicate.com/v1/predictions`.  
   - Under **Authentication**, choose **HTTP Header Auth**.  
     - Header Name: `Authorization`.  
     - Header Value: Use an expression: `Bearer {{$node["Set API Key"].json["replicate_api_key"]}}`.  
   - Set header `Content-Type` to `application/json`.  
   - Set body type to JSON and input the following body, replacing `"prompt value"` with your prompt string or a parameter:  
     ```json
     {
       "version": "8954d56c83d4db1abf8e701e86c5c9faf231e76a7469821428746972ed75fddb",
       "input": {
         "prompt": "prompt value",
         "seed": 1,
         "num_outputs": 1,
         "output_quality": 80,
         "num_inference_steps": 4
       }
     }
     ```  
   - Connect `"Set API Key"` → `"Create Prediction"`.

4. **Add Code Node to Extract Prediction ID**  
   - Add a **Code** node named `"Extract Prediction ID"`.  
   - Set mode to run once per item.  
   - Paste the following JavaScript code:  
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
   - Connect `"Create Prediction"` → `"Extract Prediction ID"`.

5. **Add Wait Node**  
   - Add a **Wait** node named `"Wait"`.  
   - Set wait time to 2 seconds.  
   - Connect `"Extract Prediction ID"` → `"Wait"`.

6. **Add HTTP Request Node to Check Prediction Status**  
   - Add an **HTTP Request** node named `"Check Prediction Status"`.  
   - Set method: `GET`.  
   - Set URL to expression: `{{$json["predictionUrl"]}}`.  
   - Set authentication same as `"Create Prediction"` (HTTP Header Auth with Bearer token from `"Set API Key"`).  
   - Connect `"Wait"` → `"Check Prediction Status"`.

7. **Add If Node to Check Completion**  
   - Add an **If** node named `"Check If Complete"`.  
   - Set condition to boolean: check if `{{$json["status"]}}` equals `"succeeded"`.  
   - Connect `"Check Prediction Status"` → `"Check If Complete"`.

8. **Add Code Node to Process Result**  
   - Add a **Code** node named `"Process Result"`.  
   - Set mode to run once per item.  
   - Paste this JavaScript code:  
     ```javascript
     const result = $input.item.json;

     return {
       status: result.status,
       output: result.output,
       metrics: result.metrics,
       created_at: result.created_at,
       completed_at: result.completed_at,
       model: 'prunaai/flux-schnell',
       image_url: result.output
     };
     ```  
   - Connect `"Check If Complete"` True branch → `"Process Result"`.

9. **Connect False Branch of If Node Back to Wait Node**  
   - Connect `"Check If Complete"` False branch → `"Wait"` to implement polling loop until prediction completes.

10. **Add Sticky Note (optional)**  
    - Add a **Sticky Note** with the following content for reference:  
      ```
      ## Prunaai Flux Schnell Image Generator

      This workflow uses the **prunaai/flux-schnell** model from Replicate to generate image content.

      ### Setup
      1. Add your Replicate API key
      2. Configure the input parameters
      3. Run the workflow

      ### Model Details
      - **Type**: Image Generation
      - **Provider**: prunaai
      - **Required Fields**: prompt
      ```

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| This workflow uses Replicate’s API and requires a valid API key from https://replicate.com/account. | Replicate API Documentation: https://replicate.com/docs |
| The model version used is `8954d56c83d4db1abf8e701e86c5c9faf231e76a7469821428746972ed75fddb` which identifies the specific prunaai/flux-schnell model version. | Model Version Reference on Replicate |
| Polling interval is fixed at 2 seconds, which balances API usage and responsiveness. Adjust as needed for efficiency. | Performance tuning advice |
| Ensure the prompt value is correctly set in the "Create Prediction" node’s JSON body before execution. | User input requirement |
| The workflow handles asynchronous processing by looping until the prediction status is "succeeded". | Asynchronous API handling pattern |

---

**Disclaimer:** The provided text and workflow originate exclusively from an automated n8n workflow. It complies strictly with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.