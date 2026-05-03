Generate Images from Text Prompts with Google Imagen 3 via Replicate API

https://n8nworkflows.xyz/workflows/generate-images-from-text-prompts-with-google-imagen-3-via-replicate-api-3176


# Generate Images from Text Prompts with Google Imagen 3 via Replicate API

### 1. Workflow Overview

This n8n workflow automates the generation of AI images from text prompts using the Google Imagen 3 model via the Replicate API. It is designed for developers, digital artists, and content creators who want to seamlessly integrate AI image generation into their processes with minimal manual steps.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Manual trigger to start the workflow and set the initial text prompt.
- **1.2 Image Generation Request:** Prepare and send the image generation request to the Replicate API.
- **1.3 Prediction Status Polling:** Wait and poll the Replicate API to check the status of the image generation.
- **1.4 Result Handling:** Evaluate the prediction status, handle errors if any, or retrieve the generated image URL.
- **1.5 Error Handling:** Stop the workflow with an error message if the image generation fails or is canceled.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block initiates the workflow manually and sets the text prompt that will be used for image generation.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’ (Manual Trigger)  
  - Set prompt (Set)

- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Starts the workflow on user action (manual click).  
    - Configuration: No parameters; simply triggers the workflow.  
    - Inputs: None  
    - Outputs: Connected to "Set prompt" node.  
    - Edge cases: None typical; user must manually trigger.

  - **Set prompt**  
    - Type: Set  
    - Role: Defines the text prompt for image generation.  
    - Configuration: Contains a field with the prompt text (e.g., a descriptive sentence for the image). This can be customized to any text or dynamic input.  
    - Inputs: From manual trigger node.  
    - Outputs: Connected to "Create prediction" node.  
    - Edge cases: If prompt text is empty or invalid, the API may return an error downstream.

---

#### 1.2 Image Generation Request

- **Overview:**  
  This block formats the request parameters and sends the image generation request to the Replicate API.

- **Nodes Involved:**  
  - Create prediction (HTTP Request)  
  - Pause (Wait)  
  - Set fields for prediction (Set)

- **Node Details:**

  - **Create prediction**  
    - Type: HTTP Request  
    - Role: Sends a POST request to Replicate API to start image generation with the prompt and parameters.  
    - Configuration:  
      - HTTP Method: POST  
      - URL: Replicate API endpoint for Google Imagen 3 model prediction creation.  
      - Authentication: Uses HTTP Header Authentication with `Authorization: Bearer YOUR_REPLICATE_API_KEY`.  
      - Body: JSON containing prompt and optional parameters (aspect ratio, safety filter, etc.).  
    - Inputs: From "Set prompt" node.  
    - Outputs: Connected to "Pause" node.  
    - Edge cases: API key invalid or missing, network errors, invalid parameters.

  - **Pause**  
    - Type: Wait  
    - Role: Waits a short duration to allow the prediction to start processing before status check.  
    - Configuration: Fixed wait time (e.g., a few seconds).  
    - Inputs: From "Create prediction".  
    - Outputs: Connected to "Set fields for prediction".  
    - Edge cases: If wait time is too short, status check may return incomplete results.

  - **Set fields for prediction**  
    - Type: Set  
    - Role: Prepares the necessary fields (e.g., prediction ID) extracted from the previous response to query the prediction status.  
    - Configuration: Extracts prediction ID from the "Create prediction" response.  
    - Inputs: From "Pause".  
    - Outputs: Connected to "Check prediction status".  
    - Edge cases: Missing or malformed prediction ID will cause status check failure.

---

#### 1.3 Prediction Status Polling

- **Overview:**  
  This block queries the Replicate API to check the current status of the image generation prediction.

- **Nodes Involved:**  
  - Check prediction status (HTTP Request)  
  - Check for success (If)  
  - Check for errors (If)  
  - Wait (Wait)

- **Node Details:**

  - **Check prediction status**  
    - Type: HTTP Request  
    - Role: Sends a GET request to Replicate API to retrieve the status and results of the prediction.  
    - Configuration:  
      - HTTP Method: GET  
      - URL: Replicate API endpoint with prediction ID parameter.  
      - Authentication: Same HTTP Header Authentication as before.  
    - Inputs: From "Set fields for prediction".  
    - Outputs: Connected to "Check for success".  
    - Edge cases: Network errors, invalid prediction ID, API rate limits.

  - **Check for success**  
    - Type: If  
    - Role: Checks if the prediction status is "succeeded".  
    - Configuration: Condition evaluates the status field in the API response.  
    - Inputs: From "Check prediction status".  
    - Outputs:  
      - True: Connects to "Get the image URL".  
      - False: Connects to "Check for errors".  
    - Edge cases: Unexpected status values.

  - **Check for errors**  
    - Type: If  
    - Role: Checks if the prediction status indicates an error (e.g., "failed" or "canceled").  
    - Configuration: Condition evaluates the status field for error states.  
    - Inputs: From "Check for success" (false branch).  
    - Outputs:  
      - True: Connects to "Stop and Error".  
      - False: Connects to "Wait" (to retry status check).  
    - Edge cases: Infinite loops if status never changes; consider max retry limits.

  - **Wait**  
    - Type: Wait  
    - Role: Pauses before retrying the status check to avoid excessive API calls.  
    - Configuration: Fixed wait time (e.g., several seconds).  
    - Inputs: From "Check for errors" (false branch).  
    - Outputs: Connects back to "Set fields for prediction" to re-check status.  
    - Edge cases: Long wait times increase total workflow duration.

---

#### 1.4 Result Handling

- **Overview:**  
  Once the prediction succeeds, this block extracts and outputs the URL of the generated image.

- **Nodes Involved:**  
  - Get the image URL (Set)

- **Node Details:**

  - **Get the image URL**  
    - Type: Set  
    - Role: Extracts the image URL from the successful prediction response and formats it for output.  
    - Configuration: Sets a field with the image URL, typically from the `output` property of the API response.  
    - Inputs: From "Check for success" (true branch).  
    - Outputs: Final output of the workflow.  
    - Edge cases: If output field is missing or empty, no image URL will be returned.

---

#### 1.5 Error Handling

- **Overview:**  
  This block stops the workflow and reports an error if the image generation fails or is canceled.

- **Nodes Involved:**  
  - Stop and Error (Stop and Error)

- **Node Details:**

  - **Stop and Error**  
    - Type: Stop and Error  
    - Role: Terminates the workflow execution with an error message.  
    - Configuration: Default error message or customized message based on failure.  
    - Inputs: From "Check for errors" (true branch).  
    - Outputs: None (workflow ends).  
    - Edge cases: None; ensures clean termination on failure.

---

### 3. Summary Table

| Node Name                 | Node Type           | Functional Role                          | Input Node(s)               | Output Node(s)               | Sticky Note                              |
|---------------------------|---------------------|----------------------------------------|-----------------------------|-----------------------------|-----------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger      | Starts workflow manually                | None                        | Set prompt                  |                                         |
| Set prompt                | Set                 | Defines text prompt for image generation | When clicking ‘Test workflow’ | Create prediction           |                                         |
| Create prediction         | HTTP Request        | Sends image generation request to API  | Set prompt                  | Pause                       |                                         |
| Pause                     | Wait                | Waits before checking prediction status | Create prediction           | Set fields for prediction   |                                         |
| Set fields for prediction | Set                 | Extracts prediction ID for status check | Pause                      | Check prediction status     |                                         |
| Check prediction status   | HTTP Request        | Queries API for prediction status       | Set fields for prediction   | Check for success           |                                         |
| Check for success         | If                  | Checks if prediction succeeded          | Check prediction status     | Get the image URL, Check for errors |                                         |
| Get the image URL         | Set                 | Extracts and outputs image URL           | Check for success (true)    | None                        |                                         |
| Check for errors          | If                  | Checks if prediction failed or canceled | Check for success (false)   | Stop and Error, Wait        |                                         |
| Wait                      | Wait                | Waits before retrying status check       | Check for errors (false)    | Set fields for prediction   |                                         |
| Stop and Error            | Stop and Error      | Stops workflow on error                   | Check for errors (true)     | None                        |                                         |
| Sticky Note               | Sticky Note         | Comments or instructions                  | None                        | None                        |                                         |
| Sticky Note1              | Sticky Note         | Comments or instructions                  | None                        | None                        |                                         |
| Sticky Note2              | Sticky Note         | Comments or instructions                  | None                        | None                        |                                         |
| Sticky Note3              | Sticky Note         | Comments or instructions                  | None                        | None                        |                                         |
| Sticky Note4              | Sticky Note         | Comments or instructions                  | None                        | None                        |                                         |
| Sticky Note5              | Sticky Note         | Comments or instructions                  | None                        | None                        |                                         |
| Sticky Note6              | Sticky Note         | Comments or instructions                  | None                        | None                        |                                         |
| Sticky Note7              | Sticky Note         | Comments or instructions                  | None                        | None                        |                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node**  
   - Name: `When clicking ‘Test workflow’`  
   - Purpose: To start the workflow manually.

2. **Add a Set node**  
   - Name: `Set prompt`  
   - Purpose: Define the text prompt for image generation.  
   - Configuration: Add a field (e.g., `prompt`) with the desired text describing the image to generate.

3. **Add an HTTP Request node**  
   - Name: `Create prediction`  
   - Purpose: Send a POST request to Replicate API to create an image generation prediction.  
   - Configuration:  
     - HTTP Method: POST  
     - URL: `https://api.replicate.com/v1/predictions` (or the specific endpoint for Google Imagen 3)  
     - Authentication: Use HTTP Header Authentication with header `Authorization: Bearer YOUR_REPLICATE_API_KEY`.  
     - Body Content-Type: JSON  
     - Body Parameters: Include the prompt and any other parameters required by the model (e.g., model version, aspect ratio).  
   - Connect input from `Set prompt`.

4. **Add a Wait node**  
   - Name: `Pause`  
   - Purpose: Wait a few seconds to allow the prediction to start processing.  
   - Configuration: Set wait time (e.g., 5-10 seconds).  
   - Connect input from `Create prediction`.

5. **Add a Set node**  
   - Name: `Set fields for prediction`  
   - Purpose: Extract the prediction ID from the previous response to use in status checks.  
   - Configuration: Set a field with the prediction ID extracted from the `Create prediction` node output.  
   - Connect input from `Pause`.

6. **Add an HTTP Request node**  
   - Name: `Check prediction status`  
   - Purpose: Send a GET request to Replicate API to check prediction status.  
   - Configuration:  
     - HTTP Method: GET  
     - URL: `https://api.replicate.com/v1/predictions/{{ $json["prediction_id"] }}` (use expression to insert prediction ID)  
     - Authentication: Same as before.  
   - Connect input from `Set fields for prediction`.

7. **Add an If node**  
   - Name: `Check for success`  
   - Purpose: Check if the prediction status is "succeeded".  
   - Configuration: Set condition to check if `status` field equals `"succeeded"`.  
   - Connect input from `Check prediction status`.

8. **Add a Set node**  
   - Name: `Get the image URL`  
   - Purpose: Extract the generated image URL from the prediction output.  
   - Configuration: Set a field with the image URL from the response (e.g., `$json["output"][0]`).  
   - Connect input from `Check for success` (true branch).

9. **Add another If node**  
   - Name: `Check for errors`  
   - Purpose: Check if the prediction status is `"failed"` or `"canceled"`.  
   - Configuration: Condition checks if `status` equals `"failed"` OR `"canceled"`.  
   - Connect input from `Check for success` (false branch).

10. **Add a Stop and Error node**  
    - Name: `Stop and Error`  
    - Purpose: Stop workflow execution and report error if prediction failed or canceled.  
    - Connect input from `Check for errors` (true branch).

11. **Add a Wait node**  
    - Name: `Wait`  
    - Purpose: Wait before retrying status check to avoid rapid polling.  
    - Configuration: Set wait time (e.g., 5-10 seconds).  
    - Connect input from `Check for errors` (false branch).

12. **Connect the Wait node output back to the Set fields for prediction node**  
    - This creates a loop to poll the prediction status until success or error.

13. **Configure Credentials**  
    - In n8n, create HTTP Header Authentication credentials named appropriately (e.g., `Replicate API Key`).  
    - Set header `Authorization` with value `Bearer YOUR_REPLICATE_API_KEY`.  
    - Assign these credentials to all HTTP Request nodes interacting with Replicate API.

14. **Test the workflow**  
    - Trigger manually and verify the image URL is returned on success or error is handled gracefully.

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                                                                                      |
|-------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| To obtain a Replicate API key, sign up at https://replicate.com and generate an API key in your dashboard. | Replicate API setup instructions.                                                                   |
| Configure HTTP Header Authentication in n8n with header name `Authorization` and value `Bearer YOUR_REPLICATE_API_KEY`. | Credential setup for API authentication.                                                            |
| Adjust wait times in "Pause" and "Wait" nodes based on expected image generation duration to optimize workflow runtime. | Performance tuning tip.                                                                              |
| Customize the prompt in the "Set prompt" node to generate different images; supports dynamic inputs for integration with other workflows. | Usage customization.                                                                                |
| For more advanced use cases, modify the "Check for success" and "Check for errors" nodes to add branching logic or notifications. | Workflow extension advice.                                                                          |

---

This documentation provides a complete, structured reference to understand, reproduce, and customize the "Generate Images from Text Prompts with Google Imagen 3 via Replicate API" workflow in n8n.