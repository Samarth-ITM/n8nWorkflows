Generate 3D Mesh Models from Images with Part Crafter AI on Replicate

https://n8nworkflows.xyz/workflows/generate-3d-mesh-models-from-images-with-part-crafter-ai-on-replicate-6878


# Generate 3D Mesh Models from Images with Part Crafter AI on Replicate

### 1. Workflow Overview

This workflow, titled **Fire Part Crafter Image Generator**, automates the process of generating 3D mesh model images through the **fire/part-crafter** AI model hosted on Replicate. It is designed for users who want to submit an input image and receive a generated 3D mesh image divided into parts with customizable parameters. The workflow handles API authentication, submission of a prediction request, polling for prediction completion, and processing the results.

The workflow can be logically divided into the following blocks:

- **1.1 Input Trigger and API Key Setup**: Manual execution triggers the workflow, and the Replicate API key is set.
- **1.2 Prediction Creation**: Submits the image generation request to the Replicate API with parameters.
- **1.3 Prediction Status Polling**: Periodically checks the prediction status until completion.
- **1.4 Result Processing**: Processes and formats the final output once the prediction succeeds.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Trigger and API Key Setup

- **Overview**: This block waits for manual execution and sets up the API key required for authentication with the Replicate API.
  
- **Nodes Involved**:  
  - On clicking 'execute'  
  - Set API Key  

- **Node Details**:

  - **On clicking 'execute'**  
    - *Type*: Manual Trigger  
    - *Role*: Initiates the workflow manually by user interaction.  
    - *Config*: No parameters, simply triggers the workflow execution.  
    - *Inputs*: None  
    - *Outputs*: Triggers "Set API Key" node.  
    - *Edge cases*: None specific; user must trigger manually.

  - **Set API Key**  
    - *Type*: Set (data manipulation)  
    - *Role*: Assigns the Replicate API key as a workflow variable for later use in authentication headers.  
    - *Config*: Sets a string variable `replicate_api_key` with the placeholder `"YOUR_REPLICATE_API_KEY"`.  
    - *Inputs*: Receives trigger from manual trigger node.  
    - *Outputs*: Passes data to "Create Prediction" node.  
    - *Edge cases*: The placeholder must be replaced with a valid API key or authentication will fail during HTTP requests.

---

#### 1.2 Prediction Creation

- **Overview**: This block sends a POST request to Replicate’s API to create a new prediction job using the fire/part-crafter model with specified input parameters.

- **Nodes Involved**:  
  - Create Prediction  
  - Extract Prediction ID  

- **Node Details**:

  - **Create Prediction**  
    - *Type*: HTTP Request  
    - *Role*: Issues a prediction creation request to Replicate API.  
    - *Config*:  
      - URL: `https://api.replicate.com/v1/predictions`  
      - Method: POST  
      - Headers:  
        - Authorization: Bearer token from `replicate_api_key`  
        - Content-Type: application/json  
      - Body (JSON):  
        ```json
        {
          "version": "a8fe4d91050b7d6497dd6676506a4105a3903b7287353681f52f3be0bb01d67f",
          "input": {
            "image": "https://example.com/input.image",
            "num_parts": 16,
            "guidance_scale": 7,
            "num_inference_steps": 50
          }
        }
        ```  
      - Timeout: 60 seconds  
      - Authentication: HTTP Header with Bearer token from Set API Key node  
    - *Inputs*: Receives API key from "Set API Key" node.  
    - *Outputs*: Passes API response (prediction object) to "Extract Prediction ID".  
    - *Edge cases*:  
      - Invalid API key leads to auth failure (401).  
      - Network timeout or API rate limits can cause request failure.  
      - Input image URL must be accessible; otherwise, prediction may fail or error out.

  - **Extract Prediction ID**  
    - *Type*: Code (JavaScript)  
    - *Role*: Parses the response from "Create Prediction" to extract the prediction ID, its initial status, and constructs a URL for polling.  
    - *Config*:  
      - Extracts `id` and `status` from response JSON.  
      - Returns an object with `predictionId`, `status`, and `predictionUrl` for subsequent polling.  
    - *Inputs*: Prediction JSON from "Create Prediction".  
    - *Outputs*: Sends extracted fields to "Wait" node.  
    - *Edge cases*:  
      - If response does not contain expected fields, code may fail.  
      - Network or JSON parsing errors.

---

#### 1.3 Prediction Status Polling

- **Overview**: This block waits for a short interval, then queries the Replicate API repeatedly to check if the prediction is complete.

- **Nodes Involved**:  
  - Wait  
  - Check Prediction Status  
  - Check If Complete  

- **Node Details**:

  - **Wait**  
    - *Type*: Wait  
    - *Role*: Pauses the workflow for 2 seconds to avoid rapid polling.  
    - *Config*: Wait duration set to 2 seconds.  
    - *Inputs*: Receives from "Extract Prediction ID" or "Check If Complete" (if not done).  
    - *Outputs*: Triggers "Check Prediction Status".  
    - *Edge cases*: None significant; just delay.

  - **Check Prediction Status**  
    - *Type*: HTTP Request  
    - *Role*: Polls the Replicate API for the current status of the prediction.  
    - *Config*:  
      - URL: Dynamic, constructed from extracted `predictionUrl`.  
      - Method: GET (default)  
      - Headers: Authorization Bearer token from API key.  
      - Authentication: HTTP Header with Bearer token.  
    - *Inputs*: Receives from "Wait".  
    - *Outputs*: Passes API response to "Check If Complete".  
    - *Edge cases*:  
      - API errors or timeouts.  
      - Prediction might enter error or failed state.

  - **Check If Complete**  
    - *Type*: If (Boolean condition)  
    - *Role*: Checks if the prediction status equals `"succeeded"`.  
    - *Config*: Condition compares `$json.status === 'succeeded'`.  
    - *Inputs*: Takes JSON from "Check Prediction Status".  
    - *Outputs*:  
      - If true: sends to "Process Result".  
      - If false: loops back to "Wait" to continue polling.  
    - *Edge cases*:  
      - Other statuses like `failed` or `canceled` are not handled explicitly and will cause indefinite polling.  
      - Could be improved by handling error states.

---

#### 1.4 Result Processing

- **Overview**: Once the prediction succeeds, this block formats and returns the relevant output data.

- **Nodes Involved**:  
  - Process Result  

- **Node Details**:

  - **Process Result**  
    - *Type*: Code (JavaScript)  
    - *Role*: Extracts and formats key information from the successful prediction result.  
    - *Config*:  
      - Outputs fields: status, output (image URL), metrics, created_at, completed_at, fixed model name string, and image URL.  
      - Returns a simplified JSON object with relevant details for further use or display.  
    - *Inputs*: Receives JSON from "Check If Complete" success branch.  
    - *Outputs*: Final output of the workflow.  
    - *Edge cases*: If output field is missing, result may be incomplete.

---

### 3. Summary Table

| Node Name              | Node Type            | Functional Role                          | Input Node(s)             | Output Node(s)            | Sticky Note                                                                                                         |
|------------------------|----------------------|----------------------------------------|---------------------------|---------------------------|---------------------------------------------------------------------------------------------------------------------|
| On clicking 'execute'   | Manual Trigger       | Workflow start trigger                  | -                         | Set API Key               |                                                                                                                     |
| Set API Key            | Set                  | Sets Replicate API key variable        | On clicking 'execute'      | Create Prediction         |                                                                                                                     |
| Create Prediction      | HTTP Request         | Creates prediction request to Replicate| Set API Key                | Extract Prediction ID     |                                                                                                                     |
| Extract Prediction ID  | Code                 | Extracts prediction ID and status      | Create Prediction          | Wait                      |                                                                                                                     |
| Wait                   | Wait                 | Delays polling by 2 seconds             | Extract Prediction ID / Check If Complete (false branch) | Check Prediction Status |                                                                                                                     |
| Check Prediction Status| HTTP Request         | Polls prediction status from API       | Wait                      | Check If Complete         |                                                                                                                     |
| Check If Complete      | If                   | Checks if prediction succeeded         | Check Prediction Status    | Process Result / Wait     |                                                                                                                     |
| Process Result         | Code                 | Formats and outputs final prediction   | Check If Complete (true)   | -                         |                                                                                                                     |
| Sticky Note            | Sticky Note          | Provides workflow description and setup| -                         | -                         | ## Fire Part Crafter Image Generator<br><br>This workflow uses the **fire/part-crafter** model from Replicate to generate image content.<br><br>### Setup<br>1. Add your Replicate API key<br>2. Configure the input parameters<br>3. Run the workflow<br><br>### Model Details<br>- **Type**: Image Generation<br>- **Provider**: fire<br>- **Required Fields**: image |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node** named `On clicking 'execute'`. No special configuration needed.

2. **Add a Set node** named `Set API Key`:  
   - Add a string field `replicate_api_key`.  
   - Set its value to your Replicate API key as a string (replace `"YOUR_REPLICATE_API_KEY"`).

3. **Connect** `On clicking 'execute'` → `Set API Key`.

4. **Add an HTTP Request node** named `Create Prediction`:  
   - Method: POST  
   - URL: `https://api.replicate.com/v1/predictions`  
   - Authentication: Generic Credential → HTTP Header Authentication  
   - Header Parameters:  
     - `Authorization`: `Bearer {{$node["Set API Key"].json["replicate_api_key"]}}`  
     - `Content-Type`: `application/json`  
   - Body Content (JSON, set "Specify Body" to JSON):  
     ```json
     {
       "version": "a8fe4d91050b7d6497dd6676506a4105a3903b7287353681f52f3be0bb01d67f",
       "input": {
         "image": "https://example.com/input.image",
         "num_parts": 16,
         "guidance_scale": 7,
         "num_inference_steps": 50
       }
     }
     ```  
   - Timeout: 60000 ms

5. **Connect** `Set API Key` → `Create Prediction`.

6. **Add a Code node** named `Extract Prediction ID`:  
   - Mode: Run Once For Each Item  
   - Code (JavaScript):  
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
   
7. **Connect** `Create Prediction` → `Extract Prediction ID`.

8. **Add a Wait node** named `Wait`:  
   - Duration: 2 seconds

9. **Connect** `Extract Prediction ID` → `Wait`.

10. **Add an HTTP Request node** named `Check Prediction Status`:  
    - Method: GET (default)  
    - URL: Expression: `{{$json["predictionUrl"]}}`  
    - Authentication: Generic Credential → HTTP Header Authentication  
    - Header Parameters:  
      - `Authorization`: `Bearer {{$node["Set API Key"].json["replicate_api_key"]}}`

11. **Connect** `Wait` → `Check Prediction Status`.

12. **Add an If node** named `Check If Complete`:  
    - Condition Type: Boolean  
    - Condition: Check if `{{$json["status"]}}` equals `"succeeded"`

13. **Connect** `Check Prediction Status` → `Check If Complete`.

14. **Add a Code node** named `Process Result`:  
    - Mode: Run Once For Each Item  
    - Code (JavaScript):  
      ```javascript
      const result = $input.item.json;

      return {
        status: result.status,
        output: result.output,
        metrics: result.metrics,
        created_at: result.created_at,
        completed_at: result.completed_at,
        model: 'fire/part-crafter',
        image_url: result.output
      };
      ```

15. **Connect** `Check If Complete` (true branch) → `Process Result`.

16. **Connect** `Check If Complete` (false branch) → `Wait` (to continue polling).

17. **Add a Sticky Note** to the canvas with the following content:  
    ```
    ## Fire Part Crafter Image Generator

    This workflow uses the **fire/part-crafter** model from Replicate to generate image content.

    ### Setup
    1. Add your Replicate API key
    2. Configure the input parameters
    3. Run the workflow

    ### Model Details
    - Type: Image Generation
    - Provider: fire
    - Required Fields: image
    ```

18. **Save and activate** the workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                       |
|-----------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------|
| This workflow uses the Replicate API with the model version ID: `a8fe4d91050b7d6497dd6676506a4105a3903b7287353681f52f3be0bb01d67f` | Replicate Model Version ID for fire/part-crafter                     |
| Replace `"https://example.com/input.image"` with a publicly accessible image URL for valid input          | Input image URL must be accessible and valid for prediction          |
| The polling mechanism does not handle prediction failure states explicitly and may poll indefinitely       | Consider adding error handling for statuses like `failed` or `canceled` |
| API key must have sufficient permissions and be valid to avoid unauthorized errors                        | Obtain API key from https://replicate.com/account/api-tokens          |
| Model details and API usage documented at https://replicate.com/fire/part-crafter                          | Official model page for further parameter tuning and info            |

---

**Disclaimer:** The provided text exclusively originates from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.