Generate Images with Monexia Nietgoed AI via Replicate

https://n8nworkflows.xyz/workflows/generate-images-with-monexia-nietgoed-ai-via-replicate-7118


# Generate Images with Monexia Nietgoed AI via Replicate

### 1. Workflow Overview

This workflow automates image or content generation using the **Monexia Nietgoed AI** model hosted on Replicate. It is designed for users who want to generate AI-driven outputs by providing a textual prompt, leveraging the Replicate API to handle model inference asynchronously.

The workflow is structured into the following logical blocks:

- **1.1 Input Trigger and API Key Setup:** Manual trigger to start the workflow and setting the Replicate API key securely.
- **1.2 Prediction Creation:** Sending the generation request to Replicate with the configured model version and input parameters.
- **1.3 Prediction Monitoring Loop:** Polling Replicate API to check the status of the asynchronous prediction until completion.
- **1.4 Result Processing:** Extracting and formatting the final generated output once prediction succeeds.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Trigger and API Key Setup

**Overview:**  
This block initiates the workflow manually and configures the Replicate API key needed for authentication in subsequent API calls.

**Nodes Involved:**  
- On clicking 'execute'  
- Set API Key

**Node Details:**

- **On clicking 'execute'**  
  - *Type:* Manual Trigger  
  - *Role:* Starts workflow execution on user command.  
  - *Configuration:* No parameters; simple manual trigger.  
  - *Inputs:* None  
  - *Outputs:* Triggers "Set API Key" node.  
  - *Failure Cases:* None expected unless system-level execution issues occur.

- **Set API Key**  
  - *Type:* Set  
  - *Role:* Assigns the Replicate API key as a workflow variable for use in HTTP requests.  
  - *Configuration:*  
    - Variable `replicate_api_key` set to string `"YOUR_REPLICATE_API_KEY"` (to be replaced by user).  
  - *Inputs:* Manual trigger node output  
  - *Outputs:* Sends data to "Create Prediction" node  
  - *Failure Cases:* Missing or invalid API key will cause authentication failures downstream.

---

#### 1.2 Prediction Creation

**Overview:**  
Sends a POST request to Replicate API to create a new prediction job with the specified model version and input parameters including prompt, seed, width, height, and lora scale.

**Nodes Involved:**  
- Create Prediction  
- Extract Prediction ID

**Node Details:**

- **Create Prediction**  
  - *Type:* HTTP Request  
  - *Role:* Makes authenticated API call to initiate prediction on Replicate.  
  - *Configuration:*  
    - Method: POST  
    - URL: `https://api.replicate.com/v1/predictions`  
    - Headers:  
      - `Authorization`: Bearer token from `replicate_api_key` variable  
      - `Content-Type`: application/json  
    - JSON Body:  
      ```json
      {
        "version": "b4eed8f86100995e0919dcf5e4af4eca54fc43ce72f4fd570e7d211cfe240b3d",
        "input": {
          "prompt": "prompt value",
          "seed": 1,
          "width": 1,
          "height": 1,
          "lora_scale": 1
        }
      }
      ```  
      The `"prompt"` field is a placeholder and must be replaced or parameterized by the user for meaningful results.  
    - Timeout: 60 seconds  
  - *Inputs:* API key node output  
  - *Outputs:* Sends response JSON to "Extract Prediction ID" node  
  - *Failure Cases:*  
    - HTTP errors (e.g., 401 Unauthorized if API key invalid)  
    - Timeout or network issues  
    - Invalid input parameter format or missing required fields

- **Extract Prediction ID**  
  - *Type:* Code (JavaScript)  
  - *Role:* Parses the initial prediction response to extract the prediction ID and status for polling.  
  - *Configuration:*  
    - Extracts `id` and `status` from API response JSON  
    - Constructs prediction URL for status checks: `https://api.replicate.com/v1/predictions/{id}`  
  - *Inputs:* Output from "Create Prediction"  
  - *Outputs:* Passes extracted predictionId, status, and URL to next node  
  - *Failure Cases:*  
    - Missing or malformed response JSON may cause code errors  
    - Unexpected data format from API

---

#### 1.3 Prediction Monitoring Loop

**Overview:**  
Implements a polling loop that waits a fixed interval and checks the prediction status repeatedly until the prediction completes successfully.

**Nodes Involved:**  
- Wait  
- Check Prediction Status  
- Check If Complete

**Node Details:**

- **Wait**  
  - *Type:* Wait  
  - *Role:* Pauses workflow execution for 2 seconds between status polls to avoid rate limiting.  
  - *Configuration:* Wait 2 seconds  
  - *Inputs:* Receives prediction URL and status from "Extract Prediction ID" or "Check If Complete" nodes  
  - *Outputs:* Triggers "Check Prediction Status"  
  - *Failure Cases:* None expected unless node execution errors.

- **Check Prediction Status**  
  - *Type:* HTTP Request  
  - *Role:* Queries Replicate API for current prediction status using the prediction URL.  
  - *Configuration:*  
    - Method: GET (implicit)  
    - URL: from `predictionUrl` variable  
    - Headers: Authorization Bearer token from stored API key  
  - *Inputs:* Output from "Wait" node  
  - *Outputs:* Sends status JSON to "Check If Complete" node  
  - *Failure Cases:*  
    - HTTP errors (e.g., 401 Unauthorized)  
    - Network or timeout failures

- **Check If Complete**  
  - *Type:* If (Boolean condition)  
  - *Role:* Determines if prediction status is `"succeeded"` to proceed or continue polling.  
  - *Configuration:* Checks if `$json.status === "succeeded"`  
  - *Inputs:* "Check Prediction Status" output  
  - *Outputs:*  
    - If true: proceeds to "Process Result"  
    - If false: loops back to "Wait" node  
  - *Failure Cases:*  
    - Unexpected or missing status field could cause logical errors.

---

#### 1.4 Result Processing

**Overview:**  
Processes the final prediction result, extracting useful metadata and output URLs for downstream usage or presentation.

**Nodes Involved:**  
- Process Result

**Node Details:**

- **Process Result**  
  - *Type:* Code (JavaScript)  
  - *Role:* Parses the completed prediction JSON to extract: status, output (image/content URLs), metrics, timestamps, and model info.  
  - *Configuration:* Returns a simplified JSON object with relevant fields.  
  - *Inputs:* Output from "Check If Complete" (success branch)  
  - *Outputs:* Final structured data containing prediction results  
  - *Failure Cases:*  
    - Missing expected fields in API response may cause errors  
    - Handling of multiple output formats or empty outputs is not explicitly defined

---

### 3. Summary Table

| Node Name              | Node Type          | Functional Role                  | Input Node(s)            | Output Node(s)           | Sticky Note                                                                                                                                                                  |
|------------------------|--------------------|--------------------------------|--------------------------|--------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| On clicking 'execute'   | Manual Trigger     | Start workflow execution        | None                     | Set API Key              |                                                                                                                                                                              |
| Set API Key            | Set                | Store Replicate API key         | On clicking 'execute'     | Create Prediction        |                                                                                                                                                                              |
| Create Prediction      | HTTP Request       | Send prediction creation request| Set API Key              | Extract Prediction ID    |                                                                                                                                                                              |
| Extract Prediction ID  | Code               | Extract prediction ID and status| Create Prediction        | Wait                     |                                                                                                                                                                              |
| Wait                   | Wait               | Pause between status checks     | Extract Prediction ID, Check If Complete (false) | Check Prediction Status |                                                                                                                                                                              |
| Check Prediction Status| HTTP Request       | Poll prediction status          | Wait                     | Check If Complete        |                                                                                                                                                                              |
| Check If Complete      | If (Boolean)       | Check if prediction succeeded   | Check Prediction Status  | Process Result (true), Wait (false) |                                                                                                                                                                              |
| Process Result         | Code               | Extract final output data       | Check If Complete (true) | None                     |                                                                                                                                                                              |
| Sticky Note            | Sticky Note        | Documentation note              | None                     | None                     | ## Monexia Nietgoed AI Generator<br><br>This workflow uses the **monexia/nietgoed** model from Replicate to generate other content.<br><br>### Setup<br>1. Add your Replicate API key<br>2. Configure the input parameters<br>3. Run the workflow<br><br>### Model Details<br>- **Type**: Other Generation<br>- **Provider**: monexia<br>- **Required Fields**: prompt |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add Manual Trigger node**  
   - Name: `On clicking 'execute'`  
   - No additional configuration needed.

3. **Add Set node**  
   - Name: `Set API Key`  
   - Add a string field named `replicate_api_key` with your Replicate API key value (replace `"YOUR_REPLICATE_API_KEY"`).

4. **Connect `On clicking 'execute'` → `Set API Key`.**

5. **Add HTTP Request node**  
   - Name: `Create Prediction`  
   - Method: POST  
   - URL: `https://api.replicate.com/v1/predictions`  
   - Authentication: Generic HTTP Header Auth  
     - Header Name: `Authorization`  
     - Header Value: `Bearer {{$node["Set API Key"].json["replicate_api_key"]}}`  
   - Headers:  
     - `Content-Type`: `application/json`  
   - Body Content Type: JSON  
   - JSON Body:  
     ```json
     {
       "version": "b4eed8f86100995e0919dcf5e4af4eca54fc43ce72f4fd570e7d211cfe240b3d",
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

6. **Connect `Set API Key` → `Create Prediction`.**

7. **Add Code node**  
   - Name: `Extract Prediction ID`  
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

8. **Connect `Create Prediction` → `Extract Prediction ID`.**

9. **Add Wait node**  
   - Name: `Wait`  
   - Wait Time: 2 seconds

10. **Connect `Extract Prediction ID` → `Wait`.**

11. **Add HTTP Request node**  
    - Name: `Check Prediction Status`  
    - Method: GET (default)  
    - URL: `={{ $json.predictionUrl }}` (use expression to get URL)  
    - Authentication: Generic HTTP Header Auth  
      - Header Name: `Authorization`  
      - Header Value: `Bearer {{$node["Set API Key"].json["replicate_api_key"]}}`

12. **Connect `Wait` → `Check Prediction Status`.**

13. **Add If node**  
    - Name: `Check If Complete`  
    - Condition: Boolean  
      - Expression: `{{$json.status === "succeeded"}}`

14. **Connect `Check Prediction Status` → `Check If Complete`.**

15. **Add Code node**  
    - Name: `Process Result`  
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
        model: 'monexia/nietgoed',
        other_url: result.output
      };
      ```

16. **Connect `Check If Complete` (true output) → `Process Result`.**

17. **Connect `Check If Complete` (false output) → `Wait`** (to continue polling).

18. **Optionally, add a Sticky Note node for documentation**  
    - Content:  
      ```
      ## Monexia Nietgoed AI Generator

      This workflow uses the **monexia/nietgoed** model from Replicate to generate other content.

      ### Setup
      1. Add your Replicate API key
      2. Configure the input parameters
      3. Run the workflow

      ### Model Details
      - Type: Other Generation
      - Provider: monexia
      - Required Fields: prompt
      ```

19. **Save and activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                                 | Context or Link                                                    |
|--------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------|
| The model version used is `b4eed8f86100995e0919dcf5e4af4eca54fc43ce72f4fd570e7d211cfe240b3d` (Monexia Nietgoed AI) | Replicate model version identifier                                |
| The workflow requires a valid Replicate API key with access permissions to the monexia/nietgoed model.         | https://replicate.com/                                            |
| Prompt input must be replaced or dynamically set for meaningful image/content generation results.             | User must customize prompt field in "Create Prediction" node     |
| Polling interval is set to 2 seconds to balance responsiveness and rate limiting on Replicate API.             | Adjust as needed for API rate limits                              |

---

**Disclaimer:**  
The provided text is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects applicable content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and publicly accessible.