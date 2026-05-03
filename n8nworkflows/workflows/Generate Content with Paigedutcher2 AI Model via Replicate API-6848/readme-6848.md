Generate Content with Paigedutcher2 AI Model via Replicate API

https://n8nworkflows.xyz/workflows/generate-content-with-paigedutcher2-ai-model-via-replicate-api-6848


# Generate Content with Paigedutcher2 AI Model via Replicate API

### 1. Workflow Overview

This workflow enables the generation of AI-driven content using the Paigedutcher2 model via the Replicate API. It is designed for users who want to programmatically create images or other AI-generated media by specifying various parameters such as prompt, image size, and model options. The workflow automates the entire process from input parameter setup, API authentication, triggering the generation request, monitoring the generation status, handling success or failure responses, and logging for monitoring purposes.

The workflow is logically divided into the following blocks:

- **1.1 Input Initialization and Authentication:** Receives a manual trigger, sets the Replicate API token, and defines all necessary parameters for the generation request.
- **1.2 API Request & Execution:** Sends the generation request to the Replicate API and logs the request details.
- **1.3 Status Monitoring Loop:** Waits and polls the API to check if the content generation is complete or has failed, with retry logic.
- **1.4 Result Handling:** Processes successful or failed generation responses, formats the output, and prepares the final result.
- **1.5 Logging and Monitoring:** Logs request data to assist in debugging and operational monitoring.
- **1.6 Documentation & Metadata:** Contains sticky notes with detailed documentation, parameter descriptions, and troubleshooting guidance.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Initialization and Authentication

**Overview:**  
This block starts the workflow via manual trigger, sets the API token for Replicate, and configures all parameters needed for the Paigedutcher2 model request.

**Nodes Involved:**  
- Manual Trigger  
- Set API Token  
- Set Other Parameters  

**Node Details:**  

- **Manual Trigger**  
  - Type: `manualTrigger`  
  - Role: Initiates the workflow manually by user action.  
  - Configuration: No parameters needed; simply a trigger node.  
  - Inputs: None  
  - Outputs: Connected to "Set API Token"  
  - Edge Cases: None, but user must manually start.  

- **Set API Token**  
  - Type: `set`  
  - Role: Stores the Replicate API token as a workflow variable.  
  - Configuration: Hardcoded placeholder `"YOUR_REPLICATE_API_TOKEN"` which must be replaced with actual token.  
  - Inputs: From "Manual Trigger"  
  - Outputs: Passes token to "Set Other Parameters"  
  - Edge Cases: Token missing or invalid will cause authentication failure downstream.  

- **Set Other Parameters**  
  - Type: `set`  
  - Role: Defines all other input parameters required by the Paigedutcher2 model.  
  - Configuration:  
    - Copies API token from previous node  
    - Sets parameters with sensible defaults, e.g., mask URL, seed (-1 for random), image URL, model name ("dev"), image dimensions (512x512), prompt ("Create something amazing"), flags (`go_fast` false), various numeric parameters controlling generation (guidance scale, steps, quality, etc.)  
  - Inputs: From "Set API Token"  
  - Outputs: To "Create Other Prediction"  
  - Edge Cases: Parameter mismatches or invalid values may cause API errors. Some parameters are optional but must be consistent (e.g., width and height vs aspect ratio).

---

#### 2.2 API Request & Execution

**Overview:**  
This block sends the generation request to Replicate and logs the request details.

**Nodes Involved:**  
- Create Other Prediction  
- Log Request  

**Node Details:**  

- **Create Other Prediction**  
  - Type: `httpRequest`  
  - Role: Sends POST request to Replicate API `/v1/predictions` endpoint to start content generation.  
  - Configuration:  
    - URL: `https://api.replicate.com/v1/predictions`  
    - Method: POST  
    - Headers: Authorization Bearer token from input, with `Prefer: wait` to wait for immediate processing if possible  
    - Body: JSON payload containing the model version and all input parameters from "Set Other Parameters" using expressions, e.g., `{{ $json.prompt }}`  
    - Response format: JSON, `neverError:true` to handle API errors gracefully  
  - Inputs: From "Set Other Parameters"  
  - Outputs: To "Log Request"  
  - Edge Cases:  
    - Authentication failures if token invalid  
    - Validation errors if parameters are incorrect  
    - Network timeouts or API rate limits may cause request failure or delays  

- **Log Request**  
  - Type: `code` (JavaScript)  
  - Role: Logs the prediction request details for monitoring and debugging.  
  - Configuration: Logs timestamp, prediction id, and model type to console.  
  - Inputs: From "Create Other Prediction"  
  - Outputs: To "Wait 5s"  
  - Edge Cases: Logging failures are unlikely but do not affect workflow operation.

---

#### 2.3 Status Monitoring Loop

**Overview:**  
This block continuously checks the status of the content generation by polling the API with delays, retrying until the generation is complete or failed.

**Nodes Involved:**  
- Wait 5s  
- Check Status  
- Is Complete?  
- Has Failed?  
- Wait 10s  

**Node Details:**  

- **Wait 5s**  
  - Type: `wait`  
  - Role: Pauses workflow for 5 seconds before status check to allow generation progress.  
  - Inputs: From "Log Request"  
  - Outputs: To "Check Status"  
  - Edge Cases: None  

- **Check Status**  
  - Type: `httpRequest`  
  - Role: Sends GET request to Replicate API to retrieve current status of the prediction.  
  - Configuration:  
    - URL: `https://api.replicate.com/v1/predictions/{{ prediction_id }}` (dynamic)  
    - Headers: Authorization Bearer token  
    - Response: JSON, `neverError:true` for graceful error handling  
  - Inputs: From "Wait 5s" and "Wait 10s" (loop)  
  - Outputs: To "Is Complete?"  
  - Edge Cases: Network issues or invalid prediction ID may cause failure or stale results.  

- **Is Complete?**  
  - Type: `if`  
  - Role: Checks if the prediction status is `"succeeded"`.  
  - Inputs: From "Check Status"  
  - Outputs:  
    - True branch: To "Success Response"  
    - False branch: To "Has Failed?"  
  - Edge Cases: Case sensitivity and missing status field may cause logic errors.  

- **Has Failed?**  
  - Type: `if`  
  - Role: Checks if the prediction status is `"failed"`.  
  - Inputs: From "Is Complete?"  
  - Outputs:  
    - True branch: To "Error Response"  
    - False branch: To "Wait 10s" (retry loop)  
  - Edge Cases: Same as above; also potential infinite loop if status is neither succeeded nor failed.  

- **Wait 10s**  
  - Type: `wait`  
  - Role: Pauses workflow for 10 seconds before retrying status check after failure or pending status.  
  - Inputs: From "Has Failed?" (false branch)  
  - Outputs: To "Check Status" (loop)  
  - Edge Cases: None  

---

#### 2.4 Result Handling

**Overview:**  
This block formats the final output based on success or failure and prepares a consistent response object.

**Nodes Involved:**  
- Success Response  
- Error Response  
- Display Result  

**Node Details:**  

- **Success Response**  
  - Type: `set`  
  - Role: Creates an object containing success status, output URLs, prediction ID, status, and message.  
  - Configuration: Uses expressions to extract output URL and prediction metadata from API response.  
  - Inputs: From "Is Complete?" (true branch)  
  - Outputs: To "Display Result"  
  - Edge Cases: Missing output data or malformed response may cause incomplete results.  

- **Error Response**  
  - Type: `set`  
  - Role: Creates an error object with failure status, error message, prediction ID, and message.  
  - Configuration: Uses fallback error messages if API error details are missing.  
  - Inputs: From "Has Failed?" (true branch)  
  - Outputs: To "Display Result"  
  - Edge Cases: May expose API error response if available; otherwise generic error message.  

- **Display Result**  
  - Type: `set`  
  - Role: Assigns the final response object (success or error) to a field `final_result` for downstream use or output.  
  - Inputs: From "Success Response" or "Error Response"  
  - Outputs: None (end node)  
  - Edge Cases: None  

---

#### 2.5 Logging and Monitoring

**Overview:**  
Logs request details for troubleshooting and operational insight.

**Nodes Involved:**  
- Log Request (already detailed in 2.2)

**Node Details:**  
See 2.2 for full details.

---

#### 2.6 Documentation & Metadata

**Overview:**  
Provides user guidance, parameter explanations, and support contact information through sticky notes.

**Nodes Involved:**  
- Sticky Note9  
- Sticky Note4  

**Node Details:**  

- **Sticky Note9**  
  - Type: `stickyNote`  
  - Content: Contact info for support, including email and links to YouTube and LinkedIn channels for tips and tutorials.  
  - Position: Top-left of the workflow for visibility.  

- **Sticky Note4**  
  - Type: `stickyNote`  
  - Content: Extensive documentation covering model overview, parameter guide, workflow explanation, benefits, quick start instructions, troubleshooting guide, and useful links:  
    - Model Documentation: https://replicate.com/paigedutcher2/paigedutcher  
    - Replicate API Docs: https://replicate.com/docs  
    - n8n Documentation: https://docs.n8n.io  
  - This acts as a comprehensive in-workflow manual for users and maintainers.

---

### 3. Summary Table

| Node Name             | Node Type          | Functional Role                        | Input Node(s)              | Output Node(s)             | Sticky Note                                                                                 |
|-----------------------|--------------------|-------------------------------------|----------------------------|----------------------------|---------------------------------------------------------------------------------------------|
| Manual Trigger        | manualTrigger      | Workflow start trigger               | -                          | Set API Token              |                                                                                             |
| Set API Token         | set                | Stores Replicate API token           | Manual Trigger             | Set Other Parameters       |                                                                                             |
| Set Other Parameters  | set                | Defines all generation input params | Set API Token              | Create Other Prediction    |                                                                                             |
| Create Other Prediction| httpRequest        | Sends generation request to Replicate| Set Other Parameters       | Log Request                |                                                                                             |
| Log Request           | code               | Logs prediction request details     | Create Other Prediction    | Wait 5s                    |                                                                                             |
| Wait 5s               | wait               | Waits 5 seconds before status check | Log Request                | Check Status               |                                                                                             |
| Check Status          | httpRequest        | Queries current prediction status   | Wait 5s, Wait 10s          | Is Complete?               |                                                                                             |
| Is Complete?          | if                 | Checks if generation succeeded      | Check Status               | Success Response, Has Failed?|                                                                                           |
| Has Failed?           | if                 | Checks if generation failed         | Is Complete?               | Error Response, Wait 10s    |                                                                                             |
| Wait 10s              | wait               | Waits 10 seconds before retry       | Has Failed?                | Check Status               |                                                                                             |
| Success Response      | set                | Builds success response object      | Is Complete? (true)        | Display Result             |                                                                                             |
| Error Response        | set                | Builds error response object        | Has Failed? (true)         | Display Result             |                                                                                             |
| Display Result        | set                | Final response assignment           | Success Response, Error Response| -                      |                                                                                             |
| Sticky Note9          | stickyNote         | Support contact info and links      | -                          | -                          | For any questions or support, please contact: Yaron@nofluff.online. YouTube and LinkedIn links provided. |
| Sticky Note4          | stickyNote         | Detailed workflow and model docs    | -                          | -                          | Extensive parameter guide, troubleshooting, quick start instructions, and official docs links. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a `Manual Trigger` node:**  
   - Name: `Manual Trigger`  
   - No parameters needed. This is the start node.

3. **Add a `Set` node:**  
   - Name: `Set API Token`  
   - Add one string field named `api_token` with value `"YOUR_REPLICATE_API_TOKEN"` (replace with your actual Replicate API token).  
   - Connect `Manual Trigger` output to this node.

4. **Add another `Set` node:**  
   - Name: `Set Other Parameters`  
   - Create multiple fields, copying the names and default values below:  
     - `api_token` (string) = `={{ $('Set API Token').item.json.api_token }}` (expression)  
     - `mask` (string) = `"https://via.placeholder.com/512x512/000000/FFFFFF.png"`  
     - `seed` (number) = `-1`  
     - `image` (string) = `"https://picsum.photos/512/512"`  
     - `model` (string) = `"dev"`  
     - `width` (number) = `512`  
     - `height` (number) = `512`  
     - `prompt` (string) = `"Create something amazing"`  
     - `go_fast` (boolean) = `false`  
     - `extra_lora` (string) = `""`  
     - `lora_scale` (number) = `1`  
     - `megapixels` (string) = `"1"`  
     - `num_outputs` (number) = `1`  
     - `aspect_ratio` (string) = `"1:1"`  
     - `output_format` (string) = `"webp"`  
     - `guidance_scale` (number) = `3`  
     - `output_quality` (number) = `80`  
     - `prompt_strength` (number) = `0.8`  
     - `extra_lora_scale` (number) = `1`  
     - `replicate_weights` (string) = `""`  
     - `num_inference_steps` (number) = `28`  
     - `disable_safety_checker` (boolean) = `false`  
   - Connect output of `Set API Token` to this node.

5. **Add an `HTTP Request` node:**  
   - Name: `Create Other Prediction`  
   - Method: `POST`  
   - URL: `https://api.replicate.com/v1/predictions`  
   - Headers:  
     - `Authorization`: `=Bearer {{ $json.api_token }}`  
     - `Prefer`: `wait`  
   - Body Content-Type: JSON  
   - Body (raw JSON with expressions):  
     ```json
     {
       "version": "paigedutcher2/paigedutcher:e82368e3a64b65a1df7c06189f9b2305f9425a9dad5283af564b7f22e81c07bf",
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
   - Connect output of `Set Other Parameters` to this node.

6. **Add a `Code` node:**  
   - Name: `Log Request`  
   - Language: JavaScript  
   - Code:  
     ```js
     const data = $input.all()[0].json;
     console.log('paigedutcher2/paigedutcher Request:', {
       timestamp: new Date().toISOString(),
       prediction_id: data.id,
       model_type: 'other'
     });
     return $input.all();
     ```  
   - Connect output of `Create Other Prediction` to this node.

7. **Add a `Wait` node:**  
   - Name: `Wait 5s`  
   - Wait time: 5 seconds  
   - Connect output of `Log Request` to this node.

8. **Add an `HTTP Request` node:**  
   - Name: `Check Status`  
   - Method: `GET`  
   - URL: `https://api.replicate.com/v1/predictions/{{ $('Create Other Prediction').item.json.id }}` (expression referencing prediction ID)  
   - Headers:  
     - `Authorization`: `=Bearer {{ $('Set API Token').item.json.api_token }}`  
   - Connect output of `Wait 5s` to this node.

9. **Add an `If` node:**  
   - Name: `Is Complete?`  
   - Condition: Check if `{{$json["status"]}}` equals `"succeeded"` (case sensitive)  
   - Connect output of `Check Status` to this node.

10. **Add another `If` node:**  
    - Name: `Has Failed?`  
    - Condition: Check if `{{$json["status"]}}` equals `"failed"`  
    - Connect `Is Complete?` false branch to this node.

11. **Add a `Wait` node:**  
    - Name: `Wait 10s`  
    - Wait time: 10 seconds  
    - Connect `Has Failed?` false branch to this node.  
    - Connect output of `Wait 10s` back to `Check Status` node (loop for retry).

12. **Add a `Set` node:**  
    - Name: `Success Response`  
    - Fields:  
      - `response` (object) with value:  
        ```js
        {
          success: true,
          result_url: $json.output,
          prediction_id: $json.id,
          status: $json.status,
          message: "Other generated successfully"
        }
        ```  
    - Connect `Is Complete?` true branch to this node.

13. **Add a `Set` node:**  
    - Name: `Error Response`  
    - Fields:  
      - `response` (object) with value:  
        ```js
        {
          success: false,
          error: $json.error || "Other generation failed",
          prediction_id: $json.id,
          status: $json.status,
          message: "Failed to generate other"
        }
        ```  
    - Connect `Has Failed?` true branch to this node.

14. **Add a `Set` node:**  
    - Name: `Display Result`  
    - Fields:  
      - `final_result` (object) assigned from `$json.response` (passes either success or error response)  
    - Connect outputs of `Success Response` and `Error Response` to this node.

15. **Add Sticky Notes** (optional but recommended for documentation):  
    - Add one with contact info and support links.  
    - Add another with detailed workflow description, parameter guide, troubleshooting, and links to external docs.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                 | Context or Link                                                                                   |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| For any questions or support, please contact: Yaron@nofluff.online. Explore more tips and tutorials on YouTube (https://www.youtube.com/@YaronBeen/videos) and LinkedIn (https://www.linkedin.com/in/yaronbeen/)             | Support contact and video resources                                                             |
| Model Documentation, Parameter Guide, and Replicate API details: https://replicate.com/paigedutcher2/paigedutcher, https://replicate.com/docs                                                                                 | Official model and API documentation                                                            |
| n8n Documentation for node references and workflow building: https://docs.n8n.io                                                                                                                                              | Official n8n documentation                                                                       |
| The workflow includes built-in retry logic with 5s and 10s waits to handle API processing delays and transient errors gracefully.                                                                                           | Workflow reliability and error resilience note                                                  |
| Replace the placeholder `YOUR_REPLICATE_API_TOKEN` with a valid token from https://replicate.com/account to enable authentication.                                                                                          | Credential setup requirement                                                                    |
| The workflow supports extensive customization for parameters, enabling fine control over generation style, quality, and speed.                                                                                              | Customization flexibility                                                                        |
| Monitor API usage and billing on Replicate to avoid quota exhaustion and unexpected errors.                                                                                                                                  | Usage monitoring recommendation                                                                 |

---

This document provides a detailed, structured reference for the "Generate Content with Paigedutcher2 AI Model via Replicate API" workflow, enabling advanced users and AI agents to fully understand, reproduce, and maintain the workflow.