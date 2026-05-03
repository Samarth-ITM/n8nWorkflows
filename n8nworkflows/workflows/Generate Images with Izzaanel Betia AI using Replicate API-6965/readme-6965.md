Generate Images with Izzaanel Betia AI using Replicate API

https://n8nworkflows.xyz/workflows/generate-images-with-izzaanel-betia-ai-using-replicate-api-6965


# Generate Images with Izzaanel Betia AI using Replicate API

### 1. Workflow Overview

This workflow, titled **"Izzaanel Betia AI Generator"**, is designed to generate images or other visual content using the **izzaanel/betia** model hosted on Replicate's API. It targets users who want to integrate AI-driven image generation into their automation processes. The workflow performs an asynchronous prediction request to the Replicate API, polls for the prediction result, and processes the output once the generation is complete.

The workflow is logically divided into the following blocks:

- **1.1 Trigger and API Key Setup:** Starts the workflow manually and sets the API key required for authentication with Replicate.
- **1.2 Initiate Prediction:** Sends a POST request to create a prediction job with the specified model and input parameters.
- **1.3 Poll Prediction Status:** Periodically checks the status of the prediction until it completes.
- **1.4 Process Completed Prediction:** Extracts relevant data and formats the final output from the completed prediction.
- **1.5 User Guidance:** A sticky note describing the workflow purpose, setup instructions, and model details.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger and API Key Setup

**Overview:**  
This block initiates the workflow manually and assigns the Replicate API key to a variable for subsequent authenticated requests.

**Nodes Involved:**  
- On clicking 'execute'  
- Set API Key

**Node Details:**

- **On clicking 'execute'**  
  - Type: Manual Trigger  
  - Role: Entry point to start the workflow manually by user interaction.  
  - Configuration: No parameters; triggers the workflow when clicked.  
  - Input: None  
  - Output: Triggers "Set API Key" node.  
  - Edge cases: None expected.  
  - Version: 1

- **Set API Key**  
  - Type: Set  
  - Role: Stores the Replicate API key in the workflow context as the variable `replicate_api_key`.  
  - Configuration: Assigns the string `"YOUR_REPLICATE_API_KEY"` to `replicate_api_key` (to be replaced by the user).  
  - Input: Trigger from manual node.  
  - Output: Passes data to "Create Prediction" node.  
  - Edge cases: Failure or misconfiguration if the API key is missing or invalid.  
  - Version: 3.3

---

#### 1.2 Initiate Prediction

**Overview:**  
This block sends a POST request to Replicate's API to start a new prediction job using the izzaanel/betia model with given input parameters.

**Nodes Involved:**  
- Create Prediction  
- Extract Prediction ID

**Node Details:**

- **Create Prediction**  
  - Type: HTTP Request  
  - Role: Calls the Replicate API endpoint `/v1/predictions` to initiate image generation.  
  - Configuration:  
    - Method: POST  
    - URL: `https://api.replicate.com/v1/predictions`  
    - Headers: Authorization (Bearer token from `replicate_api_key`), Content-Type: application/json  
    - Body (JSON):  
      ```json
      {
        "version": "cf6ebd6365656db3e6f55ab66444a8c144e960298905dadebb976816ed62451b",
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
  - Input: From "Set API Key" node with API key variable.  
  - Output: JSON response containing prediction job data including `id` and `status`.  
  - Edge cases:  
    - HTTP errors (4xx/5xx) due to invalid parameters or API key.  
    - Timeout if the request takes too long.  
  - Version: 4.2

- **Extract Prediction ID**  
  - Type: Code  
  - Role: Extracts the prediction ID and initial status from the API response for polling.  
  - Configuration: Custom JavaScript code runs once per item to create a simplified JSON with prediction ID, status, and a constructed URL for status check.  
  - Key variables:  
    - `predictionId` from response `id` field  
    - `initialStatus` from response `status` field  
    - `predictionUrl` constructed as `https://api.replicate.com/v1/predictions/{predictionId}`  
  - Input: From "Create Prediction" node's output.  
  - Output: JSON with `predictionId`, `status`, and `predictionUrl` keys.  
  - Edge cases:  
    - Missing or malformed response JSON.  
    - Expression errors if expected fields are missing.  
  - Version: 2

---

#### 1.3 Poll Prediction Status

**Overview:**  
This block waits for a short interval and then queries the prediction status repeatedly until the prediction is completed.

**Nodes Involved:**  
- Wait  
- Check Prediction Status  
- Check If Complete

**Node Details:**

- **Wait**  
  - Type: Wait  
  - Role: Pauses workflow for 2 seconds before the next status check to avoid excessive API calls.  
  - Configuration: Wait 2 seconds.  
  - Input: From "Extract Prediction ID" or "Check If Complete" (when prediction not finished).  
  - Output: Triggers "Check Prediction Status".  
  - Edge cases: None significant; long wait could delay feedback.  
  - Version: 1

- **Check Prediction Status**  
  - Type: HTTP Request  
  - Role: Sends GET request to Replicate API to fetch current prediction status.  
  - Configuration:  
    - URL: Dynamic, from `predictionUrl` variable (e.g., `https://api.replicate.com/v1/predictions/{id}`)  
    - Headers: Authorization with Bearer token from `replicate_api_key`  
    - Method: GET (default)  
  - Input: From "Wait" node.  
  - Output: Current prediction JSON including `status` field.  
  - Edge cases:  
    - HTTP errors or token expiry.  
    - Network timeouts.  
  - Version: 4.2

- **Check If Complete**  
  - Type: If  
  - Role: Branches workflow depending on whether prediction status is `"succeeded"`.  
  - Configuration: Checks if `$json.status == "succeeded"`  
  - Input: From "Check Prediction Status" node output.  
  - Output:  
    - True: proceeds to "Process Result" node.  
    - False: loops back to "Wait" node to poll again.  
  - Edge cases:  
    - Other statuses like `"failed"`, `"canceled"` are not explicitly handled and will cause an infinite loop.  
  - Version: 1

---

#### 1.4 Process Completed Prediction

**Overview:**  
Once the prediction completes successfully, this block processes and formats the output data for downstream use.

**Nodes Involved:**  
- Process Result

**Node Details:**

- **Process Result**  
  - Type: Code  
  - Role: Extracts important fields such as status, output URLs, metrics, timestamps, and model identifier from the prediction result.  
  - Configuration: Custom JavaScript that returns a simplified object with keys:  
    - `status`  
    - `output` (image URL array or similar)  
    - `metrics`  
    - `created_at`  
    - `completed_at`  
    - `model` (hardcoded to `"izzaanel/betia"`)  
    - `other_url` (same as output)  
  - Input: From "Check If Complete" (True branch).  
  - Output: Cleaned and structured output JSON.  
  - Edge cases:  
    - Missing fields in API response can cause undefined values.  
  - Version: 2

---

#### 1.5 User Guidance

**Overview:**  
Provides a visual sticky note in the editor with important notes about the workflow, model, and setup instructions.

**Nodes Involved:**  
- Sticky Note

**Node Details:**

- **Sticky Note**  
  - Type: Sticky Note  
  - Role: Documentation for users configuring or reviewing the workflow.  
  - Content Summary:  
    - Workflow title and purpose  
    - Setup steps to add API key and configure inputs  
    - Model details: Type (Other Generation), Provider (izzaanel), Required Fields (prompt)  
  - Position: Top-left for easy visibility.  
  - Edge cases: None.  
  - Version: 1

---

### 3. Summary Table

| Node Name               | Node Type           | Functional Role                     | Input Node(s)               | Output Node(s)              | Sticky Note                                                                                                 |
|-------------------------|---------------------|-----------------------------------|-----------------------------|-----------------------------|-------------------------------------------------------------------------------------------------------------|
| On clicking 'execute'   | Manual Trigger      | Starts workflow manually           | None                        | Set API Key                 |                                                                                                             |
| Set API Key             | Set                 | Stores Replicate API key           | On clicking 'execute'       | Create Prediction           |                                                                                                             |
| Create Prediction       | HTTP Request        | Sends POST to start prediction     | Set API Key                 | Extract Prediction ID       |                                                                                                             |
| Extract Prediction ID   | Code                | Extracts prediction ID and URL     | Create Prediction           | Wait                       |                                                                                                             |
| Wait                    | Wait                | Pauses before polling prediction   | Extract Prediction ID, Check If Complete (False) | Check Prediction Status |                                                                                                             |
| Check Prediction Status | HTTP Request        | Checks prediction status           | Wait                       | Check If Complete           |                                                                                                             |
| Check If Complete       | If                  | Branches on prediction completion  | Check Prediction Status     | Process Result, Wait        |                                                                                                             |
| Process Result          | Code                | Extracts and formats final output  | Check If Complete (True)    | None                       |                                                                                                             |
| Sticky Note             | Sticky Note         | Provides setup and model info      | None                       | None                       | ## Izzaanel Betia AI Generator<br>This workflow uses the **izzaanel/betia** model from Replicate to generate other content.<br><br>### Setup<br>1. Add your Replicate API key<br>2. Configure the input parameters<br>3. Run the workflow<br><br>### Model Details<br>- **Type**: Other Generation<br>- **Provider**: izzaanel<br>- **Required Fields**: prompt |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the manual trigger node:**  
   - Add node: **Manual Trigger**  
   - Name it: `On clicking 'execute'`  
   - No parameters needed.

2. **Create a Set node to store API key:**  
   - Add node: **Set**  
   - Name it: `Set API Key`  
   - Add a string field assignment:  
     - Variable name: `replicate_api_key`  
     - Value: `"YOUR_REPLICATE_API_KEY"` (replace with your actual Replicate API key).  
   - Connect output from `On clicking 'execute'` to this node.

3. **Create HTTP Request node to initiate prediction:**  
   - Add node: **HTTP Request**  
   - Name it: `Create Prediction`  
   - Configure:  
     - HTTP Method: POST  
     - URL: `https://api.replicate.com/v1/predictions`  
     - Authentication: None (will use header auth manually)  
     - Headers:  
       - `Authorization`: Use expression: `'Bearer ' + $node["Set API Key"].json["replicate_api_key"]`  
       - `Content-Type`: `application/json`  
     - Body Content Type: JSON  
     - JSON Body:  
       ```json
       {
         "version": "cf6ebd6365656db3e6f55ab66444a8c144e960298905dadebb976816ed62451b",
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

4. **Create a Code node to extract prediction ID:**  
   - Add node: **Code**  
   - Name it: `Extract Prediction ID`  
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
   - Connect output from `Create Prediction` to this node.

5. **Create a Wait node to add delay:**  
   - Add node: **Wait**  
   - Name it: `Wait`  
   - Set wait time: 2 seconds  
   - Connect output from `Extract Prediction ID` to this node.

6. **Create HTTP Request node to check prediction status:**  
   - Add node: **HTTP Request**  
   - Name it: `Check Prediction Status`  
   - Configure:  
     - HTTP Method: GET (default)  
     - URL: Use expression: `$json["predictionUrl"]`  
     - Headers:  
       - `Authorization`: `'Bearer ' + $node["Set API Key"].json["replicate_api_key"]`  
     - No authentication selected (manual header auth).  
   - Connect output from `Wait` to this node.

7. **Create an If node to check if prediction completed:**  
   - Add node: **If**  
   - Name it: `Check If Complete`  
   - Condition: Check if status equals "succeeded"  
     - Expression: `$json["status"] == "succeeded"`  
   - Connect output from `Check Prediction Status` to this node.

8. **Looping logic:**  
   - Connect **False** output of `Check If Complete` back to `Wait` node (to continue polling).  
   - Connect **True** output of `Check If Complete` to next node (`Process Result`).

9. **Create Code node to process the final result:**  
   - Add node: **Code**  
   - Name it: `Process Result`  
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
       model: 'izzaanel/betia',
       other_url: result.output
     };
     ```  
   - Connect output from `Check If Complete` (True) to this node.

10. **Add a Sticky Note for user guidance (optional but recommended):**  
    - Add node: **Sticky Note**  
    - Position it visibly in the editor.  
    - Content:  
      ```
      ## Izzaanel Betia AI Generator

      This workflow uses the **izzaanel/betia** model from Replicate to generate other content.

      ### Setup
      1. Add your Replicate API key
      2. Configure the input parameters
      3. Run the workflow

      ### Model Details
      - **Type**: Other Generation
      - **Provider**: izzaanel
      - **Required Fields**: prompt
      ```

---

### 5. General Notes & Resources

| Note Content                                                                                           | Context or Link                                                   |
|------------------------------------------------------------------------------------------------------|------------------------------------------------------------------|
| The workflow requires a valid Replicate API key with access to the `izzaanel/betia` model.            | https://replicate.com/izzaanel/betia                             |
| Prompt parameter in the Create Prediction node's JSON must be replaced with a meaningful text prompt for image generation. | User configuration                                              |
| The polling interval is set to 2 seconds; increasing this reduces API calls but delays result retrieval. | Performance tuning                                              |
| No explicit error handling for failed or canceled predictions is implemented; consider adding it for robustness. | Workflow enhancement suggestion                                 |
| The current model version string is hardcoded; update it as needed based on Replicate's model updates. | Replicate API documentation                                     |

---

_Disclaimer: The provided text is exclusively derived from an automated workflow created with n8n, a tool for integration and automation. This processing strictly respects applicable content policies and contains no illegal, offensive, or protected material. All processed data is legal and public._