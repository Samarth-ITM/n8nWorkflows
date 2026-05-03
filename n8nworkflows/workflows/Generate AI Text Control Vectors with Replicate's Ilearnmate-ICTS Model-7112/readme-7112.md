Generate AI Text Control Vectors with Replicate's Ilearnmate-ICTS Model

https://n8nworkflows.xyz/workflows/generate-ai-text-control-vectors-with-replicate-s-ilearnmate-icts-model-7112


# Generate AI Text Control Vectors with Replicate's Ilearnmate-ICTS Model

### 1. Workflow Overview

This n8n workflow, titled **"Spuuntries Ilearnmate Icts AI Generator"**, integrates with Replicate’s **spuuntries/ilearnmate-icts** AI model to generate text control vectors or other AI-generated content. It is designed to send a prediction request to Replicate’s API, poll for the prediction status until completion, then process and output the final results.

The workflow is logically divided into the following blocks:

- **1.1 Trigger and Initialization:** Starts the workflow manually and sets the required API key.
- **1.2 Prediction Creation:** Sends a request to Replicate to create a prediction job with specific parameters.
- **1.3 Prediction Status Polling:** Waits and repeatedly checks the prediction status until it is complete.
- **1.4 Result Processing:** Processes the final prediction output and formats it for further use or inspection.

### 2. Block-by-Block Analysis

---

#### 2.1 Trigger and Initialization

**Overview:**  
This block initiates the workflow via manual execution and sets the Replicate API key used for authentication in subsequent API calls.

**Nodes Involved:**  
- On clicking 'execute'  
- Set API Key

**Node Details:**

- **On clicking 'execute'**  
  - Type: Manual Trigger  
  - Role: Initiates the workflow when the user manually clicks "execute" in n8n.  
  - Configuration: Default manual trigger, no parameters.  
  - Input: None  
  - Output: Triggers the "Set API Key" node.  
  - Failure cases: None expected unless workflow execution is interrupted.

- **Set API Key**  
  - Type: Set  
  - Role: Stores the Replicate API key in a workflow variable for use in HTTP requests.  
  - Configuration: Assigns a string variable named `replicate_api_key` with the placeholder value `"YOUR_REPLICATE_API_KEY"`.  
  - Input: Trigger from manual execution  
  - Output: Passes the API key to the "Create Prediction" node.  
  - Failure cases: Missing or invalid API key will cause authentication errors downstream.

---

#### 2.2 Prediction Creation

**Overview:**  
Creates a new prediction request on the Replicate platform using the specified model version and input parameters.

**Nodes Involved:**  
- Create Prediction  
- Extract Prediction ID

**Node Details:**

- **Create Prediction**  
  - Type: HTTP Request  
  - Role: Calls Replicate API to create a prediction job.  
  - Configuration:  
    - Method: POST  
    - URL: `https://api.replicate.com/v1/predictions`  
    - Headers: Authorization with Bearer token from API key, Content-Type as application/json  
    - Body: JSON specifying the model version `e3e67a82a16a46b5ce0b4ffb2c542eaddddf8f825e9ea8c308eaa166debad6c1` and inputs `{ "seed": 1, "num_examples_per_side": 3 }`  
    - Timeout: 60 seconds  
  - Input: API key from "Set API Key"  
  - Output: JSON response containing prediction details (including prediction id and status)  
  - Failure cases:  
    - Authentication errors (invalid API key)  
    - Timeout if API is slow to respond  
    - Invalid input parameters causing API rejection

- **Extract Prediction ID**  
  - Type: Code  
  - Role: Extracts the prediction ID and initial status from the API response to use for status polling.  
  - Configuration: JavaScript code that reads prediction ID and status from input JSON, constructs a URL for polling.  
  - Input: Response from "Create Prediction"  
  - Output: Object containing `predictionId`, `status`, and `predictionUrl` for polling  
  - Failure cases: Missing or malformed prediction response, causing runtime exceptions in code.

---

#### 2.3 Prediction Status Polling

**Overview:**  
Periodically waits and checks the prediction status by querying the Replicate API until the prediction is marked as "succeeded."

**Nodes Involved:**  
- Wait  
- Check Prediction Status  
- Check If Complete

**Node Details:**

- **Wait**  
  - Type: Wait  
  - Role: Pauses workflow execution for 2 seconds before polling again to avoid overwhelming API.  
  - Configuration: Wait time of 2 seconds  
  - Input: Prediction ID and URL from "Extract Prediction ID" or "Check If Complete"  
  - Output: Triggers "Check Prediction Status"  
  - Failure cases: None expected unless workflow interrupted.

- **Check Prediction Status**  
  - Type: HTTP Request  
  - Role: Requests the current status of the prediction from the Replicate API.  
  - Configuration:  
    - Method: GET  
    - URL: Dynamic URL from previous node's `predictionUrl`  
    - Headers: Authorization with Bearer token  
  - Input: Prediction URL and API key  
  - Output: Updated prediction status and metadata  
  - Failure cases: Network errors, authentication failure, invalid URL, API downtime.

- **Check If Complete**  
  - Type: If  
  - Role: Checks if the prediction status equals "succeeded."  
  - Configuration: Boolean condition comparing `$json.status` with string `"succeeded"`  
  - Input: Response from "Check Prediction Status"  
  - Output:  
    - True branch: triggers "Process Result"  
    - False branch: loops back to "Wait" to continue polling  
  - Failure cases: Unexpected statuses, missing status field.

---

#### 2.4 Result Processing

**Overview:**  
Processes the completed prediction data to extract relevant outputs and metadata for further use or export.

**Nodes Involved:**  
- Process Result

**Node Details:**

- **Process Result**  
  - Type: Code  
  - Role: Parses the final prediction result, extracts status, output, metrics, timestamps, and adds static model info.  
  - Configuration: JavaScript code that returns a structured object with fields `status`, `output`, `metrics`, `created_at`, `completed_at`, `model`, and `other_url`.  
  - Input: Final JSON prediction data from "Check If Complete" (true branch)  
  - Output: Formatted result object for downstream usage or logging  
  - Failure cases: Unexpected or missing fields in the prediction output.

---

#### 2.5 Documentation Node (Sticky Note)

**Overview:**  
Provides in-workflow documentation and setup instructions for users and maintainers.

**Nodes Involved:**  
- Sticky Note

**Node Details:**

- **Sticky Note**  
  - Type: Sticky Note  
  - Role: Displays a detailed note describing the workflow purpose, setup steps, and model details.  
  - Content Highlights:  
    - Workflow uses spuuntries/ilearnmate-icts model  
    - Setup instructions include adding API key, configuring inputs, and running the workflow  
    - Model type: Other Generation  
    - Provider: spuuntries  
    - No required input fields specified  
  - Input/Output: None; informational only.

---

### 3. Summary Table

| Node Name              | Node Type           | Functional Role                        | Input Node(s)                  | Output Node(s)                  | Sticky Note                                                                                   |
|------------------------|---------------------|-------------------------------------|-------------------------------|--------------------------------|----------------------------------------------------------------------------------------------|
| On clicking 'execute'   | Manual Trigger      | Workflow start trigger               | None                          | Set API Key                    |                                                                                              |
| Set API Key            | Set                 | Store Replicate API key              | On clicking 'execute'          | Create Prediction              |                                                                                              |
| Create Prediction      | HTTP Request        | Create prediction job on Replicate  | Set API Key                   | Extract Prediction ID          |                                                                                              |
| Extract Prediction ID  | Code                | Extract prediction ID and status    | Create Prediction             | Wait                          |                                                                                              |
| Wait                   | Wait                | Pause before polling again           | Extract Prediction ID / Check If Complete (false) | Check Prediction Status        |                                                                                              |
| Check Prediction Status| HTTP Request        | Query prediction status              | Wait                         | Check If Complete             |                                                                                              |
| Check If Complete      | If                  | Branch based on prediction completion| Check Prediction Status       | Process Result / Wait          |                                                                                              |
| Process Result         | Code                | Process and format final output      | Check If Complete (true)      | None                         |                                                                                              |
| Sticky Note            | Sticky Note         | Documentation and setup instructions | None                          | None                         | ## Spuuntries Ilearnmate Icts AI Generator<br>This workflow uses the **spuuntries/ilearnmate-icts** model from Replicate to generate other content.<br>### Setup<br>1. Add your Replicate API key<br>2. Configure the input parameters<br>3. Run the workflow<br>### Model Details<br>- **Type**: Other Generation<br>- **Provider**: spuuntries<br>- **Required Fields**: None |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node**  
   - Name: `On clicking 'execute'`  
   - Purpose: Start the workflow manually.

2. **Add a Set Node**  
   - Name: `Set API Key`  
   - Purpose: Store your Replicate API key.  
   - Configuration:  
     - Add a string field named `replicate_api_key` with your actual Replicate API key (replace `"YOUR_REPLICATE_API_KEY"`).

3. **Create an HTTP Request Node**  
   - Name: `Create Prediction`  
   - Purpose: Send a POST request to Replicate API to create a prediction.  
   - Configuration:  
     - Method: POST  
     - URL: `https://api.replicate.com/v1/predictions`  
     - Authentication: HTTP Header Authentication  
     - Headers:  
       - Authorization: `Bearer {{$node["Set API Key"].json["replicate_api_key"]}}`  
       - Content-Type: `application/json`  
     - Body Content Type: JSON  
     - JSON Body:  
       ```json
       {
         "version": "e3e67a82a16a46b5ce0b4ffb2c542eaddddf8f825e9ea8c308eaa166debad6c1",
         "input": {
           "seed": 1,
           "num_examples_per_side": 3
         }
       }
       ```  
     - Timeout: 60 seconds

4. **Add a Code Node**  
   - Name: `Extract Prediction ID`  
   - Purpose: Extract the prediction ID and initial status from API response.  
   - Configuration (JavaScript):  
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

5. **Add a Wait Node**  
   - Name: `Wait`  
   - Purpose: Pause workflow for 2 seconds before polling again.  
   - Configuration:  
     - Time Unit: seconds  
     - Amount: 2

6. **Add an HTTP Request Node**  
   - Name: `Check Prediction Status`  
   - Purpose: Poll the prediction status URL.  
   - Configuration:  
     - Method: GET  
     - URL: `{{$json["predictionUrl"]}}` (dynamic from previous node)  
     - Authentication: HTTP Header Authentication  
     - Headers:  
       - Authorization: `Bearer {{$node["Set API Key"].json["replicate_api_key"]}}`

7. **Add an If Node**  
   - Name: `Check If Complete`  
   - Purpose: Check if prediction status is `"succeeded"`.  
   - Configuration:  
     - Condition: Boolean  
     - Expression: `{{$json["status"]}} === "succeeded"`

8. **Add a Code Node**  
   - Name: `Process Result`  
   - Purpose: Extract and format the final prediction data.  
   - Configuration (JavaScript):  
     ```js
     const result = $input.item.json;

     return {
       status: result.status,
       output: result.output,
       metrics: result.metrics,
       created_at: result.created_at,
       completed_at: result.completed_at,
       model: 'spuuntries/ilearnmate-icts',
       other_url: result.output
     };
     ```

9. **Wire the Nodes in the Following Order:**  
   - `On clicking 'execute'` → `Set API Key`  
   - `Set API Key` → `Create Prediction`  
   - `Create Prediction` → `Extract Prediction ID`  
   - `Extract Prediction ID` → `Wait`  
   - `Wait` → `Check Prediction Status`  
   - `Check Prediction Status` → `Check If Complete`  
   - `Check If Complete`  
     - True branch → `Process Result`  
     - False branch → `Wait` (loop back for polling)

10. **(Optional) Add a Sticky Note Node**  
    - Include documentation about the workflow purpose, setup, and model details for users.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                | Context or Link                                                                                   |
|---------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| The workflow uses the **spuuntries/ilearnmate-icts** model version `e3e67a82a16a46b5ce0b4ffb2c542eaddddf8f825e9ea8c308eaa166debad6c1`.      | Important for ensuring the correct model version is targeted in the API call.                    |
| Ensure your Replicate API key has necessary permissions and is kept secret.                                                                  | Avoid exposing API keys in public repositories or logs.                                         |
| The polling interval is set to 2 seconds to balance API load and responsiveness; adjust if needed based on API rate limits or response time.| API rate limits may cause errors if polling is too frequent or too many parallel workflows run.  |
| For more info on Replicate API, see: https://replicate.com/docs/reference/http                                                                 | Official API documentation for further customization or troubleshooting.                         |

---

**Disclaimer:**  
The text provided stems exclusively from an n8n automated workflow. It fully complies with current content policies and contains no illegal, offensive, or protected elements. All data processed are legal and publicly accessible.