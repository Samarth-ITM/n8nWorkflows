Generate 3D Models & Textures from Images with Hunyuan3D AI

https://n8nworkflows.xyz/workflows/generate-3d-models---textures-from-images-with-hunyuan3d-ai-7111


# Generate 3D Models & Textures from Images with Hunyuan3D AI

### 1. Workflow Overview

This workflow, titled **"Generate 3D Models & Textures from Images with Hunyuan3D AI,"** leverages the **ndreca/hunyuan3d-2.1-test** model available on Replicate to generate 3D models and textures based on input images. It is designed for users who want to automate the process of submitting an image to the AI model, monitor the asynchronous generation process, and retrieve the resulting 3D model and texture data once completed.

The workflow is organized into the following logical blocks:

- **1.1 Manual Trigger and API Key Setup:** Initiates the workflow manually and sets up the Replicate API key.
- **1.2 Prediction Creation:** Sends a request to the Replicate API to start a new prediction job using specified parameters.
- **1.3 Prediction Monitoring:** Polls the Replicate API to check the status of the prediction until it completes.
- **1.4 Result Processing:** Once the prediction succeeds, processes and extracts the relevant output data.

---

### 2. Block-by-Block Analysis

#### 1.1 Manual Trigger and API Key Setup

- **Overview:** This block starts the workflow manually and assigns the user’s Replicate API key to be used for authentication in subsequent HTTP requests.
- **Nodes Involved:**  
  - On clicking 'execute' (Manual Trigger)  
  - Set API Key (Set Node)

##### Node Details:

- **On clicking 'execute'**  
  - Type: Manual Trigger  
  - Role: Entry point to manually start the workflow by user action.  
  - Configuration: Default manual trigger, no parameters required.  
  - Input: None (external manual trigger)  
  - Output: Connected to "Set API Key" node  
  - Failures: None typical, but manual start required.

- **Set API Key**  
  - Type: Set Node  
  - Role: Stores the Replicate API key in workflow data for authentication.  
  - Configuration: Assigns a string variable `replicate_api_key` with placeholder `"YOUR_REPLICATE_API_KEY"`.  
  - Key Expression: Static string; user must replace with actual API key.  
  - Input: From manual trigger  
  - Output: To "Create Prediction" node  
  - Failures: Missing or incorrect API key will cause authentication errors downstream.

---

#### 1.2 Prediction Creation

- **Overview:** Sends a POST request to Replicate’s API to initiate a 3D model generation prediction job with specified parameters.
- **Nodes Involved:**  
  - Create Prediction (HTTP Request)  
  - Extract Prediction ID (Code)

##### Node Details:

- **Create Prediction**  
  - Type: HTTP Request  
  - Role: Calls Replicate API `/v1/predictions` endpoint to start a prediction.  
  - Configuration:  
    - Method: POST  
    - URL: `https://api.replicate.com/v1/predictions`  
    - Headers: Authorization Bearer token from `replicate_api_key`, Content-Type application/json  
    - Body (JSON):  
      - Model version: `70d0d816b74578a2ee5e7906284ab8bd26cf4f6a3fb7797dcd58059b803ae75f`  
      - Input parameters:  
        - image (URL placeholder `"https://example.com/input.other"`)  
        - seed: 1234  
        - steps: 50  
        - num_chunks: 8000  
        - max_facenum: 20000  
        - guidance_scale: 7.5  
        - generate_texture: true  
        - remove_background: true  
    - Timeout: 60 seconds (to accommodate API response delays)  
  - Input: From "Set API Key" node  
  - Output: To "Extract Prediction ID" node  
  - Failures & Edge Cases:  
    - HTTP errors (network, timeout)  
    - Authentication errors if API key invalid  
    - Validation errors if input parameters incorrect  
    - The placeholder image URL must be replaced with a valid accessible image URL.

- **Extract Prediction ID**  
  - Type: Code (JavaScript)  
  - Role: Parses the HTTP response to extract the prediction ID and initial status; constructs the prediction URL for subsequent polling.  
  - Configuration:  
    - Runs once per input item  
    - Extracts `id` and `status` from response JSON  
    - Returns an object with:  
      - `predictionId`  
      - `status`  
      - `predictionUrl` (constructed API endpoint to query prediction status)  
  - Input: Output of "Create Prediction" node  
  - Output: To "Wait" node  
  - Failures: Fails if response JSON structure changes or if `id` is missing.

---

#### 1.3 Prediction Monitoring

- **Overview:** Periodically checks the status of the prediction until it is marked as "succeeded" or otherwise completed.
- **Nodes Involved:**  
  - Wait (Wait)  
  - Check Prediction Status (HTTP Request)  
  - Check If Complete (If)

##### Node Details:

- **Wait**  
  - Type: Wait Node  
  - Role: Delays workflow execution by 2 seconds before polling the prediction status again.  
  - Configuration: Wait time set to 2 seconds  
  - Input: From "Extract Prediction ID" or from "Check If Complete" (if prediction not complete)  
  - Output: To "Check Prediction Status" node  
  - Failures: None typical; potential delay or timeout issues if external API slow.

- **Check Prediction Status**  
  - Type: HTTP Request  
  - Role: Sends GET request to the prediction URL to retrieve the current status of the prediction.  
  - Configuration:  
    - URL: Dynamically set via expression `{{$json.predictionUrl}}`  
    - Authentication: Bearer token from `replicate_api_key`  
    - Method: GET (default)  
  - Input: From "Wait" node  
  - Output: To "Check If Complete" node  
  - Failures: Network issues, authentication errors, invalid prediction URL.

- **Check If Complete**  
  - Type: If Node  
  - Role: Branches workflow based on prediction status.  
  - Configuration:  
    - Condition: Boolean check if `$json.status` equals `"succeeded"`  
    - True output: Proceed to result processing  
    - False output: Loop back to "Wait" node for another polling cycle  
  - Input: From "Check Prediction Status"  
  - Output:  
    - True: To "Process Result" node  
    - False: To "Wait" node  
  - Failures: Expression failures if `status` field missing or unexpected value.

---

#### 1.4 Result Processing

- **Overview:** Processes the successful prediction result, extracting useful metadata and outputs for final use.
- **Nodes Involved:**  
  - Process Result (Code)

##### Node Details:

- **Process Result**  
  - Type: Code (JavaScript)  
  - Role: Extracts and returns key information from the prediction result payload.  
  - Configuration:  
    - Runs once per input item  
    - Extracted fields:  
      - `status`  
      - `output` (likely URLs or data of generated 3D models/textures)  
      - `metrics` (performance or usage data)  
      - `created_at` and `completed_at` timestamps  
      - Adds static `model` field identifying the AI model used  
      - Includes `other_url` duplicating `output` for convenience  
  - Input: From "Check If Complete" (when true)  
  - Output: Workflow end or further processing (not defined)  
  - Failures: Missing fields or unexpected data structure may cause errors.

---

### 3. Summary Table

| Node Name             | Node Type          | Functional Role                  | Input Node(s)              | Output Node(s)              | Sticky Note                                                                                           |
|-----------------------|--------------------|---------------------------------|----------------------------|-----------------------------|-----------------------------------------------------------------------------------------------------|
| On clicking 'execute'  | Manual Trigger     | Start the workflow manually      | None                       | Set API Key                 |                                                                                                     |
| Set API Key           | Set                | Store Replicate API key          | On clicking 'execute'      | Create Prediction           |                                                                                                     |
| Create Prediction     | HTTP Request       | Send prediction creation request | Set API Key                | Extract Prediction ID       |                                                                                                     |
| Extract Prediction ID | Code (JavaScript)  | Extract prediction ID and URL    | Create Prediction          | Wait                       |                                                                                                     |
| Wait                  | Wait               | Delay before polling             | Extract Prediction ID / Check If Complete (false) | Check Prediction Status       |                                                                                                     |
| Check Prediction Status | HTTP Request     | Poll prediction status           | Wait                       | Check If Complete           |                                                                                                     |
| Check If Complete     | If                 | Branch: check if prediction done | Check Prediction Status    | Process Result / Wait       |                                                                                                     |
| Process Result        | Code (JavaScript)  | Extract & format output details  | Check If Complete (true)   | None                       |                                                                                                     |
| Sticky Note           | Sticky Note        | Documentation note               | None                       | None                       | ## Ndreca Hunyuan3d 2.1 Test AI Generator\n\nThis workflow uses the **ndreca/hunyuan3d-2.1-test** model from Replicate to generate other content.\n\n### Setup\n1. Add your Replicate API key\n2. Configure the input parameters\n3. Run the workflow\n\n### Model Details\n- **Type**: Other Generation\n- **Provider**: ndreca\n- **Required Fields**: image |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Name: "On clicking 'execute'"  
   - Purpose: Allows manual start of the workflow.

2. **Create Set Node for API Key**  
   - Type: Set  
   - Name: "Set API Key"  
   - Configuration: Add variable `replicate_api_key`  
   - Value: Set to your actual Replicate API key string (replace `"YOUR_REPLICATE_API_KEY"`).  
   - Connect input from "On clicking 'execute'".

3. **Create HTTP Request Node to Create Prediction**  
   - Type: HTTP Request  
   - Name: "Create Prediction"  
   - Method: POST  
   - URL: `https://api.replicate.com/v1/predictions`  
   - Authentication: HTTP Header Auth using `replicate_api_key` as Bearer token in Authorization header.  
   - Headers:  
     - Authorization: `Bearer {{$node["Set API Key"].json["replicate_api_key"]}}`  
     - Content-Type: `application/json`  
   - Body (JSON):  
     ```json
     {
       "version": "70d0d816b74578a2ee5e7906284ab8bd26cf4f6a3fb7797dcd58059b803ae75f",
       "input": {
         "image": "https://example.com/input.other",
         "seed": 1234,
         "steps": 50,
         "num_chunks": 8000,
         "max_facenum": 20000,
         "guidance_scale": 7.5,
         "generate_texture": true,
         "remove_background": true
       }
     }
     ```
   - Timeout: 60 seconds  
   - Connect input from "Set API Key".

4. **Create Code Node to Extract Prediction ID**  
   - Type: Code (JavaScript)  
   - Name: "Extract Prediction ID"  
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
   - Connect input from "Create Prediction".

5. **Create Wait Node**  
   - Type: Wait  
   - Name: "Wait"  
   - Duration: 2 seconds  
   - Connect input from "Extract Prediction ID".

6. **Create HTTP Request Node to Check Prediction Status**  
   - Type: HTTP Request  
   - Name: "Check Prediction Status"  
   - Method: GET (default)  
   - URL: Expression set to `{{$json["predictionUrl"]}}`  
   - Authentication: HTTP Header Auth with same Bearer token setup as "Create Prediction" node.  
   - Connect input from "Wait".

7. **Create If Node to Check Completion**  
   - Type: If  
   - Name: "Check If Complete"  
   - Condition: Boolean equals  
     - Value 1: `{{$json["status"]}}`  
     - Value 2: `"succeeded"`  
   - Connect input from "Check Prediction Status".

8. **Create Code Node to Process Result**  
   - Type: Code (JavaScript)  
   - Name: "Process Result"  
   - Code:  
     ```javascript
     const result = $input.item.json;

     return {
       status: result.status,
       output: result.output,
       metrics: result.metrics,
       created_at: result.created_at,
       completed_at: result.completed_at,
       model: 'ndreca/hunyuan3d-2.1-test',
       other_url: result.output
     };
     ```
   - Connect input from "Check If Complete" (true output).

9. **Loop Back for Polling**  
   - Connect "Check If Complete" (false output) back to "Wait" node to poll again.

10. **Sticky Note (Optional)**  
    - Create a Sticky Note node positioned suitably with content:  
      ```
      ## Ndreca Hunyuan3d 2.1 Test AI Generator

      This workflow uses the **ndreca/hunyuan3d-2.1-test** model from Replicate to generate other content.

      ### Setup
      1. Add your Replicate API key
      2. Configure the input parameters
      3. Run the workflow

      ### Model Details
      - **Type**: Other Generation
      - **Provider**: ndreca
      - **Required Fields**: image
      ```
    - This note provides user guidance and context.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                         | Context or Link                                                                                      |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| The workflow requires a valid Replicate API key. Obtain one from https://replicate.com/account/api-tokens                                                                                                                         | API Key Setup                                                                                       |
| Input image URL must be publicly accessible for the model to process it properly. Replace the placeholder `"https://example.com/input.other"` with a real image URL.                                                                | Input Parameter Requirement                                                                         |
| The model version ID used (`70d0d816b74578a2ee5e7906284ab8bd26cf4f6a3fb7797dcd58059b803ae75f`) is specific to the **ndreca/hunyuan3d-2.1-test** model; ensure this version ID is current by checking Replicate’s model page if errors arise. | Model Version                                                                                       |
| Polling interval is set to 2 seconds; consider adjusting based on expected prediction time and API rate limits.                                                                                                                    | Polling Configuration                                                                              |
| This workflow does not handle error states explicitly; adding error handling nodes or notifications for failed predictions or API errors is recommended for production use.                                                          | Suggested Improvement                                                                              |
| Official Replicate API documentation: https://replicate.com/docs/api-reference                                                                                                                                                      | API Documentation                                                                                  |

---

**Disclaimer:** The text provided is extracted exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected material. All data handled is legal and publicly available.