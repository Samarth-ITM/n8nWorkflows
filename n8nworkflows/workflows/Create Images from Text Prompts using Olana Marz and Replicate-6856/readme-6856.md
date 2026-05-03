Create Images from Text Prompts using Olana Marz and Replicate

https://n8nworkflows.xyz/workflows/create-images-from-text-prompts-using-olana-marz-and-replicate-6856


# Create Images from Text Prompts using Olana Marz and Replicate

### 1. Workflow Overview

This workflow enables automated generation of images based on text prompts using the Olana Marz model hosted on Replicate’s API. It is designed for use cases where users want to transform textual descriptions into AI-generated images with customizable parameters such as size, style, and generation details.

The workflow is logically divided into the following blocks:

- **1.1 Input Initialization:** Receives a manual trigger and sets API credentials and model input parameters.
- **1.2 Prediction Request:** Sends a generation request to the Replicate API with configured parameters.
- **1.3 Status Polling Loop:** Waits and periodically checks the status of the generation request, handling retries and delays intelligently.
- **1.4 Result Handling:** Processes final success or failure states, formats responses accordingly.
- **1.5 Logging and Monitoring:** Logs generation requests for audit and debugging.
- **1.6 Output Preparation:** Prepares the output data to be returned or further processed.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Initialization

**Overview:**  
This block captures the workflow start event, sets the API token for Replicate authentication, and configures all input parameters needed by the Olana Marz model.

**Nodes Involved:**  
- Manual Trigger  
- Set API Token  
- Set Other Parameters  

**Node Details:**

- **Manual Trigger**  
  - *Type:* Trigger node  
  - *Role:* Starts the workflow manually  
  - *Config:* No parameters; manual user activation  
  - *Inputs:* None  
  - *Outputs:* Connects to Set API Token  
  - *Edge Cases:* None; user must manually trigger

- **Set API Token**  
  - *Type:* Set node  
  - *Role:* Stores the Replicate API token securely for use downstream  
  - *Config:* Assigns string value `YOUR_REPLICATE_API_TOKEN` to `api_token` variable  
  - *Inputs:* Manual Trigger  
  - *Outputs:* Set Other Parameters  
  - *Edge Cases:* Must be replaced with a valid token before execution to avoid authorization errors

- **Set Other Parameters**  
  - *Type:* Set node  
  - *Role:* Prepares all parameters required for the model prediction, including prompt, image dimensions, styles, and generation options  
  - *Config:*  
    - Copies `api_token` from previous node  
    - Sets defaults for `mask`, `seed`, `image`, `model` name, dimensions (`width`, `height`), `prompt`, and many optional parameters like `go_fast`, `extra_lora`, `lora_scale`, `num_outputs`, `aspect_ratio`, `output_format`, `guidance_scale`, `output_quality`, `num_inference_steps`, and safety options.  
    - Default prompt is `"Create something amazing"`  
  - *Inputs:* Set API Token  
  - *Outputs:* Create Other Prediction  
  - *Edge Cases:* Invalid or missing parameters may cause API request failures

---

#### 2.2 Prediction Request

**Overview:**  
Sends a POST request to Replicate API to create a new prediction job using the configured parameters.

**Nodes Involved:**  
- Create Other Prediction  

**Node Details:**

- **Create Other Prediction**  
  - *Type:* HTTP Request node  
  - *Role:* Posts the generation job to Replicate API using the specified model version and input parameters  
  - *Config:*  
    - HTTP POST to `https://api.replicate.com/v1/predictions`  
    - JSON body includes model version ID and all parameters interpolated from previous node  
    - Headers include `Authorization: Bearer <api_token>` and `Prefer: wait` to request synchronous response if possible  
    - Response expected in JSON with prediction ID and status  
  - *Inputs:* Set Other Parameters  
  - *Outputs:* Log Request  
  - *Edge Cases:*  
    - API token invalid or quota exceeded may cause HTTP 401/403 errors  
    - Invalid parameters may cause HTTP 400 errors  
    - Network timeouts or Replicate API downtime

---

#### 2.3 Status Polling Loop

**Overview:**  
After submitting the prediction request, the workflow waits and polls the prediction status until completion or failure.

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
  - *Role:* Logs the prediction request details (timestamp, prediction ID, model type) for monitoring  
  - *Inputs:* Create Other Prediction  
  - *Outputs:* Wait 5s  
  - *Edge Cases:* Console logging errors (rare)

- **Wait 5s**  
  - *Type:* Wait node  
  - *Role:* Adds a 5-second delay before first status check  
  - *Inputs:* Log Request  
  - *Outputs:* Check Status  
  - *Edge Cases:* None

- **Check Status**  
  - *Type:* HTTP Request node  
  - *Role:* Retrieves current prediction status from Replicate API  
  - *Config:*  
    - GET request to `https://api.replicate.com/v1/predictions/{{prediction_id}}`  
    - Uses `Authorization: Bearer <api_token>` header  
    - Response parsed as JSON  
  - *Inputs:* Wait 5s, Wait 10s (loop back)  
  - *Outputs:* Is Complete?  
  - *Edge Cases:* API downtime, invalid prediction ID, auth errors

- **Is Complete?**  
  - *Type:* If node  
  - *Role:* Checks if prediction status is `"succeeded"`  
  - *Inputs:* Check Status  
  - *Outputs:*  
    - True: Success Response  
    - False: Has Failed?  
  - *Edge Cases:* Unexpected status values

- **Has Failed?**  
  - *Type:* If node  
  - *Role:* Checks if prediction status is `"failed"`  
  - *Inputs:* Is Complete? (False path)  
  - *Outputs:*  
    - True: Error Response  
    - False: Wait 10s (retry)  
  - *Edge Cases:* Handling other statuses such as "processing" or "starting"

- **Wait 10s**  
  - *Type:* Wait node  
  - *Role:* Delays 10 seconds before the next status check retry  
  - *Inputs:* Has Failed? (False path)  
  - *Outputs:* Check Status  
  - *Edge Cases:* None

---

#### 2.4 Result Handling

**Overview:**  
Processes final success or failure outcomes, formats the output response objects, and prepares them for display or further use.

**Nodes Involved:**  
- Success Response  
- Error Response  
- Display Result  

**Node Details:**

- **Success Response**  
  - *Type:* Set node  
  - *Role:* Sets a structured JSON response for successful generation, including the output URL, prediction ID, status, and success message  
  - *Inputs:* Is Complete? (True path)  
  - *Outputs:* Display Result  
  - *Edge Cases:* Missing or malformed output URLs

- **Error Response**  
  - *Type:* Set node  
  - *Role:* Sets a structured JSON response for failure, including error message, prediction ID, status, and failure notice  
  - *Inputs:* Has Failed? (True path)  
  - *Outputs:* Display Result  
  - *Edge Cases:* Absence of detailed error message

- **Display Result**  
  - *Type:* Set node  
  - *Role:* Final node preparing the `final_result` object for output, used as workflow return or for next steps  
  - *Inputs:* Success Response, Error Response  
  - *Outputs:* None (workflow end)  
  - *Edge Cases:* None

---

#### 2.5 Logging and Monitoring

**Overview:**  
Logs key workflow data to the console for monitoring and debugging purposes.

**Nodes Involved:**  
- Log Request  

**Node Details:**

- See details in 2.3 Status Polling Loop (Log Request node).

---

### 3. Summary Table

| Node Name             | Node Type           | Functional Role                      | Input Node(s)           | Output Node(s)          | Sticky Note                                           |
|-----------------------|---------------------|------------------------------------|------------------------|-------------------------|-------------------------------------------------------|
| Manual Trigger        | manualTrigger       | Starts workflow                    | None                   | Set API Token            |                                                       |
| Set API Token         | set                 | Stores Replicate API token         | Manual Trigger         | Set Other Parameters     |                                                       |
| Set Other Parameters  | set                 | Sets model input parameters        | Set API Token          | Create Other Prediction  |                                                       |
| Create Other Prediction| httpRequest         | Sends generation request to Replicate | Set Other Parameters | Log Request              |                                                       |
| Log Request           | code                | Logs generation request details    | Create Other Prediction| Wait 5s                  |                                                       |
| Wait 5s               | wait                | Initial delay before status check  | Log Request            | Check Status             |                                                       |
| Check Status          | httpRequest         | Polls prediction status from API   | Wait 5s, Wait 10s      | Is Complete?             |                                                       |
| Is Complete?          | if                  | Checks if prediction succeeded     | Check Status           | Success Response, Has Failed? |                                                       |
| Has Failed?           | if                  | Checks if prediction failed        | Is Complete?           | Error Response, Wait 10s |                                                       |
| Wait 10s              | wait                | Delay before retrying status check | Has Failed?            | Check Status             |                                                       |
| Success Response      | set                 | Sets success JSON response         | Is Complete?           | Display Result           |                                                       |
| Error Response        | set                 | Sets error JSON response           | Has Failed?            | Display Result           |                                                       |
| Display Result        | set                 | Prepares final output object       | Success Response, Error Response | None             |                                                       |
| Sticky Note9          | stickyNote          | Contact and support info            | None                   | None                    | See section 5 for details                              |
| Sticky Note4          | stickyNote          | Extensive documentation & usage notes | None                | None                    | See section 5 for details                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Node: Manual Trigger**  
   - Type: Manual Trigger  
   - No parameters needed  
   - Position on canvas leftmost

2. **Create Node: Set API Token**  
   - Type: Set  
   - Configure a string variable `api_token` with value `"YOUR_REPLICATE_API_TOKEN"` (replace with your actual token)  
   - Connect Manual Trigger → Set API Token

3. **Create Node: Set Other Parameters**  
   - Type: Set  
   - Assign variables:  
     - `api_token` = `={{ $('Set API Token').item.json.api_token }}` (copy token from previous node)  
     - `mask` = `"https://via.placeholder.com/512x512/000000/FFFFFF.png"`  
     - `seed` = `-1`  
     - `image` = `"https://picsum.photos/512/512"`  
     - `model` = `"dev"`  
     - `width` = `512`  
     - `height` = `512`  
     - `prompt` = `"Create something amazing"`  
     - `go_fast` = `false`  
     - `extra_lora` = `""`  
     - `lora_scale` = `1`  
     - `megapixels` = `"1"`  
     - `num_outputs` = `1`  
     - `aspect_ratio` = `"1:1"`  
     - `output_format` = `"webp"`  
     - `guidance_scale` = `3`  
     - `output_quality` = `80`  
     - `prompt_strength` = `0.8`  
     - `extra_lora_scale` = `1`  
     - `replicate_weights` = `""`  
     - `num_inference_steps` = `28`  
     - `disable_safety_checker` = `false`  
   - Connect Set API Token → Set Other Parameters

4. **Create Node: Create Other Prediction**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.replicate.com/v1/predictions`  
   - Headers:  
     - `Authorization`: `Bearer {{ $json.api_token }}`  
     - `Prefer`: `wait`  
   - Body Type: JSON  
   - Body Content:  
     ```json
     {
       "version": "biggpt1/olana-marz:2a01d27ead8a0d857a1f985ab225eb1732d0bdc3783625e628cc7dccb5835863",
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
   - Connect Set Other Parameters → Create Other Prediction

5. **Create Node: Log Request**  
   - Type: Code  
   - JavaScript code:  
     ```js
     const data = $input.all()[0].json;
     console.log('biggpt1/olana-marz Request:', {
       timestamp: new Date().toISOString(),
       prediction_id: data.id,
       model_type: 'other'
     });
     return $input.all();
     ```  
   - Connect Create Other Prediction → Log Request

6. **Create Node: Wait 5s**  
   - Type: Wait  
   - Parameters: 5 seconds  
   - Connect Log Request → Wait 5s

7. **Create Node: Check Status**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://api.replicate.com/v1/predictions/{{ $('Create Other Prediction').item.json.id }}`  
   - Header: `Authorization: Bearer {{ $('Set API Token').item.json.api_token }}`  
   - Connect Wait 5s → Check Status  
   - Also connects from Wait 10s → Check Status (for retries)

8. **Create Node: Is Complete?**  
   - Type: If  
   - Condition:  
     - Check if `{{ $json.status }}` equals `"succeeded"`  
   - Connect Check Status → Is Complete?

9. **Create Node: Success Response**  
   - Type: Set  
   - Assign variable `response`:  
     ```json
     {
       "success": true,
       "result_url": "{{ $json.output }}",
       "prediction_id": "{{ $json.id }}",
       "status": "{{ $json.status }}",
       "message": "Other generated successfully"
     }
     ```  
   - Connect Is Complete? (True) → Success Response

10. **Create Node: Has Failed?**  
    - Type: If  
    - Condition:  
      - Check if `{{ $json.status }}` equals `"failed"`  
    - Connect Is Complete? (False) → Has Failed?

11. **Create Node: Error Response**  
    - Type: Set  
    - Assign variable `response`:  
      ```json
      {
        "success": false,
        "error": "{{ $json.error || 'Other generation failed' }}",
        "prediction_id": "{{ $json.id }}",
        "status": "{{ $json.status }}",
        "message": "Failed to generate other"
      }
      ```  
    - Connect Has Failed? (True) → Error Response

12. **Create Node: Wait 10s**  
    - Type: Wait  
    - Parameters: 10 seconds  
    - Connect Has Failed? (False) → Wait 10s

13. **Connect Wait 10s → Check Status** (loop back for retries)

14. **Create Node: Display Result**  
    - Type: Set  
    - Assign variable `final_result` = `{{ $json.response }}` from either Success Response or Error Response  
    - Connect Success Response → Display Result  
    - Connect Error Response → Display Result

15. **Add Sticky Notes:**  
    - Add two sticky notes with workflow description, documentation, contact info, and usage instructions as per original content.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                         | Context or Link                                                                                          |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| =======================================  OLANA-MARZ GENERATOR ======================================= For any questions or support, please contact: Yaron@nofluff.online Explore more tips and tutorials here: - YouTube: https://www.youtube.com/@YaronBeen/videos - LinkedIn: https://www.linkedin.com/in/yaronbeen/ ======================================= | Contact and support information embedded in Sticky Note9                                                 |
| BIGGPT1/OLANA-MARZ - OTHER GENERATION WORKFLOW powered by Replicate API and n8n Automation. Detailed parameter descriptions, usage instructions, troubleshooting guide, and links to resources including model documentation (https://replicate.com/biggpt1/olana-marz), Replicate API docs (https://replicate.com/docs), and n8n docs (https://docs.n8n.io). | Documentation, parameter guide, quick start, and troubleshooting in Sticky Note4                         |

---

**Disclaimer:** The provided text and workflow originate exclusively from an automated n8n workflow. It respects all content policies and contains no illegal or protected elements. All data handled is legal and publicly accessible.