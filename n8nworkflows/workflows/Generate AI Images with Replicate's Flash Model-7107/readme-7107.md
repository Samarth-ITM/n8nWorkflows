Generate AI Images with Replicate's Flash Model

https://n8nworkflows.xyz/workflows/generate-ai-images-with-replicate-s-flash-model-7107


# Generate AI Images with Replicate's Flash Model

### 1. Workflow Overview

This workflow, titled **Settyan Flash V2.0.0 Beta.4 AI Generator**, is designed to generate AI images or other content using the **settyan/flash-v2.0.0-beta.4** model hosted on the Replicate platform. The workflow orchestrates the process of submitting a prompt to the Replicate API, polling for the prediction status until completion, and then processing and returning the generated results.

The workflow is logically divided into the following blocks:

- **1.1 Input & Initialization:** Receives manual execution trigger and sets up necessary API credentials.
- **1.2 Prediction Creation:** Sends a generation request to Replicate with configured input parameters.
- **1.3 Prediction Monitoring:** Extracts prediction ID, waits, and repeatedly checks prediction status until the generation is complete.
- **1.4 Result Processing:** Once the prediction is complete, processes and formats the output data for downstream use.

---

### 2. Block-by-Block Analysis

#### 1.1 Input & Initialization

- **Overview:**  
This block starts the workflow by manually triggering execution and sets the Replicate API key needed for authorization in subsequent API requests.

- **Nodes Involved:**  
  - On clicking 'execute'  
  - Set API Key  

- **Node Details:**

  - **On clicking 'execute'**  
    - **Type:** Manual Trigger  
    - **Role:** Serves as the entry point; triggers the workflow manually to start the generation process.  
    - **Config:** No parameters, simply triggers execution when clicked.  
    - **Input/Output:** No inputs; output to Set API Key node.  
    - **Failure Cases:** None expected; manual trigger.  

  - **Set API Key**  
    - **Type:** Set  
    - **Role:** Assigns the Replicate API key as a workflow variable for authentication.  
    - **Config:** Sets a string variable `replicate_api_key` with placeholder value `"YOUR_REPLICATE_API_KEY"`. User must replace with a valid API key.  
    - **Expressions:** None; static assignment.  
    - **Input:** From manual trigger.  
    - **Output:** To Create Prediction node.  
    - **Failure Cases:** Missing or invalid API key will cause subsequent authentication errors.

---

#### 1.2 Prediction Creation

- **Overview:**  
This block sends a POST request to the Replicate API to create a new prediction (image generation) using the specified model version and input parameters including prompt, seed, width, height, and lora_scale.

- **Nodes Involved:**  
  - Create Prediction  

- **Node Details:**

  - **Create Prediction**  
    - **Type:** HTTP Request  
    - **Role:** Makes an authenticated POST request to Replicate’s prediction endpoint to initiate generation.  
    - **Config:**  
      - URL: `https://api.replicate.com/v1/predictions`  
      - Method: POST  
      - Timeout: 60 seconds  
      - Body (JSON):  
        ```json
        {
          "version": "58e464402115a10317935ba6fe6c1dd379d42b6761badec818c3259e719b224a",
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
        - Authorization: `Bearer <replicate_api_key>` (dynamically from Set API Key node)  
        - Content-Type: application/json  
      - Authentication: Generic HTTP Header Auth with Bearer token.  
    - **Expressions:** Uses expression to build authorization header from `replicate_api_key`.  
    - **Input:** From Set API Key node.  
    - **Output:** Response JSON containing prediction ID and status to Extract Prediction ID node.  
    - **Failure Cases:**  
      - HTTP errors (e.g., 401 Unauthorized if API key invalid)  
      - Timeout if API is slow or unresponsive  
      - Invalid input parameters may cause API errors  

---

#### 1.3 Prediction Monitoring

- **Overview:**  
This block handles polling the Replicate API to monitor the status of the prediction until it is complete (status "succeeded"). It extracts the prediction ID from the creation response, waits between polls, checks status, and decides whether to wait more or proceed to processing.

- **Nodes Involved:**  
  - Extract Prediction ID  
  - Wait  
  - Check Prediction Status  
  - Check If Complete  

- **Node Details:**

  - **Extract Prediction ID**  
    - **Type:** Code  
    - **Role:** Extracts the `id` and `status` fields from the prediction creation response and constructs the prediction status URL for polling.  
    - **Config:** JavaScript code runs once per item:  
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
    - **Input:** From Create Prediction node.  
    - **Output:** JSON with `predictionId`, `status`, and `predictionUrl` to Wait node.  
    - **Failure Cases:** Missing or malformed response may cause code failure.

  - **Wait**  
    - **Type:** Wait  
    - **Role:** Pauses execution for 2 seconds between polling cycles to avoid excessive API requests.  
    - **Config:** Wait 2 seconds.  
    - **Input:** From Extract Prediction ID or from Check If Complete (loop).  
    - **Output:** To Check Prediction Status node.  
    - **Failure Cases:** None expected.

  - **Check Prediction Status**  
    - **Type:** HTTP Request  
    - **Role:** Sends a GET request to the prediction URL to check current status of the prediction.  
    - **Config:**  
      - URL: dynamic from `predictionUrl` field in JSON.  
      - Headers: Authorization with Bearer token from Set API Key node.  
      - Method: GET (default).  
      - No timeout specified (default).  
    - **Input:** From Wait node.  
    - **Output:** JSON response with current prediction status to Check If Complete node.  
    - **Failure Cases:**  
      - HTTP errors or timeout  
      - Invalid URL or expired prediction ID  

  - **Check If Complete**  
    - **Type:** If  
    - **Role:** Branches workflow based on whether prediction status is `"succeeded"`.  
    - **Config:** Condition checks if `$json.status === "succeeded"`.  
    - **Input:** From Check Prediction Status node.  
    - **Output:**  
      - True branch to Process Result node (prediction complete).  
      - False branch loops back to Wait node (continue polling).  
    - **Failure Cases:** Unexpected status values may cause infinite loop or premature exit.

---

#### 1.4 Result Processing

- **Overview:**  
Once the prediction is complete, this block processes the result JSON to extract relevant details and prepare a structured output including status, output URLs, metrics, timestamps, and model info.

- **Nodes Involved:**  
  - Process Result  

- **Node Details:**

  - **Process Result**  
    - **Type:** Code  
    - **Role:** Parses the final prediction JSON and returns a simplified object with key data fields.  
    - **Config:** JavaScript code runs once per item:  
      ```js
      const result = $input.item.json;
      return {
        status: result.status,
        output: result.output,
        metrics: result.metrics,
        created_at: result.created_at,
        completed_at: result.completed_at,
        model: 'settyan/flash-v2.0.0-beta.4',
        other_url: result.output
      };
      ```  
    - **Input:** From Check If Complete node (true branch).  
    - **Output:** Final processed JSON for downstream use or logging.  
    - **Failure Cases:** Missing or malformed fields in output; empty output array.

---

#### Additional Node: Sticky Note

- **Purpose:** Provides user instructions and model details.  
- **Content Summary:**  
  - Workflow name and model used  
  - Setup instructions for API key and inputs  
  - Model metadata: type, provider, required fields (prompt)  

---

### 3. Summary Table

| Node Name              | Node Type          | Functional Role                      | Input Node(s)               | Output Node(s)            | Sticky Note                                                                                  |
|------------------------|--------------------|------------------------------------|-----------------------------|---------------------------|----------------------------------------------------------------------------------------------|
| On clicking 'execute'   | Manual Trigger     | Start workflow manually             | —                           | Set API Key                | See overall workflow instructions and model details in Sticky Note node                      |
| Set API Key            | Set                | Assign Replicate API key            | On clicking 'execute'        | Create Prediction          | See overall workflow instructions and model details in Sticky Note node                      |
| Create Prediction      | HTTP Request       | Submit generation request to Replicate API | Set API Key                | Extract Prediction ID      | See overall workflow instructions and model details in Sticky Note node                      |
| Extract Prediction ID  | Code               | Extract prediction ID and status    | Create Prediction            | Wait                      | See overall workflow instructions and model details in Sticky Note node                      |
| Wait                   | Wait               | Pause between polling requests      | Extract Prediction ID, Check If Complete (false branch) | Check Prediction Status | See overall workflow instructions and model details in Sticky Note node                      |
| Check Prediction Status| HTTP Request       | Poll prediction status              | Wait                        | Check If Complete          | See overall workflow instructions and model details in Sticky Note node                      |
| Check If Complete      | If                 | Branch on prediction completion     | Check Prediction Status      | Process Result, Wait       | See overall workflow instructions and model details in Sticky Note node                      |
| Process Result         | Code               | Process and format final results    | Check If Complete (true)     | —                         | See overall workflow instructions and model details in Sticky Note node                      |
| Sticky Note            | Sticky Note        | User instructions and model info   | —                           | —                         | ## Settyan Flash V2.0.0 Beta.4 AI Generator; Instructions on setup and model details         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: `On clicking 'execute'`  
   - Type: Manual Trigger  
   - No configuration needed.

2. **Create Set Node for API Key**  
   - Name: `Set API Key`  
   - Type: Set  
   - Add assignment:  
     - Variable name: `replicate_api_key`  
     - Value: `"YOUR_REPLICATE_API_KEY"` (replace with your actual Replicate API key)  
   - Connect output of Manual Trigger to this node.

3. **Create HTTP Request Node to Create Prediction**  
   - Name: `Create Prediction`  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.replicate.com/v1/predictions`  
   - Authentication: Generic HTTP Header Auth  
     - Header name: `Authorization`  
     - Header value: `={{ 'Bearer ' + $('Set API Key').item.json.replicate_api_key }}`  
   - Headers: Add `Content-Type: application/json`  
   - Body Content Type: JSON  
   - JSON Body:  
     ```json
     {
       "version": "58e464402115a10317935ba6fe6c1dd379d42b6761badec818c3259e719b224a",
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
   - Connect output of Set API Key to this node.

4. **Create Code Node to Extract Prediction ID**  
   - Name: `Extract Prediction ID`  
   - Type: Code  
   - Language: JavaScript  
   - Mode: Run once per item  
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
   - Connect output of Create Prediction to this node.

5. **Create Wait Node**  
   - Name: `Wait`  
   - Type: Wait  
   - Parameters: Wait for 2 seconds  
   - Connect output of Extract Prediction ID to this node.

6. **Create HTTP Request Node to Check Prediction Status**  
   - Name: `Check Prediction Status`  
   - Type: HTTP Request  
   - Method: GET (default)  
   - URL: `={{ $json.predictionUrl }}` (dynamic from JSON input)  
   - Authentication: Generic HTTP Header Auth  
     - Header name: `Authorization`  
     - Header value: `={{ 'Bearer ' + $('Set API Key').item.json.replicate_api_key }}`  
   - Connect output of Wait node to this node.

7. **Create If Node to Check Completion**  
   - Name: `Check If Complete`  
   - Type: If  
   - Condition: Boolean  
     - Expression: `{{$json.status}}` equals `"succeeded"`  
   - Connect output of Check Prediction Status to this node.

8. **Create Code Node to Process Result**  
   - Name: `Process Result`  
   - Type: Code  
   - Language: JavaScript  
   - Mode: Run once per item  
   - Code:  
     ```js
     const result = $input.item.json;
     return {
       status: result.status,
       output: result.output,
       metrics: result.metrics,
       created_at: result.created_at,
       completed_at: result.completed_at,
       model: 'settyan/flash-v2.0.0-beta.4',
       other_url: result.output
     };
     ```  
   - Connect **true** output of Check If Complete to this node.

9. **Complete the Polling Loop**  
   - Connect **false** output of Check If Complete back to the Wait node to continue polling.

10. **Add Sticky Note for Documentation (Optional)**  
    - Content:  
      ```
      ## Settyan Flash V2.0.0 Beta.4 AI Generator

      This workflow uses the **settyan/flash-v2.0.0-beta.4** model from Replicate to generate other content.

      ### Setup
      1. Add your Replicate API key
      2. Configure the input parameters
      3. Run the workflow

      ### Model Details
      - **Type**: Other Generation
      - **Provider**: settyan
      - **Required Fields**: prompt
      ```
    - Position near the start of the workflow for easy reference.

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                                                                                                           |
|-------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------|
| The workflow requires a valid Replicate API key. Obtain one at https://replicate.com/account/api | User must replace the placeholder `YOUR_REPLICATE_API_KEY` with their actual API key to authenticate requests to Replicate. |
| Model version used: `58e464402115a10317935ba6fe6c1dd379d42b6761badec818c3259e719b224a`            | This is the specific version ID of the **settyan/flash-v2.0.0-beta.4** model on Replicate, ensuring consistent results.    |
| Input parameters such as `prompt`, `seed`, `width`, `height`, and `lora_scale` can be adjusted   | Adjust these in the Create Prediction HTTP Request node JSON body to customize generation outputs.                         |
| Polling interval is 2 seconds to balance responsiveness and API rate limits                      | Can be increased if API rate limits are tight or decreased if faster response is needed with risk of hitting limits.       |

---

**Disclaimer:** The text provided comes exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.