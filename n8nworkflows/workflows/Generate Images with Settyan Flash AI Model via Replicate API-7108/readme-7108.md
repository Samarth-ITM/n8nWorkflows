Generate Images with Settyan Flash AI Model via Replicate API

https://n8nworkflows.xyz/workflows/generate-images-with-settyan-flash-ai-model-via-replicate-api-7108


# Generate Images with Settyan Flash AI Model via Replicate API

### 1. Workflow Overview

This workflow, titled **Settyan Flash V2.0.1 Beta.10 AI Generator**, automates image generation using the Replicate API with the AI model **settyan/flash-v2.0.1-beta.10**. Its primary use case is to submit a prompt and related parameters to the AI model, poll the prediction status until completion, and finally process and extract the resulting image data and metadata.

The workflow is logically divided into the following blocks:

- **1.1 Initialization & Input Setup**: Manual trigger and API key setup.
- **1.2 Prediction Creation**: Sending a generation request to Replicate with model parameters.
- **1.3 Prediction Polling**: Repeatedly checking the prediction status with delay intervals.
- **1.4 Result Processing**: Handling the completed prediction output and extracting relevant information.
- **1.5 Documentation**: A sticky note providing usage instructions and model details.

---

### 2. Block-by-Block Analysis

#### 1.1 Initialization & Input Setup

**Overview:**  
This block initiates the workflow manually and sets the required Replicate API key as a workflow variable for authentication in subsequent HTTP requests.

**Nodes Involved:**  
- On clicking 'execute'  
- Set API Key

**Node Details:**

- **On clicking 'execute'**  
  - *Type:* Manual Trigger  
  - *Role:* Entry point for the workflow; allows manual execution.  
  - *Configuration:* No parameters; triggers workflow manually.  
  - *Connections:* Outputs to "Set API Key".  
  - *Edge Cases:* None expected.  
  - *Version:* v1

- **Set API Key**  
  - *Type:* Set Node  
  - *Role:* Assigns the Replicate API key to a workflow variable `replicate_api_key` for authentication.  
  - *Configuration:* Hardcoded string assignment; user must replace `"YOUR_REPLICATE_API_KEY"` with a valid API key.  
  - *Key Expression:* None dynamic; static string.  
  - *Connections:* Inputs from "On clicking 'execute'"; outputs to "Create Prediction".  
  - *Edge Cases:* Failure if API key is missing or invalid.  
  - *Version:* v3.3

---

#### 1.2 Prediction Creation

**Overview:**  
Sends an HTTP POST request to Replicate’s prediction endpoint to initiate image generation using the specified AI model version and parameters.

**Nodes Involved:**  
- Create Prediction  
- Extract Prediction ID

**Node Details:**

- **Create Prediction**  
  - *Type:* HTTP Request  
  - *Role:* Calls the Replicate API to create a new prediction job.  
  - *Configuration:*  
    - URL: `https://api.replicate.com/v1/predictions`  
    - Method: POST  
    - Headers:  
      - Authorization: Bearer token from `replicate_api_key`  
      - Content-Type: application/json  
    - Body (JSON):  
      ```json
      {
        "version": "92078fec46fc705fceb16d17f8654b3b9aeea8b0b8ccaad08740b080e202aa10",
        "input": {
          "prompt": "prompt value",
          "seed": 1,
          "width": 1,
          "height": 1,
          "lora_scale": 1
        }
      }
      ```  
    - Timeout: 60000 ms  
    - Authentication: HTTP Header Auth with Bearer token.  
  - *Key Expressions:* Authorization header dynamically constructed using the API key set in "Set API Key".  
  - *Connections:* Input from "Set API Key"; output to "Extract Prediction ID".  
  - *Edge Cases:* HTTP errors (401 Unauthorized, 400 Bad Request), timeout, malformed JSON body, invalid model version or parameters.  
  - *Version:* v4.2

- **Extract Prediction ID**  
  - *Type:* Code (JavaScript)  
  - *Role:* Parses the API response to extract the prediction ID and initial status, preparing for polling.  
  - *Configuration:*  
    - Runs once per item.  
    - Extracts:  
      - `predictionId` from response `id` field  
      - `status` from response `status` field  
      - Constructs `predictionUrl` for status polling endpoint.  
  - *Connections:* Input from "Create Prediction"; output to "Wait".  
  - *Edge Cases:* Missing or malformed response JSON, missing `id` or `status`.  
  - *Version:* v2

---

#### 1.3 Prediction Polling

**Overview:**  
Waits for a short duration and then repeatedly queries the prediction status endpoint until the prediction completes successfully.

**Nodes Involved:**  
- Wait  
- Check Prediction Status  
- Check If Complete

**Node Details:**

- **Wait**  
  - *Type:* Wait  
  - *Role:* Introduces a delay (2 seconds) between status checks to avoid flooding the API.  
  - *Configuration:* Wait time of 2 seconds.  
  - *Connections:* Inputs from "Extract Prediction ID" and from "Check If Complete" fallback. Outputs to "Check Prediction Status".  
  - *Edge Cases:* None significant; excessive waits could delay workflow.  
  - *Version:* v1

- **Check Prediction Status**  
  - *Type:* HTTP Request  
  - *Role:* Queries the Replicate API for the current status of the prediction job.  
  - *Configuration:*  
    - URL dynamically set via expression from `predictionUrl` extracted earlier.  
    - Method: GET (default)  
    - Authorization header with Bearer token from "Set API Key".  
  - *Connections:* Input from "Wait"; output to "Check If Complete".  
  - *Edge Cases:* HTTP errors, network issues, invalid URL, unauthorized access, or API rate limits.  
  - *Version:* v4.2

- **Check If Complete**  
  - *Type:* If  
  - *Role:* Conditional check on whether the prediction status is `"succeeded"`.  
  - *Configuration:* Boolean condition comparing `$json.status` to `"succeeded"`.  
  - *Connections:*  
    - True branch to "Process Result"  
    - False branch loops back to "Wait" for continued polling.  
  - *Edge Cases:* Unexpected status values, infinite loop if prediction never completes.  
  - *Version:* v1

---

#### 1.4 Result Processing

**Overview:**  
Once the prediction is complete, this block extracts and formats the relevant output data for downstream use.

**Nodes Involved:**  
- Process Result

**Node Details:**

- **Process Result**  
  - *Type:* Code (JavaScript)  
  - *Role:* Extracts final prediction details including status, output URLs, metrics, timestamps, and model info.  
  - *Configuration:*  
    - Runs once per item.  
    - Extracts fields: `status`, `output`, `metrics`, `created_at`, `completed_at`.  
    - Adds hardcoded model identifier `settyan/flash-v2.0.1-beta.10`.  
    - Copies `output` URL to `other_url`.  
  - *Connections:* Input from "Check If Complete" (success branch).  
  - *Edge Cases:* Missing fields in response, malformed output data.  
  - *Version:* v2

---

#### 1.5 Documentation

**Overview:**  
Provides a sticky note within the workflow interface summarizing key information and setup instructions.

**Nodes Involved:**  
- Sticky Note

**Node Details:**

- **Sticky Note**  
  - *Type:* Sticky Note (UI element)  
  - *Role:* Displays concise instructions and model metadata for users.  
  - *Content Highlights:*  
    - Model: `settyan/flash-v2.0.1-beta.10`  
    - Setup steps: Add API key, configure inputs, run workflow  
    - Model type: Other Generation  
    - Required field: prompt  
  - *Connections:* None (informational only)  
  - *Edge Cases:* None  
  - *Version:* v1

---

### 3. Summary Table

| Node Name              | Node Type          | Functional Role                  | Input Node(s)           | Output Node(s)            | Sticky Note                                                                                   |
|------------------------|--------------------|--------------------------------|------------------------|---------------------------|-----------------------------------------------------------------------------------------------|
| On clicking 'execute'   | Manual Trigger     | Entry point to start workflow   | —                      | Set API Key               |                                                                                               |
| Set API Key            | Set                | Assign API key variable         | On clicking 'execute'   | Create Prediction          |                                                                                               |
| Create Prediction      | HTTP Request       | Initiate Replicate prediction   | Set API Key            | Extract Prediction ID      |                                                                                               |
| Extract Prediction ID  | Code               | Extract prediction ID & status  | Create Prediction      | Wait                      |                                                                                               |
| Wait                   | Wait               | Delay between status checks     | Extract Prediction ID, Check If Complete (false branch) | Check Prediction Status |                                                                                               |
| Check Prediction Status| HTTP Request       | Poll prediction status          | Wait                   | Check If Complete          |                                                                                               |
| Check If Complete      | If                 | Check if prediction succeeded   | Check Prediction Status| Process Result (true), Wait (false) |                                                                                               |
| Process Result         | Code               | Extract & format final output   | Check If Complete (true) | —                        |                                                                                               |
| Sticky Note            | Sticky Note        | Instructions and model info     | —                      | —                         | ## Settyan Flash V2.0.1 Beta.10 AI Generator. This workflow uses the **settyan/flash-v2.0.1-beta.10** model from Replicate to generate other content. Setup: 1. Add your Replicate API key 2. Configure the input parameters 3. Run the workflow. Model Details: - Type: Other Generation - Provider: settyan - Required Fields: prompt |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: `On clicking 'execute'`  
   - Purpose: Manual start of the workflow.

2. **Create Set Node to Store API Key**  
   - Name: `Set API Key`  
   - Assign a new variable named `replicate_api_key` of type string.  
   - Set its value to your valid Replicate API key (replace `"YOUR_REPLICATE_API_KEY"`).  
   - Connect `On clicking 'execute'` output to this node input.

3. **Create HTTP Request Node to Create Prediction**  
   - Name: `Create Prediction`  
   - HTTP Method: POST  
   - URL: `https://api.replicate.com/v1/predictions`  
   - Authentication: HTTP Header with Bearer token.  
     - Header `Authorization`: Expression: `Bearer {{$node["Set API Key"].json["replicate_api_key"]}}`  
   - Headers: `Content-Type: application/json`  
   - Body Type: JSON  
   - JSON Body Parameters:  
     - `version`: `"92078fec46fc705fceb16d17f8654b3b9aeea8b0b8ccaad08740b080e202aa10"` (model version)  
     - `input`: object with keys:  
       - `prompt`: string (e.g., `"prompt value"`; user should configure actual prompt)  
       - `seed`: 1  
       - `width`: 1  
       - `height`: 1  
       - `lora_scale`: 1  
   - Timeout: 60,000 ms  
   - Connect output of `Set API Key` to this node.

4. **Create Code Node to Extract Prediction ID and Status**  
   - Name: `Extract Prediction ID`  
   - Code (JavaScript):  
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

5. **Create Wait Node to Delay Polling**  
   - Name: `Wait`  
   - Configuration: Wait 2 seconds  
   - Connect output of `Extract Prediction ID` to this node.

6. **Create HTTP Request Node to Poll Prediction Status**  
   - Name: `Check Prediction Status`  
   - HTTP Method: GET (default)  
   - URL: Expression: `{{$json["predictionUrl"]}}`  
   - Authentication: HTTP Header with Bearer token.  
     - Header `Authorization`: Expression: `Bearer {{$node["Set API Key"].json["replicate_api_key"]}}`  
   - Connect output of `Wait` to this node.

7. **Create If Node to Check Prediction Completion**  
   - Name: `Check If Complete`  
   - Condition: Boolean check if `$json.status` equals `"succeeded"`  
   - Connect output of `Check Prediction Status` to this node.

8. **Create Code Node to Process Completed Result**  
   - Name: `Process Result`  
   - Code (JavaScript):  
     ```js
     const result = $input.item.json;
     return {
       status: result.status,
       output: result.output,
       metrics: result.metrics,
       created_at: result.created_at,
       completed_at: result.completed_at,
       model: 'settyan/flash-v2.0.1-beta.10',
       other_url: result.output
     };
     ```  
   - Connect the **true** output of `Check If Complete` to this node.

9. **Loop Back for Polling**  
   - Connect the **false** output of `Check If Complete` back to the `Wait` node to continue polling until completion.

10. **Add Sticky Note Node** (Optional but recommended)  
    - Content:  
      ```
      ## Settyan Flash V2.0.1 Beta.10 AI Generator

      This workflow uses the **settyan/flash-v2.0.1-beta.10** model from Replicate to generate other content.

      ### Setup
      1. Add your Replicate API key
      2. Configure the input parameters
      3. Run the workflow

      ### Model Details
      - Type: Other Generation
      - Provider: settyan
      - Required Fields: prompt
      ```
    - Place it visually near the start of the workflow for user reference.

---

### 5. General Notes & Resources

| Note Content                                                                                                           | Context or Link                                                                                         |
|------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| This workflow requires a valid Replicate API key with access to the `settyan/flash-v2.0.1-beta.10` model.               | Replicate API documentation: https://replicate.com/docs/reference/http                           |
| The model version ID `92078fec46fc705fceb16d17f8654b3b9aeea8b0b8ccaad08740b080e202aa10` must be kept up to date if changed. | Model version info: https://replicate.com/settyan/flash-v2.0.1-beta.10                                |
| Polling interval is set to 2 seconds; adjust `Wait` node if faster or slower polling is desired.                      | Consider API rate limits and latency for performance optimization.                                   |
| Prompt and other input parameters (`seed`, `width`, `height`, `lora_scale`) must be configured appropriately before run. | Input validation recommended to avoid API errors.                                                     |
| Handle API errors gracefully in production by adding error handling nodes or retry mechanisms around HTTP requests.    | n8n HTTP Request node supports retry and error handling configurations.                              |

---

*Disclaimer:* The provided text is exclusively derived from an automated n8n workflow. It complies strictly with content policies and does not contain illegal, offensive, or protected material. All data processed is legal and public.