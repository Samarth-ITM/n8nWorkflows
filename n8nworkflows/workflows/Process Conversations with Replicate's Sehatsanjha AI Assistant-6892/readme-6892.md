Process Conversations with Replicate's Sehatsanjha AI Assistant

https://n8nworkflows.xyz/workflows/process-conversations-with-replicate-s-sehatsanjha-ai-assistant-6892


# Process Conversations with Replicate's Sehatsanjha AI Assistant

### 1. Workflow Overview

This workflow titled **"Aihilums Sehatsanjha AI Generator"** automates the process of invoking the Replicate AI model **aihilums/sehatsanjha** to generate content. It is designed for users who want to generate AI-based outputs via Replicate's API without manual API handling. The workflow includes the following logical blocks:

- **1.1 Manual Trigger & API Key Setup:** Starts the workflow manually and sets the necessary Replicate API key.
- **1.2 Prediction Creation:** Sends a request to Replicate to create a new prediction using the specified AI model.
- **1.3 Prediction Monitoring:** Polls the Replicate API to check the status of the prediction until it is complete.
- **1.4 Result Processing:** Once the prediction succeeds, extracts and formats the output for further use or export.

The workflow is linear and cyclic during prediction status checks, ensuring asynchronous handling of AI model execution delays.

---

### 2. Block-by-Block Analysis

#### 2.1 Manual Trigger & API Key Setup

- **Overview:**  
  This initial block allows the user to start the workflow manually and sets the Replicate API key needed for authentication in subsequent API calls.

- **Nodes Involved:**  
  - On clicking 'execute'  
  - Set API Key

- **Node Details:**

  1. **On clicking 'execute'**  
     - *Type:* Manual Trigger  
     - *Role:* Entry point to manually start the workflow.  
     - *Configuration:* No parameters required; triggers workflow execution on user demand.  
     - *Connections:* Outputs to Set API Key node.  
     - *Edge Cases:* None — manual trigger unlikely to fail but requires user action.

  2. **Set API Key**  
     - *Type:* Set Node  
     - *Role:* Defines and stores the Replicate API key as a workflow variable (`replicate_api_key`).  
     - *Configuration:* A string variable named `replicate_api_key` with placeholder `"YOUR_REPLICATE_API_KEY"` which the user must replace with a valid API key.  
     - *Connections:* Inputs from manual trigger; outputs to Create Prediction node.  
     - *Edge Cases:* Missing or invalid API key will cause authorization failures in subsequent HTTP requests.

#### 2.2 Prediction Creation

- **Overview:**  
  This block sends a POST request to Replicate's prediction API endpoint to create a new AI prediction job using the specified model version.

- **Nodes Involved:**  
  - Create Prediction  
  - Extract Prediction ID

- **Node Details:**

  1. **Create Prediction**  
     - *Type:* HTTP Request  
     - *Role:* Calls Replicate API `POST /v1/predictions` to start AI content generation.  
     - *Configuration:*  
       - URL: `https://api.replicate.com/v1/predictions`  
       - Method: POST  
       - Body: JSON with fixed `"version"` ID `8d601ed5dfe3c91d7dacdc17a127497519acfdb4c4eefbec3db17b6d87734a69` (aihilums/sehatsanjha model version) and empty input object `{}` — user can customize inputs here if needed.  
       - Headers: Authorization Bearer token with API key from the Set API Key node; Content-Type `application/json`.  
       - Timeout: 60 seconds.  
     - *Connections:* Input from Set API Key; output to Extract Prediction ID.  
     - *Edge Cases:*  
       - HTTP errors (network issues, 401 Unauthorized if API key invalid).  
       - Timeout if API is slow.  
       - Input JSON must be valid; empty input may or may not be accepted by model.

  2. **Extract Prediction ID**  
     - *Type:* Code Node (JavaScript)  
     - *Role:* Parses the API response to extract the `prediction.id` and initial status, constructs the polling URL.  
     - *Configuration:* Runs once per item; outputs an object with:  
       - `predictionId` — prediction identifier string  
       - `status` — initial status (e.g., "starting")  
       - `predictionUrl` — full endpoint URL for polling status (`https://api.replicate.com/v1/predictions/{id}`)  
     - *Connections:* Input from Create Prediction; output to Wait node.  
     - *Edge Cases:*  
       - If the response JSON is malformed or missing `id`, the node will fail.  
       - Assumes API response format is stable.

#### 2.3 Prediction Monitoring

- **Overview:**  
  This block periodically checks the status of the prediction until it completes successfully.

- **Nodes Involved:**  
  - Wait  
  - Check Prediction Status  
  - Check If Complete

- **Node Details:**

  1. **Wait**  
     - *Type:* Wait Node  
     - *Role:* Delays execution for 2 seconds to avoid rate limiting or excessive API calls.  
     - *Configuration:* Wait for 2 seconds before next check.  
     - *Connections:* Input from Extract Prediction ID and from Check If Complete fallback; output to Check Prediction Status.  
     - *Edge Cases:* Minimal, but long-running workflows may accumulate delay.

  2. **Check Prediction Status**  
     - *Type:* HTTP Request  
     - *Role:* Sends GET request to the prediction URL to fetch current status and result.  
     - *Configuration:*  
       - URL dynamically set from `predictionUrl` variable from previous node.  
       - Method: GET (default).  
       - Headers: Authorization Bearer token using API key.  
     - *Connections:* Input from Wait; output to Check If Complete.  
     - *Edge Cases:* Network errors, expired prediction ID, or 401 Unauthorized if API key invalid.

  3. **Check If Complete**  
     - *Type:* If Node  
     - *Role:* Evaluates if prediction status is `"succeeded"`.  
     - *Configuration:* Boolean condition comparing `$json.status` to string `"succeeded"`.  
     - *Connections:*  
       - If true: to Process Result node.  
       - If false: loops back to Wait node to retry.  
     - *Edge Cases:*  
       - Other terminal states like `"failed"` or `"canceled"` are not explicitly handled and will cause indefinite looping.

#### 2.4 Result Processing

- **Overview:**  
  Once prediction is successful, this block processes the output data for downstream use.

- **Nodes Involved:**  
  - Process Result

- **Node Details:**

  1. **Process Result**  
     - *Type:* Code Node (JavaScript)  
     - *Role:* Extracts key details from the prediction output and formats them into a consistent object.  
     - *Configuration:*  
       - Extracts `status`, `output`, `metrics`, `created_at`, `completed_at` from the API response.  
       - Adds static `"model": "aihilums/sehatsanjha"` for identification.  
       - Adds `other_url` referencing the output URL for convenience.  
     - *Connections:* Input from Check If Complete (true branch).  
     - *Edge Cases:*  
       - If output is empty or missing, the node will still pass but downstream consumers must handle empty data.

---

### 3. Summary Table

| Node Name              | Node Type          | Functional Role                 | Input Node(s)             | Output Node(s)           | Sticky Note                                                                                                                        |
|------------------------|--------------------|--------------------------------|---------------------------|--------------------------|----------------------------------------------------------------------------------------------------------------------------------|
| On clicking 'execute'   | Manual Trigger     | Starts workflow manually        | —                         | Set API Key               |                                                                                                                                  |
| Set API Key            | Set                | Stores Replicate API key        | On clicking 'execute'      | Create Prediction         |                                                                                                                                  |
| Create Prediction       | HTTP Request       | Creates prediction via API      | Set API Key                | Extract Prediction ID     |                                                                                                                                  |
| Extract Prediction ID   | Code               | Parses prediction ID and status | Create Prediction          | Wait                     |                                                                                                                                  |
| Wait                   | Wait               | Delays for 2 seconds            | Extract Prediction ID, Check If Complete (false) | Check Prediction Status |                                                                                                                                  |
| Check Prediction Status | HTTP Request       | Polls prediction status         | Wait                      | Check If Complete         |                                                                                                                                  |
| Check If Complete       | If                 | Checks if prediction succeeded  | Check Prediction Status    | Process Result (true), Wait (false) |                                                                                                                                  |
| Process Result          | Code               | Extracts and formats final data | Check If Complete (true)   | —                        |                                                                                                                                  |
| Sticky Note             | Sticky Note        | Workflow description and setup | —                         | —                        | ## Aihilums Sehatsanjha AI Generator<br><br>This workflow uses the **aihilums/sehatsanjha** model from Replicate to generate other content.<br><br>### Setup<br>1. Add your Replicate API key<br>2. Configure the input parameters<br>3. Run the workflow<br><br>### Model Details<br>- **Type**: Other Generation<br>- **Provider**: aihilums<br>- **Required Fields**: None |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node**  
   - Name: `On clicking 'execute'`  
   - Purpose: To start the workflow on demand. No special configuration needed.

2. **Add a Set Node for API Key**  
   - Name: `Set API Key`  
   - Add a string field assignment:  
     - Variable name: `replicate_api_key`  
     - Value: `"YOUR_REPLICATE_API_KEY"` (replace with your actual Replicate API key)  
   - Connect `On clicking 'execute'` output to this node.

3. **Add an HTTP Request Node to Create Prediction**  
   - Name: `Create Prediction`  
   - Method: POST  
   - URL: `https://api.replicate.com/v1/predictions`  
   - Authentication: None (use generic HTTP header auth)  
   - Headers:  
     - Authorization: `Bearer {{$node["Set API Key"].json["replicate_api_key"]}}`  
     - Content-Type: `application/json`  
   - Body (JSON):  
     ```json
     {
       "version": "8d601ed5dfe3c91d7dacdc17a127497519acfdb4c4eefbec3db17b6d87734a69",
       "input": {}
     }
     ```  
   - Timeout: 60 seconds  
   - Connect `Set API Key` output to this node.

4. **Add a Code Node to Extract Prediction ID**  
   - Name: `Extract Prediction ID`  
   - Mode: Run once per item  
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
   - Connect `Create Prediction` output to this node.

5. **Add a Wait Node**  
   - Name: `Wait`  
   - Set to wait for 2 seconds  
   - Connect `Extract Prediction ID` output to this node.

6. **Add an HTTP Request Node to Check Prediction Status**  
   - Name: `Check Prediction Status`  
   - Method: GET  
   - URL: `{{$json["predictionUrl"]}}` (dynamic from previous node)  
   - Authentication: None (generic HTTP header auth)  
   - Headers:  
     - Authorization: `Bearer {{$node["Set API Key"].json["replicate_api_key"]}}`  
   - Connect `Wait` output to this node.

7. **Add an If Node to Check Completion Status**  
   - Name: `Check If Complete`  
   - Condition: Boolean  
     - Value 1: `{{$json["status"]}}`  
     - Operation: Equal  
     - Value 2: `"succeeded"`  
   - Connect `Check Prediction Status` output to this node.

8. **Add a Code Node to Process Result**  
   - Name: `Process Result`  
   - Mode: Run once per item  
   - JavaScript code:  
     ```javascript
     const result = $input.item.json;
     return {
       status: result.status,
       output: result.output,
       metrics: result.metrics,
       created_at: result.created_at,
       completed_at: result.completed_at,
       model: 'aihilums/sehatsanjha',
       other_url: result.output
     };
     ```  
   - Connect the **true** branch of `Check If Complete` to this node.

9. **Connect the false branch of `Check If Complete` back to `Wait` node**  
   - This creates a polling loop until the prediction completes.

10. **Verify Credentials and API Key**  
    - Make sure your Replicate API key is correctly set in the `Set API Key` node.  
    - No OAuth2 or other complex credentials are needed.

11. **Run the workflow manually via `On clicking 'execute'` node**

---

### 5. General Notes & Resources

| Note Content                                                                                                                | Context or Link                                      |
|-----------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| This workflow uses the Replicate AI model **aihilums/sehatsanjha** to generate content without requiring input parameters.   | Workflow description sticky note                     |
| Ensure to replace `"YOUR_REPLICATE_API_KEY"` with your actual API key from https://replicate.com/account/api-tokens          | Replicate API key setup                              |
| The workflow assumes model version `8d601ed5dfe3c91d7dacdc17a127497519acfdb4c4eefbec3db17b6d87734a69` is stable and active.   | Model version reference                              |
| Polling interval is set to 2 seconds to balance timely updates and API rate limits. Adjust `Wait` node if necessary.          | Performance tuning                                  |
| The workflow does not handle prediction failure or cancellation explicitly; these cases may cause infinite polling.          | Potential improvement area                           |

---

**Disclaimer:**  
The provided text originates exclusively from an automated n8n workflow. It complies strictly with current content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.