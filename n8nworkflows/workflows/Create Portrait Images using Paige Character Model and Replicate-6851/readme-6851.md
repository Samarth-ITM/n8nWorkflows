Create Portrait Images using Paige Character Model and Replicate

https://n8nworkflows.xyz/workflows/create-portrait-images-using-paige-character-model-and-replicate-6851


# Create Portrait Images using Paige Character Model and Replicate

### 1. Workflow Overview

This workflow automates the creation of portrait images using the Paige character model hosted on Replicate’s API. It is designed for generating AI-driven “other” images, such as stylized portraits or creative visuals based on user prompts and image inputs. The workflow handles the full lifecycle from input setup, API authentication, request submission, asynchronous status polling, to success/error handling and final output delivery.

The workflow logic is organized into the following blocks:

- **1.1 Input Reception and Initialization:** Starting the workflow manually and setting the API token.
- **1.2 Parameter Configuration:** Setting all model parameters, including prompt, image, and generation settings.
- **1.3 Prediction Creation:** Sending the generation request to Replicate and receiving a prediction ID.
- **1.4 Polling and Status Checking:** Waiting, polling, and looping to check if the prediction is completed or failed.
- **1.5 Result Handling:** Processing success or failure, formatting output responses.
- **1.6 Logging and Monitoring:** Recording request details for audit and debugging.
- **1.7 Informational Notes:** Embedded sticky notes providing documentation, instructions, and contact info.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Initialization

- **Overview:**  
  This block starts the workflow manually and sets the Replicate API token required for authentication with the model endpoint.

- **Nodes Involved:**  
  - Manual Trigger  
  - Set API Token

- **Node Details:**

  - **Manual Trigger**  
    - Type: Trigger node; starts workflow execution on user command.  
    - Configuration: No parameters; user manually triggers.  
    - Inputs: None  
    - Outputs: Connects to Set API Token  
    - Edge Cases: None (manual start).  
    - Version: 1

  - **Set API Token**  
    - Type: Set node; used to store API token in workflow context.  
    - Configuration: Static string assignment for `api_token` with placeholder `"YOUR_REPLICATE_API_TOKEN"` to be replaced by actual token.  
    - Inputs: From Manual Trigger  
    - Outputs: Connects to Set Other Parameters  
    - Edge Cases: If token is not replaced or invalid, API calls will fail authentication.  
    - Version: 3.3

#### 2.2 Parameter Configuration

- **Overview:**  
  Sets all other parameters used to define the image generation request. Combines required and optional parameters with sensible defaults.

- **Nodes Involved:**  
  - Set Other Parameters

- **Node Details:**

  - **Set Other Parameters**  
    - Type: Set node; prepares all input parameters for the Replicate model request.  
    - Configuration:  
      - Copies the API token from previous node dynamically.  
      - Sets parameters such as `mask` (default placeholder URL), `seed` (-1 for random), `image` (default placeholder image URL), `model` ("dev"), `width` and `height` (512), `prompt` ("Create something amazing"), and many others controlling generation quality, style, and output format.  
    - Key expressions:  
      - `api_token` copied via expression from Set API Token node.  
      - Other parameters are static or default values.  
    - Inputs: From Set API Token  
    - Outputs: Connects to Create Other Prediction  
    - Edge Cases: Misconfiguration may lead to invalid API payloads or unexpected generation results.  
    - Version: 3.3

#### 2.3 Prediction Creation

- **Overview:**  
  Sends the generation request with all configured parameters to Replicate’s API, creating a prediction job and receiving a prediction ID for tracking.

- **Nodes Involved:**  
  - Create Other Prediction

- **Node Details:**

  - **Create Other Prediction**  
    - Type: HTTP Request node; performs POST to https://api.replicate.com/v1/predictions  
    - Configuration:  
      - Uses JSON body with version ID fixed for the Paige model.  
      - Inputs for all generation parameters populated dynamically from previous node.  
      - Headers include Authorization Bearer token from `api_token` and "Prefer: wait" for synchronous behavior.  
      - Response expected in JSON format.  
    - Inputs: From Set Other Parameters  
    - Outputs: Connects to Log Request  
    - Edge Cases:  
      - Authentication failures if token invalid.  
      - API rate limits or downtime.  
      - Payload validation errors if parameters malformatted.  
    - Version: 4.1

#### 2.4 Polling and Status Checking

- **Overview:**  
  Polls the prediction status repeatedly with waits to handle asynchronous processing, loops until the prediction either succeeds or fails.

- **Nodes Involved:**  
  - Log Request  
  - Wait 5s  
  - Check Status  
  - Is Complete?  
  - Has Failed?  
  - Wait 10s

- **Node Details:**

  - **Log Request**  
    - Type: Code node; logs prediction request details for auditing.  
    - Configuration: Logs timestamp, prediction ID, and model type to console.  
    - Inputs: From Create Other Prediction  
    - Outputs: Connects to Wait 5s  
    - Edge Cases: None critical; logging failure only affects diagnostics.  
    - Version: 2

  - **Wait 5s**  
    - Type: Wait node; pauses workflow 5 seconds before status check.  
    - Configuration: Waits for 5 seconds.  
    - Inputs: From Log Request  
    - Outputs: Connects to Check Status  
    - Version: 1

  - **Check Status**  
    - Type: HTTP Request node; GET request to prediction status endpoint using prediction ID.  
    - Configuration:  
      - URL dynamically built with prediction ID from Create Other Prediction node.  
      - Authorization header with API token.  
      - Expects JSON response containing current status.  
    - Inputs: From Wait 5s and Wait 10s (loop)  
    - Outputs: Connects to Is Complete?  
    - Edge Cases:  
      - Network issues or API downtime.  
      - Token expiration or invalidation.  
    - Version: 4.1

  - **Is Complete?**  
    - Type: If node; checks if prediction status equals "succeeded".  
    - Configuration: Compares `$json.status` to "succeeded".  
    - Inputs: From Check Status  
    - Outputs:  
      - True branch to Success Response  
      - False branch to Has Failed?  
    - Version: 2

  - **Has Failed?**  
    - Type: If node; checks if prediction status equals "failed".  
    - Configuration: Compares `$json.status` to "failed".  
    - Inputs: From Is Complete?  
    - Outputs:  
      - True branch to Error Response  
      - False branch to Wait 10s (retry polling)  
    - Version: 2

  - **Wait 10s**  
    - Type: Wait node; delays 10 seconds before rechecking status.  
    - Configuration: Waits for 10 seconds.  
    - Inputs: From Has Failed? false branch  
    - Outputs: Connects back to Check Status (looping)  
    - Version: 1

#### 2.5 Result Handling

- **Overview:**  
  Creates structured success or error responses based on prediction outcome and delivers final result.

- **Nodes Involved:**  
  - Success Response  
  - Error Response  
  - Display Result

- **Node Details:**

  - **Success Response**  
    - Type: Set node; formats a JSON object marking success with result URL, prediction ID, and status.  
    - Configuration: Assigns a `response` object with properties: `success: true`, `result_url`, `prediction_id`, `status`, and a success message.  
    - Inputs: From Is Complete? true branch  
    - Outputs: Connects to Display Result  
    - Version: 3.3

  - **Error Response**  
    - Type: Set node; formats a JSON object marking failure with error details, prediction ID, and status.  
    - Configuration: Assigns a `response` object with properties: `success: false`, `error` (uses `$json.error` or default), `prediction_id`, `status`, and failure message.  
    - Inputs: From Has Failed? true branch  
    - Outputs: Connects to Display Result  
    - Version: 3.3

  - **Display Result**  
    - Type: Set node; sets the final output field `final_result` with the response object from either success or error branch.  
    - Configuration: Assigns `final_result` to `$json.response`.  
    - Inputs: From Success Response and Error Response  
    - Outputs: End node (no further processing)  
    - Version: 3.3

#### 2.6 Logging and Monitoring

- **Overview:**  
  Logs all prediction requests for debugging and monitoring, including timestamps and prediction IDs.

- **Nodes Involved:**  
  - Log Request (already described in 2.4)

- **Node Details:**  
  See section 2.4 above.

#### 2.7 Informational Notes

- **Overview:**  
  Provides embedded documentation and contact information for users and developers.

- **Nodes Involved:**  
  - Sticky Note9  
  - Sticky Note4

- **Node Details:**

  - **Sticky Note9**  
    - Type: Sticky Note node; displays contact info and links for support and tutorials.  
    - Content Highlights:  
      - Contact email: Yaron@nofluff.online  
      - YouTube and LinkedIn links for tips  
    - Position: Top-left area  
    - Version: 1

  - **Sticky Note4**  
    - Type: Large Sticky Note node; detailed documentation of model, parameters, workflow components, benefits, instructions, and troubleshooting.  
    - Content Highlights:  
      - Model description and API details  
      - Parameter reference with required and optional fields  
      - Workflow step explanations  
      - Quick start guide and troubleshooting tips  
      - Useful links to Replicate and n8n documentation  
    - Position: Left side  
    - Version: 1

---

### 3. Summary Table

| Node Name              | Node Type           | Functional Role                                | Input Node(s)            | Output Node(s)          | Sticky Note                                                                                                   |
|------------------------|---------------------|------------------------------------------------|--------------------------|-------------------------|--------------------------------------------------------------------------------------------------------------|
| Manual Trigger         | Manual Trigger      | Starts workflow execution                       | None                     | Set API Token            |                                                                                                              |
| Set API Token          | Set                 | Stores API token for authentication            | Manual Trigger           | Set Other Parameters     |                                                                                                              |
| Set Other Parameters   | Set                 | Configures all generation parameters           | Set API Token            | Create Other Prediction  |                                                                                                              |
| Create Other Prediction| HTTP Request        | Sends generation request to Replicate API      | Set Other Parameters     | Log Request              |                                                                                                              |
| Log Request            | Code                | Logs prediction request details                 | Create Other Prediction  | Wait 5s                  |                                                                                                              |
| Wait 5s                | Wait                | Delays 5 seconds before status check            | Log Request              | Check Status             |                                                                                                              |
| Check Status           | HTTP Request        | Polls prediction status from Replicate API     | Wait 5s, Wait 10s        | Is Complete?             |                                                                                                              |
| Is Complete?           | If                  | Checks if prediction succeeded                  | Check Status             | Success Response, Has Failed? |                                                                                                              |
| Has Failed?            | If                  | Checks if prediction failed                      | Is Complete?             | Error Response, Wait 10s |                                                                                                              |
| Wait 10s               | Wait                | Delays 10 seconds before retrying status check | Has Failed?              | Check Status             |                                                                                                              |
| Success Response       | Set                 | Formats success JSON response                    | Is Complete?             | Display Result           |                                                                                                              |
| Error Response         | Set                 | Formats error JSON response                      | Has Failed?              | Display Result           |                                                                                                              |
| Display Result         | Set                 | Outputs final result object                      | Success Response, Error Response | None                    |                                                                                                              |
| Sticky Note9           | Sticky Note         | Contact info and support links                   | None                     | None                    | =======================================<br>PAIGE GENERATOR<br>For support: Yaron@nofluff.online<br>YouTube: https://www.youtube.com/@YaronBeen/videos<br>LinkedIn: https://www.linkedin.com/in/yaronbeen/ |
| Sticky Note4           | Sticky Note         | Detailed workflow and model documentation        | None                     | None                    | Model overview, parameter guide, workflow steps, benefits, quick start, troubleshooting, links to docs       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger node**  
   - Type: Manual Trigger  
   - No parameters needed.

2. **Create Set node named “Set API Token”**  
   - Type: Set  
   - Add a string field named `api_token`.  
   - Set its value to your actual Replicate API token (replace `YOUR_REPLICATE_API_TOKEN`).  
   - Connect Manual Trigger → Set API Token.

3. **Create Set node named “Set Other Parameters”**  
   - Type: Set  
   - Define fields with the following names and default values:  
     - `api_token` (string): expression `={{ $('Set API Token').item.json.api_token }}`  
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
   - Connect Set API Token → Set Other Parameters.

4. **Create HTTP Request node named “Create Other Prediction”**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.replicate.com/v1/predictions`  
   - Headers:  
     - `Authorization`: `Bearer {{$json.api_token}}` (use expression from `api_token` field)  
     - `Prefer`: `wait`  
   - Body Type: JSON  
   - Body Content: JSON object with:  
     ```json
     {
       "version": "paigedutcher2/paige:45f3e91abd75bedac5de72b4da3a63376053067c79fa1ba98a11aafd79a13732",
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
   - Connect Set Other Parameters → Create Other Prediction.

5. **Create Code node named “Log Request”**  
   - Type: Code  
   - Language: JavaScript  
   - Code:  
     ```javascript
     const data = $input.all()[0].json;
     console.log('paigedutcher2/paige Request:', {
       timestamp: new Date().toISOString(),
       prediction_id: data.id,
       model_type: 'other'
     });
     return $input.all();
     ```  
   - Connect Create Other Prediction → Log Request.

6. **Create Wait node named “Wait 5s”**  
   - Type: Wait  
   - Wait for 5 seconds  
   - Connect Log Request → Wait 5s.

7. **Create HTTP Request node named “Check Status”**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://api.replicate.com/v1/predictions/{{ $('Create Other Prediction').item.json.id }}`  
   - Headers:  
     - `Authorization`: `Bearer {{ $('Set API Token').item.json.api_token }}`  
   - Connect Wait 5s → Check Status.

8. **Create If node named “Is Complete?”**  
   - Type: If  
   - Condition: `$json.status` equals `succeeded`  
   - Connect Check Status → Is Complete?.

9. **Create Set node named “Success Response”**  
   - Type: Set  
   - Assign field `response` with an object:  
     ```json
     {
       "success": true,
       "result_url": $json.output,
       "prediction_id": $json.id,
       "status": $json.status,
       "message": "Other generated successfully"
     }
     ```  
   - Connect Is Complete? true → Success Response.

10. **Create Set node named “Display Result”**  
    - Type: Set  
    - Assign field `final_result` to `$json.response`  
    - Connect Success Response → Display Result.

11. **Create If node named “Has Failed?”**  
    - Type: If  
    - Condition: `$json.status` equals `failed`  
    - Connect Is Complete? false → Has Failed?.

12. **Create Set node named “Error Response”**  
    - Type: Set  
    - Assign field `response` with an object:  
      ```json
      {
        "success": false,
        "error": $json.error || "Other generation failed",
        "prediction_id": $json.id,
        "status": $json.status,
        "message": "Failed to generate other"
      }
      ```  
    - Connect Has Failed? true → Error Response.

13. **Connect Error Response → Display Result.**

14. **Create Wait node named “Wait 10s”**  
    - Type: Wait  
    - Wait for 10 seconds  
    - Connect Has Failed? false → Wait 10s.

15. **Connect Wait 10s → Check Status** (loop for status polling).

16. **Connect Display Result as workflow endpoint (no further nodes).**

17. **Add Sticky Notes** for documentation and contact:  
    - One with contact and support links.  
    - One with detailed workflow and parameter documentation.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | Context or Link                                                                                   |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------|
| =======================================<br>PAIGE GENERATOR<br>For any questions or support, please contact: Yaron@nofluff.online<br>Explore more tips and tutorials here:<br>• YouTube: https://www.youtube.com/@YaronBeen/videos<br>• LinkedIn: https://www.linkedin.com/in/yaronbeen/<br>=======================================                                                                                                                                                                                      | Support contact and tutorial links embedded as sticky note                                        |
| Detailed model description, parameter guide, workflow explanation, quick start instructions, troubleshooting tips, and links to Replicate and n8n docs. Key highlights include the usage of the "paigedutcher2/paige" model on Replicate, configuration of multiple parameters, error handling, and stepwise polling with retry logic.                                                                                                                                                                                                              | Detailed workflow and model documentation sticky note                                            |
| Model Documentation: https://replicate.com/paigedutcher2/paige<br>Replicate API Docs: https://replicate.com/docs<br>n8n Documentation: https://docs.n8n.io                                                                                                                                                                                                                                                                                                                                                                                                | Official documentation links                                                                      |

---

This structured documentation enables a developer or automation system to fully comprehend, reproduce, and adapt the “Create Portrait Images using Paige Character Model and Replicate” workflow in n8n with clarity on each component’s functionality, configuration, and error handling.