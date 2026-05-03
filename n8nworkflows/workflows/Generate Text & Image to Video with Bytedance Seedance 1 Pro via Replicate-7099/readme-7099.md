Generate Text & Image to Video with Bytedance Seedance 1 Pro via Replicate

https://n8nworkflows.xyz/workflows/generate-text---image-to-video-with-bytedance-seedance-1-pro-via-replicate-7099


# Generate Text & Image to Video with Bytedance Seedance 1 Pro via Replicate

### 1. Workflow Overview

This workflow, titled **Bytedance Seedance 1 Pro Video Generator**, is designed to generate video content using the AI model **bytedance/seedance-1-pro** hosted on Replicate. It accepts a text prompt as input and leverages the Replicate API to create a video based on that prompt.

The workflow is structured into the following logical blocks:

- **1.1 Manual Trigger & API Key Setup**: Initiates the workflow manually and sets the required Replicate API key.
- **1.2 Video Generation Request**: Sends a prediction request to Replicate with the prompt and seed parameters.
- **1.3 Prediction Handling & Polling**: Extracts prediction metadata, polls the status of the video generation until completion.
- **1.4 Result Processing**: Processes the successful prediction output, extracting the video URL and metadata.

---

### 2. Block-by-Block Analysis

#### 1.1 Manual Trigger & API Key Setup

**Overview:**  
This initial block allows a user to manually start the workflow and sets the Replicate API key needed for authentication in subsequent API calls.

**Nodes Involved:**  
- On clicking 'execute' (Manual Trigger)  
- Set API Key (Set Node)

**Node Details:**

- **On clicking 'execute'**  
  - *Type & Role:* Manual Trigger; entry point for workflow execution by user interaction.  
  - *Configuration:* No parameters; simply triggers on manual user action in the n8n editor.  
  - *Input/Output:* No input; outputs a trigger signal to "Set API Key".  
  - *Edge cases:* None typical; manual trigger depends on user action.

- **Set API Key**  
  - *Type & Role:* Set node; assigns the Replicate API key as a workflow variable.  
  - *Configuration:* Assigns a string parameter named `replicate_api_key`. This key must be replaced by the user with their own Replicate API key (`YOUR_REPLICATE_API_KEY`).  
  - *Input/Output:* Input from manual trigger; output to "Create Prediction".  
  - *Edge cases:* Failure if API key is missing or invalid in the next HTTP request steps.

---

#### 1.2 Video Generation Request

**Overview:**  
This block constructs and sends the video generation request to Replicate's API, specifying the model version, prompt, and seed for reproducibility.

**Nodes Involved:**  
- Create Prediction (HTTP Request)  
- Extract Prediction ID (Code)

**Node Details:**

- **Create Prediction**  
  - *Type & Role:* HTTP Request node; sends POST request to Replicate to create a new prediction.  
  - *Configuration:*  
    - URL: `https://api.replicate.com/v1/predictions`  
    - Method: POST  
    - JSON Body: Includes fixed model version `"fb4b92e4be45c1ea50c94e71ff51ffd88fd6327e2c55efb431a9d88afdfaeb86"`, and input with `"prompt": "prompt value"` and `"seed": 1`.  
    - Authentication: HTTP header with Bearer token taken dynamically from `replicate_api_key` set earlier.  
    - Timeout: 60 seconds.  
  - *Input/Output:* Input from "Set API Key"; output JSON response containing prediction metadata.  
  - *Key Expressions:* Authorization header uses expression to insert API key: `={{ 'Bearer ' + $('Set API Key').item.json.replicate_api_key }}`  
  - *Edge cases:* HTTP timeout, invalid API key, malformed prompt, or API rate limits.

- **Extract Prediction ID**  
  - *Type & Role:* Code node; extracts prediction ID and status from API response for polling.  
  - *Configuration:* Runs once per item; JavaScript extracts `id` and `status` from response, constructs prediction URL for status checks.  
  - *Input/Output:* Input is the response from "Create Prediction"; output includes `predictionId`, `status`, and `predictionUrl`.  
  - *Edge cases:* Missing or malformed response data.

---

#### 1.3 Prediction Handling & Polling

**Overview:**  
This block polls the Replicate API to check the status of the video generation prediction, waiting and looping until the prediction is complete.

**Nodes Involved:**  
- Wait (Wait)  
- Check Prediction Status (HTTP Request)  
- Check If Complete (If)

**Node Details:**

- **Wait**  
  - *Type & Role:* Wait node; pauses workflow execution to avoid excessive polling frequency.  
  - *Configuration:* Waits for 2 seconds before the next status check.  
  - *Input/Output:* Input from "Extract Prediction ID" and also from "Check If Complete" fallback; outputs to "Check Prediction Status".  
  - *Edge cases:* Insufficient wait time may lead to rate limits or API overload.

- **Check Prediction Status**  
  - *Type & Role:* HTTP Request node; GET request to Replicate API to fetch current prediction status.  
  - *Configuration:*  
    - URL dynamically set from `predictionUrl` output by "Extract Prediction ID".  
    - Auth header uses the same Bearer token as before.  
  - *Input/Output:* Input from "Wait"; output JSON status info sent to "Check If Complete".  
  - *Edge cases:* HTTP errors, network issues, expired prediction ID.

- **Check If Complete**  
  - *Type & Role:* If node; checks if prediction status equals `"succeeded"`.  
  - *Configuration:* Boolean condition comparing `$json.status` to `"succeeded"`.  
  - *Input/Output:* Input from "Check Prediction Status";  
    - If true: proceeds to "Process Result".  
    - If false: loops back to "Wait" to continue polling.  
  - *Edge cases:* Other statuses such as `"failed"` or `"canceled"` are not explicitly handled — may cause infinite loop or need additional error handling.

---

#### 1.4 Result Processing

**Overview:**  
This block processes the final successful prediction output, extracting relevant metadata and the video URL for downstream use.

**Nodes Involved:**  
- Process Result (Code)

**Node Details:**

- **Process Result**  
  - *Type & Role:* Code node; extracts and structures important output details from the completed prediction.  
  - *Configuration:* Runs once per item; JavaScript extracts fields like `status`, `output` (video URL), `metrics`, `created_at`, `completed_at`, and adds a fixed `model` name.  
  - *Input/Output:* Input from "Check If Complete" true branch; output includes structured JSON with video URL and metadata.  
  - *Edge cases:* If output is missing or incomplete, may cause errors or downstream failures.

---

#### Additional Node: Sticky Note

- **Sticky Note**  
  - *Role:* Documentation within the workflow canvas.  
  - *Content Summary:*  
    - Explains workflow purpose: "Bytedance Seedance 1 Pro Video Generator" uses Replicate's bytedance/seedance-1-pro model for video generation.  
    - Setup instructions: Add API key, configure input, run workflow.  
    - Model details: Video generation, provider bytedance, requires prompt input.  
  - *Position:* Top-left area for user reference.

---

### 3. Summary Table

| Node Name               | Node Type          | Functional Role                       | Input Node(s)               | Output Node(s)             | Sticky Note                                                                                          |
|-------------------------|--------------------|------------------------------------|----------------------------|----------------------------|----------------------------------------------------------------------------------------------------|
| On clicking 'execute'    | Manual Trigger     | Starts workflow manually            | -                          | Set API Key                |                                                                                                    |
| Set API Key             | Set                | Sets Replicate API key variable     | On clicking 'execute'       | Create Prediction          |                                                                                                    |
| Create Prediction       | HTTP Request       | Sends video generation request      | Set API Key                | Extract Prediction ID      |                                                                                                    |
| Extract Prediction ID   | Code               | Extracts prediction ID and URL      | Create Prediction          | Wait                      |                                                                                                    |
| Wait                    | Wait               | Waits 2 seconds before polling      | Extract Prediction ID, Check If Complete (false branch) | Check Prediction Status |                                                                                                    |
| Check Prediction Status | HTTP Request       | Polls prediction status             | Wait                      | Check If Complete          |                                                                                                    |
| Check If Complete       | If                 | Checks if prediction succeeded      | Check Prediction Status    | Process Result (true), Wait (false) |                                                                                                    |
| Process Result          | Code               | Extracts video URL and metadata     | Check If Complete (true)   | -                          |                                                                                                    |
| Sticky Note             | Sticky Note        | Provides workflow documentation     | -                          | -                          | ## Bytedance Seedance 1 Pro Video Generator<br>This workflow uses the **bytedance/seedance-1-pro** model from Replicate to generate video content.<br>### Setup<br>1. Add your Replicate API key<br>2. Configure the input parameters<br>3. Run the workflow<br>### Model Details<br>- **Type**: Video Generation<br>- **Provider**: bytedance<br>- **Required Fields**: prompt |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Add a **Manual Trigger** node named `"On clicking 'execute'"`. No special configuration needed.

2. **Add Set Node for API Key:**  
   - Add a **Set** node named `"Set API Key"`.  
   - Add one assignment: variable name `replicate_api_key`, type `string`, value `"YOUR_REPLICATE_API_KEY"` (replace with your actual API key).  
   - Connect `"On clicking 'execute'"` → `"Set API Key"`.

3. **Create HTTP Request Node to Initiate Prediction:**  
   - Add an **HTTP Request** node named `"Create Prediction"`.  
   - Set:  
     - HTTP Method: `POST`  
     - URL: `https://api.replicate.com/v1/predictions`  
     - Authentication: Generic HTTP Header Auth  
     - Header Parameters:  
       - `Authorization`: Expression `={{ 'Bearer ' + $('Set API Key').item.json.replicate_api_key }}`  
       - `Content-Type`: `application/json`  
     - Body Content Type: JSON  
     - Body (JSON):  
       ```json
       {
         "version": "fb4b92e4be45c1ea50c94e71ff51ffd88fd6327e2c55efb431a9d88afdfaeb86",
         "input": {
           "prompt": "prompt value",
           "seed": 1
         }
       }
       ```  
     - Timeout: 60000 ms (60 seconds)  
   - Connect `"Set API Key"` → `"Create Prediction"`.

4. **Add Code Node to Extract Prediction ID and URL:**  
   - Add a **Code** node named `"Extract Prediction ID"`.  
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
   - Connect `"Create Prediction"` → `"Extract Prediction ID"`.

5. **Add Wait Node:**  
   - Add a **Wait** node named `"Wait"`.  
   - Wait time: 2 seconds  
   - Connect `"Extract Prediction ID"` → `"Wait"`.

6. **Add HTTP Request Node to Check Prediction Status:**  
   - Add an **HTTP Request** node named `"Check Prediction Status"`.  
   - Method: GET  
   - URL: Expression `={{ $json.predictionUrl }}` (from previous node output)  
   - Authentication: Generic HTTP Header Auth with same bearer token header as in step 3.  
   - Connect `"Wait"` → `"Check Prediction Status"`.

7. **Add If Node to Check Completion:**  
   - Add an **If** node named `"Check If Complete"`.  
   - Condition: Boolean  
     - Value 1: Expression `={{ $json.status }}`  
     - Operator: Equal  
     - Value 2: `succeeded`  
   - Connect `"Check Prediction Status"` → `"Check If Complete"`.

8. **Add Code Node to Process Result:**  
   - Add a **Code** node named `"Process Result"`.  
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
       model: 'bytedance/seedance-1-pro',
       video_url: result.output
     };
     ```  
   - Connect `"Check If Complete"` (true branch) → `"Process Result"`.

9. **Loop Back for Polling:**  
   - Connect `"Check If Complete"` (false branch) → `"Wait"` to continue polling until completion.

10. **Add a Sticky Note for Documentation (Optional):**  
    - Add a **Sticky Note** node with the following content to help users:  
      ```
      ## Bytedance Seedance 1 Pro Video Generator

      This workflow uses the **bytedance/seedance-1-pro** model from Replicate to generate video content.

      ### Setup
      1. Add your Replicate API key
      2. Configure the input parameters
      3. Run the workflow

      ### Model Details
      - **Type**: Video Generation
      - **Provider**: bytedance
      - **Required Fields**: prompt
      ```

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                      |
|---------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| This workflow requires a valid Replicate API key with access to the bytedance/seedance-1-pro model.           | https://replicate.com/bytedance/seedance-1-pro                                                     |
| The prompt parameter in the JSON body should be replaced or parameterized to suit specific video generation use cases. | Custom prompt input can be implemented via Set nodes or external triggers.                         |
| Polling interval is set to 2 seconds to balance API rate limits and responsiveness; adjust as needed.         | Consider Replicate API rate limits and adjust Wait node accordingly.                               |
| No explicit error handling for failed or canceled predictions is implemented; adding such branches is recommended for production use. | Enhance with additional If nodes or error handling nodes to cover failure states.                  |
| For security, store API keys securely in n8n credentials or environment variables rather than hardcoding.    | Use n8n Credential Management for HTTP Header Auth when possible.                                  |

---

**Disclaimer:** This documentation is generated exclusively from an n8n workflow designed for integration and automation. All data handled is legal and public. No illegal, offensive, or protected content is included.