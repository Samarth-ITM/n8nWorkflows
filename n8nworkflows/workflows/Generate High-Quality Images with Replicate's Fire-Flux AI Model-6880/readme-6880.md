Generate High-Quality Images with Replicate's Fire/Flux AI Model

https://n8nworkflows.xyz/workflows/generate-high-quality-images-with-replicate-s-fire-flux-ai-model-6880


# Generate High-Quality Images with Replicate's Fire/Flux AI Model

### 1. Workflow Overview

The **Fire Flux Image Generator** workflow is designed to generate high-quality images using Replicate’s **fire/flux** AI model. It targets users who want to automate the process of sending image generation prompts to the model, monitoring the generation status, and retrieving the resulting images once ready.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Initialization**  
  Triggering the workflow manually and setting up the Replicate API key.

- **1.2 Image Generation Request**  
  Sending the prompt and parameters to Replicate’s Fire/Flux model to create a prediction.

- **1.3 Prediction Monitoring**  
  Extracting the prediction ID and polling the Replicate API to check the status of the image generation.

- **1.4 Result Processing**  
  Once the prediction is complete, processing and formatting the output data for downstream use.

- **1.5 User Guidance (Sticky Note)**  
  Provides setup instructions and model details for the user.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Initialization

**Overview:**  
This block starts the workflow manually and sets up the necessary API key for subsequent authenticated calls to Replicate.

**Nodes Involved:**  
- On clicking 'execute'  
- Set API Key

**Node Details:**

- **On clicking 'execute'**  
  - Type: Manual Trigger  
  - Role: Entry point to start the workflow manually.  
  - Configuration: No parameters; simply triggers execution on demand.  
  - Input: None  
  - Output: Triggers "Set API Key" node.  
  - Edge Cases: None.

- **Set API Key**  
  - Type: Set  
  - Role: Assigns the Replicate API key as a workflow variable for authorization.  
  - Configuration: Stores the API key string in the variable `replicate_api_key`. The key is expected to be replaced with the user's actual Replicate API key.  
  - Input: Trigger from Manual Trigger node.  
  - Output: Passes the API key to "Create Prediction" node.  
  - Edge Cases: Missing or invalid API key will cause authorization failures downstream.

#### 1.2 Image Generation Request

**Overview:**  
Sends a POST request to Replicate’s API to initiate the image generation with the fire/flux model using specified parameters.

**Nodes Involved:**  
- Create Prediction

**Node Details:**

- **Create Prediction**  
  - Type: HTTP Request  
  - Role: Sends the image generation request to Replicate API.  
  - Configuration:  
    - Method: POST  
    - URL: `https://api.replicate.com/v1/predictions`  
    - Header: Authorization Bearer token with API key, Content-Type: application/json  
    - Body (JSON):  
      ```json
      {
        "version": "da7b098b70c3e52d56a04c2b7f3632eb63dfb0f26941a0934bf223157a7c5245",
        "input": {
          "prompt": "prompt value",
          "go_fast": true,
          "num_outputs": 1,
          "output_quality": 80,
          "num_inference_steps": 4
        }
      }
      ```  
      The prompt is currently a static placeholder `"prompt value"` and should be replaced or parameterized as needed.  
  - Input: Receives API key from "Set API Key"  
  - Output: Response JSON with prediction metadata (including prediction ID).  
  - Edge Cases:  
    - HTTP errors (timeout, 401 Unauthorized if API key invalid)  
    - Malformed JSON body  
    - API rate limiting

#### 1.3 Prediction Monitoring

**Overview:**  
Extracts the prediction ID from the initial response and repeatedly polls the Replicate API until the image generation is complete.

**Nodes Involved:**  
- Extract Prediction ID  
- Wait  
- Check Prediction Status  
- Check If Complete

**Node Details:**

- **Extract Prediction ID**  
  - Type: Code  
  - Role: Extracts the prediction ID and initial status from the API response to construct the URL for polling.  
  - Configuration: JavaScript code returns:  
    - `predictionId` (string)  
    - `status` (string)  
    - `predictionUrl` built as `https://api.replicate.com/v1/predictions/{predictionId}`  
  - Input: Output from "Create Prediction"  
  - Output: Object with prediction metadata for polling  
  - Edge Cases: Missing or malformed prediction ID in response.

- **Wait**  
  - Type: Wait  
  - Role: Pauses workflow execution for 2 seconds before polling the status again.  
  - Configuration: Wait time = 2 seconds  
  - Input: Output of "Extract Prediction ID" initially, or from "Check If Complete" when status is not succeeded.  
  - Output: Triggers "Check Prediction Status" after delay.

- **Check Prediction Status**  
  - Type: HTTP Request  
  - Role: Polls the prediction status URL to get current generation status.  
  - Configuration:  
    - URL: dynamic, from `predictionUrl` variable  
    - Method: GET (default)  
    - Auth: Bearer token with API key  
  - Input: From "Wait"  
  - Output: Current prediction status JSON  
  - Edge Cases: HTTP errors, network issues, or invalid prediction URL.

- **Check If Complete**  
  - Type: If  
  - Role: Branching logic to check if the prediction status is `"succeeded"`.  
  - Configuration: Boolean condition where `$json.status` equals `"succeeded"`.  
  - Input: From "Check Prediction Status"  
  - Output:  
    - True branch: To "Process Result" node (finalization)  
    - False branch: To "Wait" node (poll again)  
  - Edge Cases: Possible statuses other than "succeeded" (e.g., "failed", "processing"). No explicit failure handling here.

#### 1.4 Result Processing

**Overview:**  
Processes the successful prediction output by extracting relevant fields and formatting them for further use or output.

**Nodes Involved:**  
- Process Result

**Node Details:**

- **Process Result**  
  - Type: Code  
  - Role: Extracts and structures the result data from the completed prediction.  
  - Configuration: JavaScript code returning:  
    - `status`  
    - `output` (image URL)  
    - `metrics`  
    - `created_at`  
    - `completed_at`  
    - `model` (hardcoded `"fire/flux"`)  
    - `image_url` (same as output)  
  - Input: From the true branch of "Check If Complete"  
  - Output: Processed object with image generation details  
  - Edge Cases: If output is missing or unexpected format, may cause errors.

#### 1.5 User Guidance (Sticky Note)

**Overview:**  
Provides user instructions and model details for configuring and understanding the workflow.

**Nodes Involved:**  
- Sticky Note

**Node Details:**

- **Sticky Note**  
  - Type: Sticky Note  
  - Role: Documentation within the workflow canvas.  
  - Content:  
    - Title: Fire Flux Image Generator  
    - Setup instructions:  
      1. Add Replicate API key  
      2. Configure input parameters  
      3. Run workflow  
    - Model details: Image generation, provider `fire`, required field `prompt`  
  - Input/Output: None  
  - Edge Cases: N/A

---

### 3. Summary Table

| Node Name               | Node Type         | Functional Role                             | Input Node(s)            | Output Node(s)           | Sticky Note                                                                                   |
|-------------------------|-------------------|---------------------------------------------|--------------------------|--------------------------|-----------------------------------------------------------------------------------------------|
| On clicking 'execute'    | Manual Trigger    | Manual start of the workflow                | —                        | Set API Key              |                                                                                               |
| Set API Key             | Set               | Stores Replicate API key for authentication| On clicking 'execute'     | Create Prediction        |                                                                                               |
| Create Prediction       | HTTP Request      | Sends image generation request to Replicate| Set API Key              | Extract Prediction ID    |                                                                                               |
| Extract Prediction ID   | Code              | Extracts prediction ID and constructs poll URL| Create Prediction        | Wait                     |                                                                                               |
| Wait                    | Wait              | Pauses for 2 seconds before polling again  | Extract Prediction ID, Check If Complete (false branch) | Check Prediction Status   |                                                                                               |
| Check Prediction Status | HTTP Request      | Polls Replicate API for prediction status  | Wait                     | Check If Complete        |                                                                                               |
| Check If Complete       | If                | Checks if prediction has completed          | Check Prediction Status  | Process Result (true), Wait (false) |                                                                                               |
| Process Result          | Code              | Extracts and formats final prediction output| Check If Complete (true) | —                        |                                                                                               |
| Sticky Note             | Sticky Note       | Provides setup and model information        | —                        | —                        | ## Fire Flux Image Generator<br>This workflow uses the **fire/flux** model from Replicate to generate image content.<br>### Setup<br>1. Add your Replicate API key<br>2. Configure the input parameters<br>3. Run the workflow<br>### Model Details<br>- **Type**: Image Generation<br>- **Provider**: fire<br>- **Required Fields**: prompt |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: `On clicking 'execute'`  
   - Purpose: To start the workflow manually.

2. **Create Set Node**  
   - Name: `Set API Key`  
   - Purpose: Store your Replicate API key.  
   - Configuration: Add a string field named `replicate_api_key` with your actual API key value (replace `"YOUR_REPLICATE_API_KEY"`).  
   - Connect output from `On clicking 'execute'` to this node.

3. **Create HTTP Request Node**  
   - Name: `Create Prediction`  
   - Purpose: Send POST request to Replicate API to start image generation.  
   - Configuration:  
     - HTTP Method: POST  
     - URL: `https://api.replicate.com/v1/predictions`  
     - Authentication: Generic HTTP Header with header parameter:  
       - Name: `Authorization`  
       - Value: `Bearer {{$node["Set API Key"].json["replicate_api_key"]}}`  
     - Headers: Add `Content-Type: application/json`  
     - Body Content Type: JSON  
     - Body:  
       ```json
       {
         "version": "da7b098b70c3e52d56a04c2b7f3632eb63dfb0f26941a0934bf223157a7c5245",
         "input": {
           "prompt": "prompt value",
           "go_fast": true,
           "num_outputs": 1,
           "output_quality": 80,
           "num_inference_steps": 4
         }
       }
       ```  
       Replace `"prompt value"` with your desired prompt, or parameterize it as input.  
   - Connect output from `Set API Key` to this node.

4. **Create Code Node**  
   - Name: `Extract Prediction ID`  
   - Purpose: Extract prediction ID and build polling URL.  
   - Configuration: Run once per item mode with this JavaScript code:  
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
   - Connect output from `Create Prediction` to this node.

5. **Create Wait Node**  
   - Name: `Wait`  
   - Purpose: Pause 2 seconds between status polls.  
   - Configuration: Amount = 2, Unit = seconds  
   - Connect output from `Extract Prediction ID` to this node.

6. **Create HTTP Request Node**  
   - Name: `Check Prediction Status`  
   - Purpose: Poll Replicate API for current prediction status.  
   - Configuration:  
     - URL: `{{$json["predictionUrl"]}}` (dynamic from previous node)  
     - Authentication: Generic HTTP Header with header parameter:  
       - Name: `Authorization`  
       - Value: `Bearer {{$node["Set API Key"].json["replicate_api_key"]}}`  
     - Method: GET (default)  
   - Connect output from `Wait` to this node.

7. **Create If Node**  
   - Name: `Check If Complete`  
   - Purpose: Check if prediction status is "succeeded".  
   - Configuration: Boolean condition: `$json["status"] == "succeeded"`  
   - Connect output from `Check Prediction Status` to this node.

8. **Create Code Node**  
   - Name: `Process Result`  
   - Purpose: Extract final prediction details.  
   - Configuration: Run once per item mode with this JavaScript code:  
     ```javascript
     const result = $input.item.json;

     return {
       status: result.status,
       output: result.output,
       metrics: result.metrics,
       created_at: result.created_at,
       completed_at: result.completed_at,
       model: 'fire/flux',
       image_url: result.output
     };
     ```  
   - Connect **true** output of `Check If Complete` to this node.

9. **Connect false output of `Check If Complete` back to `Wait`** to continue polling.

10. **Add Sticky Note Node** (optional but recommended)  
    - Name: `Sticky Note`  
    - Content:  
      ```
      ## Fire Flux Image Generator

      This workflow uses the **fire/flux** model from Replicate to generate image content.

      ### Setup
      1. Add your Replicate API key
      2. Configure the input parameters
      3. Run the workflow

      ### Model Details
      - **Type**: Image Generation
      - **Provider**: fire
      - **Required Fields**: prompt
      ```
    - Place near start nodes for user guidance.

---

### 5. General Notes & Resources

| Note Content                                                                                  | Context or Link                                          |
|-----------------------------------------------------------------------------------------------|----------------------------------------------------------|
| The workflow uses Replicate’s API version `da7b098b70c3e52d56a04c2b7f3632eb63dfb0f26941a0934bf223157a7c5245` for the fire/flux model. | Replicate API model version identifier                    |
| Replace `"prompt value"` in the `Create Prediction` node with a dynamic input or variable to customize generated images. | Essential for practical usage                             |
| Polling interval is fixed at 2 seconds; adjust `Wait` node if faster or slower polling is preferred. | Trade-off between responsiveness and API rate limits      |
| No explicit error handling for prediction failures or API errors is implemented; consider adding error nodes or alternate flows. | Robustness improvement suggestion                         |
| Authorization requires a valid Replicate API key; ensure it has sufficient permissions and is kept secure. | Security best practice                                   |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All manipulated data are legal and public.