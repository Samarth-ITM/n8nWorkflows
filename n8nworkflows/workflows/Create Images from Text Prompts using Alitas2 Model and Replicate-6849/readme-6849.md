Create Images from Text Prompts using Alitas2 Model and Replicate

https://n8nworkflows.xyz/workflows/create-images-from-text-prompts-using-alitas2-model-and-replicate-6849


# Create Images from Text Prompts using Alitas2 Model and Replicate

### 1. Workflow Overview

This workflow automates the generation of images from text prompts using the Alitas2 model via the Replicate API. It is designed to:

- Receive a manual trigger to start the generation process.
- Configure and send parameters to the Alitas2 model hosted on Replicate.
- Monitor the asynchronous generation status with retry logic and wait intervals.
- Handle success and failure cases gracefully, returning structured JSON responses.
- Log requests for monitoring and debugging.

**Logical blocks:**

- **1.1 Input Reception and Parameter Setup:** Manual trigger starts the flow; sets API token and all model parameters.
- **1.2 Generation Request:** Sends a prediction request to Replicate with configured parameters.
- **1.3 Monitoring and Retry Loop:** Periodically checks the prediction status with wait nodes and conditional branching.
- **1.4 Result Handling:** Routes successful or failed predictions to appropriate response formatting.
- **1.5 Logging:** Logs request data for traceability.
- **1.6 Output Preparation:** Prepares the final output response for downstream consumption or display.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Parameter Setup

**Overview:**  
This block initiates the workflow manually and prepares all necessary parameters including authentication for the Replicate API and model-specific inputs.

**Nodes Involved:**  
- Manual Trigger  
- Set API Token  
- Set Other Parameters  

**Node Details:**

- **Manual Trigger**  
  - Type: Trigger node to start workflow manually.  
  - Configuration: Default, no parameters.  
  - Inputs: None  
  - Outputs: To "Set API Token"  
  - Edge Cases: None  

- **Set API Token**  
  - Type: Set node to define API token variable.  
  - Configuration: Sets `api_token` string to placeholder "YOUR_REPLICATE_API_TOKEN" (to be replaced by user).  
  - Inputs: Manual Trigger  
  - Outputs: To "Set Other Parameters"  
  - Edge Cases: Missing or invalid token will cause authentication errors downstream.  

- **Set Other Parameters**  
  - Type: Set node to configure all model input parameters.  
  - Configuration:  
    - Copies `api_token` from previous node.  
    - Sets defaults for mask, seed, image, model, width, height, prompt, go_fast, extra_lora, lora_scale, megapixels, num_outputs, aspect_ratio, output_format, guidance_scale, output_quality, prompt_strength, extra_lora_scale, replicate_weights, num_inference_steps, disable_safety_checker.  
    - Example prompt: "Create something amazing"  
  - Inputs: "Set API Token"  
  - Outputs: To "Create Other Prediction"  
  - Edge Cases: Invalid parameter types or missing required parameters (e.g., prompt) may cause API errors.

---

#### 1.2 Generation Request

**Overview:**  
This block sends the configured parameters as a POST request to the Replicate API to create a new prediction using the Alitas2 model.

**Nodes Involved:**  
- Create Other Prediction  

**Node Details:**

- **Create Other Prediction**  
  - Type: HTTP Request node (POST)  
  - Configuration:  
    - URL: `https://api.replicate.com/v1/predictions`  
    - Authorization: Bearer token from `api_token` field  
    - Headers: `Prefer: wait` to instruct synchronous waiting for completion on server side.  
    - Body: JSON with model version and all inputs from "Set Other Parameters" via expressions.  
    - Response expected as JSON, with `neverError: true` to avoid node failure on HTTP errors.  
  - Inputs: "Set Other Parameters"  
  - Outputs: To "Log Request"  
  - Edge Cases:  
    - API authentication failure if token is invalid or missing.  
    - Invalid parameter values causing API errors.  
    - Network or timeout errors.  
  - Version Specifics: Uses Replicate API v1 endpoint for predictions.

---

#### 1.3 Monitoring and Retry Loop

**Overview:**  
After sending the generation request, this block waits and polls the prediction status until completion or failure, with retry delays.

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
  - Configuration: Logs timestamp, prediction ID, and model type for monitoring.  
  - Inputs: "Create Other Prediction"  
  - Outputs: To "Wait 5s"  
  - Edge Cases: Logging failure does not impact workflow.  

- **Wait 5s**  
  - Type: Wait node  
  - Configuration: Pauses for 5 seconds before next step.  
  - Inputs: "Log Request"  
  - Outputs: To "Check Status"  
  - Edge Cases: None  

- **Check Status**  
  - Type: HTTP Request node (GET)  
  - Configuration:  
    - URL built dynamically using prediction ID from "Create Other Prediction".  
    - Authorization header with API token.  
    - Response as JSON with neverError true.  
  - Inputs: "Wait 5s" and "Wait 10s" (retry loop)  
  - Outputs: To "Is Complete?"  
  - Edge Cases: Network failure or invalid prediction ID.  

- **Is Complete?**  
  - Type: If node  
  - Configuration: Checks if `status` field equals `"succeeded"`.  
  - Inputs: "Check Status"  
  - Outputs:  
    - True: To "Success Response"  
    - False: To "Has Failed?"  
  - Edge Cases: Unexpected status values or missing status.  

- **Has Failed?**  
  - Type: If node  
  - Configuration: Checks if `status` field equals `"failed"`.  
  - Inputs: "Is Complete?" (false branch)  
  - Outputs:  
    - True: To "Error Response"  
    - False: To "Wait 10s" (retry)  
  - Edge Cases: Status neither succeeded nor failed (e.g., "processing") loops retry.  

- **Wait 10s**  
  - Type: Wait node  
  - Configuration: Pause for 10 seconds before retrying status check.  
  - Inputs: "Has Failed?" (false branch)  
  - Outputs: To "Check Status" (retry)  
  - Edge Cases: None  

---

#### 1.4 Result Handling

**Overview:**  
This block formats the final response depending on success or failure and prepares the output for consumption.

**Nodes Involved:**  
- Success Response  
- Error Response  
- Display Result  

**Node Details:**

- **Success Response**  
  - Type: Set node  
  - Configuration: Sets a JSON object with keys:  
    - `success: true`  
    - `result_url`: output image URL from prediction response  
    - `prediction_id`, `status`, and a success message  
  - Inputs: "Is Complete?" (true branch)  
  - Outputs: To "Display Result"  
  - Edge Cases: Missing output URL in response.  

- **Error Response**  
  - Type: Set node  
  - Configuration: Sets a JSON object with keys:  
    - `success: false`  
    - `error`: error message or fallback string  
    - `prediction_id`, `status`, and failure message  
  - Inputs: "Has Failed?" (true branch)  
  - Outputs: To "Display Result"  
  - Edge Cases: Missing or unexpected error details.  

- **Display Result**  
  - Type: Set node  
  - Configuration: Assigns the final response object to `final_result` for downstream use or output.  
  - Inputs: "Success Response" and "Error Response"  
  - Outputs: None (end of flow)  
  - Edge Cases: None  

---

#### 1.5 Logging

**Overview:**  
Logs the details of each prediction request for traceability and debugging.

**Nodes Involved:**  
- Log Request  

**Node Details:**

- **Log Request**  
  - See above in 1.3 Monitoring and Retry Loop  
  - Additional Note: Logs are printed to console or n8n logs for developer reference.  

---

#### 1.6 Documentation and Annotations

**Overview:**  
Sticky notes provide detailed documentation, parameter references, troubleshooting tips, and contact information.

**Nodes Involved:**  
- Sticky Note9  
- Sticky Note4  

**Node Details:**

- **Sticky Note9**  
  - Content: Branding and contact info for support, links to YouTube and LinkedIn channels.  
  - Position: Top left corner near start nodes.  

- **Sticky Note4**  
  - Content: Comprehensive workflow description, model overview, parameter reference, detailed instructions, troubleshooting, and external links.  
  - Position: Large note spanning left side of workspace.  

---

### 3. Summary Table

| Node Name              | Node Type          | Functional Role                       | Input Node(s)                 | Output Node(s)             | Sticky Note                                                                                           |
|------------------------|--------------------|------------------------------------|------------------------------|----------------------------|-----------------------------------------------------------------------------------------------------|
| Manual Trigger         | Manual Trigger     | Starts workflow execution           | None                         | Set API Token               |                                                                                                     |
| Set API Token          | Set                | Defines Replicate API token         | Manual Trigger               | Set Other Parameters        |                                                                                                     |
| Set Other Parameters   | Set                | Configures all model input params   | Set API Token                | Create Other Prediction     |                                                                                                     |
| Create Other Prediction| HTTP Request       | Sends generation request to API     | Set Other Parameters         | Log Request                |                                                                                                     |
| Log Request            | Code               | Logs prediction request details     | Create Other Prediction      | Wait 5s                    |                                                                                                     |
| Wait 5s                | Wait               | Waits 5 seconds before status check | Log Request                  | Check Status               |                                                                                                     |
| Check Status           | HTTP Request       | Retrieves prediction status         | Wait 5s, Wait 10s            | Is Complete?               |                                                                                                     |
| Is Complete?           | If                 | Checks if generation succeeded      | Check Status                 | Success Response, Has Failed? |                                                                                                  |
| Has Failed?            | If                 | Checks if generation failed         | Is Complete?                 | Error Response, Wait 10s   |                                                                                                     |
| Wait 10s               | Wait               | Waits 10 seconds before retry       | Has Failed?                  | Check Status               |                                                                                                     |
| Success Response       | Set                | Formats success JSON response       | Is Complete?                 | Display Result             |                                                                                                     |
| Error Response         | Set                | Formats error JSON response         | Has Failed?                  | Display Result             |                                                                                                     |
| Display Result         | Set                | Finalizes output response           | Success Response, Error Response | None                  |                                                                                                     |
| Sticky Note9           | Sticky Note        | Branding and contact info            | None                         | None                       | ======================================= ALITAS2 GENERATOR ... YouTube: https://www.youtube.com/@YaronBeen/videos |
| Sticky Note4           | Sticky Note        | Comprehensive workflow explanation  | None                         | None                       | See detailed workflow info, parameters, troubleshooting, and external links in content              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node:**  
   - Name it "Manual Trigger".  
   - Leave default settings.  

2. **Add a Set node for API token:**  
   - Name it "Set API Token".  
   - Add a field `api_token` of type string.  
   - Set value to `"YOUR_REPLICATE_API_TOKEN"` (replace with actual token before use).  
   - Connect "Manual Trigger" main output to this node.  

3. **Add a Set node for model parameters:**  
   - Name it "Set Other Parameters".  
   - Add fields with the following keys, types, and default values:  
     - `api_token` (string): `={{ $('Set API Token').item.json.api_token }}` (expression copy)  
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
   - Connect "Set API Token" output to this node.  

4. **Add an HTTP Request node to create prediction:**  
   - Name it "Create Other Prediction".  
   - Method: POST  
   - URL: `https://api.replicate.com/v1/predictions`  
   - Authentication: None (token passed via header manually)  
   - Headers:  
     - `Authorization`: `=Bearer {{ $json.api_token }}`  
     - `Prefer`: `wait`  
   - Body Content Type: JSON  
   - Body: Use raw JSON with expressions for all parameters as:  
     ```json
     {
       "version": "alitas126/alitas2:754ca44a4c14385a511b565efa1a19a04204b629738285b3de1ef94de578e5bd",
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
   - Enable "Send Body" and "Send Headers" options.  
   - Connect "Set Other Parameters" output to this node.  

5. **Add a Code node for logging:**  
   - Name it "Log Request".  
   - JavaScript code:  
     ```js
     const data = $input.all()[0].json;
     console.log('alitas126/alitas2 Request:', {
       timestamp: new Date().toISOString(),
       prediction_id: data.id,
       model_type: 'other'
     });
     return $input.all();
     ```  
   - Connect "Create Other Prediction" output to this node.  

6. **Add a Wait node for 5 seconds:**  
   - Name it "Wait 5s".  
   - Parameters: Wait 5 seconds.  
   - Connect "Log Request" output to this node.  

7. **Add HTTP Request node to check prediction status:**  
   - Name it "Check Status".  
   - Method: GET  
   - URL: `=https://api.replicate.com/v1/predictions/{{ $('Create Other Prediction').item.json.id }}`  
   - Headers:  
     - `Authorization`: `=Bearer {{ $('Set API Token').item.json.api_token }}`  
   - Response format: JSON, never error on failure to avoid node crash.  
   - Connect "Wait 5s" output to this node.  

8. **Add an If node to check if status is 'succeeded':**  
   - Name it "Is Complete?"  
   - Condition: `{{ $json.status }}` equals `"succeeded"`  
   - Connect "Check Status" output to this node.  

9. **Add an If node to check if status is 'failed':**  
   - Name it "Has Failed?"  
   - Condition: `{{ $json.status }}` equals `"failed"`  
   - Connect "Is Complete?" false output to this node.  

10. **Add a Wait node for 10 seconds (retry delay):**  
    - Name it "Wait 10s".  
    - Wait 10 seconds.  
    - Connect "Has Failed?" false output to this node.  
    - Connect "Wait 10s" output back to "Check Status" input to create retry loop.  

11. **Add Set node for success response:**  
    - Name it "Success Response".  
    - Set field `response` as object:  
      ```js
      {
        success: true,
        result_url: $json.output,
        prediction_id: $json.id,
        status: $json.status,
        message: 'Other generated successfully'
      }
      ```  
    - Connect "Is Complete?" true output to this node.  

12. **Add Set node for error response:**  
    - Name it "Error Response".  
    - Set field `response` as object:  
      ```js
      {
        success: false,
        error: $json.error || 'Other generation failed',
        prediction_id: $json.id,
        status: $json.status,
        message: 'Failed to generate other'
      }
      ```  
    - Connect "Has Failed?" true output to this node.  

13. **Add Set node to finalize output:**  
    - Name it "Display Result".  
    - Set field `final_result` to `={{ $json.response }}`  
    - Connect "Success Response" and "Error Response" output to this node.  

14. **Add Sticky Notes for documentation:**  
    - Create two sticky notes with contents as per the original, including branding, parameter explanations, troubleshooting tips, and external links.  
    - Position for visibility near the start and left side.  

---

### 5. General Notes & Resources

| Note Content                                                                                                       | Context or Link                                          |
|--------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------|
| For any questions or support, contact: Yaron@nofluff.online                                                       | Contact email                                            |
| Explore more tips and tutorials at YouTube: https://www.youtube.com/@YaronBeen/videos                              | YouTube channel                                          |
| LinkedIn profile: https://www.linkedin.com/in/yaronbeen/                                                          | LinkedIn profile                                        |
| Model Documentation: https://replicate.com/alitas126/alitas2                                                      | Official model docs                                     |
| Replicate API Documentation: https://replicate.com/docs                                                           | API reference                                          |
| n8n Documentation: https://docs.n8n.io                                                                             | Workflow automation docs                               |
| Replace "YOUR_REPLICATE_API_TOKEN" with your actual Replicate API token before running the workflow                | Essential setup step                                   |
| Ensure API token validity and sufficient credits to avoid authentication and quota errors                          | Operational best practice                               |
| Monitor the logs for long-running predictions or failures                                                         | Troubleshooting tip                                    |
| Use default parameters initially to verify workflow function before customization                                 | Best practice for parameter tuning                     |

---

**Disclaimer:**  
The provided text derives solely from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.