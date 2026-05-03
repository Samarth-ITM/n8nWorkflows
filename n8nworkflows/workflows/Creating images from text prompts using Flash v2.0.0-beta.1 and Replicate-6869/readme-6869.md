Creating images from text prompts using Flash v2.0.0-beta.1 and Replicate

https://n8nworkflows.xyz/workflows/creating-images-from-text-prompts-using-flash-v2-0-0-beta-1-and-replicate-6869


# Creating images from text prompts using Flash v2.0.0-beta.1 and Replicate

### 1. Workflow Overview

This n8n workflow automates the process of generating images from text prompts using the AI model **Flash v2.0.0-beta.1** hosted on the **Replicate** platform. It is designed to take user input parameters, send a generation request to the Replicate API, monitor the asynchronous processing status, and finally output the generated images or error information in a structured format.

The workflow logically divides into the following blocks:

- **1.1 Input Initialization:** Receives manual trigger and sets up authentication and generation parameters.
- **1.2 AI Image Generation Request:** Sends the generation request to Replicate API with all configured parameters.
- **1.3 Status Polling Loop:** Waits and repeatedly checks the generation status until completion or failure.
- **1.4 Success and Error Handling:** Branches based on generation outcome to return success or error responses.
- **1.5 Logging and Final Output:** Logs request details for monitoring and prepares the final output for further use.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Initialization

**Overview:**  
This block starts the workflow manually and prepares all necessary input parameters, including the API token and model-specific generation parameters, to be used in the subsequent API call.

**Nodes Involved:**  
- Manual Trigger  
- Set API Token  
- Set Other Parameters  

**Node Details:**  

- **Manual Trigger**  
  - *Type:* Trigger node  
  - *Purpose:* Initiates the workflow manually.  
  - *Configuration:* Default, no parameters set.  
  - *Inputs:* None  
  - *Outputs:* Connects to "Set API Token".  
  - *Edge Cases:* None expected.

- **Set API Token**  
  - *Type:* Set node  
  - *Purpose:* Stores the Replicate API token.  
  - *Configuration:* Assigns a string variable `api_token` with placeholder `"YOUR_REPLICATE_API_TOKEN"` to be replaced with a valid token.  
  - *Inputs:* From Manual Trigger  
  - *Outputs:* Connects to "Set Other Parameters".  
  - *Edge Cases:* Failure if token is invalid or missing; token must be replaced before use.

- **Set Other Parameters**  
  - *Type:* Set node  
  - *Purpose:* Sets all parameters needed for the image generation model.  
  - *Configuration:*  
    - Copies `api_token` from "Set API Token" node.  
    - Defines parameters such as `mask`, `seed`, `image`, `model`, `width`, `height`, `prompt`, `go_fast`, `extra_lora`, `lora_scale`, `megapixels`, `num_outputs`, `aspect_ratio`, `output_format`, `guidance_scale`, `output_quality`, `prompt_strength`, `extra_lora_scale`, `replicate_weights`, `num_inference_steps`, and `disable_safety_checker`.  
    - Default values are provided, such as prompt `"Create something amazing"`, image placeholder URL, and dimensions 512x512.  
  - *Inputs:* From Set API Token  
  - *Outputs:* Connects to "Create Other Prediction".  
  - *Edge Cases:* Wrong parameter types or missing required parameters (especially `prompt`) can cause API errors.

---

#### 2.2 AI Image Generation Request

**Overview:**  
Sends a POST request to the Replicate API to start the image generation with configured parameters. This node triggers the model inference and returns a prediction ID for tracking.

**Nodes Involved:**  
- Create Other Prediction  

**Node Details:**  

- **Create Other Prediction**  
  - *Type:* HTTP Request node  
  - *Purpose:* Calls Replicate's prediction API endpoint to create a new image generation job.  
  - *Configuration:*  
    - URL: `https://api.replicate.com/v1/predictions`  
    - Method: POST  
    - Headers: Authorization with `Bearer` token from `api_token` variable; `Prefer: wait` header to wait for immediate response.  
    - JSON Body: Includes model version ID and all parameters dynamically pulled from previous node JSON, leveraging n8n expressions (e.g., `{{$json.prompt}}`).  
    - Response set to never error and expects JSON format.  
  - *Inputs:* From Set Other Parameters  
  - *Outputs:* Connects to "Log Request".  
  - *Edge Cases:*  
    - Authentication errors if token is invalid.  
    - API rate limits or quota exceeded.  
    - Invalid parameters causing API rejection.  
    - Network issues or timeouts.

---

#### 2.3 Status Polling Loop

**Overview:**  
After submitting the generation job, this block waits and repeatedly checks the job status until it is either succeeded or failed. It implements a retry mechanism with wait times of 5 and 10 seconds.

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
  - *Purpose:* Logs prediction details (timestamp, prediction ID, model type) for monitoring/debugging.  
  - *Configuration:* JavaScript code logs to console and passes data unchanged.  
  - *Inputs:* From Create Other Prediction  
  - *Outputs:* Connects to "Wait 5s".  
  - *Edge Cases:* None critical, logging failure only affects monitoring.

- **Wait 5s**  
  - *Type:* Wait node  
  - *Purpose:* Pauses execution for 5 seconds before checking status.  
  - *Configuration:* Wait for 5 seconds.  
  - *Inputs:* From Log Request  
  - *Outputs:* Connects to "Check Status".  
  - *Edge Cases:* None significant.

- **Check Status**  
  - *Type:* HTTP Request node  
  - *Purpose:* Queries the Replicate API for prediction status using the prediction ID.  
  - *Configuration:*  
    - GET request to `https://api.replicate.com/v1/predictions/{{ prediction_id }}`  
    - Authorization header with API token.  
    - Returns JSON with status.  
  - *Inputs:* From Wait 5s and Wait 10s (loop)  
  - *Outputs:* Connects to "Is Complete?".  
  - *Edge Cases:*  
    - Network failures or invalid ID.  
    - API rate limits.

- **Is Complete?**  
  - *Type:* If node  
  - *Purpose:* Checks if the prediction status equals `"succeeded"`.  
  - *Configuration:* Condition `status == "succeeded"`.  
  - *Inputs:* From Check Status  
  - *Outputs:*  
    - True: Connects to "Success Response".  
    - False: Connects to "Has Failed?".  
  - *Edge Cases:* None.

- **Has Failed?**  
  - *Type:* If node  
  - *Purpose:* Checks if the prediction status equals `"failed"`.  
  - *Configuration:* Condition `status == "failed"`.  
  - *Inputs:* From Is Complete? (False branch)  
  - *Outputs:*  
    - True: Connects to "Error Response".  
    - False: Connects to "Wait 10s" (retry loop).  
  - *Edge Cases:* None.

- **Wait 10s**  
  - *Type:* Wait node  
  - *Purpose:* Delays 10 seconds before retrying status check.  
  - *Configuration:* Wait for 10 seconds.  
  - *Inputs:* From Has Failed? (False branch)  
  - *Outputs:* Connects back to "Check Status" for polling.  
  - *Edge Cases:* None.

---

#### 2.4 Success and Error Handling

**Overview:**  
Handles the final response based on whether the generation succeeded or failed, formatting output objects accordingly.

**Nodes Involved:**  
- Success Response  
- Error Response  
- Display Result  

**Node Details:**  

- **Success Response**  
  - *Type:* Set node  
  - *Purpose:* Formats success output including result URL, prediction ID, status, and message.  
  - *Configuration:* Sets an object `response` with keys: `success: true`, `result_url` (output image URL), `prediction_id`, `status`, and a success message.  
  - *Inputs:* From Is Complete? (True branch)  
  - *Outputs:* Connects to "Display Result".  
  - *Edge Cases:* Output depends on correct presence of `output` URL from API.

- **Error Response**  
  - *Type:* Set node  
  - *Purpose:* Formats error output including error message, prediction ID, and status.  
  - *Configuration:* Sets an object `response` with keys: `success: false`, `error` (or default message), `prediction_id`, `status`, and an error message.  
  - *Inputs:* From Has Failed? (True branch)  
  - *Outputs:* Connects to "Display Result".  
  - *Edge Cases:* Handles cases where `$json.error` may be missing.

- **Display Result**  
  - *Type:* Set node  
  - *Purpose:* Prepares the final output object under `final_result` key for further use or webhook response.  
  - *Configuration:* Copies `response` object to `final_result`.  
  - *Inputs:* From Success Response and Error Response  
  - *Outputs:* Terminal node (workflow end).  
  - *Edge Cases:* None.

---

#### 2.5 Logging and Documentation

**Overview:**  
Contains sticky notes with detailed documentation and contact information for user reference.

**Nodes Involved:**  
- Sticky Note4  
- Sticky Note9  

**Node Details:**  

- **Sticky Note9**  
  - *Type:* Sticky Note  
  - *Purpose:* Displays branding and support contact info, including YouTube and LinkedIn links for tutorials and tips.  
  - *Content Highlights:*  
    ```
    FLASH-V2.0.0-BETA.1 GENERATOR
    For any questions or support, please contact:
        Yaron@nofluff.online

    Explore more tips and tutorials here:
       - YouTube: https://www.youtube.com/@YaronBeen/videos
       - LinkedIn: https://www.linkedin.com/in/yaronbeen/
    ```
  - *Inputs/Outputs:* None.

- **Sticky Note4**  
  - *Type:* Sticky Note  
  - *Purpose:* Provides an extensive explanation of the workflow, model parameters, instructions, troubleshooting guide, and resource links.  
  - *Content Highlights:*  
    - Model overview and API endpoint.  
    - Parameter reference with required and optional parameters.  
    - Step-by-step workflow components explanation.  
    - Quick start instructions for API token and usage.  
    - Troubleshooting guide.  
    - Useful links to model docs, Replicate API, and n8n documentation.  
  - *Inputs/Outputs:* None.

---

### 3. Summary Table

| Node Name           | Node Type           | Functional Role                         | Input Node(s)           | Output Node(s)            | Sticky Note                                                                                                       |
|---------------------|---------------------|---------------------------------------|-------------------------|---------------------------|------------------------------------------------------------------------------------------------------------------|
| Manual Trigger      | Trigger             | Starts workflow execution              | None                    | Set API Token             |                                                                                                                  |
| Set API Token       | Set                 | Stores Replicate API token             | Manual Trigger          | Set Other Parameters      |                                                                                                                  |
| Set Other Parameters| Set                 | Sets all generation parameters         | Set API Token           | Create Other Prediction   |                                                                                                                  |
| Create Other Prediction | HTTP Request      | Sends generation request to Replicate API | Set Other Parameters   | Log Request               |                                                                                                                  |
| Log Request         | Code                | Logs prediction details for monitoring | Create Other Prediction | Wait 5s                   |                                                                                                                  |
| Wait 5s             | Wait                | Waits 5 seconds before status check    | Log Request             | Check Status              |                                                                                                                  |
| Check Status        | HTTP Request        | Checks prediction status from API      | Wait 5s, Wait 10s       | Is Complete?              |                                                                                                                  |
| Is Complete?        | If                  | Checks if prediction succeeded         | Check Status            | Success Response, Has Failed? |                                                                                                              |
| Has Failed?         | If                  | Checks if prediction failed             | Is Complete?            | Error Response, Wait 10s  |                                                                                                                  |
| Wait 10s            | Wait                | Waits 10 seconds before retrying       | Has Failed?             | Check Status              |                                                                                                                  |
| Success Response    | Set                 | Formats success output                  | Is Complete? (True)     | Display Result            |                                                                                                                  |
| Error Response      | Set                 | Formats error output                    | Has Failed? (True)      | Display Result            |                                                                                                                  |
| Display Result      | Set                 | Prepares final workflow output         | Success Response, Error Response | None              |                                                                                                                  |
| Sticky Note9        | Sticky Note         | Branding and support contact info      | None                    | None                      | FLASH-V2.0.0-BETA.1 GENERATOR contact info and social links.                                                     |
| Sticky Note4        | Sticky Note         | Detailed workflow and model documentation | None                  | None                      | Extended explanation, parameter guide, quick start, troubleshooting, and useful links.                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node:**  
   - No parameters needed. This node will start the workflow manually.

2. **Add a Set node named "Set API Token":**  
   - Create a string variable `api_token`.  
   - Set its value to `"YOUR_REPLICATE_API_TOKEN"` (replace this placeholder with your actual token before running).  
   - Connect output of Manual Trigger to this node.

3. **Add a Set node named "Set Other Parameters":**  
   - Assign `api_token` from previous node using expression `={{ $('Set API Token').item.json.api_token }}`.  
   - Define all other parameters with these default values (adjust as needed):  
     - `mask`: `"https://via.placeholder.com/512x512/000000/FFFFFF.png"`  
     - `seed`: `-1` (integer)  
     - `image`: `"https://picsum.photos/512/512"`  
     - `model`: `"dev"`  
     - `width`: `512` (integer)  
     - `height`: `512` (integer)  
     - `prompt`: `"Create something amazing"`  
     - `go_fast`: `false` (boolean)  
     - `extra_lora`: `""` (string)  
     - `lora_scale`: `1` (number)  
     - `megapixels`: `"1"` (string)  
     - `num_outputs`: `1` (integer)  
     - `aspect_ratio`: `"1:1"`  
     - `output_format`: `"webp"`  
     - `guidance_scale`: `3` (number)  
     - `output_quality`: `80` (number)  
     - `prompt_strength`: `0.8` (number)  
     - `extra_lora_scale`: `1` (number)  
     - `replicate_weights`: `""` (string)  
     - `num_inference_steps`: `28` (integer)  
     - `disable_safety_checker`: `false` (boolean)  
   - Connect output of "Set API Token" to this node.

4. **Add an HTTP Request node named "Create Other Prediction":**  
   - Method: POST  
   - URL: `https://api.replicate.com/v1/predictions`  
   - Headers:  
     - `Authorization`: `=Bearer {{ $json.api_token }}`  
     - `Prefer`: `wait`  
   - Body Content-Type: JSON  
   - JSON Body:  
     ```json
     {
       "version": "settyan/flash-v2.0.0-beta.1:bf2e5e495b5cc32573257fe8ded8fa100321c51c27ed7327dca2176a9b3284ce",
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
   - Connect output of "Set Other Parameters" to this node.

5. **Add a Code node named "Log Request":**  
   - JavaScript code:  
     ```javascript
     const data = $input.all()[0].json;
     console.log('settyan/flash-v2.0.0-beta.1 Request:', {
       timestamp: new Date().toISOString(),
       prediction_id: data.id,
       model_type: 'other'
     });
     return $input.all();
     ```  
   - Connect output of "Create Other Prediction" to this node.

6. **Add a Wait node named "Wait 5s":**  
   - Wait time: 5 seconds  
   - Connect output of "Log Request" to this node.

7. **Add an HTTP Request node named "Check Status":**  
   - Method: GET  
   - URL: `https://api.replicate.com/v1/predictions/{{ $('Create Other Prediction').item.json.id }}` (or dynamically from current item)  
   - Headers:  
     - `Authorization`: `=Bearer {{ $('Set API Token').item.json.api_token }}`  
   - Connect output of "Wait 5s" and also "Wait 10s" (later) to this node.

8. **Add an If node named "Is Complete?":**  
   - Condition: Check if `status == "succeeded"` (loose validation)  
   - Connect output of "Check Status" to this node.  
   - True branch connects to "Success Response".  
   - False branch connects to "Has Failed?".

9. **Add an If node named "Has Failed?":**  
   - Condition: Check if `status == "failed"`  
   - Connect false branch to "Wait 10s" node (next step).  
   - True branch connects to "Error Response".

10. **Add a Wait node named "Wait 10s":**  
    - Wait time: 10 seconds  
    - Connect output of "Has Failed?" (false) to this node.  
    - Connect output of this node back to "Check Status" (creating a polling loop).

11. **Add a Set node named "Success Response":**  
    - Create an object `response` with keys:  
      - `success: true`  
      - `result_url`: `$json.output` (image URL)  
      - `prediction_id`: `$json.id`  
      - `status`: `$json.status`  
      - `message`: `"Other generated successfully"`  
    - Connect true branch of "Is Complete?" to this node.

12. **Add a Set node named "Error Response":**  
    - Create an object `response` with keys:  
      - `success: false`  
      - `error`: `$json.error` or default `"Other generation failed"`  
      - `prediction_id`: `$json.id`  
      - `status`: `$json.status`  
      - `message`: `"Failed to generate other"`  
    - Connect true branch of "Has Failed?" to this node.

13. **Add a Set node named "Display Result":**  
    - Assign `final_result` to the `response` object from either "Success Response" or "Error Response".  
    - Connect outputs of both "Success Response" and "Error Response" to this node.

---

### 5. General Notes & Resources

| Note Content                                                                                                                     | Context or Link                                                                                       |
|---------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| FLASH-V2.0.0-BETA.1 GENERATOR contact and support: Yaron@nofluff.online                                                         | Sticky Note9: provides support contact and social media links.                                      |
| Explore tutorials and tips: YouTube: https://www.youtube.com/@YaronBeen/videos, LinkedIn: https://www.linkedin.com/in/yaronbeen/ | Sticky Note9                                                                                         |
| Detailed parameter guide, troubleshooting, and workflow overview available in Sticky Note4                                       | Comprehensive explanation of model parameters, workflow steps, and best practices for usage.        |
| Model Documentation: https://replicate.com/settyan/flash-v2.0.0-beta.1                                                            | Useful for understanding model capabilities and limits.                                             |
| Replicate API Documentation: https://replicate.com/docs                                                                           | Official API docs for authentication, request format, and error codes.                              |
| n8n Official Documentation: https://docs.n8n.io                                                                                   | General n8n node usage, expressions, and workflow design principles.                                |

---

This completes the comprehensive reference document for the "Creating images from text prompts using Flash v2.0.0-beta.1 and Replicate" workflow. It fully describes the logic, nodes, parameters, reproduction steps, and contextual notes for advanced users or AI agents to understand, reproduce, or extend it.