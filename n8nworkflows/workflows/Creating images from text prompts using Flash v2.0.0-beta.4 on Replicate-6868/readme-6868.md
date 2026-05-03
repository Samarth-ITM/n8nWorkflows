Creating images from text prompts using Flash v2.0.0-beta.4 on Replicate

https://n8nworkflows.xyz/workflows/creating-images-from-text-prompts-using-flash-v2-0-0-beta-4-on-replicate-6868


# Creating images from text prompts using Flash v2.0.0-beta.4 on Replicate

### 1. Workflow Overview

This n8n workflow automates the generation of images from text prompts using the Flash v2.0.0-beta.4 AI model hosted on Replicate. It is designed for users who want to create images programmatically by providing descriptive prompts and optional image/mask inputs. The workflow handles the entire lifecycle from input reception, model invocation, polling for completion, to result retrieval and error handling.

**Logical Blocks:**

- **1.1 Input Initialization:** Receive manual trigger, set and propagate API token and model parameters.
- **1.2 Prediction Request:** Submit generation request to the Replicate API with all configured parameters.
- **1.3 Status Polling:** Periodically check the prediction status until completion or failure.
- **1.4 Response Handling:** Branch on success or failure, format output accordingly.
- **1.5 Logging & Output:** Log request details and set final response data.
- **1.6 Documentation & Notes:** Sticky notes providing extensive documentation, usage instructions, troubleshooting, and contact info.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Initialization

**Overview:**  
This block initiates workflow execution and sets up all required parameters including authentication token and generation inputs.

**Nodes Involved:**  
- Manual Trigger  
- Set API Token  
- Set Other Parameters

**Node Details:**  

- **Manual Trigger**  
  - Type: manualTrigger  
  - Role: Starts the workflow manually.  
  - Configuration: Default, no parameters; user clicks to start.  
  - Input: None  
  - Output: Triggers "Set API Token"  
  - Edge Cases: None specific; manual start only.

- **Set API Token**  
  - Type: set  
  - Role: Store Replicate API token securely for reuse.  
  - Configuration: Assigns string variable `api_token` with placeholder `"YOUR_REPLICATE_API_TOKEN"`.  
  - Key Expression: None; hardcoded string meant to be replaced by user.  
  - Input: Manual Trigger output  
  - Output: Passes `api_token` downstream  
  - Edge Cases: Failure if token is invalid or not replaced by user.

- **Set Other Parameters**  
  - Type: set  
  - Role: Define all input parameters for the image generation model, including prompt, size, output format, and optional parameters like mask and seed.  
  - Configuration: Copies `api_token` from previous node; sets defaults for 19 parameters such as `prompt` (default: "Create something amazing"), `width`, `height` (512x512), `num_inference_steps` (28), etc.  
  - Key Expressions: `={{ $('Set API Token').item.json.api_token }}` to inherit API token.  
  - Input: Set API Token output  
  - Output: Passes full parameter set to next block  
  - Edge Cases: Parameter mismatch or invalid values may cause API errors.

---

#### 2.2 Prediction Request

**Overview:**  
Sends a POST request to Replicate API to initiate the image generation process with all parameters.

**Nodes Involved:**  
- Create Other Prediction

**Node Details:**  

- **Create Other Prediction**  
  - Type: httpRequest (POST)  
  - Role: Submit prediction request to Replicate API endpoint: `https://api.replicate.com/v1/predictions`.  
  - Configuration:  
    - JSON body includes model version identifier and all parameters from previous node dynamically inserted via expressions.  
    - Headers include `Authorization: Bearer {api_token}` and `Prefer: wait` (to wait for immediate processing).  
    - Response parsed as JSON with error suppression (`neverError: true`).  
  - Input: Set Other Parameters output  
  - Output: Returns prediction ID and initial response data  
  - Edge Cases: HTTP errors, auth failures, invalid parameters, API downtime.

---

#### 2.3 Status Polling

**Overview:**  
Waits and polls the prediction status repeatedly until completion or failure is confirmed.

**Nodes Involved:**  
- Log Request  
- Wait 5s  
- Check Status  
- Is Complete? (IF node)  
- Has Failed? (IF node)  
- Wait 10s

**Node Details:**  

- **Log Request**  
  - Type: code  
  - Role: Logs prediction request details including timestamp and prediction id for monitoring and debugging.  
  - Configuration: JavaScript code logs to console and passes input unchanged.  
  - Input: Create Other Prediction output  
  - Output: Wait 5s node  
  - Edge Cases: Console logging may not be visible in some environments.

- **Wait 5s**  
  - Type: wait  
  - Role: Pause for 5 seconds before checking prediction status to avoid rapid polling.  
  - Configuration: Wait 5 seconds  
  - Input: Log Request output  
  - Output: Check Status node  
  - Edge Cases: Delay may cause minor latency.

- **Check Status**  
  - Type: httpRequest (GET)  
  - Role: Query Replicate API for current status of prediction by id.  
  - Configuration: URL constructed dynamically using prediction id; authorization header included.  
  - Input: Wait 5s output  
  - Output: Is Complete? node  
  - Edge Cases: Network errors, API errors, status unavailable.

- **Is Complete?** (IF node)  
  - Type: if  
  - Role: Check if prediction status is `succeeded`.  
  - Configuration: Condition on `$json.status == "succeeded"`.  
  - Input: Check Status output  
  - Output: Success Response (true), Has Failed? (false)  
  - Edge Cases: Status field missing or unexpected values.

- **Has Failed?** (IF node)  
  - Type: if  
  - Role: Check if prediction status is `failed`.  
  - Configuration: Condition on `$json.status == "failed"`.  
  - Input: Is Complete? false output  
  - Output: Error Response (true), Wait 10s (false)  
  - Edge Cases: Status unknown or pending.

- **Wait 10s**  
  - Type: wait  
  - Role: Delay 10 seconds before rechecking status after failure or unknown status.  
  - Configuration: Wait 10 seconds  
  - Input: Has Failed? false output  
  - Output: Check Status node (loops)  
  - Edge Cases: Longer delay increases total wait time.

---

#### 2.4 Response Handling

**Overview:**  
Formats and outputs a structured JSON response based on success or failure of the generation process.

**Nodes Involved:**  
- Success Response  
- Error Response  
- Display Result

**Node Details:**  

- **Success Response**  
  - Type: set  
  - Role: Prepare a success JSON object including success flag, output URL(s), prediction id, status, and a message.  
  - Configuration: Sets variable `response` with keys: `success: true`, `result_url`, `prediction_id`, `status`, and message.  
  - Input: Is Complete? true output  
  - Output: Display Result node  
  - Edge Cases: Missing output URL may cause empty results.

- **Error Response**  
  - Type: set  
  - Role: Prepare an error JSON object including failure flag, error message, prediction id, status, and a message.  
  - Configuration: Sets variable `response` with keys: `success: false`, `error` (from JSON or default), `prediction_id`, `status`, and message.  
  - Input: Has Failed? true output  
  - Output: Display Result node  
  - Edge Cases: No error details from API may limit diagnostics.

- **Display Result**  
  - Type: set  
  - Role: Final node to consolidate the response object under `final_result`.  
  - Configuration: Copies `response` variable to `final_result`.  
  - Input: Success Response or Error Response outputs  
  - Output: Workflow output (end)  
  - Edge Cases: None.

---

#### 2.5 Documentation & Notes

**Overview:**  
Provides extensive in-workflow documentation via sticky notes to guide users on usage, parameters, troubleshooting, and contact information.

**Nodes Involved:**  
- Sticky Note4 (Main Documentation)  
- Sticky Note9 (Contact & Support Info)

**Node Details:**  

- **Sticky Note4**  
  - Type: stickyNote  
  - Role: Contains detailed markdown text describing the workflow, model parameters, usage instructions, troubleshooting tips, and links to resources.  
  - Position: Left side, spanning vertically for visibility.  
  - Edge Cases: None; purely informational.

- **Sticky Note9**  
  - Type: stickyNote  
  - Role: Contains contact information for support and links to video tutorials and LinkedIn profile for additional help.  
  - Position: Top-left corner.  
  - Edge Cases: None.

---

### 3. Summary Table

| Node Name             | Node Type               | Functional Role                     | Input Node(s)               | Output Node(s)          | Sticky Note                                                                                     |
|-----------------------|-------------------------|-----------------------------------|-----------------------------|-------------------------|------------------------------------------------------------------------------------------------|
| Manual Trigger        | manualTrigger           | Start workflow                    | -                           | Set API Token            |                                                                                                |
| Set API Token         | set                     | Store Replicate API token         | Manual Trigger              | Set Other Parameters     |                                                                                                |
| Set Other Parameters  | set                     | Define all model input parameters | Set API Token               | Create Other Prediction  |                                                                                                |
| Create Other Prediction | httpRequest (POST)       | Submit generation request         | Set Other Parameters        | Log Request              |                                                                                                |
| Log Request           | code                    | Log prediction request details    | Create Other Prediction     | Wait 5s                  |                                                                                                |
| Wait 5s               | wait                    | Pause before status polling       | Log Request                | Check Status             |                                                                                                |
| Check Status          | httpRequest (GET)       | Query prediction status           | Wait 5s                    | Is Complete?             |                                                                                                |
| Is Complete?          | if                      | Check if generation succeeded    | Check Status               | Success Response, Has Failed? |                                                                                                |
| Has Failed?           | if                      | Check if generation failed       | Is Complete? (false)       | Error Response, Wait 10s |                                                                                                |
| Wait 10s              | wait                    | Pause before retrying status check| Has Failed? (false)         | Check Status             |                                                                                                |
| Success Response      | set                     | Format success response JSON      | Is Complete? (true)        | Display Result           |                                                                                                |
| Error Response        | set                     | Format error response JSON        | Has Failed? (true)         | Display Result           |                                                                                                |
| Display Result        | set                     | Set final result output           | Success Response, Error Response | -                      |                                                                                                |
| Sticky Note4          | stickyNote              | Main documentation and instructions| -                          | -                       | See full documentation and parameter guide inside sticky note content                           |
| Sticky Note9          | stickyNote              | Contact and support information   | -                          | -                       | Contact: Yaron@nofluff.online; YouTube & LinkedIn links                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node**  
   - Name: `Manual Trigger`  
   - Purpose: Start the workflow on demand.

2. **Create a Set node to store API token**  
   - Name: `Set API Token`  
   - Parameters:  
     - Add string variable `api_token` with value `"YOUR_REPLICATE_API_TOKEN"` (replace with actual token).  
   - Connect `Manual Trigger` → `Set API Token`.

3. **Create another Set node for all other parameters**  
   - Name: `Set Other Parameters`  
   - Parameters:  
     - Copy `api_token` from `Set API Token` using expression: `={{ $('Set API Token').item.json.api_token }}`  
     - Set the following variables with default values (adjust as needed):  
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
   - Connect `Set API Token` → `Set Other Parameters`.

4. **Create HTTP Request node to submit prediction**  
   - Name: `Create Other Prediction`  
   - Method: POST  
   - URL: `https://api.replicate.com/v1/predictions`  
   - Headers:  
     - `Authorization`: `=Bearer {{ $json.api_token }}`  
     - `Prefer`: `wait`  
   - Body (JSON): Include model version and all input parameters using expressions like `{{ $json.prompt }}`, etc.  
   - Response format: JSON, never error.  
   - Connect `Set Other Parameters` → `Create Other Prediction`.

5. **Create Code node for logging**  
   - Name: `Log Request`  
   - Language: JavaScript  
   - Code:  
     ```js
     const data = $input.all()[0].json;
     console.log('settyan/flash-v2.0.0-beta.4 Request:', {
       timestamp: new Date().toISOString(),
       prediction_id: data.id,
       model_type: 'other'
     });
     return $input.all();
     ```  
   - Connect `Create Other Prediction` → `Log Request`.

6. **Create Wait node (5 seconds)**  
   - Name: `Wait 5s`  
   - Parameters: Wait for 5 seconds  
   - Connect `Log Request` → `Wait 5s`.

7. **Create HTTP Request node to check status**  
   - Name: `Check Status`  
   - Method: GET  
   - URL: `https://api.replicate.com/v1/predictions/{{ $('Create Other Prediction').item.json.id }}`  
   - Header: `Authorization: Bearer {{ $('Set API Token').item.json.api_token }}`  
   - Response format: JSON, never error.  
   - Connect `Wait 5s` → `Check Status`.

8. **Create IF node to check if status succeeded**  
   - Name: `Is Complete?`  
   - Condition: `$json.status == "succeeded"`  
   - Connect `Check Status` → `Is Complete?`.

9. **Create IF node to check if status failed**  
   - Name: `Has Failed?`  
   - Condition: `$json.status == "failed"`  
   - Connect `Is Complete?` (false output) → `Has Failed?`.

10. **Create Wait node (10 seconds) for retry delay**  
    - Name: `Wait 10s`  
    - Parameters: Wait for 10 seconds  
    - Connect `Has Failed?` (false output) → `Wait 10s`  
    - Connect `Wait 10s` → `Check Status` (loop for polling).

11. **Create Set node for success response**  
    - Name: `Success Response`  
    - Set variable `response` to an object:  
      ```json
      {
        "success": true,
        "result_url": $json.output,
        "prediction_id": $json.id,
        "status": $json.status,
        "message": "Other generated successfully"
      }
      ```  
    - Connect `Is Complete?` (true output) → `Success Response`.

12. **Create Set node for error response**  
    - Name: `Error Response`  
    - Set variable `response` to an object:  
      ```json
      {
        "success": false,
        "error": $json.error || "Other generation failed",
        "prediction_id": $json.id,
        "status": $json.status,
        "message": "Failed to generate other"
      }
      ```  
    - Connect `Has Failed?` (true output) → `Error Response`.

13. **Create Set node to finalize output**  
    - Name: `Display Result`  
    - Copy `response` variable to `final_result`.  
    - Connect both `Success Response` and `Error Response` → `Display Result`.

14. **Add Sticky Notes for documentation**  
    - Create sticky notes containing the detailed model info, parameter guide, troubleshooting tips, contact info, and resource links as per the original content.  
    - Position appropriately for user reference.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | Context or Link                                                                                  |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| =======================================<br>FLASH-V2.0.0-BETA.4 GENERATOR<br>For any questions or support, please contact:<br>Yaron@nofluff.online<br>Explore more tips and tutorials here:<br>- YouTube: https://www.youtube.com/@YaronBeen/videos<br>- LinkedIn: https://www.linkedin.com/in/yaronbeen/<br>=======================================                                                                                                                                                | Contact and support information with useful social media links                                  |
| Full detailed documentation inside sticky note including model overview, parameter list with explanations, workflow component descriptions, quick start instructions, troubleshooting guide, and relevant links:<br>- Model Documentation: https://replicate.com/settyan/flash-v2.0.0-beta.4<br>- Replicate API Docs: https://replicate.com/docs<br>- n8n Documentation: https://docs.n8n.io                                                                                                                                                                                                                                                         | Comprehensive in-workflow documentation for end-users and developers                            |
| Replace placeholder API token in `Set API Token` node with your personal Replicate API token available at https://replicate.com/account                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               | Credential setup requirement                                                                    |
| This workflow implements retry logic with 5-second and 10-second waits to balance API polling frequency and avoid rate limits                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | Design note on robust polling and error handling                                               |
| The model supports extensive parameters for fine-tuning image generation including masking, seed control, model selection, size, output format, guidance scales, and safety checker toggling                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | Parameter flexibility enables advanced use cases                                               |

---

**Disclaimer:**  
The provided text is exclusively derived from an automated n8n workflow. It complies strictly with current content policies and contains no illegal, offensive, or restricted material. All data processed is legal and publicly available.