Create Images from Text Prompts using HeraNathalie Model and Replicate

https://n8nworkflows.xyz/workflows/create-images-from-text-prompts-using-heranathalie-model-and-replicate-6852


# Create Images from Text Prompts using HeraNathalie Model and Replicate

### 1. Workflow Overview

This workflow enables automated generation of images from text prompts using the HeraNathalie model hosted on the Replicate API platform. It is designed for users who want to convert descriptive text inputs into AI-generated images with configurable parameters such as resolution, style, and output format. The workflow handles the entire process from trigger initiation, parameter setup, API request submission, monitoring prediction status, to delivering a success or error response.

**Logical Blocks:**  
- **1.1 Trigger and Authentication Setup:** Manual initiation and API token configuration.  
- **1.2 Parameter Preparation:** Setting all input parameters required by the HeraNathalie model.  
- **1.3 Prediction Request:** Sending the generation request to Replicate API.  
- **1.4 Prediction Monitoring:** Polling the API to check prediction status with wait intervals and conditional loops.  
- **1.5 Result Handling:** Delivering success or error responses based on prediction outcome.  
- **1.6 Logging:** Recording request details for monitoring and debugging.  
- **1.7 User Information:** Sticky notes providing documentation, instructions, and contact info.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger and Authentication Setup

- **Overview:** Starts the workflow manually and sets the required API token for Replicate authentication.  
- **Nodes Involved:**  
  - Manual Trigger  
  - Set API Token  

- **Node Details:**  
  - **Manual Trigger**  
    - Type: Trigger node initiating workflow execution manually.  
    - Configuration: No parameters; simply starts the workflow on user request.  
    - Inputs: None  
    - Outputs: Connects to "Set API Token"  
    - Edge Cases: None. User must trigger manually.

  - **Set API Token**  
    - Type: Set node assigning the Replicate API token to a variable.  
    - Configuration: Holds a placeholder string `"YOUR_REPLICATE_API_TOKEN"` to be replaced by the user’s actual token.  
    - Inputs: From Manual Trigger  
    - Outputs: To "Set Other Parameters"  
    - Edge Cases: Failure if token is missing, invalid, or expired. Must be securely stored and replaced before running.

#### 2.2 Parameter Preparation

- **Overview:** Configures all input parameters for the model invocation, including both required and optional settings.  
- **Nodes Involved:**  
  - Set Other Parameters  

- **Node Details:**  
  - **Set Other Parameters**  
    - Type: Set node defining input parameters for the model call.  
    - Configuration:  
      - Copies the API token from previous node.  
      - Sets default values for: mask URL, random seed, input image URL, model variant, image dimensions, prompt text, speed flag, LoRA options, megapixels, number of outputs, aspect ratio, output format, guidance scale, output quality, prompt strength, inference steps, and safety checker flag.  
      - Example prompt default: `"Create something amazing"`  
    - Inputs: From "Set API Token"  
    - Outputs: To "Create Other Prediction"  
    - Edge Cases: Invalid parameter values may cause API errors or unexpected model behavior.

#### 2.3 Prediction Request

- **Overview:** Sends a POST request to Replicate API to create a new prediction job with the specified parameters.  
- **Nodes Involved:**  
  - Create Other Prediction  

- **Node Details:**  
  - **Create Other Prediction**  
    - Type: HTTP Request node (POST)  
    - Configuration:  
      - URL: `https://api.replicate.com/v1/predictions`  
      - Headers: Authorization using Bearer token, and Prefer header with value "wait" (to wait for prediction to finish if possible).  
      - JSON body: Includes model version ID and all input parameters from "Set Other Parameters" node, injected dynamically using expressions.  
      - Response format: JSON, never error on HTTP errors to handle gracefully.  
    - Inputs: From "Set Other Parameters"  
    - Outputs: To "Log Request"  
    - Edge Cases: Network errors, invalid token, malformed body, API downtime, or quota limits.

#### 2.4 Prediction Monitoring

- **Overview:** Implements a polling loop with wait intervals to check prediction status until completion or failure.  
- **Nodes Involved:**  
  - Log Request  
  - Wait 5s  
  - Check Status  
  - Is Complete?  
  - Has Failed?  
  - Wait 10s  

- **Node Details:**  
  - **Log Request**  
    - Type: Code node  
    - Role: Logs prediction details (timestamp, prediction ID, model type) to console for monitoring.  
    - Inputs: From "Create Other Prediction"  
    - Outputs: To "Wait 5s"  
    - Edge Cases: Logging failures do not impact workflow; just for debug.

  - **Wait 5s**  
    - Type: Wait node  
    - Role: Pauses flow for 5 seconds before checking status.  
    - Inputs: From "Log Request"  
    - Outputs: To "Check Status"  
    - Edge Cases: None.

  - **Check Status**  
    - Type: HTTP Request node (GET)  
    - Role: Queries the prediction status using prediction ID.  
    - Configuration:  
      - URL constructed dynamically based on prediction ID from "Create Other Prediction".  
      - Authorization header using API token.  
      - Response JSON parsing with error handling.  
    - Inputs: From "Wait 5s" and "Wait 10s" (retry path)  
    - Outputs: To "Is Complete?"  
    - Edge Cases: API errors, network timeouts, invalid prediction ID.

  - **Is Complete?**  
    - Type: If node  
    - Role: Checks if prediction status equals `"succeeded"`.  
    - Inputs: From "Check Status"  
    - Outputs:  
      - True branch: To "Success Response"  
      - False branch: To "Has Failed?"  
    - Edge Cases: Status could be other values like "starting," "processing," or "failed".

  - **Has Failed?**  
    - Type: If node  
    - Role: Checks if prediction status equals `"failed"`.  
    - Inputs: From "Is Complete?" false branch  
    - Outputs:  
      - True branch: To "Error Response"  
      - False branch: To "Wait 10s" (retry)  
    - Edge Cases: Handles failure explicitly; if status is still in progress, continues polling.

  - **Wait 10s**  
    - Type: Wait node  
    - Role: Pauses for 10 seconds before retrying status check.  
    - Inputs: From "Has Failed?" false branch  
    - Outputs: To "Check Status"  
    - Edge Cases: None.

#### 2.5 Result Handling

- **Overview:** Prepares structured JSON responses for both success and failure cases and outputs the final result.  
- **Nodes Involved:**  
  - Success Response  
  - Error Response  
  - Display Result  

- **Node Details:**  
  - **Success Response**  
    - Type: Set node  
    - Role: Constructs a response object containing success flag, result URL (image output), prediction ID, status, and a success message.  
    - Inputs: From "Is Complete?" true branch  
    - Outputs: To "Display Result"  
    - Edge Cases: Output URL may be missing if API response incomplete.

  - **Error Response**  
    - Type: Set node  
    - Role: Constructs a response object with success flag false, error message, prediction ID, status, and failure message.  
    - Inputs: From "Has Failed?" true branch  
    - Outputs: To "Display Result"  
    - Edge Cases: Error message may be generic if no detailed error returned.

  - **Display Result**  
    - Type: Set node  
    - Role: Wraps the final response object into a property `final_result` for output or further processing.  
    - Inputs: From "Success Response" or "Error Response"  
    - Outputs: Terminal node — end of flow.  
    - Edge Cases: None.

#### 2.6 Logging

- **Overview:** Logs request details to console for monitoring and debugging.  
- **Nodes Involved:**  
  - Log Request (described above)  

#### 2.7 User Information

- **Overview:** Provides embedded documentation, contact information, parameter references, and troubleshooting tips as sticky notes within the workflow editor.  
- **Nodes Involved:**  
  - Sticky Note9  
  - Sticky Note4  

- **Node Details:**  
  - **Sticky Note9**  
    - Content: Contact info for support (email), links to YouTube and LinkedIn channels of the workflow author.  
  - **Sticky Note4**  
    - Content: Detailed workflow description, parameter explanations, quick start instructions, troubleshooting guide, and relevant links (Replicate model docs, API docs, n8n docs).

---

### 3. Summary Table

| Node Name             | Node Type           | Functional Role                   | Input Node(s)           | Output Node(s)             | Sticky Note                                                                                   |
|-----------------------|---------------------|---------------------------------|------------------------|----------------------------|----------------------------------------------------------------------------------------------|
| Manual Trigger        | Manual Trigger      | Starts workflow execution        | None                   | Set API Token               | See Sticky Note4 for detailed usage instructions and overview                                |
| Set API Token          | Set                 | Sets Replicate API token         | Manual Trigger         | Set Other Parameters        | Replace placeholder with your actual token                                                  |
| Set Other Parameters   | Set                 | Defines all model input params   | Set API Token          | Create Other Prediction     | Contains default parameter values for testing                                              |
| Create Other Prediction| HTTP Request (POST) | Sends generation request to API | Set Other Parameters   | Log Request                | Sends model version and parameters to Replicate API                                        |
| Log Request            | Code                | Logs prediction info             | Create Other Prediction| Wait 5s                    | Logs timestamp and prediction ID for monitoring                                            |
| Wait 5s                | Wait                | Pauses 5 seconds before checking| Log Request            | Check Status               |                                                                                        |
| Check Status           | HTTP Request (GET)  | Checks prediction status         | Wait 5s, Wait 10s      | Is Complete?               | Polls status endpoint using prediction ID                                                 |
| Is Complete?           | If                  | Checks if prediction succeeded   | Check Status           | Success Response, Has Failed? |                                                                                      |
| Has Failed?            | If                  | Checks if prediction failed      | Is Complete?           | Error Response, Wait 10s   |                                                                                              |
| Wait 10s               | Wait                | Pauses 10 seconds for retry      | Has Failed?            | Check Status               |                                                                                        |
| Success Response       | Set                 | Prepares success JSON response   | Is Complete?           | Display Result             |                                                                                              |
| Error Response         | Set                 | Prepares error JSON response     | Has Failed?             | Display Result             |                                                                                              |
| Display Result         | Set                 | Outputs final result object      | Success Response, Error Response | None                    |                                                                                              |
| Sticky Note9           | Sticky Note         | Contact and support info         | None                   | None                       | Contact: Yaron@nofluff.online; YouTube & LinkedIn links                                   |
| Sticky Note4           | Sticky Note         | Full workflow documentation      | None                   | None                       | Extensive parameter reference, instructions, troubleshooting, and resource links          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Type: Manual Trigger  
   - No configuration needed.  

2. **Create Set API Token Node:**  
   - Type: Set  
   - Add string field `api_token` with value `YOUR_REPLICATE_API_TOKEN` (replace with your actual token).  
   - Connect Manual Trigger → Set API Token.  

3. **Create Set Other Parameters Node:**  
   - Type: Set  
   - Add multiple fields with these names and default values:  
     - `api_token`: `={{ $('Set API Token').item.json.api_token }}` (expression referencing previous node)  
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
   - Connect Set API Token → Set Other Parameters.  

4. **Create Create Other Prediction Node:**  
   - Type: HTTP Request (POST)  
   - URL: `https://api.replicate.com/v1/predictions`  
   - Headers:  
     - `Authorization`: `=Bearer {{ $json.api_token }}`  
     - `Prefer`: `wait`  
   - Body Type: JSON  
   - JSON Body: (use expressions to map all parameters from Set Other Parameters node)  
     ```json
     {
       "version": "digitalhera/heranathalie:c4e122b4dceba454469d84b69b855b38625b89ca6e922e1c1b1a817ea8f7e340",
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

5. **Create Log Request Node:**  
   - Type: Code (JavaScript)  
   - Code:  
     ```javascript
     const data = $input.all()[0].json;
     console.log('digitalhera/heranathalie Request:', {
       timestamp: new Date().toISOString(),
       prediction_id: data.id,
       model_type: 'other'
     });
     return $input.all();
     ```  
   - Connect Create Other Prediction → Log Request.  

6. **Create Wait 5s Node:**  
   - Type: Wait  
   - Duration: 5 seconds  
   - Connect Log Request → Wait 5s.  

7. **Create Check Status Node:**  
   - Type: HTTP Request (GET)  
   - URL: `=https://api.replicate.com/v1/predictions/{{ $('Create Other Prediction').item.json.id }}`  
   - Headers:  
     - `Authorization`: `=Bearer {{ $('Set API Token').item.json.api_token }}`  
   - Connect Wait 5s → Check Status.  

8. **Create Is Complete? Node (If):**  
   - Type: If  
   - Condition: `$json.status == 'succeeded'` (string equals)  
   - Connect Check Status → Is Complete?.  

9. **Create Success Response Node:**  
   - Type: Set  
   - Assign field `response` to:  
     ```javascript
     {
       success: true,
       result_url: $json.output,
       prediction_id: $json.id,
       status: $json.status,
       message: "Other generated successfully"
     }
     ```  
   - Connect Is Complete? (true branch) → Success Response.  

10. **Create Has Failed? Node (If):**  
    - Type: If  
    - Condition: `$json.status == 'failed'`  
    - Connect Is Complete? (false branch) → Has Failed?.  

11. **Create Error Response Node:**  
    - Type: Set  
    - Assign field `response` to:  
      ```javascript
      {
        success: false,
        error: $json.error || 'Other generation failed',
        prediction_id: $json.id,
        status: $json.status,
        message: "Failed to generate other"
      }
      ```  
    - Connect Has Failed? (true branch) → Error Response.  

12. **Create Wait 10s Node:**  
    - Type: Wait  
    - Duration: 10 seconds  
    - Connect Has Failed? (false branch) → Wait 10s.  

13. **Connect Wait 10s → Check Status** (to create polling loop).  

14. **Create Display Result Node:**  
    - Type: Set  
    - Assign field `final_result` to `={{ $json.response }}`  
    - Connect both Success Response and Error Response → Display Result.  

15. **Add two Sticky Notes for documentation and contact info:**  
    - Sticky Note with contact (email, YouTube, LinkedIn)  
    - Sticky Note with detailed workflow overview, parameter guide, instructions, and troubleshooting tips.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | Context or Link                                                                    |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------|
| For any questions or support, please contact: Yaron@nofluff.online. Explore more tips and tutorials here: YouTube: https://www.youtube.com/@YaronBeen/videos, LinkedIn: https://www.linkedin.com/in/yaronbeen/                                                                                                                                                                                                                                                                                                                                                                                                | Sticky Note9                                                                         |
| Powered by Replicate API and n8n Automation. Model owner: digitalhera, model: heranathalie (Other Generator). Required parameter: prompt. Optional parameters include mask, seed, image, model, width, height, go_fast, LoRA settings, megapixels, outputs, aspect ratio, output format, guidance scale, output quality, prompt strength, inference steps, and safety checker flag. Quick start: get API key at https://replicate.com, replace placeholder, configure parameters, run manual trigger. Troubleshooting common issues and best practices included. Links: https://replicate.com/digitalhera/heranathalie, https://replicate.com/docs, https://docs.n8n.io | Sticky Note4                                                                         |

---

**Disclaimer:** The provided content originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.