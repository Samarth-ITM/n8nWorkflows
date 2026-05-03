Create Images from Text Prompts using Lorealcantara and Replicate

https://n8nworkflows.xyz/workflows/create-images-from-text-prompts-using-lorealcantara-and-replicate-6862


# Create Images from Text Prompts using Lorealcantara and Replicate

### 1. Workflow Overview

This workflow is designed to generate images from text prompts using the AI model "lorealcantara" hosted on the Replicate platform. It automates the entire image generation process, including parameter setup, API authentication, prediction request submission, and result retrieval with status polling. The target use case is artistic or creative image generation based on textual descriptions, suitable for users wanting to leverage the "ligua033/lorealcantara" model via n8n automation.

The workflow is logically organized into these main blocks:

- **1.1 Input Initialization and Configuration:** Accepts manual trigger input, sets API authentication token, and prepares all model parameters required for the generation request.

- **1.2 Prediction Request Creation:** Sends a POST request to Replicate API to initiate image generation with the configured parameters.

- **1.3 Prediction Status Polling:** Implements a wait-and-check loop to poll the status of the generation until completion or failure, with retry delays.

- **1.4 Result Handling:** Upon completion, routes successful outputs or error responses to appropriate handlers and formats the final output.

- **1.5 Logging and Monitoring:** Logs details of each generation request for traceability and debugging.

- **1.6 Documentation and Support Notes:** Provides detailed sticky notes with instructions, parameter references, and support contacts.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Initialization and Configuration

**Overview:**  
This block starts the workflow manually, sets the Replicate API authentication token, and configures all required and optional parameters for the image generation model.

**Nodes Involved:**  
- Manual Trigger  
- Set API Token  
- Set Other Parameters

**Node Details:**  

- **Manual Trigger**  
  - Type: Trigger node (manual)  
  - Role: Starts the workflow execution manually by the user.  
  - Configuration: No parameters; activated by user click.  
  - Inputs: None  
  - Outputs: Connects to "Set API Token"  
  - Edge Cases: None typical; user must start manually.

- **Set API Token**  
  - Type: Set node  
  - Role: Stores the Replicate API token as a workflow variable.  
  - Configuration: Hardcoded string placeholder "YOUR_REPLICATE_API_TOKEN" that must be replaced with a valid token.  
  - Inputs: From "Manual Trigger"  
  - Outputs: Connects to "Set Other Parameters"  
  - Edge Cases: Failure if API token is invalid or missing.  
  - Notes: Critical for authentication.

- **Set Other Parameters**  
  - Type: Set node  
  - Role: Defines all required and optional parameters for the image generation request, including prompt, image URLs, model version, dimensions, output format, inference steps, and more.  
  - Configuration:  
    - Reads API token from "Set API Token" node via expression.  
    - Defaults set for parameters such as prompt ("Create something amazing"), width/height (512), output format (webp), number of outputs (1), etc.  
  - Inputs: From "Set API Token"  
  - Outputs: Connects to "Create Other Prediction"  
  - Edge Cases: Incorrect parameter types or missing required parameters (e.g., prompt) may cause API errors.

---

#### 2.2 Prediction Request Creation

**Overview:**  
Sends a POST request to the Replicate API to create a new prediction job with the configured parameters.

**Nodes Involved:**  
- Create Other Prediction

**Node Details:**  

- **Create Other Prediction**  
  - Type: HTTP Request node  
  - Role: Submits a generation request to Replicate API endpoint `/v1/predictions` using the "ligua033/lorealcantara" model version.  
  - Configuration:  
    - Method: POST  
    - URL: https://api.replicate.com/v1/predictions  
    - Headers: Authorization Bearer token from input JSON, Prefer header set to "wait" for synchronous response.  
    - Body: JSON payload containing model version and all input parameters interpolated from the previous node.  
  - Inputs: From "Set Other Parameters"  
  - Outputs: Connects to "Log Request"  
  - Edge Cases:  
    - HTTP errors, authentication failure if token invalid.  
    - API rate limits or quota exceeded.  
    - Malformed request parameters causing API rejection.

---

#### 2.3 Prediction Status Polling

**Overview:**  
Implements a polling mechanism to repeatedly check the status of the prediction until it is completed or failed. It waits between checks to avoid excessive API calls.

**Nodes Involved:**  
- Log Request  
- Wait 5s  
- Check Status  
- Is Complete?  
- Has Failed?  
- Wait 10s

**Node Details:**  

- **Log Request**  
  - Type: Code node  
  - Role: Logs request details such as timestamp, prediction ID, and model type for monitoring.  
  - Configuration: Custom JavaScript code that logs data to console.  
  - Inputs: From "Create Other Prediction"  
  - Outputs: Connects to "Wait 5s"  
  - Edge Cases: Logging failure does not block workflow; used for diagnostics.

- **Wait 5s**  
  - Type: Wait node  
  - Role: Pauses workflow for 5 seconds before the next status check.  
  - Configuration: Wait 5 seconds.  
  - Inputs: From "Log Request"  
  - Outputs: Connects to "Check Status"  
  - Edge Cases: None typical.

- **Check Status**  
  - Type: HTTP Request node  
  - Role: Queries the Replicate API `/v1/predictions/{id}` to check prediction status.  
  - Configuration:  
    - Method: GET  
    - URL constructed dynamically from `Create Other Prediction` node’s prediction ID.  
    - Authorization header with API token from "Set API Token".  
  - Inputs: From "Wait 5s" and also from "Wait 10s" after failure retries.  
  - Outputs: Connects to "Is Complete?"  
  - Edge Cases: API errors, network issues.

- **Is Complete?**  
  - Type: If node  
  - Role: Checks if prediction status equals "succeeded".  
  - Configuration: Condition `status == "succeeded"`.  
  - Inputs: From "Check Status"  
  - Outputs:  
    - True branch: Connects to "Success Response"  
    - False branch: Connects to "Has Failed?"  
  - Edge Cases: Status might be other states like "processing" or "failed".

- **Has Failed?**  
  - Type: If node  
  - Role: Checks if prediction status equals "failed".  
  - Configuration: Condition `status == "failed"`.  
  - Inputs: From "Is Complete?" false branch  
  - Outputs:  
    - True branch: Connects to "Error Response"  
    - False branch: Connects to "Wait 10s" (retry)  
  - Edge Cases: Prediction could be in pending states.

- **Wait 10s**  
  - Type: Wait node  
  - Role: Waits 10 seconds before retrying status check to avoid excessive polling.  
  - Configuration: Wait 10 seconds.  
  - Inputs: From "Has Failed?" false branch  
  - Outputs: Connects back to "Check Status"  
  - Edge Cases: None typical.

---

#### 2.4 Result Handling

**Overview:**  
Processes the final prediction output or error, formats a structured response object, and prepares it for downstream consumption or output.

**Nodes Involved:**  
- Success Response  
- Error Response  
- Display Result

**Node Details:**  

- **Success Response**  
  - Type: Set node  
  - Role: Creates a response object for successful generation containing success flag, output URL, prediction ID, status, and a success message.  
  - Configuration: Assigns an object with keys: success=true, result_url, prediction_id, status, and message.  
  - Inputs: From "Is Complete?" true branch  
  - Outputs: Connects to "Display Result"  
  - Edge Cases: Assumes output URL is present in the prediction response.

- **Error Response**  
  - Type: Set node  
  - Role: Creates a response object for failure with success=false, error message, prediction ID, status, and failure message.  
  - Configuration: Assigns object with error details, falling back to generic message if no specific error.  
  - Inputs: From "Has Failed?" true branch  
  - Outputs: Connects to "Display Result"  
  - Edge Cases: Handles cases with missing error details gracefully.

- **Display Result**  
  - Type: Set node  
  - Role: Assigns the final response object (success or error) to a single field `final_result` for output or further use.  
  - Configuration: Assigns `final_result` from the previous node's `response` field.  
  - Inputs: From both "Success Response" and "Error Response"  
  - Outputs: End of workflow  
  - Edge Cases: None typical.

---

#### 2.5 Logging and Monitoring

**Overview:**  
Logs metadata about each generation request for monitoring and debugging purposes.

**Nodes Involved:**  
- Log Request (already described in 2.3)

**Node Details:**  
See 2.3 "Log Request" node.

---

#### 2.6 Documentation and Support Notes

**Overview:**  
Provides detailed explanations, parameter references, and contact details as sticky notes attached alongside the workflow for user guidance.

**Nodes Involved:**  
- Sticky Note9  
- Sticky Note4

**Node Details:**  

- **Sticky Note9**  
  - Type: Sticky Note  
  - Role: Displays header and contact info for support, including YouTube and LinkedIn links for tutorials by Yaron.  
  - Content highlights:  
    - Title: LOREALCANTARA GENERATOR  
    - Contact: Yaron@nofluff.online  
    - YouTube and LinkedIn URLs  
  - Position: Top-left of the workflow

- **Sticky Note4**  
  - Type: Sticky Note  
  - Role: Extensive workflow documentation, parameter guide, troubleshooting, quick start instructions, and external resource links.  
  - Includes:  
    - Model overview and API endpoint  
    - Parameter explanations (required and optional)  
    - Workflow component descriptions  
    - Benefits and usage instructions  
    - Troubleshooting tips  
    - Links to Replicate API docs, model docs, and n8n docs  
  - Positioned along the left side, vertically spanning much of the workflow canvas.

---

### 3. Summary Table

| Node Name              | Node Type           | Functional Role                         | Input Node(s)            | Output Node(s)          | Sticky Note                                                                                                     |
|------------------------|---------------------|---------------------------------------|--------------------------|-------------------------|----------------------------------------------------------------------------------------------------------------|
| Manual Trigger         | Manual Trigger       | Starts workflow manually               | None                     | Set API Token            |                                                                                                                |
| Set API Token          | Set                 | Sets Replicate API token               | Manual Trigger           | Set Other Parameters     |                                                                                                                |
| Set Other Parameters   | Set                 | Configures all model input parameters  | Set API Token            | Create Other Prediction  |                                                                                                                |
| Create Other Prediction| HTTP Request        | Sends generation request to Replicate | Set Other Parameters     | Log Request              |                                                                                                                |
| Log Request            | Code                | Logs request metadata                   | Create Other Prediction  | Wait 5s                  |                                                                                                                |
| Wait 5s                | Wait                | Waits 5 seconds before status check    | Log Request              | Check Status             |                                                                                                                |
| Check Status           | HTTP Request        | Retrieves prediction status             | Wait 5s, Wait 10s        | Is Complete?             |                                                                                                                |
| Is Complete?           | If                  | Checks if prediction succeeded         | Check Status             | Success Response, Has Failed? |                                                                                                                |
| Has Failed?            | If                  | Checks if prediction failed             | Is Complete?             | Error Response, Wait 10s |                                                                                                                |
| Wait 10s               | Wait                | Waits 10 seconds before retrying       | Has Failed?              | Check Status             |                                                                                                                |
| Success Response       | Set                 | Formats successful result response     | Is Complete?             | Display Result           |                                                                                                                |
| Error Response         | Set                 | Formats failure response                | Has Failed?              | Display Result           |                                                                                                                |
| Display Result         | Set                 | Assigns final result for output         | Success Response, Error Response | None               |                                                                                                                |
| Sticky Note9           | Sticky Note         | Displays support & contact info         | None                     | None                    | ======================================= LOREALCANTARA GENERATOR Contact: Yaron@nofluff.online YouTube: https://www.youtube.com/@YaronBeen/videos LinkedIn: https://www.linkedin.com/in/yaronbeen/ |
| Sticky Note4           | Sticky Note         | Detailed documentation & instructions  | None                     | None                    | Extensive explanation of model, parameters, workflow, troubleshooting, and useful resource links             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add Manual Trigger node:**  
   - Name: "Manual Trigger"  
   - Purpose: Starts the workflow manually.

3. **Add Set node for API Token:**  
   - Name: "Set API Token"  
   - Add a string field named `api_token` with value `"YOUR_REPLICATE_API_TOKEN"` (replace this with a valid Replicate API token).  
   - Connect "Manual Trigger" output to this node.

4. **Add Set node for Other Parameters:**  
   - Name: "Set Other Parameters"  
   - Add the following fields and default values:  
     - api_token (string): `={{ $('Set API Token').item.json.api_token }}` (expression to inherit token)  
     - mask (string): `"https://via.placeholder.com/512x512/000000/FFFFFF.png"`  
     - seed (number): `-1`  
     - image (string): `"https://picsum.photos/512/512"`  
     - model (string): `"dev"`  
     - width (number): `512`  
     - height (number): `512`  
     - prompt (string): `"Create something amazing"`  
     - go_fast (boolean): `false`  
     - extra_lora (string): `""`  
     - lora_scale (number): `1`  
     - megapixels (string): `"1"`  
     - num_outputs (number): `1`  
     - aspect_ratio (string): `"1:1"`  
     - output_format (string): `"webp"`  
     - guidance_scale (number): `3`  
     - output_quality (number): `80`  
     - prompt_strength (number): `0.8`  
     - extra_lora_scale (number): `1`  
     - replicate_weights (string): `""`  
     - num_inference_steps (number): `28`  
     - disable_safety_checker (boolean): `false`  
   - Connect "Set API Token" output to this node.

5. **Add HTTP Request node to create prediction:**  
   - Name: "Create Other Prediction"  
   - HTTP Method: POST  
   - URL: `https://api.replicate.com/v1/predictions`  
   - Headers:  
     - Authorization: `Bearer {{ $json.api_token }}`  
     - Prefer: `wait`  
   - Body (JSON):  
     ```json
     {
       "version": "ligua033/lorealcantara:21c8ba35556f26425991a971a1419e264fccc020cd32170d54924d1958f2e397",
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
   - Connect "Set Other Parameters" output to this node.

6. **Add Code node for logging:**  
   - Name: "Log Request"  
   - Code:  
     ```javascript
     const data = $input.all()[0].json;
     console.log('ligua033/lorealcantara Request:', {
       timestamp: new Date().toISOString(),
       prediction_id: data.id,
       model_type: 'other'
     });
     return $input.all();
     ```  
   - Connect "Create Other Prediction" output to this node.

7. **Add Wait node:**  
   - Name: "Wait 5s"  
   - Wait time: 5 seconds  
   - Connect "Log Request" output to this node.

8. **Add HTTP Request node to check status:**  
   - Name: "Check Status"  
   - HTTP Method: GET  
   - URL: `https://api.replicate.com/v1/predictions/{{ $('Create Other Prediction').item.json.id }}`  
   - Headers:  
     - Authorization: `Bearer {{ $('Set API Token').item.json.api_token }}`  
   - Connect "Wait 5s" output to this node.  
   - Also connect "Wait 10s" output (created later) back to this node for retry looping.

9. **Add If node to test completion:**  
   - Name: "Is Complete?"  
   - Condition: Check if `status` field in response equals `"succeeded"`  
   - Connect "Check Status" output to this node.

10. **Add If node to test failure:**  
    - Name: "Has Failed?"  
    - Condition: Check if `status` equals `"failed"`  
    - Connect "Is Complete?" false branch to this node.

11. **Add Wait node:**  
    - Name: "Wait 10s"  
    - Wait time: 10 seconds  
    - Connect "Has Failed?" false branch to this node.

12. **Connect "Wait 10s" output back to "Check Status"** for retry polling loop.

13. **Add Set node for success response:**  
    - Name: "Success Response"  
    - Set field `response` to an object:  
      ```json
      {
        "success": true,
        "result_url": "{{ $json.output }}",
        "prediction_id": "{{ $json.id }}",
        "status": "{{ $json.status }}",
        "message": "Other generated successfully"
      }
      ```  
    - Connect "Is Complete?" true branch to this node.

14. **Add Set node for error response:**  
    - Name: "Error Response"  
    - Set field `response` to an object:  
      ```json
      {
        "success": false,
        "error": "{{ $json.error || 'Other generation failed' }}",
        "prediction_id": "{{ $json.id }}",
        "status": "{{ $json.status }}",
        "message": "Failed to generate other"
      }
      ```  
    - Connect "Has Failed?" true branch to this node.

15. **Add Set node to finalize output:**  
    - Name: "Display Result"  
    - Set field `final_result` to `{{ $json.response }}`  
    - Connect both "Success Response" and "Error Response" outputs to this node.

16. **Add two Sticky Note nodes for documentation:**  
    - Name: "Sticky Note9"  
      - Content: Header with contact and social links  
      - Position: Upper-left corner  
    - Name: "Sticky Note4"  
      - Content: Detailed workflow and parameter documentation, troubleshooting, quick start, and external resource links  
      - Position: Along the left side vertically

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                          | Context or Link                                                                                     |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| For any questions or support, contact Yaron at Yaron@nofluff.online                                                                                                                                                                                                                   | Contact email provided in Sticky Note9                                                           |
| Explore more tips and tutorials on YouTube and LinkedIn: https://www.youtube.com/@YaronBeen/videos and https://www.linkedin.com/in/yaronbeen/                                                                                                                                         | Video and professional profile links from Sticky Note9                                           |
| Model documentation and API details available at https://replicate.com/ligua033/lorealcantara and https://replicate.com/docs                                                                                                                                                        | Official model and API docs referenced in Sticky Note4                                           |
| n8n official documentation: https://docs.n8n.io                                                                                                                                                                                                                                      | For general n8n node usage and workflow design                                                   |
| Replace 'YOUR_REPLICATE_API_TOKEN' with your actual Replicate API token from https://replicate.com/account                                                                                                                                                                            | Critical for authentication and proper API access                                               |
| Recommended to test with default parameters first and monitor logs for troubleshooting.                                                                                                                                                                                               | Best practice from troubleshooting guide                                                        |
| The model supports various image generation parameters including mask, seed, prompt strength, and safety checker disabling for advanced use cases.                                                                                                                                   | Parameter reference detailed in Sticky Note4                                                     |
| This workflow demonstrates a robust polling mechanism with waits and retries to handle asynchronous generation latency and potential API delays.                                                                                                                                     | Important for reliable operation and error resilience                                           |

---

**Disclaimer:**  
The text provided is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All data handled are legal and publicly accessible.