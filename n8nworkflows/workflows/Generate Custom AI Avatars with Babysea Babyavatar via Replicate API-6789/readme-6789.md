Generate Custom AI Avatars with Babysea Babyavatar via Replicate API

https://n8nworkflows.xyz/workflows/generate-custom-ai-avatars-with-babysea-babyavatar-via-replicate-api-6789


# Generate Custom AI Avatars with Babysea Babyavatar via Replicate API

### 1. Workflow Overview

This workflow automates the generation of custom AI avatars using the Babysea Babyavatar model via the Replicate API. It is designed for users who want to create unique, AI-generated avatars by providing image inputs and prompts. The workflow handles the entire process: parameter setup, API request submission, polling for generation status, error handling, and response formatting.

The workflow is logically divided into these blocks:

- **1.1 Input Initialization and Parameter Setup:** Receives manual trigger, sets API authentication token and model input parameters with defaults.
- **1.2 Prediction Creation:** Sends the generation request to Replicate API and obtains a prediction ID.
- **1.3 Status Polling Loop:** Waits and repeatedly checks the prediction status until it completes or fails.
- **1.4 Result Handling:** Routes success and failure responses to appropriate nodes, formatting the output.
- **1.5 Logging and Monitoring:** Logs request details for debugging and tracking.
- **1.6 Output Delivery:** Prepares final structured response data.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Initialization and Parameter Setup

**Overview:**  
This block initializes the workflow execution, sets the Replicate API token, and configures all input parameters required by the Babyavatar model with sensible defaults.

**Nodes Involved:**  
- Manual Trigger  
- Set API Token  
- Set Other Parameters  

**Node Details:**

- **Manual Trigger**  
  - Type: Trigger node for manual workflow start  
  - Configuration: No parameters, simply triggers the workflow  
  - Inputs: None  
  - Outputs: Next node ("Set API Token")  
  - Edge cases: None inherently, but workflow won't proceed without manual activation  

- **Set API Token**  
  - Type: Set node  
  - Role: Stores Replicate API token as a string variable `api_token`  
  - Configuration: Static placeholder string `"YOUR_REPLICATE_API_TOKEN"` — must be replaced with a valid token  
  - Inputs: From Manual Trigger  
  - Outputs: To “Set Other Parameters”  
  - Edge cases: Invalid or missing token will cause authentication errors downstream  

- **Set Other Parameters**  
  - Type: Set node  
  - Role: Defines all model input parameters including image URL, mask URL, prompt, output dimensions, model options, and flags  
  - Configuration:  
    - Copies `api_token` from previous node  
    - Sets default values such as:  
      - mask: placeholder black image URL  
      - seed: -1 (for randomization)  
      - image: sample 512x512 photo URL  
      - width/height: 1024x1024  
      - prompt: "An astronaut riding a rainbow unicorn"  
      - refine: "no_refiner"  
      - scheduler: "K_EULER"  
      - lora_scale: 0.6  
      - num_outputs: 1  
      - refine_steps: 1  
      - guidance_scale: 7.5  
      - apply_watermark: true  
      - high_noise_frac: 0.8  
      - negative_prompt: (empty)  
      - prompt_strength: 0.8  
      - replicate_weights: (empty)  
      - num_inference_steps: 50  
      - disable_safety_checker: false  
  - Inputs: From “Set API Token”  
  - Outputs: To “Create Other Prediction”  
  - Edge cases: Invalid parameter types or values may cause API errors  

---

#### 2.2 Prediction Creation

**Overview:**  
Sends a POST request to the Replicate API to initiate an AI avatar generation prediction with the configured parameters.

**Nodes Involved:**  
- Create Other Prediction  

**Node Details:**

- **Create Other Prediction**  
  - Type: HTTP Request node  
  - Role: Sends generation request to Replicate API endpoint `/v1/predictions`  
  - Configuration:  
    - Method: POST  
    - URL: `https://api.replicate.com/v1/predictions`  
    - Headers:  
      - Authorization: Bearer token from `api_token`  
      - Prefer: wait (to await processing)  
    - JSON body: Includes model version and all input parameters dynamically injected via expressions from previous "Set Other Parameters" node  
    - Response format: JSON, with neverError flag to prevent node failure on HTTP errors  
  - Inputs: From “Set Other Parameters”  
  - Outputs: To “Log Request”  
  - Edge cases:  
    - Network errors or invalid token cause failure  
    - API rate limits or invalid parameters may cause error responses  

---

#### 2.3 Status Polling Loop

**Overview:**  
Waits and polls the prediction status until it is either succeeded or failed, enabling asynchronous processing with retries.

**Nodes Involved:**  
- Log Request  
- Wait 5s  
- Check Status  
- Is Complete?  
- Has Failed?  
- Wait 10s  

**Node Details:**

- **Log Request**  
  - Type: Code node (JavaScript)  
  - Role: Logs prediction details (timestamp, prediction ID, model type) for monitoring  
  - Inputs: From “Create Other Prediction”  
  - Outputs: To “Wait 5s”  
  - Edge cases: Console log may not be visible in some runtime environments  

- **Wait 5s**  
  - Type: Wait node  
  - Role: Delays processing for 5 seconds before checking prediction status  
  - Inputs: From “Log Request”  
  - Outputs: To “Check Status”  
  - Edge cases: Minimal, but network delays may affect timing  

- **Check Status**  
  - Type: HTTP Request node  
  - Role: GET request to fetch current prediction status by prediction ID  
  - Configuration:  
    - URL: `https://api.replicate.com/v1/predictions/{{prediction_id}}` dynamically from "Create Other Prediction"  
    - Authorization header with Bearer token  
    - Response format: JSON, neverError to avoid node failure on errors  
  - Inputs: From “Wait 5s” or “Wait 10s”  
  - Outputs: To “Is Complete?”  
  - Edge cases: API unavailability or token expiry causes failure or stale data  

- **Is Complete?**  
  - Type: If node  
  - Role: Checks if prediction status equals `"succeeded"`  
  - Inputs: From “Check Status”  
  - Outputs:  
    - True: To “Success Response”  
    - False: To “Has Failed?”  
  - Edge cases: Status string comparison could fail if API changes response format  

- **Has Failed?**  
  - Type: If node  
  - Role: Checks if prediction status equals `"failed"`  
  - Inputs: From “Is Complete?” (False branch)  
  - Outputs:  
    - True: To “Error Response”  
    - False: To “Wait 10s” (retry)  
  - Edge cases: Infinite loop risk if status is neither succeeded nor failed; no timeout implemented  

- **Wait 10s**  
  - Type: Wait node  
  - Role: Delay before retrying status check after failure status check returns false  
  - Inputs: From “Has Failed?” (False branch)  
  - Outputs: To “Check Status” (loop)  
  - Edge cases: Longer wait time balances API rate limits and responsiveness  

---

#### 2.4 Result Handling

**Overview:**  
Processes the final results of the prediction, formatting success or error responses for downstream use.

**Nodes Involved:**  
- Success Response  
- Error Response  
- Display Result  

**Node Details:**

- **Success Response**  
  - Type: Set node  
  - Role: Constructs a success JSON object with keys: `success: true`, `result_url`, `prediction_id`, `status`, and a message  
  - Inputs: From “Is Complete?” (True branch)  
  - Outputs: To “Display Result”  
  - Edge cases: Assumes `output` field exists and contains usable URL(s)  

- **Error Response**  
  - Type: Set node  
  - Role: Constructs an error JSON object with keys: `success: false`, `error` message, `prediction_id`, `status`, and a message  
  - Inputs: From “Has Failed?” (True branch)  
  - Outputs: To “Display Result”  
  - Edge cases: May receive generic error if no detailed error info present  

- **Display Result**  
  - Type: Set node  
  - Role: Sets final result object containing either success or error response for output  
  - Inputs: From both “Success Response” and “Error Response”  
  - Outputs: None (end of workflow)  
  - Edge cases: None  

---

#### 2.5 Logging and Monitoring

**Overview:**  
Provides runtime logging of requests and response details for auditing and debugging.

**Nodes Involved:**  
- Log Request (already described in 2.3)  

**Node Details:**  
- See above for “Log Request” node details.

---

#### 2.6 Output Delivery

**Overview:**  
Prepares the final structured output object (`final_result`) containing success or failure information, ready for consumption by other systems or users.

**Nodes Involved:**  
- Display Result (described above)  

---

### 3. Summary Table

| Node Name           | Node Type            | Functional Role                          | Input Node(s)              | Output Node(s)            | Sticky Note                                                                                                    |
|---------------------|----------------------|----------------------------------------|----------------------------|---------------------------|---------------------------------------------------------------------------------------------------------------|
| Manual Trigger      | Manual Trigger        | Starts workflow execution               | None                       | Set API Token             |                                                                                                               |
| Set API Token       | Set                   | Stores Replicate API token              | Manual Trigger             | Set Other Parameters      |                                                                                                               |
| Set Other Parameters| Set                   | Configures model input parameters       | Set API Token              | Create Other Prediction   |                                                                                                               |
| Create Other Prediction | HTTP Request       | Sends generation request to Replicate API | Set Other Parameters      | Log Request               |                                                                                                               |
| Log Request         | Code                  | Logs prediction details                  | Create Other Prediction    | Wait 5s                   |                                                                                                               |
| Wait 5s             | Wait                  | Waits 5 seconds before status check     | Log Request                | Check Status              |                                                                                                               |
| Check Status        | HTTP Request          | Polls prediction status from API        | Wait 5s, Wait 10s          | Is Complete?              |                                                                                                               |
| Is Complete?        | If                    | Checks if prediction succeeded          | Check Status               | Success Response, Has Failed? |                                                                                                               |
| Has Failed?         | If                    | Checks if prediction failed              | Is Complete? (False branch)| Error Response, Wait 10s  |                                                                                                               |
| Wait 10s            | Wait                  | Waits 10 seconds before retrying status | Has Failed? (False branch) | Check Status              |                                                                                                               |
| Success Response    | Set                   | Formats success JSON response            | Is Complete? (True branch) | Display Result            |                                                                                                               |
| Error Response      | Set                   | Formats error JSON response              | Has Failed? (True branch)  | Display Result            |                                                                                                               |
| Display Result      | Set                   | Sets final output object                 | Success Response, Error Response | None                 |                                                                                                               |
| Sticky Note9        | Sticky Note           | Contact info and support links           | None                       | None                      | =======================================\nBABYAVATAR GENERATOR\nFor questions: Yaron@nofluff.online\nYouTube: https://www.youtube.com/@YaronBeen/videos\nLinkedIn: https://www.linkedin.com/in/yaronbeen/\n======================================= |
| Sticky Note4        | Sticky Note           | Full workflow documentation and usage guide | None                    | None                      | ## 🤖 **BABYSEA/BABYAVATAR - OTHER GENERATION WORKFLOW**\nPowered by Replicate API and n8n Automation\nModel docs: https://replicate.com/babysea/babyavatar\nReplicate API Docs: https://replicate.com/docs\nn8n Docs: https://docs.n8n.io |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node:**  
   - Name: `Manual Trigger`  
   - Purpose: To manually start the workflow.

2. **Add a Set node to store API token:**  
   - Name: `Set API Token`  
   - Add string field `api_token` with value `"YOUR_REPLICATE_API_TOKEN"` (replace with your actual token).  
   - Connect from `Manual Trigger`.

3. **Add another Set node to configure all model parameters:**  
   - Name: `Set Other Parameters`  
   - Copy `api_token` from `Set API Token` node via expression `={{ $('Set API Token').item.json.api_token }}`.  
   - Add the following parameters with their types and default values:  
     - mask (string): `"https://via.placeholder.com/512x512/000000/FFFFFF.png"`  
     - seed (number): `-1`  
     - image (string): `"https://picsum.photos/512/512"`  
     - width (number): `1024`  
     - height (number): `1024`  
     - prompt (string): `"An astronaut riding a rainbow unicorn"`  
     - refine (string): `"no_refiner"`  
     - scheduler (string): `"K_EULER"`  
     - lora_scale (number): `0.6`  
     - num_outputs (number): `1`  
     - refine_steps (number): `1`  
     - guidance_scale (number): `7.5`  
     - apply_watermark (boolean): `true`  
     - high_noise_frac (number): `0.8`  
     - negative_prompt (string): `""` (empty)  
     - prompt_strength (number): `0.8`  
     - replicate_weights (string): `""` (empty)  
     - num_inference_steps (number): `50`  
     - disable_safety_checker (boolean): `false`  
   - Connect from `Set API Token`.

4. **Create HTTP Request node to submit prediction:**  
   - Name: `Create Other Prediction`  
   - Method: POST  
   - URL: `https://api.replicate.com/v1/predictions`  
   - Headers:  
     - Authorization: `Bearer {{ $json.api_token }}`  
     - Prefer: `wait`  
   - Body (JSON):  
     ```json
     {
       "version": "babysea/babyavatar:f879b89acc5b931e4d180de5ef0e89782a4937cd4ae1d59be9a502a01a7b8c8c",
       "input": {
         "mask": "{{ $json.mask }}",
         "seed": {{ $json.seed }},
         "image": "{{ $json.image }}",
         "width": {{ $json.width }},
         "height": {{ $json.height }},
         "prompt": "{{ $json.prompt }}",
         "refine": "{{ $json.refine }}",
         "scheduler": "{{ $json.scheduler }}",
         "lora_scale": {{ $json.lora_scale }},
         "num_outputs": {{ $json.num_outputs }},
         "refine_steps": {{ $json.refine_steps }},
         "guidance_scale": {{ $json.guidance_scale }},
         "apply_watermark": {{ $json.apply_watermark }},
         "high_noise_frac": {{ $json.high_noise_frac }},
         "negative_prompt": "{{ $json.negative_prompt }}",
         "prompt_strength": {{ $json.prompt_strength }},
         "replicate_weights": "{{ $json.replicate_weights }}",
         "num_inference_steps": {{ $json.num_inference_steps }},
         "disable_safety_checker": {{ $json.disable_safety_checker }}
       }
     }
     ```
   - Enable “Send Body” and “Send Headers” and specify body as JSON.  
   - Connect from `Set Other Parameters`.

5. **Add a Code node to log request details:**  
   - Name: `Log Request`  
   - Code:  
     ```javascript
     const data = $input.all()[0].json;
     console.log('babysea/babyavatar Request:', {
       timestamp: new Date().toISOString(),
       prediction_id: data.id,
       model_type: 'other'
     });
     return $input.all();
     ```
   - Connect from `Create Other Prediction`.

6. **Add a Wait node to delay 5 seconds:**  
   - Name: `Wait 5s`  
   - Parameters: Wait for 5 seconds  
   - Connect from `Log Request`.

7. **Add HTTP Request node to check prediction status:**  
   - Name: `Check Status`  
   - Method: GET  
   - URL: `https://api.replicate.com/v1/predictions/{{ $('Create Other Prediction').item.json.id }}`  
   - Header: Authorization: `Bearer {{ $('Set API Token').item.json.api_token }}`  
   - Response format: JSON, neverError enabled  
   - Connect from `Wait 5s` and also from `Wait 10s` (later step for retry loop).

8. **Add an If node to check if prediction succeeded:**  
   - Name: `Is Complete?`  
   - Condition: `$json.status === "succeeded"`  
   - Connect from `Check Status`.

9. **Add a Set node for success response:**  
   - Name: `Success Response`  
   - Set field `response` as object:  
     ```json
     {
       "success": true,
       "result_url": $json.output,
       "prediction_id": $json.id,
       "status": $json.status,
       "message": "Other generated successfully"
     }
     ```  
   - Connect from `Is Complete?` (True branch).

10. **Add an If node to check if prediction failed:**  
    - Name: `Has Failed?`  
    - Condition: `$json.status === "failed"`  
    - Connect from `Is Complete?` (False branch).

11. **Add a Set node for error response:**  
    - Name: `Error Response`  
    - Set field `response` as object:  
      ```json
      {
        "success": false,
        "error": $json.error || "Other generation failed",
        "prediction_id": $json.id,
        "status": $json.status,
        "message": "Failed to generate other"
      }
      ```  
    - Connect from `Has Failed?` (True branch).

12. **Add a Wait node for retry delay:**  
    - Name: `Wait 10s`  
    - Wait for 10 seconds  
    - Connect from `Has Failed?` (False branch).

13. **Close the polling loop:**  
    - Connect `Wait 10s` output back to `Check Status` input.

14. **Add a Set node to prepare final output:**  
    - Name: `Display Result`  
    - Set field `final_result` equal to `{{$json.response}}`  
    - Connect from both `Success Response` and `Error Response`.

15. **Validate connections:**  
    - `Manual Trigger` → `Set API Token` → `Set Other Parameters` → `Create Other Prediction` → `Log Request` → `Wait 5s` → `Check Status` → `Is Complete?`  
    - `Is Complete? (True)` → `Success Response` → `Display Result`  
    - `Is Complete? (False)` → `Has Failed?`  
    - `Has Failed? (True)` → `Error Response` → `Display Result`  
    - `Has Failed? (False)` → `Wait 10s` → loops back to `Check Status`

16. **Credential Setup:**  
    - Configure HTTP Request nodes with HTTP Header Authorization using the `api_token` value.  
    - Ensure `api_token` contains a valid Replicate API token with permissions to call the babyavatar model.

---

### 5. General Notes & Resources

| Note Content                                                                                                                              | Context or Link                                                                                          |
|-------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| =======================================<br>BABYAVATAR GENERATOR<br>For any questions or support, contact:<br>Yaron@nofluff.online<br>YouTube: https://www.youtube.com/@YaronBeen/videos<br>LinkedIn: https://www.linkedin.com/in/yaronbeen/<br>======================================= | Author and support contact information included as Sticky Note9                                         |
| Full workflow detailed documentation including parameter explanations, usage instructions, troubleshooting, and links to official docs.  | https://replicate.com/babysea/babyavatar<br>https://replicate.com/docs<br>https://docs.n8n.io            |
| Replace `"YOUR_REPLICATE_API_TOKEN"` with your actual valid API token from https://replicate.com account settings before running workflow. | Essential for authentication and successful API calls                                                  |
| Workflow implements polling with 5 and 10 seconds wait nodes to handle asynchronous AI generation status checking gracefully.               | Ensures efficient API usage and avoids overload                                                        |
| Default parameters are provided for quick testing but should be customized for specific avatar generation use cases.                        | Parameter details available in Sticky Note4                                                              |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.