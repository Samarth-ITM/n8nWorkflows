Create Images from Text Prompts using Lumi and Replicate

https://n8nworkflows.xyz/workflows/create-images-from-text-prompts-using-lumi-and-replicate-6808


# Create Images from Text Prompts using Lumi and Replicate

### 1. Workflow Overview

This workflow automates the process of generating AI-created images from text prompts using the "adamantiamable/lumi" model hosted on Replicate. It is designed for users who want to convert descriptive prompts into images with configurable parameters, handling the entire lifecycle from input reception, API request submission, polling for completion, to delivering the final result or error messages.

**Logical Blocks:**

- **1.1 Input Reception and Initialization:** Accept manual trigger and set authentication and generation parameters.
- **1.2 Request Submission:** Create a prediction request on Replicate’s API using the configured parameters.
- **1.3 Status Polling Loop:** Wait and repeatedly check the prediction’s status until completed or failed.
- **1.4 Result Handling and Output:** Route successful results or failures into structured JSON responses.
- **1.5 Logging and Monitoring:** Log request metadata for debugging and operational monitoring.
- **1.6 Documentation and Notes:** Embedded sticky notes provide detailed user instructions, parameter explanations, and contact/support info.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Initialization

- **Overview:** This block starts the workflow and prepares all necessary parameters, including the API token and generation parameters, to be sent to the Replicate API.
- **Nodes Involved:** Manual Trigger, Set API Token, Set Other Parameters

**Nodes:**

- **Manual Trigger**
  - Type: Manual trigger node
  - Role: Initiates the workflow execution manually.
  - Config: No parameters; simply starts the workflow on user action.
  - Connections: Output → Set API Token
  - Edge Cases: None inherent; execution requires manual start.

- **Set API Token**
  - Type: Set node
  - Role: Sets the Replicate API token as a workflow variable.
  - Config: Hardcoded string `YOUR_REPLICATE_API_TOKEN` (must be replaced with a valid token).
  - Connections: Input ← Manual Trigger; Output → Set Other Parameters
  - Edge Cases: Failure if token is missing or invalid; requires secure handling.

- **Set Other Parameters**
  - Type: Set node
  - Role: Defines all parameters for the image generation model, including prompt, image size, steps, and more.
  - Config: 
    - Copies `api_token` from previous node.
    - Sets defaults for `mask`, `seed`, `image`, `model`, `width`, `height`, `prompt`, etc.
    - Includes booleans (`go_fast`, `disable_safety_checker`), numbers (`seed`, `width`, `height`, `guidance_scale`), and strings.
  - Connections: Input ← Set API Token; Output → Create Other Prediction
  - Edge Cases: Parameter validation failure if types/values are incorrect; prompt is required.

#### 2.2 Request Submission

- **Overview:** Sends a POST request to Replicate’s API to create a new prediction job using the configured parameters.
- **Nodes Involved:** Create Other Prediction

**Node:**

- **Create Other Prediction**
  - Type: HTTP Request
  - Role: Calls Replicate API’s prediction creation endpoint to start image generation.
  - Config:
    - URL: `https://api.replicate.com/v1/predictions`
    - Method: POST
    - Headers: Includes `Authorization` with bearer token from `api_token`.
    - JSON Body: Constructs input object with parameters dynamically from previous node’s data.
    - Response handling: Parses JSON response.
    - Uses `Prefer: wait` header to wait for immediate response if possible.
  - Connections: Input ← Set Other Parameters; Output → Log Request
  - Edge Cases:
    - API errors (authentication, rate limiting, validation).
    - Network timeouts.
    - Invalid parameter formats.
  - Requirements: Valid Replicate API token; network access.

#### 2.3 Status Polling Loop

- **Overview:** Implements a loop that waits and checks the status of the prediction until it is either succeeded or failed.
- **Nodes Involved:** Log Request, Wait 5s, Check Status, Is Complete?, Has Failed?, Wait 10s

**Nodes:**

- **Log Request**
  - Type: Code node
  - Role: Logs metadata about the prediction request for monitoring.
  - Config: Outputs timestamp, prediction ID, and model type to console.
  - Connections: Input ← Create Other Prediction; Output → Wait 5s
  - Edge Cases: Logging failure does not impact workflow.

- **Wait 5s**
  - Type: Wait node
  - Role: Pauses workflow for 5 seconds before checking prediction status.
  - Config: Wait time = 5 seconds.
  - Connections: Input ← Log Request; Output → Check Status
  - Edge Cases: None significant.

- **Check Status**
  - Type: HTTP Request
  - Role: Queries Replicate API for current prediction status.
  - Config:
    - URL dynamically constructed with prediction ID.
    - Header includes Authorization with API token.
    - Parses JSON response.
  - Connections: Input ← Wait 5s or Wait 10s; Output → Is Complete?
  - Edge Cases: API call failures, network issues.

- **Is Complete?**
  - Type: IF node
  - Role: Checks if prediction status is `succeeded`.
  - Config: Condition: `$json.status == 'succeeded'`
  - Connections: 
    - True → Success Response
    - False → Has Failed?
  - Edge Cases: Status field missing or unexpected values.

- **Has Failed?**
  - Type: IF node
  - Role: Checks if prediction status is `failed`.
  - Config: Condition: `$json.status == 'failed'`
  - Connections:
    - True → Error Response
    - False → Wait 10s (retry loop)
  - Edge Cases: Other statuses (e.g., processing) lead to wait and retry.

- **Wait 10s**
  - Type: Wait node
  - Role: Pauses 10 seconds before next status check for predictions still processing.
  - Config: Wait time = 10 seconds.
  - Connections: Input ← Has Failed? (False branch); Output → Check Status
  - Edge Cases: None significant.

#### 2.4 Result Handling and Output

- **Overview:** Constructs structured success or error responses and prepares the final output object.
- **Nodes Involved:** Success Response, Error Response, Display Result

**Nodes:**

- **Success Response**
  - Type: Set node
  - Role: Creates a JSON object signaling success with prediction details and output URL.
  - Config: Sets fields `success: true`, `result_url`, `prediction_id`, `status`, and a success message.
  - Connections: Input ← Is Complete? (True); Output → Display Result
  - Edge Cases: Output URL missing or malformed.

- **Error Response**
  - Type: Set node
  - Role: Creates a JSON object signaling failure with error details.
  - Config: Sets fields `success: false`, `error` message, `prediction_id`, `status`, and failure message.
  - Connections: Input ← Has Failed? (True); Output → Display Result
  - Edge Cases: Error message missing.

- **Display Result**
  - Type: Set node
  - Role: Prepares final output object named `final_result` for downstream consumption or response.
  - Config: Sets `final_result` field with the response object from success or error.
  - Connections: Input ← Success Response or Error Response
  - Edge Cases: None.

#### 2.5 Logging and Monitoring

- **Overview:** Logs details about each prediction request for monitoring and debugging purposes.
- **Nodes Involved:** Log Request (already described in 2.3)

#### 2.6 Documentation and Notes

- **Overview:** Provides extensive notes on usage, parameters, troubleshooting, and support contacts in sticky notes.
- **Nodes Involved:** Sticky Note9, Sticky Note4

**Sticky Note9**
- Content includes contact email and links to YouTube and LinkedIn for support.

**Sticky Note4**
- Content includes:
  - Model overview and metadata
  - Parameter reference with explanations
  - Workflow component explanations
  - Benefits of the workflow
  - Quick start instructions
  - Troubleshooting guide
  - Important external links for Replicate and n8n documentation

---

### 3. Summary Table

| Node Name             | Node Type         | Functional Role                                    | Input Node(s)              | Output Node(s)           | Sticky Note                                                                                  |
|-----------------------|-------------------|---------------------------------------------------|----------------------------|--------------------------|----------------------------------------------------------------------------------------------|
| Manual Trigger        | Manual Trigger    | Starts workflow manually                           |                            | Set API Token             |                                                                                              |
| Set API Token         | Set               | Supplies Replicate API token                       | Manual Trigger             | Set Other Parameters      |                                                                                              |
| Set Other Parameters  | Set               | Defines all model input parameters                 | Set API Token              | Create Other Prediction   |                                                                                              |
| Create Other Prediction| HTTP Request      | Sends prediction creation request to Replicate    | Set Other Parameters       | Log Request               |                                                                                              |
| Log Request           | Code              | Logs prediction request metadata                    | Create Other Prediction    | Wait 5s                   |                                                                                              |
| Wait 5s               | Wait              | Pauses 5 seconds before status check               | Log Request                | Check Status              |                                                                                              |
| Check Status          | HTTP Request      | Queries prediction status                            | Wait 5s, Wait 10s          | Is Complete?              |                                                                                              |
| Is Complete?          | IF                | Checks if prediction succeeded                      | Check Status               | Success Response, Has Failed? |                                                                                              |
| Has Failed?           | IF                | Checks if prediction failed                         | Is Complete?               | Error Response, Wait 10s  |                                                                                              |
| Wait 10s              | Wait              | Pauses 10 seconds before retry status check        | Has Failed?                | Check Status              |                                                                                              |
| Success Response      | Set               | Creates success response object                      | Is Complete?               | Display Result            |                                                                                              |
| Error Response        | Set               | Creates error response object                        | Has Failed?                | Display Result            |                                                                                              |
| Display Result        | Set               | Prepares final output object                         | Success Response, Error Response |                          |                                                                                              |
| Sticky Note9          | Sticky Note       | Support contact & social links                       |                            |                          | Contact: Yaron@nofluff.online; YouTube & LinkedIn links                                    |
| Sticky Note4          | Sticky Note       | Extensive workflow & model documentation            |                            |                          | Detailed instructions, parameter guide, troubleshooting, and links to docs and model page  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node**
   - Name: `Manual Trigger`
   - No parameters needed.
   - This node will start the workflow manually.

2. **Add a Set node to store API token**
   - Name: `Set API Token`
   - Add a string field named `api_token`.
   - Set value to your Replicate API token (replace `"YOUR_REPLICATE_API_TOKEN"`).
   - Connect output of `Manual Trigger` to this node.

3. **Add a Set node to configure generation parameters**
   - Name: `Set Other Parameters`
   - Add the following fields with types and default values:
     - `api_token` (string): Expression referencing `Set API Token` node’s `api_token`.
     - `mask` (string): `"https://via.placeholder.com/512x512/000000/FFFFFF.png"`
     - `seed` (number): `-1`
     - `image` (string): `"https://picsum.photos/512/512"`
     - `model` (string): `"dev"`
     - `width` (number): `512`
     - `height` (number): `512`
     - `prompt` (string): `"Create something amazing"` (required)
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
   - Connect output of `Set API Token` to this node.

4. **Add an HTTP Request node to create prediction**
   - Name: `Create Other Prediction`
   - HTTP Method: POST
   - URL: `https://api.replicate.com/v1/predictions`
   - Headers:
     - `Authorization`: `Bearer {{$json["api_token"]}}`
     - `Prefer`: `wait`
   - Body Type: JSON
   - Body JSON (use expressions to map inputs):
     ```json
     {
       "version": "adamantiamable/lumi:33ec1160f84c9657c8238cabc9dc8ed6bb335eb3168cdedf02df3850f8f37239",
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
   - Connect output of `Set Other Parameters` to this node.

5. **Add a Code node to log request info**
   - Name: `Log Request`
   - JavaScript code:
     ```javascript
     const data = $input.all()[0].json;
     console.log('adamantiamable/lumi Request:', {
       timestamp: new Date().toISOString(),
       prediction_id: data.id,
       model_type: 'other'
     });
     return $input.all();
     ```
   - Connect output of `Create Other Prediction` to this node.

6. **Add a Wait node for 5 seconds**
   - Name: `Wait 5s`
   - Wait for 5 seconds.
   - Connect output of `Log Request` to this node.

7. **Add HTTP Request node to check prediction status**
   - Name: `Check Status`
   - HTTP Method: GET
   - URL: `https://api.replicate.com/v1/predictions/{{$json["id"]}}`
   - Headers:
     - `Authorization`: `Bearer {{$json.api_token}}` (or from `Set API Token` node)
   - Connect output of `Wait 5s` to this node.
   - Also, this node will be connected later in the retry loop.

8. **Add IF node to check for completion**
   - Name: `Is Complete?`
   - Condition: `$json.status == "succeeded"`
   - True output: success path
   - False output: check failure
   - Connect output of `Check Status` to this node.

9. **Add IF node to check for failure**
   - Name: `Has Failed?`
   - Condition: `$json.status == "failed"`
   - True output: failure handling
   - False output: wait and retry
   - Connect “False” output of `Is Complete?` to this node.

10. **Add Wait node for 10 seconds for retry**
    - Name: `Wait 10s`
    - Wait time: 10 seconds
    - Connect “False” output of `Has Failed?` to this node.
    - Connect output of `Wait 10s` back to `Check Status` node to form loop.

11. **Add Set node for success response**
    - Name: `Success Response`
    - Set JSON object:
      ```json
      {
        "success": true,
        "result_url": $json.output,
        "prediction_id": $json.id,
        "status": $json.status,
        "message": "Other generated successfully"
      }
      ```
    - Connect “True” output of `Is Complete?` to this node.

12. **Add Set node for error response**
    - Name: `Error Response`
    - Set JSON object:
      ```json
      {
        "success": false,
        "error": $json.error || "Other generation failed",
        "prediction_id": $json.id,
        "status": $json.status,
        "message": "Failed to generate other"
      }
      ```
    - Connect “True” output of `Has Failed?` to this node.

13. **Add final Set node to display result**
    - Name: `Display Result`
    - Set field `final_result` equal to response JSON from either success or error node.
    - Connect outputs of `Success Response` and `Error Response` to this node.

14. **Add Sticky Note nodes for documentation**
    - Add two Sticky Notes:
      - One with contact & social media info.
      - Another with detailed workflow description, parameters, troubleshooting, and links.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | Context or Link                                                                                   |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| **LUMI GENERATOR**<br>For any questions or support, please contact:<br>Yaron@nofluff.online<br>Explore more tips and tutorials here:<br>- YouTube: https://www.youtube.com/@YaronBeen/videos<br>- LinkedIn: https://www.linkedin.com/in/yaronbeen/                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               | Sticky Note with support contact and social media links                                          |
| **ADAMANTIAMABLE/LUMI - OTHER GENERATION WORKFLOW**<br>Powered by Replicate API and n8n Automation.<br>Includes detailed model overview, parameter reference, workflow explanation, benefits, quick start instructions, troubleshooting guide, and external documentation links.<br>Model Docs: https://replicate.com/adamantiamable/lumi<br>Replicate API Docs: https://replicate.com/docs<br>n8n Docs: https://docs.n8n.io                                                                                                                                                                                                                                                                                                                                                                                                                                  | Sticky Note with extended workflow and parameter documentation                                  |
| Replace `"YOUR_REPLICATE_API_TOKEN"` in the Set API Token node with your actual Replicate API token. Keep this token secure and never commit it to any public repository.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | Essential security note                                                                        |
| The workflow is designed to automatically poll the Replicate API for up to several minutes. Long-running jobs may require adjusting wait times or max retries.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | Operational consideration                                                                      |
| The `prompt` parameter is required and defines the text input guiding image generation. Adjust this prompt based on desired output.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               | Key input parameter                                                                            |
| The workflow supports advanced parameters like `mask` for inpainting, `go_fast` for speed-optimized generation, and `disable_safety_checker` to disable content filtering. Use with caution.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | Advanced parameter usage notes                                                                |

---

This reference document comprehensively details the entire workflow structure, logic, and configuration, enabling users or AI agents to understand, reproduce, and customize the image generation automation end-to-end.