Generate Images with Flash V2.0.0 Beta.10 AI on Replicate

https://n8nworkflows.xyz/workflows/generate-images-with-flash-v2-0-0-beta-10-ai-on-replicate-7104


# Generate Images with Flash V2.0.0 Beta.10 AI on Replicate

### 1. Workflow Overview

This workflow, titled **Settyan Flash V2.0.0 Beta.10 AI Generator**, automates the process of generating images or other content using the **settyan/flash-v2.0.0-beta.10** AI model hosted on Replicate. It is designed to:

- Accept a manual trigger to start the generation process
- Use a user-provided Replicate API key for authentication
- Submit a generation request with specific input parameters (e.g., prompt, seed, dimensions)
- Poll the Replicate API to check the prediction status until completion
- Process and output the final generated result

The workflow’s logic is grouped into the following functional blocks:

- **1.1 Input Reception and API Key Setup:** Manual trigger and API key assignment  
- **1.2 Prediction Creation:** Sending generation request to Replicate  
- **1.3 Prediction Monitoring:** Polling the API for completion status with wait intervals  
- **1.4 Result Processing:** Handling the completed prediction data  

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and API Key Setup

**Overview:**  
This block initiates the workflow manually and sets the required Replicate API key for subsequent authenticated API calls.

**Nodes Involved:**  
- On clicking 'execute' (Manual Trigger)  
- Set API Key (Set node)  

**Node Details:**  

- **On clicking 'execute'**  
  - Type: Manual Trigger  
  - Role: Entry point to start workflow execution manually  
  - Configuration: No parameters; triggers workflow when user clicks execute  
  - Inputs: None  
  - Outputs: To "Set API Key" node  
  - Failure Cases: None typical, but user must trigger manually  

- **Set API Key**  
  - Type: Set  
  - Role: Assigns the Replicate API key as a workflow variable  
  - Configuration: Sets a string variable `replicate_api_key` with placeholder `"YOUR_REPLICATE_API_KEY"` (user must edit before running)  
  - Inputs: From manual trigger  
  - Outputs: To "Create Prediction" node  
  - Failure Cases: If API key is missing or invalid, subsequent API calls will fail with auth errors  

---

#### 1.2 Prediction Creation

**Overview:**  
This block sends a POST request to Replicate’s prediction API to initiate image/content generation with the specified model version and input parameters.

**Nodes Involved:**  
- Create Prediction (HTTP Request)  
- Extract Prediction ID (Code)  

**Node Details:**  

- **Create Prediction**  
  - Type: HTTP Request  
  - Role: Sends POST request to `https://api.replicate.com/v1/predictions` to create a prediction  
  - Configuration:  
    - Method: POST  
    - Timeout: 60 seconds  
    - JSON Body with fixed model version `8a5e4f716f0a4acfb122d12d120f9c14a0f0addc05dcb264dea657595f39b9bf`  
    - Input fields in body:  
      ```json
      {
        "prompt": "prompt value",
        "seed": 1,
        "width": 1,
        "height": 1,
        "lora_scale": 1
      }
      ```  
      (Note: These are placeholders; users must provide meaningful values for real use)  
    - Headers:  
      - Authorization: Bearer token from `Set API Key` node variable  
      - Content-Type: application/json  
  - Inputs: From "Set API Key"  
  - Outputs: To "Extract Prediction ID"  
  - Failure Cases: HTTP errors, auth failures if API key is invalid, timeout if API is slow  

- **Extract Prediction ID**  
  - Type: Code (JavaScript)  
  - Role: Parses API response, extracts prediction ID, status, and constructs the status polling URL  
  - Configuration:  
    - Runs once per item  
    - JavaScript extracts `id` and `status` from response JSON and returns:  
      - `predictionId`  
      - `status`  
      - `predictionUrl` (for status checks)  
  - Inputs: From "Create Prediction"  
  - Outputs: To "Wait" node  
  - Failure Cases: If response structure changes, extraction may fail; must handle JSON parsing errors  

---

#### 1.3 Prediction Monitoring

**Overview:**  
This block repeatedly polls the Replicate API at intervals to check if the prediction is complete, using a wait node to space out requests and an IF node to determine the next action.

**Nodes Involved:**  
- Wait (Wait node)  
- Check Prediction Status (HTTP Request)  
- Check If Complete (IF node)  

**Node Details:**  

- **Wait**  
  - Type: Wait  
  - Role: Pauses workflow execution for 2 seconds between status polls to avoid rate limits  
  - Configuration: Wait 2 seconds  
  - Inputs: From "Extract Prediction ID" or "Check If Complete" (loop)  
  - Outputs: To "Check Prediction Status"  

- **Check Prediction Status**  
  - Type: HTTP Request  
  - Role: Sends GET request to prediction status URL to fetch current status  
  - Configuration:  
    - URL: Dynamically set to `predictionUrl` extracted previously  
    - Method: GET (default)  
    - Headers: Authorization with Bearer token from API key  
  - Inputs: From "Wait"  
  - Outputs: To "Check If Complete"  
  - Failure Cases: HTTP errors; token expiration; network issues  

- **Check If Complete**  
  - Type: IF  
  - Role: Branches workflow based on whether prediction status equals "succeeded"  
  - Configuration:  
    - Condition: `$json.status == "succeeded"`  
  - Inputs: From "Check Prediction Status"  
  - Outputs:  
    - True branch: To "Process Result"  
    - False branch: Loops back to "Wait" node for another polling iteration  
  - Failure Cases: If status field missing or API returns error status  

---

#### 1.4 Result Processing

**Overview:**  
This block handles the final successful prediction, extracting relevant output fields and formatting them for downstream use or storage.

**Nodes Involved:**  
- Process Result (Code)  

**Node Details:**  

- **Process Result**  
  - Type: Code (JavaScript)  
  - Role: Extracts and structures final prediction data including output images, metrics, timestamps, and model info  
  - Configuration: Runs once per item, returns JSON object with:  
    - status  
    - output (likely URLs of generated images)  
    - metrics  
    - created_at, completed_at timestamps  
    - model name (hardcoded)  
    - other_url (same as output)  
  - Inputs: From "Check If Complete" (true branch)  
  - Outputs: Terminal output of the workflow  
  - Failure Cases: If prediction result format changes, extraction may fail  

---

### 3. Summary Table

| Node Name            | Node Type         | Functional Role                   | Input Node(s)          | Output Node(s)           | Sticky Note                                                                                         |
|----------------------|-------------------|---------------------------------|-----------------------|--------------------------|---------------------------------------------------------------------------------------------------|
| On clicking 'execute' | Manual Trigger    | Start workflow manually          | None                  | Set API Key              |                                                                                                   |
| Set API Key          | Set               | Assign API key variable          | On clicking 'execute'  | Create Prediction        |                                                                                                   |
| Create Prediction    | HTTP Request      | Send generation request to Replicate | Set API Key            | Extract Prediction ID    |                                                                                                   |
| Extract Prediction ID| Code              | Extract prediction ID and status | Create Prediction      | Wait                     |                                                                                                   |
| Wait                 | Wait              | Pause between polling attempts   | Extract Prediction ID, Check If Complete (false branch) | Check Prediction Status |                                                                                                   |
| Check Prediction Status | HTTP Request    | Poll prediction status API       | Wait                  | Check If Complete        |                                                                                                   |
| Check If Complete    | IF                | Branch on prediction completion  | Check Prediction Status| Process Result (true), Wait (false) |                                                                                                   |
| Process Result       | Code              | Format and output prediction results | Check If Complete (true) | None                    |                                                                                                   |
| Sticky Note          | Sticky Note       | Workflow description and setup instructions | None                  | None                     | ## Settyan Flash V2.0.0 Beta.10 AI Generator\n\nThis workflow uses the **settyan/flash-v2.0.0-beta.10** model from Replicate to generate other content.\n\n### Setup\n1. Add your Replicate API key\n2. Configure the input parameters\n3. Run the workflow\n\n### Model Details\n- **Type**: Other Generation\n- **Provider**: settyan\n- **Required Fields**: prompt |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: `On clicking 'execute'`  
   - Type: Manual Trigger  
   - No parameters  

2. **Create Set Node for API Key**  
   - Name: `Set API Key`  
   - Type: Set  
   - Add a string field `replicate_api_key` with value `"YOUR_REPLICATE_API_KEY"` (replace with your actual Replicate API key)  
   - Connect output of Manual Trigger to input of this Set node  

3. **Create HTTP Request Node to Create Prediction**  
   - Name: `Create Prediction`  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.replicate.com/v1/predictions`  
   - Authentication: Generic Credential with HTTP Header Auth  
   - Header Parameters:  
     - `Authorization`: `Bearer {{$node["Set API Key"].json["replicate_api_key"]}}`  
     - `Content-Type`: `application/json`  
   - Body Content (JSON):  
     ```json
     {
       "version": "8a5e4f716f0a4acfb122d12d120f9c14a0f0addc05dcb264dea657595f39b9bf",
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
   - Connect output of `Set API Key` to input of this node  

4. **Create Code Node to Extract Prediction ID**  
   - Name: `Extract Prediction ID`  
   - Type: Code (JavaScript)  
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
   - Connect output of `Create Prediction` to this node  

5. **Create Wait Node to Pause Between Polls**  
   - Name: `Wait`  
   - Type: Wait  
   - Wait: 2 seconds  
   - Connect output of `Extract Prediction ID` to this node  

6. **Create HTTP Request Node to Check Prediction Status**  
   - Name: `Check Prediction Status`  
   - Type: HTTP Request  
   - Method: GET (default)  
   - URL: `{{$json["predictionUrl"]}}` (dynamic from previous node)  
   - Authentication: Generic Credential with HTTP Header Auth  
   - Header Parameters:  
     - `Authorization`: `Bearer {{$node["Set API Key"].json["replicate_api_key"]}}`  
   - Connect output of `Wait` to this node  

7. **Create IF Node to Check If Prediction Completed**  
   - Name: `Check If Complete`  
   - Type: IF  
   - Condition: Boolean equals  
     - Value 1: `{{$json["status"]}}`  
     - Value 2: `succeeded`  
   - Connect output of `Check Prediction Status` to this node  

8. **Create Code Node to Process Completed Prediction**  
   - Name: `Process Result`  
   - Type: Code (JavaScript)  
   - Code:  
     ```js
     const result = $input.item.json;

     return {
       status: result.status,
       output: result.output,
       metrics: result.metrics,
       created_at: result.created_at,
       completed_at: result.completed_at,
       model: 'settyan/flash-v2.0.0-beta.10',
       other_url: result.output
     };
     ```  
   - Connect the **true** output of `Check If Complete` to this node  

9. **Loop Back from IF Node False Branch to Wait**  
   - Connect the **false** output of `Check If Complete` back to the input of `Wait` node to repeat polling  

10. **Add Sticky Note** (optional)  
    - Add a sticky note with the workflow description, setup instructions, and model details as per section 3  

**Credentials Setup:**  
- Create a Generic Credential in n8n with HTTP Header Auth for the Replicate API token.  
- Use this credential in both HTTP Request nodes (`Create Prediction` and `Check Prediction Status`).  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                     | Context or Link                                                                                   |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This workflow uses the **settyan/flash-v2.0.0-beta.10** model from Replicate for content generation. Replace placeholder inputs with actual prompts and parameters. | Model on Replicate: https://replicate.com/settyan/flash-v2.0.0-beta.10                           |
| The API key must be kept secret and valid for authentication to avoid authorization errors.                                                                     | Replicate API Documentation: https://replicate.com/docs/reference/http                          |
| Polling interval is set to 2 seconds to balance responsiveness and rate limits; adjust if needed.                                                               | Consider API rate limits to avoid throttling                                                    |
| Input parameters such as prompt, seed, width, and height are hardcoded placeholders—modify these to customize generation.                                        |                                                                                                 |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects applicable content policies and contains no illegal, offensive, or protected elements. All manipulated data are legal and public.