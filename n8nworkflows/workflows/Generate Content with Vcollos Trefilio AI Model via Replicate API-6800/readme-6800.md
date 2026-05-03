Generate Content with Vcollos Trefilio AI Model via Replicate API

https://n8nworkflows.xyz/workflows/generate-content-with-vcollos-trefilio-ai-model-via-replicate-api-6800


# Generate Content with Vcollos Trefilio AI Model via Replicate API

### 1. Workflow Overview

This workflow automates content generation using the **vcollos/trefilio AI model** via the **Replicate API**. It is designed to generate image-based outputs (referred to as "other" generation) based on a prompt and various image-related parameters. The workflow handles the entire lifecycle from input configuration, API request submission, polling for prediction status, to delivering final results or errors.

The logical blocks are:

- **1.1 Input Setup and Trigger**: Starts the workflow and prepares API authentication and model parameters.
- **1.2 Prediction Request Submission**: Sends the generation request to the Replicate API.
- **1.3 Prediction Status Polling**: Periodically checks the status of the prediction until completion or failure.
- **1.4 Result Handling**: Processes success or failure responses and prepares the output.
- **1.5 Logging and Monitoring**: Logs request details for debugging and monitoring.
- **1.6 User Guidance and Documentation**: Sticky notes providing usage instructions, references, and contact info.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Input Setup and Trigger

**Overview:**  
This block initiates the workflow and sets all required and optional parameters, including the API token and model input parameters.

**Nodes Involved:**  
- Manual Trigger  
- Set API Token  
- Set Other Parameters  

**Node Details:**

- **Manual Trigger**  
  - Type: Manual trigger node  
  - Role: Starts workflow execution manually on user action  
  - Configuration: No parameters; simple trigger  
  - Connections: Output → Set API Token  
  - Potential Failures: None (manual trigger)  

- **Set API Token**  
  - Type: Set node  
  - Role: Stores the Replicate API token in a variable `api_token`  
  - Configuration: Hardcoded placeholder string `"YOUR_REPLICATE_API_TOKEN"` (user must replace with actual token)  
  - Connections: Input ← Manual Trigger; Output → Set Other Parameters  
  - Edge Cases: Missing or invalid token will cause API authentication failure downstream  

- **Set Other Parameters**  
  - Type: Set node  
  - Role: Defines all input parameters for the AI model generation request  
  - Configuration:  
    - Copies `api_token` from previous node  
    - Sets default values for parameters such as `mask`, `seed`, `image`, `model`, `width`, `height`, `prompt`, `go_fast`, `extra_lora`, `lora_scale`, `megapixels`, `num_outputs`, `aspect_ratio`, `output_format`, `guidance_scale`, `output_quality`, `prompt_strength`, `extra_lora_scale`, `replicate_weights`, `num_inference_steps`, and `disable_safety_checker`  
    - Defaults include a placeholder mask URL, a sample image URL, prompt "Create something amazing", and other reasonable defaults  
  - Connections: Input ← Set API Token; Output → Create Other Prediction  
  - Edge Cases: Parameter types must be respected; invalid values (e.g., negative width) may cause API errors  

---

#### Block 1.2: Prediction Request Submission

**Overview:**  
Submits a request to Replicate API to create a new prediction using the configured parameters.

**Nodes Involved:**  
- Create Other Prediction  

**Node Details:**

- **Create Other Prediction**  
  - Type: HTTP Request (POST)  
  - Role: Sends POST request to Replicate API endpoint `/v1/predictions` to initiate content generation  
  - Configuration:
    - URL: `https://api.replicate.com/v1/predictions`  
    - Method: POST  
    - Headers: Authorization with Bearer token from `api_token`, Prefer: wait  
    - Body: JSON including model version and all parameters set in previous node, using expressions to inject values  
    - Response format: JSON, with error suppression (`neverError: true`) to prevent workflow failure on HTTP errors  
  - Connections: Input ← Set Other Parameters; Output → Log Request  
  - Edge Cases:  
    - API token invalid or expired leads to 401 error  
    - Network errors/timeouts  
    - Parameter validation errors from API  
    - If API returns error, response is captured but marked not to error the workflow  

---

#### Block 1.3: Prediction Status Polling

**Overview:**  
Polls the prediction status by waiting and repeatedly checking until the prediction is succeeded or failed.

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
  - Role: Logs prediction request details such as timestamp, prediction ID, and model type for monitoring purposes  
  - Configuration: JavaScript code logs to console and passes data through unchanged  
  - Connections: Input ← Create Other Prediction; Output → Wait 5s  
  - Failure Modes: Logging itself unlikely to fail, but console output depends on environment  

- **Wait 5s**  
  - Type: Wait node  
  - Role: Pauses workflow for 5 seconds before status check  
  - Configuration: Wait for 5 seconds  
  - Connections: Input ← Log Request; Output → Check Status  
  - Edge Cases: Node may be interrupted or delayed, but reliable wait  

- **Check Status**  
  - Type: HTTP Request (GET)  
  - Role: Queries the prediction status from Replicate API using prediction ID  
  - Configuration:  
    - URL: `https://api.replicate.com/v1/predictions/{{prediction_id}}` (dynamic)  
    - Authorization header with API token  
    - Response JSON, no errors thrown on failure  
  - Connections: Input ← Wait 5s or Wait 10s; Output → Is Complete?  
  - Edge Cases: Network errors, token invalidation, prediction ID missing or expired  

- **Is Complete?**  
  - Type: If node  
  - Role: Checks if prediction status equals "succeeded"  
  - Configuration: Compares `$json.status === "succeeded"`  
  - Connections: Input ← Check Status; Output True → Success Response; Output False → Has Failed?  
  - Edge Cases: Status field missing or unexpected values  

- **Has Failed?**  
  - Type: If node  
  - Role: Checks if prediction status equals "failed"  
  - Configuration: Compares `$json.status === "failed"`  
  - Connections: Input ← Is Complete? False; Output True → Error Response; Output False → Wait 10s  
  - Edge Cases: Status unknown or other transient states  

- **Wait 10s**  
  - Type: Wait node  
  - Role: Pauses 10 seconds before retrying status check  
  - Configuration: Wait for 10 seconds  
  - Connections: Input ← Has Failed? False; Output → Check Status (loop)  
  - Edge Cases: Delay retry to avoid API rate limits and allow prediction to progress  

---

#### Block 1.4: Result Handling

**Overview:**  
Handles the final result after prediction completion, preparing structured success or error responses, and outputs the final data.

**Nodes Involved:**  
- Success Response  
- Error Response  
- Display Result  

**Node Details:**

- **Success Response**  
  - Type: Set node  
  - Role: Constructs a success JSON object including success flag, result URL, prediction ID, status, and message  
  - Configuration: Uses fields from the prediction output `$json.output`, `$json.id`, `$json.status` and sets message "Other generated successfully"  
  - Connections: Input ← Is Complete? True; Output → Display Result  
  - Edge Cases: Ensure output URL exists; otherwise response may be incomplete  

- **Error Response**  
  - Type: Set node  
  - Role: Constructs an error JSON object including success flag false, error message, prediction ID, status, and message  
  - Configuration: Uses `$json.error` if present, else defaults to "Other generation failed", includes prediction metadata  
  - Connections: Input ← Has Failed? True; Output → Display Result  
  - Edge Cases: API error messages may vary; fallback message always present  

- **Display Result**  
  - Type: Set node  
  - Role: Sets the final result in `final_result` property for downstream consumption or webhook response  
  - Configuration: Copies the `response` field from previous node (success or error) into `final_result`  
  - Connections: Input ← Success Response or Error Response; Output: End of workflow  
  - Edge Cases: None; final output preparation  

---

#### Block 1.5: Logging and Monitoring

**Overview:**  
Logs prediction request details for monitoring and debugging purposes.

**Nodes Involved:**  
- Log Request (covered above)

---

#### Block 1.6: User Guidance and Documentation

**Overview:**  
Provides detailed instructions, parameter references, contact information, and links for support and troubleshooting.

**Nodes Involved:**  
- Sticky Note9  
- Sticky Note4  

**Node Details:**

- **Sticky Note9**  
  - Type: Sticky Note  
  - Role: Displays contact email and links to YouTube and LinkedIn profiles of the workflow author for support  
  - Content Highlights:  
    - Contact: Yaron@nofluff.online  
    - YouTube: https://www.youtube.com/@YaronBeen/videos  
    - LinkedIn: https://www.linkedin.com/in/yaronbeen/  

- **Sticky Note4**  
  - Type: Sticky Note  
  - Role: Provides a comprehensive overview of the workflow, model parameters, debugging tips, quick start guide, and external links  
  - Content Highlights:  
    - Model owner and type  
    - List of required and optional parameters with explanations  
    - Workflow explanation by blocks  
    - Benefits and troubleshooting guidance  
    - Links to Replicate model docs, API docs, and n8n docs  

---

### 3. Summary Table

| Node Name              | Node Type            | Functional Role                           | Input Node(s)              | Output Node(s)         | Sticky Note                                                                                      |
|------------------------|----------------------|-----------------------------------------|----------------------------|------------------------|------------------------------------------------------------------------------------------------|
| Manual Trigger         | Manual Trigger       | Initiates workflow execution             |                            | Set API Token           |                                                                                                |
| Set API Token          | Set                  | Sets Replicate API Token                  | Manual Trigger             | Set Other Parameters    |                                                                                                |
| Set Other Parameters   | Set                  | Sets all input parameters for generation | Set API Token              | Create Other Prediction |                                                                                                |
| Create Other Prediction| HTTP Request (POST)  | Sends generation request to Replicate API| Set Other Parameters       | Log Request             |                                                                                                |
| Log Request            | Code                 | Logs prediction details for monitoring   | Create Other Prediction    | Wait 5s                 |                                                                                                |
| Wait 5s                | Wait                 | Waits 5 seconds before status check      | Log Request                | Check Status            |                                                                                                |
| Check Status           | HTTP Request (GET)   | Checks prediction status                  | Wait 5s, Wait 10s          | Is Complete?            |                                                                                                |
| Is Complete?           | If                   | Checks if prediction succeeded           | Check Status               | Success Response, Has Failed? |                                                                                                |
| Has Failed?            | If                   | Checks if prediction failed               | Is Complete?               | Error Response, Wait 10s|                                                                                                |
| Wait 10s               | Wait                 | Waits 10 seconds before retrying status  | Has Failed?                | Check Status            |                                                                                                |
| Success Response       | Set                  | Prepares success response JSON            | Is Complete?               | Display Result          |                                                                                                |
| Error Response         | Set                  | Prepares error response JSON              | Has Failed?                | Display Result          |                                                                                                |
| Display Result         | Set                  | Sets final output data                     | Success Response, Error Response |                      |                                                                                                |
| Sticky Note9           | Sticky Note          | Contact info and social links             |                            |                        | Contact: Yaron@nofluff.online; YouTube: https://www.youtube.com/@YaronBeen/videos; LinkedIn: https://www.linkedin.com/in/yaronbeen/ |
| Sticky Note4           | Sticky Note          | Detailed workflow overview and instructions|                           |                        | Comprehensive guide including parameters, troubleshooting, and links to docs                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: Start workflow manually  

2. **Create Set Node "Set API Token"**  
   - Type: Set  
   - Add string field `api_token` with value `"YOUR_REPLICATE_API_TOKEN"` (replace with your actual token)  
   - Connect output of Manual Trigger → Set API Token  

3. **Create Set Node "Set Other Parameters"**  
   - Type: Set  
   - Add fields with these default values, referencing `api_token` from previous node:  
     - `api_token`: `={{ $('Set API Token').item.json.api_token }}`  
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
   - Connect output of Set API Token → Set Other Parameters  

4. **Create HTTP Request Node "Create Other Prediction"**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.replicate.com/v1/predictions`  
   - Headers:  
     - `Authorization`: `Bearer {{$json.api_token}}`  
     - `Prefer`: `wait`  
   - Body Type: JSON  
   - Body JSON:  
     ```json
     {
       "version": "vcollos/trefilio:e1c19d7267f444e2796438803a31ccd8a1b62ef573531e35257032469e054f1a",
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
   - Options: Set to never error on HTTP errors (`neverError: true`)  
   - Connect output of Set Other Parameters → Create Other Prediction  

5. **Create Code Node "Log Request"**  
   - Type: Code (JavaScript)  
   - Code:  
     ```javascript
     const data = $input.all()[0].json;
     console.log('vcollos/trefilio Request:', {
       timestamp: new Date().toISOString(),
       prediction_id: data.id,
       model_type: 'other'
     });
     return $input.all();
     ```  
   - Connect output of Create Other Prediction → Log Request  

6. **Create Wait Node "Wait 5s"**  
   - Type: Wait  
   - Wait for 5 seconds  
   - Connect output of Log Request → Wait 5s  

7. **Create HTTP Request Node "Check Status"**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://api.replicate.com/v1/predictions/{{ $('Create Other Prediction').item.json.id }}`  
   - Headers:  
     - `Authorization`: `Bearer {{ $('Set API Token').item.json.api_token }}`  
   - Options: Set never error on HTTP errors  
   - Connect output of Wait 5s → Check Status  
   - Connect output of Wait 10s → Check Status (for retry loop)  

8. **Create If Node "Is Complete?"**  
   - Type: If  
   - Condition: `$json.status === "succeeded"`  
   - Connect output of Check Status → Is Complete?  

9. **Create Set Node "Success Response"**  
   - Type: Set  
   - Assign field `response` with:  
     ```js
     {
       success: true,
       result_url: $json.output,
       prediction_id: $json.id,
       status: $json.status,
       message: "Other generated successfully"
     }
     ```  
   - Connect True output of Is Complete? → Success Response  

10. **Create If Node "Has Failed?"**  
    - Type: If  
    - Condition: `$json.status === "failed"`  
    - Connect False output of Is Complete? → Has Failed?  

11. **Create Set Node "Error Response"**  
    - Type: Set  
    - Assign field `response` with:  
      ```js
      {
        success: false,
        error: $json.error || "Other generation failed",
        prediction_id: $json.id,
        status: $json.status,
        message: "Failed to generate other"
      }
      ```  
    - Connect True output of Has Failed? → Error Response  

12. **Create Wait Node "Wait 10s"**  
    - Type: Wait  
    - Wait for 10 seconds  
    - Connect False output of Has Failed? → Wait 10s  

13. **Create Set Node "Display Result"**  
    - Type: Set  
    - Assign field `final_result` to `{{$json.response}}`  
    - Connect output of Success Response → Display Result  
    - Connect output of Error Response → Display Result  

14. **Connect nodes according to the looping logic:**  
    - Wait 10s → Check Status (loop for retries)  

15. **Add Sticky Notes for Documentation:**  
    - Sticky Note with contact and social links (Sticky Note9)  
    - Sticky Note with detailed workflow overview, parameters, troubleshooting, and links (Sticky Note4)  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | Context or Link                                                                                                  |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Contact for questions/support: Yaron@nofluff.online. Explore more tips and tutorials on YouTube (https://www.youtube.com/@YaronBeen/videos) and LinkedIn (https://www.linkedin.com/in/yaronbeen/).                                                                                                                                                                                                                                                                                                                                                                                                                                                        | Sticky Note9                                                                                                    |
| Comprehensive workflow overview including model details, parameters guide, workflow components, benefits, quick start instructions, troubleshooting tips, and links to official documentation: Replicate model docs (https://replicate.com/vcollos/trefilio), Replicate API docs (https://replicate.com/docs), and n8n docs (https://docs.n8n.io). Emphasizes replacing the API token, parameter configuration, and monitoring usage and logs.                                                                                                                                                                                                                             | Sticky Note4                                                                                                    |
| Replicate API version used: `vcollos/trefilio:e1c19d7267f444e2796438803a31ccd8a1b62ef573531e35257032469e054f1a`. This version ID is critical for API calls and must match the deployed model version.                                                                                                                                                                                                                                                                                                                                                                                                                                                           | Workflow metadata                                                                                               |
| The workflow handles API rate limits and prediction processing delays with wait nodes and retry loops, ensuring resilience against transient errors and long-running jobs.                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | Workflow design note                                                                                            |
| Model parameters include image inpainting mask, seed, image URL, model variant, dimensions, prompt, speed mode, LoRA parameters, output format and quality, inference steps, and safety checker toggle. These parameters can be customized to tune generation output.                                                                                                                                                                                                                                                                                                                                                                                    | Parameter reference in Set Other Parameters node                                                               |

---

This document fully describes the "Generate Content with Vcollos Trefilio AI Model via Replicate API" workflow, enabling reproduction, modification, and integration with awareness of potential edge cases and error handling.