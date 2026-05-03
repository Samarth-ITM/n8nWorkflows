Generate AI Images with Digitalhera Herathaisbragatto Model via Replicate

https://n8nworkflows.xyz/workflows/generate-ai-images-with-digitalhera-herathaisbragatto-model-via-replicate-7094


# Generate AI Images with Digitalhera Herathaisbragatto Model via Replicate

### 1. Workflow Overview

This workflow automates generating AI images using the **Digitalhera Herathaisbragatto** model hosted on Replicate. It is designed to:

- Receive a manual trigger to start the process
- Configure the API key for Replicate access
- Send a generation request with user-defined parameters (prompt, seed, dimensions, scale)
- Poll Replicate’s API until the generation is complete
- Extract and process the generated image result

The workflow is logically organized into four functional blocks:

- **1.1 Input and Initialization**: Trigger and API key setup  
- **1.2 Prediction Request Creation**: Sending the generation request to Replicate  
- **1.3 Polling for Completion**: Waiting and checking prediction status until done  
- **1.4 Result Processing**: Handling the output from the completed prediction  

---

### 2. Block-by-Block Analysis

#### 1.1 Input and Initialization

**Overview:**  
This block initiates the workflow with a manual trigger and sets the Replicate API key needed for authentication in subsequent requests.

**Nodes Involved:**  
- On clicking 'execute' (Manual Trigger)  
- Set API Key (Set node)  

**Node Details:**

- **On clicking 'execute'**  
  - Type: Manual Trigger  
  - Role: Entry point to start the workflow manually  
  - Configuration: Default manual trigger, no parameters required  
  - Inputs: None  
  - Outputs: To "Set API Key" node  
  - Edge cases: None inherent; user must manually trigger  

- **Set API Key**  
  - Type: Set node  
  - Role: Stores the Replicate API key as a workflow variable  
  - Configuration: Assigns `"replicate_api_key"` with the placeholder value `"YOUR_REPLICATE_API_KEY"` (must be replaced by the user)  
  - Key expression: Static string assignment  
  - Inputs: From manual trigger  
  - Outputs: To "Create Prediction" node  
  - Edge cases: Missing or invalid API key will cause authentication failures downstream  

---

#### 1.2 Prediction Request Creation

**Overview:**  
This block creates an HTTP POST request to Replicate's API to start generating the AI image based on user parameters.

**Nodes Involved:**  
- Create Prediction (HTTP Request)  
- Extract Prediction ID (Code node)  

**Node Details:**

- **Create Prediction**  
  - Type: HTTP Request  
  - Role: Sends a POST request to Replicate API to create an image generation prediction  
  - Configuration:  
    - URL: `https://api.replicate.com/v1/predictions`  
    - Method: POST  
    - JSON Body includes:  
      - version: fixed model version ID `68f10949d36128d85f64e189dc0d9efd112bb2593245daadde6616023cd134ad`  
      - input: object with parameters such as `prompt`, `seed`, `width`, `height`, `lora_scale`  
    - Headers: Authorization Bearer token from "Set API Key", Content-Type application/json  
    - Timeout: 60 seconds  
  - Key expressions:  
    - Authorization header dynamically constructed as `Bearer <replicate_api_key>`  
    - The `"prompt"` field is currently a placeholder `"prompt value"` and should be replaced with actual input  
  - Inputs: From "Set API Key"  
  - Outputs: To "Extract Prediction ID"  
  - Edge cases:  
    - Timeout if API is slow  
    - Authentication errors if API key is invalid  
    - Malformed input parameters cause API rejection  

- **Extract Prediction ID**  
  - Type: Code (JavaScript)  
  - Role: Extracts the generated prediction ID and initial status from the response  
  - Configuration:  
    - Reads JSON response from previous node  
    - Extracts `id` and `status` fields  
    - Constructs a `predictionUrl` string for later polling  
  - Inputs: From "Create Prediction"  
  - Outputs: To "Wait" node  
  - Edge cases:  
    - Missing `id` in response results in failure  
    - Unexpected response format causes code error  

---

#### 1.3 Polling for Completion

**Overview:**  
This block implements a polling mechanism to periodically check if the image generation is complete.

**Nodes Involved:**  
- Wait (Wait node)  
- Check Prediction Status (HTTP Request)  
- Check If Complete (If node)  

**Node Details:**

- **Wait**  
  - Type: Wait  
  - Role: Pauses workflow for 2 seconds before polling again  
  - Configuration: Wait for 2 seconds  
  - Inputs: From "Extract Prediction ID" and from "Check If Complete" (false branch)  
  - Outputs: To "Check Prediction Status"  
  - Edge cases: None significant, but too short wait may cause rate limiting  

- **Check Prediction Status**  
  - Type: HTTP Request  
  - Role: GET request to Replicate API to retrieve current status of the prediction  
  - Configuration:  
    - URL: Dynamic, from `predictionUrl` in previous code node  
    - Method: GET (default)  
    - Authorization header with Bearer token from "Set API Key"  
  - Inputs: From "Wait"  
  - Outputs: To "Check If Complete"  
  - Edge cases:  
    - API errors or network errors cause failure  
    - Rate limiting if polled too frequently  

- **Check If Complete**  
  - Type: If  
  - Role: Branches workflow depending on whether prediction status is `"succeeded"`  
  - Configuration: Checks if `$json.status == 'succeeded'`  
  - Inputs: From "Check Prediction Status"  
  - Outputs:  
    - True branch: to "Process Result"  
    - False branch: loops back to "Wait" to poll again  
  - Edge cases:  
    - Other statuses (e.g., `failed`) are not explicitly handled and cause infinite loop  
    - Could be improved to handle error statuses  

---

#### 1.4 Result Processing

**Overview:**  
After successful generation, this block processes the output data for further use or display.

**Nodes Involved:**  
- Process Result (Code node)  

**Node Details:**

- **Process Result**  
  - Type: Code (JavaScript)  
  - Role: Extracts and formats relevant output fields from the completed prediction  
  - Configuration:  
    - Extracts fields: `status`, `output`, `metrics`, `created_at`, `completed_at`  
    - Adds model name as a static field `"digitalhera/herathaisbragatto"`  
    - Also provides `other_url` pointing to output location (usually the image URL)  
  - Inputs: From "Check If Complete" (true branch)  
  - Outputs: None connected further (end of workflow)  
  - Edge cases:  
    - Missing or malformed output field causes code failure  
    - No error handling for failed predictions  

---

### 3. Summary Table

| Node Name               | Node Type          | Functional Role                  | Input Node(s)              | Output Node(s)            | Sticky Note                                                                                   |
|-------------------------|--------------------|--------------------------------|----------------------------|---------------------------|----------------------------------------------------------------------------------------------|
| On clicking 'execute'    | Manual Trigger     | Workflow entry trigger          | -                          | Set API Key               |                                                                                              |
| Set API Key             | Set                 | Stores Replicate API key        | On clicking 'execute'       | Create Prediction          |                                                                                              |
| Create Prediction       | HTTP Request        | Sends generation request to Replicate | Set API Key                | Extract Prediction ID      |                                                                                              |
| Extract Prediction ID   | Code                | Extracts prediction ID & status | Create Prediction           | Wait                      |                                                                                              |
| Wait                    | Wait                | Waits before polling again      | Extract Prediction ID, Check If Complete (false) | Check Prediction Status |                                                                                              |
| Check Prediction Status | HTTP Request        | Polls Replicate for prediction status | Wait                       | Check If Complete          |                                                                                              |
| Check If Complete       | If                  | Checks if generation succeeded  | Check Prediction Status     | Process Result (true), Wait (false) |                                                                                              |
| Process Result          | Code                | Processes final prediction output | Check If Complete (true)    | -                         |                                                                                              |
| Sticky Note             | Sticky Note         | Documentation and setup info    | -                          | -                         | ## Digitalhera Herathaisbragatto AI Generator<br>This workflow uses the **digitalhera/herathaisbragatto** model from Replicate to generate other content.<br>### Setup<br>1. Add your Replicate API key<br>2. Configure the input parameters<br>3. Run the workflow<br>### Model Details<br>- **Type**: Other Generation<br>- **Provider**: digitalhera<br>- **Required Fields**: prompt |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add **Manual Trigger** node named "On clicking 'execute'"  
   - No special configuration needed  

2. **Add Set Node for API Key**  
   - Create a **Set** node named "Set API Key"  
   - Add a string field: `replicate_api_key` with value `"YOUR_REPLICATE_API_KEY"` (replace with actual key)  
   - Connect "On clicking 'execute'" → "Set API Key"  

3. **Create HTTP Request Node for Prediction Creation**  
   - Add **HTTP Request** node named "Create Prediction"  
   - Method: POST  
   - URL: `https://api.replicate.com/v1/predictions`  
   - Authentication: Generic Credential → HTTP Header Auth  
   - Headers:  
     - `Authorization`: Expression `Bearer {{$json["replicate_api_key"]}}` referencing "Set API Key" node  
     - `Content-Type`: `application/json`  
   - Body Content-Type: JSON  
   - JSON Body:  
     ```json
     {
       "version": "68f10949d36128d85f64e189dc0d9efd112bb2593245daadde6616023cd134ad",
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
   - Connect "Set API Key" → "Create Prediction"  

4. **Add Code Node to Extract Prediction ID**  
   - Add a **Code** node named "Extract Prediction ID"  
   - Language: JavaScript  
   - Code:  
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
   - Connect "Create Prediction" → "Extract Prediction ID"  

5. **Add Wait Node**  
   - Add **Wait** node named "Wait"  
   - Wait time: 2 seconds  
   - Connect "Extract Prediction ID" → "Wait"  

6. **Add HTTP Request Node to Check Prediction Status**  
   - Add **HTTP Request** node named "Check Prediction Status"  
   - Method: GET (default)  
   - URL: Expression `{{$json["predictionUrl"]}}` from previous node  
   - Authentication: Generic Credential → HTTP Header Auth  
   - Headers:  
     - `Authorization`: Expression `Bearer {{$json["replicate_api_key"]}}` referencing "Set API Key" node  
   - Connect "Wait" → "Check Prediction Status"  

7. **Add If Node to Check Completion**  
   - Add **If** node named "Check If Complete"  
   - Condition: Boolean  
     - Value 1: Expression `{{$json.status}}`  
     - Operation: Equals  
     - Value 2: `succeeded`  
   - Connect "Check Prediction Status" → "Check If Complete"  

8. **Add Code Node to Process Result**  
   - Add **Code** node named "Process Result"  
   - Language: JavaScript  
   - Code:  
     ```javascript
     const result = $input.item.json;
     return {
       status: result.status,
       output: result.output,
       metrics: result.metrics,
       created_at: result.created_at,
       completed_at: result.completed_at,
       model: 'digitalhera/herathaisbragatto',
       other_url: result.output
     };
     ```  
   - Connect "Check If Complete" (true) → "Process Result"  

9. **Loop Back If Not Complete**  
   - Connect "Check If Complete" (false) → "Wait"  

10. **Create Sticky Note** (optional but recommended)  
    - Add a **Sticky Note** with the following content:  
      ```
      ## Digitalhera Herathaisbragatto AI Generator

      This workflow uses the **digitalhera/herathaisbragatto** model from Replicate to generate other content.

      ### Setup
      1. Add your Replicate API key
      2. Configure the input parameters
      3. Run the workflow

      ### Model Details
      - **Type**: Other Generation
      - **Provider**: digitalhera
      - **Required Fields**: prompt
      ```  
    - Place it visibly near the start nodes for user reference  

---

### 5. General Notes & Resources

| Note Content                                                                                                                | Context or Link                                                                                         |
|-----------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| This workflow requires a valid Replicate API key with permission to use the digitalhera/herathaisbragatto model.             | https://replicate.com/digitalhera/herathaisbragatto                                                   |
| Input parameter `prompt` must be replaced with actual desired prompt text before execution.                                 | Input field in "Create Prediction" HTTP Request node                                                  |
| Polling interval is set to 2 seconds; adjust in "Wait" node if Replicate rate limits or for faster response needs.           | Wait node configuration                                                                               |
| No explicit error handling implemented for failed or canceled prediction states; consider adding for robustness.            | Workflow enhancement suggestion                                                                        |
| For more information on Replicate API and prediction workflow, see official docs: https://replicate.com/docs/api-reference |                                                                                                       |

---

**Disclaimer:** The provided text is exclusively derived from an automated n8n workflow. It complies strictly with content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.