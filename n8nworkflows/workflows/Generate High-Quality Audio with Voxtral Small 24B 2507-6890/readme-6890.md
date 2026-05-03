Generate High-Quality Audio with Voxtral Small 24B 2507

https://n8nworkflows.xyz/workflows/generate-high-quality-audio-with-voxtral-small-24b-2507-6890


# Generate High-Quality Audio with Voxtral Small 24B 2507

### 1. Workflow Overview

This workflow, titled **Notdaniel Voxtral Small 24b 2507 Audio Generator**, automates the process of generating high-quality audio using the **notdaniel/voxtral-small-24b-2507** AI model hosted on Replicate. It is designed to accept an input audio URL and produce a processed audio output via the Replicate prediction API. The workflow is structured into several logical blocks handling input initialization, API authorization, prediction creation and polling, and result processing.

**Logical Blocks:**

- **1.1 Manual Trigger & API Key Setup:** Starts the workflow manually and assigns the Replicate API key needed for authentication.
- **1.2 Prediction Creation:** Sends a POST request to Replicate to initiate an audio generation prediction using the specified model.
- **1.3 Prediction ID Extraction:** Extracts the unique prediction ID and initial status from the response for subsequent polling.
- **1.4 Polling Loop:** Waits and repeatedly checks the prediction status until completion.
- **1.5 Result Processing:** When the prediction succeeds, processes the output and prepares relevant information including the generated audio URL.
- **1.6 Documentation:** A sticky note summarizes the workflow purpose and setup instructions.

---

### 2. Block-by-Block Analysis

#### 2.1 Manual Trigger & API Key Setup

**Overview:**  
This block allows the user to manually initiate the workflow and sets the Replicate API key required for authenticating API requests.

**Nodes Involved:**  
- On clicking 'execute' (Manual Trigger)  
- Set API Key (Set node)

**Node Details:**

- **On clicking 'execute'**  
  - Type: Manual Trigger  
  - Role: Entry point; triggers workflow execution on user command.  
  - Configuration: No parameters; basic manual trigger.  
  - Inputs: None  
  - Outputs: Passes trigger output to "Set API Key" node.  
  - Edge cases: User must manually start; no automated triggering.  

- **Set API Key**  
  - Type: Set  
  - Role: Stores the Replicate API key as a workflow variable (`replicate_api_key`).  
  - Configuration: Assigns the string `YOUR_REPLICATE_API_KEY` to variable `replicate_api_key`.  
  - Inputs: Receives trigger from manual node.  
  - Outputs: Provides API key for downstream authentication.  
  - Edge cases: Placeholder must be replaced with a valid API key prior to execution to avoid auth failures.

---

#### 2.2 Prediction Creation

**Overview:**  
This block constructs and sends a POST request to the Replicate API to start a new audio generation prediction using the specified model and input parameters.

**Nodes Involved:**  
- Create Prediction (HTTP Request)

**Node Details:**

- **Create Prediction**  
  - Type: HTTP Request  
  - Role: Calls Replicate’s `/v1/predictions` endpoint to create a prediction job.  
  - Configuration:  
    - Method: POST  
    - URL: `https://api.replicate.com/v1/predictions`  
    - Headers: Authorization with Bearer token from `replicate_api_key`, Content-Type application/json.  
    - Body (JSON):  
      ```json
      {
        "version": "31b40d9229c3df8c13863fcdd8cc95e6c1c6eb2b6c653936934234eadcc23d41",
        "input": {
          "audio": "https://example.com/input.audio",
          "max_new_tokens": 1024
        }
      }
      ```  
    - Timeout: 60 seconds  
  - Key expressions: Authorization header dynamically uses the API key set in previous node (`={{ 'Bearer ' + $('Set API Key').item.json.replicate_api_key }}`).  
  - Inputs: Receives API key from "Set API Key" node.  
  - Outputs: JSON response containing prediction metadata, including ID and status.  
  - Edge cases:  
    - HTTP errors (timeout, 4xx/5xx responses)  
    - Invalid or missing API key causes authentication failure.  
    - Invalid input audio URL or model version leads to API errors.  
  - Version-specific: Uses n8n HTTP Request node v4.2 for advanced authentication headers.

---

#### 2.3 Prediction ID Extraction

**Overview:**  
Extracts the prediction ID and initial status from the API response to be used for polling the prediction status.

**Nodes Involved:**  
- Extract Prediction ID (Code node)

**Node Details:**

- **Extract Prediction ID**  
  - Type: Code (JavaScript)  
  - Role: Parses JSON response to extract `id`, `status`, and constructs the polling URL for prediction status updates.  
  - Configuration:  
    - Runs once per item, processes the `$input.item.json`.  
    - Returns an object containing:  
      - `predictionId`  
      - `status`  
      - `predictionUrl` (constructed as `https://api.replicate.com/v1/predictions/${predictionId}`)  
  - Inputs: JSON from "Create Prediction" node.  
  - Outputs: Object with prediction metadata for polling.  
  - Edge cases:  
    - Missing or malformed response fields could cause runtime errors.  
    - Assumes successful creation returned with `id` and `status`.  
  - Version-specific: Uses n8n Code node v2.

---

#### 2.4 Polling Loop

**Overview:**  
Implements a loop that waits for a fixed interval and polls the prediction status until it is marked as "succeeded".

**Nodes Involved:**  
- Wait (Wait node)  
- Check Prediction Status (HTTP Request)  
- Check If Complete (If node)

**Node Details:**

- **Wait**  
  - Type: Wait  
  - Role: Pauses execution for 2 seconds between status checks to avoid excessive API calls.  
  - Configuration: Wait for 2 seconds.  
  - Inputs: Receives prediction metadata from "Extract Prediction ID" or "Check If Complete" nodes.  
  - Outputs: Triggers "Check Prediction Status".  
  - Edge cases: Long wait times slow workflow; too short may hit rate limits.

- **Check Prediction Status**  
  - Type: HTTP Request  
  - Role: Sends GET request to `predictionUrl` to fetch latest prediction status and result.  
  - Configuration:  
    - URL: dynamically uses `={{ $json.predictionUrl }}` from previous node output.  
    - Headers: Authorization with Bearer token from `replicate_api_key`.  
    - Method: GET (default)  
  - Inputs: Wait node trigger.  
  - Outputs: JSON response with updated prediction status and possibly output.  
  - Edge cases: API rate limits, network errors, auth failures.

- **Check If Complete**  
  - Type: If  
  - Role: Checks if the prediction status equals "succeeded".  
  - Configuration: Boolean condition: `$json.status === "succeeded"`  
  - Inputs: JSON from "Check Prediction Status".  
  - Outputs:  
    - True branch: leads to "Process Result" node.  
    - False branch: loops back to "Wait" node for another polling cycle.  
  - Edge cases: Status values other than "succeeded" and "failed" not explicitly handled (e.g., "processing", "canceled").  
  - Possible enhancement: handle failure states explicitly.

---

#### 2.5 Result Processing

**Overview:**  
Once prediction completes successfully, this block formats and extracts relevant details from the result for downstream use.

**Nodes Involved:**  
- Process Result (Code node)

**Node Details:**

- **Process Result**  
  - Type: Code (JavaScript)  
  - Role: Extracts key metadata such as status, output URL, metrics, timestamps, and model info.  
  - Configuration:  
    - Returns an object with:  
      - `status`  
      - `output` (audio URL)  
      - `metrics`  
      - `created_at` and `completed_at` timestamps  
      - fixed model identifier string `"notdaniel/voxtral-small-24b-2507"`  
      - `audio_url` (alias of output)  
  - Inputs: JSON from "Check If Complete" node on success branch.  
  - Outputs: Structured data including audio URL for playback or download.  
  - Edge cases: Missing fields if prediction incomplete or malformed response.

---

#### 2.6 Documentation

**Overview:**  
A sticky note node summarizes the workflow purpose, setup instructions, and model details for user reference.

**Nodes Involved:**  
- Sticky Note

**Node Details:**

- **Sticky Note**  
  - Type: Sticky Note  
  - Role: Provides visual documentation inside the n8n editor.  
  - Content:  
    ```
    ## Notdaniel Voxtral Small 24b 2507 Audio Generator

    This workflow uses the **notdaniel/voxtral-small-24b-2507** model from Replicate to generate audio content.

    ### Setup
    1. Add your Replicate API key
    2. Configure the input parameters
    3. Run the workflow

    ### Model Details
    - **Type**: Audio Generation
    - **Provider**: notdaniel
    - **Required Fields**: audio
    ```
  - Inputs/Outputs: None (purely informational).  
  - Edge cases: None.

---

### 3. Summary Table

| Node Name             | Node Type        | Functional Role                          | Input Node(s)            | Output Node(s)            | Sticky Note                                                                                  |
|-----------------------|------------------|----------------------------------------|--------------------------|---------------------------|---------------------------------------------------------------------------------------------|
| On clicking 'execute'  | Manual Trigger   | Workflow start via manual user trigger | None                     | Set API Key               |                                                                                             |
| Set API Key           | Set              | Stores Replicate API key for auth      | On clicking 'execute'    | Create Prediction         |                                                                                             |
| Create Prediction     | HTTP Request     | Sends prediction creation request      | Set API Key              | Extract Prediction ID     |                                                                                             |
| Extract Prediction ID | Code             | Extracts prediction ID and status      | Create Prediction        | Wait                      |                                                                                             |
| Wait                  | Wait             | Pauses between polling attempts        | Extract Prediction ID, Check If Complete (False branch) | Check Prediction Status  |                                                                                             |
| Check Prediction Status| HTTP Request     | Polls API for updated prediction status| Wait                     | Check If Complete         |                                                                                             |
| Check If Complete     | If               | Checks if prediction finished           | Check Prediction Status  | Process Result (True), Wait (False) |                                                                                             |
| Process Result        | Code             | Processes and formats final prediction | Check If Complete (True) | None                      |                                                                                             |
| Sticky Note           | Sticky Note      | Documentation and workflow instructions| None                     | None                      | ## Notdaniel Voxtral Small 24b 2507 Audio Generator<br>This workflow uses the **notdaniel/voxtral-small-24b-2507** model from Replicate to generate audio content.<br>### Setup<br>1. Add your Replicate API key<br>2. Configure the input parameters<br>3. Run the workflow<br>### Model Details<br>- **Type**: Audio Generation<br>- **Provider**: notdaniel<br>- **Required Fields**: audio |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Name: `On clicking 'execute'`  
   - Type: Manual Trigger (n8n-nodes-base.manualTrigger)  
   - No parameters needed.

2. **Create Set Node for API Key:**  
   - Name: `Set API Key`  
   - Type: Set  
   - Add a string field named `replicate_api_key` with your actual Replicate API key as value.  
   - Connect output of `On clicking 'execute'` to this node.

3. **Create HTTP Request Node to Create Prediction:**  
   - Name: `Create Prediction`  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.replicate.com/v1/predictions`  
   - Authentication: Generic HTTP Header Auth  
     - Header Name: `Authorization`  
     - Header Value: `={{ 'Bearer ' + $('Set API Key').item.json.replicate_api_key }}`  
   - Headers: `Content-Type: application/json`  
   - Body Content-Type: JSON  
   - JSON Body:  
     ```json
     {
       "version": "31b40d9229c3df8c13863fcdd8cc95e6c1c6eb2b6c653936934234eadcc23d41",
       "input": {
         "audio": "https://example.com/input.audio",
         "max_new_tokens": 1024
       }
     }
     ```  
   - Timeout: 60 seconds  
   - Connect output of `Set API Key` to this node.

4. **Create Code Node to Extract Prediction ID:**  
   - Name: `Extract Prediction ID`  
   - Type: Code  
   - Mode: Run Once For Each Item  
   - JavaScript:  
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

5. **Create Wait Node:**  
   - Name: `Wait`  
   - Type: Wait  
   - Parameters: Wait for 2 seconds  
   - Connect output of `Extract Prediction ID` to this node.

6. **Create HTTP Request Node to Check Prediction Status:**  
   - Name: `Check Prediction Status`  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `={{ $json.predictionUrl }}`  
   - Authentication: Generic HTTP Header Auth  
     - Header Name: `Authorization`  
     - Header Value: `={{ 'Bearer ' + $('Set API Key').item.json.replicate_api_key }}`  
   - Connect output of `Wait` node to this node.

7. **Create If Node to Check Completion:**  
   - Name: `Check If Complete`  
   - Type: If  
   - Condition: Boolean  
     - Expression: `$json.status === "succeeded"`  
   - Connect output of `Check Prediction Status` to this node.

8. **Create Code Node to Process Result:**  
   - Name: `Process Result`  
   - Type: Code  
   - Mode: Run Once For Each Item  
   - JavaScript:  
     ```js
     const result = $input.item.json;
     return {
       status: result.status,
       output: result.output,
       metrics: result.metrics,
       created_at: result.created_at,
       completed_at: result.completed_at,
       model: 'notdaniel/voxtral-small-24b-2507',
       audio_url: result.output
     };
     ```  
   - Connect the **True** output of `Check If Complete` to this node.

9. **Loop Back for Polling:**  
   - Connect the **False** output of `Check If Complete` back to the `Wait` node to continue polling.

10. **Add Sticky Note for Documentation:**  
    - Content:  
      ```
      ## Notdaniel Voxtral Small 24b 2507 Audio Generator

      This workflow uses the **notdaniel/voxtral-small-24b-2507** model from Replicate to generate audio content.

      ### Setup
      1. Add your Replicate API key
      2. Configure the input parameters
      3. Run the workflow

      ### Model Details
      - **Type**: Audio Generation
      - **Provider**: notdaniel
      - **Required Fields**: audio
      ```  
    - Place near the workflow start for user clarity.

**Credentials Setup:**  
- Obtain a valid Replicate API key from https://replicate.com/account  
- Use this key in the `Set API Key` node.  
- No other credentials required.

**Input Customization:**  
- Replace `"https://example.com/input.audio"` in `Create Prediction` node’s JSON body with the actual audio file URL to be processed.  
- Adjust `"max_new_tokens"` if needed according to model limits.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                       |
|---------------------------------------------------------------------------------------------------------------|------------------------------------------------------|
| This workflow leverages Replicate’s prediction API to perform AI-driven audio generation with the Voxtral model. | Workflow purpose                                      |
| Replace the placeholder API key with a valid key to avoid authentication errors.                              | Credential setup                                      |
| The polling loop uses a 2-second wait to balance responsiveness with API rate limits.                         | Polling strategy                                      |
| Model version ID corresponds to a specific Voxtral Small 24b 2507 model snapshot on Replicate.                | Model versioning                                      |
| For more details on Replicate API and models, visit https://replicate.com/docs/api-reference                 | Replicate API documentation                           |
| Sticky notes in n8n provide helpful in-editor references but do not affect workflow execution.                | Workflow documentation practice                       |

---

**Disclaimer:**  
The provided content originates entirely from an automated workflow created with n8n, an integration and automation tool. This processing strictly follows current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.