Image Processing & Generation with Replicate's Lemaar Door Mockedup AI Model

https://n8nworkflows.xyz/workflows/image-processing---generation-with-replicate-s-lemaar-door-mockedup-ai-model-6798


# Image Processing & Generation with Replicate's Lemaar Door Mockedup AI Model

### 1. Workflow Overview

This workflow automates the generation of images or related media using the Replicate API with the model "creativeathive/lemaar-door-mockedup." It is designed for scenarios where users want to create or manipulate images by providing parameters such as masks, prompts, and other generation settings. The workflow handles authentication, parameter configuration, prediction request submission, asynchronous status polling, and response handling, including success and error management.

The workflow is logically organized into the following blocks:

- **1.1 Input Reception and Initialization:** Manual trigger and setting the API token.
- **1.2 Parameter Configuration:** Setting all required and optional parameters needed for the model request.
- **1.3 Prediction Creation:** Sending generation requests to the Replicate API.
- **1.4 Prediction Status Polling and Control Flow:** Waiting, checking, and deciding if the prediction is complete, failed, or still running; includes retry logic.
- **1.5 Result Processing:** Handling success or failure responses and preparing the final output.
- **1.6 Logging and Monitoring:** Logging prediction details for tracking and debugging.
- **1.7 Documentation and User Guidance:** Sticky notes containing detailed documentation, usage instructions, and contact information.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Initialization

**Overview:**  
Starts the workflow execution via manual trigger and sets the API token required for authentication with the Replicate API.

**Nodes Involved:**  
- Manual Trigger  
- Set API Token

**Node Details:**  
- **Manual Trigger**  
  - Type: Manual trigger node  
  - Role: Start point for the workflow; user initiates generation manually.  
  - Config: No parameters, just triggers the workflow on user action.  
  - Input: None  
  - Output: Triggers "Set API Token" node.  
  - Edge Cases: None significant; user must manually start workflow.

- **Set API Token**  
  - Type: Set node  
  - Role: Stores the Replicate API token in a variable for downstream use.  
  - Config: Hardcoded placeholder string `"YOUR_REPLICATE_API_TOKEN"` to be replaced by user.  
  - Variables: Assigns `api_token` string with the API token.  
  - Input: Manual Trigger output  
  - Output: Passes token forward to "Set Other Parameters" node.  
  - Failure: If token is invalid or missing, downstream API calls will fail authentication.

---

#### 2.2 Parameter Configuration

**Overview:**  
Sets and consolidates all parameters required by the model API, including image URLs, generation settings, and optional flags.

**Nodes Involved:**  
- Set Other Parameters

**Node Details:**  
- **Set Other Parameters**  
  - Type: Set node  
  - Role: Defines all input parameters for the Replicate model prediction, including required and optional parameters.  
  - Configuration:  
    - Copies `api_token` from previous node.  
    - Sets default values for parameters such as `mask`, `seed`, `image`, `model`, `width`, `height`, `prompt`, `go_fast`, and many others.  
    - Parameters include image URLs, numeric values (e.g., width, height, seed), booleans (e.g., `go_fast`, `disable_safety_checker`), and strings.  
    - Defaults are sensible for testing (e.g., placeholder images, prompt "Create something amazing").  
  - Variables used in expressions: Copies `api_token` value dynamically.  
  - Input: From "Set API Token"  
  - Output: Passes all parameters to "Create Other Prediction"  
  - Edge Cases: Incorrect parameter types or missing required parameters can cause API errors.

---

#### 2.3 Prediction Creation

**Overview:**  
Submits a POST request to the Replicate API to start the image generation with provided parameters, and receives a prediction ID to track the process.

**Nodes Involved:**  
- Create Other Prediction

**Node Details:**  
- **Create Other Prediction**  
  - Type: HTTP Request node  
  - Role: Sends a POST request to Replicate’s `/v1/predictions` endpoint to create a new prediction.  
  - Configuration:  
    - URL: `https://api.replicate.com/v1/predictions`  
    - Method: POST  
    - Headers: Authorization Bearer token from `api_token`  
    - Body: JSON formatted with model version and input parameters sourced from the previous node’s data.  
    - Request waits for response (`Prefer: wait`) so it receives initial prediction data immediately.  
  - Input: Parameters from "Set Other Parameters"  
  - Output: Contains prediction ID and initial status.  
  - Failures: Network errors, invalid token, malformed parameters, API rate limits.

---

#### 2.4 Prediction Status Polling and Control Flow

**Overview:**  
This block waits a short time, polls the Replicate API for the prediction status, and decides whether the prediction has succeeded, failed, or needs further waiting and retrying.

**Nodes Involved:**  
- Log Request  
- Wait 5s  
- Check Status  
- Is Complete? (If node)  
- Has Failed? (If node)  
- Wait 10s

**Node Details:**  
- **Log Request**  
  - Type: Code node  
  - Role: Logs basic prediction info (timestamp, prediction ID, model type) for monitoring/debugging.  
  - Configuration: JavaScript code logging to console; passes data unchanged downstream.  
  - Input: Output of "Create Other Prediction"  
  - Output: Triggers "Wait 5s"  
  - Failures: None expected unless console unavailable.

- **Wait 5s**  
  - Type: Wait node  
  - Role: Delays the workflow by 5 seconds before polling status to allow time for prediction progress.  
  - Input: From "Log Request"  
  - Output: Triggers "Check Status"  
  - Failures: None expected.

- **Check Status**  
  - Type: HTTP Request node  
  - Role: Fetches the current status of the prediction from the Replicate API using the prediction ID.  
  - Config:  
    - URL dynamically constructed with prediction ID from "Create Other Prediction"  
    - Method: GET  
    - Headers: Authorization with API token  
  - Input: From "Wait 5s"  
  - Output: Passes current prediction status JSON  
  - Failures: Network issues, invalid prediction ID, token expiry.

- **Is Complete?** (If node)  
  - Type: If node  
  - Role: Checks if the prediction status equals `"succeeded"`.  
  - Input: From "Check Status"  
  - Output:  
    - True branch: triggers "Success Response"  
    - False branch: triggers "Has Failed?"  
  - Failures: Expression errors if JSON paths are incorrect.

- **Has Failed?** (If node)  
  - Type: If node  
  - Role: Checks if the prediction status equals `"failed"`.  
  - Input: From "Is Complete?" False branch  
  - Output:  
    - True branch: triggers "Error Response"  
    - False branch: triggers "Wait 10s" (for retry)  
  - Failures: Same as above.

- **Wait 10s**  
  - Type: Wait node  
  - Role: Delays 10 seconds before retrying status check to avoid excessive API calls.  
  - Input: From "Has Failed?" False branch  
  - Output: Loops back to "Check Status" node to poll again.  
  - Failures: None expected.

**Notes:** This loop continues polling until the prediction is either successful or failed, implementing retry delays and failure handling.

---

#### 2.5 Result Processing

**Overview:**  
Prepares the final structured JSON response for both success and failure cases and passes it downstream for display or further use.

**Nodes Involved:**  
- Success Response  
- Error Response  
- Display Result

**Node Details:**  
- **Success Response**  
  - Type: Set node  
  - Role: Builds a JSON object indicating success, including prediction output URL, ID, status, and a message.  
  - Input: From "Is Complete?" True branch  
  - Output: Passes response to "Display Result"  
  - Variables: Includes `result_url` from prediction output JSON.  
  - Failures: If output URL is missing or malformed.

- **Error Response**  
  - Type: Set node  
  - Role: Builds a JSON object indicating failure, including error message, prediction ID, and status.  
  - Input: From "Has Failed?" True branch  
  - Output: Passes to "Display Result"  
  - Variables: Uses `$json.error` if present, else generic failure message.  
  - Failures: None expected.

- **Display Result**  
  - Type: Set node  
  - Role: Final node that sets the `final_result` variable to contain the structured response for use by downstream systems or UI.  
  - Input: From either "Success Response" or "Error Response"  
  - Output: Workflow end output.  
  - Failures: None expected.

---

#### 2.6 Logging and Monitoring

**Overview:**  
Logs prediction requests details for monitoring purposes.

**Nodes Involved:**  
- Log Request (already covered in 2.4)

---

#### 2.7 Documentation and User Guidance

**Overview:**  
Provides detailed documentation, parameter references, instructions, and contact information embedded as sticky notes for workflow maintainers and users.

**Nodes Involved:**  
- Sticky Note9  
- Sticky Note4

**Node Details:**  
- **Sticky Note9**  
  - Content: Contact info for support, YouTube and LinkedIn links of the author.  
  - Purpose: Easy access to help and additional resources.

- **Sticky Note4**  
  - Content: Extensive documentation on the model, parameters, workflow components, usage instructions, troubleshooting, and links to relevant resources including Replicate and n8n docs.  
  - Purpose: Comprehensive reference for users and developers to understand and customize the workflow.

---

### 3. Summary Table

| Node Name            | Node Type          | Functional Role                                   | Input Node(s)                 | Output Node(s)               | Sticky Note                                                                                              |
|----------------------|--------------------|-------------------------------------------------|------------------------------|-----------------------------|--------------------------------------------------------------------------------------------------------|
| Manual Trigger       | Manual Trigger     | Starts the workflow                              |                              | Set API Token                | See Sticky Note4 for full workflow overview and instructions                                           |
| Set API Token         | Set                | Stores Replicate API token                       | Manual Trigger               | Set Other Parameters         | See Sticky Note4                                                                                        |
| Set Other Parameters  | Set                | Defines model input parameters                   | Set API Token                | Create Other Prediction      | See Sticky Note4                                                                                        |
| Create Other Prediction | HTTP Request      | Sends prediction request to Replicate API       | Set Other Parameters         | Log Request                 | See Sticky Note4                                                                                        |
| Log Request           | Code               | Logs prediction metadata for monitoring         | Create Other Prediction      | Wait 5s                     | See Sticky Note4                                                                                        |
| Wait 5s               | Wait               | Delays 5 seconds before status check             | Log Request                 | Check Status                | See Sticky Note4                                                                                        |
| Check Status          | HTTP Request       | Polls prediction status from Replicate API       | Wait 5s, Wait 10s            | Is Complete?                | See Sticky Note4                                                                                        |
| Is Complete?          | If                 | Checks if prediction succeeded                    | Check Status                | Success Response, Has Failed? | See Sticky Note4                                                                                        |
| Has Failed?           | If                 | Checks if prediction failed                       | Is Complete? (False branch) | Error Response, Wait 10s     | See Sticky Note4                                                                                        |
| Wait 10s              | Wait               | Delays 10 seconds before retrying status check   | Has Failed? (False branch)   | Check Status                | See Sticky Note4                                                                                        |
| Success Response      | Set                | Builds success JSON response                       | Is Complete? (True branch)   | Display Result              | See Sticky Note4                                                                                        |
| Error Response        | Set                | Builds failure JSON response                       | Has Failed? (True branch)    | Display Result              | See Sticky Note4                                                                                        |
| Display Result        | Set                | Sets final response object                         | Success Response, Error Response |                             | See Sticky Note4                                                                                        |
| Sticky Note9          | Sticky Note        | Contact info and support links                     |                              |                             | Contact: Yaron@nofluff.online, YouTube: https://www.youtube.com/@YaronBeen/videos, LinkedIn profile    |
| Sticky Note4          | Sticky Note        | Full workflow documentation and instructions      |                              |                             | Detailed parameter guide, usage, troubleshooting, and resources links                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: To manually start the workflow.

2. **Create Set Node: "Set API Token"**  
   - Type: Set  
   - Add field `api_token` of type string.  
   - Value: `"YOUR_REPLICATE_API_TOKEN"` (replace with actual token later)  
   - Connect Manual Trigger → Set API Token.

3. **Create Set Node: "Set Other Parameters"**  
   - Type: Set  
   - Assign all required and optional parameters used by the model:  
     - `api_token`: Reference from "Set API Token" (`={{ $('Set API Token').item.json.api_token }}`)  
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
   - Connect Set API Token → Set Other Parameters.

4. **Create HTTP Request Node: "Create Other Prediction"**  
   - Method: POST  
   - URL: `https://api.replicate.com/v1/predictions`  
   - Headers:  
     - `Authorization`: `Bearer {{$json.api_token}}`  
     - `Prefer`: `wait`  
   - Body (JSON):  
     ```json
     {
       "version": "creativeathive/lemaar-door-mockedup:4980230b6447af824eef74df925bfbedcaab3ccddddfff4d4e83774683ce508c",
       "input": {
         "mask": "{{ $json.mask }}",
         "seed": {{ $json.seed }},
         "image": "{{ $json.image }}",
         "model": "{{ $json.model }}",
         "width": {{ $json.width }},
         "height": {{ $json.height }},
         "prompt": "{{ $json.prompt }}",
         "go_fast": {{ $json.go_fast }},
         "extra_lora": "{{ $json.extra_lora }}",
         "lora_scale": {{ $json.lora_scale }},
         "megapixels": "{{ $json.megapixels }}",
         "num_outputs": {{ $json.num_outputs }},
         "aspect_ratio": "{{ $json.aspect_ratio }}",
         "output_format": "{{ $json.output_format }}",
         "guidance_scale": {{ $json.guidance_scale }},
         "output_quality": {{ $json.output_quality }},
         "prompt_strength": {{ $json.prompt_strength }},
         "extra_lora_scale": {{ $json.extra_lora_scale }},
         "replicate_weights": "{{ $json.replicate_weights }}",
         "num_inference_steps": {{ $json.num_inference_steps }},
         "disable_safety_checker": {{ $json.disable_safety_checker }}
       }
     }
     ```  
   - Connect Set Other Parameters → Create Other Prediction.

5. **Create Code Node: "Log Request"**  
   - JavaScript:
     ```js
     const data = $input.all()[0].json;
     console.log('creativeathive/lemaar-door-mockedup Request:', {
       timestamp: new Date().toISOString(),
       prediction_id: data.id,
       model_type: 'other'
     });
     return $input.all();
     ```
   - Connect Create Other Prediction → Log Request.

6. **Create Wait Node: "Wait 5s"**  
   - Wait for 5 seconds.  
   - Connect Log Request → Wait 5s.

7. **Create HTTP Request Node: "Check Status"**  
   - Method: GET  
   - URL: `https://api.replicate.com/v1/predictions/{{ $('Create Other Prediction').item.json.id }}`  
   - Headers:  
     - `Authorization`: `Bearer {{ $('Set API Token').item.json.api_token }}`  
   - Connect Wait 5s → Check Status. Also connect Wait 10s → Check Status (for retry loop).

8. **Create If Node: "Is Complete?"**  
   - Condition: `$json.status === "succeeded"`  
   - True: Connect to Success Response  
   - False: Connect to Has Failed?  
   - Connect Check Status → Is Complete?.

9. **Create If Node: "Has Failed?"**  
   - Condition: `$json.status === "failed"`  
   - True: Connect to Error Response  
   - False: Connect to Wait 10s (retry)  
   - Connect Is Complete? (False branch) → Has Failed?.

10. **Create Wait Node: "Wait 10s"**  
    - Wait 10 seconds  
    - Connect Has Failed? (False branch) → Wait 10s.

11. **Create Set Node: "Success Response"**  
    - Assign a JSON object to variable `response`:  
      ```json
      {
        "success": true,
        "result_url": $json.output,
        "prediction_id": $json.id,
        "status": $json.status,
        "message": "Other generated successfully"
      }
      ```  
    - Connect Is Complete? (True branch) → Success Response.

12. **Create Set Node: "Error Response"**  
    - Assign a JSON object to variable `response`:  
      ```json
      {
        "success": false,
        "error": $json.error || "Other generation failed",
        "prediction_id": $json.id,
        "status": $json.status,
        "message": "Failed to generate other"
      }
      ```  
    - Connect Has Failed? (True branch) → Error Response.

13. **Create Set Node: "Display Result"**  
    - Assign variable `final_result` with `response` from either Success Response or Error Response.  
    - Connect Success Response → Display Result  
    - Connect Error Response → Display Result.

14. **Add Sticky Notes:**  
    - Create two sticky notes with the detailed documentation and contact info as per Sticky Note4 and Sticky Note9.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                               | Context or Link                                                                                     |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| For any questions or support, please contact: Yaron@nofluff.online                                                                                                                                                                                       | Support email                                                                                      |
| Explore more tips and tutorials: YouTube: https://www.youtube.com/@YaronBeen/videos, LinkedIn: https://www.linkedin.com/in/yaronbeen/                                                                                                                     | Author social media                                                                               |
| Model Documentation: https://replicate.com/creativeathive/lemaar-door-mockedup                                                                                                                                                                             | Model official docs                                                                               |
| Replicate API Documentation: https://replicate.com/docs                                                                                                                                                                                                    | API reference                                                                                     |
| n8n Documentation: https://docs.n8n.io                                                                                                                                                                                                                      | Workflow automation platform docs                                                                |
| Workflow includes extensive parameter guide, usage instructions, troubleshooting tips, and recommended best practices for API token management and error handling.                                                                                           | Embedded in Sticky Note4                                                                           |

---

This completes the detailed reference documentation for the "Image Processing & Generation with Replicate's Lemaar Door Mockedup AI Model" workflow. It enables understanding, reproduction, modification, and troubleshooting of the entire workflow with explicit detail.