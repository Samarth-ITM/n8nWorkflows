Creating images from text prompts using Herathaisbragatto2 on Replicate

https://n8nworkflows.xyz/workflows/creating-images-from-text-prompts-using-herathaisbragatto2-on-replicate-6857


# Creating images from text prompts using Herathaisbragatto2 on Replicate

### 1. Workflow Overview

This n8n workflow automates the creation of images from text prompts by interfacing with the "herathaisbragatto2" AI model hosted on Replicate. It targets users who want to generate images (referred to as "others" in the workflow) by supplying descriptive text prompts and optional parameters, enabling flexible image generation with control over aspects such as size, style, and output format.

The workflow is organized into the following logical blocks:

- **1.1 Input Reception and Initialization**  
  Triggering the workflow manually and setting up authentication and generation parameters.

- **1.2 Image Generation Request Submission**  
  Sending the configured parameters to Replicate’s API to initiate image generation.

- **1.3 Generation Status Monitoring Loop**  
  Waiting and polling the API to check the progress of the generation request until completion or failure.

- **1.4 Response Handling**  
  Handling success or failure outcomes by preparing structured responses.

- **1.5 Logging and Final Output Preparation**  
  Logging request details and preparing the final output for consumption.

Each block is connected sequentially to ensure smooth orchestration of the image generation process.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Initialization

- **Overview:**  
  This block initializes the workflow execution, sets the required API token for authentication, and configures all input parameters for image generation with default values.

- **Nodes Involved:**  
  - Manual Trigger  
  - Set API Token  
  - Set Other Parameters

- **Node Details:**

  - **Manual Trigger**  
    - *Type & Role:* Starts the workflow manually when a user initiates it in n8n.  
    - *Configuration:* No parameters; simple manual start.  
    - *Connections:* Outputs to "Set API Token".  
    - *Failures:* None expected; user must trigger manually.

  - **Set API Token**  
    - *Type & Role:* Sets the Replicate API token as a string variable for authentication.  
    - *Configuration:* Hardcoded placeholder `"YOUR_REPLICATE_API_TOKEN"` that users must replace with their actual token.  
    - *Key Variable:* `api_token` used downstream for API calls.  
    - *Connections:* Outputs to "Set Other Parameters".  
    - *Failures:* Failure if token is invalid or missing, but this node itself just assigns the value.  
    - *Note:* Must be replaced with a valid token before execution.

  - **Set Other Parameters**  
    - *Type & Role:* Defines all other input parameters for the image generation model, including required and optional fields.  
    - *Configuration:* Assigns parameters such as `prompt`, `mask`, `seed`, `image`, `model`, `width`, `height`, `go_fast`, `extra_lora`, `lora_scale`, `megapixels`, `num_outputs`, `aspect_ratio`, `output_format`, `guidance_scale`, `output_quality`, `prompt_strength`, `extra_lora_scale`, `replicate_weights`, `num_inference_steps`, `disable_safety_checker`.  
    - *Key Expressions:* Copies `api_token` from previous node using expression `={{ $('Set API Token').item.json.api_token }}`.  
    - *Connections:* Outputs to "Create Other Prediction".  
    - *Failures:* Potential misconfiguration if parameters are invalid or incompatible with model expectations.

#### 2.2 Image Generation Request Submission

- **Overview:**  
  Sends a POST request to Replicate API to initiate the image generation job with the configured parameters.

- **Nodes Involved:**  
  - Create Other Prediction

- **Node Details:**

  - **Create Other Prediction**  
    - *Type & Role:* HTTP Request node that calls Replicate API endpoint `/v1/predictions` to start generation.  
    - *Configuration:*  
      - Method: POST  
      - URL: `https://api.replicate.com/v1/predictions`  
      - Headers: Authorization Bearer token from `$json.api_token`, and `"Prefer": "wait"` (to wait for completion if possible)  
      - Body: JSON object including the model version and all input parameters from previous node via expressions.  
      - Response expected as JSON, with error suppression (`neverError: true`).  
    - *Connections:* Outputs to "Log Request".  
    - *Failures:* Possible HTTP errors, invalid token, invalid parameters, API rate limits, or network issues.

#### 2.3 Generation Status Monitoring Loop

- **Overview:**  
  Implements a polling mechanism to check the status of the image generation job on Replicate, waiting between polls and retrying if not complete.

- **Nodes Involved:**  
  - Log Request  
  - Wait 5s  
  - Check Status  
  - Is Complete?  
  - Has Failed?  
  - Wait 10s

- **Node Details:**

  - **Log Request**  
    - *Type & Role:* Code node that logs generation details such as timestamp, prediction ID, and model type for monitoring purposes.  
    - *Configuration:* JavaScript code logs to console, then passes data unchanged.  
    - *Connections:* Outputs to "Wait 5s".  
    - *Failures:* Logging errors unlikely; watch for code errors.

  - **Wait 5s**  
    - *Type & Role:* Waits 5 seconds to throttle polling frequency.  
    - *Configuration:* Wait duration set to 5 seconds.  
    - *Connections:* Outputs to "Check Status".  
    - *Failures:* None expected.

  - **Check Status**  
    - *Type & Role:* HTTP Request node queries Replicate API for current status of prediction by prediction ID.  
    - *Configuration:*  
      - Method: GET  
      - URL: `https://api.replicate.com/v1/predictions/{{ $('Create Other Prediction').item.json.id }}`  
      - Header: Authorization with API token from "Set API Token" node.  
      - Response JSON and error suppression enabled.  
    - *Connections:* Outputs to "Is Complete?".  
    - *Failures:* Network errors, invalid ID, expired token.

  - **Is Complete?**  
    - *Type & Role:* Conditional node checking if the prediction status is `"succeeded"`.  
    - *Configuration:* Checks `$json.status === "succeeded"`.  
    - *Connections:*  
      - True path to "Success Response".  
      - False path to "Has Failed?".  
    - *Failures:* Expression evaluation errors possible if response malformed.

  - **Has Failed?**  
    - *Type & Role:* Conditional node checking if prediction status is `"failed"`.  
    - *Configuration:* Checks `$json.status === "failed"`.  
    - *Connections:*  
      - True path to "Error Response".  
      - False path to "Wait 10s" (retry delay).  
    - *Failures:* Same as above.

  - **Wait 10s**  
    - *Type & Role:* Waits 10 seconds before retrying status check to avoid excessive API calls.  
    - *Configuration:* Wait duration set to 10 seconds.  
    - *Connections:* Loops back to "Check Status".  
    - *Failures:* None expected.

#### 2.4 Response Handling

- **Overview:**  
  Prepares structured JSON responses for both successful and failed generation attempts.

- **Nodes Involved:**  
  - Success Response  
  - Error Response  
  - Display Result

- **Node Details:**

  - **Success Response**  
    - *Type & Role:* Sets a success response object containing result URL, prediction ID, status, and message.  
    - *Configuration:* Constructs JSON object with keys: `success: true`, `result_url`, `prediction_id`, `status`, and a friendly message.  
    - *Connections:* Outputs to "Display Result".  
    - *Failures:* None expected if input data is valid.

  - **Error Response**  
    - *Type & Role:* Sets an error response object with error details, prediction ID, status, and message.  
    - *Configuration:* Constructs JSON object with keys: `success: false`, `error` (or fallback string), `prediction_id`, `status`, and message.  
    - *Connections:* Outputs to "Display Result".  
    - *Failures:* None expected unless input JSON malformed.

  - **Display Result**  
    - *Type & Role:* Final node to package the response under `final_result` property for output consumption.  
    - *Configuration:* Sets `final_result` to the response JSON from either success or error path.  
    - *Connections:* Terminal node, no further outputs.  
    - *Failures:* None expected.

#### 2.5 Logging and Metadata

- **Overview:**  
  A code node logs generation requests to the console for monitoring and debugging purposes.

- **Nodes Involved:**  
  - Log Request (already covered in 2.3)

- No additional nodes besides those already described.

---

### 3. Summary Table

| Node Name            | Node Type           | Functional Role                        | Input Node(s)            | Output Node(s)           | Sticky Note                                                                                     |
|----------------------|---------------------|-------------------------------------|--------------------------|--------------------------|------------------------------------------------------------------------------------------------|
| Manual Trigger       | manualTrigger       | Starts the workflow manually         |                          | Set API Token             | See "HERATHAISBRAGATTO2 GENERATOR" contact & support info note covering initial nodes          |
| Set API Token        | set                 | Sets API token for authentication   | Manual Trigger           | Set Other Parameters      | See note about replacing 'YOUR_REPLICATE_API_TOKEN'                                           |
| Set Other Parameters | set                 | Defines all generation parameters    | Set API Token            | Create Other Prediction   | Detailed parameter guide and model overview note applies here                                  |
| Create Other Prediction | httpRequest        | Sends generation request to Replicate API | Set Other Parameters | Log Request               | API call node with auth header and wait preference                                            |
| Log Request          | code                | Logs request data for monitoring    | Create Other Prediction  | Wait 5s                  | Part of logging and monitoring block                                                          |
| Wait 5s              | wait                | Waits 5 seconds before status check | Log Request              | Check Status              | Polling delay                                                                                  |
| Check Status         | httpRequest         | Checks generation status             | Wait 5s, Wait 10s        | Is Complete?              | Uses prediction ID to query status                                                           |
| Is Complete?         | if                  | Checks if generation succeeded       | Check Status             | Success Response, Has Failed? | Determines next action based on status                                                  |
| Has Failed?          | if                  | Checks if generation failed          | Is Complete?             | Error Response, Wait 10s  | Handles failure or retry                                                                     |
| Wait 10s             | wait                | Waits 10 seconds before retry        | Has Failed?              | Check Status              | Retry delay                                                                                   |
| Success Response     | set                 | Prepares success response payload    | Is Complete? (true path) | Display Result            | Sends back successful generation details                                                   |
| Error Response       | set                 | Prepares error response payload      | Has Failed? (true path)  | Display Result            | Sends back error information                                                               |
| Display Result       | set                 | Final output packaging                | Success Response, Error Response |                      | Final response exposed under `final_result`                                                |
| Sticky Note9         | stickyNote           | Informational note about the workflow |                         |                          | See "HERATHAISBRAGATTO2 GENERATOR" support contact and resources                            |
| Sticky Note4         | stickyNote           | Detailed workflow documentation note |                         |                          | Extensive notes covering model, parameters, instructions, and resources                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a "Manual Trigger" node:**  
   - No configuration needed. This node starts the workflow manually.

3. **Add a "Set" node named "Set API Token":**  
   - Add a string field assignment:  
     - Name: `api_token`  
     - Value: `"YOUR_REPLICATE_API_TOKEN"` (replace with your actual Replicate API token)  
   - Connect output from "Manual Trigger" to this node.

4. **Add a "Set" node named "Set Other Parameters":**  
   - Add multiple variable assignments with the following keys and default values (adjust as needed):  
     - `api_token`: expression `={{ $('Set API Token').item.json.api_token }}`  
     - `mask`: `"https://via.placeholder.com/512x512/000000/FFFFFF.png"`  
     - `seed`: `-1`  
     - `image`: `"https://picsum.photos/512/512"`  
     - `model`: `"dev"`  
     - `width`: `512`  
     - `height`: `512`  
     - `prompt`: `"Create something amazing"`  
     - `go_fast`: `false`  
     - `extra_lora`: `""`  
     - `lora_scale`: `1`  
     - `megapixels`: `"1"`  
     - `num_outputs`: `1`  
     - `aspect_ratio`: `"1:1"`  
     - `output_format`: `"webp"`  
     - `guidance_scale`: `3`  
     - `output_quality`: `80`  
     - `prompt_strength`: `0.8`  
     - `extra_lora_scale`: `1`  
     - `replicate_weights`: `""`  
     - `num_inference_steps`: `28`  
     - `disable_safety_checker`: `false`  
   - Connect output from "Set API Token" to this node.

5. **Add an "HTTP Request" node named "Create Other Prediction":**  
   - Method: POST  
   - URL: `https://api.replicate.com/v1/predictions`  
   - Headers:  
     - `Authorization`: `Bearer {{ $json.api_token }}`  
     - `Prefer`: `wait`  
   - Body Content Type: JSON  
   - Body: Use raw JSON with expressions:  
     ```json
     {
       "version": "digitalhera/herathaisbragatto2:30431431591f264ba3bb8b7f321d53d04ba573d7647c70ffa79e60a575c33117",
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
   - Connect output from "Set Other Parameters" to this node.

6. **Add a "Code" node named "Log Request":**  
   - JavaScript code:  
     ```javascript
     const data = $input.all()[0].json;
     console.log('digitalhera/herathaisbragatto2 Request:', {
       timestamp: new Date().toISOString(),
       prediction_id: data.id,
       model_type: 'other'
     });
     return $input.all();
     ```  
   - Connect output from "Create Other Prediction" to this node.

7. **Add a "Wait" node named "Wait 5s":**  
   - Wait for 5 seconds.  
   - Connect output from "Log Request" to this node.

8. **Add an "HTTP Request" node named "Check Status":**  
   - Method: GET  
   - URL: `https://api.replicate.com/v1/predictions/{{ $('Create Other Prediction').item.json.id }}`  
   - Header:  
     - `Authorization`: `Bearer {{ $('Set API Token').item.json.api_token }}`  
   - Connect output from "Wait 5s" to this node.

9. **Add an "If" node named "Is Complete?":**  
   - Condition:  
     - Check if `$json.status` equals `"succeeded"`.  
   - Connect output from "Check Status" to this node.

10. **Add a "Set" node named "Success Response":**  
    - Assign field: `response` as an object with:  
      ```json
      {
        "success": true,
        "result_url": $json.output,
        "prediction_id": $json.id,
        "status": $json.status,
        "message": "Other generated successfully"
      }
      ```  
    - Connect the "true" output of "Is Complete?" to this node.

11. **Add a "Set" node named "Display Result":**  
    - Assign field: `final_result` to `={{ $json.response }}`.  
    - Connect output from "Success Response" to this node.

12. **Add an "If" node named "Has Failed?":**  
    - Condition:  
      - Check if `$json.status` equals `"failed"`.  
    - Connect the "false" output of "Is Complete?" to this node.

13. **Add a "Set" node named "Error Response":**  
    - Assign field: `response` as an object with:  
      ```json
      {
        "success": false,
        "error": $json.error || 'Other generation failed',
        "prediction_id": $json.id,
        "status": $json.status,
        "message": "Failed to generate other"
      }
      ```  
    - Connect the "true" output of "Has Failed?" to this node.

14. **Connect "Error Response" output to "Display Result".**

15. **Add a "Wait" node named "Wait 10s":**  
    - Wait for 10 seconds.  
    - Connect the "false" output of "Has Failed?" to this node.

16. **Loop "Wait 10s" output back to "Check Status"** for retry polling.

17. **Ensure all connections match the above logic to implement the polling loop and response handling.**

18. **Test the workflow with a valid API token and adjust parameters in "Set Other Parameters" as needed.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                        | Context or Link                                                                                      |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| =======================================<br>HERATHAISBRAGATTO2 GENERATOR<br>=======================================<br>For any questions or support, please contact:<br>Yaron@nofluff.online<br>Explore more tips and tutorials here:<br>- YouTube: https://www.youtube.com/@YaronBeen/videos<br>- LinkedIn: https://www.linkedin.com/in/yaronbeen/<br>======================================= | Support contact and tutorial resources                                                             |
| Digitalhera/herathaisbragatto2 is powered by Replicate API and n8n automation, enabling advanced AI image generation workflows with extensive parameter control and error handling. See detailed workflow notes for parameter descriptions and usage tips.       | Comprehensive workflow documentation sticky note                                                  |
| Model Documentation: https://replicate.com/digitalhera/herathaisbragatto2<br>Replicate API Docs: https://replicate.com/docs<br>n8n Documentation: https://docs.n8n.io                                                                                             | Official documentation resources                                                                  |
| Replace the placeholder `YOUR_REPLICATE_API_TOKEN` with a valid token from https://replicate.com/account to enable API access.<br>Ensure API tokens are kept secure and not exposed publicly.<br>Monitor API usage and errors in logs during execution.             | Security and setup best practices                                                                 |
| The workflow implements intelligent polling with wait times of 5 and 10 seconds to avoid excessive API calls while monitoring generation status.                                                                                                                | Efficiency and error resilience notes                                                            |

---

**Disclaimer:**  
The text provided is exclusively from an automated workflow created with n8n, a tool for integration and automation. This process strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.