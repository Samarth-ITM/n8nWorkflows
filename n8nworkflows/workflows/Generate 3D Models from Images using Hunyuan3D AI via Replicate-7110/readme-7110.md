Generate 3D Models from Images using Hunyuan3D AI via Replicate

https://n8nworkflows.xyz/workflows/generate-3d-models-from-images-using-hunyuan3d-ai-via-replicate-7110


# Generate 3D Models from Images using Hunyuan3D AI via Replicate

### 1. Workflow Overview

This workflow automates the generation of 3D models from input images using the **ndreca/hunyuan3d-2-test** AI model hosted on Replicate. The process involves sending an image and parameters to the Replicate API, polling for the prediction status until completion, and then processing the results for further use.

The workflow is structured into the following logical blocks:

- **1.1 Input Trigger and API Key Setup:** Manual trigger to start the workflow and setting the Replicate API key.
- **1.2 Create Prediction Request:** Submitting the 3D model generation job to Replicate’s API with configured input parameters.
- **1.3 Polling Prediction Status:** Periodically checking the job status until it completes or fails.
- **1.4 Processing Completed Results:** Extracting and formatting the final output when the prediction succeeds.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Trigger and API Key Setup

- **Overview:** Initiates the workflow manually and sets the Replicate API key required for authentication in subsequent API calls.
- **Nodes Involved:**
  - On clicking 'execute'
  - Set API Key

##### Node: On clicking 'execute'
- **Type:** Manual Trigger
- **Role:** Starts the workflow execution on user command.
- **Configuration:** Default manual trigger without parameters.
- **Input/Output:** No input; outputs to Set API Key node.
- **Failure modes:** None (manual trigger).
- **Version:** 1

##### Node: Set API Key
- **Type:** Set
- **Role:** Defines the Replicate API key as a workflow variable for authentication.
- **Configuration:** Assigns a string variable `replicate_api_key` with placeholder `"YOUR_REPLICATE_API_KEY"`.
- **Expressions:** None.
- **Input:** From Manual Trigger.
- **Output:** To Create Prediction node.
- **Failure modes:** Missing or invalid API key leads to authorization errors downstream.
- **Version:** 3.3

---

#### 1.2 Create Prediction Request

- **Overview:** Sends a POST request to Replicate API to initiate the 3D model generation.
- **Nodes Involved:**
  - Create Prediction
  - Extract Prediction ID

##### Node: Create Prediction
- **Type:** HTTP Request
- **Role:** Submits the prediction job to Replicate’s API.
- **Configuration:**
  - Method: POST
  - URL: `https://api.replicate.com/v1/predictions`
  - JSON body includes:
    - Model version hash `"44d227e45a95353bd74bda59b1243d6b81758904e9345ef0adb20efadb166df3"`
    - Input parameters: image URL, seed, steps, chunk count, max face number, guidance scale, background removal flag.
  - Headers: Authorization Bearer token (`replicate_api_key`), Content-Type JSON.
  - Timeout: 60 seconds.
- **Expressions:** Authorization header uses expression to dynamically insert API key.
- **Input:** From Set API Key.
- **Output:** To Extract Prediction ID node.
- **Failure modes:** Network errors, invalid input parameters, authentication failure, timeout.
- **Version:** 4.2

##### Node: Extract Prediction ID
- **Type:** Code (JavaScript)
- **Role:** Parses the API response to extract the prediction ID and initial status, formats the polling URL.
- **Configuration:**
  - Runs once for each item.
  - Extracts `id`, `status` from response JSON.
  - Returns object with `predictionId`, `status`, and `predictionUrl`.
- **Input:** From Create Prediction.
- **Output:** To Wait node.
- **Failure modes:** Unexpected response structure, missing fields.
- **Version:** 2

---

#### 1.3 Polling Prediction Status

- **Overview:** Implements a polling loop that waits for 2 seconds, queries the prediction status, and decides whether to continue waiting or proceed.
- **Nodes Involved:**
  - Wait
  - Check Prediction Status
  - Check If Complete

##### Node: Wait
- **Type:** Wait
- **Role:** Pauses the workflow for 2 seconds between status checks.
- **Configuration:** Waits exactly 2 seconds.
- **Input:** From Extract Prediction ID or Check If Complete (false branch).
- **Output:** To Check Prediction Status.
- **Failure modes:** None significant; possible delays.
- **Version:** 1

##### Node: Check Prediction Status
- **Type:** HTTP Request
- **Role:** Sends a GET request to the prediction URL to fetch current status.
- **Configuration:**
  - URL taken dynamically from `$json.predictionUrl`.
  - Authorization header with Bearer token from API key.
  - No body.
- **Input:** From Wait.
- **Output:** To Check If Complete.
- **Failure modes:** Network errors, auth failures, invalid URL.
- **Version:** 4.2

##### Node: Check If Complete
- **Type:** If
- **Role:** Checks if prediction status equals "succeeded".
- **Configuration:** Condition compares `$json.status` == "succeeded".
- **Input:** From Check Prediction Status.
- **Output:** True branch to Process Result; False branch to Wait (poll again).
- **Failure modes:** If status field missing or unexpected values.
- **Version:** 1

---

#### 1.4 Processing Completed Results

- **Overview:** Processes and structures the final output once the prediction completes successfully.
- **Nodes Involved:**
  - Process Result

##### Node: Process Result
- **Type:** Code (JavaScript)
- **Role:** Extracts relevant data fields from the prediction result and formats them into a concise output object.
- **Configuration:**
  - Runs once per item.
  - Extracts status, output URLs, metrics, timestamps.
  - Adds a fixed model identifier.
  - Passes along the output URL for further usage.
- **Input:** From Check If Complete (true branch).
- **Output:** Final processed data.
- **Failure modes:** Unexpected response structure, missing fields.
- **Version:** 2

---

### 3. Summary Table

| Node Name             | Node Type           | Functional Role                    | Input Node(s)          | Output Node(s)          | Sticky Note                                                                                                                       |
|-----------------------|---------------------|----------------------------------|-----------------------|-------------------------|----------------------------------------------------------------------------------------------------------------------------------|
| On clicking 'execute'  | Manual Trigger      | Start workflow manually           | -                     | Set API Key             |                                                                                                                                  |
| Set API Key           | Set                 | Define Replicate API key          | On clicking 'execute'  | Create Prediction       |                                                                                                                                  |
| Create Prediction     | HTTP Request        | Submit 3D model generation job    | Set API Key            | Extract Prediction ID   |                                                                                                                                  |
| Extract Prediction ID | Code (JavaScript)   | Extract prediction ID and URL     | Create Prediction      | Wait                    |                                                                                                                                  |
| Wait                  | Wait                | Pause between polling attempts    | Extract Prediction ID, Check If Complete (false) | Check Prediction Status |                                                                                                                                  |
| Check Prediction Status | HTTP Request      | Check status of prediction        | Wait                   | Check If Complete       |                                                                                                                                  |
| Check If Complete     | If                  | Determine if prediction finished  | Check Prediction Status | Process Result (true), Wait (false) |                                                                                                                                  |
| Process Result        | Code (JavaScript)   | Format final prediction output    | Check If Complete (true) | -                       |                                                                                                                                  |
| Sticky Note           | Sticky Note         | Documentation and setup notes     | -                      | -                       | ## Ndreca Hunyuan3d 2 Test AI Generator<br><br>This workflow uses the **ndreca/hunyuan3d-2-test** model from Replicate to generate other content.<br><br>### Setup<br>1. Add your Replicate API key<br>2. Configure the input parameters<br>3. Run the workflow<br><br>### Model Details<br>- **Type**: Other Generation<br>- **Provider**: ndreca<br>- **Required Fields**: image |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Manual Trigger node:**
   - Name: `On clicking 'execute'`
   - Purpose: To manually start the workflow.

3. **Add a Set node:**
   - Name: `Set API Key`
   - Connect input from the Manual Trigger node.
   - Add a new string field named `replicate_api_key`.
   - Set the value to your Replicate API key string (replace `"YOUR_REPLICATE_API_KEY"` with your actual key).

4. **Add an HTTP Request node:**
   - Name: `Create Prediction`
   - Connect input from `Set API Key`.
   - Method: POST
   - URL: `https://api.replicate.com/v1/predictions`
   - Authentication: None for n8n credentials, but use HTTP Header Authentication by setting the Authorization header manually.
   - Headers:
     - `Authorization`: Use expression `Bearer {{$node["Set API Key"].json["replicate_api_key"]}}`
     - `Content-Type`: `application/json`
   - Body Content Type: JSON
   - Body:
     ```json
     {
       "version": "44d227e45a95353bd74bda59b1243d6b81758904e9345ef0adb20efadb166df3",
       "input": {
         "image": "https://example.com/input.other",
         "seed": 1234,
         "steps": 50,
         "num_chunks": 200000,
         "max_facenum": 40000,
         "guidance_scale": 5.5,
         "remove_background": true
       }
     }
     ```
   - Timeout: 60 seconds

5. **Add a Code node:**
   - Name: `Extract Prediction ID`
   - Connect input from `Create Prediction`.
   - Mode: Run Once For Each Item
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

6. **Add a Wait node:**
   - Name: `Wait`
   - Connect input from `Extract Prediction ID`.
   - Set wait time to 2 seconds.

7. **Add an HTTP Request node:**
   - Name: `Check Prediction Status`
   - Connect input from `Wait`.
   - Method: GET
   - URL: Use expression `{{$json["predictionUrl"]}}`
   - Headers:
     - `Authorization`: Use expression `Bearer {{$node["Set API Key"].json["replicate_api_key"]}}`

8. **Add an If node:**
   - Name: `Check If Complete`
   - Connect input from `Check Prediction Status`.
   - Condition: Boolean
     - Value 1: Expression `{{$json["status"]}}`
     - Operation: equals
     - Value 2: `succeeded`
   - True output: Connect to next step
   - False output: Connect back to `Wait` node (creates polling loop).

9. **Add a Code node:**
   - Name: `Process Result`
   - Connect input from `Check If Complete` (true branch).
   - Mode: Run Once For Each Item
   - Code:
     ```javascript
     const result = $input.item.json;

     return {
       status: result.status,
       output: result.output,
       metrics: result.metrics,
       created_at: result.created_at,
       completed_at: result.completed_at,
       model: 'ndreca/hunyuan3d-2-test',
       other_url: result.output
     };
     ```

10. **Optional: Add a Sticky Note node:**
    - Content:
      ```
      ## Ndreca Hunyuan3d 2 Test AI Generator

      This workflow uses the **ndreca/hunyuan3d-2-test** model from Replicate to generate other content.

      ### Setup
      1. Add your Replicate API key
      2. Configure the input parameters
      3. Run the workflow

      ### Model Details
      - Type: Other Generation
      - Provider: ndreca
      - Required Fields: image
      ```

---

### 5. General Notes & Resources

| Note Content                                                                                                              | Context or Link                              |
|---------------------------------------------------------------------------------------------------------------------------|----------------------------------------------|
| This workflow uses the Replicate API for AI-based 3D model generation with the `ndreca/hunyuan3d-2-test` model.            | Model details on https://replicate.com       |
| Ensure your Replicate API key is valid and has access permissions to the model version used in the workflow.               | https://replicate.com/docs                    |
| The workflow polls the API every 2 seconds; adjust wait time to optimize for rate limits or response times.                 | n8n Wait node documentation                   |
| Input image URL must be accessible publicly or via authentication if Replicate API requires it.                            | Input image URL configuration                  |
| The workflow assumes the standard Replicate prediction response structure. Unexpected API changes may require updates.    | Replicate API changelog                        |

---

**Disclaimer:**  
The provided text is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected material. All data handled is legal and publicly available.