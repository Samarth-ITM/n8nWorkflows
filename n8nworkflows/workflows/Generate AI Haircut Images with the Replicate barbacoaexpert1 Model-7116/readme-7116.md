Generate AI Haircut Images with the Replicate barbacoaexpert1 Model

https://n8nworkflows.xyz/workflows/generate-ai-haircut-images-with-the-replicate-barbacoaexpert1-model-7116


# Generate AI Haircut Images with the Replicate barbacoaexpert1 Model

### 1. Workflow Overview

This workflow, titled **Barbacoaexpert1 Ai Haircuts AI Generator**, is designed to generate AI-based haircut images using the Replicate platform’s model `barbacoaexpert1/ai-haircuts`. It targets users who want to programmatically create haircut visuals by providing textual prompts as input.

The workflow is logically divided into the following blocks:

- **1.1 Manual Trigger and API Key Setup:** Entry point to initiate the workflow and set up the required API authentication.
- **1.2 Prediction Creation:** Sends a request to Replicate API to start the image generation based on the provided prompt and parameters.
- **1.3 Prediction Status Polling:** Periodically checks the status of the prediction until it completes.
- **1.4 Result Processing:** Once the prediction is successful, extracts and formats the generated output and relevant metadata.

---

### 2. Block-by-Block Analysis

#### 1.1 Manual Trigger and API Key Setup

- **Overview:**  
  This block initiates the workflow manually and sets the API key for authenticating requests to Replicate.

- **Nodes Involved:**  
  - On clicking 'execute' (Manual Trigger)  
  - Set API Key (Set)

- **Node Details:**  

  - **On clicking 'execute'**  
    - Type: Manual Trigger  
    - Role: Starts the workflow execution manually.  
    - Configuration: No parameters, simply triggers the flow on user interaction.  
    - Inputs: None  
    - Outputs: Connects to “Set API Key” node.  
    - Edge Cases: None.

  - **Set API Key**  
    - Type: Set  
    - Role: Defines the Replicate API key as a workflow variable for subsequent authentication.  
    - Configuration: Sets a string variable `replicate_api_key` where users must replace `"YOUR_REPLICATE_API_KEY"` with their actual API key.  
    - Inputs: From Manual Trigger  
    - Outputs: Connects to “Create Prediction” node.  
    - Edge Cases: Missing or invalid API key will cause authentication failures downstream.

#### 1.2 Prediction Creation

- **Overview:**  
  Sends a POST request to Replicate API to create a prediction job for the AI haircut image generation model.

- **Nodes Involved:**  
  - Create Prediction (HTTP Request)  
  - Extract Prediction ID (Code)

- **Node Details:**  

  - **Create Prediction**  
    - Type: HTTP Request  
    - Role: Calls Replicate’s `/v1/predictions` endpoint to initiate image generation.  
    - Configuration:  
      - Method: POST  
      - URL: `https://api.replicate.com/v1/predictions`  
      - Body (JSON): Contains the model version ID and input parameters such as `prompt`, `seed`, `width`, `height`, and `lora_scale`. The prompt is dynamically set as `"prompt value"` (should be replaced with the actual prompt string).  
      - Headers: Authorization with Bearer token from `replicate_api_key`, Content-Type application/json.  
      - Timeout: 60 seconds.  
      - Authentication: Generic HTTP Header Auth using the API key set earlier.  
    - Inputs: From “Set API Key” node  
    - Outputs: Connects to “Extract Prediction ID” node.  
    - Edge Cases:  
      - API key invalid or missing causes HTTP 401 Unauthorized.  
      - Network timeout or API errors.  
      - Improper prompt or invalid parameters might result in API errors.

  - **Extract Prediction ID**  
    - Type: Code (JavaScript)  
    - Role: Parses the API response to extract the prediction ID and initial status for polling.  
    - Configuration:  
      - Runs once per item (runOnceForEachItem)  
      - JavaScript extracts `id` and `status` from the API response JSON and constructs a prediction URL for polling.  
    - Inputs: From “Create Prediction” node  
    - Outputs: Connects to “Wait” node.  
    - Edge Cases:  
      - API response might be malformed or missing `id` causing failure.  
      - Prediction ID extraction failure would halt the workflow.

#### 1.3 Prediction Status Polling

- **Overview:**  
  Waits briefly and then polls the prediction status endpoint repeatedly until the prediction completes successfully.

- **Nodes Involved:**  
  - Wait (Wait)  
  - Check Prediction Status (HTTP Request)  
  - Check If Complete (If)

- **Node Details:**  

  - **Wait**  
    - Type: Wait  
    - Role: Pauses execution for 2 seconds between polling attempts to avoid excessive API calls.  
    - Configuration: Wait time set to 2 seconds.  
    - Inputs: From “Extract Prediction ID” and from “Check If Complete” (if prediction not complete).  
    - Outputs: Connects to “Check Prediction Status” node.  
    - Edge Cases: None, but excessive waiting might delay workflow completion.

  - **Check Prediction Status**  
    - Type: HTTP Request  
    - Role: Queries the prediction status endpoint using the stored prediction URL.  
    - Configuration:  
      - URL dynamically set from `$json.predictionUrl` (extracted earlier).  
      - Authorization header with Bearer token.  
      - Method: GET (default)  
    - Inputs: From “Wait” node  
    - Outputs: Connects to “Check If Complete” node  
    - Edge Cases:  
      - API errors or timeouts.  
      - Invalid prediction URL or expired prediction.  
      - Authentication failures if API key invalid.

  - **Check If Complete**  
    - Type: If  
    - Role: Checks if the prediction status equals `"succeeded"` to determine if processing can continue.  
    - Configuration: Boolean condition comparing `$json.status` to `"succeeded"`.  
    - Inputs: From “Check Prediction Status” node  
    - Outputs:  
      - True branch: connects to “Process Result” node.  
      - False branch: loops back to “Wait” node to continue polling.  
    - Edge Cases:  
      - Prediction status could be `"failed"` or `"canceled"` which is not handled explicitly, potentially causing infinite polling.

#### 1.4 Result Processing

- **Overview:**  
  Processes the successful prediction response to extract relevant output data, metadata, and prepares the final output.

- **Nodes Involved:**  
  - Process Result (Code)

- **Node Details:**  

  - **Process Result**  
    - Type: Code (JavaScript)  
    - Role: Extracts and formats the prediction result data including output URL, status, timestamps, and model info.  
    - Configuration:  
      - Runs once per item.  
      - Extracts `status`, `output`, `metrics`, `created_at`, and `completed_at` fields from the prediction response JSON.  
      - Adds fixed model identifier string `barbacoaexpert1/ai-haircuts`.  
      - Returns structured JSON containing all these fields for downstream use or storage.  
    - Inputs: From “Check If Complete” node (True branch)  
    - Outputs: Terminal node (no further connected nodes)  
    - Edge Cases:  
      - Missing fields in the response could cause undefined values.  
      - Output URL might be empty if prediction incomplete or failed silently.

---

### 3. Summary Table

| Node Name               | Node Type         | Functional Role                      | Input Node(s)               | Output Node(s)              | Sticky Note                                                                                                  |
|-------------------------|-------------------|------------------------------------|-----------------------------|-----------------------------|--------------------------------------------------------------------------------------------------------------|
| On clicking 'execute'    | Manual Trigger    | Workflow entry trigger              | -                           | Set API Key                 |                                                                                                              |
| Set API Key             | Set               | Stores Replicate API key            | On clicking 'execute'        | Create Prediction           |                                                                                                              |
| Create Prediction       | HTTP Request      | Sends prediction creation request  | Set API Key                 | Extract Prediction ID       |                                                                                                              |
| Extract Prediction ID   | Code              | Extracts prediction ID and status  | Create Prediction           | Wait                        |                                                                                                              |
| Wait                    | Wait              | Delays between polling attempts    | Extract Prediction ID, Check If Complete (False branch) | Check Prediction Status |                                                                                                              |
| Check Prediction Status | HTTP Request      | Polls prediction status             | Wait                        | Check If Complete           |                                                                                                              |
| Check If Complete       | If                | Checks if prediction succeeded     | Check Prediction Status     | Process Result (True), Wait (False) |                                                                                                              |
| Process Result          | Code              | Processes final prediction output  | Check If Complete (True)    | -                           |                                                                                                              |
| Sticky Note             | Sticky Note       | Notes on usage and setup            | -                           | -                           | ## Barbacoaexpert1 Ai Haircuts AI Generator\n\nThis workflow uses the **barbacoaexpert1/ai-haircuts** model from Replicate to generate other content.\n\n### Setup\n1. Add your Replicate API key\n2. Configure the input parameters\n3. Run the workflow\n\n### Model Details\n- **Type**: Other Generation\n- **Provider**: barbacoaexpert1\n- **Required Fields**: prompt |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node**  
   - Name it: `On clicking 'execute'`  
   - No configuration needed.

2. **Add a Set node**  
   - Name it: `Set API Key`  
   - Add a new string field `replicate_api_key` with value `"YOUR_REPLICATE_API_KEY"` (replace with your actual key).  
   - Connect output of `On clicking 'execute'` to input of this node.

3. **Add an HTTP Request node**  
   - Name it: `Create Prediction`  
   - Configure as follows:  
     - HTTP Method: POST  
     - URL: `https://api.replicate.com/v1/predictions`  
     - Authentication: Generic HTTP Header Auth  
       - Header Name: Authorization  
       - Header Value: `Bearer {{$node["Set API Key"].json["replicate_api_key"]}}`  
     - Headers: Add `Content-Type: application/json`  
     - Body Content Type: JSON  
     - JSON Body:  
       ```json
       {
         "version": "0e9b8dfe391b30dfd43b9f3d1ea5c745bb47da4f7d62bd281f9a7bc56dd138fd",
         "input": {
           "prompt": "prompt value",
           "seed": 1,
           "width": 1,
           "height": 1,
           "lora_scale": 1
         }
       }
       ```  
     - Timeout: 60000 ms (60 seconds)  
   - Connect output of `Set API Key` to input of this node.

4. **Add a Code node**  
   - Name it: `Extract Prediction ID`  
   - Run Mode: Run once for each item  
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
   - Connect output of `Create Prediction` to input of this node.

5. **Add a Wait node**  
   - Name it: `Wait`  
   - Configure to wait 2 seconds.  
   - Connect output of `Extract Prediction ID` to input of this node.

6. **Add an HTTP Request node**  
   - Name it: `Check Prediction Status`  
   - Configure as:  
     - URL: `{{$json["predictionUrl"]}}`  
     - Authentication: Generic HTTP Header Auth  
       - Header Name: Authorization  
       - Header Value: `Bearer {{$node["Set API Key"].json["replicate_api_key"]}}`  
     - Method: GET (default)  
   - Connect output of `Wait` to input of this node.

7. **Add an If node**  
   - Name it: `Check If Complete`  
   - Condition: Boolean  
     - Value 1: `{{$json["status"]}}`  
     - Operation: equal  
     - Value 2: `"succeeded"`  
   - Connect output of `Check Prediction Status` to input of this node.

8. **Add a Code node**  
   - Name it: `Process Result`  
   - Run Mode: Run once for each item  
   - JavaScript Code:  
     ```javascript
     const result = $input.item.json;

     return {
       status: result.status,
       output: result.output,
       metrics: result.metrics,
       created_at: result.created_at,
       completed_at: result.completed_at,
       model: 'barbacoaexpert1/ai-haircuts',
       other_url: result.output
     };
     ```  
   - Connect the **True** output branch of `Check If Complete` to this node.

9. **Connect the False output branch of `Check If Complete` back to `Wait`** to create a polling loop until prediction completes.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                 | Context or Link                                           |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------|
| This workflow uses the **barbacoaexpert1/ai-haircuts** model from Replicate for AI-generated haircut images. Setup requires adding your Replicate API key and configuring prompt inputs. | Sticky Note content in workflow                            |
| The Replicate API version used is `0e9b8dfe391b30dfd43b9f3d1ea5c745bb47da4f7d62bd281f9a7bc56dd138fd` specific to the barbacoaexpert1/ai-haircuts model.                          | Model version ID from workflow                             |
| Polling interval is set to 2 seconds to balance responsiveness and API rate limits, but can be adjusted as needed.                                                           | Configuration in Wait node                                 |
| The workflow assumes prompt and other input parameters are fixed or replaced manually; dynamic parameterization requires modifying the JSON body in "Create Prediction".     | Input parameter handling note                              |
| No explicit error handling for failed or canceled prediction statuses; adding such logic is recommended for production workflows.                                           | Edge case observation in polling block                     |

---

**Disclaimer:**  
The text provided is extracted exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.