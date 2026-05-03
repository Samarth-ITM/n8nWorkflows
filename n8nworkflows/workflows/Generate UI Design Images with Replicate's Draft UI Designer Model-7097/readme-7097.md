Generate UI Design Images with Replicate's Draft UI Designer Model

https://n8nworkflows.xyz/workflows/generate-ui-design-images-with-replicate-s-draft-ui-designer-model-7097


# Generate UI Design Images with Replicate's Draft UI Designer Model

### 1. Workflow Overview

This workflow, titled **"Justingirard Draft Ui Designer Image Generator"**, leverages the Replicate API and the **justingirard/draft-ui-designer** model to generate UI design images from textual prompts. It is designed for users needing to create UI mockups or design drafts automatically based on descriptive input.

The workflow is structured into the following logical blocks:

- **1.1 Input Trigger & API Key Setup:** Manual execution trigger and secure assignment of the Replicate API key.
- **1.2 Prediction Creation:** Sending a request to Replicate’s API to start the image generation prediction.
- **1.3 Prediction Monitoring:** Polling the prediction status until completion.
- **1.4 Result Processing:** Extracting and formatting the output once the prediction succeeds.
- **1.5 Documentation:** A sticky note node provides user guidance and model details.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Trigger & API Key Setup

- **Overview:** Initiates the workflow manually and sets the Replicate API key for authentication.
- **Nodes Involved:** 
  - On clicking 'execute'
  - Set API Key

##### Node Details:

- **On clicking 'execute'**
  - Type: Manual Trigger
  - Role: Starts the workflow execution manually on user command.
  - Configuration: No parameters; standard manual trigger.
  - Inputs: None
  - Outputs: Connects to "Set API Key"
  - Edge Cases: None, but requires manual user execution.
  
- **Set API Key**
  - Type: Set
  - Role: Stores the Replicate API key in the workflow context.
  - Configuration: Assigns a string variable `replicate_api_key` with the placeholder "YOUR_REPLICATE_API_KEY" to be replaced by the user.
  - Inputs: From "On clicking 'execute'"
  - Outputs: Connects to "Create Prediction"
  - Edge Cases: Missing or invalid API key will cause authentication failures downstream.

#### 1.2 Prediction Creation

- **Overview:** Sends a POST request to Replicate API to create a new prediction job for UI design generation.
- **Nodes Involved:** 
  - Create Prediction
  - Extract Prediction ID

##### Node Details:

- **Create Prediction**
  - Type: HTTP Request
  - Role: Creates a prediction request at Replicate with specified model version and input parameters.
  - Configuration:
    - URL: `https://api.replicate.com/v1/predictions`
    - Method: POST
    - Headers: Authorization with Bearer token from `replicate_api_key`, Content-Type application/json.
    - Body: JSON including:
      - `version`: fixed model version hash for justingirard/draft-ui-designer
      - `input` object with keys: `prompt` (static placeholder `"prompt value"`), `seed`, `width`, `height`, `lora_scale` all set to 1.
    - Timeout: 60 seconds
  - Inputs: From "Set API Key"
  - Outputs: Connects to "Extract Prediction ID"
  - Edge Cases:
    - API key authentication error
    - Network timeout
    - Invalid or missing prompt input (currently hardcoded placeholder)
  - Version: n8n HTTP Request v4.2

- **Extract Prediction ID**
  - Type: Code
  - Role: Parses the prediction ID and initial status from the API response for polling.
  - Configuration:
    - JavaScript extracts `id` and `status` from response JSON, constructs `predictionUrl` for status checking.
  - Inputs: From "Create Prediction"
  - Outputs: Connects to "Wait"
  - Edge Cases:
    - Unexpected response shape causing extraction failure

#### 1.3 Prediction Monitoring

- **Overview:** Implements a polling mechanism to repeatedly check the prediction status until it completes successfully.
- **Nodes Involved:** 
  - Wait
  - Check Prediction Status
  - Check If Complete

##### Node Details:

- **Wait**
  - Type: Wait
  - Role: Delays execution for 2 seconds before rechecking prediction status.
  - Configuration: 2 seconds wait
  - Inputs: From "Extract Prediction ID" (first iteration) or "Check If Complete" (if not finished)
  - Outputs: Connects to "Check Prediction Status"
  - Edge Cases: Excessive polling may trigger rate limits on API.

- **Check Prediction Status**
  - Type: HTTP Request
  - Role: Fetches the current status of the prediction from Replicate API.
  - Configuration:
    - URL: uses expression `{{$json.predictionUrl}}` from previous node.
    - Method: GET (default)
    - Headers: Authorization Bearer token from the stored API key.
  - Inputs: From "Wait"
  - Outputs: Connects to "Check If Complete"
  - Edge Cases:
    - API rate limits
    - Network errors
    - Invalid prediction URL if extraction failed

- **Check If Complete**
  - Type: If
  - Role: Checks if prediction status equals "succeeded".
  - Configuration:
    - Condition: Boolean equals `$json.status == "succeeded"`
  - Inputs: From "Check Prediction Status"
  - Outputs:
    - True branch: connects to "Process Result"
    - False branch: loops back to "Wait" to continue polling
  - Edge Cases:
    - Prediction failure or cancellation state is not explicitly handled here, may cause infinite polling.

#### 1.4 Result Processing

- **Overview:** Processes the final prediction result, extracting relevant metadata and image output URL.
- **Nodes Involved:** 
  - Process Result

##### Node Details:

- **Process Result**
  - Type: Code
  - Role: Formats and outputs the prediction result including status, output images, metrics, timestamps, model name.
  - Configuration:
    - JavaScript extracts keys: `status`, `output`, `metrics`, `created_at`, `completed_at`
    - Adds static field `"model": 'justingirard/draft-ui-designer'`
    - Extracts image URL from `output`
  - Inputs: From "Check If Complete" (True branch)
  - Outputs: Final output of the workflow (not connected further)
  - Edge Cases:
    - Missing or malformed output in prediction result
    - Multiple image outputs not explicitly handled (assumes `output` is a single URL or an array)

#### 1.5 Documentation

- **Overview:** Provides inline documentation explaining the workflow purpose and setup instructions.
- **Nodes Involved:** 
  - Sticky Note

##### Node Details:

- **Sticky Note**
  - Type: Sticky Note
  - Role: User guidance and model description
  - Content:
    ```
    ## Justingirard Draft Ui Designer Image Generator

    This workflow uses the **justingirard/draft-ui-designer** model from Replicate to generate image content.

    ### Setup
    1. Add your Replicate API key
    2. Configure the input parameters
    3. Run the workflow

    ### Model Details
    - **Type**: Image Generation
    - **Provider**: justingirard
    - **Required Fields**: prompt
    ```
  - Position: Top-left area, visually separated from main flow
  - Inputs/Outputs: None
  - Edge Cases: None

---

### 3. Summary Table

| Node Name               | Node Type         | Functional Role                      | Input Node(s)            | Output Node(s)           | Sticky Note                                                                                      |
|-------------------------|-------------------|------------------------------------|--------------------------|--------------------------|------------------------------------------------------------------------------------------------|
| On clicking 'execute'    | Manual Trigger    | Start workflow manually             | None                     | Set API Key              |                                                                                                |
| Set API Key             | Set               | Store Replicate API key             | On clicking 'execute'    | Create Prediction        |                                                                                                |
| Create Prediction       | HTTP Request      | Create prediction request to API   | Set API Key              | Extract Prediction ID    |                                                                                                |
| Extract Prediction ID   | Code              | Extract prediction ID and status   | Create Prediction        | Wait                     |                                                                                                |
| Wait                    | Wait              | Delay for polling interval          | Extract Prediction ID / Check If Complete (False) | Check Prediction Status  |                                                                                                |
| Check Prediction Status | HTTP Request      | Poll prediction status              | Wait                     | Check If Complete        |                                                                                                |
| Check If Complete       | If                | Check if prediction succeeded      | Check Prediction Status  | Process Result / Wait    |                                                                                                |
| Process Result          | Code              | Format and output final result     | Check If Complete (True) | None                     |                                                                                                |
| Sticky Note             | Sticky Note       | Documentation and instructions     | None                     | None                     | ## Justingirard Draft Ui Designer Image Generator<br>This workflow uses the **justingirard/draft-ui-designer** model from Replicate to generate image content.<br>Setup steps and model details included. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node**
   - Name: "On clicking 'execute'"
   - Purpose: Start the workflow manually.
   - No parameters needed.

2. **Add a Set Node**
   - Name: "Set API Key"
   - Purpose: Store your Replicate API key.
   - Under "Values to Set", add a string variable:
     - Name: `replicate_api_key`
     - Value: `"YOUR_REPLICATE_API_KEY"` (replace with your actual key)
   - Connect output from "On clicking 'execute'" to this node.

3. **Add HTTP Request Node for Prediction Creation**
   - Name: "Create Prediction"
   - Method: POST
   - URL: `https://api.replicate.com/v1/predictions`
   - Headers:
     - Authorization: `Bearer {{$json.replicate_api_key}}` (use expression referencing "Set API Key")
     - Content-Type: `application/json`
   - Body (JSON):
     ```json
     {
       "version": "f1114b15c528a5d9e40b905a5138f6abdeecb91b57e33ddbb3a71ca10ac92b41",
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
   - Connect output from "Set API Key" to this node.

4. **Add a Code Node to Extract Prediction ID**
   - Name: "Extract Prediction ID"
   - Language: JavaScript
   - Code:
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
   - Run once per item
   - Connect output from "Create Prediction" to this node.

5. **Add a Wait Node**
   - Name: "Wait"
   - Parameters: Wait 2 seconds
   - Connect output from "Extract Prediction ID" to this node.

6. **Add HTTP Request Node to Check Prediction Status**
   - Name: "Check Prediction Status"
   - Method: GET (default)
   - URL: Use expression `{{$json.predictionUrl}}` from previous node's output
   - Headers:
     - Authorization: `Bearer {{$json.replicate_api_key}}` (reference "Set API Key" node's variable)
   - Connect output from "Wait" to this node.

7. **Add an If Node to Check Completion**
   - Name: "Check If Complete"
   - Condition: Boolean equals `$json.status == "succeeded"`
   - Connect output from "Check Prediction Status" to this node.
   - For the True branch, connect to next step.
   - For the False branch, connect back to the "Wait" node to continue polling.

8. **Add Code Node to Process Result**
   - Name: "Process Result"
   - Language: JavaScript
   - Code:
     ```js
     const result = $input.item.json;

     return {
       status: result.status,
       output: result.output,
       metrics: result.metrics,
       created_at: result.created_at,
       completed_at: result.completed_at,
       model: 'justingirard/draft-ui-designer',
       image_url: result.output
     };
     ```
   - Connect True branch of "Check If Complete" to this node.

9. **Add a Sticky Note for Documentation**
   - Content:
     ```
     ## Justingirard Draft Ui Designer Image Generator

     This workflow uses the **justingirard/draft-ui-designer** model from Replicate to generate image content.

     ### Setup
     1. Add your Replicate API key
     2. Configure the input parameters
     3. Run the workflow

     ### Model Details
     - **Type**: Image Generation
     - **Provider**: justingirard
     - **Required Fields**: prompt
     ```
   - Place visually separate from main flow.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                     | Context or Link                                                                                          |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| This workflow uses the Replicate API for AI-generated UI design images via the justingirard/draft-ui-designer model.                                            | Official model on Replicate: https://replicate.com/justingirard/draft-ui-designer                        |
| Polling interval is set to 2 seconds, which balances responsiveness and avoids API rate limiting. Adjust if needed based on Replicate API constraints.           | N8n HTTP Request node best practices for rate limits                                                   |
| The prompt parameter in the prediction request is currently hardcoded as `"prompt value"`. For real use, it should be parameterized or dynamically set.          | Workflow customization recommendation                                                                 |
| Missing or invalid API key will cause authentication errors; ensure the API key is valid and kept secure.                                                      | Replicate API documentation for authentication                                                        |
| The workflow does not explicitly handle prediction failure states (e.g., "failed", "canceled"). Adding error handling would improve robustness.                | Suggested improvement for production environments                                                     |

---

**Disclaimer:**  
The provided text originates solely from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.