Create Images from Text Prompts using Jhonpiedrahita AI01 and Replicate

https://n8nworkflows.xyz/workflows/create-images-from-text-prompts-using-jhonpiedrahita-ai01-and-replicate-6803


# Create Images from Text Prompts using Jhonpiedrahita AI01 and Replicate

### 1. Workflow Overview

This workflow automates the generation of images from text prompts using the AI model **jhonpiedrahita_ai01** via the Replicate API. It is designed to receive a manual trigger, set necessary API credentials and generation parameters, invoke the model to create images, monitor the asynchronous prediction status, handle success or failure responses, and log request details for monitoring.

The workflow is logically divided into these main blocks:

- **1.1 Input Initialization and Parameter Setup**  
  Receives trigger, sets API authentication token, and configures all required and optional parameters for image generation.

- **1.2 Image Generation Request**  
  Submits the generation request to the Replicate API and logs the request metadata.

- **1.3 Prediction Status Monitoring Loop**  
  Periodically waits and checks the prediction status until completion or failure.

- **1.4 Response Handling and Output**  
  Routes completed predictions to success processing or handles errors gracefully, packaging structured JSON responses.

- **1.5 Logging and Display**  
  Logs key request and response data for traceability and formats the final output.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Initialization and Parameter Setup

**Overview:**  
This block initializes the workflow execution from a manual trigger, sets the Replicate API token, and defines all parameters required for the image generation API call.

**Nodes Involved:**  
- Manual Trigger  
- Set API Token  
- Set Other Parameters

**Node Details:**

- **Manual Trigger**  
  - Type: Trigger node (manual)  
  - Role: Starts the workflow manually through UI  
  - Config: No parameters needed  
  - Input: None  
  - Output: Triggers next node  

- **Set API Token**  
  - Type: Set node  
  - Role: Stores the Replicate API token securely for downstream use  
  - Config: A string parameter `api_token` set to placeholder `'YOUR_REPLICATE_API_TOKEN'` (to be replaced by user)  
  - Input: From Manual Trigger  
  - Output: Passes token to next node  
  - Edge Cases: Missing or invalid token will cause auth failures downstream  

- **Set Other Parameters**  
  - Type: Set node  
  - Role: Defines all input parameters required for the model call, including prompt, image, mask, size, and advanced options  
  - Configuration Highlights:  
    - Copies `api_token` from previous node  
    - Sets default parameters such as:  
      - `mask`: placeholder image URL for inpainting mask  
      - `seed`: -1 (random)  
      - `image`: placeholder image URL  
      - `model`: "dev" (default model variant)  
      - Dimensions: 512x512  
      - `prompt`: "Create something amazing"  
      - Flags: `go_fast` false, `disable_safety_checker` false, etc.  
    - Includes multiple numeric and string parameters covering all model options  
  - Input: From Set API Token  
  - Output: Passes parameters to HTTP request node  
  - Edge Cases: Misconfigured parameters (e.g., invalid URLs, bad types) may cause API errors  

---

#### 2.2 Image Generation Request

**Overview:**  
Submits the image generation prediction request to the Replicate API using the configured parameters and logs request details.

**Nodes Involved:**  
- Create Other Prediction  
- Log Request

**Node Details:**

- **Create Other Prediction**  
  - Type: HTTP Request node (POST)  
  - Role: Sends a JSON body to Replicate API endpoint `/v1/predictions` to create a new prediction  
  - Config:  
    - URL: `https://api.replicate.com/v1/predictions`  
    - Method: POST  
    - Headers: Authorization Bearer token from `api_token`, Prefer: wait (to wait for synchronous response when possible)  
    - Body: JSON with `version` set to specific model version and `input` object with all parameters from previous node  
    - Sends body as JSON  
    - Response format: JSON, error suppressed (`neverError: true`) to handle gracefully  
  - Input: From Set Other Parameters  
  - Output: JSON response with prediction ID and initial status  
  - Edge Cases:  
    - HTTP errors or bad token lead to failure response  
    - Network timeouts possible  
    - API limits or quota exceeded  

- **Log Request**  
  - Type: Code node (JavaScript)  
  - Role: Logs important metadata about the prediction request for monitoring/debugging  
  - Code: Logs timestamp, prediction ID, and model type to console  
  - Input: From Create Other Prediction  
  - Output: Passes data downstream to Wait node  
  - Edge Cases: Logging failures do not impact workflow; purely informational  

---

#### 2.3 Prediction Status Monitoring Loop

**Overview:**  
After submitting the prediction, the workflow periodically waits and checks the status until the prediction completes successfully or fails.

**Nodes Involved:**  
- Wait 5s  
- Check Status  
- Is Complete?  
- Has Failed?  
- Wait 10s

**Node Details:**

- **Wait 5s**  
  - Type: Wait node  
  - Role: Pauses execution for 5 seconds before next status check  
  - Config: 5 seconds delay  
  - Input: From Log Request  
  - Output: Connects to Check Status node  

- **Check Status**  
  - Type: HTTP Request node (GET)  
  - Role: Queries Replicate API for current prediction status using prediction ID  
  - Config:  
    - URL constructed dynamically with prediction ID from `Create Other Prediction` node  
    - Method: GET  
    - Headers: Authorization with API token from `Set API Token` node  
    - Response format: JSON, error suppressed (`neverError: true`)  
  - Input: From Wait nodes  
  - Output: Passes status info downstream to decision nodes  

- **Is Complete?**  
  - Type: If node  
  - Role: Checks if prediction status equals `"succeeded"`  
  - Config: Condition: `$json.status == 'succeeded'`  
  - Input: From Check Status  
  - Output:  
    - True branch to Success Response node  
    - False branch to Has Failed? node  

- **Has Failed?**  
  - Type: If node  
  - Role: Checks if prediction status equals `"failed"`  
  - Config: Condition: `$json.status == 'failed'`  
  - Input: From Is Complete? (False branch)  
  - Output:  
    - True branch to Error Response node  
    - False branch to Wait 10s node (retry loop)  

- **Wait 10s**  
  - Type: Wait node  
  - Role: Pauses for 10 seconds before retrying status check (used for retry delays)  
  - Config: 10 seconds delay  
  - Input: From Has Failed? (False branch)  
  - Output: Loops back to Check Status node  

**Edge Cases:**  
- API may return intermediate statuses (e.g., processing) requiring retries  
- Network failures during status checks  
- Infinite loop if prediction neither succeeds nor fails — workflow design relies on external API to resolve status eventually  

---

#### 2.4 Response Handling and Output

**Overview:**  
Handles both successful and failed prediction outcomes by formatting structured JSON responses for downstream consumption or display.

**Nodes Involved:**  
- Success Response  
- Error Response  
- Display Result

**Node Details:**

- **Success Response**  
  - Type: Set node  
  - Role: Constructs a JSON object indicating success, including the generated image URL, prediction ID, status, and a message  
  - Config: Sets field `response` with keys: `success: true`, `result_url` (from output URL), `prediction_id`, `status`, and message string  
  - Input: From Is Complete? (True branch)  
  - Output: Passes to Display Result node  

- **Error Response**  
  - Type: Set node  
  - Role: Constructs an error JSON response with failure info, including error message or fallback text, prediction ID, and status  
  - Input: From Has Failed? (True branch)  
  - Output: Passes to Display Result node  

- **Display Result**  
  - Type: Set node  
  - Role: Sets a final output field `final_result` with the structured response object from prior node (success or error)  
  - Input: From Success Response or Error Response nodes  
  - Output: Final output of workflow (could be used downstream or as webhook response)  

**Edge Cases:**  
- Missing or malformed output URLs in success response  
- Error node relies on presence of `error` field in JSON, fallback included  
- Downstream consumers must handle this structured output format  

---

#### 2.5 Logging and User Information

**Overview:**  
Two sticky note nodes provide user guidance, support contacts, documentation links, and detailed explanations of parameters and workflow purpose.

**Nodes Involved:**  
- Sticky Note9  
- Sticky Note4

**Node Details:**

- **Sticky Note9**  
  - Content: Contact info for support (Yaron@nofluff.online), links to YouTube and LinkedIn for tips and tutorials  
  - Purpose: User assistance and branding  

- **Sticky Note4**  
  - Content: Detailed workflow description, model overview, parameter details, instructions for usage, troubleshooting guide, best practices, and resource links  
  - Purpose: In-depth contextual information and onboarding  

These nodes do not affect workflow execution but serve as embedded documentation within the n8n canvas.

---

### 3. Summary Table

| Node Name             | Node Type               | Functional Role                          | Input Node(s)                          | Output Node(s)                    | Sticky Note                                                                                                                                                                                                                  |
|-----------------------|-------------------------|----------------------------------------|--------------------------------------|----------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Manual Trigger        | n8n-nodes-base.manualTrigger | Workflow start trigger                   | None                                 | Set API Token                    | See Sticky Note4 for detailed workflow overview and instructions                                                                                                                                                             |
| Set API Token         | n8n-nodes-base.set       | Stores API authentication token         | Manual Trigger                       | Set Other Parameters             | See Sticky Note4 for parameter reference and usage                                                                                                                                                                          |
| Set Other Parameters  | n8n-nodes-base.set       | Defines all model input parameters       | Set API Token                       | Create Other Prediction          | See Sticky Note4 for detailed parameter explanations                                                                                                                                                                        |
| Create Other Prediction | n8n-nodes-base.httpRequest | Sends image generation request to API   | Set Other Parameters                 | Log Request                     | See Sticky Note4 for API endpoint and generation details                                                                                                                                                                    |
| Log Request           | n8n-nodes-base.code      | Logs prediction request metadata         | Create Other Prediction             | Wait 5s                        | See Sticky Note4 for monitoring and logging best practices                                                                                                                                                                  |
| Wait 5s               | n8n-nodes-base.wait      | Waits 5 seconds before status check      | Log Request                        | Check Status                   |                                                                                                                                                                                                                              |
| Check Status          | n8n-nodes-base.httpRequest | Checks prediction status from API        | Wait 5s, Wait 10s                  | Is Complete?                   |                                                                                                                                                                                                                              |
| Is Complete?          | n8n-nodes-base.if        | Checks if prediction succeeded           | Check Status                       | Success Response, Has Failed?   |                                                                                                                                                                                                                              |
| Has Failed?           | n8n-nodes-base.if        | Checks if prediction failed               | Is Complete?                      | Error Response, Wait 10s         |                                                                                                                                                                                                                              |
| Wait 10s              | n8n-nodes-base.wait      | Waits 10 seconds before retrying status  | Has Failed?                      | Check Status                   |                                                                                                                                                                                                                              |
| Success Response      | n8n-nodes-base.set       | Prepares structured success JSON response | Is Complete? (true branch)          | Display Result                 |                                                                                                                                                                                                                              |
| Error Response        | n8n-nodes-base.set       | Prepares structured error JSON response   | Has Failed? (true branch)           | Display Result                 |                                                                                                                                                                                                                              |
| Display Result        | n8n-nodes-base.set       | Sets final output object                   | Success Response, Error Response   | None                          |                                                                                                                                                                                                                              |
| Sticky Note9          | n8n-nodes-base.stickyNote | User contact and support info              | None                               | None                          | Contact: Yaron@nofluff.online; YouTube: https://www.youtube.com/@YaronBeen/videos; LinkedIn: https://www.linkedin.com/in/yaronbeen/                                                                                       |
| Sticky Note4          | n8n-nodes-base.stickyNote | Detailed workflow description and guide   | None                               | None                          | Comprehensive workflow documentation, parameter guide, troubleshooting, and resource links (https://replicate.com/jhonp4/jhonpiedrahita_ai01, https://replicate.com/docs, https://docs.n8n.io)                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Manual Trigger node:**  
   - Name: `Manual Trigger`  
   - No configuration needed.

3. **Add a Set node to store API token:**  
   - Name: `Set API Token`  
   - Add a string field `api_token` with value `"YOUR_REPLICATE_API_TOKEN"` (replace with your actual token).  
   - Connect from `Manual Trigger`.

4. **Add a Set node to define model parameters:**  
   - Name: `Set Other Parameters`  
   - Add the following fields (types and default values):  
     - `api_token` (string): `={{ $('Set API Token').item.json.api_token }}` (copy token from previous node)  
     - `mask` (string): `"https://via.placeholder.com/512x512/000000/FFFFFF.png"`  
     - `seed` (number): `-1`  
     - `image` (string): `"https://picsum.photos/512/512"`  
     - `model` (string): `"dev"`  
     - `width` (number): `512`  
     - `height` (number): `512`  
     - `prompt` (string): `"Create something amazing"`  
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
   - Connect from `Set API Token`.

5. **Add HTTP Request node to create prediction:**  
   - Name: `Create Other Prediction`  
   - HTTP Method: POST  
   - URL: `https://api.replicate.com/v1/predictions`  
   - Authentication: None (token passed in headers)  
   - Headers:  
     - `Authorization`: `Bearer {{ $json.api_token }}`  
     - `Prefer`: `wait`  
   - Body Content Type: JSON  
   - Body JSON:  
     ```json
     {
       "version": "jhonp4/jhonpiedrahita_ai01:fcdcce61392c00f15318346038baa7c83af0a37e3e37a0318dd6aa8de33684b2",
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
   - Connect from `Set Other Parameters`.

6. **Add Code node for logging request:**  
   - Name: `Log Request`  
   - JavaScript code:  
     ```javascript
     const data = $input.all()[0].json;
     console.log('jhonp4/jhonpiedrahita_ai01 Request:', {
       timestamp: new Date().toISOString(),
       prediction_id: data.id,
       model_type: 'other'
     });
     return $input.all();
     ```  
   - Connect from `Create Other Prediction`.

7. **Add Wait node to pause 5 seconds:**  
   - Name: `Wait 5s`  
   - Wait time: 5 seconds  
   - Connect from `Log Request`.

8. **Add HTTP Request node to check status:**  
   - Name: `Check Status`  
   - HTTP Method: GET  
   - URL: `https://api.replicate.com/v1/predictions/{{ $('Create Other Prediction').item.json.id }}`  
   - Headers:  
     - `Authorization`: `Bearer {{ $('Set API Token').item.json.api_token }}`  
   - Connect from `Wait 5s` and later from `Wait 10s`.

9. **Add If node to check if complete:**  
   - Name: `Is Complete?`  
   - Condition: Check if `$json.status == 'succeeded'`  
   - Connect from `Check Status`.

10. **Add If node to check if failed:**  
    - Name: `Has Failed?`  
    - Condition: Check if `$json.status == 'failed'`  
    - Connect from `Is Complete?` (False branch).

11. **Add Wait node to pause 10 seconds (retry delay):**  
    - Name: `Wait 10s`  
    - Wait time: 10 seconds  
    - Connect from `Has Failed?` (False branch) back to `Check Status`.

12. **Add Set node for success response:**  
    - Name: `Success Response`  
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
    - Connect from `Is Complete?` (True branch).

13. **Add Set node for error response:**  
    - Name: `Error Response`  
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
    - Connect from `Has Failed?` (True branch).

14. **Add Set node to display final result:**  
    - Name: `Display Result`  
    - Set field `final_result` to `{{ $json.response }}`  
    - Connect from both `Success Response` and `Error Response`.

15. **Add sticky notes for documentation and contact info (optional):**  
    - Add two Sticky Note nodes with detailed instructions, parameter references, and support contact as described in sticky notes.

16. **Activate and test the workflow:**  
    - Replace `'YOUR_REPLICATE_API_TOKEN'` with a valid API token.  
    - Adjust parameters in `Set Other Parameters` as needed for your use case.  
    - Trigger manually and monitor execution.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                      | Context or Link                                                                                           |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| Contact support at Yaron@nofluff.online for questions related to this workflow or model usage.                                                                                   | Sticky Note9                                                                                              |
| Explore tips and tutorials on YouTube: https://www.youtube.com/@YaronBeen/videos and LinkedIn: https://www.linkedin.com/in/yaronbeen/                                         | Sticky Note9                                                                                              |
| Model documentation and API reference available at: https://replicate.com/jhonp4/jhonpiedrahita_ai01 and https://replicate.com/docs                                            | Sticky Note4                                                                                              |
| n8n documentation for workflow customization and node references: https://docs.n8n.io                                                                                           | Sticky Note4                                                                                              |
| Best practices include securely storing API tokens, validating parameters, and monitoring API usage to avoid quota or billing issues.                                           | Sticky Note4                                                                                              |
| The workflow uses Replicate API's `"Prefer": "wait"` header for synchronous response but implements its own polling mechanism to handle asynchronous prediction completion.       | HTTP Request nodes in "Create Other Prediction" and "Check Status"                                        |
| The model supports many advanced parameters including image masking, seed for reproducibility, inference steps, and safety checker disabling for flexible image generation.      | Set Other Parameters node                                                                                 |
| Errors and retries are handled with a combination of conditional checks and wait nodes, ensuring robust automation without manual intervention during long-running predictions.| If nodes "Is Complete?" and "Has Failed?", Wait nodes "Wait 5s" and "Wait 10s"                            |

---

**Disclaimer:**  
This document describes an automated workflow created using n8n. It complies with all current content policies and contains no illegal or protected content. All data processed is public and legal.