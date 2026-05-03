Generate Images with Lemaar Door WM AI Model via Replicate API

https://n8nworkflows.xyz/workflows/generate-images-with-lemaar-door-wm-ai-model-via-replicate-api-6964


# Generate Images with Lemaar Door WM AI Model via Replicate API

### 1. Workflow Overview

This workflow, titled **Creativeathive Lemaar Door Wm AI Generator**, integrates with the Replicate API to generate images or other content using the **creativeathive/lemaar-door-wm** AI model. It is designed to take a prompt as input and request the Replicate service to generate an image, then poll the API until the generation is complete, finally processing and outputting the results.

The workflow is logically divided into the following blocks:

- **1.1 Input Initialization:** Manual trigger and API key assignment.
- **1.2 Prediction Creation:** Sending the generation request to Replicate API.
- **1.3 Prediction Monitoring:** Periodic polling of the API to check prediction status.
- **1.4 Result Processing:** Handling the completed prediction and formatting output.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Initialization

- **Overview:**  
  Starts the workflow manually and sets the Replicate API key for authentication in subsequent requests.

- **Nodes Involved:**  
  - On clicking 'execute' (Manual Trigger)  
  - Set API Key (Set node)

- **Node Details:**

  - **On clicking 'execute'**  
    - Type: Manual Trigger  
    - Role: Entry point to start the workflow manually.  
    - Configuration: No parameters; simply triggers the flow on user action.  
    - Inputs: None  
    - Outputs: Connected to "Set API Key" node  
    - Edge cases: None applicable; user must manually trigger.  

  - **Set API Key**  
    - Type: Set  
    - Role: Stores the Replicate API key as a string variable `replicate_api_key`.  
    - Configuration: Assigns the key `"YOUR_REPLICATE_API_KEY"` (to be replaced by user).  
    - Inputs: From manual trigger node  
    - Outputs: To "Create Prediction" node  
    - Edge cases: If the key is missing or invalid, subsequent API calls will fail with authorization errors.

---

#### 1.2 Prediction Creation

- **Overview:**  
  Sends an HTTP POST request to Replicate’s prediction endpoint to initiate content generation with the specified AI model and parameters.

- **Nodes Involved:**  
  - Create Prediction (HTTP Request)  
  - Extract Prediction ID (Code)

- **Node Details:**

  - **Create Prediction**  
    - Type: HTTP Request  
    - Role: Calls Replicate API to create a new prediction job.  
    - Configuration:  
      - URL: `https://api.replicate.com/v1/predictions`  
      - Method: POST  
      - Authentication: HTTP Header with Bearer token from `replicate_api_key`  
      - Request body (JSON):  
        ```json
        {
          "version": "5c5a8fdfe2e6d822912ac93f526e2d6ee9152ae5a9471032fb8d7a84a9ea607e",
          "input": {
            "prompt": "prompt value",
            "seed": 1,
            "width": 1,
            "height": 1,
            "lora_scale": 1
          }
        }
        ```  
        *Note:* The `"prompt"` field should be replaced with the actual prompt string to generate the image. Other parameters are set to default values (seed, width, height, lora_scale).  
      - Timeout: 60 seconds  
    - Inputs: From "Set API Key" node  
    - Outputs: To "Extract Prediction ID" node  
    - Edge cases:  
      - Invalid or missing API key results in 401 Unauthorized errors.  
      - Invalid prompt or model version causes API errors.  
      - Timeout if API takes too long to respond.  

  - **Extract Prediction ID**  
    - Type: Code (JavaScript)  
    - Role: Parses the response from the prediction creation to extract the prediction ID and initial status for polling.  
    - Configuration:  
      - Extracts `id` and `status` from the JSON response.  
      - Constructs a URL for the prediction status endpoint using the ID.  
    - Key expressions:  
      ```js
      const prediction = $input.item.json;
      return {
        predictionId: prediction.id,
        status: prediction.status,
        predictionUrl: `https://api.replicate.com/v1/predictions/${prediction.id}`
      };
      ```  
    - Inputs: From "Create Prediction" node  
    - Outputs: To "Wait" node  
    - Edge cases:  
      - Missing or malformed response JSON may cause script failure.  
      - No prediction ID means subsequent polling cannot proceed.

---

#### 1.3 Prediction Monitoring

- **Overview:**  
  Implements a polling mechanism that waits between requests and queries the prediction status until the generation is complete.

- **Nodes Involved:**  
  - Wait (Wait)  
  - Check Prediction Status (HTTP Request)  
  - Check If Complete (IF)

- **Node Details:**

  - **Wait**  
    - Type: Wait  
    - Role: Pauses the workflow for 2 seconds between polling attempts to avoid spamming the API.  
    - Configuration: Waits 2 seconds (unit: seconds).  
    - Inputs: From "Extract Prediction ID" or "Check If Complete" (false branch)  
    - Outputs: To "Check Prediction Status" node  
    - Edge cases: None significant, but short wait intervals may cause API rate limits.

  - **Check Prediction Status**  
    - Type: HTTP Request  
    - Role: Sends GET requests to Replicate API to retrieve current status of the prediction.  
    - Configuration:  
      - URL: Dynamic, from `predictionUrl` extracted earlier.  
      - Method: GET (default)  
      - Authentication: HTTP Bearer token via header with `replicate_api_key`.  
    - Inputs: From "Wait" node  
    - Outputs: To "Check If Complete" node  
    - Edge cases:  
      - API errors or network issues may cause failures or invalid JSON.  
      - Authorization errors if API key is invalid or expired.  

  - **Check If Complete**  
    - Type: IF  
    - Role: Evaluates if the prediction status equals `"succeeded"`.  
    - Configuration: Boolean check `$json.status === "succeeded"`  
    - Inputs: From "Check Prediction Status" node  
    - Outputs:  
      - True branch to "Process Result" node (generation complete)  
      - False branch loops back to "Wait" node (continue polling)  
    - Edge cases:  
      - If prediction enters a failed or canceled state, this condition will never be true, causing infinite polling unless handled externally.

---

#### 1.4 Result Processing

- **Overview:**  
  Once the prediction is successful, processes and formats the output data for downstream use or display.

- **Nodes Involved:**  
  - Process Result (Code)

- **Node Details:**

  - **Process Result**  
    - Type: Code (JavaScript)  
    - Role: Extracts and formats relevant data from the final prediction response.  
    - Configuration:  
      - Extracts fields: `status`, `output`, `metrics`, `created_at`, `completed_at`.  
      - Adds static `"model"` field as `"creativeathive/lemaar-door-wm"`.  
      - Passes along the output URL(s) for generated content.  
    - Key expressions:  
      ```js
      const result = $input.item.json;
      return {
        status: result.status,
        output: result.output,
        metrics: result.metrics,
        created_at: result.created_at,
        completed_at: result.completed_at,
        model: 'creativeathive/lemaar-door-wm',
        other_url: result.output
      };
      ```  
    - Inputs: From "Check If Complete" node (true branch)  
    - Outputs: Final node output for use elsewhere  
    - Edge cases:  
      - Unexpected API response structure may cause code errors.  
      - Null or empty output indicates generation failure.

---

### 3. Summary Table

| Node Name             | Node Type          | Functional Role                                  | Input Node(s)            | Output Node(s)           | Sticky Note                                                                                                  |
|-----------------------|--------------------|-------------------------------------------------|--------------------------|--------------------------|--------------------------------------------------------------------------------------------------------------|
| On clicking 'execute'  | Manual Trigger     | Entry point to manually start the workflow       | —                        | Set API Key              |                                                                                                              |
| Set API Key           | Set                | Stores Replicate API key for authentication      | On clicking 'execute'     | Create Prediction        |                                                                                                              |
| Create Prediction     | HTTP Request       | Sends prediction creation request to Replicate   | Set API Key               | Extract Prediction ID    |                                                                                                              |
| Extract Prediction ID | Code               | Extracts prediction ID and status from response  | Create Prediction         | Wait                     |                                                                                                              |
| Wait                  | Wait               | Pauses workflow between polling attempts         | Extract Prediction ID, Check If Complete (false) | Check Prediction Status |                                                                                                              |
| Check Prediction Status | HTTP Request     | Polls Replicate API for prediction status update | Wait                      | Check If Complete        |                                                                                                              |
| Check If Complete     | IF                 | Checks if prediction status is 'succeeded'       | Check Prediction Status   | Process Result (true), Wait (false) |                                                                                                              |
| Process Result        | Code               | Formats and outputs final prediction results     | Check If Complete (true)  | —                        |                                                                                                              |
| Sticky Note           | Sticky Note        | Documentation note on workflow and model details | —                        | —                        | ## Creativeathive Lemaar Door Wm AI Generator<br><br>This workflow uses the **creativeathive/lemaar-door-wm** model from Replicate to generate other content.<br><br>### Setup<br>1. Add your Replicate API key<br>2. Configure the input parameters<br>3. Run the workflow<br><br>### Model Details<br>- **Type**: Other Generation<br>- **Provider**: creativeathive<br>- **Required Fields**: prompt |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node**  
   - Type: Manual Trigger  
   - Position it as the entry point of the workflow.

2. **Add a Set Node Named "Set API Key"**  
   - Connect the output of Manual Trigger to this node.  
   - Add a string field named `replicate_api_key`.  
   - Set its value to your Replicate API key string (e.g., `"YOUR_REPLICATE_API_KEY"`).  
   - This will be used for authenticating API requests.

3. **Add an HTTP Request Node Named "Create Prediction"**  
   - Connect the output of "Set API Key" to this node.  
   - Configure as follows:  
     - URL: `https://api.replicate.com/v1/predictions`  
     - Method: POST  
     - Authentication: None in the node config; instead, add a header for Authorization.  
     - Headers:  
       - `Authorization`: `Bearer {{$node["Set API Key"].json["replicate_api_key"]}}`  
       - `Content-Type`: `application/json`  
     - Body Content: JSON (raw) with these fields:  
       ```json
       {
         "version": "5c5a8fdfe2e6d822912ac93f526e2d6ee9152ae5a9471032fb8d7a84a9ea607e",
         "input": {
           "prompt": "prompt value",
           "seed": 1,
           "width": 1,
           "height": 1,
           "lora_scale": 1
         }
       }
       ```  
       Replace `"prompt value"` with the actual prompt text or expression as needed.  
     - Timeout: 60000 ms (60 seconds)

4. **Add a Code Node Named "Extract Prediction ID"**  
   - Connect "Create Prediction" to this node.  
   - Language: JavaScript  
   - Code:  
     ```js
     const prediction = $input.item.json;
     return {
       predictionId: prediction.id,
       status: prediction.status,
       predictionUrl: `https://api.replicate.com/v1/predictions/${prediction.id}`
     };
     ```
   - This extracts the prediction ID and initial status, and constructs the polling URL.

5. **Add a Wait Node Named "Wait"**  
   - Connect "Extract Prediction ID" to this node.  
   - Configure to wait for 2 seconds.

6. **Add an HTTP Request Node Named "Check Prediction Status"**  
   - Connect "Wait" to this node.  
   - Configure as follows:  
     - URL: `{{$json["predictionUrl"]}}` (dynamic expression from previous node)  
     - Method: GET  
     - Headers:  
       - `Authorization`: `Bearer {{$node["Set API Key"].json["replicate_api_key"]}}`  
     - No body needed.

7. **Add an IF Node Named "Check If Complete"**  
   - Connect "Check Prediction Status" to this node.  
   - Condition: Boolean  
     - Expression: Check if `$json["status"] === "succeeded"`

8. **Connect the True Output of "Check If Complete" to a Code Node Named "Process Result"**  
   - Code:  
     ```js
     const result = $input.item.json;
     return {
       status: result.status,
       output: result.output,
       metrics: result.metrics,
       created_at: result.created_at,
       completed_at: result.completed_at,
       model: 'creativeathive/lemaar-door-wm',
       other_url: result.output
     };
     ```
   - This formats and outputs the final successful prediction data.

9. **Connect the False Output of "Check If Complete" back to the "Wait" node**  
   - This creates a polling loop until the prediction succeeds.

10. **Add a Sticky Note for Documentation (Optional)**  
    - Content:  
      ```
      ## Creativeathive Lemaar Door Wm AI Generator

      This workflow uses the **creativeathive/lemaar-door-wm** model from Replicate to generate other content.

      ### Setup
      1. Add your Replicate API key
      2. Configure the input parameters
      3. Run the workflow

      ### Model Details
      - Type: Other Generation
      - Provider: creativeathive
      - Required Fields: prompt
      ```

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                                                   |
|----------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Workflow integrates with Replicate API, which requires an API key obtainable from https://replicate.com/. | Replicate API documentation: https://replicate.com/docs/api                                        |
| The AI model used is `creativeathive/lemaar-door-wm` with version `5c5a8fdfe2e6d822912ac93f526e2d6ee9152ae5a9471032fb8d7a84a9ea607e`. | Model info: https://replicate.com/creativeathive/lemaar-door-wm                                  |
| Polling interval is set to 2 seconds to balance responsiveness and avoid rate limiting.                  | Adjust wait time if API rate limits are encountered.                                             |
| Prompt and input parameters in "Create Prediction" node must be customized as per user requirements.    | Prompt must be a meaningful string for best generation results.                                  |

---

**Disclaimer:**  
The content provided is based solely on an automated n8n workflow. It fully complies with applicable content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.