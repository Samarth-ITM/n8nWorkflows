Edit Images with Text Prompts using Bytedance Seededit 3.0 via Replicate API

https://n8nworkflows.xyz/workflows/edit-images-with-text-prompts-using-bytedance-seededit-3-0-via-replicate-api-6794


# Edit Images with Text Prompts using Bytedance Seededit 3.0 via Replicate API

### 1. Workflow Overview

This workflow enables automated image editing using text prompts via the Bytedance Seededit 3.0 model accessed through the Replicate API. It is designed for users who want to input an existing image and a descriptive prompt to generate a modified image reflecting the prompt’s guidance, such as altering lighting, removing objects, or changing styles while preserving original image details.

The workflow is logically divided into these blocks:

- **1.1 Input Initialization and Parameter Setup**  
  Receives manual trigger input and sets essential API tokens and image generation parameters.

- **1.2 Image Generation Request Submission**  
  Sends a prediction request to the Replicate API for the Seededit 3.0 model with configured parameters.

- **1.3 Prediction Status Polling and Monitoring**  
  Periodically checks the status of the image generation prediction until it completes successfully or fails.

- **1.4 Success and Error Handling**  
  Processes the final result or error, formats structured responses, and prepares output.

- **1.5 Logging and Response Delivery**  
  Logs request details for monitoring and delivers the formatted output.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Initialization and Parameter Setup

**Overview:**  
This block initializes the workflow on manual trigger, sets the Replicate API token for authentication, and configures the input parameters needed for the image editing model.

**Nodes Involved:**  
- Manual Trigger  
- Set API Token  
- Set Image Parameters  

**Node Details:**

- **Manual Trigger**  
  - Type: `Manual Trigger`  
  - Role: Entry point for manual workflow execution.  
  - Configuration: Default, no parameters.  
  - Inputs: None  
  - Outputs: Connects to "Set API Token"  
  - Edge Cases: None typical; user must manually trigger to start.

- **Set API Token**  
  - Type: `Set`  
  - Role: Stores the Replicate API token securely in the workflow for authentication in later API calls.  
  - Configuration: A string parameter named `api_token` set to placeholder `"YOUR_REPLICATE_API_TOKEN"` which must be replaced by the user with a valid token.  
  - Inputs: From Manual Trigger  
  - Outputs: To "Set Image Parameters"  
  - Edge Cases: Missing or invalid token will cause authentication failures downstream.

- **Set Image Parameters**  
  - Type: `Set`  
  - Role: Defines the parameters for the image generation including seed, input image URL, prompt text, and guidance scale.  
  - Configuration:  
    - `api_token` copied from "Set API Token" node output  
    - `seed`: -1 (default to random seed, allows reproducibility if changed)  
    - `image`: URL string `"https://picsum.photos/512/512"` (default example image)  
    - `prompt`: `"A beautiful landscape with mountains and a lake at sunset"` (default prompt)  
    - `guidance_scale`: 5.5 (balance of prompt adherence)  
  - Inputs: From "Set API Token"  
  - Outputs: To "Create Image Prediction"  
  - Edge Cases: Invalid URLs, empty prompts, or out-of-range guidance scales may cause API errors.

---

#### 2.2 Image Generation Request Submission

**Overview:**  
Sends a POST request to the Replicate API endpoint with the configured parameters to create a new image prediction job.

**Nodes Involved:**  
- Create Image Prediction  

**Node Details:**

- **Create Image Prediction**  
  - Type: `HTTP Request`  
  - Role: Issues a POST request to `https://api.replicate.com/v1/predictions` to start image generation.  
  - Configuration:  
    - Method: POST  
    - Body (JSON): contains model version and input parameters (`seed`, `image`, `prompt`, `guidance_scale`) dynamically injected from previous node data.  
    - Headers: Authorization with Bearer token from `api_token`, and `Prefer: wait` to request synchronous processing if possible.  
    - Response handled as JSON, with error suppression (`neverError: true`) to avoid workflow crashes on HTTP errors.  
  - Inputs: From "Set Image Parameters"  
  - Outputs: To "Log Request"  
  - Edge Cases: API rate limits, invalid parameters, network errors, or authentication failures.

---

#### 2.3 Prediction Status Polling and Monitoring

**Overview:**  
Implements a polling loop to monitor the prediction status. Waits 5 seconds between checks. If prediction is incomplete, continues polling; if failed, triggers error handling; if succeeded, proceeds to success handling.

**Nodes Involved:**  
- Log Request  
- Wait 5s  
- Check Status  
- Is Complete?  
- Has Failed?  
- Wait 10s  

**Node Details:**

- **Log Request**  
  - Type: `Code`  
  - Role: Logs prediction request details including timestamp, prediction ID, and model type for monitoring/debugging.  
  - Configuration: JavaScript code using n8n input data to log meaningful info to console.  
  - Inputs: From "Create Image Prediction"  
  - Outputs: To "Wait 5s"  
  - Edge Cases: None significant; logging failure won't block workflow.

- **Wait 5s**  
  - Type: `Wait`  
  - Role: Pauses workflow execution for 5 seconds to avoid rapid polling.  
  - Configuration: 5 seconds delay.  
  - Inputs: From "Log Request"  
  - Outputs: To "Check Status"  
  - Edge Cases: None.

- **Check Status**  
  - Type: `HTTP Request`  
  - Role: Retrieves the current status of the prediction using the prediction ID.  
  - Configuration:  
    - GET request to `https://api.replicate.com/v1/predictions/{{prediction_id}}`  
    - Authorization header with Bearer token.  
    - Response parsed as JSON with error suppression.  
  - Inputs: From "Wait 5s"  
  - Outputs: To "Is Complete?"  
  - Edge Cases: Network failures, invalid prediction ID, API limits.

- **Is Complete?**  
  - Type: `If`  
  - Role: Checks if prediction status equals `"succeeded"`.  
  - Configuration: Condition: `status == "succeeded"`  
  - Inputs: From "Check Status"  
  - Outputs:  
    - True branch → "Success Response"  
    - False branch → "Has Failed?"  
  - Edge Cases: Status field missing or unexpected values.

- **Has Failed?**  
  - Type: `If`  
  - Role: Checks if prediction status equals `"failed"`.  
  - Configuration: Condition: `status == "failed"`  
  - Inputs: From "Is Complete?" false branch  
  - Outputs:  
    - True branch → "Error Response"  
    - False branch → "Wait 10s" to retry status check later  
  - Edge Cases: Unexpected intermediate statuses like "starting" or "processing".

- **Wait 10s**  
  - Type: `Wait`  
  - Role: Delays 10 seconds before next status check retry.  
  - Configuration: 10 seconds delay  
  - Inputs: From "Has Failed?" false branch  
  - Outputs: To "Check Status" (loop back)  
  - Edge Cases: None.

---

#### 2.4 Success and Error Handling

**Overview:**  
Formats structured JSON response objects for both successful image generations and error cases.

**Nodes Involved:**  
- Success Response  
- Error Response  

**Node Details:**

- **Success Response**  
  - Type: `Set`  
  - Role: Creates an object containing success status, image URL(s), prediction ID, status, and a success message.  
  - Configuration:  
    - Property `response` set to an object with keys: `success: true`, `image_url` (from prediction output), `prediction_id`, `status`, and a friendly message.  
  - Inputs: From "Is Complete?" true branch  
  - Outputs: To "Display Result"  
  - Edge Cases: Missing output URLs or malformed data.

- **Error Response**  
  - Type: `Set`  
  - Role: Creates an object containing failure status, error messages, prediction ID, status, and a failure message.  
  - Configuration:  
    - Property `response` set to an object with keys: `success: false`, `error` (from JSON or default), `prediction_id`, `status`, and an error message.  
  - Inputs: From "Has Failed?" true branch  
  - Outputs: To "Display Result"  
  - Edge Cases: Missing error details, incomplete JSON.

---

#### 2.5 Logging and Response Delivery

**Overview:**  
Finalizes the workflow by preparing the output response for downstream consumption or user review.

**Nodes Involved:**  
- Display Result  

**Node Details:**

- **Display Result**  
  - Type: `Set`  
  - Role: Sets a property `final_result` with the response object from either success or error handlers to unify output format.  
  - Configuration:  
    - `final_result` assigned from the `response` property of prior node JSON.  
  - Inputs: From "Success Response" and "Error Response"  
  - Outputs: None (end of workflow)  
  - Edge Cases: None significant.

---

### 3. Summary Table

| Node Name             | Node Type       | Functional Role                      | Input Node(s)            | Output Node(s)           | Sticky Note                                                                                                  |
|-----------------------|-----------------|-----------------------------------|--------------------------|--------------------------|--------------------------------------------------------------------------------------------------------------|
| Manual Trigger        | Manual Trigger  | Start workflow manually             | None                     | Set API Token            | See detailed usage and support info in sticky notes                                                         |
| Set API Token         | Set             | Store Replicate API token           | Manual Trigger           | Set Image Parameters     | Replace placeholder with your actual Replicate API token                                                    |
| Set Image Parameters  | Set             | Configure image generation inputs   | Set API Token            | Create Image Prediction  | Contains default prompt and image URL; adjust as needed                                                     |
| Create Image Prediction | HTTP Request   | POST image generation request       | Set Image Parameters     | Log Request              | Sends request to Replicate API, includes auth header                                                        |
| Log Request           | Code            | Log request details for monitoring  | Create Image Prediction  | Wait 5s                  | Logs timestamp, prediction ID, and model type                                                               |
| Wait 5s               | Wait            | Pause before first status check     | Log Request              | Check Status             |                                                                                                              |
| Check Status          | HTTP Request    | Poll prediction status              | Wait 5s, Wait 10s        | Is Complete?             | Uses prediction ID to check status via Replicate API                                                        |
| Is Complete?          | If              | Check if prediction succeeded       | Check Status             | Success Response, Has Failed? |                                                                                                              |
| Has Failed?           | If              | Check if prediction failed          | Is Complete?             | Error Response, Wait 10s |                                                                                                              |
| Wait 10s              | Wait            | Delay before retrying status check  | Has Failed?              | Check Status             |                                                                                                              |
| Success Response      | Set             | Prepare success response JSON       | Is Complete?             | Display Result           |                                                                                                              |
| Error Response        | Set             | Prepare error response JSON         | Has Failed?              | Display Result           |                                                                                                              |
| Display Result        | Set             | Final output preparation            | Success Response, Error Response | None               |                                                                                                              |
| Sticky Note9          | Sticky Note     | Contact and support information     | None                     | None                     | See Section 5 for details                                                                                    |
| Sticky Note4          | Sticky Note     | Comprehensive workflow documentation | None                     | None                     | Contains detailed model description, parameters, instructions, and links                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.

2. **Add a Manual Trigger node**  
   - Name: `Manual Trigger`  
   - No special configuration.

3. **Add a Set node for API Token**  
   - Name: `Set API Token`  
   - Add a string field `api_token` with value `"YOUR_REPLICATE_API_TOKEN"` (replace with your actual token).  
   - Connect `Manual Trigger` output to this node’s input.

4. **Add a Set node for Image Parameters**  
   - Name: `Set Image Parameters`  
   - Define these fields:  
     - `api_token`: Set to expression `{{$node["Set API Token"].json["api_token"]}}`  
     - `seed`: `-1` (number)  
     - `image`: `"https://picsum.photos/512/512"` (string)  
     - `prompt`: `"A beautiful landscape with mountains and a lake at sunset"` (string)  
     - `guidance_scale`: `5.5` (number)  
   - Connect `Set API Token` output to this node.

5. **Add an HTTP Request node to create prediction**  
   - Name: `Create Image Prediction`  
   - HTTP Method: POST  
   - URL: `https://api.replicate.com/v1/predictions`  
   - Authentication: None (token in header)  
   - Headers:  
     - `Authorization`: Expression `=Bearer {{$json["api_token"]}}`  
     - `Prefer`: `wait`  
   - Body content type: JSON  
   - Request body (raw JSON with expressions):  
     ```json
     {
       "version": "bytedance/seededit-3.0:2c4be22ec2c834b160f6587130d5b9d5ba6d498a5203b80ab874f35d0ce73fa6",
       "input": {
         "seed": {{$json["seed"]}},
         "image": "{{$json["image"]}}",
         "prompt": "{{$json["prompt"]}}",
         "guidance_scale": {{$json["guidance_scale"]}}
       }
     }
     ```  
   - Connect `Set Image Parameters` output to this node.

6. **Add a Code node to log request details**  
   - Name: `Log Request`  
   - JavaScript code:  
     ```javascript
     const data = $input.all()[0].json;
     console.log('bytedance/seededit-3.0 Request:', {
       timestamp: new Date().toISOString(),
       prediction_id: data.id,
       model_type: 'image'
     });
     return $input.all();
     ```  
   - Connect `Create Image Prediction` output to this node.

7. **Add a Wait node to delay 5 seconds**  
   - Name: `Wait 5s`  
   - Delay: 5 seconds  
   - Connect `Log Request` output to this node.

8. **Add an HTTP Request node to check prediction status**  
   - Name: `Check Status`  
   - HTTP Method: GET  
   - URL: Expression: `https://api.replicate.com/v1/predictions/{{$node["Create Image Prediction"].json["id"]}}`  
   - Headers:  
     - `Authorization`: Expression `=Bearer {{$node["Set API Token"].json["api_token"]}}`  
   - Connect `Wait 5s` output to this node. Also connect retry path from Wait 10s node (see below).

9. **Add an If node to check if status is "succeeded"**  
   - Name: `Is Complete?`  
   - Condition: `{{$json["status"]}} == "succeeded"`  
   - Connect `Check Status` output to this node.

10. **Add a Set node for success response**  
    - Name: `Success Response`  
    - Set field `response` (object) with:  
      ```json
      {
        "success": true,
        "image_url": $json.output,
        "prediction_id": $json.id,
        "status": $json.status,
        "message": "Image generated successfully"
      }
      ```  
    - Connect `Is Complete?` true branch to this node.

11. **Add a Set node for error response**  
    - Name: `Error Response`  
    - Set field `response` (object) with:  
      ```json
      {
        "success": false,
        "error": $json.error || "Image generation failed",
        "prediction_id": $json.id,
        "status": $json.status,
        "message": "Failed to generate image"
      }
      ```  
    - Connect `Has Failed?` true branch (see next step) to this node.

12. **Add an If node to check if status is "failed"**  
    - Name: `Has Failed?`  
    - Condition: `{{$json["status"]}} == "failed"`  
    - Connect `Is Complete?` false branch to this node.

13. **Add a Wait node for retry delay 10 seconds**  
    - Name: `Wait 10s`  
    - Delay: 10 seconds  
    - Connect `Has Failed?` false branch to this node.

14. **Connect `Wait 10s` output back to `Check Status` for polling loop.**

15. **Add a Set node to unify output**  
    - Name: `Display Result`  
    - Set field `final_result` to `{{$json["response"]}}`  
    - Connect outputs of `Success Response` and `Error Response` to this node.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                 | Context or Link                                                                                          |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| =======================================<br>SEEDEDIT-3.0 GENERATOR<br>=======================================<br>For any questions or support, please contact:<br>Yaron@nofluff.online<br><br>Explore more tips and tutorials here:<br>- YouTube: https://www.youtube.com/@YaronBeen/videos<br>- LinkedIn: https://www.linkedin.com/in/yaronbeen/<br>======================================= | Contact and support information included as a sticky note in the workflow                               |
| **BYTEDANCE/SEEDEDIT-3.0 - IMAGE GENERATION WORKFLOW**<br>Powered by Replicate API and n8n Automation.<br><br>Model Owner: bytedance<br>Model Type: Image Generation<br>API Endpoint: https://api.replicate.com/v1/predictions<br><br>Key Parameters: prompt and image required; seed and guidance_scale optional.<br><br>Quick start:<br>1. Get API key at https://replicate.com<br>2. Replace token in workflow<br>3. Run and monitor results<br><br>More info:<br>- Model Doc: https://replicate.com/bytedance/seededit-3.0<br>- Replicate API: https://replicate.com/docs<br>- n8n Docs: https://docs.n8n.io | Detailed overview, parameter guide, and instructions provided in a large sticky note for user reference |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, a workflow automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.