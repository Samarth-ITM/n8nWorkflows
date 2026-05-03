Generate Images with Replicate's Nazanin AI Model

https://n8nworkflows.xyz/workflows/generate-images-with-replicate-s-nazanin-ai-model-7115


# Generate Images with Replicate's Nazanin AI Model

### 1. Workflow Overview

This workflow, titled **"Generate Images with Replicate's Nazanin AI Model"**, automates the process of generating images using the AI model **citoreh/nazanin** hosted on the Replicate platform. It is designed for users who want to programmatically create AI-generated images by submitting prompts and polling for the generation status until completion.

The workflow is structured into the following logical blocks:

- **1.1 Input Reception & Initialization:** Trigger the workflow manually and set up the necessary API key for authentication.
- **1.2 Prediction Creation:** Submit an image generation request to Replicate’s API with configured parameters.
- **1.3 Prediction Monitoring:** Poll the prediction status from Replicate until the generation is complete.
- **1.4 Result Processing:** Once the prediction succeeds, extract and format the output data for downstream use or display.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Initialization

- **Overview:** This block starts the workflow manually and sets the Replicate API key required for authenticated requests.
- **Nodes Involved:** 
  - On clicking 'execute'
  - Set API Key

**Node Details:**

- **On clicking 'execute'**
  - Type: Manual Trigger
  - Role: Starts the workflow by manual execution.
  - Configuration: Default manual trigger with no parameters.
  - Inputs: None
  - Outputs: Passes control to the "Set API Key" node.
  - Edge Cases: User forgets to trigger manually; no automatic triggering.

- **Set API Key**
  - Type: Set
  - Role: Stores the Replicate API key as a workflow variable for authentication.
  - Configuration: Assigns a string variable `replicate_api_key` with the value `"YOUR_REPLICATE_API_KEY"`.
  - Inputs: From Manual Trigger
  - Outputs: Supplies API key to HTTP request nodes.
  - Edge Cases:
    - Missing or invalid API key will cause authorization errors downstream.
    - User must replace placeholder with a valid key.
  - Notes: The API key is used in the Authorization header for Replicate API calls.

#### 1.2 Prediction Creation

- **Overview:** This block sends a POST request to Replicate’s API to create an image generation prediction based on input parameters.
- **Nodes Involved:** 
  - Create Prediction
  - Extract Prediction ID

**Node Details:**

- **Create Prediction**
  - Type: HTTP Request
  - Role: Initiates a prediction request to Replicate’s `v1/predictions` endpoint.
  - Configuration:
    - Method: POST
    - URL: `https://api.replicate.com/v1/predictions`
    - Headers: 
      - Authorization: Bearer token from `replicate_api_key`
      - Content-Type: application/json
    - Body (JSON): 
      - `version`: fixed model version hash `"836298cc5e8c69b4b0718c85ae57d7750817e293bb9c8113182dac2fe582cd6f"`
      - `input`: 
        - `prompt`: placeholder `"prompt value"` (should be replaced or parameterized)
        - `seed`, `width`, `height`, `lora_scale`: all set to 1 by default
    - Timeout: 60 seconds
    - Authentication: Generic HTTP Header with Bearer token
  - Inputs: From "Set API Key"
  - Outputs: JSON response with prediction metadata including prediction ID and status
  - Edge Cases:
    - API timeout or network issues
    - Invalid prompt or parameters result in errors
    - Unauthorized requests due to API key issues

- **Extract Prediction ID**
  - Type: Code
  - Role: Parses the JSON response from "Create Prediction" to extract the prediction ID and initial status.
  - Configuration: JavaScript code extracting `id`, `status`, and constructing the prediction status URL.
  - Inputs: From "Create Prediction"
  - Outputs: Object with `predictionId`, `status`, and `predictionUrl`
  - Edge Cases:
    - Unexpected response structure causes code failure.
    - Missing `id` or `status` fields.
  - Notes: Stores the prediction URL for subsequent polling.

#### 1.3 Prediction Monitoring

- **Overview:** This block repeatedly checks the prediction status by polling Replicate’s API until the prediction completes successfully.
- **Nodes Involved:** 
  - Wait
  - Check Prediction Status
  - Check If Complete

**Node Details:**

- **Wait**
  - Type: Wait
  - Role: Pauses the workflow for 2 seconds between polling requests to avoid rate limiting.
  - Configuration: Fixed 2 seconds delay
  - Inputs: From "Extract Prediction ID" or "Check If Complete" (in retry path)
  - Outputs: Passes control to "Check Prediction Status"
  - Edge Cases: None significant; too short or long wait may affect performance or API limits.

- **Check Prediction Status**
  - Type: HTTP Request
  - Role: Queries the prediction status by hitting the prediction URL.
  - Configuration:
    - Method: GET (default for HTTP request)
    - URL: dynamic, set from `predictionUrl` extracted earlier
    - Headers:
      - Authorization: Bearer token from `replicate_api_key`
    - Authentication: Generic HTTP header
  - Inputs: From "Wait"
  - Outputs: JSON response with updated prediction status
  - Edge Cases:
    - Network or API errors
    - Expired or invalid prediction ID
    - Unauthorized errors if key is invalid

- **Check If Complete**
  - Type: If
  - Role: Evaluates if the prediction status is `"succeeded"`.
  - Configuration:
    - Condition: `$json.status === "succeeded"`
  - Inputs: From "Check Prediction Status"
  - Outputs:
    - True branch: proceeds to "Process Result"
    - False branch: loops back to "Wait" to retry polling
  - Edge Cases:
    - Prediction may fail or get stuck in other statuses; no explicit failure handling.
    - Infinite loop possible if status never reaches "succeeded".

#### 1.4 Result Processing

- **Overview:** This block formats and outputs the final prediction result once the generation completes successfully.
- **Nodes Involved:** 
  - Process Result

**Node Details:**

- **Process Result**
  - Type: Code
  - Role: Extracts and structures relevant data from the completed prediction response.
  - Configuration: JavaScript code that returns:
    - `status`
    - `output` (likely the generated image URLs)
    - `metrics` (generation metadata)
    - `created_at` and `completed_at` timestamps
    - Static field `model` set to `"citoreh/nazanin"`
    - `other_url` duplicating output for convenience
  - Inputs: From "Check If Complete" (true branch)
  - Outputs: Clean JSON object with formatted result data
  - Edge Cases:
    - Unexpected structure in prediction output
    - Missing fields in response

---

### 3. Summary Table

| Node Name               | Node Type           | Functional Role                     | Input Node(s)          | Output Node(s)           | Sticky Note                                                                                                 |
|-------------------------|---------------------|-----------------------------------|-----------------------|--------------------------|-------------------------------------------------------------------------------------------------------------|
| On clicking 'execute'    | Manual Trigger      | Initiates workflow manually       | None                  | Set API Key              |                                                                                                             |
| Set API Key             | Set                 | Stores Replicate API key           | On clicking 'execute'  | Create Prediction        |                                                                                                             |
| Create Prediction        | HTTP Request        | Sends prediction creation request | Set API Key            | Extract Prediction ID    |                                                                                                             |
| Extract Prediction ID    | Code                | Extracts prediction ID and status | Create Prediction      | Wait                     |                                                                                                             |
| Wait                    | Wait                | Delays polling to avoid rate limit| Extract Prediction ID, Check If Complete (false branch) | Check Prediction Status |                                                                                                             |
| Check Prediction Status  | HTTP Request        | Polls prediction status            | Wait                   | Check If Complete        |                                                                                                             |
| Check If Complete        | If                  | Checks if prediction succeeded     | Check Prediction Status| Process Result (true), Wait (false) |                                                                                                             |
| Process Result           | Code                | Formats completed prediction data  | Check If Complete      | None                     |                                                                                                             |
| Sticky Note             | Sticky Note         | Workflow information & instructions| None                   | None                     | ## Citoreh Nazanin AI Generator\n\nThis workflow uses the **citoreh/nazanin** model from Replicate to generate other content.\n\n### Setup\n1. Add your Replicate API key\n2. Configure the input parameters\n3. Run the workflow\n\n### Model Details\n- **Type**: Other Generation\n- **Provider**: citoreh\n- **Required Fields**: prompt |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Manual Trigger node:**
   - Name: `On clicking 'execute'`
   - Purpose: Manually start the workflow.

3. **Add a Set node:**
   - Name: `Set API Key`
   - Connect input from `On clicking 'execute'`.
   - In Parameters → Values to Set:
     - Add a string variable: `replicate_api_key`
     - Set value to your Replicate API key (replace `"YOUR_REPLICATE_API_KEY"`).

4. **Add an HTTP Request node:**
   - Name: `Create Prediction`
   - Connect input from `Set API Key`.
   - Settings:
     - HTTP Method: POST
     - URL: `https://api.replicate.com/v1/predictions`
     - Authentication: Generic Credential → HTTP Header Auth
     - Header Parameters:
       - `Authorization`: `Bearer {{$node["Set API Key"].json["replicate_api_key"]}}`
       - `Content-Type`: `application/json`
     - Body Content Type: JSON
     - Request Body JSON:
       ```
       {
         "version": "836298cc5e8c69b4b0718c85ae57d7750817e293bb9c8113182dac2fe582cd6f",
         "input": {
           "prompt": "prompt value",
           "seed": 1,
           "width": 1,
           "height": 1,
           "lora_scale": 1
         }
       }
       ```
     - Timeout: 60000 ms (60 seconds)
   - Note: Replace `"prompt value"` with your desired prompt or parameterize it.

5. **Add a Code node:**
   - Name: `Extract Prediction ID`
   - Connect input from `Create Prediction`.
   - JavaScript code (run once per item):
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
   - Name: `Wait`
   - Connect input from `Extract Prediction ID`.
   - Set wait time to 2 seconds.

7. **Add an HTTP Request node:**
   - Name: `Check Prediction Status`
   - Connect input from `Wait`.
   - Settings:
     - HTTP Method: GET (default)
     - URL: `{{$json["predictionUrl"]}}`
     - Authentication: Generic Credential → HTTP Header Auth
     - Header Parameters:
       - `Authorization`: `Bearer {{$node["Set API Key"].json["replicate_api_key"]}}`

8. **Add an If node:**
   - Name: `Check If Complete`
   - Connect input from `Check Prediction Status`.
   - Condition:
     - Field 1: `{{$json["status"]}}`
     - Operation: equals
     - Field 2: `"succeeded"`
   - True output: connect to next node `Process Result`.
   - False output: connect back to `Wait` node (to continue polling).

9. **Add a Code node:**
   - Name: `Process Result`
   - Connect input from the true output of `Check If Complete`.
   - JavaScript code (run once per item):
     ```javascript
     const result = $input.item.json;

     return {
       status: result.status,
       output: result.output,
       metrics: result.metrics,
       created_at: result.created_at,
       completed_at: result.completed_at,
       model: 'citoreh/nazanin',
       other_url: result.output
     };
     ```

10. **Optional: Add a Sticky Note node** to document workflow purpose and usage:
    - Content:
      ```
      ## Citoreh Nazanin AI Generator

      This workflow uses the **citoreh/nazanin** model from Replicate to generate other content.

      ### Setup
      1. Add your Replicate API key
      2. Configure the input parameters
      3. Run the workflow

      ### Model Details
      - **Type**: Other Generation
      - **Provider**: citoreh
      - **Required Fields**: prompt
      ```

11. **Save and activate the workflow.**

12. **Testing:**
    - Replace `"prompt value"` in `Create Prediction` node with your desired prompt.
    - Manually execute the workflow.
    - Observe the polling until the prediction completes.
    - Review the processed results.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                            | Context or Link               |
|---------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------|
| This workflow relies on the Replicate API; ensure you have a valid API key and network connectivity.                                                    | Replicate API Documentation  |
| The model version string is fixed; check Replicate for updates or new model versions to use.                                                            | https://replicate.com         |
| Polling interval is set to 2 seconds; adjust cautiously to balance API rate limits and responsiveness.                                                  |                              |
| No explicit error handling for failed predictions or API errors; consider adding nodes to handle failure statuses or exceptions in a production setup. |                              |
| The `prompt` input in the HTTP Request node should be parameterized for dynamic use cases.                                                              |                              |

---

*Disclaimer: The provided text is derived solely from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.*