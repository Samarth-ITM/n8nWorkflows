Generate AI Images with Replicate's Digitalhera Heranathalie Model

https://n8nworkflows.xyz/workflows/generate-ai-images-with-replicate-s-digitalhera-heranathalie-model-7055


# Generate AI Images with Replicate's Digitalhera Heranathalie Model

### 1. Workflow Overview

This workflow, titled **Digitalhera Heranathalie AI Generator**, facilitates generating AI images using the **digitalhera/heranathalie** model hosted on Replicate. It is designed for users who want to programmatically request image generation based on text prompts and retrieve the results once the AI model completes processing.

The workflow is composed of the following logical blocks:

- **1.1 Input Trigger and API Key Setup**: Initiates the workflow manually and sets the Replicate API key used for authentication.
- **1.2 Prediction Creation**: Sends a request to Replicate’s API to start an image generation prediction with specified parameters.
- **1.3 Prediction Tracking and Polling**: Extracts the prediction ID and repeatedly checks the prediction status until completion.
- **1.4 Result Processing**: Once the prediction is complete, processes the output data for downstream usage or export.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Trigger and API Key Setup

- **Overview:**  
  This block starts the workflow manually and assigns the Replicate API key to a workflow variable for secure authentication in subsequent API calls.

- **Nodes Involved:**  
  - On clicking 'execute'  
  - Set API Key

- **Node Details:**  

  - **On clicking 'execute'**  
    - Type: Manual Trigger  
    - Role: Entry point to manually start the workflow execution.  
    - Configuration: No parameters, triggered manually by user action.  
    - Inputs: None  
    - Outputs: Connected to "Set API Key" node.  
    - Edge cases: None; manual trigger ensures explicit start.  

  - **Set API Key**  
    - Type: Set  
    - Role: Assigns the Replicate API key to a workflow variable `replicate_api_key`.  
    - Configuration: Stores a string variable `replicate_api_key` where users must input their API key.  
    - Inputs: From "On clicking 'execute'"  
    - Outputs: To "Create Prediction"  
    - Edge cases: Missing or invalid API key will cause authentication failures in later HTTP requests.

#### 1.2 Prediction Creation

- **Overview:**  
  Sends an HTTP POST request to Replicate’s Prediction API to initiate the AI image generation with the model version and input parameters.

- **Nodes Involved:**  
  - Create Prediction

- **Node Details:**  

  - **Create Prediction**  
    - Type: HTTP Request  
    - Role: Calls Replicate API to create a new prediction job.  
    - Configuration:  
      - URL: `https://api.replicate.com/v1/predictions`  
      - Method: POST  
      - Body: JSON specifying the model version and input parameters including prompt, seed, width, height, lora_scale. The prompt is inserted dynamically.  
      - Authentication: HTTP header with Bearer token from `replicate_api_key`.  
      - Timeout: 60 seconds  
    - Expressions: Authorization header is constructed as `'Bearer ' + replicate_api_key` from the Set API Key node.  
    - Inputs: From "Set API Key"  
    - Outputs: To "Extract Prediction ID"  
    - Edge cases:  
      - Authentication failure due to invalid API key.  
      - Request timeout or network errors.  
      - Invalid input parameters causing API errors.

#### 1.3 Prediction Tracking and Polling

- **Overview:**  
  Extracts the prediction ID from the creation response, waits a short period, polls the prediction status repeatedly until the prediction is marked as succeeded.

- **Nodes Involved:**  
  - Extract Prediction ID  
  - Wait  
  - Check Prediction Status  
  - Check If Complete

- **Node Details:**  

  - **Extract Prediction ID**  
    - Type: Code  
    - Role: Parses the prediction creation response to extract the prediction ID, initial status, and constructs the URL for status checking.  
    - Configuration: JavaScript code runs once per item, returns an object containing `predictionId`, `status`, and `predictionUrl`.  
    - Inputs: From "Create Prediction"  
    - Outputs: To "Wait"  
    - Edge cases: Unexpected response format causing parsing errors.

  - **Wait**  
    - Type: Wait  
    - Role: Pauses execution for 2 seconds before polling.  
    - Configuration: Fixed delay of 2 seconds.  
    - Inputs: From "Extract Prediction ID" or "Check If Complete" (if prediction not complete)  
    - Outputs: To "Check Prediction Status"  
    - Edge cases: None significant, but short wait time may cause rapid polling.

  - **Check Prediction Status**  
    - Type: HTTP Request  
    - Role: Queries Replicate API for current prediction status.  
    - Configuration:  
      - URL: Dynamically set from `predictionUrl` variable.  
      - Method: GET (default)  
      - Authentication: Bearer token from `replicate_api_key`.  
    - Inputs: From "Wait"  
    - Outputs: To "Check If Complete"  
    - Edge cases:  
      - Authentication failure.  
      - Network errors.  
      - API rate limiting.

  - **Check If Complete**  
    - Type: If  
    - Role: Branches workflow based on whether prediction status is “succeeded”.  
    - Configuration: Checks if `$json.status == 'succeeded'`.  
    - Inputs: From "Check Prediction Status"  
    - Outputs:  
      - True branch: to "Process Result"  
      - False branch: back to "Wait" for another polling cycle  
    - Edge cases:  
      - Other final statuses like “failed” are not explicitly handled; could cause infinite polling.

#### 1.4 Result Processing

- **Overview:**  
  Processes the completed prediction data to extract relevant output fields such as generated images and metadata.

- **Nodes Involved:**  
  - Process Result

- **Node Details:**  

  - **Process Result**  
    - Type: Code  
    - Role: Extracts and formats final output data including status, output URLs, metrics, timestamps, and model info.  
    - Configuration: Runs JavaScript code once per item, returns a simplified JSON object with fields: status, output, metrics, created_at, completed_at, model, other_url.  
    - Inputs: From "Check If Complete" (success branch)  
    - Outputs: None (end of workflow)  
    - Edge cases: Missing or malformed output data may cause errors in downstream processes.

---

### 3. Summary Table

| Node Name               | Node Type        | Functional Role                       | Input Node(s)           | Output Node(s)          | Sticky Note                                                                                              |
|-------------------------|------------------|------------------------------------|------------------------|-------------------------|---------------------------------------------------------------------------------------------------------|
| On clicking 'execute'    | Manual Trigger   | Manual workflow entry point         | None                   | Set API Key             |                                                                                                         |
| Set API Key             | Set              | Stores Replicate API key             | On clicking 'execute'  | Create Prediction       |                                                                                                         |
| Create Prediction       | HTTP Request     | Sends request to create prediction  | Set API Key            | Extract Prediction ID   |                                                                                                         |
| Extract Prediction ID   | Code             | Extracts prediction ID and URL       | Create Prediction      | Wait                    |                                                                                                         |
| Wait                    | Wait             | Delays before polling prediction     | Extract Prediction ID, Check If Complete (false) | Check Prediction Status |                                                                                                         |
| Check Prediction Status | HTTP Request     | Polls prediction status              | Wait                   | Check If Complete       |                                                                                                         |
| Check If Complete       | If               | Checks if prediction succeeded       | Check Prediction Status| Process Result (true), Wait (false) |                                                                                                         |
| Process Result          | Code             | Processes final prediction output    | Check If Complete (true) | None                   |                                                                                                         |
| Sticky Note             | Sticky Note      | Provides workflow overview and setup | None                   | None                    | ## Digitalhera Heranathalie AI Generator<br><br>This workflow uses the **digitalhera/heranathalie** model from Replicate to generate other content.<br><br>### Setup<br>1. Add your Replicate API key<br>2. Configure the input parameters<br>3. Run the workflow<br><br>### Model Details<br>- **Type**: Other Generation<br>- **Provider**: digitalhera<br>- **Required Fields**: prompt |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: Start the workflow manually.

2. **Add Set Node to Store API Key**  
   - Connect from Manual Trigger node.  
   - Add a string variable named `replicate_api_key`.  
   - Set its value to your actual Replicate API key (`YOUR_REPLICATE_API_KEY` placeholder).  
   - No credentials needed here.

3. **Create HTTP Request Node to Create Prediction**  
   - Connect from Set API Key node.  
   - Method: POST  
   - URL: `https://api.replicate.com/v1/predictions`  
   - Authentication: HTTP Header with Bearer token using `replicate_api_key` variable from Set node.  
   - Headers:  
     - `Authorization`: `Bearer {{$node["Set API Key"].json["replicate_api_key"]}}`  
     - `Content-Type`: `application/json`  
   - Body (raw JSON):  
     ```json
     {
       "version": "c4e122b4dceba454469d84b69b855b38625b89ca6e922e1c1b1a817ea8f7e340",
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
   - Enable sending body and headers.

4. **Add Code Node to Extract Prediction ID**  
   - Connect from Create Prediction node.  
   - Mode: Run Once for Each Item  
   - JavaScript code:  
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

5. **Add Wait Node**  
   - Connect from Extract Prediction ID node initially.  
   - Set delay to 2 seconds.

6. **Add HTTP Request Node to Check Prediction Status**  
   - Connect from Wait node.  
   - Method: GET (default)  
   - URL: Use expression to get URL from previous node: `{{$json["predictionUrl"]}}`  
   - Authentication: Same HTTP Header Bearer token with `replicate_api_key` as before.

7. **Add If Node to Check Completion Status**  
   - Connect from Check Prediction Status node.  
   - Condition: Boolean check if `$json.status == 'succeeded'`.  
   - True output: proceed to Process Result node.  
   - False output: loop back to Wait node for another polling cycle.

8. **Add Code Node to Process Result**  
   - Connect from True output of If node.  
   - Mode: Run Once for Each Item  
   - JavaScript code:  
     ```javascript
     const result = $input.item.json;
     return {
       status: result.status,
       output: result.output,
       metrics: result.metrics,
       created_at: result.created_at,
       completed_at: result.completed_at,
       model: 'digitalhera/heranathalie',
       other_url: result.output
     };
     ```

9. **Optionally Add Sticky Note**  
   - Add a sticky note anywhere in your canvas to describe the workflow purpose and setup steps.

10. **Configure Credentials**  
    - No special credentials node required; API key is handled via Set node and HTTP header authentication.

---

### 5. General Notes & Resources

| Note Content                                                                                  | Context or Link                                  |
|-----------------------------------------------------------------------------------------------|-------------------------------------------------|
| This workflow uses the **digitalhera/heranathalie** model from Replicate for image generation. | Workflow purpose description                      |
| Setup instructions: 1) Add your Replicate API key 2) Configure prompt and input parameters 3) Run workflow | Sticky note content in the workflow               |
| Model Version in API request: `c4e122b4dceba454469d84b69b855b38625b89ca6e922e1c1b1a817ea8f7e340` | Critical for model API call                        |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.