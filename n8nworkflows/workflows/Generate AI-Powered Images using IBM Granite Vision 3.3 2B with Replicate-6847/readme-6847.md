Generate AI-Powered Images using IBM Granite Vision 3.3 2B with Replicate

https://n8nworkflows.xyz/workflows/generate-ai-powered-images-using-ibm-granite-vision-3-3-2b-with-replicate-6847


# Generate AI-Powered Images using IBM Granite Vision 3.3 2B with Replicate

### 1. Workflow Overview

This workflow automates the generation of AI-powered images using the IBM Granite Vision 3.3 2B model accessed via the Replicate API. It is designed for users who want to input parameters and generate images programmatically, with robust handling of asynchronous prediction status, error conditions, and logging.

The workflow is logically divided into the following blocks:

- **1.1 Input Initialization**: Captures manual trigger input and sets the Replicate API token.
- **1.2 Parameter Configuration**: Prepares all input parameters for the image generation model, including defaults and user inputs.
- **1.3 Prediction Request and Logging**: Sends the image generation request to Replicate and logs the request details.
- **1.4 Status Polling Loop**: Waits and polls the prediction status until the image generation completes or fails, with retry logic.
- **1.5 Result Handling**: Processes the final outcome, distinguishes between success and failure, and formats the response.
- **1.6 Output Presentation**: Displays the final structured result for downstream use or user consumption.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Input Initialization

- **Overview:**  
  Initiates the workflow manually and securely sets the Replicate API token for authentication.

- **Nodes Involved:**  
  - Manual Trigger  
  - Set API Token

- **Node Details:**

  - **Manual Trigger**  
    - Type: Trigger node  
    - Role: Starts the workflow execution on demand  
    - Configuration: No parameters; manual user activation required  
    - Input: None  
    - Output: Triggers the next node to set API token  
    - Edge Cases: None

  - **Set API Token**  
    - Type: Set node  
    - Role: Stores the Replicate API token securely within the workflow execution context  
    - Configuration: Assigns string value "YOUR_REPLICATE_API_TOKEN" to variable `api_token` (placeholder to be replaced by user)  
    - Input: Trigger from Manual Trigger  
    - Output: Passes `api_token` to the next node  
    - Edge Cases: Failure if token is missing or invalid in subsequent API calls

#### Block 1.2: Parameter Configuration

- **Overview:**  
  Defines all user-configurable and default parameters required by the IBM Granite Vision model to generate images.

- **Nodes Involved:**  
  - Set Image Parameters

- **Node Details:**

  - **Set Image Parameters**  
    - Type: Set node  
    - Role: Consolidates and prepares input parameters for the image generation request  
    - Configuration:  
      - Copies `api_token` from Set API Token node  
      - Defines parameters such as `seed` (-1 for random), `image` (default placeholder URL), `top_k` (50), `top_p` (0.9), `images` (empty array), `prompt` (empty string), and other generation controls like `max_tokens`, `temperature`, penalties, etc.  
    - Input: Receives `api_token` from Set API Token  
    - Output: Passes all parameters to HTTP request node  
    - Edge Cases: Empty or invalid parameters could cause API rejection; user should customize before execution

#### Block 1.3: Prediction Request and Logging

- **Overview:**  
  Sends the POST request to Replicate API to create a new image generation prediction and logs the request details for monitoring.

- **Nodes Involved:**  
  - Create Image Prediction  
  - Log Request

- **Node Details:**

  - **Create Image Prediction**  
    - Type: HTTP Request node  
    - Role: Initiates the image generation process by sending parameters to the Replicate prediction endpoint  
    - Configuration:  
      - URL: `https://api.replicate.com/v1/predictions`  
      - Method: POST  
      - Headers: Authorization Bearer token from `api_token`, Prefer: wait (waits for synchronous response if possible)  
      - Body: JSON containing model version and input parameters (mapped from prior Set node variables)  
      - Response: JSON parsed, with neverError=true to prevent node failure on HTTP errors  
    - Input: Parameters from Set Image Parameters  
    - Output: Prediction response containing prediction `id` and status details  
    - Edge Cases: Authorization failure if token is invalid; API downtime; malformed parameter data

  - **Log Request**  
    - Type: Code node (JavaScript)  
    - Role: Logs prediction request details (timestamp, prediction ID, model type) to console for debugging and monitoring  
    - Configuration: Custom JS code accessing the input JSON  
    - Input: Response from Create Image Prediction  
    - Output: Passes data forward unchanged  
    - Edge Cases: None significant; logging failures do not impact workflow

#### Block 1.4: Status Polling Loop

- **Overview:**  
  Implements a polling mechanism to check the status of the asynchronous image generation prediction, with waits and conditional retries.

- **Nodes Involved:**  
  - Wait 5s  
  - Check Status  
  - Is Complete?  
  - Has Failed?  
  - Wait 10s

- **Node Details:**

  - **Wait 5s**  
    - Type: Wait node  
    - Role: Pauses workflow for 5 seconds before status check to allow prediction processing  
    - Configuration: 5 seconds delay  
    - Input: From Log Request  
    - Output: Triggers Check Status node  

  - **Check Status**  
    - Type: HTTP Request node  
    - Role: Queries Replicate API for current prediction status using prediction `id`  
    - Configuration:  
      - URL dynamically set to `https://api.replicate.com/v1/predictions/{{prediction_id}}`  
      - Method: GET  
      - Authorization header with `api_token`  
      - neverError=true, JSON response parsing  
    - Input: From Wait nodes or conditionals  
    - Output: Prediction status JSON  
    - Edge Cases: API errors, timeouts, invalid IDs

  - **Is Complete?**  
    - Type: If node  
    - Role: Checks if prediction status equals "succeeded" indicating completion  
    - Configuration: Checks `status === "succeeded"`  
    - Input: From Check Status  
    - Output:  
      - True branch: proceed to Success Response  
      - False branch: proceed to Has Failed? check  

  - **Has Failed?**  
    - Type: If node  
    - Role: Checks if prediction status equals "failed" indicating error in generation  
    - Configuration: Checks `status === "failed"`  
    - Input: From Is Complete? (false branch)  
    - Output:  
      - True branch: proceed to Error Response  
      - False branch: loops back via Wait 10s to Check Status  

  - **Wait 10s**  
    - Type: Wait node  
    - Role: Implements delay before retrying status check after no success/failure state  
    - Configuration: 10 seconds delay  
    - Input: From Has Failed? (false branch)  
    - Output: Loops to Check Status  
    - Edge Cases: Long-running predictions may cause extended delays; workflow execution timeout risks

#### Block 1.5: Result Handling

- **Overview:**  
  Prepares structured JSON responses for both successful and failed image generation outcomes.

- **Nodes Involved:**  
  - Success Response  
  - Error Response

- **Node Details:**

  - **Success Response**  
    - Type: Set node  
    - Role: Constructs an object with success status, image URL, prediction ID, status, and user message  
    - Configuration: Uses output fields from prediction JSON (`output`, `id`, `status`)  
    - Input: From Is Complete? (true branch)  
    - Output: A standardized success response object  
    - Edge Cases: Missing or malformed output URL

  - **Error Response**  
    - Type: Set node  
    - Role: Constructs an error object with failure status, error message, prediction ID, status, and user message  
    - Configuration: Uses error field if present, else generic failure message  
    - Input: From Has Failed? (true branch)  
    - Output: A standardized error response object  
    - Edge Cases: No error detail in response; ambiguous failure reasons

#### Block 1.6: Output Presentation

- **Overview:**  
  Finalizes and prepares the response object for output or further integration.

- **Nodes Involved:**  
  - Display Result

- **Node Details:**

  - **Display Result**  
    - Type: Set node  
    - Role: Wraps the response object (whether success or error) into a final output variable `final_result`  
    - Configuration: Sets field `final_result` to the incoming response object  
    - Input: From Success Response or Error Response  
    - Output: Final structured result for downstream consumption  
    - Edge Cases: None significant

---

### 3. Summary Table

| Node Name             | Node Type          | Functional Role                           | Input Node(s)                 | Output Node(s)                 | Sticky Note                                                                                                                           |
|-----------------------|--------------------|-----------------------------------------|------------------------------|-------------------------------|--------------------------------------------------------------------------------------------------------------------------------------|
| Manual Trigger        | Trigger            | Starts workflow execution                | None                         | Set API Token                  | See "GRANITE-VISION-3.3-2B GENERATOR" introductory sticky note for support and contact info                                         |
| Set API Token         | Set                | Sets Replicate API token                 | Manual Trigger               | Set Image Parameters           | See workflow overview sticky note with detailed info                                                                                 |
| Set Image Parameters  | Set                | Configures input parameters for API     | Set API Token                | Create Image Prediction        | See workflow overview sticky note with parameter descriptions                                                                         |
| Create Image Prediction| HTTP Request       | Sends image generation request           | Set Image Parameters          | Log Request                   | See workflow overview sticky note with API endpoint details                                                                          |
| Log Request           | Code               | Logs prediction request details          | Create Image Prediction       | Wait 5s                       | See workflow overview sticky note with monitoring notes                                                                              |
| Wait 5s               | Wait               | Waits before status check                 | Log Request                  | Check Status                  |                                                                                                                                    |
| Check Status          | HTTP Request       | Checks status of image generation        | Wait 5s, Wait 10s, Has Failed? | Is Complete?                 |                                                                                                                                    |
| Is Complete?          | If                 | Checks if prediction succeeded           | Check Status                 | Success Response, Has Failed?  |                                                                                                                                    |
| Has Failed?           | If                 | Checks if prediction failed               | Is Complete?                 | Error Response, Wait 10s       |                                                                                                                                    |
| Wait 10s              | Wait               | Waits before retrying status check       | Has Failed?                  | Check Status                  |                                                                                                                                    |
| Success Response      | Set                | Formats success response object           | Is Complete?                 | Display Result                |                                                                                                                                    |
| Error Response        | Set                | Formats error response object             | Has Failed?                  | Display Result                |                                                                                                                                    |
| Display Result        | Set                | Prepares final response output            | Success Response, Error Response | None                        |                                                                                                                                    |
| Sticky Note9          | Sticky Note        | Provides contact/support info              | None                         | None                         | "GRANITE-VISION-3.3-2B GENERATOR" with contact and tutorial links                                                                    |
| Sticky Note4          | Sticky Note        | Detailed overview, parameter guide, usage | None                         | None                         | "IBM-GRANITE/GRANITE-VISION-3.3-2B - IMAGE GENERATION WORKFLOW" with extensive documentation                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger node**  
   - Type: Manual Trigger  
   - No parameters; this node initiates the workflow.

2. **Create Set node named "Set API Token"**  
   - Assign a string variable `api_token` with value `"YOUR_REPLICATE_API_TOKEN"` (replace with actual token).  
   - Connect from Manual Trigger node.

3. **Create Set node named "Set Image Parameters"**  
   - Assign multiple variables:  
     - `api_token`: Expression `{{$node["Set API Token"].json.api_token}}`  
     - `seed`: `-1` (random seed)  
     - `image`: `"https://picsum.photos/512/512"` (default placeholder)  
     - `top_k`: `50`  
     - `top_p`: `0.9`  
     - `images`: `[]` (empty array)  
     - `prompt`: `""` (empty string)  
     - `max_tokens`: `512`  
     - `min_tokens`: `0`  
     - `temperature`: `0.6`  
     - `chat_template`, `system_prompt`, `stop_sequences`: `""` (empty strings)  
     - `presence_penalty`, `frequency_penalty`: `0`  
   - Connect from Set API Token node.

4. **Create HTTP Request node named "Create Image Prediction"**  
   - Method: POST  
   - URL: `https://api.replicate.com/v1/predictions`  
   - Headers:  
     - `Authorization`: Expression `Bearer {{$json.api_token}}`  
     - `Prefer`: `wait`  
   - Body (raw JSON):  
     ```json
     {
       "version": "ibm-granite/granite-vision-3.3-2b:3339e8453ca94104383f6f085a511d7f26cca2d0cab2f6018986737b6cf7d391",
       "input": {
         "seed": {{$json.seed}},
         "image": "{{$json.image}}",
         "top_k": {{$json.top_k}},
         "top_p": {{$json.top_p}},
         "images": "{{$json.images}}",
         "prompt": "{{$json.prompt}}",
         "max_tokens": {{$json.max_tokens}},
         "min_tokens": {{$json.min_tokens}},
         "temperature": {{$json.temperature}},
         "chat_template": "{{$json.chat_template}}",
         "system_prompt": "{{$json.system_prompt}}",
         "stop_sequences": "{{$json.stop_sequences}}",
         "presence_penalty": {{$json.presence_penalty}},
         "frequency_penalty": {{$json.frequency_penalty}}
       }
     }
     ```
   - Response parse as JSON, never error on HTTP error  
   - Connect from Set Image Parameters node.

5. **Create Code node named "Log Request"**  
   - JavaScript code:  
     ```javascript
     const data = $input.all()[0].json;

     console.log('ibm-granite/granite-vision-3.3-2b Request:', {
       timestamp: new Date().toISOString(),
       prediction_id: data.id,
       model_type: 'image'
     });

     return $input.all();
     ```  
   - Connect from Create Image Prediction node.

6. **Create Wait node named "Wait 5s"**  
   - Wait for 5 seconds  
   - Connect from Log Request node.

7. **Create HTTP Request node named "Check Status"**  
   - Method: GET  
   - URL: Expression `https://api.replicate.com/v1/predictions/{{$json.id}}` (use prediction ID from Create Image Prediction)  
   - Header: `Authorization` Bearer token from expression `{{$node["Set API Token"].json.api_token}}`  
   - Response JSON parse, never error  
   - Connect from Wait 5s node and later Wait 10s node and Has Failed? node (loop back)

8. **Create If node named "Is Complete?"**  
   - Condition: Check if `{{$json.status}}` equals "succeeded"  
   - Connect from Check Status node.  
   - True path: Connect to Success Response node  
   - False path: Connect to Has Failed? node

9. **Create If node named "Has Failed?"**  
   - Condition: Check if `{{$json.status}}` equals "failed"  
   - Connect from Is Complete? false path  
   - True path: Connect to Error Response node  
   - False path: Connect to Wait 10s node

10. **Create Wait node named "Wait 10s"**  
    - Wait for 10 seconds  
    - Connect from Has Failed? false path  
    - Output connects back to Check Status node (loop)

11. **Create Set node named "Success Response"**  
    - Assign field `response` to an object:  
      ```json
      {
        "success": true,
        "image_url": {{$json.output}},
        "prediction_id": {{$json.id}},
        "status": {{$json.status}},
        "message": "Image generated successfully"
      }
      ```  
    - Connect from Is Complete? true path

12. **Create Set node named "Error Response"**  
    - Assign field `response` to an object:  
      ```json
      {
        "success": false,
        "error": {{$json.error || 'Image generation failed'}},
        "prediction_id": {{$json.id}},
        "status": {{$json.status}},
        "message": "Failed to generate image"
      }
      ```  
    - Connect from Has Failed? true path

13. **Create Set node named "Display Result"**  
    - Assign field `final_result` to `{{$json.response}}`  
    - Connect from both Success Response and Error Response nodes

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | Context or Link                                                                                             |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| ===============================<br>GRANITE-VISION-3.3-2B GENERATOR<br>================================<br>For any questions or support, please contact:<br>Yaron@nofluff.online<br><br>Explore more tips and tutorials here:<br>- YouTube: https://www.youtube.com/@YaronBeen/videos<br>- LinkedIn: https://www.linkedin.com/in/yaronbeen/<br>================================ | Contact and support information; tutorial resources                                                        |
| IBM-GRANITE/GRANITE-VISION-3.3-2B - IMAGE GENERATION WORKFLOW<br>Powered by Replicate API and n8n Automation<br><br>Model Owner: ibm-granite<br>Model Type: Image Generation<br>API Endpoint: https://api.replicate.com/v1/predictions<br><br>Model extracts information from visual documents (tables, charts, etc.)<br><br>Parameter guide and workflow explanation included.<br><br>Troubleshooting tips and best practices.<br><br>Useful external resources:<br>- Model Docs: https://replicate.com/ibm-granite/granite-vision-3.3-2b<br>- Replicate API Docs: https://replicate.com/docs<br>- n8n Documentation: https://docs.n8n.io | Comprehensive workflow documentation and usage guide embedded as sticky note                                   |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.