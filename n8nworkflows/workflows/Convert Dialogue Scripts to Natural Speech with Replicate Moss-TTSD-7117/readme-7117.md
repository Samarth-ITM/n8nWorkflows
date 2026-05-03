Convert Dialogue Scripts to Natural Speech with Replicate Moss-TTSD

https://n8nworkflows.xyz/workflows/convert-dialogue-scripts-to-natural-speech-with-replicate-moss-ttsd-7117


# Convert Dialogue Scripts to Natural Speech with Replicate Moss-TTSD

### 1. Workflow Overview

This workflow, titled **Dessix Moss Ttsd Text Generator**, automates the process of generating natural speech text outputs using the **dessix/moss-ttsd** model hosted on the Replicate platform. It is designed for converting dialogue scripts or text inputs into natural speech representations via AI text generation.

The workflow is structured into logical processing blocks as follows:

- **1.1 Input Trigger and API Key Setup:** Manual start point to initiate the workflow and set the Replicate API key.
- **1.2 Prediction Creation:** Sends a request to Replicate to start a new text generation prediction using the moss-ttsd model.
- **1.3 Prediction Monitoring:** Polls the Replicate API to check the status of the prediction until completion.
- **1.4 Result Processing:** Extracts and formats the prediction output once the generation is completed.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Trigger and API Key Setup

**Overview:**  
This block initiates the workflow manually and securely sets the API key required to authenticate requests to Replicate’s API.

**Nodes Involved:**  
- On clicking 'execute'  
- Set API Key

**Node Details:**

- **On clicking 'execute'**  
  - Type: Manual Trigger  
  - Role: Entry point for manual execution of the workflow.  
  - Configuration: No parameters, simply a user-initiated trigger.  
  - Inputs: None  
  - Outputs: Connects to "Set API Key" node  
  - Edge Cases: None, but user must manually trigger each run.

- **Set API Key**  
  - Type: Set Node  
  - Role: Stores the Replicate API key in a variable for downstream use.  
  - Configuration: Sets the string variable `replicate_api_key` to `"YOUR_REPLICATE_API_KEY"` (to be replaced by user).  
  - Inputs: Manual Trigger node output  
  - Outputs: Connects to "Create Prediction"  
  - Edge Cases: If the API key is missing or invalid, subsequent API calls will fail with authentication errors.

---

#### 1.2 Prediction Creation

**Overview:**  
This block sends an HTTP POST request to create a new text generation prediction on Replicate, specifying the model version and input parameters.

**Nodes Involved:**  
- Create Prediction  
- Extract Prediction ID

**Node Details:**

- **Create Prediction**  
  - Type: HTTP Request  
  - Role: Sends the prediction creation request to Replicate API endpoint (`https://api.replicate.com/v1/predictions`).  
  - Configuration:  
    - Method: POST  
    - Headers: Authorization bearer token from API key, Content-Type application/json  
    - Body JSON:  
      ```json
      {
        "version": "2026fb3e22b7faa6b667ad396b86462a1b85313d18a936b458455c44e9c9cc76",
        "input": {
          "seed": 42,
          "use_normalize": true
        }
      }
      ```  
    - Timeout: 60 seconds  
  - Inputs: From "Set API Key" node  
  - Outputs: Connects to "Extract Prediction ID"  
  - Edge Cases: HTTP timeouts, authentication errors, invalid JSON body, API rate limiting.

- **Extract Prediction ID**  
  - Type: Code (JavaScript)  
  - Role: Parses the response to extract the prediction ID, initial status, and constructs the status URL for polling.  
  - Configuration:  
    - Runs once per item  
    - Code extracts `id` and `status` from response JSON, formats prediction URL:  
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
  - Inputs: From "Create Prediction" node  
  - Outputs: Connects to "Wait" node  
  - Edge Cases: Missing or malformed response JSON, no prediction ID returned.

---

#### 1.3 Prediction Monitoring

**Overview:**  
This block implements polling by repeatedly checking the prediction status until it reaches completion.

**Nodes Involved:**  
- Wait  
- Check Prediction Status  
- Check If Complete

**Node Details:**

- **Wait**  
  - Type: Wait Node  
  - Role: Delays execution for 2 seconds between polling attempts to avoid hitting API rate limits.  
  - Configuration: Wait for 2 seconds  
  - Inputs: From "Extract Prediction ID" or "Check If Complete" (if not complete)  
  - Outputs: Connects to "Check Prediction Status"  
  - Edge Cases: Excessive waiting if prediction takes very long, potential workflow timeout if API is slow.

- **Check Prediction Status**  
  - Type: HTTP Request  
  - Role: Requests the current status of the prediction from the previously stored prediction URL.  
  - Configuration:  
    - Method: GET (default)  
    - URL: Expression `{{$json.predictionUrl}}` dynamically set from previous node  
    - Headers: Authorization bearer token from API key  
  - Inputs: From "Wait" node  
  - Outputs: Connects to "Check If Complete"  
  - Edge Cases: HTTP request failures, invalid URL, auth errors.

- **Check If Complete**  
  - Type: If Node  
  - Role: Checks if the prediction status equals `"succeeded"`.  
  - Configuration: Boolean condition comparing `$json.status` to `"succeeded"`  
  - Inputs: From "Check Prediction Status"  
  - Outputs:  
    - True branch: Connects to "Process Result"  
    - False branch: Connects back to "Wait" (to continue polling)  
  - Edge Cases: Status values other than `"succeeded"` or `"failed"` need consideration; no explicit failure path implemented.

---

#### 1.4 Result Processing

**Overview:**  
This block processes the completed prediction output, extracting relevant metadata and the generated text URL.

**Nodes Involved:**  
- Process Result

**Node Details:**

- **Process Result**  
  - Type: Code (JavaScript)  
  - Role: Formats the final output object to include status, output URL, metrics, timestamps, and model name.  
  - Configuration:  
    - Runs once per item  
    - Extracts and returns:  
      ```js
      const result = $input.item.json;
      return {
        status: result.status,
        output: result.output,
        metrics: result.metrics,
        created_at: result.created_at,
        completed_at: result.completed_at,
        model: 'dessix/moss-ttsd',
        text_url: result.output
      };
      ```  
  - Inputs: From "Check If Complete" (true branch)  
  - Outputs: None (end of workflow)  
  - Edge Cases: Missing fields in response, empty or invalid output URL.

---

### 3. Summary Table

| Node Name               | Node Type          | Functional Role                         | Input Node(s)               | Output Node(s)             | Sticky Note                                                                                                                                                                           |
|-------------------------|--------------------|---------------------------------------|-----------------------------|----------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| On clicking 'execute'    | Manual Trigger     | Workflow manual start trigger          | None                        | Set API Key                |                                                                                                                                                                                       |
| Set API Key             | Set                | Stores Replicate API key                | On clicking 'execute'        | Create Prediction          |                                                                                                                                                                                       |
| Create Prediction       | HTTP Request       | Sends prediction creation request      | Set API Key                 | Extract Prediction ID      |                                                                                                                                                                                       |
| Extract Prediction ID   | Code               | Extracts prediction ID and status      | Create Prediction           | Wait                      |                                                                                                                                                                                       |
| Wait                    | Wait               | Waits 2 seconds between polls           | Extract Prediction ID, Check If Complete (false) | Check Prediction Status    |                                                                                                                                                                                       |
| Check Prediction Status | HTTP Request       | Polls prediction status                 | Wait                       | Check If Complete          |                                                                                                                                                                                       |
| Check If Complete       | If                 | Checks if prediction is complete        | Check Prediction Status     | Process Result (true), Wait (false) |                                                                                                                                                                                       |
| Process Result          | Code               | Processes completed prediction output   | Check If Complete (true)    | None                      |                                                                                                                                                                                       |
| Sticky Note             | Sticky Note        | Documentation and setup instructions    | None                        | None                      | ## Dessix Moss Ttsd Text Generator<br><br>This workflow uses the **dessix/moss-ttsd** model from Replicate to generate text content.<br><br>### Setup<br>1. Add your Replicate API key<br>2. Configure the input parameters<br>3. Run the workflow<br><br>### Model Details<br>- **Type**: Text Generation<br>- **Provider**: dessix<br>- **Required Fields**: None |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a **Manual Trigger** node named `"On clicking 'execute'"`.  
   - No configuration needed.

2. **Create Set Node for API Key**  
   - Add a **Set** node named `"Set API Key"`.  
   - Add a string variable called `replicate_api_key` with the placeholder value `"YOUR_REPLICATE_API_KEY"`.  
   - Connect `"On clicking 'execute'"` output to this node input.

3. **Create HTTP Request Node to Create Prediction**  
   - Add an **HTTP Request** node named `"Create Prediction"`.  
   - Configure:  
     - Method: POST  
     - URL: `https://api.replicate.com/v1/predictions`  
     - Authentication: HTTP Header Auth  
     - Header Parameters:  
       - `Authorization`: `Bearer {{$node["Set API Key"].json["replicate_api_key"]}}`  
       - `Content-Type`: `application/json`  
     - Body Content Type: JSON  
     - JSON Body:  
       ```json
       {
         "version": "2026fb3e22b7faa6b667ad396b86462a1b85313d18a936b458455c44e9c9cc76",
         "input": {
           "seed": 42,
           "use_normalize": true
         }
       }
       ```  
     - Timeout: 60000 ms  
   - Connect `"Set API Key"` output to this node.

4. **Create Code Node to Extract Prediction ID**  
   - Add a **Code** node named `"Extract Prediction ID"`.  
   - Set to run once per item.  
   - Use this JavaScript code:  
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
   - Configure:  
     - URL: Expression `{{$json["predictionUrl"]}}`  
     - Authentication: HTTP Header Auth  
     - Header Parameters:  
       - `Authorization`: `Bearer {{$node["Set API Key"].json["replicate_api_key"]}}`  
     - Method: GET (default)  
   - Connect `"Wait"` output to this node.

7. **Create If Node to Check Completion**  
   - Add an **If** node named `"Check If Complete"`.  
   - Condition: Boolean  
     - Value 1: Expression `{{$json["status"]}}`  
     - Operation: Equal  
     - Value 2: `succeeded`  
   - Connect `"Check Prediction Status"` output to this node.

8. **Create Code Node to Process Result**  
   - Add a **Code** node named `"Process Result"`.  
   - Set to run once per item.  
   - Use this JavaScript code:  
     ```js
     const result = $input.item.json;
     return {
       status: result.status,
       output: result.output,
       metrics: result.metrics,
       created_at: result.created_at,
       completed_at: result.completed_at,
       model: 'dessix/moss-ttsd',
       text_url: result.output
     };
     ```  
   - Connect the **true** output of `"Check If Complete"` to this node.

9. **Loop Back If Not Complete**  
   - Connect the **false** output of `"Check If Complete"` back to the `"Wait"` node to continue polling.

10. **Verify and Save Workflow**  
    - Confirm all connections and node configurations.  
    - Replace `"YOUR_REPLICATE_API_KEY"` with your actual Replicate API key in the `"Set API Key"` node.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                    | Context or Link                                            |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------|
| This workflow uses the **dessix/moss-ttsd** model from Replicate to generate text content.                                                                     | Sticky Note on the workflow canvas                          |
| Setup instructions: 1. Add your Replicate API key 2. Configure input parameters 3. Run the workflow                                                             | Sticky Note on the workflow canvas                          |
| Model details: Type: Text Generation; Provider: dessix; Required fields: None                                                                                   | Sticky Note on the workflow canvas                          |
| Replicate API documentation for predictions: https://replicate.com/docs/api-reference/predictions                                                             | External reference for API usage                            |
| Consider adding error handling for prediction failures or timeouts to improve robustness                                                                        | Best practice suggestion                                   |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.