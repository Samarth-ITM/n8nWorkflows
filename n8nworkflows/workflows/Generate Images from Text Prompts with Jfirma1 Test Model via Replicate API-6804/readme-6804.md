Generate Images from Text Prompts with Jfirma1 Test Model via Replicate API

https://n8nworkflows.xyz/workflows/generate-images-from-text-prompts-with-jfirma1-test-model-via-replicate-api-6804


# Generate Images from Text Prompts with Jfirma1 Test Model via Replicate API

### 1. Workflow Overview

This workflow automates the generation of images from text prompts using the "test_model" by the user jfirma1 via the Replicate API. It is designed primarily for users who want to send detailed parameters to an AI model hosted on Replicate, receive generated images, and handle the asynchronous nature of the API prediction status.

The workflow logically divides into these blocks:

- **1.1 Trigger & Authentication Setup**: Initiates the workflow manually and sets the Replicate API token.
- **1.2 Parameter Configuration**: Defines all input parameters for the image generation request, including defaults.
- **1.3 Prediction Request & Logging**: Sends a generation request to Replicate API and logs the request details.
- **1.4 Status Polling & Retry Loop**: Waits and repeatedly checks the prediction status until completion or failure, with retry logic.
- **1.5 Result Handling**: Routes outputs into success or error responses based on status.
- **1.6 Output & Finalization**: Prepares and displays the final structured response.
- **1.7 Documentation & Notes**: Embedded sticky notes providing metadata, instructions, and support contacts.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger & Authentication Setup

- **Overview:** Starts the workflow on manual trigger and sets the API token for Replicate requests.
- **Nodes Involved:** `Manual Trigger`, `Set API Token`

**Manual Trigger**

- Type: Manual trigger node
- Role: Entry point to start the workflow on demand
- Config: No parameters; user clicks to start
- Inputs: None
- Outputs: Passes empty data downstream
- Edge cases: None significant; manual user error possible if not triggered
- Version: 1

**Set API Token**

- Type: Set node
- Role: Stores the Replicate API token as a workflow variable `api_token`
- Config: Hardcoded string placeholder `"YOUR_REPLICATE_API_TOKEN"` to be replaced by user
- Inputs: From Manual Trigger
- Outputs: Adds `api_token` to JSON data for downstream use
- Edge cases: Failure if token invalid or missing; user must replace placeholder
- Version: 3.3

---

#### 2.2 Parameter Configuration

- **Overview:** Defines all required and optional parameters for the model inference request, passing on the API token from previous node.
- **Nodes Involved:** `Set Other Parameters`

**Set Other Parameters**

- Type: Set node
- Role: Aggregates all parameters for the API call, including the API token copied from `Set API Token`
- Configuration:
  - Copies `api_token` from previous node to keep token in context
  - Sets parameters such as `mask`, `seed`, `image`, `model`, `width`, `height`, `prompt`, and many others with default values
  - Parameters reflect the model’s expected inputs, e.g., image URLs, numeric parameters, booleans
- Inputs: From `Set API Token`
- Outputs: JSON with complete parameter set for the prediction request
- Edge cases:
  - Parameters must conform to model expectations (e.g., numeric types, string URLs)
  - User-defined prompt critical; default is "Create something amazing"
  - If parameters are invalid or missing, API may reject request
- Version: 3.3

---

#### 2.3 Prediction Request & Logging

- **Overview:** Sends the image generation request to Replicate API and logs the request details for monitoring.
- **Nodes Involved:** `Create Other Prediction`, `Log Request`

**Create Other Prediction**

- Type: HTTP Request node
- Role: POST request to Replicate API endpoint `/v1/predictions` to start generation
- Configuration:
  - URL: `https://api.replicate.com/v1/predictions`
  - Method: POST
  - Headers: Authorization bearer token from `api_token`; `Prefer: wait` for synchronous-like response
  - Body: JSON containing model version and all input parameters dynamically injected via expressions
  - Response: JSON, never error to allow error handling downstream
- Inputs: From `Set Other Parameters`
- Outputs: Prediction response containing `id`, `status`, and other metadata
- Edge cases:
  - API token invalid or expired → auth failure
  - Network timeout or API downtime → request failure
  - Input parameters invalid → API rejects request
- Version: 4.1

**Log Request**

- Type: Code node
- Role: Logs request info to console for debugging and monitoring
- Configuration:
  - Extracts prediction ID, timestamp, and model type
  - Prints data to console log
  - Returns same data downstream unchanged
- Inputs: From `Create Other Prediction`
- Outputs: Passes data for status checking
- Edge cases:
  - Console logging failure is unlikely but would not break workflow
- Version: 2

---

#### 2.4 Status Polling & Retry Loop

- **Overview:** Implements a polling mechanism to wait for prediction completion or failure, with waits and retries.
- **Nodes Involved:** `Wait 5s`, `Check Status`, `Is Complete?`, `Has Failed?`, `Wait 10s`

**Wait 5s**

- Type: Wait node
- Role: Pauses workflow for 5 seconds before checking status
- Configuration: 5 seconds delay
- Inputs: From `Log Request`
- Outputs: Proceeds to `Check Status`
- Edge cases: Timer failure unlikely

**Check Status**

- Type: HTTP Request node
- Role: GET request to `/v1/predictions/{id}` to query current status
- Configuration:
  - URL dynamically built using prediction ID from `Create Other Prediction`
  - Authorization header with API token
  - Response JSON, never error
- Inputs: From `Wait 5s` and `Wait 10s` (retry)
- Outputs: JSON status data
- Edge cases:
  - API token invalid, network errors
  - Prediction ID invalid/missing
  - Rate limits on API calls

**Is Complete?**

- Type: If node
- Role: Checks if prediction status equals "succeeded"
- Inputs: From `Check Status`
- Outputs:
  - Yes: `Success Response`
  - No: `Has Failed?`

**Has Failed?**

- Type: If node
- Role: Checks if prediction status equals "failed"
- Inputs: From `Is Complete?`
- Outputs:
  - Yes: `Error Response`
  - No: `Wait 10s` (retry)

**Wait 10s**

- Type: Wait node
- Role: Waits 10 seconds before re-checking status to avoid API flooding
- Inputs: From `Has Failed?` (No branch)
- Outputs: Loops back to `Check Status`
- Edge cases:
  - Ensures workflow doesn't overwhelm API with requests
  - Long-running predictions handled gracefully

---

#### 2.5 Result Handling

- **Overview:** Prepares structured success or error response objects with relevant data.
- **Nodes Involved:** `Success Response`, `Error Response`

**Success Response**

- Type: Set node
- Role: Constructs a JSON object indicating success, including output URL, prediction ID, status, and message
- Inputs: From `Is Complete?` (Yes branch)
- Outputs: To `Display Result`
- Edge cases: None expected if status is succeeded

**Error Response**

- Type: Set node
- Role: Constructs a JSON object indicating failure, including error message, prediction ID, and status
- Inputs: From `Has Failed?` (Yes branch)
- Outputs: To `Display Result`
- Edge cases: Error message defaults to generic if none provided by API

---

#### 2.6 Output & Finalization

- **Overview:** Outputs the final response object for downstream consumption or display.
- **Nodes Involved:** `Display Result`

**Display Result**

- Type: Set node
- Role: Assigns the final response (success or error) to `final_result` property for output or further processing
- Inputs: From either `Success Response` or `Error Response`
- Outputs: End of workflow output
- Edge cases: None

---

#### 2.7 Documentation & Notes

- **Overview:** Provides extensive workflow metadata, usage instructions, parameter explanations, and support contacts via sticky notes.
- **Nodes Involved:** `Sticky Note9`, `Sticky Note4`

**Sticky Note9**

- Type: Sticky Note
- Content: Contact info for support, including email and links to YouTube and LinkedIn of the author

**Sticky Note4**

- Type: Sticky Note
- Content: Detailed explanation of the model, parameters, workflow components, quick start instructions, troubleshooting, and useful links

---

### 3. Summary Table

| Node Name             | Node Type           | Functional Role                          | Input Node(s)                 | Output Node(s)             | Sticky Note                                                                                 |
|-----------------------|---------------------|----------------------------------------|------------------------------|----------------------------|---------------------------------------------------------------------------------------------|
| Manual Trigger        | Manual Trigger       | Starts workflow manually                | None                         | Set API Token              |                                                                                             |
| Set API Token          | Set                 | Sets Replicate API token                | Manual Trigger               | Set Other Parameters       |                                                                                             |
| Set Other Parameters   | Set                 | Configures all model input parameters   | Set API Token                | Create Other Prediction    |                                                                                             |
| Create Other Prediction| HTTP Request        | Sends generation request to Replicate   | Set Other Parameters         | Log Request                |                                                                                             |
| Log Request            | Code                | Logs request details for monitoring     | Create Other Prediction      | Wait 5s                    |                                                                                             |
| Wait 5s                | Wait                | Waits 5 seconds before status check     | Log Request                  | Check Status               |                                                                                             |
| Check Status           | HTTP Request        | Queries prediction status                | Wait 5s, Wait 10s            | Is Complete?               |                                                                                             |
| Is Complete?           | If                  | Checks if prediction succeeded          | Check Status                 | Success Response, Has Failed? |                                                                                             |
| Has Failed?            | If                  | Checks if prediction failed             | Is Complete?                 | Error Response, Wait 10s   |                                                                                             |
| Wait 10s               | Wait                | Waits 10 seconds before retry           | Has Failed?                  | Check Status               |                                                                                             |
| Success Response       | Set                 | Formats success response JSON            | Is Complete?                 | Display Result             |                                                                                             |
| Error Response         | Set                 | Formats error response JSON              | Has Failed?                  | Display Result             |                                                                                             |
| Display Result         | Set                 | Outputs final result object               | Success Response, Error Response | None                    |                                                                                             |
| Sticky Note9           | Sticky Note         | Support contact and social links         | None                        | None                      | Contact: Yaron@nofluff.online; YouTube: https://www.youtube.com/@YaronBeen/videos; LinkedIn: https://www.linkedin.com/in/yaronbeen/ |
| Sticky Note4           | Sticky Note         | Detailed workflow, parameters, instructions | None                     | None                      | Extensive documentation and quick start guide embedded in this note                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node**
   - Name: `Manual Trigger`
   - No parameters
   - Position: Start of workflow

2. **Add a Set Node for API Token**
   - Name: `Set API Token`
   - Add assignment: `api_token` (string) = `"YOUR_REPLICATE_API_TOKEN"`
   - Connect `Manual Trigger` output to this node
   - Note: Replace `"YOUR_REPLICATE_API_TOKEN"` with your actual Replicate API token

3. **Add a Set Node for Other Parameters**
   - Name: `Set Other Parameters`
   - Copy `api_token` from `Set API Token` via expression: `={{ $('Set API Token').item.json.api_token }}`
   - Define assignments for the following parameters with these default values:
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
   - Connect `Set API Token` output to this node

4. **Add HTTP Request Node to Create Prediction**
   - Name: `Create Other Prediction`
   - Method: POST
   - URL: `https://api.replicate.com/v1/predictions`
   - Headers:
     - `Authorization`: `Bearer {{ $json.api_token }}`
     - `Prefer`: `wait`
   - Body Content (JSON):
     ```json
     {
       "version": "jfirma1/test_model:591b4311afe2900e6927e2b9802496dffd78fe8baa8f0dbb6e4961172fd3bb1d",
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
   - Set "Send Body" and "Send Headers" to true
   - Set "Response Format" to JSON, enable "Never Error" to allow error handling
   - Connect `Set Other Parameters` output here

5. **Add Code Node for Logging**
   - Name: `Log Request`
   - JavaScript code:
     ```javascript
     const data = $input.all()[0].json;
     console.log('jfirma1/test_model Request:', {
       timestamp: new Date().toISOString(),
       prediction_id: data.id,
       model_type: 'other'
     });
     return $input.all();
     ```
   - Connect `Create Other Prediction` output here

6. **Add Wait Node (5 seconds)**
   - Name: `Wait 5s`
   - Wait 5 seconds before next step
   - Connect `Log Request` output here

7. **Add HTTP Request Node to Check Status**
   - Name: `Check Status`
   - Method: GET
   - URL: `https://api.replicate.com/v1/predictions/{{ $('Create Other Prediction').item.json.id }}`
   - Headers:
     - `Authorization`: `Bearer {{ $('Set API Token').item.json.api_token }}`
   - Response Format: JSON, enable "Never Error"
   - Connect `Wait 5s` output here
   - Also will connect retry loop from `Wait 10s`

8. **Add If Node to Check Completion**
   - Name: `Is Complete?`
   - Condition: Check if `status` equals `"succeeded"`
   - Connect `Check Status` output here

9. **Add If Node to Check Failure**
   - Name: `Has Failed?`
   - Condition: Check if `status` equals `"failed"`
   - Connect `Is Complete?` (No branch) here

10. **Add Wait Node (10 seconds)**
    - Name: `Wait 10s`
    - Wait 10 seconds before retry
    - Connect `Has Failed?` (No branch) here
    - Connect `Wait 10s` output back to `Check Status` to form polling loop

11. **Add Set Node for Success Response**
    - Name: `Success Response`
    - Create JSON assignment:
      ```json
      {
        "success": true,
        "result_url": $json.output,
        "prediction_id": $json.id,
        "status": $json.status,
        "message": "Other generated successfully"
      }
      ```
    - Connect `Is Complete?` (Yes branch) here

12. **Add Set Node for Error Response**
    - Name: `Error Response`
    - Create JSON assignment:
      ```json
      {
        "success": false,
        "error": $json.error || "Other generation failed",
        "prediction_id": $json.id,
        "status": $json.status,
        "message": "Failed to generate other"
      }
      ```
    - Connect `Has Failed?` (Yes branch) here

13. **Add Set Node to Display Final Result**
    - Name: `Display Result`
    - Assign `final_result` = `$json.response` (from either success or error)
    - Connect `Success Response` and `Error Response` outputs here

14. **Add Sticky Notes**
    - Add two sticky notes with the provided content documenting:
      - Workflow description, parameters, instructions, support contact

---

### 5. General Notes & Resources

| Note Content                                                                                                                     | Context or Link                                                      |
|---------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------|
| Workflow designed for generating images with jfirma1's test_model via Replicate API                                            | Model Owner: jfirma1, Model: test_model, API: https://replicate.com |
| Contact for support: Yaron@nofluff.online                                                                                       | Email contact for workflow queries                                 |
| YouTube channel for tips and tutorials: https://www.youtube.com/@YaronBeen/videos                                               | Video resources for n8n and Replicate integration                  |
| LinkedIn profile: https://www.linkedin.com/in/yaronbeen/                                                                       | Professional contact                                               |
| Model documentation: https://replicate.com/jfirma1/test_model                                                                   | Official model info and parameters                                 |
| Replicate API docs: https://replicate.com/docs                                                                                  | API reference                                                     |
| n8n documentation: https://docs.n8n.io                                                                                          | Automation platform docs                                          |

---

This comprehensive reference document enables advanced users and AI agents to understand, reproduce, customize, and troubleshoot the image generation workflow from text prompts using the jfirma1 test model on Replicate via n8n automation.