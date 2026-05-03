Create Images from Text Prompts using Flash V2.0.0 Beta 9 and Replicate

https://n8nworkflows.xyz/workflows/create-images-from-text-prompts-using-flash-v2-0-0-beta-9-and-replicate-6866


# Create Images from Text Prompts using Flash V2.0.0 Beta 9 and Replicate

### 1. Workflow Overview

This n8n workflow automates the generation of images from text prompts using the AI model **flash-v2.0.0-beta.9** via the Replicate API. It is designed primarily for users who want to create images programmatically by submitting descriptive prompts and additional parameters to a sophisticated AI model.

The workflow is logically divided into the following blocks:

- **1.1 Input Setup and Initialization**: Receiving a manual trigger and setting the necessary API token and generation parameters.
- **1.2 Prediction Creation**: Sending a request to Replicate’s API to start the image generation process.
- **1.3 Status Monitoring and Retry Loop**: Periodically checking the status of the generation until completion or failure, with automated wait periods and retry logic.
- **1.4 Result Handling**: Distinguishing success from failure, formatting the output accordingly.
- **1.5 Logging and Final Output**: Logging request details for monitoring and preparing the final structured response.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Setup and Initialization

**Overview:**  
This block initializes the workflow execution via manual trigger and sets the API token and all necessary parameters required to instruct the AI model.

**Nodes Involved:**  
- Manual Trigger  
- Set API Token  
- Set Other Parameters

**Node Details:**

- **Manual Trigger**  
  - *Type*: Trigger node (manual)  
  - *Role*: Starts the workflow manually.  
  - *Config*: No parameters; user initiates workflow in n8n UI.  
  - *Input/Output*: No input; outputs to "Set API Token".  
  - *Failures*: None expected here.

- **Set API Token**  
  - *Type*: Set node  
  - *Role*: Stores the Replicate API token as a workflow variable.  
  - *Config*: Assigns the string `"YOUR_REPLICATE_API_TOKEN"` to `api_token`.  
  - *Input/Output*: Input from "Manual Trigger"; output to "Set Other Parameters".  
  - *Failures*: If token is invalid or missing, API calls will fail downstream.

- **Set Other Parameters**  
  - *Type*: Set node  
  - *Role*: Defines all other model input parameters (prompt, image URLs, mask, dimensions, etc.), including copying the API token from the previous node.  
  - *Config*:  
    - Copies `api_token` from "Set API Token".  
    - Sets default values for mask URL, seed, image URL, model, width, height, prompt, and various generation controls such as guidance scale, number of steps, output format, and safety checker flag.  
  - *Input/Output*: Input from "Set API Token"; output to "Create Other Prediction".  
  - *Failures*: Misconfigured or invalid parameters may cause generation errors.

---

#### 2.2 Prediction Creation

**Overview:**  
This block sends a POST request to the Replicate API to initiate the image generation based on the parameters set previously.

**Nodes Involved:**  
- Create Other Prediction  
- Log Request

**Node Details:**

- **Create Other Prediction**  
  - *Type*: HTTP Request node  
  - *Role*: Posts the generation request to Replicate’s `/v1/predictions` endpoint.  
  - *Config*:  
    - URL: `https://api.replicate.com/v1/predictions`  
    - Method: POST  
    - Headers: Authorization Bearer token from `api_token`, Prefer header set to "wait" (waits for prediction to complete before responding).  
    - Body: JSON including model version, and all input parameters interpolated from previous node’s JSON data.  
  - *Input/Output*: Input from "Set Other Parameters"; output to "Log Request".  
  - *Failures*: Possible HTTP errors, authorization failures, API limits, or invalid input data errors.

- **Log Request**  
  - *Type*: Code node (JavaScript)  
  - *Role*: Logs generation details to console for monitoring purposes.  
  - *Config*: Prints timestamp, prediction ID, and model type to n8n logs.  
  - *Input/Output*: Input from "Create Other Prediction"; output to "Wait 5s".  
  - *Failures*: None expected. Logging only.

---

#### 2.3 Status Monitoring and Retry Loop

**Overview:**  
Implements a polling loop to repeatedly check the prediction status until it is either succeeded or failed. Wait nodes introduce delays between status checks to manage API usage and avoid rate limits.

**Nodes Involved:**  
- Wait 5s  
- Check Status  
- Is Complete?  
- Has Failed?  
- Wait 10s

**Node Details:**

- **Wait 5s**  
  - *Type*: Wait node  
  - *Role*: Pauses workflow for 5 seconds before status checking.  
  - *Config*: Wait time = 5 seconds.  
  - *Input/Output*: Input from "Log Request"; output to "Check Status".  
  - *Failures*: None expected.

- **Check Status**  
  - *Type*: HTTP Request node  
  - *Role*: GET request to Replicate API for prediction status.  
  - *Config*:  
    - URL: `https://api.replicate.com/v1/predictions/{{ prediction_id }}` where `prediction_id` is from "Create Other Prediction" node.  
    - Headers: Authorization Bearer token from "Set API Token".  
  - *Input/Output*: Input from "Wait 5s" or "Wait 10s"; output to "Is Complete?".  
  - *Failures*: HTTP errors, expired tokens, or network issues.

- **Is Complete?**  
  - *Type*: If node  
  - *Role*: Checks if prediction status equals `"succeeded"`.  
  - *Config*: Condition: `{{ $json.status }} === 'succeeded'`  
  - *Input/Output*: Input from "Check Status"; outputs:  
    - If true: to "Success Response"  
    - If false: to "Has Failed?"  
  - *Failures*: Expression evaluation issues if `status` field is missing.

- **Has Failed?**  
  - *Type*: If node  
  - *Role*: Checks if prediction status equals `"failed"`.  
  - *Config*: Condition: `{{ $json.status }} === 'failed'`  
  - *Input/Output*: Input from "Is Complete?" (false branch); outputs:  
    - If true: to "Error Response"  
    - If false: to "Wait 10s" (retry loop)  
  - *Failures*: Same as above for expression issues.

- **Wait 10s**  
  - *Type*: Wait node  
  - *Role*: Pauses workflow for 10 seconds before retrying status check.  
  - *Config*: Wait time = 10 seconds.  
  - *Input/Output*: Input from "Has Failed?" (false branch); output to "Check Status" (loop back).  
  - *Failures*: None expected.

---

#### 2.4 Result Handling

**Overview:**  
Once the generation is complete or failed, this block formats the output into a structured JSON response for downstream consumption or user display.

**Nodes Involved:**  
- Success Response  
- Error Response  
- Display Result

**Node Details:**

- **Success Response**  
  - *Type*: Set node  
  - *Role*: Constructs a JSON object signaling success with generated image URL and metadata.  
  - *Config*:  
    - Assigns `response` object with keys: `success: true`, `result_url` (image output URL), `prediction_id`, `status`, and a success message.  
  - *Input/Output*: Input from "Is Complete?" (true branch); output to "Display Result".  
  - *Failures*: None expected unless input data missing.

- **Error Response**  
  - *Type*: Set node  
  - *Role*: Constructs a JSON object signaling failure with error details.  
  - *Config*:  
    - Assigns `response` object with keys: `success: false`, `error` message, `prediction_id`, status, and failure message.  
    - Falls back to generic message if no specific error provided.  
  - *Input/Output*: Input from "Has Failed?" (true branch); output to "Display Result".  
  - *Failures*: None expected unless input malformed.

- **Display Result**  
  - *Type*: Set node  
  - *Role*: Finalizes and exposes the `final_result` object for output or further processing.  
  - *Config*: Copies the `response` object to `final_result`.  
  - *Input/Output*: Input from both "Success Response" and "Error Response"; output is terminal.  
  - *Failures*: None expected.

---

#### 2.5 Logging and Final Output

**Overview:**  
Logs all prediction requests for audit and debugging purposes, facilitating monitoring of usage and troubleshooting.

**Nodes Involved:**  
- Log Request (already detailed in 2.2)

---

### 3. Summary Table

| Node Name              | Node Type             | Functional Role                      | Input Node(s)             | Output Node(s)               | Sticky Note                                                                                   |
|------------------------|-----------------------|------------------------------------|---------------------------|-----------------------------|-----------------------------------------------------------------------------------------------|
| Manual Trigger         | Manual Trigger        | Starts workflow                    | —                         | Set API Token                |                                                                                               |
| Set API Token          | Set                   | Stores API token                   | Manual Trigger             | Set Other Parameters         |                                                                                               |
| Set Other Parameters   | Set                   | Defines model input parameters     | Set API Token              | Create Other Prediction      |                                                                                               |
| Create Other Prediction| HTTP Request          | Sends generation request           | Set Other Parameters       | Log Request                 |                                                                                               |
| Log Request            | Code                  | Logs generation details            | Create Other Prediction    | Wait 5s                      |                                                                                               |
| Wait 5s                | Wait                  | Waits 5 seconds before status check| Log Request                | Check Status                 |                                                                                               |
| Check Status           | HTTP Request          | Fetches prediction status          | Wait 5s, Wait 10s          | Is Complete?                 |                                                                                               |
| Is Complete?           | If                    | Checks if status = succeeded       | Check Status               | Success Response, Has Failed?|                                                                                               |
| Has Failed?            | If                    | Checks if status = failed          | Is Complete?               | Error Response, Wait 10s     |                                                                                               |
| Wait 10s               | Wait                  | Waits 10 seconds before retry      | Has Failed?                | Check Status                 |                                                                                               |
| Success Response       | Set                   | Formats success output             | Is Complete?               | Display Result               |                                                                                               |
| Error Response         | Set                   | Formats error output               | Has Failed?                | Display Result               |                                                                                               |
| Display Result         | Set                   | Final output object                | Success Response, Error Response | —                     |                                                                                               |
| Sticky Note9           | Sticky Note           | Contact/support info               | —                         | —                           | =======================================<br>FLASH-V2.0.0-BETA.9 GENERATOR<br>For questions: Yaron@nofluff.online<br>YouTube: https://www.youtube.com/@YaronBeen/videos<br>LinkedIn: https://www.linkedin.com/in/yaronbeen/ |
| Sticky Note4           | Sticky Note           | Detailed workflow and model notes | —                         | —                           | Detailed model overview, parameter reference, workflow explanation, quick start, troubleshooting, and resource links (full content as provided). |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node**  
   - No parameters needed. This node starts the workflow manually.

2. **Add a Set node named "Set API Token"**  
   - Create a string field `api_token` with value `"YOUR_REPLICATE_API_TOKEN"` (replace with your actual token).  
   - Connect output of "Manual Trigger" to this node.

3. **Add a Set node named "Set Other Parameters"**  
   - Create fields with these types and default values:  
     - `api_token` (string) = `={{ $('Set API Token').item.json.api_token }}` (copies from previous node)  
     - `mask` (string) = `"https://via.placeholder.com/512x512/000000/FFFFFF.png"`  
     - `seed` (number) = `-1`  
     - `image` (string) = `"https://picsum.photos/512/512"`  
     - `model` (string) = `"dev"`  
     - `width` (number) = `512`  
     - `height` (number) = `512`  
     - `prompt` (string) = `"Create something amazing"`  
     - `go_fast` (boolean) = `false`  
     - `extra_lora` (string) = `""`  
     - `lora_scale` (number) = `1`  
     - `megapixels` (string) = `"1"`  
     - `num_outputs` (number) = `1`  
     - `aspect_ratio` (string) = `"1:1"`  
     - `output_format` (string) = `"webp"`  
     - `guidance_scale` (number) = `3`  
     - `output_quality` (number) = `80`  
     - `prompt_strength` (number) = `0.8`  
     - `extra_lora_scale` (number) = `1`  
     - `replicate_weights` (string) = `""`  
     - `num_inference_steps` (number) = `28`  
     - `disable_safety_checker` (boolean) = `false`  
   - Connect output of "Set API Token" to this node.

4. **Add an HTTP Request node named "Create Other Prediction"**  
   - Method: POST  
   - URL: `https://api.replicate.com/v1/predictions`  
   - Authentication: None (use header for token)  
   - Headers:  
     - `Authorization`: `Bearer {{ $json.api_token }}`  
     - `Prefer`: `wait`  
   - Body (JSON):  
   ```json
   {
     "version": "settyan/flash-v2.0.0-beta.9:3053b0f8f3cff9282f18eb7782feb7b8d8b2e8dffb256d316cab1225a2cf3dad",
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

5. **Add a Code node named "Log Request"**  
   - JavaScript code:  
   ```javascript
   const data = $input.all()[0].json;
   console.log('settyan/flash-v2.0.0-beta.9 Request:', {
     timestamp: new Date().toISOString(),
     prediction_id: data.id,
     model_type: 'other'
   });
   return $input.all();
   ```  
   - Connect output of "Create Other Prediction" to this node.

6. **Add a Wait node named "Wait 5s"**  
   - Wait for 5 seconds.  
   - Connect output of "Log Request" to this node.

7. **Add an HTTP Request node named "Check Status"**  
   - Method: GET  
   - URL: `https://api.replicate.com/v1/predictions/{{ $('Create Other Prediction').item.json.id }}`  
   - Headers:  
     - `Authorization`: `Bearer {{ $('Set API Token').item.json.api_token }}`  
   - Connect output of "Wait 5s" and "Wait 10s" to this node (for looping).

8. **Add an If node named "Is Complete?"**  
   - Condition: `{{ $json.status }} === 'succeeded'`  
   - If true: connect to "Success Response"  
   - If false: connect to "Has Failed?"  
   - Connect output of "Check Status" to this node.

9. **Add an If node named "Has Failed?"**  
   - Condition: `{{ $json.status }} === 'failed'`  
   - If true: connect to "Error Response"  
   - If false: connect to "Wait 10s"  
   - Connect output (false branch) of "Is Complete?" to this node.

10. **Add a Wait node named "Wait 10s"**  
    - Wait for 10 seconds.  
    - Connect output of "Has Failed?" (false branch) to this node.  
    - Connect output of this node back to "Check Status" to create a polling loop.

11. **Add a Set node named "Success Response"**  
    - Create an object field `response` with:  
      ```json
      {
        "success": true,
        "result_url": "{{ $json.output }}",
        "prediction_id": "{{ $json.id }}",
        "status": "{{ $json.status }}",
        "message": "Other generated successfully"
      }
      ```  
    - Connect output of "Is Complete?" (true branch) to this node.

12. **Add a Set node named "Error Response"**  
    - Create an object field `response` with:  
      ```json
      {
        "success": false,
        "error": "{{ $json.error || 'Other generation failed' }}",
        "prediction_id": "{{ $json.id }}",
        "status": "{{ $json.status }}",
        "message": "Failed to generate other"
      }
      ```  
    - Connect output of "Has Failed?" (true branch) to this node.

13. **Add a Set node named "Display Result"**  
    - Create an object field `final_result` assigned from `response` field.  
    - Connect outputs of "Success Response" and "Error Response" to this node.

14. **Add Sticky Notes (optional)** for documentation and support info as per original workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                | Context or Link                                                                                                             |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------|
| FLASH-V2.0.0-BETA.9 GENERATOR contact: Yaron@nofluff.online, YouTube: https://www.youtube.com/@YaronBeen/videos, LinkedIn: https://www.linkedin.com/in/yaronbeen/ | Support and community resources for the model and workflow                                                                  |
| Detailed workflow description with parameter guide, troubleshooting tips, quick start instructions, and links to Replicate and n8n documentation included. | Sticky note content within the workflow provides a comprehensive reference for users and developers                         |
| Model documentation: https://replicate.com/settyan/flash-v2.0.0-beta.9                                                                                      | Official model page with details on usage and parameters                                                                     |
| Replicate API docs: https://replicate.com/docs                                                                                                              | REST API specification and examples for interacting with Replicate                                                          |
| n8n documentation: https://docs.n8n.io                                                                                                                     | Official n8n user and developer guides                                                                                        |

---

**Disclaimer:** The provided text and workflow originate exclusively from an automated process created with n8n, a tool for integration and automation. All processing strictly complies with content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.