Generate AI Images with Replicate's Lemaar Door Mockedup Model

https://n8nworkflows.xyz/workflows/generate-ai-images-with-replicate-s-lemaar-door-mockedup-model-6889


# Generate AI Images with Replicate's Lemaar Door Mockedup Model

### 1. Workflow Overview

This workflow, titled **Creativeathive Lemaar Door Mockedup AI Generator**, is designed to generate images using the Replicate API with the **creativeathive/lemaar-door-mockedup** model. It is intended for users who want to create AI-generated door mockup images based on textual prompts.

The workflow is logically divided into the following blocks:

- **1.1 Input Trigger and API Key Setup**: Manual trigger to start the workflow and set the Replicate API key.
- **1.2 Prediction Creation**: Sending a request to Replicate to create an AI prediction with the user-defined parameters.
- **1.3 Polling for Completion**: Periodically checking the status of the prediction until it is complete.
- **1.4 Result Processing**: Extracting and formatting the final output once the prediction succeeds.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Trigger and API Key Setup

**Overview:**  
This block starts the workflow manually and sets the necessary API key for authentication with the Replicate API.

**Nodes Involved:**  
- On clicking 'execute'  
- Set API Key

**Node Details:**

- **On clicking 'execute'**  
  - Type: Manual Trigger  
  - Role: Workflow entry point, initiates execution on user command.  
  - Configuration: No parameters; simply triggers downstream nodes.  
  - Inputs: None  
  - Outputs: Connected to "Set API Key" node  
  - Potential Failures: None expected; user must manually trigger.

- **Set API Key**  
  - Type: Set  
  - Role: Assigns the Replicate API key to a workflow variable for reuse.  
  - Configuration: Hardcoded string placeholder `YOUR_REPLICATE_API_KEY` assigned to variable `replicate_api_key`.  
  - Inputs: From manual trigger  
  - Outputs: To "Create Prediction" node  
  - Edge Cases: If the API key is missing or invalid, downstream API calls will fail with authentication errors.  
  - Note: Users must replace the placeholder with a valid Replicate API key.

---

#### 1.2 Prediction Creation

**Overview:**  
This block submits a prediction request to the Replicate API to start image generation using the specified model and input parameters.

**Nodes Involved:**  
- Create Prediction  
- Extract Prediction ID

**Node Details:**

- **Create Prediction**  
  - Type: HTTP Request  
  - Role: Sends a POST request to Replicate’s prediction endpoint to create a new prediction job.  
  - Configuration:  
    - URL: `https://api.replicate.com/v1/predictions`  
    - Method: POST  
    - Headers: Authorization Bearer token from `replicate_api_key`; Content-Type application/json  
    - Body (JSON): Contains version ID of the model and input parameters (prompt, seed, width, height, lora_scale). These inputs are currently static placeholders (e.g., `"prompt": "prompt value"`, `"seed": 1`).  
    - Timeout: 60 seconds  
    - Authentication: HTTP Header with Bearer token  
  - Inputs: From "Set API Key"  
  - Outputs: JSON response with prediction metadata  
  - Edge Cases:  
    - API key invalid or expired: 401 Unauthorized  
    - API rate limits: 429 Too Many Requests  
    - Network timeout or connection errors  
    - Invalid input parameters causing 400 Bad Request  
  - Notes: Users must customize input parameters for meaningful generation.

- **Extract Prediction ID**  
  - Type: Code  
  - Role: Parses the response from the prediction creation to extract the prediction ID and initial status, formats the prediction status URL for polling.  
  - Configuration: Custom JavaScript code that reads `prediction.id` and `prediction.status` from input JSON, returning an object with `predictionId`, `status`, and `predictionUrl`.  
  - Inputs: From "Create Prediction"  
  - Outputs: To "Wait" node  
  - Edge Cases: If the response is malformed or lacks expected properties, code errors may occur.

---

#### 1.3 Polling for Completion

**Overview:**  
This block repeatedly polls the Replicate API to check if the prediction is complete, introducing delays between checks to avoid excessive API calls.

**Nodes Involved:**  
- Wait  
- Check Prediction Status  
- Check If Complete

**Node Details:**

- **Wait**  
  - Type: Wait  
  - Role: Pauses workflow execution for a fixed time before the next status check.  
  - Configuration: Wait for 2 seconds  
  - Inputs: From "Extract Prediction ID" and from "Check If Complete" (loop path)  
  - Outputs: To "Check Prediction Status"  
  - Edge Cases: None significant; fixed delay.

- **Check Prediction Status**  
  - Type: HTTP Request  
  - Role: GET request to the prediction status endpoint to check current prediction state.  
  - Configuration:  
    - URL: dynamic, from `predictionUrl` in JSON input  
    - Method: GET (default)  
    - Headers: Authorization Bearer token from `replicate_api_key`  
    - Authentication: HTTP Header with Bearer token  
  - Inputs: From "Wait"  
  - Outputs: To "Check If Complete"  
  - Edge Cases:  
    - API errors or network failures  
    - Rate limiting  
    - Invalid or expired prediction ID

- **Check If Complete**  
  - Type: If  
  - Role: Conditionally directs the flow depending on the prediction status.  
  - Configuration: Checks if `status` equals `"succeeded"`  
  - Inputs: From "Check Prediction Status"  
  - Outputs:  
    - True: To "Process Result"  
    - False: To "Wait" (loop for polling)  
  - Edge Cases:  
    - Prediction could be in error or failed states, but these are not explicitly handled here.  
    - Potential infinite loop if status never becomes "succeeded".

---

#### 1.4 Result Processing

**Overview:**  
Once the prediction is completed successfully, this block processes the result to extract useful data for downstream use or display.

**Nodes Involved:**  
- Process Result

**Node Details:**

- **Process Result**  
  - Type: Code  
  - Role: Extracts and formats the final prediction output, including metadata like creation and completion time, output URLs, and model reference.  
  - Configuration: Custom JavaScript that reads the prediction JSON input and returns an object with keys: status, output, metrics, created_at, completed_at, model (fixed string), other_url (same as output).  
  - Inputs: From "Check If Complete" (True branch)  
  - Outputs: Final processed data for use or export  
  - Edge Cases: If the prediction output is missing or malformed, output processing may fail.

---

#### Additional Node: Sticky Note

- **Sticky Note**  
  - Role: Documentation within the workflow canvas describing the workflow purpose, setup instructions, and model details.  
  - Position: Top-left corner for visibility.  
  - Content highlights:  
    - Workflow uses the creativeathive/lemaar-door-mockedup model  
    - Setup steps: add API key, configure inputs, run workflow  
    - Model type and required fields

---

### 3. Summary Table

| Node Name               | Node Type          | Functional Role                      | Input Node(s)               | Output Node(s)             | Sticky Note                                                                                              |
|-------------------------|--------------------|------------------------------------|----------------------------|----------------------------|--------------------------------------------------------------------------------------------------------|
| On clicking 'execute'    | Manual Trigger     | Workflow entry point                | —                          | Set API Key                |                                                                                                |
| Set API Key             | Set                | Assign Replicate API key            | On clicking 'execute'       | Create Prediction          |                                                                                                |
| Create Prediction        | HTTP Request       | Send prediction creation request   | Set API Key                | Extract Prediction ID      |                                                                                                |
| Extract Prediction ID    | Code               | Extract prediction ID and URL      | Create Prediction          | Wait                      |                                                                                                |
| Wait                    | Wait               | Pause between polls                 | Extract Prediction ID, Check If Complete (False branch) | Check Prediction Status    |                                                                                                |
| Check Prediction Status  | HTTP Request       | Poll prediction status              | Wait                       | Check If Complete          |                                                                                                |
| Check If Complete        | If                 | Check if prediction succeeded      | Check Prediction Status    | Process Result (True), Wait (False) |                                                                                                |
| Process Result           | Code               | Process and format final output    | Check If Complete (True)   | —                         |                                                                                                |
| Sticky Note              | Sticky Note        | Documentation                      | —                          | —                         | ## Creativeathive Lemaar Door Mockedup AI Generator<br>This workflow uses the **creativeathive/lemaar-door-mockedup** model from Replicate to generate other content.<br>### Setup<br>1. Add your Replicate API key<br>2. Configure the input parameters<br>3. Run the workflow<br>### Model Details<br>- **Type**: Other Generation<br>- **Provider**: creativeathive<br>- **Required Fields**: prompt |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node**  
   - Name: `On clicking 'execute'`  
   - Purpose: Entry point for manual execution.

2. **Add a Set node to assign the Replicate API key**  
   - Name: `Set API Key`  
   - Add a string field named `replicate_api_key`.  
   - Set its value to your actual Replicate API key (replace `YOUR_REPLICATE_API_KEY` placeholder).

3. **Add an HTTP Request node to create the prediction**  
   - Name: `Create Prediction`  
   - Method: POST  
   - URL: `https://api.replicate.com/v1/predictions`  
   - Authentication: HTTP Header with Bearer token  
     - Header `Authorization`: `Bearer {{$node["Set API Key"].json["replicate_api_key"]}}`  
     - Header `Content-Type`: `application/json`  
   - Body Content (JSON):  
     ```json
     {
       "version": "4980230b6447af824eef74df925bfbedcaab3ccddddfff4d4e83774683ce508c",
       "input": {
         "prompt": "prompt value",
         "seed": 1,
         "width": 1,
         "height": 1,
         "lora_scale": 1
       }
     }
     ```  
   - Timeout: 60000 ms  
   - Connect input from `Set API Key`.

4. **Add a Code node to extract prediction ID and status**  
   - Name: `Extract Prediction ID`  
   - Code (JavaScript):  
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
   - Connect input from `Create Prediction`.

5. **Add a Wait node to pause before polling**  
   - Name: `Wait`  
   - Wait Time: 2 seconds  
   - Connect input from `Extract Prediction ID` and later loop back from the conditional check.

6. **Add an HTTP Request node to check prediction status**  
   - Name: `Check Prediction Status`  
   - Method: GET (default)  
   - URL: `{{$json["predictionUrl"]}}` (dynamic from previous node output)  
   - Authentication: HTTP Header Bearer token same as in `Create Prediction`.  
   - Connect input from `Wait`.

7. **Add an If node to check if prediction is complete**  
   - Name: `Check If Complete`  
   - Condition: Boolean check if `{{$json["status"]}}` equals `"succeeded"`  
   - True output: Connect to `Process Result`  
   - False output: Connect back to `Wait` (loop for polling).

8. **Add a Code node to process the final result**  
   - Name: `Process Result`  
   - Code (JavaScript):  
     ```javascript
     const result = $input.item.json;
     return {
       status: result.status,
       output: result.output,
       metrics: result.metrics,
       created_at: result.created_at,
       completed_at: result.completed_at,
       model: 'creativeathive/lemaar-door-mockedup',
       other_url: result.output
     };
     ```  
   - Connect input from `Check If Complete` (true branch).

9. **Add a Sticky Note for documentation** (optional)  
   - Include the workflow description, setup instructions, and model details for clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                             | Context or Link                                              |
|--------------------------------------------------------------------------------------------------------|--------------------------------------------------------------|
| This workflow uses the **creativeathive/lemaar-door-mockedup** model from Replicate for AI image generation. | Model provider and usage context                             |
| Setup requires a valid Replicate API key; replace the placeholder in the Set node before running.       | API key management                                           |
| Polling interval is fixed at 2 seconds; adjust the Wait node if API rate limits or response times change.| Polling and API considerations                               |
| The workflow currently uses static placeholder input values (e.g., prompt). Customize these to specific use cases. | Input parameter customization                                |
| Possible failure points include API authentication errors, rate limiting, network timeouts, and invalid inputs. | Error and edge case awareness                                |
| For more information on Replicate API usage, visit: https://replicate.com/docs/api-reference          | External API documentation                                   |

---

**Disclaimer:** The provided text exclusively originates from an automated workflow created with n8n, a tool for integration and automation. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.