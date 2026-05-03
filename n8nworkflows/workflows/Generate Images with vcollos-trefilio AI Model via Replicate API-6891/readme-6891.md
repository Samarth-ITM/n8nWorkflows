Generate Images with vcollos/trefilio AI Model via Replicate API

https://n8nworkflows.xyz/workflows/generate-images-with-vcollos-trefilio-ai-model-via-replicate-api-6891


# Generate Images with vcollos/trefilio AI Model via Replicate API

### 1. Workflow Overview

This workflow titled **"Vcollos Trefilio AI Generator"** is designed to generate images or other content using the **vcollos/trefilio** AI model through the Replicate API. It is aimed at users who want to automate the process of submitting prompts to the AI model, monitor the generation status, and retrieve the final output once ready.

The workflow is logically divided into the following functional blocks:

- **1.1 Input Trigger and API Key Setup:** Manual initiation and setting the Replicate API authentication key.
- **1.2 Prediction Creation:** Sending a request to Replicate API to create a prediction job with specified parameters.
- **1.3 Prediction Monitoring:** Periodically checking the prediction status until completion.
- **1.4 Result Processing:** Handling the final prediction output once the job succeeds.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Trigger and API Key Setup

- **Overview:**  
  Initiates the workflow manually and sets the necessary API key for authentication with Replicate.

- **Nodes Involved:**  
  - On clicking 'execute' (Manual Trigger)  
  - Set API Key (Set Node)

- **Node Details:**

  - **On clicking 'execute'**  
    - Type: `Manual Trigger`  
    - Role: Starts the workflow upon manual execution.  
    - Configuration: No parameters; default manual trigger.  
    - Inputs: None (trigger node)  
    - Outputs: Connects to "Set API Key" node.  
    - Edge Cases: None typical; user must manually trigger.  

  - **Set API Key**  
    - Type: `Set`  
    - Role: Defines and stores the Replicate API key for use in subsequent HTTP requests.  
    - Configuration: Sets a variable `replicate_api_key` with placeholder `"YOUR_REPLICATE_API_KEY"` (to be replaced by the user's actual API key).  
    - Inputs: From manual trigger node.  
    - Outputs: Connects to "Create Prediction" node.  
    - Edge Cases: Missing or invalid API key will cause authentication failures downstream.

#### 1.2 Prediction Creation

- **Overview:**  
  Sends a POST request to Replicate's prediction endpoint to start image/content generation with specified parameters.

- **Nodes Involved:**  
  - Create Prediction (HTTP Request)  
  - Extract Prediction ID (Code)

- **Node Details:**

  - **Create Prediction**  
    - Type: `HTTP Request`  
    - Role: Submits a prediction creation request to Replicate API.  
    - Configuration:  
      - Method: POST  
      - URL: `https://api.replicate.com/v1/predictions`  
      - Body (JSON): Contains model version ID and input parameters (`prompt`, `seed`, `width`, `height`, `lora_scale`). Notably, `prompt` is set as `"prompt value"` placeholder for user input.  
      - Headers: Authorization with Bearer token dynamically set from the API key stored in "Set API Key" node; Content-Type: application/json.  
      - Timeout: 60 seconds  
    - Inputs: From "Set API Key"  
    - Outputs: To "Extract Prediction ID"  
    - Edge Cases:  
      - Authentication failure (invalid/missing API key)  
      - API timeout or connectivity issues  
      - Invalid input parameters causing API rejection  

  - **Extract Prediction ID**  
    - Type: `Code` (JavaScript)  
    - Role: Parses the response from the prediction creation to extract the prediction ID and initial status, constructs the URL for status polling.  
    - Configuration: Runs once per incoming item; extracts `id` and `status` from JSON response; returns prediction ID, status, and URL for polling.  
    - Inputs: From "Create Prediction"  
    - Outputs: To "Wait" node  
    - Edge Cases:  
      - Unexpected or malformed API response  
      - Missing `id` or `status` fields  

#### 1.3 Prediction Monitoring

- **Overview:**  
  Implements a polling loop to wait and repeatedly check the status of the prediction until it is complete.

- **Nodes Involved:**  
  - Wait (Wait node)  
  - Check Prediction Status (HTTP Request)  
  - Check If Complete (IF node)

- **Node Details:**

  - **Wait**  
    - Type: `Wait`  
    - Role: Pauses workflow execution for 2 seconds between polling attempts.  
    - Configuration: Wait period set to 2 seconds.  
    - Inputs: From "Extract Prediction ID" initially, and from "Check If Complete" if prediction is not complete.  
    - Outputs: To "Check Prediction Status"  
    - Edge Cases: None critical; excessive delays could slow workflow responsiveness.

  - **Check Prediction Status**  
    - Type: `HTTP Request`  
    - Role: Queries the Replicate API for the current status of the prediction job using the prediction URL.  
    - Configuration:  
      - Method: GET (default)  
      - URL: dynamically set to the prediction status URL extracted previously.  
      - Headers: Authorization Bearer token set from "Set API Key".  
    - Inputs: From "Wait"  
    - Outputs: To "Check If Complete"  
    - Edge Cases:  
      - Authentication issues  
      - API rate limits or errors  
      - Network failures  

  - **Check If Complete**  
    - Type: `IF`  
    - Role: Evaluates if the prediction status equals `"succeeded"`.  
    - Configuration: Boolean condition comparing `$json.status == "succeeded"`  
    - Inputs: From "Check Prediction Status"  
    - Outputs:  
      - True branch to "Process Result"  
      - False branch back to "Wait" for another polling cycle  
    - Edge Cases:  
      - Status values other than `succeeded` or expected failure states  
      - Potential infinite loop if status never changes  

#### 1.4 Result Processing

- **Overview:**  
  Finalizes the workflow by processing and formatting the prediction output once the job is complete.

- **Nodes Involved:**  
  - Process Result (Code)

- **Node Details:**

  - **Process Result**  
    - Type: `Code` (JavaScript)  
    - Role: Extracts relevant information from the completed prediction, including status, output URLs, metrics, timestamps, and model identifier, formatting it for downstream use or output.  
    - Configuration: Runs once per item; returns a structured object with fields: `status`, `output` (likely URLs to generated images), `metrics`, `created_at`, `completed_at`, `model` (hardcoded), and `other_url` (duplicate of output).  
    - Inputs: From "Check If Complete" (true branch)  
    - Outputs: Workflow end (no further nodes)  
    - Edge Cases:  
      - Missing or incomplete output data  
      - Unexpected API response changes  

---

### 3. Summary Table

| Node Name             | Node Type       | Functional Role              | Input Node(s)        | Output Node(s)         | Sticky Note                                                                                     |
|-----------------------|-----------------|-----------------------------|----------------------|------------------------|------------------------------------------------------------------------------------------------|
| On clicking 'execute'  | Manual Trigger  | Initiates workflow manually | None                 | Set API Key            |                                                                                                |
| Set API Key           | Set             | Stores Replicate API key     | On clicking 'execute' | Create Prediction      |                                                                                                |
| Create Prediction     | HTTP Request    | Submits prediction request  | Set API Key          | Extract Prediction ID  |                                                                                                |
| Extract Prediction ID | Code            | Extracts prediction ID & URL| Create Prediction    | Wait                   |                                                                                                |
| Wait                   | Wait            | Waits 2 seconds between polls| Extract Prediction ID, Check If Complete (false branch) | Check Prediction Status |                                                                                                |
| Check Prediction Status| HTTP Request    | Polls prediction status     | Wait                  | Check If Complete      |                                                                                                |
| Check If Complete      | IF              | Checks if prediction finished| Check Prediction Status| Process Result, Wait   |                                                                                                |
| Process Result         | Code            | Processes final prediction  | Check If Complete (true) | None                  |                                                                                                |
| Sticky Note            | Sticky Note     | Informational note          | None                  | None                   | ## Vcollos Trefilio AI Generator\n\nThis workflow uses the **vcollos/trefilio** model from Replicate to generate other content.\n\n### Setup\n1. Add your Replicate API key\n2. Configure the input parameters\n3. Run the workflow\n\n### Model Details\n- **Type**: Other Generation\n- **Provider**: vcollos\n- **Required Fields**: prompt |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: `On clicking 'execute'`  
   - Type: Manual Trigger  
   - No special configuration required.

2. **Create Set Node for API Key**  
   - Name: `Set API Key`  
   - Type: Set  
   - Add a string field: `replicate_api_key`  
   - Set value to your Replicate API key (replace `"YOUR_REPLICATE_API_KEY"` placeholder).  
   - Connect output from Manual Trigger.

3. **Create HTTP Request Node to Create Prediction**  
   - Name: `Create Prediction`  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.replicate.com/v1/predictions`  
   - Authentication: Generic HTTP Header Authentication  
     - Header Name: `Authorization`  
     - Header Value: `=Bearer {{$node["Set API Key"].json["replicate_api_key"]}}`  
   - Headers: Add `Content-Type: application/json`  
   - Body Content (JSON):  
     ```json
     {
       "version": "e1c19d7267f444e2796438803a31ccd8a1b62ef573531e35257032469e054f1a",
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
   - Connect output from `Set API Key`.

4. **Create Code Node to Extract Prediction ID**  
   - Name: `Extract Prediction ID`  
   - Type: Code (JavaScript)  
   - Code (run once per item):  
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
   - Connect output from `Create Prediction`.

5. **Create Wait Node**  
   - Name: `Wait`  
   - Type: Wait  
   - Set to wait for `2` seconds.  
   - Connect output from `Extract Prediction ID`.

6. **Create HTTP Request Node to Check Prediction Status**  
   - Name: `Check Prediction Status`  
   - Type: HTTP Request  
   - Method: GET (default)  
   - URL: `={{ $json["predictionUrl"] }}` (dynamic from previous node)  
   - Authentication: Generic HTTP Header Authentication  
     - Header Name: `Authorization`  
     - Header Value: `=Bearer {{$node["Set API Key"].json["replicate_api_key"]}}`  
   - Connect output from `Wait`.

7. **Create IF Node to Check Completion**  
   - Name: `Check If Complete`  
   - Type: IF  
   - Condition: Boolean  
     - Expression: `={{ $json["status"] === "succeeded" }}` (checks if status equals "succeeded")  
   - Connect output from `Check Prediction Status`.

8. **Create Code Node to Process Result**  
   - Name: `Process Result`  
   - Type: Code (JavaScript)  
   - Code:  
     ```js
     const result = $input.item.json;
     return {
       status: result.status,
       output: result.output,
       metrics: result.metrics,
       created_at: result.created_at,
       completed_at: result.completed_at,
       model: 'vcollos/trefilio',
       other_url: result.output
     };
     ```  
   - Connect the **true** output of `Check If Complete` to this node.

9. **Connect False Branch of IF Node Back to Wait Node**  
   - This creates a polling loop until the prediction completes.

10. **Add a Sticky Note for Documentation (optional)**  
    - Content:  
      ```
      ## Vcollos Trefilio AI Generator

      This workflow uses the **vcollos/trefilio** model from Replicate to generate other content.

      ### Setup
      1. Add your Replicate API key
      2. Configure the input parameters
      3. Run the workflow

      ### Model Details
      - Type: Other Generation
      - Provider: vcollos
      - Required Fields: prompt
      ```

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                          | Context or Link                           |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------|
| This workflow uses the Replicate API to interface with the vcollos/trefilio model for AI content generation. Ensure you have an active Replicate account and valid API key.                                                                            | Replicate API main site: https://replicate.com/ |
| The model version ID (`e1c19d7267f444e2796438803a31ccd8a1b62ef573531e35257032469e054f1a`) is fixed and specific to the vcollos/trefilio model version. Update only if model is upgraded.                                                               | Replicate model versioning documentation |
| Prompt parameter is mandatory for meaningful generation. Adjust other input parameters (`seed`, `width`, `height`, `lora_scale`) as needed per model documentation.                                                                                   | Model input parameter guidelines         |
| The workflow implements a simple polling mechanism with a 2-second wait interval to check prediction status. For longer-running jobs, consider increasing wait time or implementing exponential backoff to reduce API load and avoid rate limits.      | Polling best practices                    |
| Errors such as authentication failure, API timeout, or unexpected response formats should be handled externally or by adding additional error handling nodes if needed.                                                                                 | Error handling recommendations           |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly respects content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.