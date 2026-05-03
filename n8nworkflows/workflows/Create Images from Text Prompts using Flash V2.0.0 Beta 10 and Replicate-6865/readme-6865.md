Create Images from Text Prompts using Flash V2.0.0 Beta 10 and Replicate

https://n8nworkflows.xyz/workflows/create-images-from-text-prompts-using-flash-v2-0-0-beta-10-and-replicate-6865


# Create Images from Text Prompts using Flash V2.0.0 Beta 10 and Replicate

### 1. Workflow Overview

This workflow automates the generation of images from text prompts using the Flash V2.0.0 Beta 10 AI model hosted on the Replicate platform. It is designed to take user input parameters, send them as a prediction request to the Replicate API, monitor the asynchronous generation process, handle success or failure outcomes, and return the result in a structured response.

The workflow is logically divided into the following functional blocks:

- **1.1 Input Initialization:** Receives a manual trigger and sets up authentication and generation parameters.
- **1.2 Prediction Request Submission:** Sends the configured parameters to the Replicate API to initiate image generation.
- **1.3 Status Polling Loop:** Waits and checks periodically for completion or failure of the prediction.
- **1.4 Outcome Handling:** Routes the workflow based on success or failure, formatting appropriate responses.
- **1.5 Logging and Monitoring:** Logs request data for debugging and tracking.
- **1.6 Result Output:** Final node that consolidates and outputs the structured result.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Initialization

**Overview:**  
This block starts the workflow via manual trigger and configures essential API credentials and all other model input parameters with default or user-provided values.

**Nodes Involved:**  
- Manual Trigger  
- Set API Token  
- Set Other Parameters  

**Node Details:**

- **Manual Trigger**  
  - Type: Trigger node  
  - Role: Initiates the workflow manually by user interaction  
  - Configuration: Default (no parameters)  
  - Input: None  
  - Output: Triggers downstream nodes  
  - Edge Cases: None  

- **Set API Token**  
  - Type: Set node  
  - Role: Assigns the Replicate API token for authentication  
  - Configuration: Sets variable `api_token` with placeholder `"YOUR_REPLICATE_API_TOKEN"` to be replaced by the user  
  - Input: From Manual Trigger  
  - Output: Provides `api_token` for downstream use  
  - Edge Cases: Missing or invalid token will cause authentication failure downstream  

- **Set Other Parameters**  
  - Type: Set node  
  - Role: Configures input parameters for the Flash model, including prompt, image size, seed, and other generation options  
  - Configuration:  
    - Copies `api_token` from previous node  
    - Defines parameters like `mask`, `seed`, `image`, `model`, `width`, `height`, `prompt`, `go_fast`, `extra_lora`, `lora_scale`, `megapixels`, `num_outputs`, `aspect_ratio`, `output_format`, `guidance_scale`, `output_quality`, `prompt_strength`, `extra_lora_scale`, `replicate_weights`, `num_inference_steps`, `disable_safety_checker` with sensible defaults  
  - Input: From Set API Token  
  - Output: All parameters ready for API submission  
  - Edge Cases: Invalid or missing parameters could lead to API errors  

---

#### 2.2 Prediction Request Submission

**Overview:**  
Submits the image generation request to the Replicate API using the configured parameters and authentication token, initiating the prediction process.

**Nodes Involved:**  
- Create Other Prediction  

**Node Details:**

- **Create Other Prediction**  
  - Type: HTTP Request node  
  - Role: Sends POST request to `https://api.replicate.com/v1/predictions` to create an image generation prediction  
  - Configuration:  
    - HTTP Method: POST  
    - Body: JSON object containing the model version ID and all input parameters dynamically injected via expressions  
    - Headers: Authorization with Bearer token from `api_token`, Prefer header set to `wait` for immediate response when possible  
    - Response Handling: JSON response, never error to allow graceful handling  
  - Input: From Set Other Parameters  
  - Output: Prediction response including `id` for tracking  
  - Edge Cases:  
    - API authentication errors (invalid token)  
    - Network timeouts  
    - Model version mismatch  
    - Invalid parameters causing server errors  

---

#### 2.3 Status Polling Loop

**Overview:**  
Implements an asynchronous polling mechanism that waits and checks the prediction status repeatedly until completion or failure.

**Nodes Involved:**  
- Log Request  
- Wait 5s  
- Check Status  
- Is Complete?  
- Has Failed?  
- Wait 10s  

**Node Details:**

- **Log Request**  
  - Type: Code node (JavaScript)  
  - Role: Logs prediction details (timestamp, prediction ID, model type) for monitoring/debugging  
  - Configuration: Outputs the input data unchanged after logging  
  - Input: From Create Other Prediction  
  - Output: Passes data to Wait 5s  
  - Edge Cases: Console logging issues are generally non-blocking  

- **Wait 5s**  
  - Type: Wait node  
  - Role: Pauses workflow for 5 seconds between status checks  
  - Configuration: Wait 5 seconds  
  - Input: From Log Request  
  - Output: To Check Status  
  - Edge Cases: Minimal, but workflow delays if API is slow  

- **Check Status**  
  - Type: HTTP Request node  
  - Role: Queries the Replicate API for the current status of the prediction using prediction ID  
  - Configuration:  
    - HTTP Method: GET  
    - URL dynamically built using prediction ID from Create Other Prediction response  
    - Authorization header with Bearer token  
    - JSON response expected  
  - Input: From Wait 5s or Wait 10s  
  - Output: To Is Complete?  
  - Edge Cases: API errors, network timeout, invalid prediction ID  

- **Is Complete?**  
  - Type: If node  
  - Role: Checks if the prediction status is "succeeded"  
  - Configuration: Compares `$json.status` equals "succeeded"  
  - Input: From Check Status  
  - Output:  
    - True branch to Success Response  
    - False branch to Has Failed?  
  - Edge Cases: Status field missing or unexpected values  

- **Has Failed?**  
  - Type: If node  
  - Role: Checks if the prediction status is "failed"  
  - Configuration: Compares `$json.status` equals "failed"  
  - Input: From Is Complete? (False branch)  
  - Output:  
    - True branch to Error Response  
    - False branch to Wait 10s (poll again)  
  - Edge Cases: Unexpected status values leading to indefinite looping  

- **Wait 10s**  
  - Type: Wait node  
  - Role: Pauses 10 seconds before the next status check retry  
  - Configuration: Wait 10 seconds  
  - Input: From Has Failed? (False branch)  
  - Output: Back to Check Status, forming a polling loop  
  - Edge Cases: Potential long wait times or infinite loop if prediction hangs  

---

#### 2.4 Outcome Handling

**Overview:**  
Handles successful or failed prediction results and formats structured JSON responses for downstream consumption.

**Nodes Involved:**  
- Success Response  
- Error Response  
- Display Result  

**Node Details:**

- **Success Response**  
  - Type: Set node  
  - Role: Constructs a JSON object for successful prediction response, including the result URL, prediction ID, status, and success message  
  - Configuration: Creates field `response` with keys: `success: true`, `result_url`, `prediction_id`, `status`, and a message string  
  - Input: From Is Complete? (True branch)  
  - Output: To Display Result  
  - Edge Cases: Missing output URLs or incomplete data  

- **Error Response**  
  - Type: Set node  
  - Role: Constructs a JSON object for failed prediction response with error details, prediction ID, status, and failure message  
  - Configuration: Creates field `response` with keys: `success: false`, `error`, `prediction_id`, `status`, and a message string  
  - Input: From Has Failed? (True branch)  
  - Output: To Display Result  
  - Edge Cases: Missing or unclear error messages  

- **Display Result**  
  - Type: Set node  
  - Role: Final consolidation of the response object (success or error) into `final_result`  
  - Configuration: Assigns `final_result` to the incoming `response` object  
  - Input: From Success Response or Error Response  
  - Output: Final output of the workflow  
  - Edge Cases: None  

---

#### 2.5 Logging and Monitoring

**Overview:**  
Provides logging of prediction requests for tracking and debugging purposes.

**Nodes Involved:**  
- Log Request  

**Node Details:**  
(Already described above in 2.3)  

---

### 3. Summary Table

| Node Name              | Node Type           | Functional Role                      | Input Node(s)              | Output Node(s)            | Sticky Note                                                                                                                   |
|------------------------|---------------------|------------------------------------|----------------------------|---------------------------|-------------------------------------------------------------------------------------------------------------------------------|
| Manual Trigger         | Manual Trigger      | Initiate workflow manually          | None                       | Set API Token              |                                                                                                                               |
| Set API Token          | Set                 | Set Replicate API token             | Manual Trigger             | Set Other Parameters       |                                                                                                                               |
| Set Other Parameters   | Set                 | Configure model input parameters    | Set API Token              | Create Other Prediction    |                                                                                                                               |
| Create Other Prediction| HTTP Request        | Submit prediction request to API    | Set Other Parameters       | Log Request               |                                                                                                                               |
| Log Request            | Code                | Log prediction details for monitoring| Create Other Prediction    | Wait 5s                   |                                                                                                                               |
| Wait 5s                | Wait                | Pause 5 seconds before checking status| Log Request             | Check Status              |                                                                                                                               |
| Check Status           | HTTP Request        | Check prediction status             | Wait 5s, Wait 10s          | Is Complete?              |                                                                                                                               |
| Is Complete?           | If                  | Check if prediction succeeded       | Check Status               | Success Response, Has Failed? |                                                                                                                               |
| Has Failed?            | If                  | Check if prediction failed          | Is Complete?               | Error Response, Wait 10s  |                                                                                                                               |
| Wait 10s               | Wait                | Pause 10 seconds before retry       | Has Failed?                | Check Status              |                                                                                                                               |
| Success Response       | Set                 | Prepare success response object     | Is Complete?               | Display Result            |                                                                                                                               |
| Error Response         | Set                 | Prepare error response object       | Has Failed?                | Display Result            |                                                                                                                               |
| Display Result         | Set                 | Final output consolidation          | Success Response, Error Response | None                 |                                                                                                                               |
| Sticky Note9           | Sticky Note         | Informational note with contact info| None                      | None                      | =======================================\nFLASH-V2.0.0-BETA.10 GENERATOR\nFor support contact: Yaron@nofluff.online\nYouTube & LinkedIn links |
| Sticky Note4           | Sticky Note         | Detailed workflow documentation      | None                      | None                      | Comprehensive documentation, usage guide, parameter reference, troubleshooting, and resource links embedded in note            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Position: Starting point  
   - No parameters required  

2. **Create Set Node "Set API Token"**  
   - Type: Set  
   - Assign a string variable named `api_token`  
   - Value: `"YOUR_REPLICATE_API_TOKEN"` (replace with actual token)  
   - Connect output of Manual Trigger to this node  

3. **Create Set Node "Set Other Parameters"**  
   - Type: Set  
   - Assign variables for all model inputs including:  
     - `api_token`: `={{ $('Set API Token').item.json.api_token }}` (expression to copy token)  
     - `mask`: `"https://via.placeholder.com/512x512/000000/FFFFFF.png"`  
     - `seed`: `-1`  
     - `image`: `"https://picsum.photos/512/512"`  
     - `model`: `"dev"`  
     - `width`: `512`  
     - `height`: `512`  
     - `prompt`: `"Create something amazing"`  
     - `go_fast`: `false`  
     - `extra_lora`: `""`  
     - `lora_scale`: `1`  
     - `megapixels`: `"1"`  
     - `num_outputs`: `1`  
     - `aspect_ratio`: `"1:1"`  
     - `output_format`: `"webp"`  
     - `guidance_scale`: `3`  
     - `output_quality`: `80`  
     - `prompt_strength`: `0.8`  
     - `extra_lora_scale`: `1`  
     - `replicate_weights`: `""`  
     - `num_inference_steps`: `28`  
     - `disable_safety_checker`: `false`  
   - Connect output of Set API Token to this node  

4. **Create HTTP Request Node "Create Other Prediction"**  
   - Type: HTTP Request  
   - HTTP Method: POST  
   - URL: `https://api.replicate.com/v1/predictions`  
   - Headers:  
     - `Authorization`: `Bearer {{ $json.api_token }}`  
     - `Prefer`: `wait`  
   - Body Content Type: JSON  
   - Body: JSON object with:  
     - `version`: `"settyan/flash-v2.0.0-beta.10:8a5e4f716f0a4acfb122d12d120f9c14a0f0addc05dcb264dea657595f39b9bf"`  
     - `input`: all parameters from previous node using expressions, e.g. `"mask": "{{ $json.mask }}"` etc.  
   - Connect output of Set Other Parameters to this node  

5. **Create Code Node "Log Request"**  
   - Type: Code (JavaScript)  
   - Code:  
     ```javascript
     const data = $input.all()[0].json;
     console.log('settyan/flash-v2.0.0-beta.10 Request:', {
       timestamp: new Date().toISOString(),
       prediction_id: data.id,
       model_type: 'other'
     });
     return $input.all();
     ```  
   - Connect output of Create Other Prediction to this node  

6. **Create Wait Node "Wait 5s"**  
   - Type: Wait  
   - Duration: 5 seconds  
   - Connect output of Log Request to this node  

7. **Create HTTP Request Node "Check Status"**  
   - Type: HTTP Request  
   - HTTP Method: GET  
   - URL: `https://api.replicate.com/v1/predictions/{{ $('Create Other Prediction').item.json.id }}`  
   - Headers:  
     - `Authorization`: `Bearer {{ $('Set API Token').item.json.api_token }}`  
   - Response format: JSON  
   - Connect output of Wait 5s to this node  
   - Also, connect output of Wait 10s (step 11) to this node to form polling loop  

8. **Create If Node "Is Complete?"**  
   - Type: If  
   - Condition: `$json.status` equals `"succeeded"` (case sensitive false)  
   - Connect output of Check Status to this node  

9. **Create Set Node "Success Response"**  
   - Type: Set  
   - Assign field `response` as an object:  
     ```json
     {
       "success": true,
       "result_url": "{{ $json.output }}",
       "prediction_id": "{{ $json.id }}",
       "status": "{{ $json.status }}",
       "message": "Other generated successfully"
     }
     ```  
   - Connect True output of Is Complete? to this node  

10. **Create If Node "Has Failed?"**  
    - Type: If  
    - Condition: `$json.status` equals `"failed"`  
    - Connect False output of Is Complete? to this node  

11. **Create Set Node "Error Response"**  
    - Type: Set  
    - Assign field `response` as an object:  
      ```json
      {
        "success": false,
        "error": "{{ $json.error || 'Other generation failed' }}",
        "prediction_id": "{{ $json.id }}",
        "status": "{{ $json.status }}",
        "message": "Failed to generate other"
      }
      ```  
    - Connect True output of Has Failed? to this node  

12. **Create Wait Node "Wait 10s"**  
    - Type: Wait  
    - Duration: 10 seconds  
    - Connect False output of Has Failed? to this node  

13. **Connect Wait 10s output back to Check Status** to form the polling loop  

14. **Create Set Node "Display Result"**  
    - Type: Set  
    - Assign field `final_result` to the incoming `response` object  
    - Connect outputs of Success Response and Error Response to this node  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                  | Context or Link                                                                                              |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| =======================================<br> FLASH-V2.0.0-BETA.10 GENERATOR<br>For any questions or support, please contact:<br>Yaron@nofluff.online<br>Explore more tips and tutorials here:<br>- YouTube: https://www.youtube.com/@YaronBeen/videos<br>- LinkedIn: https://www.linkedin.com/in/yaronbeen/ | Sticky Note9 content at workflow start                                                                      |
| Comprehensive documentation, usage instructions, parameter reference, troubleshooting guide, model and API links.<br><br>Includes detailed explanations of parameters, workflow blocks, benefits, quick start instructions, and best practices.<br><br>Links:<br>- Model Docs: https://replicate.com/settyan/flash-v2.0.0-beta.10<br>- Replicate API Docs: https://replicate.com/docs<br>- n8n Docs: https://docs.n8n.io | Sticky Note4 content covering full workflow description and resources                                        |

---

This document fully details the structure, configuration, and logic of the "Create Images from Text Prompts using Flash V2.0.0 Beta 10 and Replicate" n8n workflow. It enables both manual reconstruction and automated parsing for modification or integration into other systems.