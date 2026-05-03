Generate Videos from Images with Wan 2.2 I2V A14B AI Model

https://n8nworkflows.xyz/workflows/generate-videos-from-images-with-wan-2-2-i2v-a14b-ai-model-6966


# Generate Videos from Images with Wan 2.2 I2V A14B AI Model

### 1. Workflow Overview

This workflow, titled **"Wan Video Wan 2.2 I2V A14B AI Model"**, automates the generation of videos from images using the **wan-video/wan-2.2-i2v-a14b** AI model hosted on the Replicate platform. It is designed to take a textual prompt and an image URL as inputs, submit these to the AI model for video generation, then poll the prediction status until completion and finally extract and process the generated video URL.

The workflow is structured into the following logical blocks:

- **1.1 Manual Trigger and API Key Setup**: Receives manual execution input and securely sets the Replicate API key.
- **1.2 Video Generation Request**: Submits the video generation request to the Replicate API.
- **1.3 Prediction ID Extraction and Polling**: Extracts the prediction ID from the initial API response and initiates a polling loop to check prediction status.
- **1.4 Completion Check and Result Processing**: Determines if the video generation is complete and processes the resulting video URL.

---

### 2. Block-by-Block Analysis

#### 1.1 Manual Trigger and API Key Setup

**Overview:**  
This block initiates the workflow manually and sets the Replicate API key as a workflow variable for authentication in subsequent API calls.

**Nodes Involved:**  
- On clicking 'execute'  
- Set API Key

**Node Details:**

- **On clicking 'execute'**  
  - Type: Manual Trigger  
  - Role: Starts workflow execution manually.  
  - Configuration: No parameters; simply triggers workflow on user command.  
  - Inputs: None  
  - Outputs: Passes execution to Set API Key node.  
  - Edge Cases: User forgetting to trigger manually; no automatic scheduling.

- **Set API Key**  
  - Type: Set  
  - Role: Stores the Replicate API key in the workflow context under `replicate_api_key`.  
  - Configuration: Assigns a string value `"YOUR_REPLICATE_API_KEY"` which must be replaced with a valid key.  
  - Inputs: From manual trigger node.  
  - Outputs: Passes to Create Prediction node.  
  - Edge Cases: Missing or invalid API key will cause authentication failures in API requests.

---

#### 1.2 Video Generation Request

**Overview:**  
Sends a POST request to the Replicate API to create a new video generation prediction based on the provided prompt and image URL.

**Nodes Involved:**  
- Create Prediction

**Node Details:**

- **Create Prediction**  
  - Type: HTTP Request  
  - Role: Submits video generation job to Replicate API.  
  - Configuration:  
    - URL: `https://api.replicate.com/v1/predictions`  
    - Method: POST  
    - Body: JSON containing model version, prompt, image URL, seed, number of frames, sampling parameters, and FPS.  
    - Headers: Authorization Bearer token from `replicate_api_key`, Content-Type application/json.  
    - Timeout: 60 seconds.  
  - Key Expressions: Uses expression to insert API key from `Set API Key` node.  
  - Inputs: From Set API Key node.  
  - Outputs: JSON response with prediction ID and status.  
  - Edge Cases: HTTP errors, timeout, invalid input parameters, authentication errors.

---

#### 1.3 Prediction ID Extraction and Polling

**Overview:**  
Extracts the prediction ID from the initial creation response and implements a polling mechanism with a delay to check the status of the video generation until it completes.

**Nodes Involved:**  
- Extract Prediction ID  
- Wait  
- Check Prediction Status  
- Check If Complete

**Node Details:**

- **Extract Prediction ID**  
  - Type: Code  
  - Role: Parses the API response to extract the prediction ID and initial status, constructs the URL for status polling.  
  - Configuration: JavaScript code runs once per item; returns predictionId, status, and predictionUrl.  
  - Inputs: From Create Prediction node.  
  - Outputs: Data with prediction ID and polling URL.  
  - Edge Cases: Missing or malformed response data can cause reference errors.

- **Wait**  
  - Type: Wait  
  - Role: Introduces a 2-second delay before polling the prediction status again.  
  - Configuration: Wait for 2 seconds.  
  - Inputs: From Extract Prediction ID and from Check If Complete if prediction not done.  
  - Outputs: To Check Prediction Status node.  
  - Edge Cases: Network delays longer than wait might cause inefficient polling.

- **Check Prediction Status**  
  - Type: HTTP Request  
  - Role: Polls the Replicate API to get the current status of the prediction.  
  - Configuration:  
    - URL: Dynamically set to `predictionUrl` extracted earlier.  
    - Method: GET (default).  
    - Headers: Authorization Bearer token.  
  - Inputs: From Wait node.  
  - Outputs: JSON with current prediction status.  
  - Edge Cases: API downtime, invalid prediction ID, authorization errors.

- **Check If Complete**  
  - Type: If  
  - Role: Checks if the prediction status equals "succeeded".  
  - Configuration: Boolean condition comparing `$json.status == "succeeded"`.  
  - Inputs: From Check Prediction Status.  
  - Outputs:  
    - True: Proceed to Process Result node.  
    - False: Loop back to Wait node for another poll.  
  - Edge Cases: Other statuses like "failed", "canceled" are not explicitly handled here.

---

#### 1.4 Completion Check and Result Processing

**Overview:**  
Once the prediction is complete, this block processes the result, extracting relevant metadata and the generated video URL for downstream use.

**Nodes Involved:**  
- Process Result

**Node Details:**

- **Process Result**  
  - Type: Code  
  - Role: Extracts and formats the completed prediction data including status, output URL, metrics, timestamps, and model identifier.  
  - Configuration: JavaScript code returning a structured JSON containing the video URL and metadata.  
  - Inputs: From Check If Complete node (true branch).  
  - Outputs: Final processed data ready for consumption or export.  
  - Edge Cases: Unexpected null or missing output fields.

---

### 3. Summary Table

| Node Name            | Node Type          | Functional Role                         | Input Node(s)            | Output Node(s)           | Sticky Note                                                                                                          |
|----------------------|--------------------|---------------------------------------|--------------------------|--------------------------|----------------------------------------------------------------------------------------------------------------------|
| On clicking 'execute' | Manual Trigger     | Workflow manual start point            | None                     | Set API Key              |                                                                                                                      |
| Set API Key          | Set                | Stores Replicate API key               | On clicking 'execute'    | Create Prediction        |                                                                                                                      |
| Create Prediction    | HTTP Request       | Sends video generation request to API | Set API Key              | Extract Prediction ID    |                                                                                                                      |
| Extract Prediction ID| Code               | Extracts prediction ID and status      | Create Prediction        | Wait                     |                                                                                                                      |
| Wait                 | Wait               | Delay between polling attempts         | Extract Prediction ID, Check If Complete (false) | Check Prediction Status |                                                                                                                      |
| Check Prediction Status| HTTP Request     | Polls prediction status from API       | Wait                     | Check If Complete        |                                                                                                                      |
| Check If Complete    | If                 | Checks if prediction is done            | Check Prediction Status  | Process Result (true), Wait (false) |                                                                                                                      |
| Process Result       | Code               | Processes and formats final output     | Check If Complete (true) | None                     |                                                                                                                      |
| Sticky Note          | Sticky Note        | Informational note on workflow          | None                     | None                     | ## Wan Video Wan 2.2 I2v A14b Video Generator\n\nThis workflow uses the **wan-video/wan-2.2-i2v-a14b** model from Replicate to generate video content.\n\n### Setup\n1. Add your Replicate API key\n2. Configure the input parameters\n3. Run the workflow\n\n### Model Details\n- **Type**: Video Generation\n- **Provider**: wan-video\n- **Required Fields**: prompt, image |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: `On clicking 'execute'`  
   - Type: Manual Trigger  
   - No additional parameters.

2. **Create Set Node to Store API Key**  
   - Name: `Set API Key`  
   - Type: Set  
   - Add assignment:  
     - Variable name: `replicate_api_key`  
     - Type: String  
     - Value: `"YOUR_REPLICATE_API_KEY"` (Replace with your valid Replicate API key.)  
   - Connect output of `On clicking 'execute'` to this node.

3. **Create HTTP Request Node to Create Prediction**  
   - Name: `Create Prediction`  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.replicate.com/v1/predictions`  
   - Body Content Type: JSON  
   - Body Parameters:  
     ```json
     {
       "version": "9c49fe41d6b2a0e62199dc96bee4a9dd3565a4c563f9b80998358f14322c34f6",
       "input": {
         "prompt": "prompt value",
         "image": "https://example.com/input.video",
         "seed": 1,
         "num_frames": 81,
         "sample_shift": 5,
         "sample_steps": 30,
         "frames_per_second": 16
       }
     }
     ```  
     Replace `"prompt value"` and `"https://example.com/input.video"` with your actual prompt and image URL.  
   - Headers:  
     - Authorization: `Bearer {{$node["Set API Key"].json["replicate_api_key"]}}` (use expression)  
     - Content-Type: `application/json`  
   - Timeout: 60000 ms (60 seconds)  
   - Connect output of `Set API Key` to this node.

4. **Create Code Node to Extract Prediction ID**  
   - Name: `Extract Prediction ID`  
   - Type: Code  
   - Language: JavaScript  
   - Code:  
     ```js
     const prediction = $input.item.json;
     const predictionId = prediction.id;
     const initialStatus = prediction.status;
     return {
       predictionId,
       status: initialStatus,
       predictionUrl: `https://api.replicate.com/v1/predictions/${predictionId}`
     };
     ```  
   - Connect output of `Create Prediction` to this node.

5. **Create Wait Node for Delay**  
   - Name: `Wait`  
   - Type: Wait  
   - Wait Duration: 2 seconds  
   - Connect output of `Extract Prediction ID` to this node.

6. **Create HTTP Request Node to Check Prediction Status**  
   - Name: `Check Prediction Status`  
   - Type: HTTP Request  
   - Method: GET (default)  
   - URL (Expression): `{{$json["predictionUrl"]}}`  
   - Headers:  
     - Authorization: `Bearer {{$node["Set API Key"].json["replicate_api_key"]}}`  
   - Connect output of `Wait` node to this node.

7. **Create If Node to Check Completion**  
   - Name: `Check If Complete`  
   - Type: If  
   - Condition: Boolean  
     - Expression: `{{$json["status"] === "succeeded"}}`  
   - Connect output of `Check Prediction Status` to this node.

8. **Create Code Node to Process Result**  
   - Name: `Process Result`  
   - Type: Code  
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
       model: 'wan-video/wan-2.2-i2v-a14b',
       video_url: result.output
     };
     ```  
   - Connect the "true" output of `Check If Complete` to this node.

9. **Loop Back for Polling**  
   - Connect the "false" output of `Check If Complete` back to the `Wait` node to continue polling until completion.

10. **Add Sticky Note for Documentation (Optional)**  
    - Content:  
      ```
      ## Wan Video Wan 2.2 I2v A14b Video Generator

      This workflow uses the **wan-video/wan-2.2-i2v-a14b** model from Replicate to generate video content.

      ### Setup
      1. Add your Replicate API key
      2. Configure the input parameters
      3. Run the workflow

      ### Model Details
      - Type: Video Generation
      - Provider: wan-video
      - Required Fields: prompt, image
      ```

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                            |
|---------------------------------------------------------------------------------------------------------------|------------------------------------------------------------|
| Please replace `"YOUR_REPLICATE_API_KEY"` in the Set API Key node with a valid API key from Replicate.         | API Key Setup                                              |
| The video generation model requires a valid prompt and image URL to function correctly.                       | Model Input Requirements                                   |
| Polling interval is set to 2 seconds to balance API usage and responsiveness; adjust if needed.                | Polling Strategy                                           |
| The Replicate API version used is identified by the version string `9c49fe41d6b2a0e62199dc96bee4a9dd3565a4c563f9b80998358f14322c34f6`. | Model Version Reference                                   |
| If the status returned is other than "succeeded" (e.g., "failed"), the workflow currently loops indefinitely; consider adding failure/error handling. | Potential Improvements                                     |
| Replicate API documentation: https://replicate.com/docs/reference/http | Official API docs for troubleshooting and advanced usage |

---

**Disclaimer:**  
The text provided is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected material. All data handled are legal and public.