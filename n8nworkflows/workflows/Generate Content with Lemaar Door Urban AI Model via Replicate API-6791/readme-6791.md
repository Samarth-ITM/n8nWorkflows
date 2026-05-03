Generate Content with Lemaar Door Urban AI Model via Replicate API

https://n8nworkflows.xyz/workflows/generate-content-with-lemaar-door-urban-ai-model-via-replicate-api-6791


# Generate Content with Lemaar Door Urban AI Model via Replicate API

### 1. Workflow Overview

This n8n workflow automates the generation of creative content using the "lemaar-door-urban" AI model hosted on Replicate's API. It is designed for users who want to produce AI-generated images or other creative outputs by specifying parameters such as prompts, masks, image inputs, and various generation controls.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Initialization**: Receives manual trigger input and sets the API authentication token.
- **1.2 Parameter Configuration**: Defines all model input parameters, including defaults and user overrides.
- **1.3 Prediction Request Submission**: Sends a generation request to Replicate API and receives a prediction ID.
- **1.4 Asynchronous Status Polling and Handling**: Waits and repeatedly checks the prediction status until completion or failure.
- **1.5 Success and Error Response Construction**: Processes the final output or error, formats structured JSON responses.
- **1.6 Logging and Result Display**: Logs request details for monitoring and prepares the final output for return.
- **1.7 Documentation and Support Notes**: Informative sticky notes provide detailed model info, usage instructions, and troubleshooting.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Initialization

- **Overview:**  
  Starts the workflow manually and initializes the API token needed for authentication with Replicate API.

- **Nodes Involved:**  
  - Manual Trigger  
  - Set API Token

- **Node Details:**  

  - **Manual Trigger**  
    - *Type:* Trigger  
    - *Role:* Starts the workflow execution manually on user command.  
    - *Configuration:* Default manual trigger, no parameters.  
    - *Inputs:* None (start node)  
    - *Outputs:* Connects to "Set API Token" node.  
    - *Edge Cases:* None typical, but workflow won’t start without manual trigger.

  - **Set API Token**  
    - *Type:* Set  
    - *Role:* Stores the Replicate API token as a workflow variable for use in subsequent nodes.  
    - *Configuration:* Static string assignment with placeholder `"YOUR_REPLICATE_API_TOKEN"` that must be replaced with a valid token.  
    - *Inputs:* From Manual Trigger  
    - *Outputs:* Connects to "Set Other Parameters"  
    - *Edge Cases:* Missing or invalid token will cause authentication failures downstream.

#### 2.2 Parameter Configuration

- **Overview:**  
  Defines all the input parameters required by the "lemaar-door-urban" model, including defaults for prompt, image URLs, generation settings, and flags.

- **Nodes Involved:**  
  - Set Other Parameters

- **Node Details:**  

  - **Set Other Parameters**  
    - *Type:* Set  
    - *Role:* Assigns all required and optional parameters for the Replicate API request, referencing the API token from previous node.  
    - *Configuration:*  
      - Copies `api_token` from "Set API Token".  
      - Sets parameters like `mask`, `seed`, `image`, `model`, `width`, `height`, `prompt`, `go_fast` flag, `extra_lora`, and many others with initial values or placeholders.  
      - Uses expressions to dynamically reference previous node data (e.g., `={{ $('Set API Token').item.json.api_token }}`).  
    - *Inputs:* From "Set API Token"  
    - *Outputs:* Connects to "Create Other Prediction"  
    - *Edge Cases:* Incorrect parameter types or missing required fields (like prompt) may cause API errors.

#### 2.3 Prediction Request Submission

- **Overview:**  
  Sends a POST request to Replicate API to create a new prediction job with the configured parameters, and obtains a prediction ID for tracking.

- **Nodes Involved:**  
  - Create Other Prediction

- **Node Details:**  

  - **Create Other Prediction**  
    - *Type:* HTTP Request  
    - *Role:* Submits the generation request to Replicate’s `v1/predictions` endpoint using a JSON body.  
    - *Configuration:*  
      - Method: POST  
      - URL: https://api.replicate.com/v1/predictions  
      - Headers: Authorization Bearer token (from `api_token`), Prefer: wait (to wait for immediate response if possible)  
      - JSON Body: Includes model version hash and all input parameters dynamically interpolated.  
      - Response Format: JSON, with `neverError` to avoid throwing on HTTP errors.  
    - *Inputs:* From "Set Other Parameters"  
    - *Outputs:* Connects to "Log Request"  
    - *Edge Cases:*  
      - API token invalid or expired -> authentication failure.  
      - Parameter validation errors from API.  
      - Network timeouts or transient API errors.

#### 2.4 Asynchronous Status Polling and Handling

- **Overview:**  
  Implements a polling mechanism that waits and checks the prediction job status repeatedly until it finishes successfully or fails.

- **Nodes Involved:**  
  - Log Request  
  - Wait 5s  
  - Check Status  
  - Is Complete?  
  - Has Failed?  
  - Wait 10s

- **Node Details:**  

  - **Log Request**  
    - *Type:* Code  
    - *Role:* Logs prediction request details (timestamp, prediction_id, model_type) to console for monitoring/debugging.  
    - *Configuration:* Uses JavaScript to print log with JSON input data.  
    - *Inputs:* From "Create Other Prediction"  
    - *Outputs:* Connects to "Wait 5s"  
    - *Edge Cases:* Logging failure does not affect flow.

  - **Wait 5s**  
    - *Type:* Wait  
    - *Role:* Delays execution for 5 seconds before status check to allow prediction to progress.  
    - *Configuration:* Fixed 5 seconds delay.  
    - *Inputs:* From "Log Request"  
    - *Outputs:* Connects to "Check Status"  
    - *Edge Cases:* None significant; delay time can be adjusted depending on API response times.

  - **Check Status**  
    - *Type:* HTTP Request  
    - *Role:* Fetches current prediction status from Replicate API by prediction ID.  
    - *Configuration:*  
      - GET request to `https://api.replicate.com/v1/predictions/{{ prediction_id }}`  
      - Authorization header with Bearer token from "Set API Token" node.  
      - JSON response parsing enabled.  
    - *Inputs:* From "Wait 5s" and "Wait 10s"  
    - *Outputs:* Connects to "Is Complete?"  
    - *Edge Cases:* API errors, network issues, invalid prediction ID.

  - **Is Complete?**  
    - *Type:* If  
    - *Role:* Checks if prediction status is `"succeeded"`.  
    - *Configuration:* Condition: `$json.status == "succeeded"`  
    - *Inputs:* From "Check Status"  
    - *Outputs:*  
      - True branch: "Success Response"  
      - False branch: "Has Failed?"  
    - *Edge Cases:* Status may be pending or running; loop continues.

  - **Has Failed?**  
    - *Type:* If  
    - *Role:* Checks if prediction status is `"failed"`.  
    - *Configuration:* Condition: `$json.status == "failed"`  
    - *Inputs:* From "Is Complete?" (false branch)  
    - *Outputs:*  
      - True branch: "Error Response"  
      - False branch: "Wait 10s" (retry loop)  
    - *Edge Cases:* Rare states may require additional handling.

  - **Wait 10s**  
    - *Type:* Wait  
    - *Role:* Delays 10 seconds before retrying status check after failure check or intermediate status.  
    - *Configuration:* Fixed 10 seconds delay.  
    - *Inputs:* From "Has Failed?" (false branch)  
    - *Outputs:* Connects back to "Check Status"  
    - *Edge Cases:* May cause longer total wait times if prediction is slow.

#### 2.5 Success and Error Response Construction

- **Overview:**  
  Creates structured JSON responses for both success and failure outcomes to provide clear feedback.

- **Nodes Involved:**  
  - Success Response  
  - Error Response

- **Node Details:**  

  - **Success Response**  
    - *Type:* Set  
    - *Role:* Constructs a success response object containing success flag, result URL(s), prediction ID, status, and message.  
    - *Configuration:* Uses expressions to assign values from input JSON (e.g., output URL, prediction ID).  
    - *Inputs:* From "Is Complete?" (true branch)  
    - *Outputs:* Connects to "Display Result"  
    - *Edge Cases:* Missing output fields may result in incomplete data.

  - **Error Response**  
    - *Type:* Set  
    - *Role:* Constructs an error response object with failure flag, error message, prediction ID, status, and message.  
    - *Configuration:* Uses fallback error message if none provided.  
    - *Inputs:* From "Has Failed?" (true branch)  
    - *Outputs:* Connects to "Display Result"  
    - *Edge Cases:* May not capture all error details if API response is limited.

#### 2.6 Logging and Result Display

- **Overview:**  
  Finalizes the workflow by preparing the final result object for output and display.

- **Nodes Involved:**  
  - Display Result

- **Node Details:**  

  - **Display Result**  
    - *Type:* Set  
    - *Role:* Sets the final output object under `final_result` to be returned or used by other systems.  
    - *Configuration:* Passes the response object (success or error) as `final_result`.  
    - *Inputs:* From "Success Response" and "Error Response"  
    - *Outputs:* None (end node)  
    - *Edge Cases:* None significant.

#### 2.7 Documentation and Support Notes

- **Overview:**  
  Two sticky notes provide comprehensive documentation, usage instructions, parameter explanations, and support contacts.

- **Nodes Involved:**  
  - Sticky Note9  
  - Sticky Note4

- **Node Details:**  

  - **Sticky Note9**  
    - *Type:* Sticky Note  
    - *Role:* Displays contact and branding information for support purposes.  
    - *Content:* Contact email, YouTube channel, LinkedIn profile of the author.  
    - *Position:* Top-left corner for visibility.

  - **Sticky Note4**  
    - *Type:* Sticky Note  
    - *Role:* Extensive documentation including model overview, parameter usage, workflow explanation, quick start, and troubleshooting.  
    - *Content:* Markdown formatted guide and links to external resources.  
    - *Position:* Left side covering multiple nodes for reference during editing.

---

### 3. Summary Table

| Node Name             | Node Type          | Functional Role                                | Input Node(s)           | Output Node(s)           | Sticky Note                                                                                  |
|-----------------------|--------------------|-----------------------------------------------|-------------------------|--------------------------|----------------------------------------------------------------------------------------------|
| Manual Trigger        | Trigger            | Starts workflow manually                       | None                    | Set API Token             |                                                                                              |
| Set API Token         | Set                | Stores Replicate API token                     | Manual Trigger          | Set Other Parameters      |                                                                                              |
| Set Other Parameters  | Set                | Defines model input parameters                  | Set API Token           | Create Other Prediction   |                                                                                              |
| Create Other Prediction | HTTP Request      | Sends generation request to Replicate API     | Set Other Parameters    | Log Request               |                                                                                              |
| Log Request           | Code               | Logs prediction request details                | Create Other Prediction | Wait 5s                   |                                                                                              |
| Wait 5s               | Wait               | Delays 5 seconds before status check           | Log Request             | Check Status              |                                                                                              |
| Check Status          | HTTP Request       | Checks prediction job status                    | Wait 5s, Wait 10s       | Is Complete?              |                                                                                              |
| Is Complete?          | If                 | Determines if prediction succeeded             | Check Status            | Success Response, Has Failed? |                                                                                              |
| Has Failed?           | If                 | Determines if prediction failed                | Is Complete?            | Error Response, Wait 10s  |                                                                                              |
| Wait 10s              | Wait               | Delays 10 seconds before retrying status check| Has Failed?             | Check Status              |                                                                                              |
| Success Response      | Set                | Constructs success JSON response                | Is Complete? (true)     | Display Result            |                                                                                              |
| Error Response        | Set                | Constructs error JSON response                  | Has Failed? (true)      | Display Result            |                                                                                              |
| Display Result        | Set                | Prepares final output object                    | Success Response, Error Response | None               |                                                                                              |
| Sticky Note9          | Sticky Note        | Support contact and branding info               | None                    | None                     | Contact: Yaron@nofluff.online, YouTube: https://www.youtube.com/@YaronBeen/videos, LinkedIn: https://www.linkedin.com/in/yaronbeen/ |
| Sticky Note4          | Sticky Note        | Full documentation and parameter guide          | None                    | None                     | Extensive model info, usage instructions, troubleshooting, and links to Replicate and n8n docs |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - No parameters needed  
   - Position: Start node

2. **Create Set Node - "Set API Token"**  
   - Assign a string parameter named `api_token` with value `"YOUR_REPLICATE_API_TOKEN"` (replace with your actual token)  
   - Connect Manual Trigger → Set API Token

3. **Create Set Node - "Set Other Parameters"**  
   - Assign multiple parameters:  
     - Copy `api_token` from "Set API Token" via expression `={{ $('Set API Token').item.json.api_token }}`  
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
   - Connect Set API Token → Set Other Parameters

4. **Create HTTP Request Node - "Create Other Prediction"**  
   - Method: POST  
   - URL: `https://api.replicate.com/v1/predictions`  
   - Headers:  
     - `Authorization`: `Bearer {{$json.api_token}}` (dynamic from input JSON)  
     - `Prefer`: `wait`  
   - Body Format: JSON  
   - JSON Body:  
     ```json
     {
       "version": "creativeathive/lemaar-door-urban:d7eff9d576b4f25f674adfab16f4dbdd061a0a87041daf5edf0273130c84477a",
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
   - Enable JSON response parsing, set `neverError` to true  
   - Connect Set Other Parameters → Create Other Prediction

5. **Create Code Node - "Log Request"**  
   - JavaScript code:  
     ```javascript
     const data = $input.all()[0].json;
     console.log('creativeathive/lemaar-door-urban Request:', {
       timestamp: new Date().toISOString(),
       prediction_id: data.id,
       model_type: 'other'
     });
     return $input.all();
     ```  
   - Connect Create Other Prediction → Log Request

6. **Create Wait Node - "Wait 5s"**  
   - Wait time: 5 seconds  
   - Connect Log Request → Wait 5s

7. **Create HTTP Request Node - "Check Status"**  
   - Method: GET  
   - URL: `https://api.replicate.com/v1/predictions/{{$('Create Other Prediction').item.json.id}}`  
   - Header: Authorization Bearer token from "Set API Token" node  
   - JSON response parsing enabled, `neverError` true  
   - Connect Wait 5s → Check Status  
   - Connect Wait 10s → Check Status (for retry loop)

8. **Create If Node - "Is Complete?"**  
   - Condition: `$json.status == "succeeded"`  
   - Connect Check Status → Is Complete?

9. **Create If Node - "Has Failed?"**  
   - Condition: `$json.status == "failed"`  
   - Connect Is Complete? (false branch) → Has Failed?

10. **Create Set Node - "Success Response"**  
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
    - Connect Is Complete? (true branch) → Success Response

11. **Create Set Node - "Error Response"**  
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
    - Connect Has Failed? (true branch) → Error Response

12. **Create Wait Node - "Wait 10s"**  
    - Wait time: 10 seconds  
    - Connect Has Failed? (false branch) → Wait 10s

13. **Create Set Node - "Display Result"**  
    - Assign input JSON `response` to `final_result` field  
    - Connect Success Response → Display Result  
    - Connect Error Response → Display Result

14. **Add Sticky Notes for Documentation**  
    - Sticky Note with contact info (Yaron@nofluff.online), YouTube and LinkedIn links  
    - Sticky Note with detailed model overview, parameter guide, usage instructions, troubleshooting, and external links to Replicate and n8n docs

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                 | Context or Link                                                                                          |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| For any questions or support, please contact: Yaron@nofluff.online. Explore more tips and tutorials on YouTube: https://www.youtube.com/@YaronBeen/videos and LinkedIn: https://www.linkedin.com/in/yaronbeen/                                                                                                                                                                                                                                | Sticky Note with contact and branding information                                                       |
| Model Documentation: https://replicate.com/creativeathive/lemaar-door-urban. Replicate API Documentation: https://replicate.com/docs. n8n Documentation: https://docs.n8n.io                                                                                                                                                                                                                                                                     | Sticky note detailed documentation section                                                             |
| Workflow is designed to handle asynchronous API calls with intelligent polling and retry logic to accommodate variable generation times. Parameters are extensively customizable for various use cases including inpainting, image-to-image, and prompt-based generation.                                                                                                                                                                       | General workflow design notes                                                                           |
| Replace `"YOUR_REPLICATE_API_TOKEN"` with a valid token from https://replicate.com/account to enable API calls. Ensure the token has sufficient quota and permissions.                                                                                                                                                                                                                                                                            | Credential setup note                                                                                   |
| Recommended to test initially with default parameters and simple prompts to validate connectivity and functionality before customization. Monitor logs for troubleshooting.                                                                                                                                                                                                                                                                    | Best practices                                                                                          |
| Safety checker can be disabled via parameter `disable_safety_checker` but default is enabled for content compliance.                                                                                                                                                                                                                                                                                                                        | Parameter caution note                                                                                   |

---

**Disclaimer:** The text above is exclusively derived from an automated n8n workflow created with n8n integration and automation tool. The processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly available.