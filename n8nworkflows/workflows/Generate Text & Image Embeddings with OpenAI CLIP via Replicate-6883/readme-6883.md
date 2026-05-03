Generate Text & Image Embeddings with OpenAI CLIP via Replicate

https://n8nworkflows.xyz/workflows/generate-text---image-embeddings-with-openai-clip-via-replicate-6883


# Generate Text & Image Embeddings with OpenAI CLIP via Replicate

### 1. Workflow Overview

This workflow, titled **Openai Clip Image Generator**, is designed to generate images using the **openai/clip** model hosted on Replicate. It demonstrates how to programmatically create a prediction request to Replicate’s API, poll for its completion status, and process the generated image output. The target use case is automated image generation through AI, suitable for applications requiring AI-driven creative content generation or embedding generation with OpenAI’s CLIP model.

The logical flow is structured into the following blocks:

- **1.1 Trigger and Setup:** Manual start and API key configuration  
- **1.2 Prediction Creation:** Submitting a new image generation request to Replicate  
- **1.3 Prediction Monitoring:** Polling the status of the asynchronous prediction until completion  
- **1.4 Result Processing:** Extracting and formatting the final output data  

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger and Setup

**Overview:**  
This block initiates the workflow manually and sets the necessary API key for authenticating requests to Replicate’s API.

**Nodes Involved:**  
- On clicking 'execute'  
- Set API Key  

**Node Details:**  

- **On clicking 'execute'**  
  - Type: Manual Trigger  
  - Role: Entry point to start the workflow on-demand  
  - Configuration: Default manual trigger, no parameters  
  - Inputs: None  
  - Outputs: Connected to "Set API Key" node  
  - Edge Cases: None, user must manually trigger  
  - Version: v1  

- **Set API Key**  
  - Type: Set  
  - Role: Stores the Replicate API key as a workflow variable for later HTTP requests  
  - Configuration: Sets a string variable `replicate_api_key` with placeholder value "YOUR_REPLICATE_API_KEY" (to be replaced by user)  
  - Inputs: From manual trigger  
  - Outputs: To "Create Prediction" node  
  - Edge Cases: API key missing or invalid will cause authentication failures downstream  
  - Version: v3.3  

---

#### 1.2 Prediction Creation

**Overview:**  
Sends a POST request to Replicate’s API to create a new prediction using the openai/clip model version specified. The request body is currently empty (`"input": {}`) which implies default model behavior or no input parameters.

**Nodes Involved:**  
- Create Prediction  
- Extract Prediction ID  

**Node Details:**  

- **Create Prediction**  
  - Type: HTTP Request  
  - Role: Initiates a new prediction job by calling Replicate’s API endpoint `/v1/predictions`  
  - Configuration:  
    - URL: `https://api.replicate.com/v1/predictions`  
    - Method: POST  
    - Body: JSON containing the model version ID and empty input object  
    - Headers: Authorization with Bearer token from `replicate_api_key`, Content-Type application/json  
    - Timeout: 60 seconds  
  - Inputs: From "Set API Key"  
  - Outputs: To "Extract Prediction ID"  
  - Edge Cases:  
    - API key invalid or missing → 401 Unauthorized  
    - Timeout or network issues  
    - Model version deprecated or invalid → 4xx errors  
  - Version: v4.2  

- **Extract Prediction ID**  
  - Type: Code  
  - Role: Extracts the prediction ID and initial status from the API response, constructs polling URL  
  - Configuration: Custom JavaScript code runs once per item:  
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
  - Inputs: From "Create Prediction"  
  - Outputs: To "Wait"  
  - Edge Cases: Missing or malformed response → code may fail or produce undefined URLs  
  - Version: v2  

---

#### 1.3 Prediction Monitoring

**Overview:**  
Polls the prediction status by repeatedly requesting the prediction endpoint until the status indicates success. Implements a wait cycle of 2 seconds between each poll to avoid excessive requests.

**Nodes Involved:**  
- Wait  
- Check Prediction Status  
- Check If Complete  

**Node Details:**  

- **Wait**  
  - Type: Wait  
  - Role: Pauses workflow execution for 2 seconds before the next status check  
  - Configuration: 2 seconds delay  
  - Inputs: From "Extract Prediction ID" initially, and from "Check If Complete" when prediction incomplete  
  - Outputs: To "Check Prediction Status"  
  - Edge Cases: None critical, but long wait times may affect responsiveness  
  - Version: v1  

- **Check Prediction Status**  
  - Type: HTTP Request  
  - Role: Fetches the current status of the prediction using the prediction URL  
  - Configuration:  
    - URL: Dynamic, from `predictionUrl` extracted previously  
    - Method: GET (default)  
    - Headers: Authorization with Bearer token as before  
  - Inputs: From "Wait"  
  - Outputs: To "Check If Complete"  
  - Edge Cases:  
    - Network issues or invalid URL → request failures  
    - Unauthorized access if API key expired  
  - Version: v4.2  

- **Check If Complete**  
  - Type: If  
  - Role: Checks if the status returned is `"succeeded"`  
  - Configuration: Condition: `$json.status === "succeeded"`  
  - Inputs: From "Check Prediction Status"  
  - Outputs:  
    - True: To "Process Result"  
    - False: Back to "Wait" (loop continues)  
  - Edge Cases:  
    - Other statuses (failed, canceled) are not explicitly handled and will cause indefinite looping  
  - Version: v1  

---

#### 1.4 Result Processing

**Overview:**  
Processes the successful prediction output, extracting relevant metadata and the generated image URL for further use or storage.

**Nodes Involved:**  
- Process Result  

**Node Details:**  

- **Process Result**  
  - Type: Code  
  - Role: Extracts and formats prediction output details into a simplified JSON object  
  - Configuration: Custom JavaScript code, single run per item:  
    ```js
    const result = $input.item.json;

    return {
      status: result.status,
      output: result.output,
      metrics: result.metrics,
      created_at: result.created_at,
      completed_at: result.completed_at,
      model: 'openai/clip',
      image_url: result.output
    };
    ```  
  - Inputs: From "Check If Complete" (true branch)  
  - Outputs: Final item with processed prediction information  
  - Edge Cases: If output missing or incomplete, extracted fields may be undefined  
  - Version: v2  

---

### 3. Summary Table

| Node Name            | Node Type          | Functional Role                    | Input Node(s)          | Output Node(s)           | Sticky Note                                                                                                                                                  |
|----------------------|--------------------|----------------------------------|-----------------------|--------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------|
| On clicking 'execute' | Manual Trigger     | Start workflow manually           | None                  | Set API Key              |                                                                                                                                                              |
| Set API Key          | Set                | Store Replicate API key           | On clicking 'execute'  | Create Prediction        |                                                                                                                                                              |
| Create Prediction    | HTTP Request       | Create prediction request on Replicate API | Set API Key            | Extract Prediction ID    |                                                                                                                                                              |
| Extract Prediction ID| Code               | Extract prediction ID and status  | Create Prediction      | Wait                     |                                                                                                                                                              |
| Wait                 | Wait               | Pause before polling again        | Extract Prediction ID, Check If Complete (false path) | Check Prediction Status |                                                                                                                                                              |
| Check Prediction Status | HTTP Request    | Poll prediction status            | Wait                  | Check If Complete        |                                                                                                                                                              |
| Check If Complete    | If                 | Check if prediction succeeded     | Check Prediction Status| Process Result (true), Wait (false) |                                                                                                                                                              |
| Process Result       | Code               | Format final prediction output    | Check If Complete (true) | None                    |                                                                                                                                                              |
| Sticky Note          | Sticky Note        | Workflow overview and instructions| None                  | None                     | ## Openai Clip Image Generator\n\nThis workflow uses the **openai/clip** model from Replicate to generate image content.\n\n### Setup\n1. Add your Replicate API key\n2. Configure the input parameters\n3. Run the workflow\n\n### Model Details\n- **Type**: Image Generation\n- **Provider**: openai\n- **Required Fields**: None |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node**  
   - Name: `On clicking 'execute'`  
   - Purpose: Start the workflow manually

2. **Create a Set node**  
   - Name: `Set API Key`  
   - Purpose: Store your Replicate API key  
   - Configuration: Add a string field named `replicate_api_key` with value `"YOUR_REPLICATE_API_KEY"` (replace with your actual key)  
   - Connect output of `On clicking 'execute'` to input of this node

3. **Create an HTTP Request node**  
   - Name: `Create Prediction`  
   - Purpose: Submit prediction request to Replicate API  
   - Settings:  
     - HTTP Method: POST  
     - URL: `https://api.replicate.com/v1/predictions`  
     - Authentication: HTTP Header Auth  
     - Header Parameters:  
       - `Authorization`: `Bearer {{$node["Set API Key"].json["replicate_api_key"]}}`  
       - `Content-Type`: `application/json`  
     - Body Content Type: JSON  
     - Body:  
       ```json
       {
         "version": "fd95fe35085b5b9a63d830d3126311ee6b32a7a976c78eb5f210a3a007bcdda6",
         "input": {}
       }
       ```  
     - Timeout: 60000 ms (60 seconds)  
   - Connect output of `Set API Key` to input of this node

4. **Create a Code node**  
   - Name: `Extract Prediction ID`  
   - Purpose: Extract prediction ID and prepare polling URL  
   - Settings:  
     - Mode: Run Once For Each Item  
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
   - Connect output of `Create Prediction` to input of this node

5. **Create a Wait node**  
   - Name: `Wait`  
   - Purpose: Delay between polling attempts  
   - Settings:  
     - Unit: seconds  
     - Amount: 2  
   - Connect output of `Extract Prediction ID` to input of this node

6. **Create an HTTP Request node**  
   - Name: `Check Prediction Status`  
   - Purpose: Poll the status of the prediction  
   - Settings:  
     - HTTP Method: GET  
     - URL: `{{$json["predictionUrl"]}}` (dynamic URL from previous step)  
     - Authentication: HTTP Header Auth  
     - Header Parameters:  
       - `Authorization`: `Bearer {{$node["Set API Key"].json["replicate_api_key"]}}`  
   - Connect output of `Wait` to input of this node

7. **Create an If node**  
   - Name: `Check If Complete`  
   - Purpose: Check if prediction is done  
   - Settings:  
     - Condition Type: Boolean  
     - Condition:  
       - Value 1: `{{$json["status"]}}`  
       - Operation: equal  
       - Value 2: `succeeded`  
   - Connect output of `Check Prediction Status` to input of this node

8. **Connect the If node outputs:**  
   - **True Output:** Connect to a new Code node `Process Result`  
   - **False Output:** Connect back to `Wait` node to continue polling

9. **Create a Code node**  
   - Name: `Process Result`  
   - Purpose: Extract and format the prediction output for downstream use  
   - Settings:  
     - Mode: Run Once For Each Item  
     - Code:  
       ```js
       const result = $input.item.json;

       return {
         status: result.status,
         output: result.output,
         metrics: result.metrics,
         created_at: result.created_at,
         completed_at: result.completed_at,
         model: 'openai/clip',
         image_url: result.output
       };
       ```  
   - Connect the true branch of `Check If Complete` to this node

10. **Optional:** Add a Sticky Note node near the start of the workflow with the content:  
    ```
    ## Openai Clip Image Generator

    This workflow uses the **openai/clip** model from Replicate to generate image content.

    ### Setup
    1. Add your Replicate API key
    2. Configure the input parameters
    3. Run the workflow

    ### Model Details
    - **Type**: Image Generation
    - **Provider**: openai
    - **Required Fields**: None
    ```

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                  |
|-----------------------------------------------------------------------------------------------------------|-------------------------------------------------|
| The workflow uses the **Replicate API** to run an OpenAI CLIP model for image generation.                  | Replicate API docs: https://replicate.com/docs  |
| Ensure your Replicate API key has the necessary permissions and is kept secure to avoid unauthorized usage.| Security best practices for API keys             |
| The polling loop does not handle failure or cancellation statuses explicitly; consider adding such checks. | Enhancing robustness for production use          |
| Model version ID string is hardcoded; update if the model version changes on Replicate.                    | Model version management on Replicate            |

---

**Disclaimer:** The provided text is exclusively derived from an automated n8n workflow. It complies fully with content policies and contains no illegal, offensive, or protected material. All data processed is legal and public.