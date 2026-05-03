Generate Images with Replicate's Lemaar Doorhandle AI Model

https://n8nworkflows.xyz/workflows/generate-images-with-replicate-s-lemaar-doorhandle-ai-model-7058


# Generate Images with Replicate's Lemaar Doorhandle AI Model

### 1. Workflow Overview

This workflow automates image generation using the Replicate AI model **creativeathive/lemaar-doorhandle-newset**, which specializes in generating doorhandle-related images based on text prompts. Targeted at creative professionals or developers who want to integrate AI image generation into their processes, it handles authentication, prediction creation, status polling, and result processing.

The workflow is logically divided into the following blocks:

- **1.1 Input & Initialization:** Manual trigger and API key setup for Replicate authentication.
- **1.2 Prediction Creation:** Submitting a generation request with user-defined parameters to Replicate.
- **1.3 Status Polling:** Periodically checking the prediction status until completion.
- **1.4 Result Handling:** Extracting and formatting the final output once the generation is successful.

---

### 2. Block-by-Block Analysis

#### 1.1 Input & Initialization

- **Overview:**  
  This block initiates workflow execution manually and sets the Replicate API key for authentication in subsequent requests.

- **Nodes Involved:**  
  - On clicking 'execute' (Manual Trigger)  
  - Set API Key (Set Node)

- **Node Details:**

  - **On clicking 'execute'**  
    - Type: Manual Trigger  
    - Role: Entry point, triggers workflow execution on user command.  
    - Configuration: No parameters; simple manual start.  
    - Inputs: None  
    - Outputs: To "Set API Key" node  
    - Edge Cases: User must manually trigger; no automated start.

  - **Set API Key**  
    - Type: Set Node  
    - Role: Stores Replicate API key in workflow data for authorization headers.  
    - Configuration: Assigns string variable `replicate_api_key` with placeholder `"YOUR_REPLICATE_API_KEY"`.  
    - Inputs: From Manual Trigger  
    - Outputs: To "Create Prediction"  
    - Edge Cases: Missing or invalid API key leads to authorization errors in later HTTP requests.

#### 1.2 Prediction Creation

- **Overview:**  
  Constructs and sends a POST request to the Replicate API to start an image generation prediction with preset parameters.

- **Nodes Involved:**  
  - Create Prediction (HTTP Request)  
  - Extract Prediction ID (Code)

- **Node Details:**

  - **Create Prediction**  
    - Type: HTTP Request  
    - Role: Initiates the AI model prediction on Replicate with provided input parameters.  
    - Configuration:  
      - URL: `https://api.replicate.com/v1/predictions`  
      - Method: POST  
      - Headers:  
        - Authorization: Bearer token from `replicate_api_key` variable  
        - Content-Type: application/json  
      - Body (JSON):  
        ```json
        {
          "version": "1f9fbbe605ef7c2c91e1eb396413073e1166575e55bc7daecfe7706b0180f10c",
          "input": {
            "prompt": "prompt value",
            "seed": 1,
            "width": 1,
            "height": 1,
            "lora_scale": 1
          }
        }
        ```  
      - Timeout: 60000 ms (60 seconds) to accommodate model processing delays.  
      - Authentication: HTTP Header (Bearer token)  
    - Inputs: From "Set API Key"  
    - Outputs: To "Extract Prediction ID"  
    - Edge Cases:  
      - HTTP errors (4xx/5xx) if API key invalid or parameters malformed  
      - Timeout if Replicate API is slow  
      - "prompt" currently hardcoded as `"prompt value"` — requires manual update or parameterization for real use.

  - **Extract Prediction ID**  
    - Type: Code (JavaScript)  
    - Role: Parses response to extract prediction ID and initial status, constructs a URL for polling.  
    - Configuration:  
      - Runs once per item  
      - Extracts `id` and `status` from response JSON  
      - Outputs:  
        - `predictionId`: string  
        - `status`: string  
        - `predictionUrl`: constructed API URL to check status  
    - Inputs: From "Create Prediction"  
    - Outputs: To "Wait" node  
    - Edge Cases:  
      - Fails if response JSON structure changes or missing fields  
      - No fallback if no prediction ID extracted

#### 1.3 Status Polling

- **Overview:**  
  Waits for a short interval, then sends repeated requests to check if the prediction completed successfully.

- **Nodes Involved:**  
  - Wait (Wait Node)  
  - Check Prediction Status (HTTP Request)  
  - Check If Complete (If Node)

- **Node Details:**

  - **Wait**  
    - Type: Wait  
    - Role: Pauses workflow for 2 seconds between polling requests to avoid excessive API calls.  
    - Configuration: Wait 2 seconds  
    - Inputs: From "Extract Prediction ID" and from "Check If Complete" (loop back)  
    - Outputs: To "Check Prediction Status"  
    - Edge Cases:  
      - Fixed wait time may cause delays; no exponential backoff implemented.

  - **Check Prediction Status**  
    - Type: HTTP Request  
    - Role: Fetches current status of prediction using the stored `predictionUrl`.  
    - Configuration:  
      - URL: Dynamic from `$json.predictionUrl`  
      - Method: GET (default)  
      - Headers: Authorization Bearer token from `replicate_api_key`  
    - Inputs: From "Wait"  
    - Outputs: To "Check If Complete"  
    - Edge Cases:  
      - API errors or network issues can interrupt status checking  
      - Missing or expired predictionUrl causes failures

  - **Check If Complete**  
    - Type: If Node  
    - Role: Branches flow based on whether prediction status is `"succeeded"`.  
    - Configuration:  
      - Condition: boolean equality check  
      - Compare `$json.status` with `"succeeded"`  
    - Inputs: From "Check Prediction Status"  
    - Outputs:  
      - True: To "Process Result"  
      - False: Loops back to "Wait" for continued polling  
    - Edge Cases:  
      - Does not handle failure or error status explicitly (e.g., "failed", "canceled")  
      - Could cause infinite loop if prediction never succeeds

#### 1.4 Result Handling

- **Overview:**  
  Processes the successful prediction response to extract relevant data and prepare it for downstream use or output.

- **Nodes Involved:**  
  - Process Result (Code)

- **Node Details:**

  - **Process Result**  
    - Type: Code (JavaScript)  
    - Role: Extracts key response fields such as output URLs, timestamps, and metrics; adds static model identifier.  
    - Configuration:  
      - Runs once per item  
      - Returns object with:  
        - `status`, `output` (likely image URLs), `metrics`, `created_at`, `completed_at`  
        - Static field: `model` set to `"creativeathive/lemaar-doorhandle-newset"`  
        - `other_url`: duplicate of output field for convenience  
    - Inputs: From "Check If Complete" (true branch)  
    - Outputs: End of workflow  
    - Edge Cases:  
      - Assumes output structure is consistent  
      - No error handling if output missing or malformed

---

### 3. Summary Table

| Node Name              | Node Type          | Functional Role                      | Input Node(s)                | Output Node(s)             | Sticky Note                                                                                                                                   |
|------------------------|--------------------|------------------------------------|-----------------------------|----------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------|
| On clicking 'execute'   | Manual Trigger     | Starts the workflow manually       | None                        | Set API Key                |                                                                                                                                               |
| Set API Key            | Set                | Stores Replicate API key           | On clicking 'execute'        | Create Prediction          |                                                                                                                                               |
| Create Prediction      | HTTP Request       | Sends prediction creation request  | Set API Key                 | Extract Prediction ID      |                                                                                                                                               |
| Extract Prediction ID  | Code               | Extracts prediction ID and status  | Create Prediction           | Wait                      |                                                                                                                                               |
| Wait                   | Wait               | Pauses before polling status       | Extract Prediction ID, Check If Complete (false branch) | Check Prediction Status     |                                                                                                                                               |
| Check Prediction Status | HTTP Request       | Retrieves current prediction status| Wait                       | Check If Complete          |                                                                                                                                               |
| Check If Complete      | If                 | Checks if prediction succeeded     | Check Prediction Status     | Process Result (true), Wait (false) |                                                                                                                                               |
| Process Result         | Code               | Extracts and formats final output  | Check If Complete (true)    | None                      |                                                                                                                                               |
| Sticky Note            | Sticky Note        | Explains workflow purpose & setup  | None                        | None                      | ## Creativeathive Lemaar Doorhandle Newset AI Generator<br><br>This workflow uses the **creativeathive/lemaar-doorhandle-newset** model from Replicate to generate other content.<br><br>### Setup<br>1. Add your Replicate API key<br>2. Configure the input parameters<br>3. Run the workflow<br><br>### Model Details<br>- **Type**: Other Generation<br>- **Provider**: creativeathive<br>- **Required Fields**: prompt |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node**  
   - Name: `On clicking 'execute'`  
   - Purpose: To manually start the workflow.

2. **Add a Set node**  
   - Name: `Set API Key`  
   - Connect input from `On clicking 'execute'`.  
   - Add a string field named `replicate_api_key` with value set to your actual Replicate API key (replace `"YOUR_REPLICATE_API_KEY"`).  
   - No special credentials needed here; just storing the key in workflow data.

3. **Add an HTTP Request node**  
   - Name: `Create Prediction`  
   - Connect input from `Set API Key`.  
   - Configure:  
     - Method: POST  
     - URL: `https://api.replicate.com/v1/predictions`  
     - Authentication: None (use HTTP Header authentication manually)  
     - Headers:  
       - Authorization: `Bearer {{$node["Set API Key"].json["replicate_api_key"]}}`  
       - Content-Type: `application/json`  
     - Body Parameters (raw JSON):  
       ```json
       {
         "version": "1f9fbbe605ef7c2c91e1eb396413073e1166575e55bc7daecfe7706b0180f10c",
         "input": {
           "prompt": "prompt value",
           "seed": 1,
           "width": 1,
           "height": 1,
           "lora_scale": 1
         }
       }
       ```  
     - Set “Send Body” and “Send Headers” to true.  
     - Set timeout to 60 seconds.

4. **Add a Code node**  
   - Name: `Extract Prediction ID`  
   - Connect input from `Create Prediction`.  
   - Paste JavaScript code to extract `id` and `status` from response, and build polling URL:  
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

5. **Add a Wait node**  
   - Name: `Wait`  
   - Connect input from `Extract Prediction ID`.  
   - Configure wait for 2 seconds.

6. **Add an HTTP Request node**  
   - Name: `Check Prediction Status`  
   - Connect input from `Wait`.  
   - Configure:  
     - Method: GET (default)  
     - URL: `{{$json["predictionUrl"]}}` (dynamic from previous node output)  
     - Headers:  
       - Authorization: `Bearer {{$node["Set API Key"].json["replicate_api_key"]}}`  
     - Send Headers: true 

7. **Add an If node**  
   - Name: `Check If Complete`  
   - Connect input from `Check Prediction Status`.  
   - Condition:  
     - Boolean check if `$json.status` equals `"succeeded"`.  
   - True branch: Connect to next step (`Process Result`).  
   - False branch: Loop back to `Wait` node for another polling cycle.

8. **Add a Code node**  
   - Name: `Process Result`  
   - Connect input from `Check If Complete` (true branch).  
   - Paste JavaScript code to extract and format useful fields:  
     ```javascript
     const result = $input.item.json;
     return {
       status: result.status,
       output: result.output,
       metrics: result.metrics,
       created_at: result.created_at,
       completed_at: result.completed_at,
       model: 'creativeathive/lemaar-doorhandle-newset',
       other_url: result.output
     };
     ```

9. **Add a Sticky Note** (optional for documentation)  
   - Content:  
     ```
     ## Creativeathive Lemaar Doorhandle Newset AI Generator

     This workflow uses the **creativeathive/lemaar-doorhandle-newset** model from Replicate to generate other content.

     ### Setup
     1. Add your Replicate API key
     2. Configure the input parameters
     3. Run the workflow

     ### Model Details
     - Type: Other Generation
     - Provider: creativeathive
     - Required Fields: prompt
     ```
   - Position near the start nodes for quick reference.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                      |
|---------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| This workflow requires a valid Replicate API key with permission to use the creativeathive/lemaar-doorhandle-newset model. | Replicate API documentation: https://replicate.com/docs/api-reference                            |
| The "prompt" field in the creation request is currently hardcoded; adapt this by adding dynamic input for practical use. | n8n expressions and parameterization guide: https://docs.n8n.io/nodes/expressions/                  |
| Polling interval is fixed at 2 seconds; consider implementing exponential backoff or maximum retries for production use.| Best practices for API polling and error handling                                                  |
| The workflow does not explicitly handle failed or canceled prediction statuses; add error handling for robustness.|                                                                                                    |
| Model version ID used is specific and must be kept up to date if the model updates on Replicate platform.     | Check model version at https://replicate.com/creativeathive/lemaar-doorhandle-newset                |

---

**Disclaimer:**  
The provided text is generated exclusively from an automated n8n workflow. It strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All processed data are legal and public.