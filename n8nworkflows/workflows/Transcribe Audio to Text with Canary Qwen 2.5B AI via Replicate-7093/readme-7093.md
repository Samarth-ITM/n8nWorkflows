Transcribe Audio to Text with Canary Qwen 2.5B AI via Replicate

https://n8nworkflows.xyz/workflows/transcribe-audio-to-text-with-canary-qwen-2-5b-ai-via-replicate-7093


# Transcribe Audio to Text with Canary Qwen 2.5B AI via Replicate

---

### 1. Workflow Overview

This workflow automates the transcription of audio content into text by leveraging the **zsxkib/canary-qwen-2.5b** AI model hosted on Replicate. It is designed for use cases requiring automated, AI-powered audio-to-text transcription with timestamp inclusion, such as podcast transcription, meeting minutes generation, or audio content indexing.

The workflow is logically divided into the following blocks:

- **1.1 Manual Trigger & API Key Setup:** Initiates the workflow manually and sets the Replicate API key.
- **1.2 Prediction Creation:** Sends a transcription request to the Replicate API with the target audio URL and required parameters.
- **1.3 Prediction Monitoring:** Extracts the prediction ID and polls the Replicate API until the transcription process completes.
- **1.4 Result Processing:** Once the transcription is complete, processes and formats the output for further use or export.

---

### 2. Block-by-Block Analysis

#### 1.1 Manual Trigger & API Key Setup

**Overview:**  
This block starts the workflow execution manually and configures the necessary API key to authenticate requests to the Replicate API.

**Nodes Involved:**  
- On clicking 'execute'  
- Set API Key  

**Node Details:**

- **On clicking 'execute'**  
  - Type: Manual Trigger  
  - Role: Serves as the entry point, allowing users to start the workflow on demand.  
  - Configuration: No parameters needed; a manual button trigger.  
  - Inputs: None  
  - Outputs: Connects to "Set API Key" node.  
  - Edge Cases: None typical; user must initiate manually.

- **Set API Key**  
  - Type: Set  
  - Role: Stores the Replicate API key in a workflow variable for reuse in HTTP requests.  
  - Configuration: Assigns a string variable `replicate_api_key` with placeholder `"YOUR_REPLICATE_API_KEY"` that users must replace with their actual key.  
  - Inputs: Receives manual trigger signal.  
  - Outputs: Passes data to "Create Prediction" node.  
  - Edge Cases: Failure if API key is missing or invalid; must ensure user updates the placeholder.

---

#### 1.2 Prediction Creation

**Overview:**  
This block constructs and sends the transcription request to Replicate’s API, initiating the AI prediction job.

**Nodes Involved:**  
- Create Prediction  

**Node Details:**

- **Create Prediction**  
  - Type: HTTP Request  
  - Role: Sends a POST request to Replicate’s `/v1/predictions` endpoint to start audio transcription.  
  - Configuration:  
    - Method: POST  
    - URL: `https://api.replicate.com/v1/predictions`  
    - Authentication: HTTP Header with Bearer token using API key from "Set API Key".  
    - Request Body (JSON):  
      ```json
      {
        "version": "afba731fc7a4082730943a246233b09c7fa3dfb2c24b07fe199c1408a7c8cb2f",
        "input": {
          "audio": "https://example.com/input.text",
          "include_timestamps": true
        }
      }
      ```  
    - Key Expressions: Authorization header set dynamically as `Bearer {{ replicate_api_key }}` from previous node.  
  - Inputs: From "Set API Key" node.  
  - Outputs: To "Extract Prediction ID" node.  
  - Edge Cases:  
    - Timeout if API response delayed (timeout set to 60s).  
    - Failure if audio URL is invalid or inaccessible.  
    - Authentication errors if API key is incorrect.

---

#### 1.3 Prediction Monitoring

**Overview:**  
Extracts the prediction ID from the initial response and polls the Replicate API at intervals until the prediction completes.

**Nodes Involved:**  
- Extract Prediction ID  
- Wait  
- Check Prediction Status  
- Check If Complete  

**Node Details:**

- **Extract Prediction ID**  
  - Type: Code  
  - Role: Parses the API response to extract the prediction ID, initial status, and constructs the polling URL.  
  - Configuration: JavaScript code returning predictionId, status, and predictionUrl for polling.  
  - Inputs: From "Create Prediction".  
  - Outputs: To "Wait".  
  - Edge Cases: If response format changes or prediction ID missing, code may fail.

- **Wait**  
  - Type: Wait  
  - Role: Delays workflow execution by 2 seconds before polling again.  
  - Configuration: Wait for 2 seconds.  
  - Inputs: From "Extract Prediction ID" and also from "Check If Complete" when prediction not done.  
  - Outputs: To "Check Prediction Status".  
  - Edge Cases: None, but too short or too long wait times impact efficiency.

- **Check Prediction Status**  
  - Type: HTTP Request  
  - Role: Queries the Replicate API to get the current status of the prediction job.  
  - Configuration:  
    - URL: dynamically set from `predictionUrl` extracted previously.  
    - Authentication: Bearer token header with API key.  
    - Method: GET (default).  
  - Inputs: From "Wait".  
  - Outputs: To "Check If Complete".  
  - Edge Cases: Network errors, API downtime, bad token issues.

- **Check If Complete**  
  - Type: If  
  - Role: Checks if the prediction status is `succeeded`. If yes, proceeds to process results; if no, loops back to "Wait".  
  - Configuration: Boolean condition comparing `$json.status` to `"succeeded"`.  
  - Inputs: From "Check Prediction Status".  
  - Outputs:  
    - True: To "Process Result"  
    - False: To "Wait" (continues polling)  
  - Edge Cases: Infinite loops if prediction stuck in non-terminal states; no timeout limit implemented.

---

#### 1.4 Result Processing

**Overview:**  
Processes the completed prediction output, extracting transcription text, timestamps, metadata, and formats them for downstream use.

**Nodes Involved:**  
- Process Result  

**Node Details:**

- **Process Result**  
  - Type: Code  
  - Role: Extracts key fields (status, output, metrics, timestamps, created/completed dates) from prediction result and labels the model name.  
  - Configuration: JavaScript code extracting these fields and returning a structured JSON object.  
  - Inputs: From "Check If Complete" (true branch).  
  - Outputs: Final output for further use or export.  
  - Edge Cases: If output is unexpectedly empty or malformed, downstream consumers must handle it.

---

#### Additional Node

- **Sticky Note**  
  - Role: Provides user documentation and instructions about the workflow, including setup steps, model details, and usage notes.  
  - Covers introductory information, no runtime function.

---

### 3. Summary Table

| Node Name              | Node Type          | Functional Role                          | Input Node(s)           | Output Node(s)            | Sticky Note                                                                                                          |
|------------------------|--------------------|----------------------------------------|------------------------|---------------------------|----------------------------------------------------------------------------------------------------------------------|
| On clicking 'execute'   | Manual Trigger     | Workflow entry point                    | -                      | Set API Key               | See sticky note for workflow overview and setup instructions                                                        |
| Set API Key            | Set                | Stores API key for authentication      | On clicking 'execute'   | Create Prediction         | See sticky note for workflow overview and setup instructions                                                        |
| Create Prediction      | HTTP Request       | Initiates transcription prediction     | Set API Key             | Extract Prediction ID     | See sticky note for workflow overview and setup instructions                                                        |
| Extract Prediction ID  | Code               | Extracts prediction ID and status      | Create Prediction       | Wait                      |                                                                                                                      |
| Wait                   | Wait               | Waits before polling                    | Extract Prediction ID, Check If Complete (false branch) | Check Prediction Status    |                                                                                                                      |
| Check Prediction Status| HTTP Request       | Polls prediction status                 | Wait                    | Check If Complete         |                                                                                                                      |
| Check If Complete      | If                 | Checks if transcription is complete    | Check Prediction Status | Process Result (true), Wait (false) |                                                                                                                      |
| Process Result         | Code               | Processes and formats transcription output | Check If Complete (true) | -                         |                                                                                                                      |
| Sticky Note            | Sticky Note        | Documentation and setup instructions   | -                      | -                         | ## Zsxkib Canary Qwen 2.5b Text Generator<br>Use the **zsxkib/canary-qwen-2.5b** model from Replicate.<br>Setup steps:<br>1. Add your Replicate API key<br>2. Configure input parameters<br>3. Run the workflow<br>Model type: Text Generation<br>Provider: zsxkib<br>Required field: audio |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: `On clicking 'execute'`  
   - Type: Manual Trigger  
   - No parameters needed.

2. **Create Set Node**  
   - Name: `Set API Key`  
   - Type: Set  
   - Add a string field:  
     - Name: `replicate_api_key`  
     - Value: `YOUR_REPLICATE_API_KEY` (replace with your actual API key)  
   - Connect output of `On clicking 'execute'` to input of this node.

3. **Create HTTP Request Node for Prediction**  
   - Name: `Create Prediction`  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.replicate.com/v1/predictions`  
   - Authentication: HTTP Header Authentication  
     - Header Name: `Authorization`  
     - Header Value: `Bearer {{$node["Set API Key"].json["replicate_api_key"]}}`  
   - Headers:  
     - `Content-Type`: `application/json`  
   - Body (JSON):  
     ```json
     {
       "version": "afba731fc7a4082730943a246233b09c7fa3dfb2c24b07fe199c1408a7c8cb2f",
       "input": {
         "audio": "https://example.com/input.text",
         "include_timestamps": true
       }
     }
     ```  
   - Connect output of `Set API Key` to this node.

4. **Create Code Node to Extract Prediction ID**  
   - Name: `Extract Prediction ID`  
   - Type: Code  
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
   - Connect output of `Create Prediction` to this node.

5. **Create Wait Node**  
   - Name: `Wait`  
   - Type: Wait  
   - Wait Time: 2 seconds  
   - Connect output of `Extract Prediction ID` to this node.

6. **Create HTTP Request Node to Check Prediction Status**  
   - Name: `Check Prediction Status`  
   - Type: HTTP Request  
   - Method: GET (default)  
   - URL: `={{ $json["predictionUrl"] }}` (dynamic URL from previous node)  
   - Authentication: HTTP Header Authentication  
     - Header Name: `Authorization`  
     - Header Value: `Bearer {{$node["Set API Key"].json["replicate_api_key"]}}`  
   - Connect output of `Wait` to this node.

7. **Create If Node to Check Completion**  
   - Name: `Check If Complete`  
   - Type: If  
   - Condition: Boolean  
     - Expression: `{{$json["status"]}}` equals `succeeded`  
   - Connect output of `Check Prediction Status` to this node.

8. **Create Code Node to Process Result**  
   - Name: `Process Result`  
   - Type: Code  
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
       model: 'zsxkib/canary-qwen-2.5b',
       text_url: result.output
     };
     ```
   - Connect the "true" output of `Check If Complete` to this node.

9. **Loop Back for Polling**  
   - Connect the "false" output of `Check If Complete` back to the `Wait` node to continue polling until completion.

10. **Add Sticky Note for Documentation (Optional)**  
    - Content: Include setup instructions, model info, and usage notes as described in the Sticky Note node.

---

### 5. General Notes & Resources

| Note Content                                                                                                                     | Context or Link                                                   |
|---------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------|
| This workflow uses the **zsxkib/canary-qwen-2.5b** model accessed via Replicate for audio-to-text transcription with timestamps. | Model info on Replicate: https://replicate.com/zsxkib/canary-qwen-2.5b |
| Replace `"YOUR_REPLICATE_API_KEY"` with a valid API key from your Replicate account to authenticate requests.                   | Replicate API key management: https://replicate.com/account/api-keys |
| The audio input URL must be publicly accessible for the API to fetch and process the audio content.                             | Ensure audio URL is reachable via HTTPS.                         |
| Polling interval is fixed at 2 seconds; adjust if API rate limits or latency require different timing.                          | Consider API rate limits for Replicate.                          |
| No built-in timeout for infinite polling; consider adding maximum retries or timeout to prevent indefinite execution.           | Add error handling or timeout logic if needed.                   |

---

This documentation fully describes the workflow, enabling clear understanding, reproduction, and troubleshooting.