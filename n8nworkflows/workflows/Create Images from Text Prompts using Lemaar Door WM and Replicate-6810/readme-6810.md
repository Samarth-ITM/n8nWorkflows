Create Images from Text Prompts using Lemaar Door WM and Replicate

https://n8nworkflows.xyz/workflows/create-images-from-text-prompts-using-lemaar-door-wm-and-replicate-6810


# Create Images from Text Prompts using Lemaar Door WM and Replicate

### 1. Workflow Overview

This workflow automates the generation of images from text prompts using the "lemaar-door-wm" AI model hosted on the Replicate platform. It is designed for creative image synthesis based on user-defined parameters, supporting features like image inpainting, custom seeds, multiple output formats, and advanced model options.

**Target Use Cases:**  
- Artists or designers automating image creation from textual descriptions.  
- Developers integrating AI-generated visuals into applications or pipelines.  
- Experimentation with advanced model parameters for customized image generation.

**Logical Blocks:**  
- **1.1 Input Reception and Parameter Setup:** Handles manual triggering, API token configuration, and parameter assignment for the AI model.  
- **1.2 AI Prediction Request:** Sends a generation request to Replicate’s API with configured parameters.  
- **1.3 Prediction Monitoring:** Waits and polls the Replicate API to check the prediction status until completion or failure.  
- **1.4 Response Handling:** Processes success and error responses into structured JSON outputs.  
- **1.5 Logging:** Records generation request details for monitoring and debugging.  
- **1.6 User Feedback and Final Output:** Prepares the final response for downstream consumption or display.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Parameter Setup

**Overview:**  
This block initiates the workflow manually, sets the Replicate API token, and configures all input parameters needed for the generation request.

**Nodes Involved:**  
- Manual Trigger  
- Set API Token  
- Set Other Parameters

**Node Details:**

- **Manual Trigger**  
  - *Type:* Trigger node  
  - *Role:* Starts workflow execution manually via UI interaction.  
  - *Config:* Default, no parameters.  
  - *Input:* None (trigger)  
  - *Output:* Triggers next node.  
  - *Edge Cases:* None; always ready on manual click.

- **Set API Token**  
  - *Type:* Set node  
  - *Role:* Stores the Replicate API token as a workflow variable.  
  - *Config:* Assigns a string parameter `api_token` with placeholder `"YOUR_REPLICATE_API_TOKEN"` (to be replaced with a valid token).  
  - *Input:* Output from Manual Trigger  
  - *Output:* Provides `api_token` for downstream nodes.  
  - *Edge Cases:* Failure if token is invalid or missing; no runtime validation here.

- **Set Other Parameters**  
  - *Type:* Set node  
  - *Role:* Defines all parameters for the AI image generation, including the token, prompt, model options, image dimensions, and advanced controls.  
  - *Config:* Assigns multiple fields like `mask`, `seed`, `image`, `model`, `width`, `height`, `prompt`, `go_fast`, `extra_lora`, `lora_scale`, `megapixels`, `num_outputs`, `aspect_ratio`, `output_format`, `guidance_scale`, `output_quality`, `prompt_strength`, `extra_lora_scale`, `replicate_weights`, `num_inference_steps`, `disable_safety_checker`. Uses expression to inherit the API token from the previous node.  
  - *Input:* Receives `api_token` from Set API Token node  
  - *Output:* Passes all parameters forward for the API request.  
  - *Edge Cases:* Incorrect or malformed parameter values may cause API request failure. Seed set to -1 by default means random seeding.

#### 2.2 AI Prediction Request

**Overview:**  
Sends a POST request to Replicate’s API to initiate the image generation with the configured parameters.

**Nodes Involved:**  
- Create Other Prediction

**Node Details:**

- **Create Other Prediction**  
  - *Type:* HTTP Request node  
  - *Role:* Submits the image generation request to Replicate’s prediction API endpoint.  
  - *Config:*  
    - Method: POST  
    - URL: `https://api.replicate.com/v1/predictions`  
    - Headers: Authorization Bearer token from `api_token` field; Prefer: wait (makes API respond only after prediction is complete if possible).  
    - JSON Body: Contains version ID of the model and all parameters dynamically filled from previous node’s JSON.  
    - Response: JSON, with `neverError` true to prevent node failure on HTTP errors.  
  - *Input:* Parameters from Set Other Parameters node  
  - *Output:* API response, including prediction `id` and initial status.  
  - *Edge Cases:*  
    - HTTP errors or rate limits may occur.  
    - Authorization failure if token invalid.  
    - Model or parameter validation errors from API.  
  - *Version-Specific:* Uses Replicate API v1 with model version ID embedded.

#### 2.3 Prediction Monitoring

**Overview:**  
Implements a loop with timed waits and status checks to poll the prediction status until the image generation completes or fails.

**Nodes Involved:**  
- Log Request  
- Wait 5s  
- Check Status  
- Is Complete?  
- Has Failed?  
- Wait 10s

**Node Details:**

- **Log Request**  
  - *Type:* Code node  
  - *Role:* Logs generation request details for monitoring.  
  - *Config:* Logs timestamp, prediction ID, and model type to console; returns input unchanged.  
  - *Input:* Response from Create Other Prediction  
  - *Output:* Passes data to Wait 5s node  
  - *Edge Cases:* Logging failures do not block workflow.

- **Wait 5s**  
  - *Type:* Wait node  
  - *Role:* Pauses the workflow for 5 seconds before the next status check.  
  - *Config:* Wait time set to 5 seconds.  
  - *Input:* From Log Request  
  - *Output:* Triggers Check Status node

- **Check Status**  
  - *Type:* HTTP Request node  
  - *Role:* Queries Replicate API for current prediction status by prediction ID.  
  - *Config:*  
    - Method: GET  
    - URL dynamically set to `https://api.replicate.com/v1/predictions/{{ prediction_id }}`  
    - Authorization header with Bearer token  
    - Response JSON, never error true  
  - *Input:* From Wait 5s or Wait 10s nodes  
  - *Output:* Passes status data to Is Complete? node

- **Is Complete?**  
  - *Type:* If node  
  - *Role:* Checks if prediction status is "succeeded".  
  - *Config:* Condition: `$json.status === "succeeded"`  
  - *Input:* Status from Check Status node  
  - *Output:*  
    - True branch: Success Response node  
    - False branch: Has Failed? node

- **Has Failed?**  
  - *Type:* If node  
  - *Role:* Checks if prediction status is "failed".  
  - *Config:* Condition: `$json.status === "failed"`  
  - *Input:* Status from Is Complete? node false branch  
  - *Output:*  
    - True branch: Error Response node  
    - False branch: Wait 10s node (retry polling)

- **Wait 10s**  
  - *Type:* Wait node  
  - *Role:* Pauses 10 seconds before retrying status check.  
  - *Config:* Wait time 10 seconds  
  - *Input:* From Has Failed? false branch  
  - *Output:* Loops back to Check Status node

**Edge Cases:**  
- API rate limits or timeouts during status check.  
- Prediction stuck in non-terminal states (e.g., "starting" or "processing") handled by loop.  
- Failure states handled explicitly.

#### 2.4 Response Handling

**Overview:**  
Formats the final success or error response into a structured JSON object for downstream use or display.

**Nodes Involved:**  
- Success Response  
- Error Response  
- Display Result

**Node Details:**

- **Success Response**  
  - *Type:* Set node  
  - *Role:* Creates a JSON object indicating success, including result URL, prediction ID, status, and message.  
  - *Config:* Sets `response` object with keys: `success: true`, `result_url`, `prediction_id`, `status`, `message` ("Other generated successfully").  
  - *Input:* From Is Complete? true branch  
  - *Output:* To Display Result node

- **Error Response**  
  - *Type:* Set node  
  - *Role:* Creates a JSON object indicating failure with error details, prediction ID, status, and message.  
  - *Config:* Sets `response` object with keys: `success: false`, `error` (from JSON or default), `prediction_id`, `status`, `message` ("Failed to generate other").  
  - *Input:* From Has Failed? true branch  
  - *Output:* To Display Result node

- **Display Result**  
  - *Type:* Set node  
  - *Role:* Assigns the final `response` JSON object to `final_result` for output.  
  - *Config:* Copies `response` field to `final_result`.  
  - *Input:* From Success Response or Error Response  
  - *Output:* Final output of workflow

**Edge Cases:**  
- Errors without detailed message default gracefully.  
- Output JSON format consistent for downstream parsing.

#### 2.5 Logging

**Overview:**  
Logs key request data to the console for traceability and debugging purposes.

**Nodes Involved:**  
- Log Request (already described in 2.3)

**Edge Cases:**  
- Logging errors do not interrupt workflow.

---

### 3. Summary Table

| Node Name             | Node Type          | Functional Role                          | Input Node(s)               | Output Node(s)         | Sticky Note                                                                                                                                            |
|-----------------------|--------------------|----------------------------------------|-----------------------------|------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------|
| Manual Trigger        | Manual Trigger      | Starts workflow execution manually     | None                        | Set API Token           | See Sticky Note4 for detailed workflow explanation and usage instructions.                                                                            |
| Set API Token         | Set                | Stores Replicate API token              | Manual Trigger              | Set Other Parameters    | See Sticky Note4 for detailed workflow explanation and usage instructions.                                                                            |
| Set Other Parameters  | Set                | Defines all input parameters for model | Set API Token               | Create Other Prediction | See Sticky Note4 for detailed workflow explanation and usage instructions.                                                                            |
| Create Other Prediction | HTTP Request       | Sends generation request to Replicate  | Set Other Parameters        | Log Request             | See Sticky Note4 for detailed workflow explanation and usage instructions.                                                                            |
| Log Request           | Code               | Logs request data for monitoring        | Create Other Prediction     | Wait 5s                 | See Sticky Note4 for detailed workflow explanation and usage instructions.                                                                            |
| Wait 5s               | Wait               | Waits 5 seconds before status check     | Log Request                 | Check Status            | See Sticky Note4 for detailed workflow explanation and usage instructions.                                                                            |
| Check Status          | HTTP Request       | Polls Replicate API for prediction status | Wait 5s, Wait 10s           | Is Complete?            | See Sticky Note4 for detailed workflow explanation and usage instructions.                                                                            |
| Is Complete?          | If                 | Checks if prediction succeeded          | Check Status                | Success Response, Has Failed? | See Sticky Note4 for detailed workflow explanation and usage instructions.                                                                            |
| Has Failed?           | If                 | Checks if prediction failed              | Is Complete?                | Error Response, Wait 10s | See Sticky Note4 for detailed workflow explanation and usage instructions.                                                                            |
| Wait 10s              | Wait               | Waits 10 seconds before retry            | Has Failed?                 | Check Status            | See Sticky Note4 for detailed workflow explanation and usage instructions.                                                                            |
| Success Response      | Set                | Builds success response object           | Is Complete? true branch    | Display Result          | See Sticky Note4 for detailed workflow explanation and usage instructions.                                                                            |
| Error Response        | Set                | Builds error response object             | Has Failed? true branch     | Display Result          | See Sticky Note4 for detailed workflow explanation and usage instructions.                                                                            |
| Display Result        | Set                | Prepares final output response           | Success Response, Error Response | None               | See Sticky Note4 for detailed workflow explanation and usage instructions.                                                                            |
| Sticky Note9          | Sticky Note        | Provides contact info and support links | None                        | None                   | Provides contact: Yaron@nofluff.online and links to YouTube and LinkedIn channels for support and tutorials.                                         |
| Sticky Note4          | Sticky Note        | Detailed workflow documentation and tips | None                        | None                   | Extensive explanation of model, parameters, workflow components, troubleshooting, and quick start instructions.                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - No parameters needed.  

2. **Create Set API Token Node**  
   - Type: Set  
   - Add a string field `api_token` with value `"YOUR_REPLICATE_API_TOKEN"` (replace with your actual token).  
   - Connect output of Manual Trigger to this node.

3. **Create Set Other Parameters Node**  
   - Type: Set  
   - Add the following fields (types and default values):  
     - `api_token` (string): Expression: `{{$node["Set API Token"].json.api_token}}`  
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
   - Connect from Set API Token node.

4. **Create HTTP Request Node "Create Other Prediction"**  
   - URL: `https://api.replicate.com/v1/predictions`  
   - Method: POST  
   - Headers:  
     - `Authorization`: `Bearer {{$json["api_token"]}}` (set using expression)  
     - `Prefer`: `wait`  
   - Body Format: JSON  
   - Body Content: JSON object with keys:  
     - `version`: `"creativeathive/lemaar-door-wm:5c5a8fdfe2e6d822912ac93f526e2d6ee9152ae5a9471032fb8d7a84a9ea607e"`  
     - `input`: all other parameters from previous node using expressions like `{{$json.mask}}`, `{{$json.seed}}`, etc.  
   - Response Format: JSON  
   - Options: Set "Never Error" to true to prevent node failure on HTTP errors.  
   - Connect from Set Other Parameters node.

5. **Create Code Node "Log Request"**  
   - JavaScript code:  
     ```javascript
     const data = $input.all()[0].json;
     console.log('creativeathive/lemaar-door-wm Request:', {
       timestamp: new Date().toISOString(),
       prediction_id: data.id,
       model_type: 'other'
     });
     return $input.all();
     ```  
   - Connect from Create Other Prediction node.

6. **Create Wait Node "Wait 5s"**  
   - Wait Time: 5 seconds  
   - Connect from Log Request node.

7. **Create HTTP Request Node "Check Status"**  
   - URL: `https://api.replicate.com/v1/predictions/{{$node["Create Other Prediction"].json.id}}` (dynamic)  
   - Method: GET  
   - Headers:  
     - `Authorization`: `Bearer {{$node["Set API Token"].json.api_token}}`  
   - Response Format: JSON  
   - Options: "Never Error" true  
   - Connect from Wait 5s node.

8. **Create If Node "Is Complete?"**  
   - Condition: Check if `$json.status === "succeeded"`  
   - Connect from Check Status node.

9. **Create If Node "Has Failed?"**  
   - Condition: Check if `$json.status === "failed"`  
   - Connect from Is Complete? node’s false output.

10. **Create Wait Node "Wait 10s"**  
    - Wait Time: 10 seconds  
    - Connect from Has Failed? node’s false output.

11. **Connect Wait 10s output back to Check Status node**  
    - Creates a polling loop.

12. **Create Set Node "Success Response"**  
    - Set field `response` to object:  
      ```json
      {
        "success": true,
        "result_url": "{{$json.output}}",
        "prediction_id": "{{$json.id}}",
        "status": "{{$json.status}}",
        "message": "Other generated successfully"
      }
      ```  
    - Connect from Is Complete? node’s true output.

13. **Create Set Node "Error Response"**  
    - Set field `response` to object:  
      ```json
      {
        "success": false,
        "error": "{{$json.error || 'Other generation failed'}}",
        "prediction_id": "{{$json.id}}",
        "status": "{{$json.status}}",
        "message": "Failed to generate other"
      }
      ```  
    - Connect from Has Failed? node’s true output.

14. **Create Set Node "Display Result"**  
    - Set field `final_result` to `{{$json.response}}`  
    - Connect from both Success Response and Error Response nodes.

15. **Optional: Add Sticky Notes**  
    - Add descriptive sticky notes containing contact info, workflow overview, parameter references, and helpful links as per Sticky Note9 and Sticky Note4 content.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 | Context or Link                                                                                              |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| Contact Yaron for support regarding this workflow at Yaron@nofluff.online. Additional tutorials and tips are available on YouTube and LinkedIn.                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | Contact info and support links in Sticky Note9: YouTube https://www.youtube.com/@YaronBeen/videos LinkedIn https://www.linkedin.com/in/yaronbeen/ |
| This workflow uses the Replicate API with the model "creativeathive/lemaar-door-wm" version `5c5a8fdfe2e6d822912ac93f526e2d6ee9152ae5a9471032fb8d7a84a9ea607e`. Ensure your API token corresponds to an account with access and sufficient credits.                                                                                                                                                                                                                                                                                                                                 | Model documentation: https://replicate.com/creativeathive/lemaar-door-wm Replicate API Docs: https://replicate.com/docs |
| The workflow includes built-in retry logic with waits of 5 and 10 seconds for robust handling of asynchronous prediction status updates. It is production-ready with structured JSON outputs, logging, and error handling.                                                                                                                                                                                                                                                                                                                                                                                                     | Detailed workflow explanation available in Sticky Note4.                                                     |
| Best practices include testing with default parameters first, securing API tokens, monitoring your Replicate usage, and adjusting parameters to your use case needs.                                                                                                                                                                                                                                                                                                                                                                                                                                                     | See troubleshooting and quick start instructions in Sticky Note4.                                           |

---

**Disclaimer:**  
The text provided is extracted exclusively from an n8n automated workflow. The workflow respects all applicable content policies and contains no illegal, offensive, or protected elements. All data processed is legal and publicly available.