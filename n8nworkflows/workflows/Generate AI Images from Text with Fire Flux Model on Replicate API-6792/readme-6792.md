Generate AI Images from Text with Fire Flux Model on Replicate API

https://n8nworkflows.xyz/workflows/generate-ai-images-from-text-with-fire-flux-model-on-replicate-api-6792


# Generate AI Images from Text with Fire Flux Model on Replicate API

### 1. Workflow Overview

This n8n workflow automates the generation of AI images from text prompts using the Fire Flux model hosted on the Replicate API. It is designed to take a user-initiated trigger, send a request with configurable parameters to the Replicate image generation endpoint, poll for operation completion, and finally deliver the generated image URL or an error response. The workflow is well suited for developers or AI practitioners who want to integrate AI-powered image creation into applications or pipelines with minimal manual intervention.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Initialization**: Manual trigger and API token setup.
- **1.2 Parameter Configuration**: Setting of image generation parameters including prompt and quality controls.
- **1.3 Image Generation Request**: Sending the request to Replicate API and initiating prediction.
- **1.4 Polling and Status Checking**: Loop for waiting and checking prediction status until completion or failure.
- **1.5 Success and Failure Handling**: Branching to structured success or error responses.
- **1.6 Logging and Result Display**: Logging generation details and outputting the final result.
- **1.7 Documentation and User Guidance**: Sticky notes for instructions, parameter explanations, and troubleshooting.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Initialization

- **Overview**: This block initiates the workflow manually and securely sets the Replicate API token required for authentication.
- **Nodes Involved**: Manual Trigger, Set API Token

**Node Details:**

- **Manual Trigger**
  - Type: manualTrigger
  - Role: Starts workflow execution on user command.
  - Configuration: Default, no parameters.
  - Inputs: None.
  - Outputs: Triggers the next node.
  - Edge Cases: User must manually trigger; no automation on schedule.
  
- **Set API Token**
  - Type: set
  - Role: Stores the Replicate API token as a string.
  - Configuration: Assigns a string variable `api_token` with placeholder `"YOUR_REPLICATE_API_TOKEN"`.
  - Inputs: Manual Trigger output.
  - Outputs: Supplies `api_token` to downstream nodes.
  - Edge Cases: Failure if token is invalid or missing; user must replace placeholder with a valid token.

#### 1.2 Parameter Configuration

- **Overview**: Defines all image generation input parameters including prompt text, image dimensions, quality, and generation options.
- **Nodes Involved**: Set Image Parameters

**Node Details:**

- **Set Image Parameters**
  - Type: set
  - Role: Prepares all parameters for the prediction request.
  - Configuration:
    - Copies `api_token` from previous node.
    - Sets parameters with defaults:
      - `seed`: 0 (for reproducible randomness)
      - `prompt`: "A beautiful landscape with mountains and a lake at sunset"
      - `go_fast`: true (speed-optimized model)
      - `megapixels`: "1"
      - `num_outputs`: 1
      - `aspect_ratio`: "2:1"
      - `output_format`: "png"
      - `output_quality`: 80 (on a scale 0-100)
      - `num_inference_steps`: 4 (processing steps)
      - `disable_safety_checker`: false
  - Inputs: Output from Set API Token.
  - Outputs: Parameters object to HTTP request node.
  - Edge Cases:
    - Incorrect parameter types or ranges could cause API errors.
    - User can customize parameters for different output styles.

#### 1.3 Image Generation Request

- **Overview**: Sends the configured parameters to the Replicate API to create an image prediction and obtains a prediction ID.
- **Nodes Involved**: Create Image Prediction

**Node Details:**

- **Create Image Prediction**
  - Type: httpRequest
  - Role: POSTs a prediction request to Replicate API endpoint `/v1/predictions`.
  - Configuration:
    - URL: `https://api.replicate.com/v1/predictions`
    - Method: POST
    - Headers:
      - Authorization: Bearer token from `api_token`
      - Prefer: wait (waits for immediate completion if possible)
    - JSON Body:
      - Contains model version id and all input parameters dynamically injected.
    - Response format: JSON, with error suppression enabled (`neverError`: true).
  - Inputs: Parameters from Set Image Parameters.
  - Outputs: Prediction response including prediction ID and initial status.
  - Edge Cases:
    - API token invalid or insufficient permissions lead to auth errors.
    - Network or API downtime could cause request failures.
    - Parameter validation errors returned by API.

#### 1.4 Polling and Status Checking

- **Overview**: Implements waiting and repeated status checks on the prediction until it is either succeeded or failed.
- **Nodes Involved**: Log Request, Wait 5s, Check Status, Is Complete?, Has Failed?, Wait 10s

**Node Details:**

- **Log Request**
  - Type: code
  - Role: Logs prediction request details for monitoring.
  - Configuration: Prints timestamp, prediction ID, and model type to console.
  - Inputs: Output from Create Image Prediction.
  - Outputs: Passes data to Wait 5s.
  - Edge Cases: Logging failure won't affect workflow but could reduce traceability.

- **Wait 5s**
  - Type: wait
  - Role: Pauses workflow for 5 seconds before checking status.
  - Inputs: Log Request output.
  - Outputs: Connects to Check Status.
  - Edge Cases: Minimal; blocking delay.

- **Check Status**
  - Type: httpRequest
  - Role: GET request to check prediction status using prediction ID.
  - Configuration:
    - URL: Dynamic URL using prediction ID from Create Image Prediction.
    - Headers: Authorization with API token.
    - Response: JSON with `neverError` true.
  - Inputs: Wait 5s output or Wait 10s output.
  - Outputs: Passes status to Is Complete? node.
  - Edge Cases:
    - API rate limiting or network issues.
    - Prediction ID invalid or expired.

- **Is Complete?**
  - Type: if
  - Role: Branches workflow based on whether prediction status equals "succeeded".
  - Inputs: Check Status output.
  - Outputs:
    - True branch to Success Response.
    - False branch to Has Failed? node.

- **Has Failed?**
  - Type: if
  - Role: Checks if prediction status equals "failed".
  - Inputs: Is Complete? false branch.
  - Outputs:
    - True branch to Error Response.
    - False branch to Wait 10s node (retry polling).

- **Wait 10s**
  - Type: wait
  - Role: Waits 10 seconds before retrying status check to reduce API calls.
  - Inputs: Has Failed? false branch.
  - Outputs: Connects back to Check Status node for polling loop.
  - Edge Cases: Infinite retry loop if status never reaches succeeded or failed; consider max retries.

#### 1.5 Success and Failure Handling

- **Overview**: Structures the final response JSON depending on success or failure, to be forwarded to the end user or system.
- **Nodes Involved**: Success Response, Error Response

**Node Details:**

- **Success Response**
  - Type: set
  - Role: Creates a JSON object indicating success, containing:
    - `success`: true
    - `image_url`: URL of generated image from `output`
    - `prediction_id`, `status`, and a success message.
  - Inputs: Is Complete? true branch.
  - Outputs: Passes to Display Result.

- **Error Response**
  - Type: set
  - Role: Creates a JSON object indicating failure, containing:
    - `success`: false
    - `error`: error message or fallback text
    - `prediction_id`, `status`, and failure message.
  - Inputs: Has Failed? true branch.
  - Outputs: Passes to Display Result.

#### 1.6 Logging and Result Display

- **Overview**: Finalizes the workflow by displaying the structured result for further consumption or UI feedback.
- **Nodes Involved**: Display Result

**Node Details:**

- **Display Result**
  - Type: set
  - Role: Assigns the final response object (success or error) to a variable `final_result`.
  - Inputs: From Success Response or Error Response.
  - Outputs: End of workflow.
  - Edge Cases: None.

#### 1.7 Documentation and User Guidance

- **Overview**: Provides detailed user-facing instructions, parameter explanations, troubleshooting, and contact info via sticky notes.
- **Nodes Involved**: Sticky Note4, Sticky Note9

**Node Details:**

- **Sticky Note4**
  - Type: stickyNote
  - Role: Extensive documentation about the workflow, model details, parameters, usage, and troubleshooting.
  - Position: Left side for easy reference.
  - Content: Includes links to YouTube, LinkedIn, Replicate docs, and step-by-step usage notes.

- **Sticky Note9**
  - Type: stickyNote
  - Role: Branding and contact information for support.
  - Content: Contact email and links to video tutorials and professional networking.

---

### 3. Summary Table

| Node Name           | Node Type          | Functional Role                            | Input Node(s)                | Output Node(s)               | Sticky Note                                                                                      |
|---------------------|--------------------|--------------------------------------------|-----------------------------|-----------------------------|------------------------------------------------------------------------------------------------|
| Manual Trigger      | manualTrigger      | Starts workflow execution                  | None                        | Set API Token               |                                                                                                |
| Set API Token       | set                | Stores API token for authentication       | Manual Trigger              | Set Image Parameters        |                                                                                                |
| Set Image Parameters| set                | Defines image generation parameters        | Set API Token               | Create Image Prediction     |                                                                                                |
| Create Image Prediction | httpRequest      | Sends image generation request to Replicate API | Set Image Parameters         | Log Request                 |                                                                                                |
| Log Request         | code               | Logs prediction details for monitoring     | Create Image Prediction     | Wait 5s                    |                                                                                                |
| Wait 5s             | wait               | Pauses 5 seconds before status check       | Log Request                 | Check Status                |                                                                                                |
| Check Status        | httpRequest        | Checks current prediction status            | Wait 5s, Wait 10s           | Is Complete?                |                                                                                                |
| Is Complete?        | if                 | Branches if prediction succeeded            | Check Status                | Success Response, Has Failed? |                                                                                                |
| Has Failed?         | if                 | Branches if prediction failed               | Is Complete? (false branch) | Error Response, Wait 10s    |                                                                                                |
| Wait 10s            | wait               | Pauses 10 seconds before retrying status check | Has Failed? (false branch)  | Check Status                |                                                                                                |
| Success Response    | set                | Formats successful prediction response      | Is Complete? (true branch)  | Display Result              |                                                                                                |
| Error Response      | set                | Formats failed prediction response          | Has Failed? (true branch)   | Display Result              |                                                                                                |
| Display Result      | set                | Prepares final result payload                | Success Response, Error Response | None                    |                                                                                                |
| Sticky Note4        | stickyNote         | Documentation and parameter reference       | None                        | None                       | See detailed workflow and parameter guide with links to YouTube, LinkedIn, documentation       |
| Sticky Note9        | stickyNote         | Contact and branding information             | None                        | None                       | Contact: Yaron@nofluff.online; YouTube and LinkedIn links                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger**
   - Add a **Manual Trigger** node.
   - No configuration needed.
   - This node starts the workflow manually.

2. **Set API Token**
   - Add a **Set** node.
   - Connect Manual Trigger → Set API Token.
   - Configure one string field named `api_token`.
   - Set value to `"YOUR_REPLICATE_API_TOKEN"` (replace with actual token).

3. **Set Image Parameters**
   - Add a **Set** node.
   - Connect Set API Token → Set Image Parameters.
   - Assign variables:
     - `api_token`: copy from previous node expression `={{ $('Set API Token').item.json.api_token }}`
     - `seed`: 0 (number)
     - `prompt`: "A beautiful landscape with mountains and a lake at sunset" (string)
     - `go_fast`: true (boolean)
     - `megapixels`: "1" (string)
     - `num_outputs`: 1 (number)
     - `aspect_ratio`: "2:1" (string)
     - `output_format`: "png" (string)
     - `output_quality`: 80 (number)
     - `num_inference_steps`: 4 (number)
     - `disable_safety_checker`: false (boolean)

4. **Create Image Prediction HTTP Request**
   - Add an **HTTP Request** node.
   - Connect Set Image Parameters → Create Image Prediction.
   - Method: POST
   - URL: `https://api.replicate.com/v1/predictions`
   - Headers:
     - `Authorization`: `Bearer {{ $json.api_token }}`
     - `Prefer`: `wait`
   - Body Content Type: JSON
   - JSON Body (use expressions to inject variables):
     ```
     {
       "version": "fire/flux:da7b098b70c3e52d56a04c2b7f3632eb63dfb0f26941a0934bf223157a7c5245",
       "input": {
         "seed": {{ $json.seed }},
         "prompt": "{{ $json.prompt }}",
         "go_fast": {{ $json.go_fast }},
         "megapixels": "{{ $json.megapixels }}",
         "num_outputs": {{ $json.num_outputs }},
         "aspect_ratio": "{{ $json.aspect_ratio }}",
         "output_format": "{{ $json.output_format }}",
         "output_quality": {{ $json.output_quality }},
         "num_inference_steps": {{ $json.num_inference_steps }},
         "disable_safety_checker": {{ $json.disable_safety_checker }}
       }
     }
     ```
   - Response Format: JSON
   - Enable `neverError` to avoid workflow failure on HTTP errors.

5. **Add Code Node for Logging**
   - Add a **Code** node.
   - Connect Create Image Prediction → Log Request.
   - Paste JS code to log timestamp, prediction ID, and model type.
   - Return input data unmodified.

6. **Add Wait 5 Seconds**
   - Add a **Wait** node.
   - Connect Log Request → Wait 5s.
   - Configure to wait 5 seconds.

7. **Add Check Status HTTP Request**
   - Add an **HTTP Request** node.
   - Connect Wait 5s → Check Status.
   - Method: GET
   - URL: `https://api.replicate.com/v1/predictions/{{ $('Create Image Prediction').item.json.id }}`
   - Headers:
     - `Authorization`: Bearer token from `Set API Token` node.
   - Response Format: JSON
   - Enable `neverError`.

8. **Add If Node "Is Complete?"**
   - Add an **If** node.
   - Connect Check Status → Is Complete?
   - Condition: Check if `status` field equals `"succeeded"`.

9. **Add Success Response Set Node**
   - Add a **Set** node.
   - Connect Is Complete? (true branch) → Success Response.
   - Assign variable `response` with an object containing:
     - `success: true`
     - `image_url`: from JSON output field `output`
     - `prediction_id`, `status`, and success message.

10. **Add If Node "Has Failed?"**
    - Add an **If** node.
    - Connect Is Complete? (false branch) → Has Failed?
    - Condition: Check if `status` equals `"failed"`.

11. **Add Error Response Set Node**
    - Add a **Set** node.
    - Connect Has Failed? (true branch) → Error Response.
    - Assign variable `response` with an object containing:
      - `success: false`
      - `error`: from JSON `error` field or fallback text
      - `prediction_id`, `status`, failure message.

12. **Add Wait 10 Seconds Node**
    - Add a **Wait** node.
    - Connect Has Failed? (false branch) → Wait 10s.
    - Configure to wait 10 seconds.

13. **Loop Back from Wait 10s to Check Status**
    - Connect Wait 10s → Check Status to retry polling.

14. **Add Display Result Set Node**
    - Add a **Set** node.
    - Connect both Success Response and Error Response → Display Result.
    - Assign variable `final_result` to the JSON from `response`.

15. **Create Sticky Notes**
    - Add two **Sticky Note** nodes.
    - Place Sticky Note4 with detailed documentation and usage instructions.
    - Place Sticky Note9 with contact info and branding.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                          | Context or Link                                                                                     |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Workflow powered by Replicate API and n8n automation integrating Fire Flux image generation model.                                                                                                                                                                                                                                                                                   | Model docs: https://replicate.com/fire/flux                                                       |
| For support or inquiries, contact: Yaron@nofluff.online. Explore tutorials and updates on YouTube: https://www.youtube.com/@YaronBeen/videos and LinkedIn: https://www.linkedin.com/in/yaronbeen/.                                                                                                                                                                                    | Contact and video tutorials                                                                        |
| Key troubleshooting tips include verifying API token validity, parameter correctness, and monitoring API usage to avoid timeouts or rate limits.                                                                                                                                                                                                                                     | Troubleshooting guidance                                                                           |
| The workflow uses intelligent polling with 5s and 10s waits to optimize API usage and prevent excessive calls during image generation.                                                                                                                                                                                                                                              | Performance optimization                                                                           |
| Replace `"YOUR_REPLICATE_API_TOKEN"` with a valid token obtained from https://replicate.com/account. Keep tokens secure.                                                                                                                                                                                                                                                               | API token acquisition                                                                              |
| Model version and endpoint are fixed to Fire Flux version id `da7b098b70c3e52d56a04c2b7f3632eb63dfb0f26941a0934bf223157a7c5245` ensuring consistent model behavior.                                                                                                                                                                                                                   | Model version locking                                                                              |

---

**Disclaimer:**  
This document is a detailed technical reference for an n8n workflow automating AI image generation using the Fire Flux model via the Replicate API. All configurations comply with current content policies and use only legal and publicly available data.