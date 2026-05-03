Generate AI Images with Prunaai Flux.1 Dev Model via Replicate API

https://n8nworkflows.xyz/workflows/generate-ai-images-with-prunaai-flux-1-dev-model-via-replicate-api-7121


# Generate AI Images with Prunaai Flux.1 Dev Model via Replicate API

### 1. Workflow Overview

This workflow, named **Prunaai Flux.1 Dev Image Generator**, automates the generation of AI images using the **prunaai/flux.1-dev** model via the Replicate API. It is designed for users who want to generate images from textual prompts by leveraging Replicate’s hosted AI models.

The workflow is logically divided into these main blocks:

- **1.1 Input Reception:** Manual trigger to start the workflow.
- **1.2 API Key Setup:** Assign the Replicate API key for authorization.
- **1.3 Prediction Creation:** Submit a request to Replicate to generate an image based on the prompt and model parameters.
- **1.4 Prediction Monitoring:** Poll the prediction endpoint until the image generation is complete.
- **1.5 Result Processing:** Extract and structure the image generation results.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  The workflow starts with a manual trigger node allowing the user to initiate the image generation process on demand.

- **Nodes Involved:**  
  - On clicking 'execute'

- **Node Details:**  
  - **On clicking 'execute'**  
    - Type: Manual Trigger  
    - Role: Entry point to manually start the workflow.  
    - Configuration: No parameters; triggers workflow execution when clicked.  
    - Inputs: None  
    - Outputs: Connects to "Set API Key" node.  
    - Edge cases: None expected.  
    - Version: 1

#### 1.2 API Key Setup

- **Overview:**  
  This block assigns the Replicate API key to a workflow variable to be used for authenticated API calls.

- **Nodes Involved:**  
  - Set API Key

- **Node Details:**  
  - **Set API Key**  
    - Type: Set Node  
    - Role: Stores the Replicate API key as a string variable `replicate_api_key`.  
    - Configuration: User must replace `"YOUR_REPLICATE_API_KEY"` with their actual API key.  
    - Key variables: `replicate_api_key`  
    - Inputs: From manual trigger  
    - Outputs: To "Create Prediction" node  
    - Edge cases: Missing or invalid API key will cause authentication errors downstream.  
    - Version: 3.3

#### 1.3 Prediction Creation

- **Overview:**  
  Submits a POST request to the Replicate API to start image generation with the specified model version and input parameters.

- **Nodes Involved:**  
  - Create Prediction

- **Node Details:**  
  - **Create Prediction**  
    - Type: HTTP Request  
    - Role: Calls Replicate API to create an image generation prediction.  
    - Configuration:  
      - URL: `https://api.replicate.com/v1/predictions`  
      - Method: POST  
      - Body: JSON specifying model version `b0306d92aa025bb747dc74162f3c27d6ed83798e08e5f8977adf3d859d0536a3` and input parameters including prompt (currently literal "prompt value"), seed, guidance, image size, output quality, and inference steps.  
      - Headers: Authorization using Bearer token from `replicate_api_key`, Content-Type `application/json`.  
      - Timeout: 60 seconds  
    - Key expressions: Authorization header uses expression `={{ 'Bearer ' + $('Set API Key').item.json.replicate_api_key }}`  
    - Inputs: From "Set API Key"  
    - Outputs: To "Extract Prediction ID"  
    - Edge cases: API errors, timeout, invalid prompt or parameters, authentication errors.  
    - Version: 4.2

#### 1.4 Prediction Monitoring

- **Overview:**  
  This block handles polling the Replicate API prediction endpoint repeatedly until the prediction status becomes "succeeded".

- **Nodes Involved:**  
  - Extract Prediction ID  
  - Wait  
  - Check Prediction Status  
  - Check If Complete

- **Node Details:**  
  - **Extract Prediction ID**  
    - Type: Code  
    - Role: Extracts the prediction ID and initial status from the API response and builds the URL to poll.  
    - Configuration: Custom JavaScript extracts `id` and `status` from input JSON, returns these plus a constructed prediction URL.  
    - Inputs: From "Create Prediction"  
    - Outputs: To "Wait"  
    - Edge cases: Missing or malformed response data.  
    - Version: 2

  - **Wait**  
    - Type: Wait  
    - Role: Pauses workflow execution for 2 seconds before polling again to avoid API rate limits.  
    - Configuration: Wait 2 seconds  
    - Inputs: From "Extract Prediction ID" and "Check If Complete" (loop)  
    - Outputs: To "Check Prediction Status"  
    - Edge cases: None significant.

  - **Check Prediction Status**  
    - Type: HTTP Request  
    - Role: Sends GET request to the prediction URL to check current status.  
    - Configuration:  
      - URL: from `$json.predictionUrl` (dynamic)  
      - Headers: Authorization with Bearer token from API key  
      - Method: GET (default)  
    - Inputs: From "Wait"  
    - Outputs: To "Check If Complete"  
    - Edge cases: API errors, network issues, authorization problems.  
    - Version: 4.2

  - **Check If Complete**  
    - Type: If  
    - Role: Checks if the prediction status equals "succeeded".  
    - Configuration: Boolean condition comparing `$json.status` to "succeeded".  
    - Inputs: From "Check Prediction Status"  
    - Outputs:  
      - True branch: To "Process Result"  
      - False branch: Loops back to "Wait" for continued polling  
    - Edge cases: Other statuses like "failed" or "canceled" are not explicitly handled.

#### 1.5 Result Processing

- **Overview:**  
  Processes the final successful prediction result, extracting relevant metadata and the image URL for further use or display.

- **Nodes Involved:**  
  - Process Result

- **Node Details:**  
  - **Process Result**  
    - Type: Code  
    - Role: Extracts fields such as status, output URL, metrics, timestamps, and sets a fixed model name.  
    - Configuration: JavaScript returns a clean JSON object with keys: status, output, metrics, created_at, completed_at, model, and image_url (same as output).  
    - Inputs: From "Check If Complete" true branch  
    - Outputs: Terminal node (no further connections)  
    - Edge cases: Assumes result.output contains a valid image URL or array; no error handling for missing data.

#### Additional

- **Sticky Note**

  - Provides documentation and instructions:  
    - Explains the workflow’s purpose and the model used.  
    - Setup steps: add API key, configure inputs, run workflow.  
    - Model details and required fields.  
  - Positioned visually at the start, no functional role in execution.

---

### 3. Summary Table

| Node Name               | Node Type         | Functional Role                      | Input Node(s)          | Output Node(s)          | Sticky Note                                                                                                            |
|-------------------------|-------------------|------------------------------------|------------------------|-------------------------|------------------------------------------------------------------------------------------------------------------------|
| On clicking 'execute'    | Manual Trigger    | Starts the workflow manually       | —                      | Set API Key             |                                                                                                                        |
| Set API Key             | Set               | Stores Replicate API key            | On clicking 'execute'   | Create Prediction       |                                                                                                                        |
| Create Prediction       | HTTP Request      | Sends image generation request     | Set API Key             | Extract Prediction ID   |                                                                                                                        |
| Extract Prediction ID   | Code              | Extracts prediction ID & status    | Create Prediction       | Wait                    |                                                                                                                        |
| Wait                    | Wait              | Pauses workflow for polling delay | Extract Prediction ID, Check If Complete (false branch) | Check Prediction Status |                                                                                                                        |
| Check Prediction Status | HTTP Request      | Polls prediction status            | Wait                    | Check If Complete       |                                                                                                                        |
| Check If Complete       | If                | Checks if prediction completed     | Check Prediction Status | Process Result (true), Wait (false) |                                                                                                                        |
| Process Result          | Code              | Processes final prediction data    | Check If Complete (true) | —                      |                                                                                                                        |
| Sticky Note             | Sticky Note       | Provides workflow documentation    | —                      | —                       | ## Prunaai Flux.1 Dev Image Generator\n\nThis workflow uses the **prunaai/flux.1-dev** model from Replicate to generate image content.\n\n### Setup\n1. Add your Replicate API key\n2. Configure the input parameters\n3. Run the workflow\n\n### Model Details\n- **Type**: Image Generation\n- **Provider**: prunaai\n- **Required Fields**: prompt |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Manual Trigger node**  
   - Name: `On clicking 'execute'`  
   - No additional configuration required.

3. **Add a Set node**  
   - Name: `Set API Key`  
   - Add a string field named `replicate_api_key` with value `"YOUR_REPLICATE_API_KEY"` (replace with your actual API key).  
   - Connect `On clicking 'execute'` output to this node’s input.

4. **Add an HTTP Request node**  
   - Name: `Create Prediction`  
   - Set HTTP Method to POST  
   - URL: `https://api.replicate.com/v1/predictions`  
   - Request body format: JSON  
   - Body JSON:  
     ```json
     {
       "version": "b0306d92aa025bb747dc74162f3c27d6ed83798e08e5f8977adf3d859d0536a3",
       "input": {
         "prompt": "prompt value",
         "seed": -1,
         "guidance": 3.5,
         "image_size": 1024,
         "output_quality": 80,
         "num_inference_steps": 28
       }
     }
     ```  
   - Add HTTP Header Authentication:  
     - Type: Generic Credential (HTTP Header Auth)  
     - Header Name: Authorization  
     - Header Value: `={{ 'Bearer ' + $('Set API Key').item.json.replicate_api_key }}`  
     - Additionally set header `Content-Type: application/json`  
   - Timeout: 60 seconds  
   - Connect output of `Set API Key` to this node.

5. **Add a Code node**  
   - Name: `Extract Prediction ID`  
   - Set mode to “Run once for each item”  
   - Paste this JavaScript code:  
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
   - Connect output of `Create Prediction` to this node.

6. **Add a Wait node**  
   - Name: `Wait`  
   - Configure to wait 2 seconds  
   - Connect output of `Extract Prediction ID` to this node.

7. **Add another HTTP Request node**  
   - Name: `Check Prediction Status`  
   - Set HTTP Method to GET (default)  
   - URL: `={{ $json.predictionUrl }}` (dynamic expression)  
   - Authentication: Generic Credential (HTTP Header Auth)  
   - Header Name: Authorization  
   - Header Value: `={{ 'Bearer ' + $('Set API Key').item.json.replicate_api_key }}`  
   - Connect output of `Wait` to this node.

8. **Add an If node**  
   - Name: `Check If Complete`  
   - Condition type: Boolean  
   - Expression: Check if `$json.status` equals `"succeeded"`  
   - Connect output of `Check Prediction Status` to this node.

9. **Add a Code node**  
   - Name: `Process Result`  
   - Mode: Run once for each item  
   - JavaScript code:  
     ```js
     const result = $input.item.json;

     return {
       status: result.status,
       output: result.output,
       metrics: result.metrics,
       created_at: result.created_at,
       completed_at: result.completed_at,
       model: 'prunaai/flux.1-dev',
       image_url: result.output
     };
     ```  
   - Connect the “true” output of `Check If Complete` to this node.

10. **Connect the “false” output of `Check If Complete` back to `Wait`**  
    - This creates a polling loop to keep checking status until completion.

11. **(Optional) Add a Sticky Note node**  
    - Add documentation content describing the workflow, setup steps, and model details for clarity.

12. **Credentials Setup:**  
    - Create Credential for Generic HTTP Header Authentication with no username/password needed; the Bearer token is passed dynamically via header in nodes.  
    - Ensure your Replicate API key is valid and has access to the model.

13. **Final Checks:**  
    - Replace `"prompt value"` in the `Create Prediction` node’s JSON body with the actual prompt string or an expression if dynamic input is desired.  
    - Save and activate the workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                                  |
|-------------------------------------------------------------------------------------------------|-----------------------------------------------------------------|
| This workflow uses the **prunaai/flux.1-dev** model to generate images through Replicate API.   | Explained in Sticky Note node content.                           |
| Setup requires a valid Replicate API key with sufficient permissions for the model.             | API key set in "Set API Key" node.                              |
| Model version used: `b0306d92aa025bb747dc74162f3c27d6ed83798e08e5f8977adf3d859d0536a3`           | Required for the Replicate API call in "Create Prediction" node.|
| Polling interval set to 2 seconds to avoid excessive API requests and respect rate limits.      | Configured in "Wait" node.                                       |

---

**Disclaimer:** The provided text is extracted exclusively from an n8n automated workflow. It complies strictly with prevailing content policies and contains no illegal or protected material. All data handled are legal and public.