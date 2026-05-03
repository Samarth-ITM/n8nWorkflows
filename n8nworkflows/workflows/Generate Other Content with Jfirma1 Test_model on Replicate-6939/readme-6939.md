Generate Other Content with Jfirma1 Test_model on Replicate

https://n8nworkflows.xyz/workflows/generate-other-content-with-jfirma1-test_model-on-replicate-6939


# Generate Other Content with Jfirma1 Test_model on Replicate

### 1. Workflow Overview

This workflow, titled **Jfirma1 Test_model AI Generator**, is designed to interact with the Replicate API to generate content using the AI model **jfirma1/test_model**. It is targeted at users who want to automate content generation tasks by submitting prompts to the model and retrieving generated results asynchronously.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Manual trigger to start the workflow execution.
- **1.2 API Key Configuration:** Setting up the Replicate API key for authentication.
- **1.3 Prediction Creation:** Sending a prediction request to the Replicate API with user-defined parameters.
- **1.4 Prediction Tracking:** Extracting the prediction ID and polling the prediction status until completion.
- **1.5 Result Processing:** Upon successful completion, processing and formatting the output from the prediction.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block initiates the workflow manually. It acts as the entry point for the execution.

- **Nodes Involved:**  
  - On clicking 'execute'

- **Node Details:**  
  - **On clicking 'execute'**  
    - Type: Manual Trigger  
    - Configuration: Default manual trigger with no parameters; starts the workflow on user command.  
    - Inputs: None (manual start)  
    - Outputs: Connects to "Set API Key" node.  
    - Potential Failures: None (manual execution)  
    - Notes: Serves as a clean entry point to run the workflow on demand.

#### 1.2 API Key Configuration

- **Overview:**  
  This block sets the Replicate API key in a variable to be used for authentication in subsequent HTTP requests.

- **Nodes Involved:**  
  - Set API Key

- **Node Details:**  
  - **Set API Key**  
    - Type: Set  
    - Configuration: Assigns a string named `replicate_api_key` with the placeholder `"YOUR_REPLICATE_API_KEY"`.  
    - Inputs: Receives trigger from Manual Trigger node.  
    - Outputs: Feeds the API key to the "Create Prediction" HTTP Request node.  
    - Key Expressions: None; static assignment.  
    - Edge Cases: Missing or invalid API key will cause authentication errors downstream.  
    - Notes: User must replace placeholder with their actual Replicate API key before running.

#### 1.3 Prediction Creation

- **Overview:**  
  This block sends a POST request to Replicate’s API to create a new prediction job based on the input prompt and parameters.

- **Nodes Involved:**  
  - Create Prediction

- **Node Details:**  
  - **Create Prediction**  
    - Type: HTTP Request  
    - Configuration:  
      - Method: POST  
      - URL: `https://api.replicate.com/v1/predictions`  
      - Headers:  
        - Authorization: Bearer token from `replicate_api_key` variable.  
        - Content-Type: application/json  
      - Body (JSON):  
        ```json
        {
          "version": "591b4311afe2900e6927e2b9802496dffd78fe8baa8f0dbb6e4961172fd3bb1d",
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
    - Inputs: Receives API key from "Set API Key" node.  
    - Outputs: Sends response JSON to "Extract Prediction ID" node.  
    - Expressions: Authorization header uses expression `={{ 'Bearer ' + $('Set API Key').item.json.replicate_api_key }}`.  
    - Edge Cases:  
      - API key invalid or missing → 401 Unauthorized.  
      - Incorrect JSON format or missing required fields → 400 Bad Request.  
      - Network timeout if API is unresponsive.  
    - Notes: The `prompt` value is static in this configuration and should be parameterized for dynamic usage.

#### 1.4 Prediction Tracking

- **Overview:**  
  This block extracts the prediction ID from the creation response, waits briefly, then polls the prediction status repeatedly until it is complete.

- **Nodes Involved:**  
  - Extract Prediction ID  
  - Wait  
  - Check Prediction Status  
  - Check If Complete

- **Node Details:**  
  - **Extract Prediction ID**  
    - Type: Code  
    - Configuration: JavaScript code extracts the prediction ID and status from the API response and constructs the polling URL.  
    - Code Summary:  
      - Extracts `id` and `status` from JSON.  
      - Returns `predictionId`, `status`, and `predictionUrl` (for polling).  
    - Inputs: Receives JSON response from "Create Prediction".  
    - Outputs: Feeds data to "Wait" node.  
    - Edge Cases: Missing or malformed response JSON will cause code errors.  
  - **Wait**  
    - Type: Wait  
    - Configuration: Waits for 2 seconds before next action.  
    - Inputs: Receives from "Extract Prediction ID" or "Check If Complete" node (if status incomplete).  
    - Outputs: Sends to "Check Prediction Status".  
    - Edge Cases: None significant.  
  - **Check Prediction Status**  
    - Type: HTTP Request  
    - Configuration:  
      - Method: GET (default)  
      - URL: Uses expression to poll URL from `$json.predictionUrl`.  
      - Headers: Authorization with Bearer token.  
    - Inputs: Receives from "Wait" node.  
    - Outputs: Sends JSON response to "Check If Complete".  
    - Edge Cases:  
      - API key invalid → 401 Unauthorized.  
      - Network issues or rate limiting.  
  - **Check If Complete**  
    - Type: If  
    - Configuration: Checks if `$json.status` equals `"succeeded"`.  
    - Inputs: Receives JSON status from "Check Prediction Status".  
    - Outputs:  
      - If true: proceeds to "Process Result".  
      - If false: loops back to "Wait" node to poll again.  
    - Edge Cases:  
      - Status may be `"failed"`, `"canceled"`, or other values not handled explicitly (could cause indefinite looping).  
      - No explicit timeout or maximum retries; risk of infinite polling.

#### 1.5 Result Processing

- **Overview:**  
  Once the prediction is complete, this block processes the final result, extracting key metadata and output URLs.

- **Nodes Involved:**  
  - Process Result

- **Node Details:**  
  - **Process Result**  
    - Type: Code  
    - Configuration: JavaScript code that formats the final prediction result into a simplified JSON object.  
    - Code Summary:  
      - Extracts `status`, `output`, `metrics`, `created_at`, `completed_at` from the response.  
      - Adds a static `model` identifier: `'jfirma1/test_model'`.  
      - Sets `other_url` as the same as `output`.  
    - Inputs: Receives JSON from "Check If Complete" node when status is `"succeeded"`.  
    - Outputs: Final structured data for downstream use or export.  
    - Edge Cases: If output data is missing or malformed, processing may fail silently.

---

### 3. Summary Table

| Node Name             | Node Type            | Functional Role                    | Input Node(s)           | Output Node(s)           | Sticky Note                                                                                                      |
|-----------------------|----------------------|----------------------------------|------------------------|--------------------------|-----------------------------------------------------------------------------------------------------------------|
| On clicking 'execute'  | Manual Trigger       | Entry point for workflow execution | None                   | Set API Key              |                                                                                                                 |
| Set API Key           | Set                  | Sets Replicate API key for auth  | On clicking 'execute'  | Create Prediction        |                                                                                                                 |
| Create Prediction     | HTTP Request         | Creates a prediction on Replicate| Set API Key            | Extract Prediction ID    |                                                                                                                 |
| Extract Prediction ID | Code                 | Extracts prediction ID and constructs polling URL | Create Prediction     | Wait                     |                                                                                                                 |
| Wait                  | Wait                 | Waits 2 seconds before polling   | Extract Prediction ID, Check If Complete (false) | Check Prediction Status |                                                                                                                 |
| Check Prediction Status| HTTP Request        | Polls prediction status from API | Wait                   | Check If Complete        |                                                                                                                 |
| Check If Complete     | If                   | Checks if prediction succeeded   | Check Prediction Status| Process Result (true), Wait (false) |                                                                                                                 |
| Process Result        | Code                 | Processes and formats final result| Check If Complete (true)| None                     |                                                                                                                 |
| Sticky Note           | Sticky Note          | Documentation note about workflow| None                   | None                     | ## Jfirma1 Test_model AI Generator  This workflow uses the **jfirma1/test_model** model from Replicate to generate other content.  Setup 1. Add your Replicate API key 2. Configure the input parameters 3. Run the workflow  Model Details - **Type**: Other Generation - **Provider**: jfirma1 - **Required Fields**: prompt |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node**  
   - Name: `On clicking 'execute'`  
   - No special settings required.

2. **Create a Set node**  
   - Name: `Set API Key`  
   - Add a string field named `replicate_api_key` with value `"YOUR_REPLICATE_API_KEY"` (replace with your actual API key).  
   - Connect output from `On clicking 'execute'` to this node.

3. **Create an HTTP Request node**  
   - Name: `Create Prediction`  
   - Method: POST  
   - URL: `https://api.replicate.com/v1/predictions`  
   - Authentication: None in node settings; use "Generic Credential" with HTTP Header Auth.  
   - Headers:  
     - `Authorization`: Set expression to `={{ 'Bearer ' + $('Set API Key').item.json.replicate_api_key }}`  
     - `Content-Type`: `application/json`  
   - Body Content Type: JSON  
   - Body JSON:  
     ```json
     {
       "version": "591b4311afe2900e6927e2b9802496dffd78fe8baa8f0dbb6e4961172fd3bb1d",
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
   - Connect output from `Set API Key` to this node.

4. **Create a Code node**  
   - Name: `Extract Prediction ID`  
   - Mode: Run once for each item  
   - JavaScript code:  
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
   - Connect output from `Create Prediction` to this node.

5. **Create a Wait node**  
   - Name: `Wait`  
   - Wait time: 2 seconds  
   - Connect output from `Extract Prediction ID` to this node.

6. **Create another HTTP Request node**  
   - Name: `Check Prediction Status`  
   - Method: GET (default)  
   - URL: Use expression `={{ $json.predictionUrl }}` to poll prediction status.  
   - Authentication: Generic Credential with HTTP Header Auth.  
   - Header:  
     - `Authorization`: `={{ 'Bearer ' + $('Set API Key').item.json.replicate_api_key }}`  
   - Connect output from `Wait` node to this node.

7. **Create an If node**  
   - Name: `Check If Complete`  
   - Condition: Boolean  
     - `{{$json.status}}` equals `"succeeded"`  
   - Connect output from `Check Prediction Status` to this node.

8. **Create a Code node**  
   - Name: `Process Result`  
   - Mode: Run once for each item  
   - JavaScript code:  
     ```javascript
     const result = $input.item.json;

     return {
       status: result.status,
       output: result.output,
       metrics: result.metrics,
       created_at: result.created_at,
       completed_at: result.completed_at,
       model: 'jfirma1/test_model',
       other_url: result.output
     };
     ```  
   - Connect the "true" output of `Check If Complete` node to this node.

9. **Create a connection from the "false" output of `Check If Complete` back to the `Wait` node**  
   - This forms the polling loop while the prediction is not completed.

10. **Optional: Add a Sticky Note**  
    - Content:  
      ```
      ## Jfirma1 Test_model AI Generator

      This workflow uses the **jfirma1/test_model** model from Replicate to generate other content.

      ### Setup
      1. Add your Replicate API key
      2. Configure the input parameters
      3. Run the workflow

      ### Model Details
      - **Type**: Other Generation
      - **Provider**: jfirma1
      - **Required Fields**: prompt
      ```

**Credentials Setup:**  
- Create a generic credential with HTTP Header Authentication type.  
- No username/password needed, only the header key `Authorization` with value `Bearer YOUR_REPLICATE_API_KEY` set dynamically in each HTTP request via expressions.

**Parameterization Note:**  
- The `prompt` and other input parameters in the `Create Prediction` node should be made dynamic by replacing the static `"prompt value"` with expressions or input data for practical use.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                   | Context or Link                                |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------|
| This workflow uses the Replicate API to asynchronously generate content using the jfirma1/test_model model. Ensure you have an active Replicate account and valid API key. | Project context                               |
| The workflow currently lacks error handling for failed or canceled prediction statuses; consider adding maximum retry limits or failure branches for robustness.             | Improvement suggestion                        |
| For more about Replicate API and model versions, visit: https://replicate.com/docs/api-reference/predictions                                                                | Official API documentation                    |
| To avoid infinite polling, enhance the "Check If Complete" node with additional conditions or a timeout counter.                                                             | Best practice in asynchronous polling loops  |

---

**Disclaimer:** The text provided here is exclusively derived from an automated workflow created with n8n integration and automation tool. The processing strictly respects current content policies and contains no illegal, offensive, or protected material. All data handled is legal and public.