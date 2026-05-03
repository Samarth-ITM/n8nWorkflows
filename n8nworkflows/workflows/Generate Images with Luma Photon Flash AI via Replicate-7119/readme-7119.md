Generate Images with Luma Photon Flash AI via Replicate

https://n8nworkflows.xyz/workflows/generate-images-with-luma-photon-flash-ai-via-replicate-7119


# Generate Images with Luma Photon Flash AI via Replicate

### 1. Workflow Overview

This workflow, titled **Luma Photon Flash Image Generator**, automates the generation of images using the **luma/photon-flash** AI model hosted on Replicate. Its primary use case is to take a textual prompt and produce an AI-generated image based on that prompt by interacting with the Replicate API. The workflow is designed to be triggered manually and handles the full process of submitting a prediction request, polling for completion, and then processing the resulting image URL.

The logical structure of the workflow can be grouped into the following blocks:

- **1.1 Manual Trigger and Setup:** Starts the workflow manually and sets the API key required for authentication with Replicate.
- **1.2 Prediction Creation:** Sends a request to Replicate to create an image generation prediction based on input parameters.
- **1.3 Prediction Monitoring:** Polls the Replicate API repeatedly to check the status of the prediction until it completes.
- **1.4 Result Processing:** Once the prediction is successful, processes and extracts useful information such as the image URL and metadata.

---

### 2. Block-by-Block Analysis

#### 1.1 Manual Trigger and Setup

**Overview:**  
This block initiates the workflow when manually executed and sets the Replicate API key for authentication in subsequent HTTP requests.

**Nodes Involved:**  
- On clicking 'execute' (Manual Trigger)  
- Set API Key (Set node)

**Node Details:**  

- **On clicking 'execute'**  
  - Type: Manual Trigger  
  - Role: Starts the workflow on demand through user interaction.  
  - Configuration: Default manual trigger with no parameters.  
  - Inputs: None  
  - Outputs: Connects to the "Set API Key" node.  
  - Failure cases: None expected; user must trigger manually.

- **Set API Key**  
  - Type: Set  
  - Role: Stores the Replicate API key as a workflow variable for reuse.  
  - Configuration: Assigns a string variable `replicate_api_key` with placeholder `"YOUR_REPLICATE_API_KEY"` which must be replaced by the user’s actual API key.  
  - Inputs: From manual trigger node.  
  - Outputs: Connects to "Create Prediction".  
  - Failure cases: If the API key is missing or incorrect downstream calls will fail with authorization errors.

---

#### 1.2 Prediction Creation

**Overview:**  
This block creates a new prediction request on Replicate by sending a POST request with the model version and prompt parameters.

**Nodes Involved:**  
- Create Prediction (HTTP Request)  
- Extract Prediction ID (Code)

**Node Details:**  

- **Create Prediction**  
  - Type: HTTP Request  
  - Role: Sends a POST request to `https://api.replicate.com/v1/predictions` to create an image generation job.  
  - Configuration:  
    - Method: POST  
    - URL: Replicate predictions endpoint  
    - Body: JSON with model version `4235af608e50dac14e9244198cef089049efbd83ba05f2aa4e271076a6f613ee` (specific to luma/photon-flash), and input parameters:  
      - `prompt`: `"prompt value"` (should be replaced or parameterized for dynamic input)  
      - `seed`: 1 (fixed seed for reproducibility)  
      - `image_reference_weight` and `style_reference_weight`: 0.85 (weights for image and style influence)  
    - Headers include:  
      - Authorization: Bearer token from the `replicate_api_key` set earlier  
      - Content-Type: application/json  
    - Timeout: 60 seconds  
  - Inputs: From "Set API Key"  
  - Outputs: Connects to "Extract Prediction ID"  
  - Failure cases: HTTP errors (network, 4xx/5xx), authentication errors, invalid or missing prompt, timeout.

- **Extract Prediction ID**  
  - Type: Code (JavaScript)  
  - Role: Parses the response from prediction creation to extract the prediction ID and status, preparing data for polling.  
  - Configuration:  
    - Extracts `id` and `status` from JSON response  
    - Constructs a polling URL using the prediction ID  
  - Input: From "Create Prediction"  
  - Output: Connects to "Wait"  
  - Failure cases: If response JSON structure changes or is empty, extraction will fail.

---

#### 1.3 Prediction Monitoring

**Overview:**  
This block polls the Replicate API for the prediction status every 2 seconds until the prediction is marked as "succeeded."

**Nodes Involved:**  
- Wait (Wait)  
- Check Prediction Status (HTTP Request)  
- Check If Complete (If)

**Node Details:**  

- **Wait**  
  - Type: Wait  
  - Role: Pauses workflow execution for 2 seconds between status checks to avoid API rate limits.  
  - Configuration: Wait for 2 seconds  
  - Input: From "Extract Prediction ID" (initial wait) and from "Check If Complete" (if prediction not complete)  
  - Output: Connects to "Check Prediction Status"  
  - Failure cases: Minimal, but potential workflow timeout if prediction takes very long.

- **Check Prediction Status**  
  - Type: HTTP Request  
  - Role: Sends a GET request to the prediction URL to check current status.  
  - Configuration:  
    - URL is dynamically set from JSON field `predictionUrl`  
    - Headers include authorization using previously set API key  
    - Method: GET (default)  
  - Input: From "Wait"  
  - Output: Connects to "Check If Complete"  
  - Failure cases: Network issues, expired/invalid API key, or if prediction ID no longer valid.

- **Check If Complete**  
  - Type: If  
  - Role: Checks if the prediction status is `"succeeded"` to decide whether to continue polling or proceed to result processing.  
  - Configuration: Boolean condition comparing `$json.status` to `"succeeded"`  
  - Input: From "Check Prediction Status"  
  - Outputs:  
    - If true: Connects to "Process Result"  
    - If false: Connects back to "Wait" to continue polling  
  - Failure cases: If response JSON lacks `status` field or contains unexpected status values.

---

#### 1.4 Result Processing

**Overview:**  
Once the prediction completes successfully, this block processes the returned data, extracting the image URL and relevant metadata for downstream use.

**Nodes Involved:**  
- Process Result (Code)

**Node Details:**  

- **Process Result**  
  - Type: Code (JavaScript)  
  - Role: Extracts and formats the final prediction output including image URL, status, and timestamps.  
  - Configuration:  
    - Extracts fields: `status`, `output` (image URL), `metrics`, `created_at`, and `completed_at`  
    - Adds constant `model` field with value `"luma/photon-flash"`  
    - Returns an object containing the extracted details for further processing or output.  
  - Input: From "Check If Complete" when prediction status is succeeded  
  - Outputs: None connected (end of workflow)  
  - Failure cases: If expected fields are missing or malformed in the prediction result.

---

### 3. Summary Table

| Node Name               | Node Type           | Functional Role                      | Input Node(s)               | Output Node(s)              | Sticky Note                                                                                         |
|-------------------------|---------------------|------------------------------------|----------------------------|-----------------------------|---------------------------------------------------------------------------------------------------|
| On clicking 'execute'    | Manual Trigger      | Initiates workflow manually         | None                       | Set API Key                 |                                                                                                   |
| Set API Key             | Set                 | Stores Replicate API key             | On clicking 'execute'       | Create Prediction            |                                                                                                   |
| Create Prediction        | HTTP Request        | Creates image generation prediction | Set API Key                | Extract Prediction ID        |                                                                                                   |
| Extract Prediction ID    | Code                | Extracts prediction ID and status   | Create Prediction          | Wait                        |                                                                                                   |
| Wait                    | Wait                | Delays between polling attempts     | Extract Prediction ID, Check If Complete (false branch) | Check Prediction Status     |                                                                                                   |
| Check Prediction Status  | HTTP Request        | Polls prediction status             | Wait                       | Check If Complete            |                                                                                                   |
| Check If Complete        | If                  | Checks if prediction succeeded      | Check Prediction Status    | Process Result (true), Wait (false) |                                                                                                   |
| Process Result           | Code                | Processes final prediction output   | Check If Complete          | None                        |                                                                                                   |
| Sticky Note             | Sticky Note         | Documentation note on workflow      | None                       | None                        | ## Luma Photon Flash Image Generator\n\nThis workflow uses the **luma/photon-flash** model from Replicate to generate image content.\n\n### Setup\n1. Add your Replicate API key\n2. Configure the input parameters\n3. Run the workflow\n\n### Model Details\n- **Type**: Image Generation\n- **Provider**: luma\n- **Required Fields**: prompt |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a **Manual Trigger** node named `"On clicking 'execute'"`.  
   - No special configuration needed.

2. **Create Set Node for API Key**  
   - Add a **Set** node named `"Set API Key"`.  
   - Add a string field named `replicate_api_key` with value `"YOUR_REPLICATE_API_KEY"` (replace with your actual API key).  
   - Connect output of Manual Trigger to this node.

3. **Create HTTP Request Node for Prediction Creation**  
   - Add an **HTTP Request** node named `"Create Prediction"`.  
   - Configure:  
     - Method: POST  
     - URL: `https://api.replicate.com/v1/predictions`  
     - Authentication: Generic HTTP Header Auth  
       - Header `Authorization`: `Bearer {{$node["Set API Key"].json["replicate_api_key"]}}`  
       - Header `Content-Type`: `application/json`  
     - Body Content-Type: JSON  
     - JSON Body:  
       ```json
       {
         "version": "4235af608e50dac14e9244198cef089049efbd83ba05f2aa4e271076a6f613ee",
         "input": {
           "prompt": "prompt value",
           "seed": 1,
           "image_reference_weight": 0.85,
           "style_reference_weight": 0.85
         }
       }
       ```  
       - Replace `"prompt value"` with desired prompt or parameterize as needed.  
     - Timeout: 60000 ms (60 seconds)  
   - Connect output of "Set API Key" node to this node.

4. **Create Code Node to Extract Prediction ID**  
   - Add a **Code** node named `"Extract Prediction ID"`.  
   - Use the following JavaScript code:  
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
   - Connect output of "Create Prediction" to this node.

5. **Add Wait Node**  
   - Add a **Wait** node named `"Wait"`.  
   - Configure to wait 2 seconds.  
   - Connect output of "Extract Prediction ID" to this node.

6. **Add HTTP Request Node to Check Prediction Status**  
   - Add an **HTTP Request** node named `"Check Prediction Status"`.  
   - Configure:  
     - Method: GET (default)  
     - URL: `{{$json["predictionUrl"]}}` (dynamic from previous node)  
     - Authentication: Generic HTTP Header Auth  
       - Header `Authorization`: `Bearer {{$node["Set API Key"].json["replicate_api_key"]}}`  
   - Connect output of "Wait" node to this node.

7. **Add If Node to Check Completion**  
   - Add an **If** node named `"Check If Complete"`.  
   - Condition: Boolean  
     - Value 1: `{{$json.status}}`  
     - Operation: equals  
     - Value 2: `succeeded`  
   - Connect output of "Check Prediction Status" to this node.

8. **Add Code Node to Process Result**  
   - Add a **Code** node named `"Process Result"`.  
   - Use the following JavaScript code:  
     ```javascript
     const result = $input.item.json;

     return {
       status: result.status,
       output: result.output,
       metrics: result.metrics,
       created_at: result.created_at,
       completed_at: result.completed_at,
       model: 'luma/photon-flash',
       image_url: result.output
     };
     ```  
   - Connect the **true** output of "Check If Complete" to this node.

9. **Loop Back for Polling**  
   - Connect the **false** output of "Check If Complete" back to the "Wait" node to continue polling.

10. **Add Sticky Note for Documentation** (optional)  
    - Add a **Sticky Note** node with content:  
      ```
      ## Luma Photon Flash Image Generator

      This workflow uses the **luma/photon-flash** model from Replicate to generate image content.

      ### Setup
      1. Add your Replicate API key
      2. Configure the input parameters
      3. Run the workflow

      ### Model Details
      - **Type**: Image Generation
      - **Provider**: luma
      - **Required Fields**: prompt
      ```  
    - Position visually near the start for user reference.

---

### 5. General Notes & Resources

| Note Content                                                                                                                        | Context or Link                           |
|-------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------|
| The model version ID used (`4235af608e50dac14e9244198cef089049efbd83ba05f2aa4e271076a6f613ee`) is specific to the luma/photon-flash model and may change if the model is updated. Verify the latest version ID on Replicate’s model page. | Model version management                  |
| The workflow requires a valid Replicate API key. Obtain one by creating an account on https://replicate.com and generating an API token. | API Key setup                             |
| Polling with a 2-second wait balances responsiveness and API rate limits; adjust if necessary based on usage patterns or API constraints. | Polling strategy                          |
| Input prompt is hardcoded in the "Create Prediction" node; for dynamic use, consider parameterizing or using an input node for prompts. | Input customization                       |
| For error handling, consider adding nodes to catch HTTP errors or unexpected statuses to improve robustness.                      | Workflow robustness                       |
| Replicate API documentation: https://replicate.com/docs/reference/api                                                      | Official API reference                     |

---

*Disclaimer: The text provided derives exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.*