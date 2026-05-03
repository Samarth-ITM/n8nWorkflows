Generate AI Images with Replicate's Settyan Flash Model

https://n8nworkflows.xyz/workflows/generate-ai-images-with-replicate-s-settyan-flash-model-7102


# Generate AI Images with Replicate's Settyan Flash Model

### 1. Workflow Overview

This workflow, titled **Settyan Flash V2.0.2 Beta.10 AI Generator**, automates the generation of AI images using the **settyan/flash-v2.0.2-beta.10** model hosted on Replicate. It is designed to submit an image generation request based on a prompt, poll the prediction status until completion, and then process and output the generated image data.

The workflow is logically divided into these blocks:

- **1.1 Manual Trigger & API Key Setup:** Initiates the workflow manually and sets the necessary API key for Replicate.
- **1.2 Prediction Creation:** Sends a request to Replicate’s API to start an AI image generation prediction.
- **1.3 Prediction Polling:** Periodically checks the status of the prediction until it completes.
- **1.4 Result Processing:** Handles the completed prediction output for downstream use or storage.

---

### 2. Block-by-Block Analysis

#### 1.1 Manual Trigger & API Key Setup

**Overview:**  
This block allows a user to manually start the workflow and securely assign the Replicate API key needed for authentication in subsequent API calls.

**Nodes Involved:**  
- On clicking 'execute'  
- Set API Key

**Node Details:**

- **On clicking 'execute'**  
  - *Type:* Manual Trigger  
  - *Role:* Entry point to start the workflow on demand.  
  - *Configuration:* No parameters; triggers the workflow when executed manually.  
  - *Input:* None  
  - *Output:* Triggers the next node to set API key.  
  - *Edge Cases:* None typical; manual start only.  

- **Set API Key**  
  - *Type:* Set  
  - *Role:* Assigns the Replicate API key as a string variable (`replicate_api_key`).  
  - *Configuration:* Stores the user’s API key in a variable named `replicate_api_key`. The user must replace `"YOUR_REPLICATE_API_KEY"` with their actual API key.  
  - *Input:* Trigger from manual execution.  
  - *Output:* Provides the API key variable to the next HTTP request node.  
  - *Edge Cases:* Missing or invalid API key will cause authentication failure downstream.  

---

#### 1.2 Prediction Creation

**Overview:**  
This block sends a POST request to Replicate’s API to create a new prediction job for image generation based on specified input parameters.

**Nodes Involved:**  
- Create Prediction  
- Extract Prediction ID

**Node Details:**

- **Create Prediction**  
  - *Type:* HTTP Request  
  - *Role:* Calls Replicate’s `/v1/predictions` endpoint to initiate a prediction.  
  - *Configuration:*  
    - Method: POST  
    - URL: `https://api.replicate.com/v1/predictions`  
    - Headers: Authorization with Bearer token derived from `replicate_api_key`, Content-Type `application/json`  
    - Body (JSON):  
      ```json
      {
        "version": "40267110a09dae0b3b9fcf3efcb31fea64e7edf9d3cfaf8d5bed342418de3453",
        "input": {
          "prompt": "prompt value",
          "seed": 1,
          "width": 1,
          "height": 1,
          "lora_scale": 1
        }
      }
      ```
    - *Note:* The prompt and other input parameters are currently static placeholders and should be replaced or parameterized for real use.  
    - Timeout: 60 seconds  
  - *Input:* Receives API key variable from Set API Key node.  
  - *Output:* JSON response from Replicate containing prediction ID and initial status.  
  - *Edge Cases:*  
    - API key invalid or missing → 401 Unauthorized  
    - Timeout or network failure → request failure  
    - Invalid input parameters → API error response  

- **Extract Prediction ID**  
  - *Type:* Code  
  - *Role:* Parses the prediction response to extract `id`, initial `status`, and constructs the prediction URL for polling.  
  - *Configuration:* JavaScript code executed once per item:  
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
  - *Input:* Prediction creation response JSON.  
  - *Output:* Object containing prediction ID, status, and URL for status checking.  
  - *Edge Cases:* Missing `id` or malformed response → code failure or invalid URL for polling.  

---

#### 1.3 Prediction Polling

**Overview:**  
This block implements a polling loop that waits for 2 seconds between each check and calls the Replicate API to query the prediction status until the job succeeds.

**Nodes Involved:**  
- Wait  
- Check Prediction Status  
- Check If Complete

**Node Details:**

- **Wait**  
  - *Type:* Wait  
  - *Role:* Pauses workflow execution for 2 seconds before the next status check.  
  - *Configuration:* Wait time: 2 seconds  
  - *Input:* Triggered after extracting prediction ID or from "Check If Complete" when prediction is incomplete.  
  - *Output:* Triggers the next status check HTTP request.  
  - *Edge Cases:* None typical; delay length can affect responsiveness or API rate limits.  

- **Check Prediction Status**  
  - *Type:* HTTP Request  
  - *Role:* Performs a GET request to the stored prediction URL to retrieve the current prediction status.  
  - *Configuration:*  
    - URL: Dynamic from `predictionUrl` variable  
    - Headers: Authorization Bearer with `replicate_api_key`  
    - Method: GET (default)  
  - *Input:* Triggered after wait node or initially after prediction ID extraction.  
  - *Output:* JSON response with the current prediction status.  
  - *Edge Cases:*  
    - API key invalid → 401 Unauthorized  
    - Network failure or timeout  
    - Prediction ID not found → 404 error  

- **Check If Complete**  
  - *Type:* If  
  - *Role:* Checks if the prediction status equals "succeeded" to decide whether to proceed or continue polling.  
  - *Configuration:* Condition: `status == "succeeded"`  
  - *Input:* Status JSON from Check Prediction Status node.  
  - *Output:*  
    - True branch: Prediction complete → forward to processing node.  
    - False branch: Prediction incomplete → loop back to Wait node.  
  - *Edge Cases:*  
    - Other statuses like "failed" or "canceled" are not explicitly handled, which may cause infinite polling or logical errors.  

---

#### 1.4 Result Processing

**Overview:**  
This block processes the completed prediction output, extracting key metadata and output URLs for further use or storage.

**Nodes Involved:**  
- Process Result

**Node Details:**

- **Process Result**  
  - *Type:* Code  
  - *Role:* Extracts relevant fields from the prediction JSON and formats them in a simplified object.  
  - *Configuration:* JavaScript code:  
    ```javascript
    const result = $input.item.json;

    return {
      status: result.status,
      output: result.output,
      metrics: result.metrics,
      created_at: result.created_at,
      completed_at: result.completed_at,
      model: 'settyan/flash-v2.0.2-beta.10',
      other_url: result.output
    };
    ```  
  - *Input:* JSON from the completed prediction status response.  
  - *Output:* Structured object containing status, output URLs, timing metadata, and model identifier.  
  - *Edge Cases:*  
    - Missing output data or unexpected structure could cause undefined fields.  
    - No error handling for failed prediction statuses.

---

### 3. Summary Table

| Node Name              | Node Type          | Functional Role                      | Input Node(s)              | Output Node(s)               | Sticky Note                                                                                                                                                              |
|------------------------|--------------------|------------------------------------|----------------------------|-----------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| On clicking 'execute'  | Manual Trigger     | Starts workflow manually            | —                          | Set API Key                 |                                                                                                                                                                        |
| Set API Key            | Set                | Stores Replicate API key            | On clicking 'execute'       | Create Prediction           |                                                                                                                                                                        |
| Create Prediction      | HTTP Request       | Creates prediction request on Replicate | Set API Key                | Extract Prediction ID       |                                                                                                                                                                        |
| Extract Prediction ID  | Code               | Extracts prediction ID and status  | Create Prediction           | Wait                        |                                                                                                                                                                        |
| Wait                   | Wait               | Waits 2 seconds before status check | Extract Prediction ID, Check If Complete (false branch) | Check Prediction Status      |                                                                                                                                                                        |
| Check Prediction Status| HTTP Request       | Queries prediction status           | Wait                       | Check If Complete           |                                                                                                                                                                        |
| Check If Complete      | If                 | Checks if prediction succeeded     | Check Prediction Status     | Process Result (true), Wait (false) |                                                                                                                                                                        |
| Process Result         | Code               | Processes and formats completed prediction output | Check If Complete (true) | —                           |                                                                                                                                                                        |
| Sticky Note            | Sticky Note        | Documentation note about workflow  | —                          | —                           | ## Settyan Flash V2.0.2 Beta.10 AI Generator  This workflow uses the **settyan/flash-v2.0.2-beta.10** model from Replicate to generate other content.  Setup: 1. Add your Replicate API key 2. Configure the input parameters 3. Run the workflow  Model Details: - Type: Other Generation - Provider: settyan - Required Fields: prompt |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - No configuration needed. This node will start the workflow manually.

2. **Create Set Node "Set API Key"**  
   - Type: Set  
   - Add a string variable named `replicate_api_key`.  
   - Set its value to your actual Replicate API key (replace `"YOUR_REPLICATE_API_KEY"`).  
   - Connect the Manual Trigger node's output to this node's input.

3. **Create HTTP Request Node "Create Prediction"**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.replicate.com/v1/predictions`  
   - Authentication: None (use HTTP header auth via header parameters)  
   - Headers:  
     - `Authorization: Bearer {{$node["Set API Key"].json["replicate_api_key"]}}`  
     - `Content-Type: application/json`  
   - Body Content Type: JSON  
   - Body (JSON):  
     ```json
     {
       "version": "40267110a09dae0b3b9fcf3efcb31fea64e7edf9d3cfaf8d5bed342418de3453",
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
   - Connect the "Set API Key" node output to this node input.  

4. **Create Code Node "Extract Prediction ID"**  
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
   - Connect the output of "Create Prediction" to this node input.

5. **Create Wait Node "Wait"**  
   - Type: Wait  
   - Wait for: 2 seconds  
   - Connect the output of "Extract Prediction ID" to this node input.  

6. **Create HTTP Request Node "Check Prediction Status"**  
   - Type: HTTP Request  
   - Method: GET (default)  
   - URL: `{{$json["predictionUrl"]}}`  
   - Headers:  
     - `Authorization: Bearer {{$node["Set API Key"].json["replicate_api_key"]}}`  
   - Connect the output of "Wait" to this node input.

7. **Create If Node "Check If Complete"**  
   - Type: If  
   - Condition: Boolean  
     - Value 1: `{{$json["status"]}}`  
     - Operation: Equals  
     - Value 2: `succeeded`  
   - Connect the output of "Check Prediction Status" to this node input.

8. **Create Code Node "Process Result"**  
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
       model: 'settyan/flash-v2.0.2-beta.10',
       other_url: result.output
     };
     ```  
   - Connect the "true" output of "Check If Complete" node to this node input.

9. **Loop Back from "Check If Complete" False Branch to "Wait"**  
   - Connect the "false" output (prediction not yet succeeded) back to the "Wait" node input to continue polling.

10. **Add a Sticky Note**  
    - Add a sticky note with content:  
      ```
      ## Settyan Flash V2.0.2 Beta.10 AI Generator

      This workflow uses the **settyan/flash-v2.0.2-beta.10** model from Replicate to generate other content.

      ### Setup
      1. Add your Replicate API key
      2. Configure the input parameters
      3. Run the workflow

      ### Model Details
      - Type: Other Generation
      - Provider: settyan
      - Required Fields: prompt
      ```

---

### 5. General Notes & Resources

| Note Content                                                                                                      | Context or Link                                                                                         |
|------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| This workflow relies on the Replicate API and requires a valid API key. Replace `"YOUR_REPLICATE_API_KEY"` accordingly. | Replicate API Documentation: https://replicate.com/docs/api-reference/predictions/create            |
| The polling mechanism currently only checks for status `succeeded`. Other statuses like `failed` or `canceled` are not handled explicitly and may cause indefinite loops. | Consider adding error handling or timeout to improve robustness.                                      |
| Input parameters in the "Create Prediction" node (`prompt`, `seed`, `width`, `height`, `lora_scale`) are placeholders and should be parameterized or dynamically set for practical use. | User customization required for meaningful image generation.                                          |
| The model version ID: `40267110a09dae0b3b9fcf3efcb31fea64e7edf9d3cfaf8d5bed342418de3453` corresponds to **settyan/flash-v2.0.2-beta.10** on Replicate. | Model details and updates available on Replicate’s platform.                                         |

---

**Disclaimer:**  
The text provided derives exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies with no illegal, offensive, or protected content. All data handled is legal and publicly accessible.