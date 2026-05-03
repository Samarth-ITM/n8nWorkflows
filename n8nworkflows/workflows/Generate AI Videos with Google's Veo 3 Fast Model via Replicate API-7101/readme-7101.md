Generate AI Videos with Google's Veo 3 Fast Model via Replicate API

https://n8nworkflows.xyz/workflows/generate-ai-videos-with-google-s-veo-3-fast-model-via-replicate-api-7101


# Generate AI Videos with Google's Veo 3 Fast Model via Replicate API

### 1. Workflow Overview

This workflow, titled **Google Veo 3 Fast Video Generator**, automates the generation of AI-created videos by interacting with the **google/veo-3-fast** model hosted on Replicate via their API. It is designed to accept a text prompt and produce a corresponding video output by submitting a prediction request, polling for completion, and finally processing the result.

The workflow is logically divided into the following blocks:

- **1.1 Input Initialization:** Triggering the workflow manually and setting the Replicate API key.
- **1.2 Prediction Creation:** Sending the video generation request to the Replicate API with the user’s prompt.
- **1.3 Prediction Monitoring:** Extracting the prediction ID and polling the API until the video generation is complete.
- **1.4 Result Processing:** Handling the completed prediction, extracting the video URL and metadata.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Initialization

**Overview:**  
This block initiates the workflow manually and sets the required authentication credentials for subsequent API calls.

**Nodes Involved:**  
- On clicking 'execute' (Manual Trigger)  
- Set API Key

**Node Details:**

- **On clicking 'execute'**  
  - *Type & Role:* Manual Trigger node, entry point for manual execution.  
  - *Configuration:* No parameters; simply triggers the workflow when executed by the user.  
  - *Inputs:* None  
  - *Outputs:* Connects to "Set API Key" node.  
  - *Edge Cases:* None, except user forgetting to trigger manually.

- **Set API Key**  
  - *Type & Role:* Set node, used to assign the Replicate API key to a variable for reuse.  
  - *Configuration:* Defines a string variable `replicate_api_key` with the placeholder value `"YOUR_REPLICATE_API_KEY"`.  
  - *Inputs:* Receives trigger from manual node.  
  - *Outputs:* Passes the API key to the next node "Create Prediction".  
  - *Edge Cases:* Failure if the API key is missing or invalid; workflow requires the user to replace the placeholder with a valid key.  
  - *Note:* Critical to set this key correctly to authenticate API requests.

#### 2.2 Prediction Creation

**Overview:**  
This block sends an HTTP POST request to Replicate’s API to create a new prediction for video generation using the specified model and input prompt.

**Nodes Involved:**  
- Create Prediction

**Node Details:**

- **Create Prediction**  
  - *Type & Role:* HTTP Request node, sends a POST request to Replicate API to start video generation.  
  - *Configuration:*  
    - URL: `https://api.replicate.com/v1/predictions`  
    - Method: POST  
    - Body: JSON specifying the model version (`7bb9e55061e89fd125fef046ce3983dbfdecb61b38399f381fb8309a77869b63`) and input parameters: a prompt string `"prompt value"` and seed = 1.  
    - Headers: Authorization with Bearer token from `replicate_api_key`, Content-Type application/json.  
    - Timeout: 60 seconds.  
  - *Inputs:* Receives API key from "Set API Key".  
  - *Outputs:* Passes API response to "Extract Prediction ID".  
  - *Expressions:* Uses expression to inject API key from previous node.  
  - *Edge Cases:*  
    - API key invalid or missing → 401 Unauthorized.  
    - Request timeout (exceeding 60s).  
    - Malformed prompt or API request errors.  
  - *Note:* The prompt is statically set to `"prompt value"` and should be replaced or parameterized for real use.

#### 2.3 Prediction Monitoring

**Overview:**  
After creating the prediction, this block extracts the prediction ID and polls the Replicate API repeatedly until the video generation status is “succeeded”.

**Nodes Involved:**  
- Extract Prediction ID  
- Wait  
- Check Prediction Status  
- Check If Complete

**Node Details:**

- **Extract Prediction ID**  
  - *Type & Role:* Code node, parses the prediction creation response to extract the prediction ID, status, and constructs the polling URL.  
  - *Configuration:* JavaScript code that extracts `id` and `status` from the JSON response and returns an object with `predictionId`, `status`, and `predictionUrl`.  
  - *Inputs:* Receives output from "Create Prediction".  
  - *Outputs:* Passes extracted data to "Wait".  
  - *Edge Cases:* If the response does not contain `id` or `status`, the node will fail or produce invalid URLs.

- **Wait**  
  - *Type & Role:* Wait node, delays execution for a fixed interval before polling again.  
  - *Configuration:* Waits for 2 seconds.  
  - *Inputs:* Receives from "Extract Prediction ID" initially and from "Check If Complete" when prediction is not yet complete.  
  - *Outputs:* Connects to "Check Prediction Status".  
  - *Edge Cases:* None significant; 2 seconds may be too short or long depending on model speed.

- **Check Prediction Status**  
  - *Type & Role:* HTTP Request node, polls the prediction status endpoint using the previously extracted URL.  
  - *Configuration:*  
    - URL: dynamic via expression `{{$json.predictionUrl}}`  
    - Method: GET (default)  
    - Header Authorization with Bearer token from API key.  
  - *Inputs:* From "Wait".  
  - *Outputs:* Passes status JSON to "Check If Complete".  
  - *Edge Cases:*  
    - Network errors or API downtime.  
    - Invalid prediction ID → 404 errors.  
    - Rate limiting by API.

- **Check If Complete**  
  - *Type & Role:* If node, evaluates if the prediction status is "succeeded".  
  - *Configuration:* Boolean condition comparing `$json.status` to `"succeeded"`.  
  - *Inputs:* From "Check Prediction Status".  
  - *Outputs:*  
    - True: proceeds to "Process Result".  
    - False: loops back to "Wait" to continue polling.  
  - *Edge Cases:*  
    - Other statuses like "failed" or "canceled" not explicitly handled, which may cause infinite polling.

#### 2.4 Result Processing

**Overview:**  
This block handles the successful prediction output by extracting relevant data such as the video URL, timestamps, and metadata for downstream use.

**Nodes Involved:**  
- Process Result

**Node Details:**

- **Process Result**  
  - *Type & Role:* Code node to parse the completed prediction JSON and format the output.  
  - *Configuration:* JavaScript code extracts status, output URL, metrics, creation and completion timestamps, and includes the model name `"google/veo-3-fast"`.  
  - *Inputs:* Receives from "Check If Complete" when status is "succeeded".  
  - *Outputs:* Outputs structured JSON containing video URL and metadata.  
  - *Edge Cases:* If the output field is missing or malformed, downstream usage may fail.

---

### 3. Summary Table

| Node Name               | Node Type            | Functional Role                      | Input Node(s)           | Output Node(s)            | Sticky Note                                                                                                                                                                                     |
|-------------------------|----------------------|------------------------------------|------------------------|---------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| On clicking 'execute'    | Manual Trigger       | Workflow manual start trigger      | —                      | Set API Key               |                                                                                                                                                                                                |
| Set API Key             | Set                  | Assign Replicate API key            | On clicking 'execute'   | Create Prediction         |                                                                                                                                                                                                |
| Create Prediction       | HTTP Request         | Create video generation prediction | Set API Key             | Extract Prediction ID     |                                                                                                                                                                                                |
| Extract Prediction ID   | Code                 | Extract prediction ID & status      | Create Prediction       | Wait                      |                                                                                                                                                                                                |
| Wait                    | Wait                 | Delay between status polls          | Extract Prediction ID; Check If Complete (false branch) | Check Prediction Status |                                                                                                                                                                                                |
| Check Prediction Status | HTTP Request         | Poll prediction status              | Wait                   | Check If Complete         |                                                                                                                                                                                                |
| Check If Complete       | If                   | Check if prediction succeeded       | Check Prediction Status | Process Result (true); Wait (false) |                                                                                                                                                                                                |
| Process Result          | Code                 | Format & output final video data    | Check If Complete       | —                         |                                                                                                                                                                                                |
| Sticky Note             | Sticky Note          | Informational overview & setup tips | —                      | —                         | ## Google Veo 3 Fast Video Generator<br><br>This workflow uses the **google/veo-3-fast** model from Replicate to generate video content.<br><br>### Setup<br>1. Add your Replicate API key<br>2. Configure the input parameters<br>3. Run the workflow<br><br>### Model Details<br>- **Type**: Video Generation<br>- **Provider**: google<br>- **Required Fields**: prompt |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.

2. **Add a Manual Trigger node**  
   - Name: `On clicking 'execute'`  
   - Purpose: To manually start the workflow.

3. **Add a Set node**  
   - Name: `Set API Key`  
   - Connect the manual trigger's output to this node's input.  
   - Configuration:  
     - Add a new field `replicate_api_key` of type string.  
     - Set its value to `YOUR_REPLICATE_API_KEY` (replace this placeholder with your actual Replicate API key).

4. **Add an HTTP Request node**  
   - Name: `Create Prediction`  
   - Connect `Set API Key` node output to this node.  
   - Configure:  
     - HTTP Method: POST  
     - URL: `https://api.replicate.com/v1/predictions`  
     - Authentication: None (use HTTP Header Auth manually)  
     - Headers:  
       - `Authorization`: Expression: `'Bearer ' + $node["Set API Key"].json["replicate_api_key"]`  
       - `Content-Type`: `application/json`  
     - Body Content Type: JSON  
     - Request Body (raw JSON):  
       ```json
       {
         "version": "7bb9e55061e89fd125fef046ce3983dbfdecb61b38399f381fb8309a77869b63",
         "input": {
           "prompt": "prompt value",
           "seed": 1
         }
       }
       ```  
     - Timeout: 60000 ms (60 seconds)

5. **Add a Code node**  
   - Name: `Extract Prediction ID`  
   - Connect `Create Prediction` output to this node.  
   - Paste the following JavaScript code:  
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

6. **Add a Wait node**  
   - Name: `Wait`  
   - Connect `Extract Prediction ID` output to this node.  
   - Set wait time to 2 seconds.

7. **Add another HTTP Request node**  
   - Name: `Check Prediction Status`  
   - Connect `Wait` output to this node.  
   - Configure:  
     - HTTP Method: GET  
     - URL: Expression: `{{$json["predictionUrl"]}}`  
     - Headers:  
       - `Authorization`: Expression: `'Bearer ' + $node["Set API Key"].json["replicate_api_key"]`

8. **Add an If node**  
   - Name: `Check If Complete`  
   - Connect `Check Prediction Status` output to this node.  
   - Configure:  
     - Condition Type: Boolean  
     - Expression: `$json["status"] === "succeeded"`  
   - True branch: Connect to next node "Process Result".  
   - False branch: Connect back to `Wait` node to repeat polling.

9. **Add a final Code node**  
   - Name: `Process Result`  
   - Connect `Check If Complete` true output to this node.  
   - Paste the following JavaScript code:  
     ```javascript
     const result = $input.item.json;

     return {
       status: result.status,
       output: result.output,
       metrics: result.metrics,
       created_at: result.created_at,
       completed_at: result.completed_at,
       model: 'google/veo-3-fast',
       video_url: result.output
     };
     ```

10. **Add a Sticky Note** (optional)  
    - Place a sticky note near the start of the workflow with the following content:  
      ```
      ## Google Veo 3 Fast Video Generator

      This workflow uses the **google/veo-3-fast** model from Replicate to generate video content.

      ### Setup
      1. Add your Replicate API key
      2. Configure the input parameters
      3. Run the workflow

      ### Model Details
      - **Type**: Video Generation
      - **Provider**: google
      - **Required Fields**: prompt
      ```

11. **Replace the prompt `"prompt value"`** in the `Create Prediction` node with your desired text prompt or modify the workflow to accept it as an input parameter.

12. **Save and activate the workflow**.  
    - Ensure your Replicate API key is valid.  
    - Trigger the workflow manually using the manual trigger node.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                |
|----------------------------------------------------------------------------------------------------|---------------------------------------------------------------|
| This workflow uses the Replicate API to generate AI-powered videos using Google’s veo-3-fast model. | Replicate API documentation: https://replicate.com/docs       |
| The polling mechanism uses a fixed 2-second wait interval; adjust based on expected model latency. | Consider API rate limits and possible exponential backoff.    |
| The workflow currently uses a hardcoded prompt value; parameterize for dynamic inputs.              | Modify `Create Prediction` node's JSON body accordingly.      |
| Replicate API key must be securely stored and never shared publicly.                                | Use n8n Credential management or environment variables.        |

---

**Disclaimer:** The provided text is extracted exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.