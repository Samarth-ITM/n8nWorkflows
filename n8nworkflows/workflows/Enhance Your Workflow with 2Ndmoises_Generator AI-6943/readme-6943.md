Enhance Your Workflow with 2Ndmoises_Generator AI

https://n8nworkflows.xyz/workflows/enhance-your-workflow-with-2ndmoises_generator-ai-6943


# Enhance Your Workflow with 2Ndmoises_Generator AI

### 1. Workflow Overview

This workflow leverages the **moicarmonas/2ndmoises_generator** AI model hosted on Replicate to generate content based on a user-provided prompt. It is designed for users who want to automate the generation of AI-based outputs by seamlessly integrating with Replicate’s prediction API.

The workflow is structured into the following logical blocks:

- **1.1 Input Reception and Initialization**: Manual trigger to start the workflow and setting the API key for Replicate authentication.
- **1.2 Prediction Creation**: Submitting a generation request to Replicate with the specified parameters.
- **1.3 Prediction Monitoring**: Polling the Replicate API to check the status of the prediction until it completes.
- **1.4 Result Processing**: Extracting and formatting the output once the prediction succeeds.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Initialization

**Overview:**  
This block starts the workflow manually and sets up the necessary API key for authenticating with the Replicate API.

**Nodes Involved:**  
- On clicking 'execute'  
- Set API Key

**Node Details:**

- **On clicking 'execute'**  
  - Type: Manual Trigger  
  - Role: Entry point to manually start the workflow.  
  - Configuration: No parameters; triggers workflow execution when clicked.  
  - Input: None  
  - Output: Triggers the "Set API Key" node.  
  - Edge Cases: None.

- **Set API Key**  
  - Type: Set Node  
  - Role: Assigns the Replicate API key to a workflow variable for use in HTTP requests.  
  - Configuration: A string assignment named `replicate_api_key` with a placeholder value `"YOUR_REPLICATE_API_KEY"`.  
  - Key Expressions: None; static assignment.  
  - Input: Trigger from manual start node.  
  - Output: Passes the API key to the "Create Prediction" node.  
  - Edge Cases: Failure if the key is invalid or missing; user must replace placeholder with a valid key.

---

#### 1.2 Prediction Creation

**Overview:**  
Submits a POST request to the Replicate API to create a new prediction job based on input parameters such as prompt, seed, width, height, and lora_scale.

**Nodes Involved:**  
- Create Prediction  
- Extract Prediction ID

**Node Details:**

- **Create Prediction**  
  - Type: HTTP Request  
  - Role: Sends a POST request to Replicate’s `/v1/predictions` endpoint to start generation.  
  - Configuration:  
    - URL: `https://api.replicate.com/v1/predictions`  
    - Method: POST  
    - Body: JSON containing model version `"7ae928ae3e6f1614129b139f34503ff4e91e922e7487658a47ad600bca16d5b0"` and input parameters (`prompt`, `seed`, `width`, `height`, `lora_scale`).  
    - Headers: Authorization Bearer token using the API key from "Set API Key", and `Content-Type: application/json`.  
    - Timeout: 60 seconds.  
  - Key Expressions:  
    - Authorization header: `"Bearer " + $('Set API Key').item.json.replicate_api_key` (dynamic).  
    - The input prompt is a static string `"prompt value"` in the JSON body but can be parameterized externally.  
  - Input: API key from "Set API Key".  
  - Output: JSON response with prediction ID and status.  
  - Edge Cases:  
    - HTTP errors, e.g., 401 Unauthorized if API key is invalid.  
    - Timeout after 60 seconds.  
    - API limits or malformed input.

- **Extract Prediction ID**  
  - Type: Code Node (JavaScript)  
  - Role: Parses the API response to extract the prediction ID and initial status; prepares URL for polling.  
  - Configuration:  
    - Runs once per item.  
    - JavaScript extracts `id` and `status` from input JSON, constructs a polling URL.  
  - Key Expressions:  
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
  - Input: JSON output from "Create Prediction".  
  - Output: Object with prediction ID, status, and URL for status checking.  
  - Edge Cases: If response lacks required fields or malformed JSON.

---

#### 1.3 Prediction Monitoring

**Overview:**  
Repeatedly polls Replicate’s API to check the prediction status, waiting between attempts until the prediction is complete.

**Nodes Involved:**  
- Wait  
- Check Prediction Status  
- Check If Complete

**Node Details:**

- **Wait**  
  - Type: Wait Node  
  - Role: Pauses workflow execution for 2 seconds before polling again, preventing API spamming.  
  - Configuration: Wait for 2 seconds.  
  - Input: From "Extract Prediction ID" initially and from "Check If Complete" if prediction is not done.  
  - Output: Triggers "Check Prediction Status".  
  - Edge Cases: None, but excessive wait time increases total workflow duration.

- **Check Prediction Status**  
  - Type: HTTP Request  
  - Role: Sends GET request to the Replicate prediction URL to retrieve current status.  
  - Configuration:  
    - URL: Dynamic, from `predictionUrl` field using expression `{{$json.predictionUrl}}`  
    - Headers: Authorization Bearer token with API key.  
  - Input: From "Wait".  
  - Output: JSON with updated prediction details.  
  - Edge Cases:  
    - HTTP errors (e.g., 404 if prediction ID invalid).  
    - API rate limits.

- **Check If Complete**  
  - Type: If Node  
  - Role: Evaluates if the prediction status equals `"succeeded"`.  
  - Configuration: Boolean condition comparing `$json.status` to `"succeeded"`.  
  - Input: From "Check Prediction Status".  
  - Output:  
    - True branch: proceeds to "Process Result".  
    - False branch: loops back to "Wait" to retry polling.  
  - Edge Cases:  
    - Status other than `"succeeded"` or `"failed"` may cause indefinite loops.  
    - No explicit handling for failure states.

---

#### 1.4 Result Processing

**Overview:**  
Processes the final prediction result after successful completion, extracting relevant information for downstream use.

**Nodes Involved:**  
- Process Result

**Node Details:**

- **Process Result**  
  - Type: Code Node (JavaScript)  
  - Role: Extracts and formats prediction output details such as status, output URL, metrics, and timestamps.  
  - Configuration:  
    - Run once per item.  
    - Returns an object with fields: status, output, metrics, creation/completion timestamps, model name, and other_url (same as output).  
  - Key Expressions:  
    ```js
    const result = $input.item.json;
    return {
      status: result.status,
      output: result.output,
      metrics: result.metrics,
      created_at: result.created_at,
      completed_at: result.completed_at,
      model: 'moicarmonas/2ndmoises_generator',
      other_url: result.output
    };
    ```
  - Input: Prediction data from "Check If Complete" true branch.  
  - Output: Structured final data for further use or export.  
  - Edge Cases: If output is empty or unexpected format.

---

### 3. Summary Table

| Node Name               | Node Type          | Functional Role                      | Input Node(s)           | Output Node(s)           | Sticky Note                                                                                                  |
|-------------------------|--------------------|------------------------------------|------------------------|--------------------------|--------------------------------------------------------------------------------------------------------------|
| On clicking 'execute'    | Manual Trigger     | Start the workflow manually         | None                   | Set API Key              |                                                                                                              |
| Set API Key             | Set                | Store Replicate API key              | On clicking 'execute'  | Create Prediction        |                                                                                                              |
| Create Prediction       | HTTP Request       | Submit generation request to Replicate | Set API Key             | Extract Prediction ID    |                                                                                                              |
| Extract Prediction ID   | Code               | Extract prediction ID and prepare polling URL | Create Prediction       | Wait                     |                                                                                                              |
| Wait                    | Wait               | Wait 2 seconds before polling       | Extract Prediction ID, Check If Complete (False branch) | Check Prediction Status |                                                                                                              |
| Check Prediction Status | HTTP Request       | Poll Replicate API for prediction status | Wait                    | Check If Complete        |                                                                                                              |
| Check If Complete       | If                 | Evaluate if prediction succeeded    | Check Prediction Status | Process Result (True), Wait (False) |                                                                                                              |
| Process Result          | Code               | Format and extract prediction output | Check If Complete (True) | None                    |                                                                                                              |
| Sticky Note             | Sticky Note        | Documentation and instructions      | None                   | None                     | ## Moicarmonas 2ndmoises_generator AI Generator<br><br>This workflow uses the **moicarmonas/2ndmoises_generator** model from Replicate to generate other content.<br><br>### Setup<br>1. Add your Replicate API key<br>2. Configure the input parameters<br>3. Run the workflow<br><br>### Model Details<br>- **Type**: Other Generation<br>- **Provider**: moicarmonas<br>- **Required Fields**: prompt |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a **Manual Trigger** node named `"On clicking 'execute'"`.  
   - No parameters needed. This will manually start the workflow.

2. **Create Set Node for API Key**  
   - Add a **Set** node named `"Set API Key"`.  
   - In the node, create one field assignment:  
     - Name: `replicate_api_key`  
     - Type: String  
     - Value: `"YOUR_REPLICATE_API_KEY"` (replace with your actual Replicate API key).  
   - Connect `"On clicking 'execute'"` output to this node’s input.

3. **Create HTTP Request Node for Prediction Creation**  
   - Add an **HTTP Request** node named `"Create Prediction"`.  
   - Configure as follows:  
     - HTTP Method: POST  
     - URL: `https://api.replicate.com/v1/predictions`  
     - Authentication: Generic Credential with HTTP Header Auth.  
       - Header Name: `Authorization`  
       - Header Value: `Bearer {{$node["Set API Key"].json["replicate_api_key"]}}`  
     - Additional Header: `Content-Type: application/json`  
     - Request Body Type: JSON  
     - Request Body (JSON):  
       ```json
       {
         "version": "7ae928ae3e6f1614129b139f34503ff4e91e922e7487658a47ad600bca16d5b0",
         "input": {
           "prompt": "prompt value",
           "seed": 1,
           "width": 1,
           "height": 1,
           "lora_scale": 1
         }
       }
       ```  
     - Timeout: 60000 ms (60 seconds).  
   - Connect `"Set API Key"` output to this node.

4. **Create Code Node to Extract Prediction ID**  
   - Add a **Code** node named `"Extract Prediction ID"`.  
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
   - Connect `"Create Prediction"` output to this node.

5. **Create Wait Node**  
   - Add a **Wait** node named `"Wait"`.  
   - Set to wait for 2 seconds.  
   - Connect `"Extract Prediction ID"` output to this node.

6. **Create HTTP Request Node to Check Prediction Status**  
   - Add an **HTTP Request** node named `"Check Prediction Status"`.  
   - Configure as:  
     - HTTP Method: GET  
     - URL: Use expression: `{{$json["predictionUrl"]}}`  
     - Authentication: Generic Credential HTTP Header Auth with the same API key header as before.  
       - Header Name: `Authorization`  
       - Header Value: `Bearer {{$node["Set API Key"].json["replicate_api_key"]}}`  
   - Connect `"Wait"` output to this node.

7. **Create If Node to Check Completion**  
   - Add an **If** node named `"Check If Complete"`.  
   - Condition: Boolean  
     - Value 1: `{{$json["status"]}}`  
     - Operation: equals  
     - Value 2: `succeeded`  
   - Connect `"Check Prediction Status"` output to this node.

8. **Create Code Node to Process Result**  
   - Add a **Code** node named `"Process Result"`.  
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
       model: 'moicarmonas/2ndmoises_generator',
       other_url: result.output
     };
     ```  
   - Connect `"Check If Complete"` true output to this node.

9. **Loop Back for Polling**  
   - Connect `"Check If Complete"` false output back to `"Wait"` node to continue polling until completion.

10. **(Optional) Add Sticky Note**  
    - Add a **Sticky Note** node near the start to document the workflow purpose and setup instructions as per the original.  
    - Content example:  
      ```
      ## Moicarmonas 2ndmoises_generator AI Generator

      This workflow uses the moicarmonas/2ndmoises_generator model from Replicate to generate other content.

      Setup:
      1. Add your Replicate API key
      2. Configure the input parameters
      3. Run the workflow

      Model Details:
      - Type: Other Generation
      - Provider: moicarmonas
      - Required Fields: prompt
      ```

---

### 5. General Notes & Resources

| Note Content                                                                                                                         | Context or Link                                              |
|-------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------|
| This workflow requires a valid Replicate API key. Replace `"YOUR_REPLICATE_API_KEY"` with your actual key for authentication.     | Setup requirement                                            |
| The model version used is `"7ae928ae3e6f1614129b139f34503ff4e91e922e7487658a47ad600bca16d5b0"`, representing moicarmonas/2ndmoises_generator. | Model version identifier                                     |
| Input parameters like prompt, seed, width, height, and lora_scale can be adjusted for different generation results.                | Customization options                                        |
| Polling interval is fixed at 2 seconds to balance between timely updates and API rate limits.                                        | Performance consideration                                   |
| The workflow does not explicitly handle prediction failures or errors beyond polling for "succeeded" status. Consider adding error handling for robustness. | Recommended enhancement                                     |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with prevailing content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.