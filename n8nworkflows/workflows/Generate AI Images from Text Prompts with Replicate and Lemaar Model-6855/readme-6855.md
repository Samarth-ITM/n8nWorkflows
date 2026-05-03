Generate AI Images from Text Prompts with Replicate and Lemaar Model

https://n8nworkflows.xyz/workflows/generate-ai-images-from-text-prompts-with-replicate-and-lemaar-model-6855


# Generate AI Images from Text Prompts with Replicate and Lemaar Model

### 1. Workflow Overview

This workflow automates the generation of AI images from text prompts using the Replicate API and the "creativeathive/lemaar-doorhandle-newset" model. It is designed for use cases requiring AI-driven image synthesis or modification based on textual input, including optional parameters for image inpainting, style adjustment, and output customization.

The workflow is logically divided into the following blocks:

- **1.1 Input Initialization:** Receives a manual trigger and sets the necessary API token and model parameters.
- **1.2 Prediction Request:** Submits a generation request to the Replicate API with configured parameters.
- **1.3 Status Polling Loop:** Waits and polls the API for prediction status until completion or failure.
- **1.4 Result Handling:** Processes successful or failed prediction responses and formats output.
- **1.5 Logging:** Logs request details for monitoring and debugging.
- **1.6 User Guidance & Documentation:** Provides embedded notes for users with instructions, troubleshooting, and resource links.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Initialization

**Overview:**  
This block starts the workflow on manual trigger and configures authentication along with all input parameters required for the AI generation model.

**Nodes Involved:**  
- Manual Trigger  
- Set API Token  
- Set Other Parameters

**Node Details:**

- **Manual Trigger**  
  - Type: manualTrigger  
  - Role: Initiates the workflow manually by user action.  
  - Configuration: Default with no parameters; simply triggers workflow execution.  
  - Inputs: None  
  - Outputs: Connects to Set API Token node.  
  - Edge cases: None, but requires user to start the workflow manually.

- **Set API Token**  
  - Type: set  
  - Role: Defines the Replicate API token used for authentication in subsequent calls.  
  - Configuration: A string variable `api_token` set to a placeholder `"YOUR_REPLICATE_API_TOKEN"` which must be replaced with the user’s actual token.  
  - Inputs: Receives trigger from Manual Trigger node.  
  - Outputs: Passes token to Set Other Parameters node.  
  - Edge cases: Missing or invalid API token results in authentication errors downstream.

- **Set Other Parameters**  
  - Type: set  
  - Role: Aggregates all input parameters for the model including the API token and model-specific options.  
  - Configuration:  
    - Copies API token from previous node dynamically.  
    - Sets parameters such as `mask`, `seed`, `image`, `model`, `width`, `height`, `prompt`, `go_fast`, `extra_lora`, `lora_scale`, `megapixels`, `num_outputs`, `aspect_ratio`, `output_format`, `guidance_scale`, `output_quality`, `prompt_strength`, `extra_lora_scale`, `replicate_weights`, `num_inference_steps`, `disable_safety_checker`.  
    - Defaults include a placeholder image URL and prompt text `"Create something amazing"`.  
  - Inputs: Takes API token from Set API Token node.  
  - Outputs: Sends parameterized data to Create Other Prediction node.  
  - Edge cases: Improper parameter types or missing required parameters (e.g., prompt) may cause API errors.

---

#### 2.2 Prediction Request

**Overview:**  
This block sends a POST request to the Replicate API to start the AI image generation based on the parameters set in the previous block.

**Nodes Involved:**  
- Create Other Prediction

**Node Details:**

- **Create Other Prediction**  
  - Type: httpRequest  
  - Role: Sends generation request to Replicate API and waits for initial response synchronously.  
  - Configuration:  
    - URL: `https://api.replicate.com/v1/predictions`  
    - Method: POST  
    - Headers: Authorization Bearer token from `api_token`, and `Prefer: wait` to wait for immediate processing if possible.  
    - Body: JSON constructed dynamically using parameters from previous node.  
    - Response handling: Parses JSON response; configured to never error on HTTP errors to allow graceful handling downstream.  
  - Inputs: Receives input parameters from Set Other Parameters node.  
  - Outputs: Forwards prediction response containing prediction id and initial status to Log Request node.  
  - Edge cases: Network errors, invalid token, malformed request body, or API limits could cause failures.

---

#### 2.3 Status Polling Loop

**Overview:**  
This block implements a polling mechanism with delays to check the status of the prediction until it is either completed successfully or failed.

**Nodes Involved:**  
- Log Request  
- Wait 5s  
- Check Status  
- Is Complete?  
- Has Failed?  
- Wait 10s

**Node Details:**

- **Log Request**  
  - Type: code  
  - Role: Logs the prediction request details such as timestamp, prediction id, and model type for monitoring.  
  - Configuration: JavaScript code logs to console and returns input data unchanged.  
  - Inputs: Receives response from Create Other Prediction node.  
  - Outputs: Connects to Wait 5s node.  
  - Edge cases: Console logging failure is unlikely but would not break workflow.

- **Wait 5s**  
  - Type: wait  
  - Role: Pauses execution for 5 seconds to allow prediction progress before status check.  
  - Configuration: Wait duration set to 5 seconds.  
  - Inputs: Triggered from Log Request node.  
  - Outputs: Connects to Check Status node.  
  - Edge cases: Delay too short may lead to frequent polling; too long delays affect responsiveness.

- **Check Status**  
  - Type: httpRequest  
  - Role: Queries Replicate API for current status of the prediction using prediction id.  
  - Configuration:  
    - URL uses dynamic interpolation to insert prediction id from Create Other Prediction node.  
    - Method: GET (default).  
    - Header Authorization uses API token from Set API Token node.  
    - Response parsed as JSON; configured to never error for robustness.  
  - Inputs: Triggered from Wait 5s and Wait 10s nodes.  
  - Outputs: Connects to Is Complete? node.  
  - Edge cases: Network or API errors, invalid prediction id.

- **Is Complete?**  
  - Type: if  
  - Role: Checks if prediction status equals `"succeeded"`.  
  - Configuration: String equality check on `$json.status == "succeeded"`.  
  - Inputs: Receives status from Check Status node.  
  - Outputs:  
    - True branch: sends to Success Response node.  
    - False branch: sends to Has Failed? node.  
  - Edge cases: Unexpected status strings or missing field can cause logic errors.

- **Has Failed?**  
  - Type: if  
  - Role: Checks if prediction status equals `"failed"` to detect failure.  
  - Configuration: String equality check on `$json.status == "failed"`.  
  - Inputs: From false branch of Is Complete? node.  
  - Outputs:  
    - True branch: sends to Error Response node.  
    - False branch: sends to Wait 10s node for retry.  
  - Edge cases: Other intermediate statuses may cause indefinite polling.

- **Wait 10s**  
  - Type: wait  
  - Role: Delays for 10 seconds before rechecking the status to reduce API request frequency on pending predictions.  
  - Configuration: Wait duration set to 10 seconds.  
  - Inputs: From false branch of Has Failed? node.  
  - Outputs: Connects back to Check Status node, closing the polling loop.  
  - Edge cases: Long wait times increase total completion time; short wait times increase API calls.

---

#### 2.4 Result Handling

**Overview:**  
Handles the final output of the workflow by formatting success or error responses and preparing the data for display or further use.

**Nodes Involved:**  
- Success Response  
- Error Response  
- Display Result

**Node Details:**

- **Success Response**  
  - Type: set  
  - Role: Creates a structured success response object with prediction details and result URL.  
  - Configuration: Sets a JSON object `response` containing:  
    - `success: true`  
    - `result_url` (output image URL)  
    - `prediction_id`  
    - `status`  
    - `message` stating success.  
  - Inputs: From true branch of Is Complete? node.  
  - Outputs: Connects to Display Result node.  
  - Edge cases: Missing output data could cause incomplete response.

- **Error Response**  
  - Type: set  
  - Role: Creates a structured error response object with failure details.  
  - Configuration: Sets JSON object `response` containing:  
    - `success: false`  
    - `error` message from API or default message  
    - `prediction_id`  
    - `status`  
    - `message` stating failure.  
  - Inputs: From true branch of Has Failed? node.  
  - Outputs: Connects to Display Result node.  
  - Edge cases: Lack of error details from API may reduce clarity.

- **Display Result**  
  - Type: set  
  - Role: Final node to output the constructed response object for consumption by downstream processes or UI.  
  - Configuration: Sets `final_result` to the `response` object from Success or Error Response node.  
  - Inputs: From both Success Response and Error Response nodes.  
  - Outputs: Workflow end node effectively.  
  - Edge cases: None significant.

---

#### 2.5 Logging

**Overview:**  
Provides basic monitoring capabilities by logging details of each prediction request.

**Nodes Involved:**  
- Log Request (also part of polling loop)

**Node Details:**  
See above in 2.3 Status Polling Loop.

---

#### 2.6 User Guidance & Documentation

**Overview:**  
Two sticky note nodes provide embedded documentation, contact information, parameter explanations, troubleshooting, and quick start instructions directly in the workflow interface.

**Nodes Involved:**  
- Sticky Note9  
- Sticky Note4

**Node Details:**

- **Sticky Note9**  
  - Type: stickyNote  
  - Role: Displays model branding, contact email, and links to tutorial videos and LinkedIn.  
  - Content includes:  
    - Title banner for model  
    - Contact: Yaron@nofluff.online  
    - YouTube and LinkedIn links

- **Sticky Note4**  
  - Type: stickyNote  
  - Role: Extensive detailed documentation with model overview, parameter list, workflow explanation, quick start guide, troubleshooting, and resource links.  
  - Content includes:  
    - Model and API info  
    - Parameter descriptions  
    - Workflow component explanations  
    - Benefits and instructions  
    - Troubleshooting tips  
    - External links (Replicate model page, API docs, n8n docs)

**Edge cases:** None applicable; these are informational.

---

### 3. Summary Table

| Node Name             | Node Type      | Functional Role                                  | Input Node(s)         | Output Node(s)          | Sticky Note                                                                                         |
|-----------------------|----------------|-------------------------------------------------|-----------------------|-------------------------|--------------------------------------------------------------------------------------------------|
| Manual Trigger        | manualTrigger  | Starts the workflow manually                     | None                  | Set API Token            |                                                                                                  |
| Set API Token         | set            | Sets Replicate API authentication token         | Manual Trigger        | Set Other Parameters     |                                                                                                  |
| Set Other Parameters  | set            | Defines all model input parameters                | Set API Token         | Create Other Prediction  |                                                                                                  |
| Create Other Prediction | httpRequest   | Sends generation request to Replicate API        | Set Other Parameters  | Log Request              |                                                                                                  |
| Log Request           | code           | Logs prediction request details                   | Create Other Prediction | Wait 5s                  |                                                                                                  |
| Wait 5s               | wait           | Delay before checking prediction status           | Log Request           | Check Status             |                                                                                                  |
| Check Status          | httpRequest    | Polls API for prediction status                    | Wait 5s, Wait 10s     | Is Complete?             |                                                                                                  |
| Is Complete?          | if             | Checks if prediction succeeded                     | Check Status          | Success Response, Has Failed? |                                                                                                  |
| Has Failed?           | if             | Checks if prediction failed                         | Is Complete?          | Error Response, Wait 10s |                                                                                                  |
| Wait 10s              | wait           | Delay before retrying status check                  | Has Failed?           | Check Status             |                                                                                                  |
| Success Response      | set            | Formats success output response                     | Is Complete?          | Display Result           |                                                                                                  |
| Error Response        | set            | Formats error output response                       | Has Failed?            | Display Result           |                                                                                                  |
| Display Result        | set            | Outputs final response object                        | Success Response, Error Response | None             |                                                                                                  |
| Sticky Note9          | stickyNote     | Model branding and contact info                     | None                  | None                    | Contact: Yaron@nofluff.online; YouTube & LinkedIn links                                         |
| Sticky Note4          | stickyNote     | Detailed workflow documentation and instructions   | None                  | None                    | Extensive parameter guide, troubleshooting, quick start, external links                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger**  
   - Node Type: Manual Trigger  
   - Purpose: To start workflow execution manually.  
   - No parameters required.

2. **Create Set API Token Node**  
   - Node Type: Set  
   - Purpose: Store your Replicate API token.  
   - Add a string field named `api_token` with value `"YOUR_REPLICATE_API_TOKEN"` (replace with actual token).  
   - Connect Manual Trigger output to this node's input.

3. **Create Set Other Parameters Node**  
   - Node Type: Set  
   - Purpose: Define all the input parameters for the model request.  
   - Add the following fields with these types and default values:  
     - `api_token` (string): Expression `={{ $('Set API Token').item.json.api_token }}` to copy token.  
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
   - Connect Set API Token node output here.

4. **Create Create Other Prediction Node**  
   - Node Type: HTTP Request  
   - Purpose: POST request to start AI image generation on Replicate API.  
   - Settings:  
     - URL: `https://api.replicate.com/v1/predictions`  
     - Method: POST  
     - Headers:  
       - `Authorization`: `Bearer {{ $json.api_token }}`  
       - `Prefer`: `wait`  
     - Body: JSON type with the following structure (use expressions to pull from incoming JSON):  
       ```json
       {
         "version": "creativeathive/lemaar-doorhandle-newset:1f9fbbe605ef7c2c91e1eb396413073e1166575e55bc7daecfe7706b0180f10c",
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
     - Configure to never error on HTTP errors for graceful handling.  
   - Connect Set Other Parameters output here.

5. **Create Log Request Node**  
   - Node Type: Code  
   - Purpose: Log prediction request details for monitoring.  
   - Code:  
     ```js
     const data = $input.all()[0].json;
     console.log('creativeathive/lemaar-doorhandle-newset Request:', {
       timestamp: new Date().toISOString(),
       prediction_id: data.id,
       model_type: 'other'
     });
     return $input.all();
     ```  
   - Connect Create Other Prediction output here.

6. **Create Wait 5s Node**  
   - Node Type: Wait  
   - Purpose: Delay 5 seconds before checking status.  
   - Parameters: 5 seconds.  
   - Connect Log Request output here.

7. **Create Check Status Node**  
   - Node Type: HTTP Request  
   - Purpose: Poll prediction status from Replicate API.  
   - Settings:  
     - URL: `https://api.replicate.com/v1/predictions/{{ $('Create Other Prediction').item.json.id }}`  
     - Method: GET  
     - Header: `Authorization: Bearer {{ $('Set API Token').item.json.api_token }}`  
     - Response Format: JSON  
     - Never error on HTTP errors.  
   - Connect Wait 5s output here.  
   - Also connect Wait 10s node output here (see below).

8. **Create Is Complete? Node**  
   - Node Type: If  
   - Purpose: Check if prediction status equals `"succeeded"`.  
   - Condition: `$json.status == "succeeded"`  
   - Connect Check Status output here.

9. **Create Has Failed? Node**  
   - Node Type: If  
   - Purpose: Check if prediction status equals `"failed"`.  
   - Condition: `$json.status == "failed"`  
   - Connect false branch of Is Complete? node here.

10. **Create Wait 10s Node**  
    - Node Type: Wait  
    - Purpose: Delay 10 seconds before retrying status check if not failed or succeeded.  
    - Parameters: 10 seconds.  
    - Connect false branch of Has Failed? node here.  
    - Connect its output back to Check Status node to continue polling loop.

11. **Create Success Response Node**  
    - Node Type: Set  
    - Purpose: Format success JSON response.  
    - Set `response` to:  
      ```js
      {
        success: true,
        result_url: $json.output,
        prediction_id: $json.id,
        status: $json.status,
        message: 'Other generated successfully'
      }
      ```  
    - Connect true branch of Is Complete? node here.

12. **Create Error Response Node**  
    - Node Type: Set  
    - Purpose: Format failure JSON response.  
    - Set `response` to:  
      ```js
      {
        success: false,
        error: $json.error || 'Other generation failed',
        prediction_id: $json.id,
        status: $json.status,
        message: 'Failed to generate other'
      }
      ```  
    - Connect true branch of Has Failed? node here.

13. **Create Display Result Node**  
    - Node Type: Set  
    - Purpose: Output final response object.  
    - Set `final_result` to the `response` object from either success or error response.  
    - Connect Success Response and Error Response outputs here.

14. **Create Sticky Notes**  
    - Sticky Note9: Add branding, contact, and links.  
    - Sticky Note4: Add detailed documentation, parameter explanations, troubleshooting, and resource links.  
    - These are informational and do not connect to other nodes.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | Context or Link                                                                                   |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Contact for support: Yaron@nofluff.online. Explore tutorials and tips on YouTube: https://www.youtube.com/@YaronBeen/videos and LinkedIn: https://www.linkedin.com/in/yaronbeen/                                                                                                                                                                                                                                                                                                                                                     | Sticky Note9                                                                                     |
| Detailed workflow guide including parameter descriptions, usage instructions, troubleshooting guide, and quick start instructions. Also includes links to the model's Replicate page (https://replicate.com/creativeathive/lemaar-doorhandle-newset), Replicate API documentation (https://replicate.com/docs), and n8n official documentation (https://docs.n8n.io).                                                                                                                                                                                                                             | Sticky Note4                                                                                     |
| The workflow uses the Replicate API version identified by `version_id` = "1f9fbbe605ef7c2c91e1eb396413073e1166575e55bc7daecfe7706b0180f10c" and the model "creativeathive/lemaar-doorhandle-newset" which is an "Other Generator" type model.                                                                                                                                                                                                                                                                                                        | Meta section of workflow                                                                         |
| Best practice: Replace `"YOUR_REPLICATE_API_TOKEN"` with your valid API token from https://replicate.com/account. Keep tokens secure and monitor usage to avoid service interruptions.                                                                                                                                                                                                                                                                                                                                            | Parameter and troubleshooting notes                                                             |
| The workflow implements intelligent polling with increasing wait times (5s initial wait, 10s retry wait) to balance API request frequency and responsiveness.                                                                                                                                                                                                                                                                                                                                                                       | Polling logic in status checking loop                                                           |
| The model supports many optional parameters to customize image generation, including inpainting mask, seed, model selection, output quality, and prompt strength. Adjust these parameters in the "Set Other Parameters" node as needed.                                                                                                                                                                                                                                                                                             | Parameter reference in Sticky Note4                                                             |

---

**Disclaimer:**  
The provided text exclusively originates from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.