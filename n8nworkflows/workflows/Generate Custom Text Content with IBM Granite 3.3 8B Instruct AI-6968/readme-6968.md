Generate Custom Text Content with IBM Granite 3.3 8B Instruct AI

https://n8nworkflows.xyz/workflows/generate-custom-text-content-with-ibm-granite-3-3-8b-instruct-ai-6968


# Generate Custom Text Content with IBM Granite 3.3 8B Instruct AI

### 1. Workflow Overview

This workflow, titled **"Ibm Granite Granite 3.3 8b Instruct Text Generator"**, automates the process of generating custom text content using the IBM Granite 3.3 8B Instruct AI model via the Replicate API. It is designed primarily for users who want to programmatically generate text outputs based on configurable input parameters and monitor the prediction progress until completion.

The workflow is logically divided into the following blocks:

- **1.1 Manual Trigger and API Key Setup:** Starts the workflow manually and assigns the Replicate API key.
- **1.2 Prediction Request Creation:** Sends a request to Replicate API to initiate text generation with IBM Granite 3.3 8B Instruct model.
- **1.3 Prediction ID Extraction and Polling Setup:** Extracts the prediction ID and prepares to poll for the prediction status.
- **1.4 Polling and Status Checking:** Periodically checks the prediction status until the result is ready.
- **1.5 Result Processing:** Once completed, extracts and formats the generated text and metadata for output.

---

### 2. Block-by-Block Analysis

#### 2.1 Manual Trigger and API Key Setup

- **Overview:**  
  This block initiates the workflow manually and sets the required API key for authentication with Replicate API.

- **Nodes Involved:**  
  - On clicking 'execute'  
  - Set API Key

- **Node Details:**

  - **On clicking 'execute'**  
    - Type: Manual Trigger  
    - Role: Entry point to start the workflow manually.  
    - Configuration: No parameters; triggers workflow execution on user command.  
    - Input: None  
    - Output: Triggers "Set API Key" node.  
    - Edge Cases: None specific; user must trigger execution.  

  - **Set API Key**  
    - Type: Set  
    - Role: Stores the Replicate API key as a workflow variable for subsequent use.  
    - Configuration: Assigns a string value named `replicate_api_key` with placeholder `"YOUR_REPLICATE_API_KEY"` which must be replaced by the user.  
    - Input: Triggered by manual execution node.  
    - Output: Passes API key to next HTTP request node.  
    - Edge Cases: Missing or invalid API key will cause authentication failures downstream.

---

#### 2.2 Prediction Request Creation

- **Overview:**  
  Sends an HTTP POST request to Replicate's prediction API to create a new text generation prediction using the IBM Granite 3.3 8B Instruct model with specified input parameters.

- **Nodes Involved:**  
  - Create Prediction

- **Node Details:**

  - **Create Prediction**  
    - Type: HTTP Request  
    - Role: Calls Replicate API `/v1/predictions` endpoint to start text generation.  
    - Configuration:  
      - URL: `https://api.replicate.com/v1/predictions`  
      - Method: POST  
      - JSON Body includes:  
        - Model version: `"3ff9e6e20ff1f31263bf4f36c242bd9be1acb2025122daeefe2b06e883df0996"` (IBM Granite 3.3 8B Instruct)  
        - Input parameters: `seed=1`, `top_k=50`, `top_p=0.9`, `max_tokens=512`, `temperature=0.6`  
      - Headers: Authorization Bearer token using the API key set in previous node, and Content-Type `application/json`  
      - Timeout: 60 seconds  
    - Input: API key from "Set API Key" node  
    - Output: JSON response containing prediction ID and initial status  
    - Version: Uses HTTP Request node version 4.2  
    - Edge Cases:  
      - Authentication failure due to bad API key  
      - Timeout if API is slow or unresponsive  
      - Invalid input parameters causing API errors  

---

#### 2.3 Prediction ID Extraction and Polling Setup

- **Overview:**  
  Extracts the prediction ID and status from the API response to enable polling for the prediction result.

- **Nodes Involved:**  
  - Extract Prediction ID

- **Node Details:**

  - **Extract Prediction ID**  
    - Type: Code (JavaScript)  
    - Role: Parses the JSON response from the prediction creation to retrieve prediction ID, status, and constructs the URL for status polling.  
    - Configuration:  
      - Extracts `id` and `status` from the incoming JSON.  
      - Constructs `predictionUrl` as `https://api.replicate.com/v1/predictions/{predictionId}` for later polling.  
    - Input: JSON from "Create Prediction" node  
    - Output: An object with `predictionId`, `status`, and `predictionUrl` fields  
    - Edge Cases:  
      - Missing fields in API response  
      - Unexpected response structure causing code failure  
    - Version: Code Node v2  

---

#### 2.4 Polling and Status Checking

- **Overview:**  
  Periodically polls the prediction status URL to check if the text generation is complete, implementing a wait and retry loop.

- **Nodes Involved:**  
  - Wait  
  - Check Prediction Status  
  - Check If Complete

- **Node Details:**

  - **Wait**  
    - Type: Wait  
    - Role: Pauses the workflow for 2 seconds between polling attempts to avoid excessive API calls.  
    - Configuration: Delay of 2 seconds.  
    - Input: Prediction URL and status object from "Extract Prediction ID" or previous polling loop.  
    - Output: Triggers "Check Prediction Status" node.  
    - Edge Cases:  
      - Too short delay could lead to API rate limiting  
      - Too long delay increases overall workflow runtime  

  - **Check Prediction Status**  
    - Type: HTTP Request  
    - Role: Sends GET request to the prediction URL to retrieve current prediction status.  
    - Configuration:  
      - URL: Dynamic expression using `{{$json.predictionUrl}}`  
      - Headers: Authorization Bearer token with API key  
      - Method: GET (default)  
    - Input: Output of "Wait" node  
    - Output: JSON including updated prediction status and results if available  
    - Edge Cases:  
      - Network issues or API downtime  
      - Authorization errors  
      - Unexpected JSON structure  

  - **Check If Complete**  
    - Type: If  
    - Role: Conditional node that evaluates whether the prediction status equals `"succeeded"`.  
    - Configuration:  
      - Condition: Checks if `$json.status === "succeeded"`  
      - True branch: Proceeds to "Process Result" node  
      - False branch: Loops back to "Wait" node to retry polling  
    - Edge Cases:  
      - Status may enter error or failed states which are not handled explicitly here  
      - Infinite polling if prediction never succeeds or stalls  

---

#### 2.5 Result Processing

- **Overview:**  
  Processes the completed prediction's output, extracting key information and formatting for downstream use.

- **Nodes Involved:**  
  - Process Result

- **Node Details:**

  - **Process Result**  
    - Type: Code (JavaScript)  
    - Role: Extracts and organizes the prediction output, including status, generated text URL, metrics, timestamps, and model identifier.  
    - Configuration:  
      - Extracts fields: `status`, `output`, `metrics`, `created_at`, `completed_at` from input JSON  
      - Adds constant field: `model` set to `"ibm-granite/granite-3.3-8b-instruct"`  
      - Uses `output` field (text URL) for `text_url`  
    - Input: JSON from "Check If Complete" node on success branch  
    - Output: Processed, structured data about the prediction result  
    - Edge Cases:  
      - Missing or malformed output fields  
      - Output URL may be empty or invalid if prediction failed silently  

---

### 3. Summary Table

| Node Name               | Node Type        | Functional Role                          | Input Node(s)               | Output Node(s)                | Sticky Note                                                                                           |
|-------------------------|------------------|----------------------------------------|-----------------------------|------------------------------|-----------------------------------------------------------------------------------------------------|
| On clicking 'execute'    | Manual Trigger   | Starts workflow manually                | None                        | Set API Key                  |                                                                                                     |
| Set API Key             | Set              | Stores Replicate API key                 | On clicking 'execute'        | Create Prediction            |                                                                                                     |
| Create Prediction       | HTTP Request     | Sends prediction creation request       | Set API Key                 | Extract Prediction ID        |                                                                                                     |
| Extract Prediction ID   | Code             | Extracts prediction ID and status        | Create Prediction           | Wait                        |                                                                                                     |
| Wait                    | Wait             | Delays workflow for polling interval    | Extract Prediction ID / Check If Complete (False branch) | Check Prediction Status     |                                                                                                     |
| Check Prediction Status | HTTP Request     | Polls prediction status                   | Wait                        | Check If Complete            |                                                                                                     |
| Check If Complete       | If               | Checks if prediction succeeded           | Check Prediction Status     | Process Result / Wait        |                                                                                                     |
| Process Result          | Code             | Processes and formats final prediction   | Check If Complete (True)    | None                        |                                                                                                     |
| Sticky Note             | Sticky Note      | Provides workflow overview and instructions | None                      | None                        | ## Ibm Granite Granite 3.3 8b Instruct Text Generator\n\nThis workflow uses the **ibm-granite/granite-3.3-8b-instruct** model from Replicate to generate text content.\n\n### Setup\n1. Add your Replicate API key\n2. Configure the input parameters\n3. Run the workflow\n\n### Model Details\n- **Type**: Text Generation\n- **Provider**: ibm-granite\n- **Required Fields**: None |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node**  
   - Name: `On clicking 'execute'`  
   - Purpose: To manually start the workflow execution.

2. **Add a Set Node to Store API Key**  
   - Name: `Set API Key`  
   - Add a string field assignment:  
     - Variable: `replicate_api_key`  
     - Value: `YOUR_REPLICATE_API_KEY` (replace this placeholder with your actual Replicate API key)  
   - Connect the output of `On clicking 'execute'` to this node.

3. **Add HTTP Request Node to Create Prediction**  
   - Name: `Create Prediction`  
   - HTTP Method: POST  
   - URL: `https://api.replicate.com/v1/predictions`  
   - Authentication: Generic HTTP Header Authentication  
     - Header Name: `Authorization`  
     - Header Value: Expression: `={{ 'Bearer ' + $('Set API Key').item.json.replicate_api_key }}`  
   - Headers: Add `Content-Type: application/json`  
   - Request Body (raw JSON):  
     ```json
     {
       "version": "3ff9e6e20ff1f31263bf4f36c242bd9be1acb2025122daeefe2b06e883df0996",
       "input": {
         "seed": 1,
         "top_k": 50,
         "top_p": 0.9,
         "max_tokens": 512,
         "temperature": 0.6
       }
     }
     ```  
   - Set "Send Body" and "Send Headers" to true.  
   - Timeout: 60 seconds.  
   - Connect output of `Set API Key` to this node.

4. **Add Code Node to Extract Prediction ID**  
   - Name: `Extract Prediction ID`  
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
   - Connect output of `Create Prediction` to this node.

5. **Add Wait Node**  
   - Name: `Wait`  
   - Wait Time: 2 seconds  
   - Connect output of `Extract Prediction ID` to this node.

6. **Add HTTP Request Node to Poll Prediction Status**  
   - Name: `Check Prediction Status`  
   - HTTP Method: GET (default)  
   - URL: Expression: `={{ $json.predictionUrl }}`  
   - Authentication: Generic HTTP Header Authentication  
     - Header Name: `Authorization`  
     - Header Value: Expression: `={{ 'Bearer ' + $('Set API Key').item.json.replicate_api_key }}`  
   - Connect output of `Wait` node to this node.

7. **Add If Node to Check Completion Status**  
   - Name: `Check If Complete`  
   - Condition: Boolean check if `$json.status` equals `"succeeded"`  
   - True branch: Connect to next processing node.  
   - False branch: Loop back to `Wait` node to continue polling.  
   - Connect output of `Check Prediction Status` node here.

8. **Add Code Node to Process Final Result**  
   - Name: `Process Result`  
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
       model: 'ibm-granite/granite-3.3-8b-instruct',
       text_url: result.output
     };
     ```  
   - Connect True branch of `Check If Complete` to this node.

9. **Add a Sticky Note (Optional)**  
   - Content includes overview and setup instructions for the workflow as described in the sticky note content.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                 | Context or Link                                            |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------|
| This workflow uses the **ibm-granite/granite-3.3-8b-instruct** model from Replicate to generate text content.                                                                                                 | Sticky Note content on workflow overview                    |
| Setup requires you to add your Replicate API key and configure input parameters before running.                                                                                                               | Sticky Note instructions                                    |
| Model details: Text Generation by ibm-granite, with no mandatory input fields besides those specified.                                                                                                        | Sticky Note model info                                      |
| Replicate API documentation for text generation models: https://replicate.com/docs/api-reference                                                                                                              | Official API documentation (external reference)             |
| Be aware of API rate limits and potential delays; consider error handling for API failures or unexpected statuses beyond "succeeded".                                                                         | General best practice note                                  |
| The delay between polling cycles is set to 2 seconds; adjust if you experience rate limiting or long wait times.                                                                                              | Polling strategy observation                                |

---

*Disclaimer: The text provided is extracted exclusively from an automated workflow created with n8n, an integration and automation tool. The process strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.*