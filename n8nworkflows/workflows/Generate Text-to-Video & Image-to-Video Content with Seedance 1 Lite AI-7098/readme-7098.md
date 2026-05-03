Generate Text-to-Video & Image-to-Video Content with Seedance 1 Lite AI

https://n8nworkflows.xyz/workflows/generate-text-to-video---image-to-video-content-with-seedance-1-lite-ai-7098


# Generate Text-to-Video & Image-to-Video Content with Seedance 1 Lite AI

### 1. Workflow Overview

This workflow, titled **Bytedance Seedance 1 Lite Video Generator**, automates the generation of video content using the **bytedance/seedance-1-lite** AI model hosted on Replicate. It is designed to accept a text prompt and a seed value, submit these parameters to the Replicate API to initiate video generation, then poll the API until the video generation completes, and finally process and output the resulting video URL along with relevant metadata.

The workflow is logically divided into the following blocks:

- **1.1 Manual Trigger & API Key Setup:** Initiates the workflow manually and configures the API key credential for authentication with Replicate.
- **1.2 Video Generation Request:** Sends a POST request to create a video generation prediction on Replicate using the specified model and prompt.
- **1.3 Prediction Status Handling:** Extracts the prediction ID and polls the API repeatedly with delays until the prediction status becomes "succeeded".
- **1.4 Result Processing:** Once the video generation is complete, processes the result to extract and structure key output data.

---

### 2. Block-by-Block Analysis

#### 1.1 Manual Trigger & API Key Setup

**Overview:**  
This block starts the workflow manually and sets the Replicate API key as a variable for use in subsequent HTTP requests.

**Nodes Involved:**  
- On clicking 'execute' (Manual Trigger)  
- Set API Key

**Node Details:**

- **On clicking 'execute'**  
  - *Type:* Manual Trigger  
  - *Role:* Entry point to manually start the workflow.  
  - *Config:* No parameters; triggers workflow execution on user command.  
  - *Input:* None  
  - *Output:* Triggers "Set API Key" node.  
  - *Edge Cases:* None, except user must initiate manually.

- **Set API Key**  
  - *Type:* Set  
  - *Role:* Stores the Replicate API key as a workflow variable `replicate_api_key`.  
  - *Config:* Assigns a string variable `replicate_api_key` with placeholder `"YOUR_REPLICATE_API_KEY"`.  
  - *Key Expressions:* Hardcoded placeholder string; user must replace with actual API key.  
  - *Input:* Trigger from Manual node  
  - *Output:* Passes API key to "Create Prediction" node.  
  - *Edge Cases:* Missing or invalid API key will cause authentication failures downstream.

---

#### 1.2 Video Generation Request

**Overview:**  
Sends a POST request to the Replicate API to create a new prediction (video generation job) with the specified prompt and seed.

**Nodes Involved:**  
- Create Prediction

**Node Details:**

- **Create Prediction**  
  - *Type:* HTTP Request  
  - *Role:* Initiates video generation by submitting a prediction job to Replicate.  
  - *Config:*  
    - Method: POST  
    - URL: `https://api.replicate.com/v1/predictions`  
    - Body (JSON):  
      ```json
      {
        "version": "b6519549e375404f45af5ef2e4b01f651d4014f3b57d3270b430e0523bad9835",
        "input": {
          "prompt": "prompt value",
          "seed": 1
        }
      }
      ```  
    - Headers:  
      - Authorization: Bearer token derived from `replicate_api_key` set earlier.  
      - Content-Type: application/json  
    - Timeout: 60 seconds  
  - *Key Expressions:*  
    - Authorization header dynamically constructed as `'Bearer ' + $('Set API Key').item.json.replicate_api_key`  
    - Prompt is statically assigned `"prompt value"` here (should be parameterized in practical use).  
  - *Input:* Receives API key from "Set API Key"  
  - *Output:* JSON response containing prediction ID and initial status, passed to "Extract Prediction ID".  
  - *Edge Cases:*  
    - API authentication errors if key invalid.  
    - Timeout if Replicate API slow or unreachable.  
    - Static prompt requires modification for dynamic input.

---

#### 1.3 Prediction Status Handling

**Overview:**  
Extracts the prediction ID from the response, then repeatedly polls the API to check the prediction status every 2 seconds until the job succeeds.

**Nodes Involved:**  
- Extract Prediction ID  
- Wait  
- Check Prediction Status  
- Check If Complete

**Node Details:**

- **Extract Prediction ID**  
  - *Type:* Code  
  - *Role:* Parses the prediction response to extract `id`, `status`, and constructs the prediction status URL.  
  - *Config:* JavaScript runs once per item, returns an object with:  
    - `predictionId` from response `id`  
    - `status` from response `status`  
    - `predictionUrl` to poll status (constructed as `https://api.replicate.com/v1/predictions/${predictionId}`)  
  - *Input:* Response from "Create Prediction"  
  - *Output:* JSON with predictionId, status, and URL for "Wait" node.  
  - *Edge Cases:* Malformed or missing data in API response could cause failures.

- **Wait**  
  - *Type:* Wait  
  - *Role:* Delays workflow execution for 2 seconds between polling attempts.  
  - *Config:* Duration set to 2 seconds.  
  - *Input:* Triggered after extracting prediction ID or after polling if not complete.  
  - *Output:* Triggers "Check Prediction Status" node.  
  - *Edge Cases:* None likely; short wait time to avoid rate limits.

- **Check Prediction Status**  
  - *Type:* HTTP Request  
  - *Role:* Sends GET request to the prediction status URL to retrieve current prediction state.  
  - *Config:*  
    - URL dynamically set from `predictionUrl` in input JSON  
    - Authorization header with Bearer token from API key  
  - *Input:* Triggered after wait  
  - *Output:* JSON response with updated prediction status, passed to "Check If Complete".  
  - *Edge Cases:*  
    - API rate limits or network issues causing failures or delays.  
    - Authorization failure if API key invalidated.

- **Check If Complete**  
  - *Type:* If  
  - *Role:* Checks if the prediction status equals `"succeeded"`.  
  - *Config:* Boolean condition comparing `$json.status` to `"succeeded"`.  
  - *Input:* Current prediction status JSON  
  - *Output:*  
    - If true: triggers "Process Result" node.  
    - If false: loops back to "Wait" node to poll again.  
  - *Edge Cases:*  
    - Status values other than "succeeded" (e.g., "failed", "canceled") are not explicitly handled; could cause infinite polling.

---

#### 1.4 Result Processing

**Overview:**  
Processes the successful prediction result to extract relevant metadata and the generated video URL.

**Nodes Involved:**  
- Process Result

**Node Details:**

- **Process Result**  
  - *Type:* Code  
  - *Role:* Extracts key information from the completed prediction result for downstream use.  
  - *Config:* JavaScript code extracts and returns:  
    - `status` (should be "succeeded")  
    - `output` (video URL)  
    - `metrics` (processing metrics from API)  
    - `created_at` and `completed_at` timestamps  
    - `model` name hardcoded as 'bytedance/seedance-1-lite'  
    - `video_url` as the output URL from result  
  - *Input:* Full JSON from completed prediction  
  - *Output:* Structured JSON with essential video generation details  
  - *Edge Cases:* If output is missing or incomplete, downstream tasks may fail.

---

### 3. Summary Table

| Node Name             | Node Type          | Functional Role                          | Input Node(s)             | Output Node(s)           | Sticky Note                                                                                           |
|-----------------------|--------------------|----------------------------------------|---------------------------|--------------------------|-----------------------------------------------------------------------------------------------------|
| On clicking 'execute'  | Manual Trigger     | Manual start of workflow                | None                      | Set API Key              |                                                                                                     |
| Set API Key           | Set                | Stores Replicate API key for auth       | On clicking 'execute'     | Create Prediction        |                                                                                                     |
| Create Prediction     | HTTP Request       | Initiates video generation on Replicate | Set API Key               | Extract Prediction ID    |                                                                                                     |
| Extract Prediction ID | Code               | Extracts prediction ID and status       | Create Prediction         | Wait                     |                                                                                                     |
| Wait                  | Wait               | Pauses workflow between polling attempts| Extract Prediction ID, Check If Complete | Check Prediction Status |                                                                                                     |
| Check Prediction Status| HTTP Request      | Polls Replicate API for prediction status| Wait                      | Check If Complete        |                                                                                                     |
| Check If Complete     | If                 | Checks if prediction has succeeded      | Check Prediction Status   | Process Result, Wait     |                                                                                                     |
| Process Result        | Code               | Extracts final video URL and metadata   | Check If Complete         | None                     |                                                                                                     |
| Sticky Note           | Sticky Note        | Documentation and instructions          | None                      | None                     | ## Bytedance Seedance 1 Lite Video Generator\n\nThis workflow uses the **bytedance/seedance-1-lite** model from Replicate to generate video content.\n\n### Setup\n1. Add your Replicate API key\n2. Configure the input parameters\n3. Run the workflow\n\n### Model Details\n- **Type**: Video Generation\n- **Provider**: bytedance\n- **Required Fields**: prompt |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node:**  
   - Name: `On clicking 'execute'`  
   - Purpose: Manually start the workflow.

2. **Add a Set Node for API Key:**  
   - Name: `Set API Key`  
   - Create a string variable named `replicate_api_key` with value `"YOUR_REPLICATE_API_KEY"` (replace with your actual API key).  
   - Connect `On clicking 'execute'` → `Set API Key`.

3. **Add HTTP Request Node to Create Prediction:**  
   - Name: `Create Prediction`  
   - HTTP Method: POST  
   - URL: `https://api.replicate.com/v1/predictions`  
   - Authentication: None (use HTTP Header Auth with Bearer token manually set)  
   - Headers:  
     - `Authorization`: `Bearer {{$node["Set API Key"].json["replicate_api_key"]}}`  
     - `Content-Type`: `application/json`  
   - Body Type: JSON  
   - Body Content (JSON):  
     ```json
     {
       "version": "b6519549e375404f45af5ef2e4b01f651d4014f3b57d3270b430e0523bad9835",
       "input": {
         "prompt": "prompt value",
         "seed": 1
       }
     }
     ```  
   - Timeout: 60000 ms  
   - Connect `Set API Key` → `Create Prediction`.

4. **Add a Code Node to Extract Prediction ID:**  
   - Name: `Extract Prediction ID`  
   - Run once per item  
   - JavaScript Code:  
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
   - Connect `Create Prediction` → `Extract Prediction ID`.

5. **Add a Wait Node:**  
   - Name: `Wait`  
   - Wait time: 2 seconds  
   - Connect `Extract Prediction ID` → `Wait`.

6. **Add HTTP Request Node to Check Prediction Status:**  
   - Name: `Check Prediction Status`  
   - HTTP Method: GET  
   - URL: `{{$json["predictionUrl"]}}`  
   - Headers:  
     - `Authorization`: `Bearer {{$node["Set API Key"].json["replicate_api_key"]}}`  
   - Connect `Wait` → `Check Prediction Status`.

7. **Add an If Node to Check Completion:**  
   - Name: `Check If Complete`  
   - Condition: Boolean  
     - Expression: `{{$json.status}} === "succeeded"`  
   - True output connects to `Process Result`  
   - False output loops back to `Wait` for polling again  
   - Connect `Check Prediction Status` → `Check If Complete`.

8. **Add a Code Node to Process the Result:**  
   - Name: `Process Result`  
   - Run once per item  
   - JavaScript Code:  
     ```javascript
     const result = $input.item.json;
     return {
       status: result.status,
       output: result.output,
       metrics: result.metrics,
       created_at: result.created_at,
       completed_at: result.completed_at,
       model: 'bytedance/seedance-1-lite',
       video_url: result.output
     };
     ```  
   - Connect `Check If Complete` (true) → `Process Result`.

9. **(Optional) Add a Sticky Note Node for Documentation:**  
   - Content:  
     ```
     ## Bytedance Seedance 1 Lite Video Generator

     This workflow uses the **bytedance/seedance-1-lite** model from Replicate to generate video content.

     ### Setup
     1. Add your Replicate API key
     2. Configure the input parameters
     3. Run the workflow

     ### Model Details
     - **Type**: Video Generation
     - **Provider**: bytedance
     - **Required Fields**: prompt
     ```  
   - Position it appropriately on the canvas.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                          |
|---------------------------------------------------------------------------------------------------------------|-----------------------------------------|
| The workflow requires a valid Replicate API key to authenticate requests. Obtain your key from https://replicate.com/account | Replicate API Key Setup                  |
| Model version ID `b6519549e375404f45af5ef2e4b01f651d4014f3b57d3270b430e0523bad9835` identifies the bytedance/seedance-1-lite model version | Model Version Identifier                 |
| The prompt input is currently hardcoded as `"prompt value"`; to generate custom videos, modify this dynamically | Input Parameter Customization            |
| Polling interval is 2 seconds; consider API rate limits and adjust accordingly                                | Polling Strategy                        |
| The workflow does not handle prediction failures explicitly; adding error handling for statuses like `"failed"` is recommended | Error Handling Improvement               |

---

**Disclaimer:** The provided text is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.