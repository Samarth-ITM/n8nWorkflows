Generate Images with Lemaar Door Blurrred AI via Replicate

https://n8nworkflows.xyz/workflows/generate-images-with-lemaar-door-blurrred-ai-via-replicate-7095


# Generate Images with Lemaar Door Blurrred AI via Replicate

### 1. Workflow Overview

This workflow, titled **Creativeathive Lemaar Door Blurrred AI Generator**, automates the generation of images using the Replicate API with the model **creativeathive/lemaar-door-blurrred**. Its primary use case is to produce AI-generated content based on a text prompt, leveraging Replicate’s prediction endpoints to asynchronously request, monitor, and retrieve generated images.

The workflow is logically divided into the following blocks:

- **1.1 Input Trigger and API Key Setup:** Manual trigger to start the workflow and assignment of the Replicate API key.
- **1.2 Create Prediction Request:** Sends a POST request to Replicate to initiate image generation.
- **1.3 Extract Prediction Details:** Extracts prediction ID and status to prepare for polling.
- **1.4 Polling Loop:** Repeatedly checks the prediction status until completion by waiting and querying the status.
- **1.5 Process Completed Result:** Processes the final prediction output once the generation is successful.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Trigger and API Key Setup

**Overview:**  
This block initiates the workflow manually and sets the API key needed for authentication with the Replicate API.

**Nodes Involved:**  
- On clicking 'execute' (Manual Trigger)  
- Set API Key (Set Node)

**Node Details:**

- **On clicking 'execute'**  
  - Type: Manual Trigger  
  - Role: Entry point for manual execution  
  - Configuration: Default manual trigger with no parameters  
  - Inputs: None  
  - Outputs: Connected to "Set API Key" node  
  - Edge cases: None typical; user must manually trigger

- **Set API Key**  
  - Type: Set  
  - Role: Stores Replicate API key as a variable for authentication  
  - Configuration: Sets `replicate_api_key` as a string variable with placeholder value "YOUR_REPLICATE_API_KEY"  
  - Inputs: From manual trigger  
  - Outputs: To "Create Prediction" node  
  - Edge cases: Missing or invalid API key will cause authentication errors downstream  
  - Note: User must replace the placeholder with a valid Replicate API key

---

#### 1.2 Create Prediction Request

**Overview:**  
Sends a POST request to the Replicate API to create a new image generation prediction using the specified model version and input parameters.

**Nodes Involved:**  
- Create Prediction (HTTP Request)

**Node Details:**

- **Create Prediction**  
  - Type: HTTP Request  
  - Role: Initiates image generation on Replicate by POSTing to `/v1/predictions`  
  - Configuration:  
    - URL: `https://api.replicate.com/v1/predictions`  
    - Method: POST  
    - Headers: Authorization Bearer token (from `replicate_api_key`), Content-Type application/json  
    - Body: JSON specifying model version and input parameters including prompt, seed, width, height, lora_scale  
    - Timeout: 60 seconds  
  - Key Expressions: Uses expression `={{ 'Bearer ' + $('Set API Key').item.json.replicate_api_key }}` for auth header  
  - Inputs: From "Set API Key" node  
  - Outputs: To "Extract Prediction ID" node  
  - Edge cases:  
    - Timeout if Replicate API is slow  
    - HTTP errors if API key invalid or parameters malformed  
    - User must replace `"prompt value"` with actual prompt or dynamic input for real use

---

#### 1.3 Extract Prediction Details

**Overview:**  
Extracts the prediction ID and initial status from the API response to enable status polling.

**Nodes Involved:**  
- Extract Prediction ID (Code)

**Node Details:**

- **Extract Prediction ID**  
  - Type: Code (JavaScript)  
  - Role: Parses API response JSON to extract prediction ID, status, and constructs the status check URL  
  - Configuration:  
    - Runs once per item  
    - Extracts fields: `id`, `status` from response  
    - Constructs `predictionUrl` for subsequent GET requests  
  - Inputs: From "Create Prediction" node  
  - Outputs: To "Wait" node  
  - Edge cases: If response lacks expected fields, code may fail or produce undefined URLs  
  - Version requirements: Compatible with n8n v1+ Code node

---

#### 1.4 Polling Loop

**Overview:**  
Implements a polling mechanism with wait intervals, repeatedly checking the status of the prediction until it is complete (status = 'succeeded').

**Nodes Involved:**  
- Wait  
- Check Prediction Status (HTTP Request)  
- Check If Complete (If)

**Node Details:**

- **Wait**  
  - Type: Wait  
  - Role: Pauses workflow execution for 2 seconds before polling again  
  - Configuration: Fixed 2-second wait  
  - Inputs: From "Extract Prediction ID" (first iteration) or "Check If Complete" (loop)  
  - Outputs: To "Check Prediction Status" node  
  - Edge cases: Excessive polling could hit rate limits; insufficient wait times may cause API throttling

- **Check Prediction Status**  
  - Type: HTTP Request  
  - Role: Sends GET request to Replicate API to retrieve current prediction status  
  - Configuration:  
    - URL dynamic: uses `{{$json.predictionUrl}}`  
    - Authorization header with Bearer token (from Set API Key)  
  - Inputs: From "Wait" node  
  - Outputs: To "Check If Complete" node  
  - Edge cases: Network failure, expired tokens, or invalid prediction ID causing 404 errors

- **Check If Complete**  
  - Type: If  
  - Role: Evaluates if prediction status is "succeeded"  
  - Configuration: Boolean condition comparing `$json.status` equals "succeeded"  
  - Inputs: From "Check Prediction Status" node  
  - Outputs:  
    - True branch: To "Process Result" node  
    - False branch: Loops back to "Wait" node to continue polling  
  - Edge cases: If status changes to fail or error, workflow will loop indefinitely unless further conditions added

---

#### 1.5 Process Completed Result

**Overview:**  
Processes and formats the final prediction result, extracting important metadata and the output URL for the generated image.

**Nodes Involved:**  
- Process Result (Code)

**Node Details:**

- **Process Result**  
  - Type: Code (JavaScript)  
  - Role: Reads completed prediction JSON, extracts output image URL, status, timestamps, and adds model information  
  - Configuration:  
    - Runs once per item  
    - Returns an object with status, output, metrics, creation and completion timestamps, model name, and output URL  
  - Inputs: From "Check If Complete" node (true path)  
  - Outputs: Final output of the workflow (can be used downstream or for display)  
  - Edge cases: If output field is missing or null, the node may return incomplete data

---

### 3. Summary Table

| Node Name              | Node Type         | Functional Role                                    | Input Node(s)           | Output Node(s)          | Sticky Note                                                                                                         |
|------------------------|-------------------|---------------------------------------------------|-------------------------|-------------------------|---------------------------------------------------------------------------------------------------------------------|
| On clicking 'execute'   | Manual Trigger    | Manual start of the workflow                       | None                    | Set API Key             |                                                                                                                     |
| Set API Key            | Set               | Stores Replicate API key for authentication       | On clicking 'execute'   | Create Prediction       |                                                                                                                     |
| Create Prediction       | HTTP Request      | Sends POST request to Replicate to start prediction | Set API Key             | Extract Prediction ID   |                                                                                                                     |
| Extract Prediction ID   | Code              | Extracts prediction ID and status for polling     | Create Prediction       | Wait                    |                                                                                                                     |
| Wait                   | Wait              | Delays workflow to allow time for prediction      | Extract Prediction ID, Check If Complete (false path) | Check Prediction Status |                                                                                                                     |
| Check Prediction Status | HTTP Request      | Retrieves current prediction status from Replicate | Wait                    | Check If Complete       |                                                                                                                     |
| Check If Complete       | If                | Checks if prediction is complete ("succeeded")    | Check Prediction Status | Process Result (true), Wait (false) |                                                                                                                     |
| Process Result          | Code              | Processes final prediction output                  | Check If Complete (true) | None                    |                                                                                                                     |
| Sticky Note            | Sticky Note       | Documentation and instructions                     | None                    | None                    | ## Creativeathive Lemaar Door Blurrred AI Generator<br><br>This workflow uses the **creativeathive/lemaar-door-blurrred** model from Replicate to generate other content.<br><br>### Setup<br>1. Add your Replicate API key<br>2. Configure the input parameters<br>3. Run the workflow<br><br>### Model Details<br>- **Type**: Other Generation<br>- **Provider**: creativeathive<br>- **Required Fields**: prompt |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.

2. **Add Manual Trigger node**  
   - Name: `On clicking 'execute'`  
   - No configuration needed; default manual trigger.

3. **Add Set node**  
   - Name: `Set API Key`  
   - Add a string field named `replicate_api_key` with value `"YOUR_REPLICATE_API_KEY"` (replace with actual key before running).  
   - Connect `On clicking 'execute'` output to this node.

4. **Add HTTP Request node**  
   - Name: `Create Prediction`  
   - Method: POST  
   - URL: `https://api.replicate.com/v1/predictions`  
   - Authentication: Generic Credential → HTTP Header Auth  
   - Headers:  
     - `Authorization`: Expression `={{ 'Bearer ' + $('Set API Key').item.json.replicate_api_key }}`  
     - `Content-Type`: `application/json`  
   - Body Content Type: JSON  
   - Body:  
     ```json
     {
       "version": "0e435a9516abafe4df04d7057bccacbb598d674db68316ea2397dc6363f62963",
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
   - Connect `Set API Key` output to this node.

5. **Add Code node**  
   - Name: `Extract Prediction ID`  
   - Mode: Run Once For Each Item  
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
   - Connect `Create Prediction` output to this node.

6. **Add Wait node**  
   - Name: `Wait`  
   - Wait Time: 2 seconds  
   - Connect `Extract Prediction ID` output to this node.

7. **Add HTTP Request node**  
   - Name: `Check Prediction Status`  
   - Method: GET  
   - URL: Expression `={{ $json.predictionUrl }}`  
   - Authentication: Generic Credential → HTTP Header Auth  
   - Headers:  
     - `Authorization`: Expression `={{ 'Bearer ' + $('Set API Key').item.json.replicate_api_key }}`  
   - Connect `Wait` output to this node.

8. **Add If node**  
   - Name: `Check If Complete`  
   - Condition: Boolean  
     - `{{$json.status}}` equals `"succeeded"`  
   - Connect `Check Prediction Status` output to this node.

9. **Add Code node**  
   - Name: `Process Result`  
   - Mode: Run Once For Each Item  
   - Code:
     ```javascript
     const result = $input.item.json;
     return {
       status: result.status,
       output: result.output,
       metrics: result.metrics,
       created_at: result.created_at,
       completed_at: result.completed_at,
       model: 'creativeathive/lemaar-door-blurrred',
       other_url: result.output
     };
     ```
   - Connect the **true** output of `Check If Complete` to this node.

10. **Loop the polling**  
    - Connect the **false** output of `Check If Complete` back to the `Wait` node to continue polling.

11. **Save and test**  
    - Replace `"prompt value"` in the `Create Prediction` node with actual prompt text or dynamic input.  
    - Replace `"YOUR_REPLICATE_API_KEY"` in the `Set API Key` node with your actual Replicate API key.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                            | Context or Link                                      |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| This workflow uses the **creativeathive/lemaar-door-blurrred** model from Replicate to generate creative AI images based on prompts.                                  | Sticky Note content in workflow                      |
| Model Version used: `0e435a9516abafe4df04d7057bccacbb598d674db68316ea2397dc6363f62963`                                                                                   | Create Prediction node parameter                     |
| API Authentication requires a valid Replicate API key with access to the prediction API.                                                                               | Set API Key node parameter                           |
| Polling interval is set to 2 seconds; adjust based on API rate limits or expected generation times.                                                                   | Wait node configuration                              |
| If the prediction status never changes to "succeeded," the workflow will continuously poll; consider adding error or timeout handling for robustness.                 | Suggested improvement                                |
| For more details on Replicate API: https://replicate.com/docs/reference/http                                                                                          | External documentation link                          |

---

**Disclaimer:** The provided text is exclusively derived from an automated n8n workflow and complies with all relevant content policies. No illegal or protected content is included. All data is legal and public.