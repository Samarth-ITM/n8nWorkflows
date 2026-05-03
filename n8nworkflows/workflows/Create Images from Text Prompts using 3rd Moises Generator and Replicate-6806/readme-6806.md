Create Images from Text Prompts using 3rd Moises Generator and Replicate

https://n8nworkflows.xyz/workflows/create-images-from-text-prompts-using-3rd-moises-generator-and-replicate-6806


# Create Images from Text Prompts using 3rd Moises Generator and Replicate

### 1. Workflow Overview

This workflow automates the creation of images from text prompts using the "3rdmoises_generator_oldversion" AI model hosted on Replicate. It is designed to handle the entire process from user input to generating and retrieving images via API calls, including status polling, error handling, and response formatting. The workflow is suitable for users wishing to integrate AI-powered image generation into their automation pipelines, with configurable parameters for fine-tuning outputs.

The workflow is logically grouped into the following blocks:

- **1.1 Input Initialization:** Receives manual trigger and sets API authentication and generation parameters.
- **1.2 Prediction Request:** Sends a generation request to the Replicate API using the configured parameters.
- **1.3 Status Polling and Handling:** Waits and polls the prediction status until completion or failure, with retry logic.
- **1.4 Response Processing:** Processes the final result or error, prepares structured output.
- **1.5 Logging:** Logs prediction requests for monitoring and debugging purposes.
- **1.6 Informational Notes:** Contains sticky notes with documentation, instructions, and references.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Initialization

- **Overview:**  
  Starts the workflow manually, sets the API token, and configures all generation parameters including prompt, image size, model options, and other generation controls.

- **Nodes Involved:**  
  - Manual Trigger  
  - Set API Token  
  - Set Other Parameters

- **Node Details:**

  - **Manual Trigger**  
    - Type: Trigger  
    - Role: Initiates the workflow manually.  
    - Config: No parameters; simply a manual start point.  
    - Connections: Output to "Set API Token".  
    - Edge cases: None significant; user must trigger manually.

  - **Set API Token**  
    - Type: Set  
    - Role: Sets the Replicate API token for authentication in downstream API calls.  
    - Config: Assigns a string variable `api_token` with placeholder "YOUR_REPLICATE_API_TOKEN" which must be replaced with a valid token.  
    - Connections: Outputs to "Set Other Parameters".  
    - Edge cases: Failure if token is invalid or missing.

  - **Set Other Parameters**  
    - Type: Set  
    - Role: Defines all input parameters for the image generation model, including prompt, image settings, model options, and advanced generation controls.  
    - Config:  
      - Copies `api_token` from previous node.  
      - Sets default values for parameters such as mask URL, seed (-1 for random), base image URL, model version, dimensions (512x512), prompt text ("Create something amazing"), speed mode (`go_fast` false), LORA parameters, megapixels, output format, guidance scale, output quality, prompt strength, number of outputs, aspect ratio, inference steps, and safety checker toggle.  
    - Connections: Outputs to "Create Other Prediction".  
    - Edge cases: Parameter mismatches or invalid values can cause API rejection or generation errors.

---

#### 2.2 Prediction Request

- **Overview:**  
  Sends a POST request to the Replicate API `/v1/predictions` endpoint to create a new image generation prediction using the input parameters.

- **Nodes Involved:**  
  - Create Other Prediction

- **Node Details:**

  - **Create Other Prediction**  
    - Type: HTTP Request  
    - Role: Initiates the image generation prediction on Replicate by sending all configured parameters and authentication headers.  
    - Config:  
      - POST method to `https://api.replicate.com/v1/predictions`  
      - JSON body includes model version and input parameters dynamically referenced from previous node JSON data.  
      - Headers include `Authorization: Bearer {api_token}` and `Prefer: wait` to wait for the prediction to complete server-side if possible.  
      - Response format JSON.  
    - Connections: Outputs to "Log Request".  
    - Edge cases: HTTP errors, invalid tokens, malformed parameters, API downtime, or rate limits may cause failures.

---

#### 2.3 Status Polling and Handling

- **Overview:**  
  Implements a loop that waits, then queries the prediction status repeatedly until the generation succeeds or fails. Includes delays and conditional branching for error retries.

- **Nodes Involved:**  
  - Log Request  
  - Wait 5s  
  - Check Status  
  - Is Complete?  
  - Has Failed?  
  - Wait 10s

- **Node Details:**

  - **Log Request**  
    - Type: Code  
    - Role: Logs prediction details for monitoring including timestamp, prediction ID, and model type.  
    - Config: JavaScript code accessing input JSON to log relevant information.  
    - Connections: Outputs to "Wait 5s".  
    - Edge cases: Logging failures do not affect main process.

  - **Wait 5s**  
    - Type: Wait  
    - Role: Pauses workflow execution for 5 seconds before checking prediction status.  
    - Config: Fixed 5-second delay.  
    - Connections: Outputs to "Check Status".  
    - Edge cases: None significant.

  - **Check Status**  
    - Type: HTTP Request  
    - Role: Requests the current status of the prediction from Replicate using prediction ID.  
    - Config:  
      - GET request to `https://api.replicate.com/v1/predictions/{prediction_id}`  
      - Authorization header using stored API token.  
      - Response JSON format.  
    - Connections: Outputs to "Is Complete?".  
    - Edge cases: Network errors, API downtime, invalid prediction ID.

  - **Is Complete?**  
    - Type: If  
    - Role: Checks if prediction status equals "succeeded" to determine if the generation completed successfully.  
    - Config: Condition `status == "succeeded"`.  
    - Connections:  
      - True branch to "Success Response".  
      - False branch to "Has Failed?".  
    - Edge cases: Status field missing or unexpected values.

  - **Has Failed?**  
    - Type: If  
    - Role: Checks if prediction status equals "failed" to determine if generation failed.  
    - Config: Condition `status == "failed"`.  
    - Connections:  
      - True branch to "Error Response".  
      - False branch to "Wait 10s" (retry).  
    - Edge cases: Unhandled statuses may cause unexpected behavior.

  - **Wait 10s**  
    - Type: Wait  
    - Role: Waits 10 seconds before retrying status check to avoid excessive polling on failure or pending states.  
    - Config: Fixed 10-second delay.  
    - Connections: Outputs back to "Check Status" (creates polling loop).  
    - Edge cases: Long delays may increase total execution time.

---

#### 2.4 Response Processing

- **Overview:**  
  Prepares structured JSON responses for success or failure, and outputs final results for downstream consumption or display.

- **Nodes Involved:**  
  - Success Response  
  - Error Response  
  - Display Result

- **Node Details:**

  - **Success Response**  
    - Type: Set  
    - Role: Constructs a JSON object on successful prediction containing success flag, result URL(s), prediction ID, status, and message.  
    - Config: Uses data from prediction JSON fields like `output`, `id`, `status`.  
    - Connections: Outputs to "Display Result".  
    - Edge cases: Missing output URLs or incomplete data.

  - **Error Response**  
    - Type: Set  
    - Role: Constructs a JSON error object containing failure flag, error message, prediction ID, status, and message.  
    - Config: Uses error field if present or defaults to generic failure message.  
    - Connections: Outputs to "Display Result".  
    - Edge cases: Some API errors may lack detailed messages.

  - **Display Result**  
    - Type: Set  
    - Role: Sets a final output object called `final_result` containing either success or error response for use by other workflows or UI.  
    - Config: Copies the `response` object from previous node.  
    - Connections: Terminal node in the workflow.  
    - Edge cases: None significant.

---

#### 2.5 Logging

- **Overview:**  
  Logs each generation request with timestamp and prediction ID for monitoring and debugging.

- **Nodes Involved:**  
  - Log Request (already described in 2.3)

---

#### 2.6 Informational Notes (Sticky Notes)

- **Overview:**  
  Provides rich documentation and user guidance within the workflow canvas, including contact info, usage instructions, parameter explanations, and troubleshooting tips.

- **Nodes Involved:**  
  - Sticky Note9  
  - Sticky Note4

- **Node Details:**

  - **Sticky Note9**  
    - Content: Title block with contact email (Yaron@nofluff.online) and links to YouTube and LinkedIn channels for support and tutorials.

  - **Sticky Note4**  
    - Content:  
      - Detailed model overview including owner, model type, API endpoint.  
      - Parameter reference with required and optional parameters.  
      - Description of workflow components and their roles.  
      - Quick start instructions for API token setup and execution.  
      - Troubleshooting guide and best practices.  
      - Useful links to model docs, API docs, and n8n docs.

---

### 3. Summary Table

| Node Name            | Node Type          | Functional Role                                | Input Node(s)                 | Output Node(s)              | Sticky Note                                                                                     |
|----------------------|--------------------|-----------------------------------------------|------------------------------|-----------------------------|------------------------------------------------------------------------------------------------|
| Manual Trigger       | Trigger            | Starts the workflow manually                   |                              | Set API Token                |                                                                                                |
| Set API Token        | Set                | Sets API token for authentication              | Manual Trigger               | Set Other Parameters         |                                                                                                |
| Set Other Parameters | Set                | Sets generation parameters for the model       | Set API Token                | Create Other Prediction      |                                                                                                |
| Create Other Prediction | HTTP Request     | Sends prediction creation request to Replicate| Set Other Parameters         | Log Request                 |                                                                                                |
| Log Request          | Code               | Logs prediction request details                 | Create Other Prediction      | Wait 5s                     |                                                                                                |
| Wait 5s              | Wait               | Waits 5 seconds before status check             | Log Request                  | Check Status                |                                                                                                |
| Check Status         | HTTP Request       | Checks the prediction status                     | Wait 5s, Wait 10s            | Is Complete?                |                                                                                                |
| Is Complete?         | If                 | Checks if prediction succeeded                   | Check Status                 | Success Response, Has Failed? |                                                                                                |
| Has Failed?          | If                 | Checks if prediction failed                      | Is Complete?                 | Error Response, Wait 10s    |                                                                                                |
| Wait 10s             | Wait               | Waits 10 seconds before retrying status check   | Has Failed?                  | Check Status                |                                                                                                |
| Success Response     | Set                | Formats success response object                   | Is Complete?                 | Display Result              |                                                                                                |
| Error Response       | Set                | Formats error response object                     | Has Failed?                  | Display Result              |                                                                                                |
| Display Result       | Set                | Final output of success or error response        | Success Response, Error Response |                            |                                                                                                |
| Sticky Note9         | Sticky Note        | Contact info and support links                    |                              |                             | ======================================= 3RDMOISES_GENERATOR_OLDVERSION GENERATOR Contact: Yaron@nofluff.online YouTube: https://www.youtube.com/@YaronBeen/videos LinkedIn: https://www.linkedin.com/in/yaronbeen/ |
| Sticky Note4         | Sticky Note        | Comprehensive documentation and instructions     |                              |                             | See detailed model overview, parameter guide, workflow explanation, quick start, troubleshooting, and additional resources in the sticky note content. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add node of type "Manual Trigger". No parameters needed. This starts the workflow manually.

2. **Add Set Node "Set API Token"**  
   - Type: Set  
   - Add string field `api_token` with value: `YOUR_REPLICATE_API_TOKEN` (replace with your actual Replicate API token).  
   - Connect Manual Trigger output to this node input.

3. **Add Set Node "Set Other Parameters"**  
   - Type: Set  
   - Assign variables as follows (copy exactly):  
     - `api_token`: Reference from previous node `{{$node["Set API Token"].json["api_token"]}}`  
     - `mask`: `"https://via.placeholder.com/512x512/000000/FFFFFF.png"`  
     - `seed`: `-1` (number)  
     - `image`: `"https://picsum.photos/512/512"`  
     - `model`: `"dev"`  
     - `width`: `512`  
     - `height`: `512`  
     - `prompt`: `"Create something amazing"`  
     - `go_fast`: `false` (boolean)  
     - `extra_lora`: `""` (empty string)  
     - `lora_scale`: `1`  
     - `megapixels`: `"1"`  
     - `num_outputs`: `1`  
     - `aspect_ratio`: `"1:1"`  
     - `output_format`: `"webp"`  
     - `guidance_scale`: `3`  
     - `output_quality`: `80`  
     - `prompt_strength`: `0.8`  
     - `extra_lora_scale`: `1`  
     - `replicate_weights`: `""` (empty string)  
     - `num_inference_steps`: `28`  
     - `disable_safety_checker`: `false` (boolean)  
   - Connect output from "Set API Token" node to this node.

4. **Add HTTP Request Node "Create Other Prediction"**  
   - Method: POST  
   - URL: `https://api.replicate.com/v1/predictions`  
   - Headers:  
     - `Authorization: Bearer {{$json["api_token"]}}`  
     - `Prefer: wait`  
   - Body (JSON):  
     ```json
     {
       "version": "moicarmonas/3rdmoises_generator_oldversion:214591a08f5e60ddfc97576ebe4562491b8d559920b0e17ae04caa1fc0291270",
       "input": {
         "mask": "{{$json.mask}}",
         "seed": {{$json.seed}},
         "image": "{{$json.image}}",
         "model": "{{$json.model}}",
         "width": {{$json.width}},
         "height": {{$json.height}},
         "prompt": "{{$json.prompt}}",
         "go_fast": {{$json.go_fast}},
         "extra_lora": "{{$json.extra_lora}}",
         "lora_scale": {{$json.lora_scale}},
         "megapixels": "{{$json.megapixels}}",
         "num_outputs": {{$json.num_outputs}},
         "aspect_ratio": "{{$json.aspect_ratio}}",
         "output_format": "{{$json.output_format}}",
         "guidance_scale": {{$json.guidance_scale}},
         "output_quality": {{$json.output_quality}},
         "prompt_strength": {{$json.prompt_strength}},
         "extra_lora_scale": {{$json.extra_lora_scale}},
         "replicate_weights": "{{$json.replicate_weights}}",
         "num_inference_steps": {{$json.num_inference_steps}},
         "disable_safety_checker": {{$json.disable_safety_checker}}
       }
     }
     ```
   - Response Format: JSON  
   - Connect output from "Set Other Parameters".

5. **Add Code Node "Log Request"**  
   - JavaScript code:  
     ```javascript
     const data = $input.all()[0].json;
     console.log('moicarmonas/3rdmoises_generator_oldversion Request:', {
       timestamp: new Date().toISOString(),
       prediction_id: data.id,
       model_type: 'other'
     });
     return $input.all();
     ```  
   - Connect output of "Create Other Prediction" node.

6. **Add Wait Node "Wait 5s"**  
   - Set to wait 5 seconds.  
   - Connect output of "Log Request".

7. **Add HTTP Request Node "Check Status"**  
   - Method: GET  
   - URL: `https://api.replicate.com/v1/predictions/{{$node["Create Other Prediction"].json["id"]}}`  
   - Headers:  
     - `Authorization: Bearer {{$node["Set API Token"].json["api_token"]}}`  
   - Response Format: JSON  
   - Connect output of "Wait 5s".

8. **Add If Node "Is Complete?"**  
   - Condition: `{{$json.status}}` equals `"succeeded"`  
   - Connect output of "Check Status".

9. **Add Set Node "Success Response"**  
   - Assign field `response` as an object:  
     ```json
     {
       "success": true,
       "result_url": {{$json.output}},
       "prediction_id": {{$json.id}},
       "status": {{$json.status}},
       "message": "Other generated successfully"
     }
     ```  
   - Connect true output of "Is Complete?".

10. **Add Set Node "Display Result"**  
    - Assign field `final_result` to: `{{$json.response}}` (copy from previous).  
    - Connect output of "Success Response".

11. **Add If Node "Has Failed?"**  
    - Condition: `{{$json.status}}` equals `"failed"`  
    - Connect false output of "Is Complete?".

12. **Add Set Node "Error Response"**  
    - Assign field `response` as:  
      ```json
      {
        "success": false,
        "error": {{$json.error || "Other generation failed"}},
        "prediction_id": {{$json.id}},
        "status": {{$json.status}},
        "message": "Failed to generate other"
      }
      ```  
    - Connect true output of "Has Failed?".

13. **Connect Error Response output to "Display Result"** (same as success).

14. **Add Wait Node "Wait 10s"**  
    - Wait for 10 seconds.  
    - Connect false output of "Has Failed?".

15. **Connect "Wait 10s" output back to "Check Status"** (loop to keep polling).

16. **Add Sticky Notes for documentation** (optional but recommended) with provided content for contact, instructions, and parameter references.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | Context or Link                                                                                          |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| ======================================= 3RDMOISES_GENERATOR_OLDVERSION GENERATOR For any questions or support, please contact: Yaron@nofluff.online Explore more tips and tutorials here: - YouTube: https://www.youtube.com/@YaronBeen/videos - LinkedIn: https://www.linkedin.com/in/yaronbeen/ =======================================                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | Contact and support details provided inside Sticky Note9.                                               |
| Comprehensive documentation includes model overview, parameter reference (required and optional), detailed parameter guide, workflow components explanation, quick start instructions for API token and execution, troubleshooting guide, best practices, and links to model docs, Replicate API docs, and n8n docs.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | Provided inside Sticky Note4 for user guidance and workflow understanding.                              |
| Model endpoint: https://api.replicate.com/v1/predictions Documentation: https://replicate.com/moicarmonas/3rdmoises_generator_oldversion Replicate API Docs: https://replicate.com/docs n8n Docs: https://docs.n8n.io                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Official documentation and API references links.                                                        |
| Troubleshooting tips: Ensure API token validity and permissions, verify parameter types and values, monitor API usage and billing, use default parameters for initial testing, and handle longer generation times with patience.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | Best practices and common issues for stable operation.                                                  |
| Disclaimer: The provided text is generated from an automated n8n workflow. It complies with content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | General disclaimer for content safety and compliance.                                                  |

---

This completes the comprehensive documentation and reference for the "Create Images from Text Prompts using 3rd Moises Generator and Replicate" n8n workflow. This document enables understanding, reproduction, modification, and troubleshooting by both advanced users and automation agents.