Generate Text-to-Video Content with Google Veo 3 via Replicate API

https://n8nworkflows.xyz/workflows/generate-text-to-video-content-with-google-veo-3-via-replicate-api-7100


# Generate Text-to-Video Content with Google Veo 3 via Replicate API

### 1. Workflow Overview

This workflow automates the generation of video content using the Google Veo 3 model hosted on Replicate via their API. It is designed to take a text prompt input and produce a corresponding video by interacting with Replicate’s prediction endpoints. The workflow continuously polls the prediction status until completion and then processes the final video output URL.

The logical blocks of the workflow are:

- **1.1 Input Trigger and API Key Setup:** Starts the workflow manually and sets the Replicate API key.
- **1.2 Prediction Creation:** Submits a video generation request to the Replicate API with the specified prompt.
- **1.3 Prediction Monitoring:** Polls the API repeatedly to check prediction status until the video generation is completed.
- **1.4 Result Processing:** Extracts and formats the final video output data after successful completion.
- **1.5 Documentation:** Includes a sticky note summarizing the workflow purpose, setup instructions, and model details.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Trigger and API Key Setup

**Overview:**  
This block initiates the workflow manually and assigns the user’s Replicate API key to be used in subsequent API calls.

**Nodes Involved:**  
- On clicking 'execute'  
- Set API Key

**Node Details:**

- **On clicking 'execute'**  
  - Type: Manual Trigger  
  - Role: Entry point to start the workflow manually in the n8n editor.  
  - Configuration: No parameters; triggers workflow on user execution.  
  - Inputs: None  
  - Outputs: Connected to "Set API Key" node  
  - Edge Cases: None, but workflow will not run unless manually triggered.

- **Set API Key**  
  - Type: Set  
  - Role: Defines and stores the Replicate API key as a workflow variable.  
  - Configuration: Assigns string value `YOUR_REPLICATE_API_KEY` to variable `replicate_api_key`. This must be replaced with a valid API key before running.  
  - Inputs: From manual trigger node  
  - Outputs: Connected to "Create Prediction" node  
  - Edge Cases: Missing or invalid API key will cause authentication failures downstream.

---

#### 1.2 Prediction Creation

**Overview:**  
Sends a POST request to Replicate’s prediction endpoint to start video generation using the Google Veo 3 model, passing the user’s prompt and additional parameters.

**Nodes Involved:**  
- Create Prediction  
- Extract Prediction ID

**Node Details:**

- **Create Prediction**  
  - Type: HTTP Request  
  - Role: Sends a POST request to `https://api.replicate.com/v1/predictions` to initiate a new prediction.  
  - Configuration:  
    - Method: POST  
    - Body (JSON):  
      ```json
      {
        "version": "6698506de7dd6e005c4734d511a81efcd8e8f9433c619416bbcdc681712d4bf2",
        "input": {
          "prompt": "prompt value",
          "seed": 1
        }
      }
      ```  
      Note: The `"prompt"` string should be replaced or dynamically set for actual content.  
    - Headers:  
      - Authorization: Bearer token using the `replicate_api_key` variable from "Set API Key" node.  
      - Content-Type: application/json  
    - Timeout: 60 seconds  
  - Inputs: From "Set API Key"  
  - Outputs: Connected to "Extract Prediction ID" node  
  - Edge Cases:  
    - Invalid API key leads to 401 Unauthorized errors.  
    - Network timeouts or 5xx server errors from Replicate API.  
    - Improper prompt format may cause prediction failure.

- **Extract Prediction ID**  
  - Type: Code (JavaScript)  
  - Role: Extracts the prediction ID and initial status from the response to store them for later polling.  
  - Configuration: Runs once per item; reads `id` and `status` from the prediction JSON response.  
  - Key Expressions:  
    - `predictionId = prediction.id`  
    - `initialStatus = prediction.status`  
    - Forms `predictionUrl` for polling using the ID.  
  - Inputs: From "Create Prediction" node  
  - Outputs: Connected to "Wait" node  
  - Edge Cases:  
    - Missing or malformed response JSON could cause extraction errors.  
    - If prediction ID is not returned, polling cannot proceed.

---

#### 1.3 Prediction Monitoring

**Overview:**  
Polls the Replicate API prediction endpoint periodically (every 2 seconds) to check if the video generation is complete.

**Nodes Involved:**  
- Wait  
- Check Prediction Status  
- Check If Complete

**Node Details:**

- **Wait**  
  - Type: Wait  
  - Role: Pauses workflow for 2 seconds between polling attempts to avoid excessive requests.  
  - Configuration: Wait time set to 2 seconds.  
  - Inputs: From "Extract Prediction ID" or "Check If Complete" (if not complete)  
  - Outputs: Connected to "Check Prediction Status" node  
  - Edge Cases: None directly, but long-running workflows may be resource-intensive.

- **Check Prediction Status**  
  - Type: HTTP Request  
  - Role: Sends a GET request to the prediction URL to retrieve the current status and output.  
  - Configuration:  
    - Method: GET (default)  
    - URL: Dynamically set from `predictionUrl` extracted previously.  
    - Header: Authorization Bearer token with `replicate_api_key`.  
  - Inputs: From "Wait" node  
  - Outputs: Connected to "Check If Complete" node  
  - Edge Cases:  
    - API rate limits may block frequent polling.  
    - Network errors or invalid URLs may cause failures.

- **Check If Complete**  
  - Type: If  
  - Role: Conditional node that checks if the prediction status equals "succeeded".  
  - Configuration: Checks if `$json.status === 'succeeded'`.  
  - Inputs: From "Check Prediction Status"  
  - Outputs:  
    - True branch: Connected to "Process Result" node  
    - False branch: Connected back to "Wait" node (continue polling)  
  - Edge Cases:  
    - If the status is "failed" or other unexpected values, workflow may loop indefinitely unless handled externally.

---

#### 1.4 Result Processing

**Overview:**  
Processes the completed prediction to extract and format relevant output data, such as video URL and metadata.

**Nodes Involved:**  
- Process Result

**Node Details:**

- **Process Result**  
  - Type: Code (JavaScript)  
  - Role: Extracts key fields from the prediction result and prepares an output object with video URL and metadata.  
  - Configuration:  
    - Reads fields: `status`, `output` (expected video URL), `metrics`, `created_at`, `completed_at`.  
    - Adds static field `model` with value `"google/veo-3"`.  
    - Returns an object including `video_url` set to the prediction’s output URL.  
  - Inputs: From "Check If Complete" (true branch)  
  - Outputs: Terminal node (no further connections)  
  - Edge Cases:  
    - If output is missing or malformed, video URL may be invalid.  
    - No error handling for failed or incomplete statuses here.

---

#### 1.5 Documentation

**Overview:**  
Provides an inline sticky note within the workflow editor summarizing purpose, setup, and model details.

**Nodes Involved:**  
- Sticky Note

**Node Details:**

- **Sticky Note**  
  - Type: Sticky Note  
  - Role: Documentation inside the workflow canvas for user reference.  
  - Content Summary:  
    - Describes the workflow as a Google Veo 3 Video Generator using Replicate API.  
    - Setup instructions including placing API key and configuring prompt.  
    - Model details: Type (Video Generation), Provider (google), Required Fields (prompt).  
  - Inputs/Outputs: None  
  - Edge Cases: Not applicable.

---

### 3. Summary Table

| Node Name                | Node Type           | Functional Role                      | Input Node(s)           | Output Node(s)             | Sticky Note                                                                                               |
|--------------------------|---------------------|------------------------------------|------------------------|----------------------------|----------------------------------------------------------------------------------------------------------|
| On clicking 'execute'    | Manual Trigger      | Manual workflow start               | None                   | Set API Key                |                                                                                                          |
| Set API Key              | Set                 | Stores Replicate API key            | On clicking 'execute'  | Create Prediction           |                                                                                                          |
| Create Prediction        | HTTP Request        | Sends video generation request      | Set API Key            | Extract Prediction ID       |                                                                                                          |
| Extract Prediction ID    | Code (JavaScript)   | Extracts prediction ID and URL      | Create Prediction      | Wait                       |                                                                                                          |
| Wait                     | Wait                | Delay between polling cycles        | Extract Prediction ID, Check If Complete (false branch) | Check Prediction Status |                                                                                                          |
| Check Prediction Status  | HTTP Request        | Polls prediction status             | Wait                   | Check If Complete           |                                                                                                          |
| Check If Complete        | If                  | Checks if prediction succeeded      | Check Prediction Status | Process Result (true), Wait (false) |                                                                                                          |
| Process Result           | Code (JavaScript)   | Final result extraction and formatting | Check If Complete (true) | None                      |                                                                                                          |
| Sticky Note              | Sticky Note         | Documentation                      | None                   | None                       | ## Google Veo 3 Video Generator\n\nThis workflow uses the **google/veo-3** model from Replicate to generate video content.\n\n### Setup\n1. Add your Replicate API key\n2. Configure the input parameters\n3. Run the workflow\n\n### Model Details\n- **Type**: Video Generation\n- **Provider**: google\n- **Required Fields**: prompt |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a node of type **Manual Trigger** named "On clicking 'execute'".  
   - No parameters needed.

2. **Add Set Node for API Key**  
   - Add a **Set** node named "Set API Key".  
   - Add a string field named `replicate_api_key` with value `"YOUR_REPLICATE_API_KEY"`. Replace this with your actual Replicate API key before running.  
   - Connect "On clicking 'execute'" output to this node.

3. **Add HTTP Request Node to Create Prediction**  
   - Add an **HTTP Request** node named "Create Prediction".  
   - Set:  
     - Method: POST  
     - URL: `https://api.replicate.com/v1/predictions`  
     - Authentication: Generic HTTP Header Auth  
     - Headers:  
       - `Authorization`: Expression: `Bearer {{$node["Set API Key"].json["replicate_api_key"]}}`  
       - `Content-Type`: `application/json`  
     - Body Content Type: JSON  
     - Body:  
       ```json
       {
         "version": "6698506de7dd6e005c4734d511a81efcd8e8f9433c619416bbcdc681712d4bf2",
         "input": {
           "prompt": "prompt value",
           "seed": 1
         }
       }
       ```  
       Replace `"prompt value"` with your desired text prompt.  
     - Timeout: 60000 ms (60 seconds)  
   - Connect "Set API Key" to this node.

4. **Add Code Node to Extract Prediction ID**  
   - Add a **Code** node named "Extract Prediction ID".  
   - Set mode to run once per item.  
   - Use this JavaScript code:  
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
   - Connect "Create Prediction" node output to this node.

5. **Add Wait Node**  
   - Add a **Wait** node named "Wait".  
   - Set to wait for 2 seconds.  
   - Connect "Extract Prediction ID" output to this node.

6. **Add HTTP Request Node to Check Prediction Status**  
   - Add an **HTTP Request** node named "Check Prediction Status".  
   - Set:  
     - Method: GET (default)  
     - URL: Expression: `{{$json["predictionUrl"]}}`  
     - Authentication: Generic HTTP Header Auth  
     - Headers:  
       - `Authorization`: Expression: `Bearer {{$node["Set API Key"].json["replicate_api_key"]}}`  
   - Connect "Wait" node output to this node.

7. **Add If Node to Check Completion**  
   - Add an **If** node named "Check If Complete".  
   - Condition: Boolean  
     - Value 1: Expression: `{{$json["status"]}}`  
     - Operation: Equal  
     - Value 2: `succeeded`  
   - Connect "Check Prediction Status" output to this node.

8. **Add Code Node to Process Result**  
   - Add a **Code** node named "Process Result".  
   - Mode: Run once per item.  
   - JavaScript code:  
     ```javascript
     const result = $input.item.json;

     return {
       status: result.status,
       output: result.output,
       metrics: result.metrics,
       created_at: result.created_at,
       completed_at: result.completed_at,
       model: 'google/veo-3',
       video_url: result.output
     };
     ```  
   - Connect the "true" output of "Check If Complete" to this node.

9. **Connect False Branch Back to Wait Node**  
   - Connect the "false" output of "Check If Complete" back to the "Wait" node to enable polling loop.

10. **Add Sticky Note for Documentation** (Optional)  
    - Add a **Sticky Note** node named "Sticky Note".  
    - Content:  
      ```
      ## Google Veo 3 Video Generator

      This workflow uses the **google/veo-3** model from Replicate to generate video content.

      ### Setup
      1. Add your Replicate API key
      2. Configure the input parameters
      3. Run the workflow

      ### Model Details
      - **Type**: Video Generation
      - **Provider**: google
      - **Required Fields**: prompt
      ```
    - Place it visibly near the start nodes for reference.

---

### 5. General Notes & Resources

| Note Content                                                                                             | Context or Link                                 |
|--------------------------------------------------------------------------------------------------------|------------------------------------------------|
| The Replicate API requires a valid API key which can be obtained by creating an account on replicate.com. | Replicate API documentation: https://replicate.com/docs |
| The model version ID `"6698506de7dd6e005c4734d511a81efcd8e8f9433c619416bbcdc681712d4bf2"` corresponds to Google Veo 3 video generation model on Replicate. | Model page: https://replicate.com/google/veo-3 |
| The polling interval of 2 seconds balances between responsiveness and API rate limits; adjust as needed. | Best practice for API polling.                  |
| The workflow requires manual trigger; can be adapted to accept dynamic prompt inputs or be triggered via webhook for automation. | n8n webhook node usage for automation.          |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created with n8n, a workflow automation tool. The process strictly complies with content policies and contains no illegal, offensive, or protected elements. All handled data are legal and public.