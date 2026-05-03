Generate Custom Images from Text Prompts with Stability AI SDXL via Replicate

https://n8nworkflows.xyz/workflows/generate-custom-images-from-text-prompts-with-stability-ai-sdxl-via-replicate-6785


# Generate Custom Images from Text Prompts with Stability AI SDXL via Replicate

### 1. Workflow Overview

This workflow automates the generation of custom images from text prompts using the Stability AI SDXL model via the Replicate API. It is designed for users who want to transform descriptive text prompts into high-quality images seamlessly. The workflow handles API authentication, parameter configuration, prediction submission, status monitoring with retry logic, and structured success or error responses.

Logical blocks:

- **1.1 Input Initialization:** Starting the workflow via manual trigger and setting the API token.
- **1.2 Parameter Configuration:** Defining all image generation parameters including prompt, image size, and model options.
- **1.3 Image Generation Request:** Sending the image generation request to Replicate API and receiving prediction ID.
- **1.4 Status Monitoring Loop:** Periodically checking the status of the image generation prediction, with wait nodes and conditional branching.
- **1.5 Response Handling:** Differentiating between success and failure, creating JSON responses accordingly.
- **1.6 Logging and Output:** Logging generation details and outputting the final result for downstream use or display.
- **1.7 Documentation and Notes:** Embedded sticky notes providing detailed instructions, parameter descriptions, and support contacts.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Initialization

- **Overview:** Starts workflow execution and sets the Replicate API token required for authentication.
- **Nodes Involved:** Manual Trigger, Set API Token
- **Node Details:**

  - **Manual Trigger**
    - Type: Trigger node to start workflow manually.
    - Configuration: No parameters; simply user-initiated.
    - Inputs: None.
    - Outputs: To Set API Token node.
    - Edge Cases: None; manual start only.
  
  - **Set API Token**
    - Type: Set node to assign the Replicate API token.
    - Configuration: Stores token string under variable `api_token`.
    - Inputs: Manual Trigger output.
    - Outputs: To Set Image Parameters node.
    - Edge Cases: Requires user to replace placeholder `YOUR_REPLICATE_API_TOKEN` with a valid token; failure to set valid token will cause API auth errors downstream.

#### 1.2 Parameter Configuration

- **Overview:** Prepares all required and optional parameters for the image generation request, including prompt, image size, seed, and various model-specific options.
- **Nodes Involved:** Set Image Parameters
- **Node Details:**

  - **Set Image Parameters**
    - Type: Set node defining parameters for the Replicate SDXL model.
    - Configuration:
      - Copies `api_token` from previous node.
      - Sets default values for:
        - mask: placeholder image URL (black/white PNG).
        - seed: -1 (randomized).
        - image: sample image URL for img2img mode.
        - width & height: 1024 x 1024 pixels.
        - prompt: "An astronaut riding a rainbow unicorn".
        - refine: "no_refiner".
        - scheduler: "K_EULER".
        - lora_scale, num_outputs, refine_steps, guidance_scale, apply_watermark, etc.
      - Includes flags like disable_safety_checker (default false).
    - Inputs: From Set API Token node.
    - Outputs: To Create Image Prediction node.
    - Edge Cases:
      - Incorrect parameter types or values may cause API errors.
      - Seed value -1 triggers random seed generation.
      - URL parameters (mask, image) must be valid URLs.
      - Missing or invalid prompt will cause generation to fail.

#### 1.3 Image Generation Request

- **Overview:** Sends a POST request to the Replicate API to start image generation and obtains a prediction ID to track progress.
- **Nodes Involved:** Create Image Prediction
- **Node Details:**

  - **Create Image Prediction**
    - Type: HTTP Request node.
    - Configuration:
      - POST to `https://api.replicate.com/v1/predictions`.
      - JSON body includes all parameters from previous node.
      - Headers include `Authorization: Bearer <api_token>` and `Prefer: wait` to wait for prediction completion if possible.
      - Response format: JSON, with `neverError` set to true to handle errors gracefully.
    - Inputs: From Set Image Parameters node.
    - Outputs: To Log Request node.
    - Edge Cases:
      - API token invalid or expired causes 401 errors.
      - Network issues or API rate limits may cause request failures.
      - Model version ID must be accurate; otherwise, API rejects request.
      - Large images or complex prompts may increase processing time.

#### 1.4 Status Monitoring Loop

- **Overview:** Implements a polling mechanism to check prediction status until completion or failure, with wait periods in between.
- **Nodes Involved:** Log Request, Wait 5s, Check Status, Is Complete?, Has Failed?, Wait 10s
- **Node Details:**

  - **Log Request**
    - Type: Code node (JavaScript).
    - Configuration:
      - Logs current prediction ID, timestamp, and model type for monitoring.
    - Inputs: From Create Image Prediction node.
    - Outputs: To Wait 5s node.
    - Edge Cases: Logging failures do not affect workflow; console output only.

  - **Wait 5s**
    - Type: Wait node.
    - Configuration: Waits 5 seconds before next action.
    - Inputs: From Log Request.
    - Outputs: To Check Status node.
    - Edge Cases: None.

  - **Check Status**
    - Type: HTTP Request node.
    - Configuration:
      - GET request to `https://api.replicate.com/v1/predictions/{prediction_id}`.
      - Uses token from Set API Token node.
      - Response JSON with status field.
    - Inputs: From Wait 5s or Wait 10s nodes.
    - Outputs: To Is Complete? node.
    - Edge Cases:
      - Invalid prediction ID returns 404.
      - API rate limits or network issues may cause errors.

  - **Is Complete?**
    - Type: If node.
    - Configuration:
      - Checks if `status == "succeeded"`.
      - If true, routes to Success Response.
      - Else routes to Has Failed? node.
    - Inputs: From Check Status.
    - Outputs: To Success Response or Has Failed?.
    - Edge Cases: Unexpected status values require fallback handling.

  - **Has Failed?**
    - Type: If node.
    - Configuration:
      - Checks if `status == "failed"`.
      - If true, routes to Error Response.
      - Else routes to Wait 10s node to retry.
    - Inputs: From Is Complete?.
    - Outputs: To Error Response or Wait 10s.
    - Edge Cases: Unknown or intermediate statuses trigger retries.

  - **Wait 10s**
    - Type: Wait node.
    - Configuration: Waits 10 seconds before retrying status check.
    - Inputs: From Has Failed?.
    - Outputs: To Check Status.
    - Edge Cases: None.

#### 1.5 Response Handling

- **Overview:** Constructs structured JSON responses depending on success or failure of image generation.
- **Nodes Involved:** Success Response, Error Response, Display Result
- **Node Details:**

  - **Success Response**
    - Type: Set node.
    - Configuration:
      - Sets response object with:
        - success: true
        - image_url: output URL from prediction
        - prediction_id, status, message.
    - Inputs: From Is Complete? (success branch).
    - Outputs: To Display Result node.
    - Edge Cases: Missing output URL would lead to incomplete response.

  - **Error Response**
    - Type: Set node.
    - Configuration:
      - Sets response object with:
        - success: false
        - error message from JSON or default text
        - prediction_id, status, message.
    - Inputs: From Has Failed? (failure branch).
    - Outputs: To Display Result node.
    - Edge Cases: Error details might be missing in some API responses.

  - **Display Result**
    - Type: Set node.
    - Configuration:
      - Assigns the final response object to `final_result` variable.
      - Intended for output or further use.
    - Inputs: From Success Response or Error Response.
    - Outputs: None (end node).
    - Edge Cases: None.

#### 1.6 Logging and Output

- **Overview:** Logs key details of the generation request for monitoring and debugging.
- **Nodes Involved:** Log Request (already covered in 1.4)
- **Node Details:** See above.

#### 1.7 Documentation and Notes

- **Overview:** Sticky notes provide rich documentation, instructions, and support references embedded in the workflow.
- **Nodes Involved:** Sticky Note9, Sticky Note4
- **Node Details:**

  - **Sticky Note9**
    - Content: Contact info (Yaron@nofluff.online), links to YouTube and LinkedIn for support.
    - Position: Top-left, visually separated.
  
  - **Sticky Note4**
    - Content: Extensive documentation including model overview, parameter guide, workflow explanation, benefits, quick start instructions, troubleshooting, and resource links.
    - Position: Large note on the left side of the workflow canvas.
  
  - Edge Cases: None; purely informational.

---

### 3. Summary Table

| Node Name            | Node Type              | Functional Role                  | Input Node(s)              | Output Node(s)              | Sticky Note                                                                                               |
|----------------------|------------------------|---------------------------------|----------------------------|-----------------------------|----------------------------------------------------------------------------------------------------------|
| Manual Trigger       | Manual Trigger          | Start workflow                  | None                       | Set API Token               |                                                                                                          |
| Set API Token        | Set                    | Store Replicate API token       | Manual Trigger             | Set Image Parameters        |                                                                                                          |
| Set Image Parameters | Set                    | Configure image generation params| Set API Token              | Create Image Prediction     |                                                                                                          |
| Create Image Prediction | HTTP Request          | Submit image generation request | Set Image Parameters       | Log Request                 |                                                                                                          |
| Log Request          | Code                   | Log prediction details          | Create Image Prediction    | Wait 5s                     |                                                                                                          |
| Wait 5s              | Wait                   | Pause before status check       | Log Request                | Check Status                |                                                                                                          |
| Check Status         | HTTP Request           | Query prediction status         | Wait 5s, Wait 10s          | Is Complete?                |                                                                                                          |
| Is Complete?         | If                     | Check if prediction succeeded   | Check Status               | Success Response, Has Failed? |                                                                                                          |
| Has Failed?          | If                     | Check if prediction failed      | Is Complete?               | Error Response, Wait 10s    |                                                                                                          |
| Wait 10s             | Wait                   | Retry delay before next status check | Has Failed?            | Check Status                |                                                                                                          |
| Success Response     | Set                    | Build success JSON response     | Is Complete? (success)     | Display Result              |                                                                                                          |
| Error Response       | Set                    | Build error JSON response       | Has Failed? (failure)      | Display Result              |                                                                                                          |
| Display Result       | Set                    | Output final response           | Success Response, Error Response | None                   |                                                                                                          |
| Sticky Note9         | Sticky Note            | Contact & support information   | None                       | None                       | Contact: Yaron@nofluff.online; YouTube & LinkedIn links                                                  |
| Sticky Note4         | Sticky Note            | Full workflow & parameter documentation | None                   | None                       | Detailed documentation, instructions, troubleshooting, links                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger node:**
   - No configuration needed.
   - Position it as your workflow start node.

2. **Create Set node named "Set API Token":**
   - Add a string variable `api_token`.
   - Set value to your Replicate API token string (replace `"YOUR_REPLICATE_API_TOKEN"`).
   - Connect Manual Trigger output to this node.

3. **Create Set node named "Set Image Parameters":**
   - Add multiple variables:
     - `api_token` as string, copy from previous node with expression `{{$node["Set API Token"].json.api_token}}`.
     - `mask` (string), default: `https://via.placeholder.com/512x512/000000/FFFFFF.png`.
     - `seed` (number), default: -1.
     - `image` (string), default: `https://picsum.photos/512/512`.
     - `width` (number), default: 1024.
     - `height` (number), default: 1024.
     - `prompt` (string), default: `An astronaut riding a rainbow unicorn`.
     - `refine` (string), default: `no_refiner`.
     - `scheduler` (string), default: `K_EULER`.
     - `lora_scale` (number), default: 0.6.
     - `num_outputs` (number), default: 1.
     - `refine_steps` (number), default: 1.
     - `guidance_scale` (number), default: 7.5.
     - `apply_watermark` (boolean), default: true.
     - `high_noise_frac` (number), default: 0.8.
     - `negative_prompt` (string), default: empty.
     - `prompt_strength` (number), default: 0.8.
     - `replicate_weights` (string), default: empty.
     - `num_inference_steps` (number), default: 50.
     - `disable_safety_checker` (boolean), default: false.
   - Connect output of "Set API Token" node here.

4. **Create HTTP Request node named "Create Image Prediction":**
   - Method: POST
   - URL: `https://api.replicate.com/v1/predictions`
   - Headers:
     - Authorization: `Bearer {{$json.api_token}}`
     - Prefer: `wait`
   - Body (JSON):
     ```json
     {
       "version": "stability-ai/sdxl:7762fd07cf82c948538e41f63f77d685e02b063e37e496e96eefd46c929f9bdc",
       "input": {
         "mask": "{{$json.mask}}",
         "seed": {{$json.seed}},
         "image": "{{$json.image}}",
         "width": {{$json.width}},
         "height": {{$json.height}},
         "prompt": "{{$json.prompt}}",
         "refine": "{{$json.refine}}",
         "scheduler": "{{$json.scheduler}}",
         "lora_scale": {{$json.lora_scale}},
         "num_outputs": {{$json.num_outputs}},
         "refine_steps": {{$json.refine_steps}},
         "guidance_scale": {{$json.guidance_scale}},
         "apply_watermark": {{$json.apply_watermark}},
         "high_noise_frac": {{$json.high_noise_frac}},
         "negative_prompt": "{{$json.negative_prompt}}",
         "prompt_strength": {{$json.prompt_strength}},
         "replicate_weights": "{{$json.replicate_weights}}",
         "num_inference_steps": {{$json.num_inference_steps}},
         "disable_safety_checker": {{$json.disable_safety_checker}}
       }
     }
     ```
   - Set "Send Body" and "Send Headers" to true.
   - Response format: JSON, never error.
   - Connect output of "Set Image Parameters" node here.

5. **Create Code node named "Log Request":**
   - JavaScript code:
     ```javascript
     const data = $input.all()[0].json;
     console.log('stability-ai/sdxl Request:', {
       timestamp: new Date().toISOString(),
       prediction_id: data.id,
       model_type: 'image'
     });
     return $input.all();
     ```
   - Connect output of "Create Image Prediction" node here.

6. **Create Wait node named "Wait 5s":**
   - Unit: seconds
   - Amount: 5
   - Connect output of "Log Request" node here.

7. **Create HTTP Request node named "Check Status":**
   - Method: GET
   - URL: `https://api.replicate.com/v1/predictions/{{$node["Create Image Prediction"].json.id}}`
   - Header:
     - Authorization: `Bearer {{$node["Set API Token"].json.api_token}}`
   - Response format: JSON, never error.
   - Connect output of "Wait 5s" node here.

8. **Create If node named "Is Complete?":**
   - Condition: Check if `{{$json.status}} === "succeeded"`
   - If true: connect to "Success Response"
   - If false: connect to "Has Failed?"
   - Connect output of "Check Status" node here.

9. **Create If node named "Has Failed?":**
   - Condition: Check if `{{$json.status}} === "failed"`
   - If true: connect to "Error Response"
   - If false: connect to "Wait 10s"
   - Connect false output of "Is Complete?" node here.

10. **Create Wait node named "Wait 10s":**
    - Unit: seconds
    - Amount: 10
    - Connect false output of "Has Failed?" node here.
    - Connect output back to "Check Status" node for retry loop.

11. **Create Set node named "Success Response":**
    - Assign variable `response` (object) with:
      ```json
      {
        "success": true,
        "image_url": {{$json.output}},
        "prediction_id": {{$json.id}},
        "status": {{$json.status}},
        "message": "Image generated successfully"
      }
      ```
    - Connect true output of "Is Complete?" node here.

12. **Create Set node named "Error Response":**
    - Assign variable `response` (object) with:
      ```json
      {
        "success": false,
        "error": {{$json.error || "Image generation failed"}},
        "prediction_id": {{$json.id}},
        "status": {{$json.status}},
        "message": "Failed to generate image"
      }
      ```
    - Connect true output of "Has Failed?" node here.

13. **Create Set node named "Display Result":**
    - Assign variable `final_result` (object) to `{{$json.response}}`.
    - Connect outputs of both "Success Response" and "Error Response" nodes here (merge branches).
    - This node represents the final output of the workflow.

14. **Add Sticky Notes for documentation:**
    - Add one with contact and social links.
    - Add another large note with detailed parameter explanations, workflow overview, quick start, troubleshooting, and resource links.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | Context or Link                                                                                  |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| =======================================  SDXL GENERATOR ======================================= For any questions or support, please contact: Yaron@nofluff.online Explore more tips and tutorials here: - YouTube: https://www.youtube.com/@YaronBeen/videos - LinkedIn: https://www.linkedin.com/in/yaronbeen/ =======================================                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               | Contact & support information embedded as sticky note in workflow canvas                       |
| ## 🤖 **STABILITY-AI/SDXL - IMAGE GENERATION WORKFLOW** Powered by Replicate API and n8n Automation Model Owner: stability-ai Model: sdxl Type: Image Generation API Endpoint: https://api.replicate.com/v1/predictions What This Model Does: Text-to-image generative AI creating beautiful images Parameter Guide: mask, seed, image, width, height, prompt, refine, scheduler, lora_scale, num_outputs, refine_steps, guidance_scale, apply_watermark, high_noise_frac, negative_prompt, prompt_strength, replicate_weights, num_inference_steps, disable_safety_checker Workflow Components: Manual Trigger, API Token Setup, Parameter Setting, Prediction Request, Status Polling, Success/Error Handling, Logging Benefits: Instant generation, Automation, Error Resilience, Production readiness, Customizable, Efficient processing Quick Start: Get API Key at https://replicate.com Configure workflow parameters Run workflow Monitor logs and outputs Troubleshooting: Invalid token, Parameter validation, Timeouts, Output format Resources: Model Docs: https://replicate.com/stability-ai/sdxl Replicate API Docs: https://replicate.com/docs n8n Docs: https://docs.n8n.io | Comprehensive workflow and parameter documentation in sticky note on canvas |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.