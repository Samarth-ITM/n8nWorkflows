Generate AI Images with Settyan Flash Model via Replicate

https://n8nworkflows.xyz/workflows/generate-ai-images-with-settyan-flash-model-via-replicate-7109


# Generate AI Images with Settyan Flash Model via Replicate

---

### 1. Workflow Overview

This workflow, titled **Settyan Flash V2.0.0 Beta.0 AI Generator**, is designed to generate AI images or visual content by leveraging the **settyan/flash-v2.0.0-beta.0** model hosted on Replicate. It is intended for users who want to automate image generation tasks using AI models accessible via API.

The workflow is structured into the following logical blocks:

- **1.1 Manual Trigger and API Key Setup:** Start the workflow manually and set the Replicate API key for authentication.
- **1.2 Create Prediction Request:** Send a request to the Replicate API to initiate an AI image generation prediction.
- **1.3 Extract and Poll Prediction Status:** Extract the prediction ID from the response and repeatedly check the prediction status until completion.
- **1.4 Process Completed Prediction Result:** After the prediction succeeds, process and output the generated content details.

---

### 2. Block-by-Block Analysis

#### 2.1 Manual Trigger and API Key Setup

- **Overview:**  
  This block initiates the workflow execution manually and sets the Replicate API key required for authenticating API requests.

- **Nodes Involved:**  
  - On clicking 'execute' (Manual Trigger)  
  - Set API Key (Set)

- **Node Details:**  

  - **On clicking 'execute'**  
    - *Type:* Manual Trigger  
    - *Role:* Entry point for the workflow, allowing manual start.  
    - *Configuration:* No parameters; simply triggers workflow execution.  
    - *Input:* None  
    - *Output:* Triggers the next node "Set API Key".  
    - *Version-specific:* Standard manual trigger node.  
    - *Edge cases:* None expected; user must manually trigger.  

  - **Set API Key**  
    - *Type:* Set  
    - *Role:* Defines and stores the Replicate API key in the workflow context for downstream nodes.  
    - *Configuration:* Assigns a string variable `replicate_api_key` with placeholder `"YOUR_REPLICATE_API_KEY"` (must be replaced with a valid key).  
    - *Expression:* None, static assignment.  
    - *Input:* Triggered by Manual Trigger node.  
    - *Output:* Passes API key variable to "Create Prediction" node.  
    - *Edge cases:* Missing or invalid API key will cause authentication errors downstream.

#### 2.2 Create Prediction Request

- **Overview:**  
  Sends a POST HTTP request to Replicate’s prediction API to create a new AI image generation task.

- **Nodes Involved:**  
  - Create Prediction (HTTP Request)  

- **Node Details:**  

  - **Create Prediction**  
    - *Type:* HTTP Request  
    - *Role:* Calls Replicate API endpoint `/v1/predictions` with model version and input parameters.  
    - *Configuration:*  
      - Method: POST  
      - URL: `https://api.replicate.com/v1/predictions`  
      - Headers:  
        - Authorization: Bearer token dynamically taken from the `replicate_api_key` variable set in previous node.  
        - Content-Type: application/json  
      - Body: JSON specifying:  
        - version: fixed model version `babd558e31787a18bd9add7bbcd7a8913345df389b9470f4d7ff3d84f8fb79c4`  
        - input: contains parameters:  
          - prompt: `"prompt value"` (placeholder, should be replaced with actual prompt)  
          - seed, width, height, lora_scale: all set to 1 (default example values)  
      - Timeout: 60 seconds  
    - *Expressions:* Authorization header uses expression to concatenate Bearer token.  
    - *Input:* Receives API key from "Set API Key" node.  
    - *Output:* JSON response containing prediction ID and initial status forwarded to next node.  
    - *Edge cases:*  
      - Authentication failure if API key invalid.  
      - Timeout possible if API slow.  
      - Input parameter placeholders need to be replaced with realistic values.  

#### 2.3 Extract and Poll Prediction Status

- **Overview:**  
  Extracts the prediction ID from the initial response and polls the Replicate API repeatedly until the prediction completes successfully.

- **Nodes Involved:**  
  - Extract Prediction ID (Code)  
  - Wait (Wait)  
  - Check Prediction Status (HTTP Request)  
  - Check If Complete (If)

- **Node Details:**  

  - **Extract Prediction ID**  
    - *Type:* Code  
    - *Role:* Parses the prediction response to extract the `id`, status, and constructs the URL for subsequent polling.  
    - *Configuration:* Uses JavaScript to extract:  
      - `predictionId = prediction.id`  
      - `status = prediction.status`  
      - `predictionUrl` built as `https://api.replicate.com/v1/predictions/${predictionId}`  
    - *Input:* JSON from "Create Prediction" node.  
    - *Output:* Object containing `predictionId`, `status`, and `predictionUrl` for polling.  
    - *Edge cases:* If response lacks expected fields, extraction fails.  

  - **Wait**  
    - *Type:* Wait  
    - *Role:* Pauses workflow for 2 seconds between polling attempts to avoid rate limits.  
    - *Configuration:* Wait time of 2 seconds.  
    - *Input:* Triggered after extracting prediction ID or if prediction not complete.  
    - *Output:* Triggers next polling request.  
    - *Edge cases:* None critical; wait time can be tuned.  

  - **Check Prediction Status**  
    - *Type:* HTTP Request  
    - *Role:* Requests current prediction status from Replicate API using `predictionUrl`.  
    - *Configuration:*  
      - URL: dynamically set from `predictionUrl` extracted earlier  
      - Method: GET (default)  
      - Header: Authorization bearer token from stored API key  
    - *Input:* Triggered after wait period.  
    - *Output:* Returns updated prediction status JSON.  
    - *Edge cases:* Authentication failure, network errors, or invalid prediction ID.  

  - **Check If Complete**  
    - *Type:* If  
    - *Role:* Checks if prediction status equals `"succeeded"` to determine if polling should continue or proceed to result processing.  
    - *Configuration:* Boolean condition `status == "succeeded"`  
    - *Input:* JSON status from "Check Prediction Status".  
    - *Output:*  
      - If true: proceeds to "Process Result".  
      - If false: loops back to "Wait" to poll again.  
    - *Edge cases:* If prediction fails or is canceled, no handling is defined (could cause infinite loop).  

#### 2.4 Process Completed Prediction Result

- **Overview:**  
  Processes the completed prediction data, extracting useful fields and formatting output for downstream use.

- **Nodes Involved:**  
  - Process Result (Code)

- **Node Details:**  

  - **Process Result**  
    - *Type:* Code  
    - *Role:* Extracts final prediction output including status, output URL(s), metrics, timestamps, and adds model info.  
    - *Configuration:* JavaScript extracts:  
      - `status`  
      - `output` (usually URLs to generated images)  
      - `metrics` (performance data)  
      - `created_at` and `completed_at` timestamps  
      - Static `model` field set to `"settyan/flash-v2.0.0-beta.0"`  
      - `other_url` set to the output value (likely image URL)  
    - *Input:* JSON from successful prediction status node.  
    - *Output:* Structured object with relevant prediction details.  
    - *Edge cases:* If output is empty or malformed, downstream consumers may fail.

---

### 3. Summary Table

| Node Name             | Node Type        | Functional Role                         | Input Node(s)            | Output Node(s)           | Sticky Note                                                                                  |
|-----------------------|------------------|---------------------------------------|--------------------------|--------------------------|----------------------------------------------------------------------------------------------|
| On clicking 'execute'  | Manual Trigger   | Starts workflow manually               | -                        | Set API Key              |                                                                                              |
| Set API Key           | Set              | Stores Replicate API key               | On clicking 'execute'     | Create Prediction        |                                                                                              |
| Create Prediction     | HTTP Request     | Sends prediction creation request     | Set API Key              | Extract Prediction ID    |                                                                                              |
| Extract Prediction ID | Code             | Extracts prediction ID and status     | Create Prediction        | Wait                     |                                                                                              |
| Wait                  | Wait             | Pauses before polling prediction      | Extract Prediction ID, Check If Complete (false) | Check Prediction Status |                                                                                              |
| Check Prediction Status | HTTP Request   | Polls prediction status                | Wait                     | Check If Complete        |                                                                                              |
| Check If Complete     | If               | Checks if prediction succeeded        | Check Prediction Status  | Process Result (true), Wait (false) |                                                                                              |
| Process Result        | Code             | Processes and formats final result    | Check If Complete (true) | -                        |                                                                                              |
| Sticky Note           | Sticky Note      | Notes on workflow and model usage     | -                        | -                        | ## Settyan Flash V2.0.0 Beta.0 AI Generator<br>This workflow uses the **settyan/flash-v2.0.0-beta.0** model from Replicate to generate other content.<br>### Setup<br>1. Add your Replicate API key<br>2. Configure the input parameters<br>3. Run the workflow<br>### Model Details<br>- **Type**: Other Generation<br>- **Provider**: settyan<br>- **Required Fields**: prompt |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a **Manual Trigger** node named `"On clicking 'execute'"`. No special configuration needed.  

2. **Create Set Node to Store API Key**  
   - Add a **Set** node named `"Set API Key"`.  
   - Add a field named `replicate_api_key` as a string.  
   - Set value to your actual Replicate API key (replace placeholder `"YOUR_REPLICATE_API_KEY"`).  
   - Connect `"On clicking 'execute'"` output to this node input.  

3. **Create HTTP Request Node to Create Prediction**  
   - Add an **HTTP Request** node named `"Create Prediction"`.  
   - Configure:  
     - HTTP Method: POST  
     - URL: `https://api.replicate.com/v1/predictions`  
     - Authentication: None (use header auth)  
     - Headers:  
       - `Authorization`: set to expression `Bearer {{$node["Set API Key"].json["replicate_api_key"]}}`  
       - `Content-Type`: `application/json`  
     - Body Content Type: JSON  
     - JSON Body:
       ```json
       {
         "version": "babd558e31787a18bd9add7bbcd7a8913345df389b9470f4d7ff3d84f8fb79c4",
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
   - Connect `"Set API Key"` output to this node input.  

4. **Add Code Node to Extract Prediction ID**  
   - Add a **Code** node named `"Extract Prediction ID"`.  
   - Configure to run once per item with this JavaScript:
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
   - Connect `"Create Prediction"` output to this node.  

5. **Add Wait Node**  
   - Add a **Wait** node named `"Wait"`.  
   - Configure to wait 2 seconds.  
   - Connect `"Extract Prediction ID"` output to this node.  

6. **Add HTTP Request Node to Check Prediction Status**  
   - Add an **HTTP Request** node named `"Check Prediction Status"`.  
   - Configure:  
     - HTTP Method: GET (default)  
     - URL: expression `{{$json["predictionUrl"]}}` (dynamic URL from previous node)  
     - Headers:  
       - `Authorization`: expression `Bearer {{$node["Set API Key"].json["replicate_api_key"]}}`  
   - Connect `"Wait"` output to this node.  

7. **Add If Node to Check If Prediction is Complete**  
   - Add an **If** node named `"Check If Complete"`.  
   - Configure:  
     - Condition: Boolean  
     - Expression: `{{$json["status"]}} == "succeeded"`  
   - Connect `"Check Prediction Status"` output to this node.  

8. **Add Code Node to Process Completed Result**  
   - Add a **Code** node named `"Process Result"`.  
   - Configure with JavaScript to extract and format results:
     ```javascript
     const result = $input.item.json;

     return {
       status: result.status,
       output: result.output,
       metrics: result.metrics,
       created_at: result.created_at,
       completed_at: result.completed_at,
       model: 'settyan/flash-v2.0.0-beta.0',
       other_url: result.output
     };
     ```
   - Connect `"Check If Complete"` true output to this node.  

9. **Loop Back to Wait for Incomplete Predictions**  
   - Connect `"Check If Complete"` false output back to `"Wait"` node to continue polling.  

10. **Add Sticky Note for Documentation** (optional)  
    - Create a **Sticky Note** node near the start with content:
      ```
      ## Settyan Flash V2.0.0 Beta.0 AI Generator

      This workflow uses the **settyan/flash-v2.0.0-beta.0** model from Replicate to generate other content.

      ### Setup
      1. Add your Replicate API key
      2. Configure the input parameters
      3. Run the workflow

      ### Model Details
      - Type: Other Generation
      - Provider: settyan
      - Required Fields: prompt
      ```

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                  | Context or Link                                           |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------|
| This workflow requires a valid Replicate API key with permissions to use the **settyan/flash-v2.0.0-beta.0** model. Replace the placeholder in the "Set API Key" node before running.                                                          | API key setup                                              |
| The polling mechanism uses a fixed 2-second wait between status checks; this should be adjusted if API rate limits or performance needs vary.                                                                                                | Polling configuration                                     |
| The model version is fixed by the version hash; to use a different model or updated version, update the version string in the "Create Prediction" node JSON body.                                                                             | Model versioning                                           |
| If prediction status results in failure or error states, the workflow currently lacks explicit error handling and may loop indefinitely; consider adding error branches or timeouts.                                                           | Error handling improvement suggestion                      |
| Official Replicate API documentation: https://replicate.com/docs/api-reference/predictions#create-prediction                                                                                                                                | API reference                                             |
| The workflow is designed for image generation but can be adapted for other content types by modifying input parameters and processing logic accordingly.                                                                                      | Adaptability note                                         |

---

**Disclaimer:**  
The provided text is exclusively derived from an automated workflow created with n8n, a workflow automation tool. This processing strictly follows applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.

---