Generate Images with Settyan Flash AI Model via Replicate API

https://n8nworkflows.xyz/workflows/generate-images-with-settyan-flash-ai-model-via-replicate-api-7103


# Generate Images with Settyan Flash AI Model via Replicate API

---

### 1. Workflow Overview

This workflow, titled **Settyan Flash V2.0.1 Beta.10 AI Generator**, facilitates the generation of images using the **settyan/flash-v2.0.1-beta.10** AI model via the Replicate API. It is designed for users who want to programmatically generate content based on textual prompts by interacting with the Replicate prediction endpoint.

The workflow logically segments into the following functional blocks:

- **1.1 Manual Trigger & API Key Setup:** Initiates the workflow manually and sets the required Replicate API key.
- **1.2 Prediction Creation:** Sends a request to the Replicate API to create a prediction job for image generation using the specified model and input parameters.
- **1.3 Prediction ID Extraction:** Extracts the prediction ID and constructs the status check URL for subsequent polling.
- **1.4 Polling for Prediction Completion:** Periodically checks the prediction status until it is complete.
- **1.5 Result Processing:** Processes the final output and metadata once the prediction is successful.

---

### 2. Block-by-Block Analysis

#### 2.1 Manual Trigger & API Key Setup

- **Overview:**  
  This block starts the workflow manually and sets the Replicate API key as a string variable for authorization in API requests.

- **Nodes Involved:**  
  - On clicking 'execute'  
  - Set API Key

- **Node Details:**

  - **On clicking 'execute'**  
    - Type: Manual Trigger  
    - Role: Entry point to manually start the workflow.  
    - Configuration: No parameters; simply triggers the flow on user action.  
    - Input: None  
    - Output: Connected to "Set API Key" node.  
    - Failure modes: None as it is manual trigger.  

  - **Set API Key**  
    - Type: Set  
    - Role: Stores the Replicate API key in a workflow variable for reuse.  
    - Configuration: Sets variable `replicate_api_key` with placeholder `YOUR_REPLICATE_API_KEY`.  
    - Key expressions: None dynamic, static string placeholder.  
    - Input: From manual trigger  
    - Output: To "Create Prediction" node.  
    - Edge cases: If not replaced with a valid key, API calls will fail with authorization errors.

---

#### 2.2 Prediction Creation

- **Overview:**  
  Sends a POST request to the Replicate API to initiate a prediction job for image generation. Uses the AI model version and user input prompt.

- **Nodes Involved:**  
  - Create Prediction

- **Node Details:**

  - **Create Prediction**  
    - Type: HTTP Request  
    - Role: Creates a new prediction job on Replicate.  
    - Configuration:  
      - URL: `https://api.replicate.com/v1/predictions`  
      - Method: POST  
      - Body: JSON containing:  
        - `version` set to the specific model version ID `"92078fec46fc705fceb16d17f8654b3b9aeea8b0b8ccaad08740b080e202aa10"`  
        - `input` object with parameters: `prompt`, `seed`, `width`, `height`, `lora_scale` (all currently hardcoded, with prompt as a placeholder "prompt value" and others set to 1)  
      - Headers: Authorization using Bearer token dynamically from `replicate_api_key` set in previous node; Content-Type `application/json`.  
      - Timeout: 60 seconds to accommodate model processing initiation.  
    - Input: From "Set API Key" node  
    - Output: To "Extract Prediction ID" node  
    - Edge cases:  
      - Invalid API key → 401 Unauthorized  
      - Network timeout  
      - Invalid input parameters → 400 Bad Request  
      - API rate limits or quota exceeded  
    - Version-specific: Uses HTTP header auth with generic credential type.  

---

#### 2.3 Prediction ID Extraction

- **Overview:**  
  Extracts the unique prediction ID and initial status from the API response. Constructs a URL for polling the prediction status.

- **Nodes Involved:**  
  - Extract Prediction ID

- **Node Details:**

  - **Extract Prediction ID**  
    - Type: Code (JavaScript)  
    - Role: Parses the response JSON to extract `id` and `status` fields, then builds a polling URL for status checking.  
    - Configuration:  
      - Runs once per item  
      - JS code extracts:  
        - `predictionId` from `id` field  
        - `initialStatus` from `status` field  
        - Constructs `predictionUrl` as `https://api.replicate.com/v1/predictions/${predictionId}`  
    - Input: From "Create Prediction" node output  
    - Output: To "Wait" node  
    - Edge cases:  
      - Missing or malformed response fields → code failure or undefined variables  
      - Empty or null `id` or `status`  
    - Version-specific: None

---

#### 2.4 Polling for Prediction Completion

- **Overview:**  
  Implements a polling mechanism that waits briefly, then requests the prediction status repeatedly until the prediction is complete.

- **Nodes Involved:**  
  - Wait  
  - Check Prediction Status  
  - Check If Complete

- **Node Details:**

  - **Wait**  
    - Type: Wait  
    - Role: Pauses workflow execution for 2 seconds between polls to avoid excessive API requests.  
    - Configuration: 2 seconds delay  
    - Input: From "Extract Prediction ID" initially, later also from "Check If Complete" if prediction incomplete  
    - Output: To "Check Prediction Status"  
    - Edge cases: None significant; short wait time may cause rapid polling if misused.  

  - **Check Prediction Status**  
    - Type: HTTP Request  
    - Role: Sends GET request to the constructed prediction URL to fetch current status and output.  
    - Configuration:  
      - URL dynamically assigned from `predictionUrl` extracted earlier  
      - Uses Bearer token from `replicate_api_key` for authorization  
      - GET request implied by no method override (default GET)  
    - Input: From "Wait"  
    - Output: To "Check If Complete"  
    - Edge cases:  
      - Unauthorized access (expired/invalid token)  
      - Network errors  
      - API downtime  
      - Rate limit errors  

  - **Check If Complete**  
    - Type: If  
    - Role: Branches workflow based on whether the prediction status is `"succeeded"` (complete).  
    - Configuration:  
      - Condition: Boolean equals, comparing `$json.status` to `"succeeded"`  
    - Input: From "Check Prediction Status"  
    - Output:  
      - True branch: To "Process Result"  
      - False branch: Loops back to "Wait" for another polling iteration  
    - Edge cases:  
      - Other statuses such as `"failed"`, `"processing"`, `"starting"` are not explicitly handled and will loop indefinitely if not "succeeded"  
      - No timeout or maximum retry count implemented  

---

#### 2.5 Result Processing

- **Overview:**  
  Once the prediction is successful, this block processes and formats the output and metadata for downstream use or export.

- **Nodes Involved:**  
  - Process Result

- **Node Details:**

  - **Process Result**  
    - Type: Code (JavaScript)  
    - Role: Extracts relevant fields from the prediction result JSON and formats them into a simplified output structure.  
    - Configuration:  
      - Runs once per item  
      - Extracted fields:  
        - `status`  
        - `output` (likely URLs or data of generated images)  
        - `metrics` (performance or resource data)  
        - `created_at` and `completed_at` timestamps  
        - Adds static model identifier `"settyan/flash-v2.0.1-beta.10"`  
        - Adds `other_url` duplicating output field for convenience  
    - Input: From "Check If Complete" true branch  
    - Output: Terminal node (no further nodes connected)  
    - Edge cases:  
      - Missing output data if prediction result malformed  
      - Null or incomplete timestamps or metrics  

---

### 3. Summary Table

| Node Name               | Node Type        | Functional Role                     | Input Node(s)            | Output Node(s)             | Sticky Note                                                                                                           |
|-------------------------|------------------|-----------------------------------|--------------------------|----------------------------|-----------------------------------------------------------------------------------------------------------------------|
| On clicking 'execute'    | Manual Trigger   | Manual workflow start              | None                     | Set API Key                |                                                                                                                       |
| Set API Key             | Set              | Stores Replicate API key           | On clicking 'execute'     | Create Prediction          |                                                                                                                       |
| Create Prediction       | HTTP Request     | Requests prediction creation       | Set API Key              | Extract Prediction ID      |                                                                                                                       |
| Extract Prediction ID   | Code             | Extracts prediction ID & URL       | Create Prediction        | Wait                      |                                                                                                                       |
| Wait                    | Wait             | Waits 2 seconds between polls      | Extract Prediction ID, Check If Complete (false branch) | Check Prediction Status |                                                                                                                       |
| Check Prediction Status | HTTP Request     | Polls prediction status            | Wait                     | Check If Complete          |                                                                                                                       |
| Check If Complete       | If               | Checks if prediction succeeded     | Check Prediction Status  | Process Result (true), Wait (false) |                                                                                                                       |
| Process Result          | Code             | Processes and formats final output | Check If Complete (true) | None                      |                                                                                                                       |
| Sticky Note             | Sticky Note      | Documentation note                 | None                     | None                      | ## Settyan Flash V2.0.1 Beta.10 AI Generator  This workflow uses the **settyan/flash-v2.0.1-beta.10** model from Replicate to generate other content.  ### Setup 1. Add your Replicate API key 2. Configure the input parameters 3. Run the workflow  ### Model Details - **Type**: Other Generation - **Provider**: settyan - **Required Fields**: prompt |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: `On clicking 'execute'`  
   - Type: Manual Trigger  
   - No parameters needed.

2. **Create Set Node for API Key**  
   - Name: `Set API Key`  
   - Type: Set  
   - Add a string field named `replicate_api_key` with value `YOUR_REPLICATE_API_KEY` (replace with your actual Replicate API key).  
   - Connect output of `On clicking 'execute'` to this node.

3. **Create HTTP Request Node to Create Prediction**  
   - Name: `Create Prediction`  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.replicate.com/v1/predictions`  
   - Body Content Type: JSON  
   - JSON Body:  
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
   - Authentication: Generic HTTP Header Authentication  
     - Header Name: `Authorization`  
     - Header Value: `Bearer {{$node["Set API Key"].json["replicate_api_key"]}}`  
   - Add Header: `Content-Type: application/json`  
   - Timeout: 60 seconds  
   - Connect output of `Set API Key` to this node.

4. **Create Code Node to Extract Prediction ID**  
   - Name: `Extract Prediction ID`  
   - Type: Code (JavaScript)  
   - Run once per item  
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
   - Wait for 2 seconds  
   - Connect output of `Extract Prediction ID` to this node.

6. **Create HTTP Request Node to Check Prediction Status**  
   - Name: `Check Prediction Status`  
   - Type: HTTP Request  
   - URL: `{{$json["predictionUrl"]}}` (dynamic URL from previous code node)  
   - Authentication: Generic HTTP Header Authentication  
     - Header Name: `Authorization`  
     - Header Value: `Bearer {{$node["Set API Key"].json["replicate_api_key"]}}`  
   - Connect output of `Wait` to this node.

7. **Create If Node to Check Completion**  
   - Name: `Check If Complete`  
   - Type: If  
   - Condition: Boolean equals  
     - Value 1: `{{$json["status"]}}`  
     - Value 2: `succeeded`  
   - Connect output of `Check Prediction Status` to this node.

8. **Create Code Node to Process Result**  
   - Name: `Process Result`  
   - Type: Code (JavaScript)  
   - Run once per item  
   - Code:  
     ```javascript
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
   - Connect the **true** branch of `Check If Complete` to this node.

9. **Loop Back for Polling**  
   - Connect the **false** branch of `Check If Complete` back to the `Wait` node to repeat polling until completion.

10. **Optional: Add Sticky Note**  
    - Create a Sticky Note node with the following content (for user reference):  
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
    - Position it visually near the manual trigger node.

---

### 5. General Notes & Resources

| Note Content                                                                                       | Context or Link                                |
|--------------------------------------------------------------------------------------------------|------------------------------------------------|
| This workflow requires a valid **Replicate API key** with access to the model version specified. | Replicate API: https://replicate.com/docs/api |
| The polling mechanism lacks a timeout or max retries; consider adding to avoid infinite loops.   | Workflow improvement suggestion                 |
| Model input parameters are currently hardcoded; for production, make `prompt` and other inputs dynamic. | Workflow customization suggestion               |
| Sticky note explains setup and model details for easy onboarding.                                | User instruction embedded in workflow          |

---

**Disclaimer:** The text provided is derived exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with prevailing content policies and contains no illegal, offensive, or protected elements. All manipulated data is lawful and public.

---