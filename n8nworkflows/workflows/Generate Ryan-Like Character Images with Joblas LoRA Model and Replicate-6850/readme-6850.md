Generate Ryan-Like Character Images with Joblas LoRA Model and Replicate

https://n8nworkflows.xyz/workflows/generate-ryan-like-character-images-with-joblas-lora-model-and-replicate-6850


# Generate Ryan-Like Character Images with Joblas LoRA Model and Replicate

### 1. Workflow Overview

This n8n workflow automates the generation of Ryan-like character images using the Joblas LoRA model via the Replicate API. It is designed to send image generation requests, monitor their processing status, handle success or failure responses, and provide structured output for further use. The workflow is suited for AI image generation tasks that require advanced custom model invocation with automated status polling and error handling.

The workflow is logically divided into the following blocks:

- **1.1 Input Initialization**: Starting point and configuration of API authentication and generation parameters.
- **1.2 Prediction Request**: Submitting the image generation job to the Replicate API.
- **1.3 Status Polling and Retry Logic**: Periodically checking the generation status with wait nodes and conditional branching.
- **1.4 Outcome Handling**: Processing success or failure results and preparing the final response.
- **1.5 Logging and Monitoring**: Capturing request details for debugging and analysis.
- **1.6 Documentation and User Guidance**: Embedded sticky notes providing instructions, parameter explanations, and support information.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Initialization

**Overview:**  
This block sets up the initial trigger and configures API credentials and generation parameters required for the model request.

**Nodes Involved:**  
- Manual Trigger  
- Set API Token  
- Set Other Parameters

**Node Details:**

- **Manual Trigger**  
  - *Type:* Trigger node  
  - *Role:* Initiates the workflow manually on user command.  
  - *Configuration:* No parameters; user clicks to start.  
  - *Connections:* Outputs to Set API Token.  
  - *Potential Failures:* None inherent; workflow not started if user does not trigger.  

- **Set API Token**  
  - *Type:* Set node  
  - *Role:* Stores the Replicate API token for authentication.  
  - *Configuration:* Assigns a string variable `api_token` with placeholder `"YOUR_REPLICATE_API_TOKEN"`.  
  - *Connections:* Input from Manual Trigger; output to Set Other Parameters.  
  - *Edge Cases:* Failure if token is invalid or not replaced with a real token.  

- **Set Other Parameters**  
  - *Type:* Set node  
  - *Role:* Defines all input parameters for the Joblas Ryan-LoRA model including prompt, dimensions, seed, and generation options.  
  - *Configuration:* Copies `api_token` from previous node; sets image URL, mask, prompt, width/height, model version, scaling parameters, output format, inference steps, and safety checker flag with default or user-customizable values.  
  - *Connections:* Input from Set API Token; output to Create Other Prediction.  
  - *Potential Failures:* Incorrect parameter types or missing required fields may cause API errors.

---

#### 2.2 Prediction Request

**Overview:**  
This block sends a POST request to the Replicate API to initiate the image generation using provided parameters.

**Nodes Involved:**  
- Create Other Prediction

**Node Details:**

- **Create Other Prediction**  
  - *Type:* HTTP Request node  
  - *Role:* Sends a JSON payload to Replicate’s `/v1/predictions` endpoint to start generation.  
  - *Configuration:*  
    - URL: `https://api.replicate.com/v1/predictions`  
    - Method: POST  
    - Headers: Authorization bearer token from `api_token` variable, plus "Prefer: wait" to wait for prediction completion synchronously.  
    - Body: JSON containing model version ID and all input parameters from Set Other Parameters node, using expressions to interpolate values.  
    - Response handling: JSON, never error (to avoid node failure on API errors).  
  - *Connections:* Input from Set Other Parameters; output to Log Request.  
  - *Potential Failures:* Network issues, invalid API token, malformed JSON, or API rejection due to invalid parameters.

---

#### 2.3 Status Polling and Retry Logic

**Overview:**  
This block implements waiting and polling cycles to check the prediction status until completion or failure, with retry delays on failure.

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
  - *Role:* Logs prediction request details (timestamp, prediction ID, model type) to console for monitoring.  
  - *Code:* JavaScript accessing incoming JSON to log info, then passing data forward.  
  - *Connections:* Input from Create Other Prediction; output to Wait 5s.  
  - *Failures:* Logging errors unlikely but possible if malformed input.  

- **Wait 5s**  
  - *Type:* Wait node  
  - *Role:* Pauses execution for 5 seconds before status check.  
  - *Parameters:* 5 seconds delay.  
  - *Connections:* Input from Log Request; output to Check Status.  

- **Check Status**  
  - *Type:* HTTP Request node  
  - *Role:* GET request to check current status of prediction using prediction ID from Create Other Prediction node.  
  - *Configuration:*  
    - URL: `https://api.replicate.com/v1/predictions/{{prediction_id}}`  
    - Headers: Authorization bearer token  
    - Response: JSON, never error  
  - *Connections:* Input from Wait 5s or Wait 10s; output to Is Complete?.  
  - *Failures:* Network issues, invalid prediction ID, API errors.

- **Is Complete?**  
  - *Type:* If node  
  - *Role:* Checks if prediction status equals `"succeeded"`.  
  - *Connections:* Input from Check Status; if true to Success Response; else to Has Failed?.  
  - *Edge Cases:* If status is neither succeeded nor failed, proceeds to failure check.

- **Has Failed?**  
  - *Type:* If node  
  - *Role:* Checks if prediction status equals `"failed"`.  
  - *Connections:* Input from Is Complete?; if true to Error Response; else to Wait 10s (retry).  
  - *Edge Cases:* Covers transient or unknown states by retrying after 10 seconds.

- **Wait 10s**  
  - *Type:* Wait node  
  - *Role:* Delays 10 seconds before retrying status check.  
  - *Parameters:* 10 seconds delay.  
  - *Connections:* Input from Has Failed? false branch; output to Check Status.  

---

#### 2.4 Outcome Handling

**Overview:**  
This block processes the final result or error, formatting a structured response object to be passed onwards.

**Nodes Involved:**  
- Success Response  
- Error Response  
- Display Result

**Node Details:**

- **Success Response**  
  - *Type:* Set node  
  - *Role:* Creates an object indicating success, including output URL, prediction ID, status, and a success message.  
  - *Configuration:* Uses JSON fields from the prediction status response.  
  - *Connections:* Input from Is Complete? true branch; output to Display Result.  
  - *Failures:* Failure if expected fields missing in response JSON.

- **Error Response**  
  - *Type:* Set node  
  - *Role:* Creates an object indicating failure, including error message, prediction ID, status, and failure message.  
  - *Connections:* Input from Has Failed? true branch; output to Display Result.  
  - *Failures:* If error details are missing, defaults to generic failure message.

- **Display Result**  
  - *Type:* Set node  
  - *Role:* Wraps the final response object into a `final_result` key for output.  
  - *Connections:* Input from Success Response or Error Response; workflow end point.  

---

#### 2.5 Logging and Monitoring

**Overview:**  
A dedicated code node logs key details of each prediction request to the console for audit and debugging.

**Nodes Involved:**  
- Log Request (already detailed in 2.3)

---

#### 2.6 Documentation and User Guidance

**Overview:**  
Sticky notes provide comprehensive documentation, instructions, parameter references, and contact information embedded within the workflow canvas for user assistance.

**Nodes Involved:**  
- Sticky Note9  
- Sticky Note4

**Node Details:**

- **Sticky Note9**  
  - *Content:* Contact info for support, including email and links to YouTube and LinkedIn channels.  
  - *Purpose:* User support and community resources.

- **Sticky Note4**  
  - *Content:* Detailed workflow documentation including model overview, parameter descriptions, workflow explanation, benefits, quick start instructions, troubleshooting, and resource links.  
  - *Purpose:* In-depth user guide and reference directly inside workflow editor.

---

### 3. Summary Table

| Node Name            | Node Type           | Functional Role                  | Input Node(s)         | Output Node(s)         | Sticky Note                                                                                          |
|----------------------|---------------------|---------------------------------|-----------------------|------------------------|----------------------------------------------------------------------------------------------------|
| Manual Trigger       | Trigger             | Starts workflow manually         | —                     | Set API Token           |                                                                                                |
| Set API Token        | Set                 | Stores Replicate API token       | Manual Trigger         | Set Other Parameters    |                                                                                                |
| Set Other Parameters | Set                 | Defines model input parameters   | Set API Token          | Create Other Prediction |                                                                                                |
| Create Other Prediction | HTTP Request       | Sends generation request to API | Set Other Parameters   | Log Request             |                                                                                                |
| Log Request          | Code                | Logs request details             | Create Other Prediction| Wait 5s                 |                                                                                                |
| Wait 5s              | Wait                | Waits 5 seconds before status check | Log Request         | Check Status            |                                                                                                |
| Check Status         | HTTP Request        | Gets current prediction status  | Wait 5s, Wait 10s      | Is Complete?            |                                                                                                |
| Is Complete?         | If                  | Checks if generation succeeded  | Check Status           | Success Response, Has Failed? |                                                                                                |
| Has Failed?          | If                  | Checks if generation failed     | Is Complete?           | Error Response, Wait 10s |                                                                                                |
| Wait 10s             | Wait                | Waits 10 seconds before retry   | Has Failed?            | Check Status            |                                                                                                |
| Success Response     | Set                 | Formats successful response     | Is Complete?           | Display Result          |                                                                                                |
| Error Response       | Set                 | Formats error response          | Has Failed?            | Display Result          |                                                                                                |
| Display Result       | Set                 | Prepares final output           | Success Response, Error Response | —                 |                                                                                                |
| Sticky Note9         | Sticky Note         | Support contact info            | —                      | —                      | For any questions or support, please contact: Yaron@nofluff.online; YouTube & LinkedIn links      |
| Sticky Note4         | Sticky Note         | Full workflow documentation     | —                      | —                      | Detailed model info, parameters, instructions, troubleshooting, and external resource links        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node** named `Manual Trigger` with no parameters.

2. **Add a Set node** named `Set API Token`:
   - Add a string field `api_token`.
   - Set its value to `"YOUR_REPLICATE_API_TOKEN"` (replace with your actual Replicate API token).
   - Connect `Manual Trigger` output to this node.

3. **Add a Set node** named `Set Other Parameters`:
   - Add multiple fields matching model parameters:
     - `api_token`: set to expression `{{$node["Set API Token"].json["api_token"]}}`
     - `mask`: `"https://via.placeholder.com/512x512/000000/FFFFFF.png"`
     - `seed`: `-1` (number)
     - `image`: `"https://picsum.photos/512/512"`
     - `model`: `"dev"`
     - `width`: `512`
     - `height`: `512`
     - `prompt`: `"Create something amazing"`
     - `go_fast`: `false` (boolean)
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
   - Connect `Set API Token` output to this node.

4. **Add an HTTP Request node** named `Create Other Prediction`:
   - Set method to `POST`.
   - URL: `https://api.replicate.com/v1/predictions`
   - Headers:
     - `Authorization`: `Bearer {{$json.api_token}}`
     - `Prefer`: `wait`
   - Body Content Type: JSON
   - Body JSON (use expressions to fill with values from `Set Other Parameters` node):
     ```json
     {
       "version": "joblas/ryan-lora:b17a69d6cd0974390a2d6844df603e4b3f0342c34e8e0594dc80ffa892c2fae2",
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
   - Connect `Set Other Parameters` output to this node.

5. **Add a Code node** named `Log Request`:
   - Use JavaScript code to log the prediction request details:
     ```javascript
     const data = $input.all()[0].json;
     console.log('joblas/ryan-lora Request:', {
       timestamp: new Date().toISOString(),
       prediction_id: data.id,
       model_type: 'other'
     });
     return $input.all();
     ```
   - Connect `Create Other Prediction` output to this node.

6. **Add a Wait node** named `Wait 5s`:
   - Set to wait 5 seconds.
   - Connect `Log Request` output to this node.

7. **Add an HTTP Request node** named `Check Status`:
   - Method: GET
   - URL: `https://api.replicate.com/v1/predictions/{{$node["Create Other Prediction"].json["id"]}}`
   - Header:
     - `Authorization`: `Bearer {{$node["Set API Token"].json["api_token"]}}`
   - Response format: JSON
   - Connect `Wait 5s` output and later `Wait 10s` output to this node.

8. **Add an If node** named `Is Complete?`:
   - Condition: Check if `{{$json["status"]}}` equals `"succeeded"`.
   - Connect `Check Status` output to this node.

9. **Add an If node** named `Has Failed?`:
   - Condition: Check if `{{$json["status"]}}` equals `"failed"`.
   - Connect `Is Complete?` false output to this node.

10. **Add a Set node** named `Success Response`:
    - Set field `response` to an object:
      ```json
      {
        "success": true,
        "result_url": {{$json["output"]}},
        "prediction_id": {{$json["id"]}},
        "status": {{$json["status"]}},
        "message": "Other generated successfully"
      }
      ```
    - Connect `Is Complete?` true output to this node.

11. **Add a Set node** named `Error Response`:
    - Set field `response` to an object:
      ```json
      {
        "success": false,
        "error": {{$json["error"] || "Other generation failed"}},
        "prediction_id": {{$json["id"]}},
        "status": {{$json["status"]}},
        "message": "Failed to generate other"
      }
      ```
    - Connect `Has Failed?` true output to this node.

12. **Add a Wait node** named `Wait 10s`:
    - Set wait time to 10 seconds.
    - Connect `Has Failed?` false output to this node.
    - Connect `Wait 10s` output back to `Check Status`.

13. **Add a Set node** named `Display Result`:
    - Set field `final_result` to `{{$json["response"]}}`.
    - Connect outputs of both `Success Response` and `Error Response` to this node.

14. **Connect all nodes in the order described above**, ensuring branches from If nodes follow the logic for success, failure, or retry loops.

15. **Add two Sticky Note nodes** named `Sticky Note9` and `Sticky Note4`:
    - `Sticky Note9`: Add contact info with email and social links.
    - `Sticky Note4`: Add detailed workflow documentation and parameter reference text as per the original content.

16. **Save and activate the workflow**.

---

### 5. General Notes & Resources

| Note Content                                                                                                                            | Context or Link                                                                                      |
|-----------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| For any questions or support, please contact: Yaron@nofluff.online                                                                      | Support contact information (Sticky Note9)                                                        |
| Explore more tips and tutorials here: YouTube: https://www.youtube.com/@YaronBeen/videos, LinkedIn: https://www.linkedin.com/in/yaronbeen/ | Community and learning resources (Sticky Note9)                                                   |
| Model Documentation: https://replicate.com/joblas/ryan-lora                                                                              | Official model page on Replicate                                                                   |
| Replicate API Docs: https://replicate.com/docs                                                                                          | API reference for developers                                                                       |
| n8n Documentation: https://docs.n8n.io                                                                                                  | Official n8n documentation                                                                          |
| Replace 'YOUR_REPLICATE_API_TOKEN' with your actual Replicate API token before running the workflow                                      | Critical credential setup note                                                                     |
| Monitor API usage and ensure sufficient quota to avoid generation failures                                                               | Important operational advice                                                                       |
| Use default parameters initially to verify workflow correctness before customization                                                    | Best practice for troubleshooting                                                                 |
| The workflow includes retry logic with 10 seconds wait to handle transient errors or longer generation times                            | Reliability and robustness feature                                                                 |

---

This documentation provides a complete, detailed understanding of the Ryan-LoRA image generation workflow, enabling reproduction, customization, error anticipation, and effective operation.