Create Images from Text Prompts using 2nd Moises Generator and Replicate

https://n8nworkflows.xyz/workflows/create-images-from-text-prompts-using-2nd-moises-generator-and-replicate-6807


# Create Images from Text Prompts using 2nd Moises Generator and Replicate

### 1. Workflow Overview

This workflow automates the process of generating images from text prompts using the "2nd Moises Generator" model hosted on Replicate's API. It is designed for users who want to create AI-generated images by providing customizable prompts and parameters, with the workflow handling API token management, request submission, status polling, error handling, and result delivery.

The logical structure is divided into the following blocks:

- **1.1 Input Reception**: Starts the workflow manually.
- **1.2 API Token and Parameter Setup**: Sets the Replicate API token and prepares all generation parameters.
- **1.3 Image Generation Request**: Sends the generation request to Replicate and logs the request.
- **1.4 Status Polling Loop**: Waits and checks the prediction status until completion or failure.
- **1.5 Result Handling**: Processes success or failure responses and prepares the final output.
- **1.6 Logging and Monitoring**: Logs relevant request details for traceability.
- **1.7 Documentation and Support Notes**: Provides detailed explanatory notes about the model, parameters, usage, and troubleshooting.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview**: Initiates the workflow manually by the user.
- **Nodes Involved**: Manual Trigger
- **Node Details**:
  - **Manual Trigger**
    - Type: `manualTrigger` — Starts workflow execution on user command.
    - Configuration: Default manual trigger without inputs.
    - Input: None (start node).
    - Output: Triggers the next node "Set API Token".
    - Edge Cases: None inherent; user must manually trigger to start.

#### 2.2 API Token and Parameter Setup

- **Overview**: Sets Replicate API token for authentication and configures all parameters required for image generation, including prompt, image size, model options, and output preferences.
- **Nodes Involved**: Set API Token, Set Other Parameters
- **Node Details**:
  - **Set API Token**
    - Type: `set` — Stores the API token as a workflow variable.
    - Configuration: Set string variable `api_token` with placeholder "YOUR_REPLICATE_API_TOKEN" to be replaced by user.
    - Input: Triggered by Manual Trigger.
    - Output: Passes token to "Set Other Parameters".
    - Edge Cases: Missing or invalid token will cause authentication failures downstream.
  - **Set Other Parameters**
    - Type: `set` — Configures a comprehensive set of generation parameters.
    - Configuration: Copies the API token from previous node; defines parameters such as:
      - `mask`: URL to mask image for inpainting.
      - `seed`: Integer seed for reproducibility (-1 = random).
      - `image`: Input image URL.
      - `model`: Model variant ("dev" default).
      - `width` and `height`: Image dimensions (512x512 default).
      - `prompt`: Text prompt for image creation ("Create something amazing" default).
      - Other parameters: `go_fast`, `extra_lora`, `lora_scale`, `megapixels`, `num_outputs`, `aspect_ratio`, `output_format`, `guidance_scale`, `output_quality`, `prompt_strength`, `extra_lora_scale`, `replicate_weights`, `num_inference_steps`, `disable_safety_checker`.
    - Input: Receives API token from "Set API Token".
    - Output: Sends full parameter set to "Create Other Prediction".
    - Edge Cases: Parameter type mismatches or invalid URLs may cause API errors or generation failures.

#### 2.3 Image Generation Request

- **Overview**: Sends a POST request to Replicate API to start image generation with the configured parameters.
- **Nodes Involved**: Create Other Prediction, Log Request
- **Node Details**:
  - **Create Other Prediction**
    - Type: `httpRequest` — Submits generation job to Replicate API.
    - Configuration:
      - URL: `https://api.replicate.com/v1/predictions`
      - Method: POST
      - Headers: Authorization Bearer token from API token variable; Prefer: wait (to wait for immediate response).
      - Body: JSON including model version, all input parameters interpolated from previous node.
      - Response handling: JSON, no error throwing on HTTP errors (neverError=true).
    - Input: Receives parameters from "Set Other Parameters".
    - Output: Returns prediction ID and initial status.
    - Edge Cases: Network issues, invalid token, malformed request body, API rate limiting.
  - **Log Request**
    - Type: `code` — Logs generation request details for tracking.
    - Configuration: JavaScript code logs timestamp, prediction ID, and model type.
    - Input: Receives output from "Create Other Prediction".
    - Output: Passes data to "Wait 5s".
    - Edge Cases: Logging failure does not impact flow but may lose traceability.

#### 2.4 Status Polling Loop

- **Overview**: Implements a polling mechanism to periodically check the status of the generation until it completes successfully or fails.
- **Nodes Involved**: Wait 5s, Check Status, Is Complete?, Has Failed?, Wait 10s
- **Node Details**:
  - **Wait 5s**
    - Type: `wait` — Pauses for 5 seconds before status check.
    - Input: From "Log Request".
    - Output: Proceeds to "Check Status".
  - **Check Status**
    - Type: `httpRequest` — Retrieves current prediction status.
    - Configuration:
      - URL: Uses prediction ID from "Create Other Prediction" to query status.
      - Headers: Authorization with API token.
      - Response: JSON, no error thrown on failure.
    - Input: From "Wait 5s" or "Wait 10s".
    - Output: Passes status JSON to "Is Complete?".
    - Edge Cases: Network or API errors; prediction ID missing or expired.
  - **Is Complete?**
    - Type: `if` — Checks if `status` equals "succeeded".
    - Input: From "Check Status".
    - Output:
      - If true: to "Success Response".
      - If false: to "Has Failed?".
  - **Has Failed?**
    - Type: `if` — Checks if `status` equals "failed".
    - Input: From "Is Complete?" false branch.
    - Output:
      - If true: to "Error Response".
      - If false: to "Wait 10s" (retry).
  - **Wait 10s**
    - Type: `wait` — Delays 10 seconds before retrying status check.
    - Input: From "Has Failed?" false branch.
    - Output: Loops back to "Check Status".
    - Edge Cases: Long-running jobs may delay workflow; possible infinite loops if status never changes.

#### 2.5 Result Handling

- **Overview**: Constructs final success or error responses and formats output for downstream use.
- **Nodes Involved**: Success Response, Error Response, Display Result
- **Node Details**:
  - **Success Response**
    - Type: `set` — Sets a JSON object indicating success, with generated image URL, prediction ID, status, and message.
    - Input: From "Is Complete?" true branch.
    - Output: Passes to "Display Result".
  - **Error Response**
    - Type: `set` — Sets a JSON object indicating failure, with error message, prediction ID, status, and message.
    - Input: From "Has Failed?" true branch.
    - Output: Passes to "Display Result".
  - **Display Result**
    - Type: `set` — Sets the final workflow output variable `final_result` to the response object from success or error.
    - Input: From either "Success Response" or "Error Response".
    - Output: Workflow end node, outputs final data.
    - Edge Cases: None; ensures consistent output format.

#### 2.6 Logging and Monitoring

- **Overview**: Logs request details for monitoring and debugging purposes.
- **Nodes Involved**: Log Request (already covered)
- **Node Details**: See section 2.3.

#### 2.7 Documentation and Support Notes

- **Overview**: Provides detailed in-workflow notes on usage, parameters, troubleshooting, and contact information.
- **Nodes Involved**: Sticky Note9, Sticky Note4
- **Node Details**:
  - **Sticky Note9**
    - Content: Contact info, YouTube and LinkedIn links for support.
  - **Sticky Note4**
    - Content: Full workflow description, model details, parameter reference, instructions for usage, troubleshooting tips, and external links.
  - These notes serve as in-editor documentation and do not affect workflow logic.

---

### 3. Summary Table

| Node Name           | Node Type           | Functional Role              | Input Node(s)               | Output Node(s)              | Sticky Note                                                                                           |
|---------------------|---------------------|-----------------------------|-----------------------------|-----------------------------|-----------------------------------------------------------------------------------------------------|
| Manual Trigger      | manualTrigger       | Start workflow manually      | None                        | Set API Token               |                                                                                                     |
| Set API Token       | set                 | Store Replicate API token    | Manual Trigger              | Set Other Parameters        |                                                                                                     |
| Set Other Parameters| set                 | Configure generation inputs  | Set API Token               | Create Other Prediction     |                                                                                                     |
| Create Other Prediction | httpRequest      | Submit generation request    | Set Other Parameters        | Log Request                 |                                                                                                     |
| Log Request         | code                | Log request info             | Create Other Prediction     | Wait 5s                     |                                                                                                     |
| Wait 5s             | wait                | Pause before status check    | Log Request                 | Check Status                |                                                                                                     |
| Check Status        | httpRequest         | Check generation status      | Wait 5s, Wait 10s           | Is Complete?                |                                                                                                     |
| Is Complete?        | if                  | Check if generation succeeded| Check Status                | Success Response, Has Failed? |                                                                                                     |
| Has Failed?         | if                  | Check if generation failed   | Is Complete?                | Error Response, Wait 10s    |                                                                                                     |
| Wait 10s            | wait                | Wait before retrying status  | Has Failed?                 | Check Status                |                                                                                                     |
| Success Response    | set                 | Format success output        | Is Complete?                | Display Result              |                                                                                                     |
| Error Response      | set                 | Format error output          | Has Failed?                 | Display Result              |                                                                                                     |
| Display Result      | set                 | Set final workflow output    | Success Response, Error Response | None                  |                                                                                                     |
| Sticky Note9        | stickyNote          | Support contact & links      | None                        | None                       | "2NDMOISES_GENERATOR GENERATOR\nContact: Yaron@nofluff.online\nYouTube & LinkedIn links"             |
| Sticky Note4        | stickyNote          | Full workflow documentation  | None                        | None                       | Extensive documentation, parameter guide, troubleshooting, and external links                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**
   - Type: `Manual Trigger`
   - Purpose: Start the workflow manually.

2. **Create Set Node "Set API Token"**
   - Type: `Set`
   - Purpose: Store your Replicate API token.
   - Configuration:
     - Add string field `api_token`
     - Value: Replace `"YOUR_REPLICATE_API_TOKEN"` with your actual Replicate API token.
   - Connect output of Manual Trigger to this node.

3. **Create Set Node "Set Other Parameters"**
   - Type: `Set`
   - Purpose: Define all generation parameters.
   - Configuration:
     - Copy `api_token` from previous node: `={{ $('Set API Token').item.json.api_token }}`
     - Add parameters (with default values):
       - `mask` (string): `https://via.placeholder.com/512x512/000000/FFFFFF.png`
       - `seed` (number): `-1`
       - `image` (string): `https://picsum.photos/512/512`
       - `model` (string): `dev`
       - `width` (number): `512`
       - `height` (number): `512`
       - `prompt` (string): `Create something amazing`
       - `go_fast` (boolean): `false`
       - `extra_lora` (string): `""`
       - `lora_scale` (number): `1`
       - `megapixels` (string): `"1"`
       - `num_outputs` (number): `1`
       - `aspect_ratio` (string): `"1:1"`
       - `output_format` (string): `"webp"`
       - `guidance_scale` (number): `3`
       - `output_quality` (number): `80`
       - `prompt_strength` (number): `0.8`
       - `extra_lora_scale` (number): `1`
       - `replicate_weights` (string): `""`
       - `num_inference_steps` (number): `28`
       - `disable_safety_checker` (boolean): `false`
   - Connect output of "Set API Token" to this node.

4. **Create HTTP Request Node "Create Other Prediction"**
   - Type: `HTTP Request`
   - Purpose: Submit prediction request to Replicate API.
   - Configuration:
     - URL: `https://api.replicate.com/v1/predictions`
     - Method: POST
     - Headers:
       - `Authorization`: `Bearer {{ $json.api_token }}`
       - `Prefer`: `wait`
     - Body Content Type: JSON
     - Body:
       ```json
       {
         "version": "moicarmonas/2ndmoises_generator:7ae928ae3e6f1614129b139f34503ff4e91e922e7487658a47ad600bca16d5b0",
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
     - Response Format: JSON
     - Enable "Never Error" to avoid workflow stop on HTTP errors.
   - Connect output of "Set Other Parameters" to this node.

5. **Create Code Node "Log Request"**
   - Type: `Code`
   - Purpose: Log the request details.
   - Configuration (JavaScript):
     ```javascript
     const data = $input.all()[0].json;
     console.log('moicarmonas/2ndmoises_generator Request:', {
       timestamp: new Date().toISOString(),
       prediction_id: data.id,
       model_type: 'other'
     });
     return $input.all();
     ```
   - Connect output of "Create Other Prediction" to this node.

6. **Create Wait Node "Wait 5s"**
   - Type: `Wait`
   - Purpose: Pause 5 seconds before checking status.
   - Configuration: 5 seconds
   - Connect output of "Log Request" to this node.

7. **Create HTTP Request Node "Check Status"**
   - Type: `HTTP Request`
   - Purpose: Poll prediction status.
   - Configuration:
     - URL: `https://api.replicate.com/v1/predictions/{{ $('Create Other Prediction').item.json.id }}`
     - Method: GET
     - Headers:
       - `Authorization`: `Bearer {{ $('Set API Token').item.json.api_token }}`
     - Response Format: JSON
     - Enable "Never Error".
   - Connect output of "Wait 5s" and "Wait 10s" to this node.

8. **Create If Node "Is Complete?"**
   - Type: `If`
   - Purpose: Check if status is "succeeded".
   - Condition:
     - If `$json.status === 'succeeded'`
   - Connect output of "Check Status" to this node.

9. **Create If Node "Has Failed?"**
   - Type: `If`
   - Purpose: Check if status is "failed".
   - Condition:
     - If `$json.status === 'failed'`
   - Connect false output of "Is Complete?" to this node.

10. **Create Set Node "Success Response"**
    - Type: `Set`
    - Purpose: Format success response JSON.
    - Configuration:
      - Set variable `response` to object:
        ```json
        {
          "success": true,
          "result_url": $json.output,
          "prediction_id": $json.id,
          "status": $json.status,
          "message": "Other generated successfully"
        }
        ```
    - Connect true output of "Is Complete?" to this node.

11. **Create Set Node "Error Response"**
    - Type: `Set`
    - Purpose: Format error response JSON.
    - Configuration:
      - Set variable `response` to object:
        ```json
        {
          "success": false,
          "error": $json.error || "Other generation failed",
          "prediction_id": $json.id,
          "status": $json.status,
          "message": "Failed to generate other"
        }
        ```
    - Connect true output of "Has Failed?" to this node.

12. **Create Wait Node "Wait 10s"**
    - Type: `Wait`
    - Purpose: Delay 10 seconds before retrying status check.
    - Configuration: 10 seconds
    - Connect false output of "Has Failed?" to this node.

13. **Connect "Wait 10s" output back to "Check Status"** to complete polling loop.

14. **Create Set Node "Display Result"**
    - Type: `Set`
    - Purpose: Set the final output variable `final_result` with the response.
    - Configuration:
      - Set variable `final_result` to `={{ $json.response }}`
    - Connect output of both "Success Response" and "Error Response" to this node.

15. **Final Output**
    - The workflow ends after "Display Result", returning `final_result` as the output.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | Context or Link                                                                                                         |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------|
| ======================================= 2NDMOISES_GENERATOR GENERATOR ======================================= For any questions or support, please contact: Yaron@nofluff.online Explore more tips and tutorials here: - YouTube: https://www.youtube.com/@YaronBeen/videos - LinkedIn: https://www.linkedin.com/in/yaronbeen/ =======================================                                                                                                                                                                                                                                                                                                                                                                                         | Support Contact and Video/LinkedIn Resource                                                                              |
| **Model Overview:** Owner: moicarmonas Model: 2ndmoises_generator Type: Other Generation API Endpoint: https://api.replicate.com/v1/predictions **Parameter Reference:** Required: prompt Optional: mask, seed, image, model, width, height, go_fast, extra_lora, lora_scale, megapixels, num_outputs, aspect_ratio, output_format, guidance_scale, output_quality, prompt_strength, extra_lora_scale, replicate_weights, num_inference_steps, disable_safety_checker **Quick Start:** 1) Get API token at https://replicate.com 2) Replace token in workflow 3) Adjust parameters 4) Run workflow 5) Retrieve generated image URL **Troubleshooting:** Invalid token, parameter validation, generation timeout, output format checks Recommended to test default params first, monitor usage, secure tokens. | Full workflow documentation and usage instructions embedded as sticky notes in the workflow. Links to model and docs: https://replicate.com/moicarmonas/2ndmoises_generator |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.