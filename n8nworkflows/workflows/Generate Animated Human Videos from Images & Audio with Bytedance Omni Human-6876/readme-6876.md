Generate Animated Human Videos from Images & Audio with Bytedance Omni Human

https://n8nworkflows.xyz/workflows/generate-animated-human-videos-from-images---audio-with-bytedance-omni-human-6876


# Generate Animated Human Videos from Images & Audio with Bytedance Omni Human

### 1. Workflow Overview

The **Bytedance Omni Human Video Generator** workflow automates the process of generating animated human videos from images and audio files using the **bytedance/omni-human** model on the Replicate platform. It is designed for users who want to create video content by providing an image and an audio source, leveraging AI-driven video generation.

The workflow is logically divided into the following blocks:

- **1.1 Manual Trigger & API Key Setup:** Initiates the workflow manually and sets up the required authentication key for Replicate API.
- **1.2 Video Generation Request:** Sends a video generation request to the Replicate API with the specified image and audio URLs.
- **1.3 Prediction Tracking:** Extracts the prediction ID from the response and polls the API until the video generation completes.
- **1.4 Result Processing:** Processes and formats the completed prediction output for downstream use or storage.

---

### 2. Block-by-Block Analysis

#### 2.1 Manual Trigger & API Key Setup

- **Overview:**  
  This block allows the user to start the workflow manually and sets the Replicate API key needed for authenticated requests.

- **Nodes Involved:**  
  - On clicking 'execute'  
  - Set API Key

- **Node Details:**

  - **On clicking 'execute'**  
    - Type: Manual Trigger  
    - Role: Starts the workflow manually upon user action.  
    - Configuration: No parameters; activated by user clicking "execute".  
    - Inputs: None (entry point)  
    - Outputs: Triggers next node "Set API Key"  
    - Potential Failures: None inherent; user must trigger manually.

  - **Set API Key**  
    - Type: Set  
    - Role: Stores the Replicate API key as a workflow variable.  
    - Configuration: Assigns the string value `YOUR_REPLICATE_API_KEY` to variable `replicate_api_key`. This should be replaced with a valid API key before running.  
    - Inputs: Receives manual trigger signal  
    - Outputs: Passes key to "Create Prediction" node  
    - Edge Cases: Forgetting to replace placeholder key results in authentication failure downstream.

---

#### 2.2 Video Generation Request

- **Overview:**  
  This block sends the video generation request to Replicate API, providing the input image and audio URLs, and receives an initial prediction response.

- **Nodes Involved:**  
  - Create Prediction  
  - Extract Prediction ID

- **Node Details:**

  - **Create Prediction**  
    - Type: HTTP Request  
    - Role: Sends a POST request to Replicate API to start video generation.  
    - Configuration:  
      - URL: `https://api.replicate.com/v1/predictions`  
      - Method: POST  
      - Headers:  
        - Authorization: Bearer token using `replicate_api_key` from "Set API Key" node  
        - Content-Type: application/json  
      - Body (JSON):  
        ```json
        {
          "version": "7ec44f5140c7338b3496cbf99ee8ea391a4bc18ff5d1677a146dfc936a91f65b",
          "input": {
            "image": "https://example.com/input.video",
            "audio": "https://example.com/input.video"
          }
        }
        ```  
      - Timeout: 60 seconds  
    - Inputs: Receives API key from "Set API Key"  
    - Outputs: JSON response containing prediction metadata  
    - Edge Cases:  
      - Invalid API key -> 401 Unauthorized error  
      - Invalid input URLs or missing required fields -> API returns error  
      - Network timeout or API downtime -> request failure

  - **Extract Prediction ID**  
    - Type: Code (JavaScript)  
    - Role: Parses the initial prediction response to extract ID and status for polling.  
    - Configuration:  
      - Runs once per incoming item  
      - Extracts `id` and `status` from JSON response  
      - Constructs a polling URL `https://api.replicate.com/v1/predictions/{id}`  
    - Inputs: Receives JSON response from "Create Prediction"  
    - Outputs: Passes the prediction ID, status, and polling URL to next node  
    - Edge Cases:  
      - Missing or malformed response JSON leads to code failure  
      - Missing `id` or `status` fields breaks polling logic

---

#### 2.3 Prediction Tracking

- **Overview:**  
  This block implements polling to check prediction status until the video generation is complete.

- **Nodes Involved:**  
  - Wait  
  - Check Prediction Status  
  - Check If Complete

- **Node Details:**

  - **Wait**  
    - Type: Wait  
    - Role: Pauses workflow execution for 2 seconds between status checks to avoid excessive API calls.  
    - Configuration: Wait time = 2 seconds  
    - Inputs: Receives polling URL and status  
    - Outputs: Triggers "Check Prediction Status" node  
    - Edge Cases: None significant; long delays may slow workflow.

  - **Check Prediction Status**  
    - Type: HTTP Request  
    - Role: Sends GET request to Replicate API to retrieve current status of prediction.  
    - Configuration:  
      - URL: Dynamic, from `predictionUrl` variable  
      - Headers: Authorization Bearer with API key  
      - Method: GET (default)  
    - Inputs: Receives polling URL and API key  
    - Outputs: JSON with updated prediction status  
    - Edge Cases:  
      - API rate limits or downtime may cause errors  
      - Authentication errors if API key invalidated mid-run

  - **Check If Complete**  
    - Type: If  
    - Role: Checks if prediction status equals "succeeded".  
    - Configuration: Condition compares `status` field from JSON to string `"succeeded"`.  
    - Inputs: Receives status from "Check Prediction Status"  
    - Outputs:  
      - **True:** Passes to "Process Result" node  
      - **False:** Loops back to "Wait" node for another polling cycle  
    - Edge Cases:  
      - Status other than 'succeeded' or 'failed' requires handling (no 'failed' condition here)  
      - Potential infinite loop if prediction never completes or errors occur unnoticed.

---

#### 2.4 Result Processing

- **Overview:**  
  After successful prediction, this block extracts and formats the relevant output data for further use.

- **Nodes Involved:**  
  - Process Result

- **Node Details:**

  - **Process Result**  
    - Type: Code (JavaScript)  
    - Role: Extracts key fields from the prediction result and formats output including video URL.  
    - Configuration:  
      - Runs once per item  
      - Extracts fields: `status`, `output`, `metrics`, `created_at`, `completed_at`  
      - Adds fixed `model` name "bytedance/omni-human"  
      - Sets `video_url` to the prediction output URL  
    - Inputs: Receives JSON from "Check If Complete" (on success path)  
    - Outputs: Structured object with relevant metadata and video URL  
    - Edge Cases:  
      - Missing output URL or unexpected data format may cause failure  
      - No downstream nodes defined to handle output beyond this step (user to extend)

---

### 3. Summary Table

| Node Name               | Node Type           | Functional Role                                 | Input Node(s)           | Output Node(s)            | Sticky Note                                                                                                                  |
|-------------------------|---------------------|------------------------------------------------|-------------------------|---------------------------|------------------------------------------------------------------------------------------------------------------------------|
| On clicking 'execute'    | Manual Trigger      | Entry point to start workflow manually          | None                    | Set API Key               | ## Bytedance Omni Human Video Generator<br><br>This workflow uses the **bytedance/omni-human** model from Replicate to generate video content.<br><br>### Setup<br>1. Add your Replicate API key<br>2. Configure the input parameters<br>3. Run the workflow<br><br>### Model Details<br>- **Type**: Video Generation<br>- **Provider**: bytedance<br>- **Required Fields**: image, audio |
| Set API Key             | Set                 | Stores Replicate API key for authentication     | On clicking 'execute'   | Create Prediction         | (See above)                                                                                                                  |
| Create Prediction       | HTTP Request        | Sends video generation request to Replicate API | Set API Key             | Extract Prediction ID     | (See above)                                                                                                                  |
| Extract Prediction ID   | Code (JavaScript)   | Extracts prediction ID and prepares polling URL | Create Prediction       | Wait                      | (See above)                                                                                                                  |
| Wait                    | Wait                | Pauses execution for 2 seconds between polls    | Extract Prediction ID, Check If Complete (false path) | Check Prediction Status  | (See above)                                                                                                                  |
| Check Prediction Status | HTTP Request        | Polls Replicate API for prediction status       | Wait                    | Check If Complete         | (See above)                                                                                                                  |
| Check If Complete       | If                  | Checks if prediction succeeded or not            | Check Prediction Status | Process Result (true), Wait (false) | (See above)                                                                                                                  |
| Process Result          | Code (JavaScript)   | Processes final prediction result and extracts output | Check If Complete (true) | None                      | (See above)                                                                                                                  |
| Sticky Note             | Sticky Note         | Provides setup and model information             | None                    | None                      | ## Bytedance Omni Human Video Generator<br><br>This workflow uses the **bytedance/omni-human** model from Replicate to generate video content.<br><br>### Setup<br>1. Add your Replicate API key<br>2. Configure the input parameters<br>3. Run the workflow<br><br>### Model Details<br>- **Type**: Video Generation<br>- **Provider**: bytedance<br>- **Required Fields**: image, audio |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a "Manual Trigger" node named `On clicking 'execute'`. This will start the workflow manually.

2. **Add Set Node for API Key**  
   - Add a "Set" node named `Set API Key`.  
   - Add an assignment:  
     - Variable name: `replicate_api_key`  
     - Type: String  
     - Value: `YOUR_REPLICATE_API_KEY` (replace with your actual Replicate API key)  
   - Connect `On clicking 'execute'` output to this node.

3. **Add HTTP Request Node to Create Prediction**  
   - Add an "HTTP Request" node named `Create Prediction`.  
   - Set method to `POST`.  
   - Set URL to `https://api.replicate.com/v1/predictions`.  
   - Under Authentication, select "Generic Credential" with HTTP Header Auth.  
   - Add header parameters:  
     - `Authorization`: Expression `Bearer {{$node["Set API Key"].json["replicate_api_key"]}}`  
     - `Content-Type`: `application/json`  
   - Set request body type to JSON.  
   - Provide JSON body:  
     ```json
     {
       "version": "7ec44f5140c7338b3496cbf99ee8ea391a4bc18ff5d1677a146dfc936a91f65b",
       "input": {
         "image": "https://example.com/input.video",
         "audio": "https://example.com/input.video"
       }
     }
     ```  
   - Set timeout to 60 seconds.  
   - Connect output of `Set API Key` to this node.

4. **Add Code Node to Extract Prediction ID**  
   - Add a "Code" node named `Extract Prediction ID`.  
   - Set mode to "Run Once For Each Item".  
   - Use the following JavaScript code:  
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
   - Connect output of `Create Prediction` to this node.

5. **Add Wait Node**  
   - Add a "Wait" node named `Wait`.  
   - Configure to wait 2 seconds.  
   - Connect output of `Extract Prediction ID` to this node.

6. **Add HTTP Request Node to Check Prediction Status**  
   - Add an "HTTP Request" node named `Check Prediction Status`.  
   - Set method to `GET` (default).  
   - Set URL to expression `{{$json["predictionUrl"]}}`.  
   - Under Authentication, use same generic credential type with HTTP Header Auth.  
   - Add header parameter:  
     - `Authorization`: Expression `Bearer {{$node["Set API Key"].json["replicate_api_key"]}}`  
   - Connect output of `Wait` node to this node.

7. **Add If Node to Check Completion**  
   - Add an "If" node named `Check If Complete`.  
   - Set condition type to Boolean.  
   - Condition: `{{$json["status"]}}` equals `succeeded`.  
   - Connect output of `Check Prediction Status` to this node.

8. **Add Code Node to Process Result**  
   - Add a "Code" node named `Process Result`.  
   - Set mode to "Run Once For Each Item".  
   - JavaScript code:  
     ```javascript
     const result = $input.item.json;

     return {
       status: result.status,
       output: result.output,
       metrics: result.metrics,
       created_at: result.created_at,
       completed_at: result.completed_at,
       model: 'bytedance/omni-human',
       video_url: result.output
     };
     ```  
   - Connect the "true" output of `Check If Complete` to this node.

9. **Loop Back for Polling**  
   - Connect the "false" output of `Check If Complete` back to the `Wait` node to continue polling.

10. **Add Sticky Note for Documentation**  
    - Add a "Sticky Note" node with the following content:  
      ```
      ## Bytedance Omni Human Video Generator

      This workflow uses the **bytedance/omni-human** model from Replicate to generate video content.

      ### Setup
      1. Add your Replicate API key
      2. Configure the input parameters
      3. Run the workflow

      ### Model Details
      - **Type**: Video Generation
      - **Provider**: bytedance
      - **Required Fields**: image, audio
      ```  
    - Place it visually near the start nodes for easy reference.

---

### 5. General Notes & Resources

| Note Content                                                                                             | Context or Link                                                                                     |
|--------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| The workflow requires a valid Replicate API key with permissions to use the "bytedance/omni-human" model. | [Replicate API Documentation](https://replicate.com/docs)                                         |
| Input URLs for image and audio must be publicly accessible and valid URLs pointing to the respective media. | If inputs are inaccessible, the model will return errors or fail to process.                       |
| The polling mechanism uses a 2-second interval to avoid excessive API requests while waiting for completion. | Adjust the wait time if API rate limits are strict or workflow needs to be faster/slower.          |
| No explicit error handling for failed or canceled predictions is implemented; consider adding logic for those states. | Enhancements could include handling statuses like "failed" or "canceled" and retry mechanisms.    |
| The output video URL is provided in the `video_url` field at the end; further steps can save or process this link. | Workflow can be extended to download, email, or upload the video post-processing.                  |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.