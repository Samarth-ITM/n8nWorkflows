Generate Text with IBM Granite Speech 3.3 8B Model via Replicate

https://n8nworkflows.xyz/workflows/generate-text-with-ibm-granite-speech-3-3-8b-model-via-replicate-6884


# Generate Text with IBM Granite Speech 3.3 8B Model via Replicate

### 1. Workflow Overview

This workflow automates text generation using the **IBM Granite Speech 3.3 8B** model via the Replicate API. It is designed to trigger manually, send a text generation request with predefined parameters to the model hosted on Replicate, poll the prediction status until completion, and then extract and process the generated text result.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Manual trigger to start the workflow.
- **1.2 API Key Setup:** Setting the Replicate API key for authentication.
- **1.3 Prediction Creation:** Sending a request to Replicate to create a text generation prediction.
- **1.4 Prediction Monitoring:** Polling the prediction status at intervals until it completes.
- **1.5 Result Processing:** Extracting and formatting the final output from the completed prediction.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Starts the workflow execution manually by the user.
- **Nodes Involved:**  
  - On clicking 'execute'
- **Node Details:**

| Node Name            | Details                                                                                                                       |
|----------------------|-------------------------------------------------------------------------------------------------------------------------------|
| On clicking 'execute' | - Type: Manual Trigger<br>- Role: User manually initiates the workflow<br>- Configuration: No parameters<br>- Inputs: None<br>- Outputs: Starts API Key Setup node<br>- Edge Cases: None, unless workflow is not active<br>- Version: 1 |

---

#### 2.2 API Key Setup

- **Overview:** Sets the Replicate API key in the workflow context to be used for authenticated API calls.
- **Nodes Involved:**  
  - Set API Key
- **Node Details:**

| Node Name   | Details                                                                                                                               |
|-------------|-----------------------------------------------------------------------------------------------------------------------------------------|
| Set API Key | - Type: Set node<br>- Role: Stores the Replicate API key as a workflow variable<br>- Configuration: assigns string variable `replicate_api_key` with placeholder `"YOUR_REPLICATE_API_KEY"` (to be replaced by actual key)<br>- Inputs: from manual trigger<br>- Outputs: to Create Prediction node<br>- Edge Cases: Missing or invalid key will cause authentication failure downstream<br>- Version: 3.3 |

---

#### 2.3 Prediction Creation

- **Overview:** Sends an HTTP POST request to Replicate API to create a new prediction task with specified input parameters.
- **Nodes Involved:**  
  - Create Prediction
  - Extract Prediction ID
- **Node Details:**

| Node Name          | Details                                                                                                                                                                                                                                                    |
|--------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Create Prediction   | - Type: HTTP Request<br>- Role: Calls Replicate API to create prediction for text generation<br>- Configuration: POST to `https://api.replicate.com/v1/predictions` with JSON body specifying model version and input parameters (seed=1, top_k=50, top_p=0.9, max_tokens=512, temperature=0.6)<br>- Headers: Authorization Bearer token dynamically set from `Set API Key` node, Content-Type application/json<br>- Inputs: from Set API Key node<br>- Outputs: JSON with prediction ID and initial status<br>- Timeout: 60 seconds<br>- Edge Cases: API errors, invalid parameters, authentication failure, timeout<br>- Version: 4.2 |
| Extract Prediction ID | - Type: Code (JavaScript)<br>- Role: Extracts prediction ID and initial status from API response, constructs URL for status polling<br>- Configuration: Runs once per item, outputs `predictionId`, `status`, and `predictionUrl`<br>- Inputs: from Create Prediction<br>- Outputs: to Wait node<br>- Edge Cases: Missing or malformed response, undefined fields<br>- Version: 2 |

---

#### 2.4 Prediction Monitoring

- **Overview:** Polls the Replicate prediction endpoint after a wait period, repeatedly checking if the prediction is completed.
- **Nodes Involved:**  
  - Wait  
  - Check Prediction Status  
  - Check If Complete
- **Node Details:**

| Node Name             | Details                                                                                                                                                                                              |
|-----------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Wait                  | - Type: Wait<br>- Role: Pauses workflow for 2 seconds to allow prediction progress before polling status<br>- Configuration: 2 seconds delay<br>- Inputs: from Extract Prediction ID or Check If Complete (false branch)<br>- Outputs: to Check Prediction Status<br>- Edge Cases: None, but too short wait may increase API calls; too long delays slow workflow<br>- Version: 1 |
| Check Prediction Status | - Type: HTTP Request<br>- Role: Calls Replicate API to get current prediction status<br>- Configuration: GET request to `predictionUrl` from previous node, with Authorization header using stored API key<br>- Inputs: from Wait node<br>- Outputs: to Check If Complete<br>- Edge Cases: API errors, network failures, auth failures, JSON parsing errors<br>- Version: 4.2 |
| Check If Complete      | - Type: If node<br>- Role: Evaluates if prediction status equals "succeeded"<br>- Configuration: Checks boolean condition `$json.status === 'succeeded'`<br>- Inputs: from Check Prediction Status<br>- Outputs: true branch to Process Result, false branch back to Wait for polling<br>- Edge Cases: Unexpected status values, infinite loops if prediction stuck<br>- Version: 1 |

---

#### 2.5 Result Processing

- **Overview:** Processes the final prediction output once completed, extracting relevant details for further use.
- **Nodes Involved:**  
  - Process Result
- **Node Details:**

| Node Name      | Details                                                                                                                                                                                                                      |
|----------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Process Result | - Type: Code (JavaScript)<br>- Role: Parses completed prediction JSON, extracts output URL and metadata, formats result for downstream use<br>- Configuration: Runs once per item, returns object with status, output URL, metrics, timestamps, and model name<br>- Inputs: from Check If Complete (true branch)<br>- Outputs: final processed data<br>- Edge Cases: Missing output data, inconsistent formats, API changes<br>- Version: 2 |

---

#### Additional Node: Sticky Note

- **Overview:** Provides descriptive information and instructions for the workflow.
- **Nodes Involved:**  
  - Sticky Note
- **Node Details:**

| Node Name   | Details                                                                                                                                                                                                                                     |
|-------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Sticky Note | - Type: Sticky Note<br>- Role: Documentation within workflow<br>- Content: Title, model description, setup instructions (add API key, configure inputs, run workflow), and model details (type, provider, required fields)<br>- Inputs/Outputs: None<br>- Version: 1 |

---

### 3. Summary Table

| Node Name              | Node Type          | Functional Role                          | Input Node(s)                     | Output Node(s)               | Sticky Note                                                                                               |
|------------------------|--------------------|----------------------------------------|----------------------------------|-----------------------------|----------------------------------------------------------------------------------------------------------|
| On clicking 'execute'   | Manual Trigger     | Workflow entry point                    | None                             | Set API Key                 |                                                                                                          |
| Set API Key            | Set                | Stores Replicate API key                | On clicking 'execute'             | Create Prediction           |                                                                                                          |
| Create Prediction      | HTTP Request       | Sends prediction creation request      | Set API Key                     | Extract Prediction ID       |                                                                                                          |
| Extract Prediction ID  | Code               | Extracts prediction ID and status      | Create Prediction               | Wait                       |                                                                                                          |
| Wait                   | Wait               | Waits before polling prediction status | Extract Prediction ID, Check If Complete (false branch) | Check Prediction Status    |                                                                                                          |
| Check Prediction Status| HTTP Request       | Retrieves current prediction status    | Wait                           | Check If Complete           |                                                                                                          |
| Check If Complete      | If                 | Checks if prediction succeeded         | Check Prediction Status         | Process Result (true), Wait (false) |                                                                                                          |
| Process Result         | Code               | Processes final prediction output      | Check If Complete (true)        | None                       |                                                                                                          |
| Sticky Note            | Sticky Note        | Workflow documentation and instructions| None                          | None                       | ## Ibm Granite Granite Speech 3.3 8b Text Generator\n\nThis workflow uses the **ibm-granite/granite-speech-3.3-8b** model from Replicate to generate text content.\n\n### Setup\n1. Add your Replicate API key\n2. Configure the input parameters\n3. Run the workflow\n\n### Model Details\n- **Type**: Text Generation\n- **Provider**: ibm-granite\n- **Required Fields**: None |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: `On clicking 'execute'`  
   - Type: Manual Trigger (no parameters)  
   - Position: Starting node  

2. **Add Set Node for API Key**  
   - Name: `Set API Key`  
   - Type: Set  
   - Parameters: Add assignment variable `replicate_api_key` (string) with value `"YOUR_REPLICATE_API_KEY"` (replace with your actual Replicate API key)  
   - Connect output from `On clicking 'execute'` to this node  

3. **Create HTTP Request Node to Create Prediction**  
   - Name: `Create Prediction`  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.replicate.com/v1/predictions`  
   - Authentication: Generic Credential Type → HTTP Header Auth  
   - Headers:  
     - `Authorization`: `Bearer {{$node["Set API Key"].json["replicate_api_key"]}}`  
     - `Content-Type`: `application/json`  
   - Body Content Type: JSON  
   - Body:  
     ```json
     {
       "version": "688e7a943167401c310f0975cb68f1a35e0bddc3b65f60bde89c37860e07edf1",
       "input": {
         "seed": 1,
         "top_k": 50,
         "top_p": 0.9,
         "max_tokens": 512,
         "temperature": 0.6
       }
     }
     ```  
   - Timeout: 60 seconds  
   - Connect output from `Set API Key` to this node  

4. **Add Code Node to Extract Prediction ID**  
   - Name: `Extract Prediction ID`  
   - Type: Code  
   - Language: JavaScript  
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
   - Connect output from `Create Prediction` to this node  

5. **Add Wait Node to Delay Polling**  
   - Name: `Wait`  
   - Type: Wait  
   - Parameters: Wait for 2 seconds  
   - Connect output from `Extract Prediction ID` to this node  

6. **Add HTTP Request Node to Check Prediction Status**  
   - Name: `Check Prediction Status`  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `{{$json["predictionUrl"]}}` (dynamic URL from previous node’s output)  
   - Authentication: Generic Credential Type → HTTP Header Auth  
   - Headers:  
     - `Authorization`: `Bearer {{$node["Set API Key"].json["replicate_api_key"]}}`  
   - Connect output from `Wait` to this node  

7. **Add If Node to Check If Prediction Is Complete**  
   - Name: `Check If Complete`  
   - Type: If  
   - Condition: Boolean →  
     - Value 1: `{{$json["status"]}}`  
     - Operation: Equal  
     - Value 2: `succeeded`  
   - Connect output from `Check Prediction Status` to this node  

8. **Add Code Node to Process Result**  
   - Name: `Process Result`  
   - Type: Code  
   - Language: JavaScript  
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
       model: 'ibm-granite/granite-speech-3.3-8b',
       text_url: result.output
     };
     ```  
   - Connect true branch output of `Check If Complete` to this node  

9. **Connect False Branch of `Check If Complete` Back to `Wait`**  
   - To continue polling until completion  

10. **Optionally, Add Sticky Note Node for Documentation**  
    - Name: `Sticky Note`  
    - Content:  
      ```
      ## Ibm Granite Granite Speech 3.3 8b Text Generator

      This workflow uses the **ibm-granite/granite-speech-3.3-8b** model from Replicate to generate text content.

      ### Setup
      1. Add your Replicate API key
      2. Configure the input parameters
      3. Run the workflow

      ### Model Details
      - **Type**: Text Generation
      - **Provider**: ibm-granite
      - **Required Fields**: None
      ```  
    - Place anywhere convenient, no inputs or outputs  

---

### 5. General Notes & Resources

| Note Content                                                                                               | Context or Link                                                                                                                                                      |
|------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| This workflow requires a valid Replicate API key with access to the ibm-granite/granite-speech-3.3-8b model. | Replace `"YOUR_REPLICATE_API_KEY"` string in the Set node with your actual API key.                                                                                 |
| Polling interval is set to 2 seconds to balance responsiveness and API usage limits.                        | Adjust the `Wait` node duration if needed depending on API rate limits or model latency.                                                                            |
| Model version string `"688e7a943167401c310f0975cb68f1a35e0bddc3b65f60bde89c37860e07edf1"` is specific to this model version on Replicate. | Ensure it remains current; update if Replicate releases new versions or your use case requires a different model version.                                           |
| For more details on Replicate API usage and authentication: https://replicate.com/docs/reference/http | Official documentation for API endpoints, authentication, and error handling.                                                                                       |

---

**Disclaimer:**  
The text provided derives exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.