Generate Images with prunaai Hidream E1.1 Model via Replicate API

https://n8nworkflows.xyz/workflows/generate-images-with-prunaai-hidream-e1-1-model-via-replicate-api-7114


# Generate Images with prunaai Hidream E1.1 Model via Replicate API

---
### 1. Workflow Overview

This workflow titled **"Prunaai Hidream E1.1 Image Generator"** automates the generation of images using the **prunaai/hidream-e1.1** AI model via the Replicate API. It is designed for users seeking to create AI-generated images based on textual prompts, encapsulating the entire lifecycle from input trigger, API authentication, prediction creation, status polling, and result processing.

**Target Use Cases:**  
- Automated generation of images from text prompts using AI.  
- Integration of Replicate's image generation model into larger automation pipelines.  
- Batch or manual execution for creative or content production workflows.

**Logical Blocks:**

- **1.1 Input Reception & Initialization:** Manual trigger and API key setup.  
- **1.2 Creating Image Prediction:** Sending the generation request to Replicate API.  
- **1.3 Prediction Tracking & Polling:** Extracting the prediction ID, waiting, and repeatedly checking status until completion.  
- **1.4 Result Processing:** Handling the final output, extracting relevant data for further use or storage.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Initialization

**Overview:**  
This block initiates the workflow manually and sets the necessary API key for authentication with the Replicate service.

**Nodes Involved:**  
- On clicking 'execute' (Manual Trigger)  
- Set API Key

**Node Details:**

- **On clicking 'execute'**  
  - *Type:* Manual Trigger  
  - *Role:* Entry point to start the workflow on demand.  
  - *Configuration:* Defaults; no parameters set.  
  - *Inputs:* None  
  - *Outputs:* Connects to "Set API Key" node.  
  - *Edge Cases:* None; user-initiated.

- **Set API Key**  
  - *Type:* Set node  
  - *Role:* Assigns the Replicate API key to a workflow variable for authentication in subsequent requests.  
  - *Configuration:*  
    - Variable name: `replicate_api_key`  
    - Value: `"YOUR_REPLICATE_API_KEY"` (placeholder to be replaced by user)  
  - *Inputs:* From Manual Trigger  
  - *Outputs:* Connects to "Create Prediction" node.  
  - *Edge Cases:* Missing or invalid API key will cause authentication failures downstream.

---

#### 1.2 Creating Image Prediction

**Overview:**  
This block sends a POST request to the Replicate API to start an image generation prediction using the given model version and input parameters.

**Nodes Involved:**  
- Create Prediction

**Node Details:**

- **Create Prediction**  
  - *Type:* HTTP Request  
  - *Role:* Initiates the image generation process on Replicate.  
  - *Configuration:*  
    - URL: `https://api.replicate.com/v1/predictions`  
    - Method: POST  
    - Body (JSON):  
      ```json
      {
        "version": "433436facdc1172b6efcb801eb6f345d7858a32200d24e5febaccfb4b44ad66f",
        "input": {
          "prompt": "prompt value",
          "seed": -1,
          "clip_cfg_norm": true,
          "guidance_scale": 2.5,
          "output_quality": 100,
          "refine_strength": 0.3,
          "num_inference_steps": 28,
          "image_guidance_scale": 1
        }
      }
      ```
    - Authentication: HTTP Header with Bearer token from `Set API Key` node (`Authorization: Bearer {replicate_api_key}`)  
    - Headers: Content-Type: application/json  
    - Timeout: 60 seconds  
  - *Inputs:* From "Set API Key"  
  - *Outputs:* Connects to "Extract Prediction ID"  
  - *Key Expressions:* Uses expression to inject API key dynamically in headers.  
  - *Edge Cases:*  
    - Network errors or timeouts.  
    - 4xx/5xx HTTP errors.  
    - Incorrect prompt or malformed input JSON.  
    - API key invalid or missing.

---

#### 1.3 Prediction Tracking & Polling

**Overview:**  
After initiating the prediction, this block extracts the prediction ID, waits briefly, polls the status repeatedly until the prediction is complete, and branches logic accordingly.

**Nodes Involved:**  
- Extract Prediction ID  
- Wait  
- Check Prediction Status  
- Check If Complete

**Node Details:**

- **Extract Prediction ID**  
  - *Type:* Code (JavaScript)  
  - *Role:* Parses the response from "Create Prediction" to extract the prediction ID and initial status, and constructs the URL for subsequent status polling.  
  - *Configuration:*  
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
  - *Inputs:* From "Create Prediction"  
  - *Outputs:* Connects to "Wait"  
  - *Edge Cases:* Missing `id` or `status` in response leading to runtime errors.

- **Wait**  
  - *Type:* Wait  
  - *Role:* Delays execution for 2 seconds before polling status to avoid rate limits or excessive requests.  
  - *Configuration:* 2 seconds wait  
  - *Inputs:* From "Extract Prediction ID" or from "Check If Complete" (if prediction incomplete)  
  - *Outputs:* Connects to "Check Prediction Status"  
  - *Edge Cases:* None significant; delay configurable.

- **Check Prediction Status**  
  - *Type:* HTTP Request  
  - *Role:* Polls the Replicate API endpoint for the current prediction status.  
  - *Configuration:*  
    - URL: dynamic `{{$json.predictionUrl}}` from previous node  
    - Method: GET (default)  
    - Authentication: Bearer token in headers (same as before)  
  - *Inputs:* From "Wait"  
  - *Outputs:* Connects to "Check If Complete"  
  - *Edge Cases:* Network errors, API rate limits, invalid URL, expired token.

- **Check If Complete**  
  - *Type:* If  
  - *Role:* Tests if the prediction status equals `"succeeded"`.  
  - *Configuration:* Boolean condition: `$json.status === "succeeded"`  
  - *Inputs:* From "Check Prediction Status"  
  - *Outputs:*  
    - True branch: "Process Result"  
    - False branch: "Wait" (loop for polling)  
  - *Edge Cases:* Status could be `"failed"` or other values, which are not explicitly handled here (could cause indefinite polling).

---

#### 1.4 Result Processing

**Overview:**  
Upon successful completion, this block processes the prediction output and prepares relevant data for downstream use or storage.

**Nodes Involved:**  
- Process Result

**Node Details:**

- **Process Result**  
  - *Type:* Code (JavaScript)  
  - *Role:* Extracts and structures key output fields such as status, output image URLs, metrics, timestamps, and model info.  
  - *Configuration:*  
    ```js
    const result = $input.item.json;

    return {
      status: result.status,
      output: result.output,
      metrics: result.metrics,
      created_at: result.created_at,
      completed_at: result.completed_at,
      model: 'prunaai/hidream-e1.1',
      image_url: result.output
    };
    ```
  - *Inputs:* From "Check If Complete" (true branch)  
  - *Outputs:* None (end of workflow)  
  - *Edge Cases:* If output field is missing or malformed, downstream usage might fail.

---

### 3. Summary Table

| Node Name           | Node Type       | Functional Role                             | Input Node(s)               | Output Node(s)              | Sticky Note                                                                                  |
|---------------------|-----------------|---------------------------------------------|-----------------------------|-----------------------------|---------------------------------------------------------------------------------------------|
| On clicking 'execute'| Manual Trigger  | Entry point to start the workflow           | None                        | Set API Key                 |                                                                                             |
| Set API Key         | Set             | Assigns Replicate API key for authentication| On clicking 'execute'       | Create Prediction           |                                                                                             |
| Create Prediction   | HTTP Request    | Sends image generation request to Replicate| Set API Key                 | Extract Prediction ID       |                                                                                             |
| Extract Prediction ID| Code            | Extracts prediction ID and status            | Create Prediction           | Wait                        |                                                                                             |
| Wait                | Wait            | Pauses workflow before polling status       | Extract Prediction ID, Check If Complete (false) | Check Prediction Status    |                                                                                             |
| Check Prediction Status| HTTP Request  | Polls Replicate API for prediction status   | Wait                        | Check If Complete           |                                                                                             |
| Check If Complete   | If              | Branches based on prediction completion status| Check Prediction Status    | Process Result, Wait        |                                                                                             |
| Process Result      | Code            | Processes and formats final prediction output| Check If Complete (true)    | None                       |                                                                                             |
| Sticky Note        | Sticky Note     | Workflow description and setup instructions | None                        | None                       | ## Prunaai Hidream E1.1 Image Generator<br>This workflow uses the **prunaai/hidream-e1.1** model from Replicate to generate image content.<br>### Setup<br>1. Add your Replicate API key<br>2. Configure the input parameters<br>3. Run the workflow<br>### Model Details<br>- **Type**: Image Generation<br>- **Provider**: prunaai<br>- **Required Fields**: prompt |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: `On clicking 'execute'`  
   - Purpose: Entry point to manually execute the workflow.  
   - No special configuration needed.

2. **Create Set Node**  
   - Name: `Set API Key`  
   - Purpose: Store Replicate API key for authentication.  
   - Configuration:  
     - Create a string variable `replicate_api_key` with value set to your actual Replicate API key (replace placeholder `"YOUR_REPLICATE_API_KEY"`).  
   - Connect output of `On clicking 'execute'` to this node.

3. **Create HTTP Request Node**  
   - Name: `Create Prediction`  
   - Purpose: Send POST request to Replicate API to start image generation.  
   - Configuration:  
     - URL: `https://api.replicate.com/v1/predictions`  
     - Method: POST  
     - Authentication: HTTP Header Auth  
       - Header Name: `Authorization`  
       - Header Value: `Bearer {{$node["Set API Key"].json["replicate_api_key"]}}`  
     - Additional Header: `Content-Type: application/json`  
     - Request Body Type: JSON  
     - JSON Body:  
       ```json
       {
         "version": "433436facdc1172b6efcb801eb6f345d7858a32200d24e5febaccfb4b44ad66f",
         "input": {
           "prompt": "prompt value",
           "seed": -1,
           "clip_cfg_norm": true,
           "guidance_scale": 2.5,
           "output_quality": 100,
           "refine_strength": 0.3,
           "num_inference_steps": 28,
           "image_guidance_scale": 1
         }
       }
       ```  
     - Timeout: 60 seconds  
   - Connect output of `Set API Key` to this node.

4. **Create Code Node**  
   - Name: `Extract Prediction ID`  
   - Purpose: Extract prediction ID and initial status from API response and build status URL.  
   - Configuration:  
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

5. **Create Wait Node**  
   - Name: `Wait`  
   - Purpose: Delay polling to prevent API overload.  
   - Configuration: 2 seconds wait.  
   - Connect output of `Extract Prediction ID` to this node.

6. **Create HTTP Request Node**  
   - Name: `Check Prediction Status`  
   - Purpose: Poll Replicate API for current prediction status.  
   - Configuration:  
     - URL: `{{$json["predictionUrl"]}}` (dynamic from previous node)  
     - Method: GET (default)  
     - Authentication: HTTP Header Auth  
       - Header Name: `Authorization`  
       - Header Value: `Bearer {{$node["Set API Key"].json["replicate_api_key"]}}`  
   - Connect output of `Wait` to this node.

7. **Create If Node**  
   - Name: `Check If Complete`  
   - Purpose: Determine if prediction has succeeded.  
   - Configuration:  
     - Condition Type: Boolean  
     - Expression: `$json["status"] === "succeeded"`  
   - Connect output of `Check Prediction Status` to this node.

8. **Create Code Node**  
   - Name: `Process Result`  
   - Purpose: Format final prediction output for downstream use.  
   - Configuration:  
     ```js
     const result = $input.item.json;

     return {
       status: result.status,
       output: result.output,
       metrics: result.metrics,
       created_at: result.created_at,
       completed_at: result.completed_at,
       model: 'prunaai/hidream-e1.1',
       image_url: result.output
     };
     ```  
   - Connect **true** branch of `Check If Complete` to this node.

9. **Connect False Branch of `Check If Complete` back to `Wait` Node**  
   - This closes the polling loop until the prediction status is `"succeeded"`.

10. **Optional: Add a Sticky Note**  
    - Content:  
      ```
      ## Prunaai Hidream E1.1 Image Generator

      This workflow uses the **prunaai/hidream-e1.1** model from Replicate to generate image content.

      ### Setup
      1. Add your Replicate API key
      2. Configure the input parameters
      3. Run the workflow

      ### Model Details
      - Type: Image Generation
      - Provider: prunaai
      - Required Fields: prompt
      ```

---

### 5. General Notes & Resources

| Note Content                                                                                           | Context or Link                                                                                  |
|------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| The Replicate API requires an API key which can be obtained by creating an account at https://replicate.com/ | Replicate official website                                                                       |
| The model version used (`433436facdc1172b6efcb801eb6f345d7858a32200d24e5febaccfb4b44ad66f`) is specific to prunaai/hidream-e1.1 and should be updated if a newer version is desired | Model versioning details at Replicate API documentation                                          |
| The workflow polls the API every 2 seconds; adjust `Wait` node timing if API rate limits or response times vary | To avoid API rate limit errors or excessive delays                                              |
| No explicit error handling for failed or canceled predictions is implemented; consider adding logic to manage statuses other than "succeeded" | Enhancing robustness and failure detection                                                      |

---

**Disclaimer:**  
The provided text originates exclusively from an automation workflow created with n8n, a tool for integration and automation. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly available.