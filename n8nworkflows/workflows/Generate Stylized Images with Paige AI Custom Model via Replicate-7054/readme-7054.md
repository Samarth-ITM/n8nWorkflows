Generate Stylized Images with Paige AI Custom Model via Replicate

https://n8nworkflows.xyz/workflows/generate-stylized-images-with-paige-ai-custom-model-via-replicate-7054


# Generate Stylized Images with Paige AI Custom Model via Replicate

### 1. Workflow Overview

This workflow, titled **"Generate Stylized Images with Paige AI Custom Model via Replicate"**, leverages the Replicate API to generate stylized images using the **paigedutcher2/paige** AI model. It automates the process of submitting an image generation prompt, monitoring the prediction status asynchronously, and processing the final output once the model completes the generation.

The workflow is structured into the following logical blocks:

- **1.1 Trigger & API Key Setup:** Initiates the workflow manually and sets the Replicate API key for authentication.
- **1.2 Prediction Creation:** Sends a POST request to Replicate to start the image generation prediction with specified parameters.
- **1.3 Prediction ID Extraction & Initial Wait:** Extracts the prediction ID and status from the response and waits briefly before polling.
- **1.4 Polling for Prediction Completion:** Repeatedly checks the prediction status until it reports success.
- **1.5 Result Processing:** Once the prediction is complete, processes and formats the output data.

This design ensures reliable asynchronous handling of the Replicate model prediction lifecycle with error handling potential at each HTTP request and polling step.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger & API Key Setup

- **Overview:**  
  Starts the workflow manually and assigns the user’s Replicate API key to a variable for authenticated API calls.

- **Nodes Involved:**  
  - On clicking 'execute' (Manual Trigger)  
  - Set API Key (Set Node)

- **Node Details:**

  - **On clicking 'execute'**  
    - *Type:* Manual Trigger  
    - *Role:* Entry point for manual execution.  
    - *Configuration:* No parameters, triggers the workflow when user clicks execute.  
    - *Input/Output:* No input, outputs a single trigger event.  
    - *Potential Failures:* None intrinsic, manual start only.

  - **Set API Key**  
    - *Type:* Set Node  
    - *Role:* Stores Replicate API key in workflow context for authentication.  
    - *Configuration:* Assigns string value `YOUR_REPLICATE_API_KEY` to variable `replicate_api_key`.  
    - *Key Expressions:* None dynamic; user must replace placeholder with actual API key.  
    - *Input:* Trigger from manual node  
    - *Output:* Provides JSON containing `replicate_api_key` for downstream nodes.  
    - *Potential Failures:* Misconfiguration if API key is missing or invalid.

---

#### 1.2 Prediction Creation

- **Overview:**  
  Sends a POST request to Replicate’s prediction endpoint to start image generation using the specified model version and input parameters.

- **Nodes Involved:**  
  - Create Prediction (HTTP Request)

- **Node Details:**

  - **Create Prediction**  
    - *Type:* HTTP Request  
    - *Role:* Initiates prediction on Replicate API.  
    - *Configuration:*  
      - URL: `https://api.replicate.com/v1/predictions`  
      - Method: POST  
      - Body (JSON):  
        ```json
        {
          "version": "45f3e91abd75bedac5de72b4da3a63376053067c79fa1ba98a11aafd79a13732",
          "input": {
            "prompt": "prompt value",
            "seed": 1,
            "width": 1,
            "height": 1,
            "lora_scale": 1
          }
        }
        ```  
      - Headers:  
        - Authorization: Bearer token dynamically constructed from `replicate_api_key`  
        - Content-Type: application/json  
      - Timeout: 60 seconds  
    - *Expressions:* Authorization header uses expression: `={{ 'Bearer ' + $('Set API Key').item.json.replicate_api_key }}`  
    - *Input:* JSON with `replicate_api_key` from Set API Key node  
    - *Output:* JSON response containing prediction metadata including `id` and initial `status`.  
    - *Potential Failures:*  
      - HTTP errors (401 Unauthorized if API key invalid)  
      - Timeout if API slow to respond  
      - JSON parsing errors if response malformed  
      - Input parameter errors if prompt or version invalid  

---

#### 1.3 Prediction ID Extraction & Initial Wait

- **Overview:**  
  Extracts the prediction ID and initial status from the API response and prepares the URL for polling. Then waits 2 seconds before polling starts.

- **Nodes Involved:**  
  - Extract Prediction ID (Code)  
  - Wait (Wait)

- **Node Details:**

  - **Extract Prediction ID**  
    - *Type:* Code (JavaScript)  
    - *Role:* Parses API response to extract prediction ID, status, and constructs polling URL.  
    - *Code Summary:*  
      - Reads `id` and `status` from input JSON  
      - Constructs URL `https://api.replicate.com/v1/predictions/{predictionId}` for polling  
      - Returns object with keys: `predictionId`, `status`, `predictionUrl`  
    - *Input:* JSON response from Create Prediction node  
    - *Output:* JSON with extracted values for downstream polling  
    - *Potential Failures:* If response missing `id` or `status`, code may error or produce invalid URL.

  - **Wait**  
    - *Type:* Wait  
    - *Role:* Pauses workflow execution for 2 seconds before next polling call.  
    - *Configuration:* Wait time set to 2 seconds.  
    - *Input:* Output from Extract Prediction ID node  
    - *Output:* Passes through input after delay  
    - *Potential Failures:* Minimal; workflow delay only.

---

#### 1.4 Polling for Prediction Completion

- **Overview:**  
  Periodically queries the prediction status endpoint until the prediction’s status is `succeeded`. If not complete, waits and retries.

- **Nodes Involved:**  
  - Check Prediction Status (HTTP Request)  
  - Check If Complete (If)  
  - Wait (Wait) [looped from If node]

- **Node Details:**

  - **Check Prediction Status**  
    - *Type:* HTTP Request  
    - *Role:* Fetches latest prediction status from Replicate API.  
    - *Configuration:*  
      - URL: Dynamic from `predictionUrl` field  
      - Method: GET (default)  
      - Headers: Authorization with Bearer token (same as before)  
    - *Input:* JSON containing `predictionUrl` and API key  
    - *Output:* JSON with current prediction status and output if ready  
    - *Potential Failures:*  
      - HTTP errors (401, 404 if prediction ID invalid)  
      - Network timeouts  
      - JSON parse errors  

  - **Check If Complete**  
    - *Type:* If Node  
    - *Role:* Checks if prediction status equals "succeeded".  
    - *Condition:* Boolean equality check: `$json.status === "succeeded"`  
    - *Input:* JSON from Check Prediction Status  
    - *Output:*  
      - True branch: proceeds to Process Result  
      - False branch: loops back to Wait node to retry  
    - *Potential Failures:* If `status` field missing or unexpected values, logic may not behave as expected.  

  - **Wait** (reused)  
    - *Role:* Pauses 2 seconds before next polling iteration if not complete.  
    - *Loop:* Connects back to Check Prediction Status, enabling repeated polling.  

---

#### 1.5 Result Processing

- **Overview:**  
  Processes the final successful prediction output, extracting relevant metadata and returning a structured summary.

- **Nodes Involved:**  
  - Process Result (Code)

- **Node Details:**

  - **Process Result**  
    - *Type:* Code (JavaScript)  
    - *Role:* Extracts final prediction details and formats output for downstream use or storage.  
    - *Code Summary:*  
      - Reads entire prediction JSON result  
      - Returns an object with keys:  
        - `status` (prediction status)  
        - `output` (generated image URLs or data)  
        - `metrics` (performance metrics)  
        - `created_at` and `completed_at` timestamps  
        - `model` fixed string `paigedutcher2/paige`  
        - `other_url` (duplicate of output)  
    - *Input:* JSON of completed prediction from Check If Complete node  
    - *Output:* Structured JSON summarizing prediction results  
    - *Potential Failures:* If expected fields missing, output may be incomplete.

---

### 3. Summary Table

| Node Name               | Node Type           | Functional Role                        | Input Node(s)           | Output Node(s)             | Sticky Note                                                                                                                        |
|-------------------------|---------------------|-------------------------------------|------------------------|----------------------------|----------------------------------------------------------------------------------------------------------------------------------|
| On clicking 'execute'    | Manual Trigger      | Manual start of workflow             | —                      | Set API Key                |                                                                                                                                  |
| Set API Key             | Set                 | Store Replicate API key              | On clicking 'execute'   | Create Prediction          |                                                                                                                                  |
| Create Prediction       | HTTP Request        | Start prediction request to Replicate | Set API Key             | Extract Prediction ID      |                                                                                                                                  |
| Extract Prediction ID   | Code                | Extract prediction ID & status, prepare polling URL | Create Prediction       | Wait                       |                                                                                                                                  |
| Wait                    | Wait                | Pause before polling                 | Extract Prediction ID / Check If Complete (False branch) | Check Prediction Status      |                                                                                                                                  |
| Check Prediction Status | HTTP Request        | Poll prediction status               | Wait                   | Check If Complete          |                                                                                                                                  |
| Check If Complete       | If                  | Verify if prediction completed       | Check Prediction Status | Process Result / Wait      |                                                                                                                                  |
| Process Result          | Code                | Process final prediction output      | Check If Complete       | —                          |                                                                                                                                  |
| Sticky Note             | Sticky Note         | Documentation & instructions         | —                      | —                          | ## Paigedutcher2 Paige AI Generator<br><br>This workflow uses the **paigedutcher2/paige** model from Replicate to generate other content.<br><br>### Setup<br>1. Add your Replicate API key<br>2. Configure the input parameters<br>3. Run the workflow<br><br>### Model Details<br>- **Type**: Other Generation<br>- **Provider**: paigedutcher2<br>- **Required Fields**: prompt |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: `On clicking 'execute'`  
   - Purpose: To manually start the workflow.

2. **Add Set Node for API Key**  
   - Name: `Set API Key`  
   - Add a string field named `replicate_api_key`.  
   - Set value to your actual Replicate API key (replace placeholder `YOUR_REPLICATE_API_KEY`).  
   - Connect output of Manual Trigger node to this Set node.

3. **Create HTTP Request Node to Create Prediction**  
   - Name: `Create Prediction`  
   - HTTP Method: POST  
   - URL: `https://api.replicate.com/v1/predictions`  
   - Authentication: None (use HTTP header with Bearer token)  
   - Headers:  
     - `Authorization`: Expression: `'Bearer ' + $json["replicate_api_key"]` (reference from Set API Key node)  
     - `Content-Type`: `application/json`  
   - Body (Raw JSON):  
     ```json
     {
       "version": "45f3e91abd75bedac5de72b4da3a63376053067c79fa1ba98a11aafd79a13732",
       "input": {
         "prompt": "prompt value",
         "seed": 1,
         "width": 1,
         "height": 1,
         "lora_scale": 1
       }
     }
     ```  
   - Timeout: 60 seconds  
   - Connect output of Set API Key node to this node.

4. **Add Code Node to Extract Prediction ID**  
   - Name: `Extract Prediction ID`  
   - Mode: Run Once for Each Item  
   - Code (JavaScript):  
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
   - Connect output of Create Prediction node to this node.

5. **Add Wait Node**  
   - Name: `Wait`  
   - Wait Time: 2 seconds  
   - Connect output of Extract Prediction ID node to this node.

6. **Add HTTP Request Node to Check Prediction Status**  
   - Name: `Check Prediction Status`  
   - HTTP Method: GET (default)  
   - URL: Expression: `{{$json["predictionUrl"]}}`  
   - Headers:  
     - `Authorization`: Expression: `'Bearer ' + $json["replicate_api_key"]` (reference from Set API Key node)  
   - Connect output of Wait node to this node.

7. **Add If Node to Check Completion**  
   - Name: `Check If Complete`  
   - Condition: Boolean  
     - Value 1: Expression: `{{$json["status"]}}`  
     - Operation: Equal  
     - Value 2: `succeeded`  
   - Connect output of Check Prediction Status node to this node.

8. **Add Code Node to Process Result**  
   - Name: `Process Result`  
   - Mode: Run Once for Each Item  
   - Code (JavaScript):  
     ```javascript
     const result = $input.item.json;
     return {
       status: result.status,
       output: result.output,
       metrics: result.metrics,
       created_at: result.created_at,
       completed_at: result.completed_at,
       model: 'paigedutcher2/paige',
       other_url: result.output
     };
     ```  
   - Connect True output (success) of If node to this node.

9. **Loop Back for Polling**  
   - Connect False output (not complete) of If node back to the Wait node to retry polling.

10. **Adjust Node Positions for Clarity**

11. **Add a Sticky Note Node** (Optional)  
    - Content:  
      ```
      ## Paigedutcher2 Paige AI Generator

      This workflow uses the **paigedutcher2/paige** model from Replicate to generate other content.

      ### Setup
      1. Add your Replicate API key
      2. Configure the input parameters
      3. Run the workflow

      ### Model Details
      - Type: Other Generation
      - Provider: paigedutcher2
      - Required Fields: prompt
      ```

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                 | Context or Link                                                    |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------|
| This workflow depends on having a valid Replicate API key with permissions to use the `paigedutcher2/paige` model version `45f3e91abd75bedac5de72b4da3a63376053067c79fa1ba98a11aafd79a13732`. | Replicate API documentation: https://replicate.com/docs         |
| The prompt and other input parameters (`seed`, `width`, `height`, `lora_scale`) should be customized in the Create Prediction node to suit desired outputs. | Model input customization                                         |
| Polling interval is set to 2 seconds; adjust based on expected prediction time and API rate limits.                                                         | Performance tuning                                                |
| If the API key is invalid or quota exceeded, HTTP 401 or 429 errors will occur during Create Prediction or Polling nodes.                                 | Error scenarios                                                  |

---

**Disclaimer:** The text provided is extracted exclusively from an automated workflow designed with n8n, a workflow automation and integration tool. This processing strictly complies with content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.