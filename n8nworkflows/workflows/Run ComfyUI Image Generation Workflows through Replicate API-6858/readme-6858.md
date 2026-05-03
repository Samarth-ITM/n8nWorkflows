Run ComfyUI Image Generation Workflows through Replicate API

https://n8nworkflows.xyz/workflows/run-comfyui-image-generation-workflows-through-replicate-api-6858


# Run ComfyUI Image Generation Workflows through Replicate API

### 1. Workflow Overview

This workflow facilitates running image generation workflows created with ComfyUI via the Replicate API. It is designed for users who want to automate the execution of custom ComfyUI workflows remotely, leveraging Replicate’s hosted inference service. The workflow handles authentication, parameter setup, prediction request submission, polling for completion status, error handling, and result delivery.

Logical blocks:

- **1.1 Trigger & Authentication Setup**: Receives manual start and sets the Replicate API token.
- **1.2 Parameter Configuration**: Prepares input parameters for the ComfyUI workflow execution.
- **1.3 Prediction Creation & Logging**: Submits a generation request to Replicate and logs request details.
- **1.4 Status Polling Loop**: Waits and checks prediction status repeatedly until completion or failure.
- **1.5 Outcome Handling**: Processes success or failure responses and formats final output.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger & Authentication Setup

**Overview:**  
This block initiates the workflow execution manually and configures the API token required to authenticate requests to the Replicate API.

**Nodes Involved:**  
- Manual Trigger  
- Set API Token

**Node Details:**

- **Manual Trigger**  
  - Type: Trigger node  
  - Role: Starts the workflow manually by user action.  
  - Configuration: No parameters; simple manual start.  
  - Input: None  
  - Output: Triggers next node  
  - Edge Cases: None significant; manual initiation only.

- **Set API Token**  
  - Type: Set node  
  - Role: Assigns the Replicate API token to a variable for authentication.  
  - Configuration: Sets a string variable `api_token` with placeholder `"YOUR_REPLICATE_API_TOKEN"` that must be replaced by user.  
  - Input: Receives trigger from Manual Trigger  
  - Output: Passes `api_token` to downstream nodes  
  - Edge Cases: Missing or invalid token will cause API authentication failures downstream.

---

#### 1.2 Parameter Configuration

**Overview:**  
Prepares all required and optional input parameters for the ComfyUI workflow run, including input image URL, output format, workflow JSON, and various flags.

**Nodes Involved:**  
- Set Other Parameters

**Node Details:**

- **Set Other Parameters**  
  - Type: Set node  
  - Role: Collects and sets parameters for the prediction request.  
  - Configuration:  
    - Copies `api_token` from previous node.  
    - Sets:  
      - `input_file` (default: `"https://picsum.photos/512/512"`)  
      - `output_format` (default: `"webp"`)  
      - `workflow_json` (default: empty string)  
      - `output_quality` (default: 95)  
      - `randomise_seeds` (default: true)  
      - `force_reset_cache` (default: false)  
      - `return_temp_files` (default: false)  
  - Input: From Set API Token  
  - Output: Passes parameters to prediction creation  
  - Edge Cases: Invalid URLs or malformed workflow JSON may cause prediction errors.

---

#### 1.3 Prediction Creation & Logging

**Overview:**  
Sends a POST request to the Replicate API to create a new prediction job for the ComfyUI workflow, then logs the request details for monitoring.

**Nodes Involved:**  
- Create Other Prediction  
- Log Request

**Node Details:**

- **Create Other Prediction**  
  - Type: HTTP Request node  
  - Role: Submits generation request to Replicate API.  
  - Configuration:  
    - POST `https://api.replicate.com/v1/predictions`  
    - Body JSON includes the version string of the ComfyUI workflow and input parameters from previous node.  
    - Headers include `Authorization: Bearer <api_token>` and `Prefer: wait` for synchronous response.  
    - Response format JSON with no error throw (`neverError: true`) to allow custom error handling.  
  - Input: Parameters from Set Other Parameters  
  - Output: Contains prediction ID and initial status  
  - Edge Cases: Possible auth errors, network timeouts, or invalid parameter errors.

- **Log Request**  
  - Type: Code node (JavaScript)  
  - Role: Logs prediction request details (timestamp, prediction ID, model type) to console for monitoring.  
  - Configuration: Custom JS code logging JSON data of the request.  
  - Input: From Create Other Prediction  
  - Output: Forwards data downstream for further processing  
  - Edge Cases: Logging failures do not affect workflow operation.

---

#### 1.4 Status Polling Loop

**Overview:**  
Implements a polling mechanism that waits and checks the status of the prediction job until it completes successfully or fails.

**Nodes Involved:**  
- Wait 5s  
- Check Status  
- Is Complete? (If node)  
- Has Failed? (If node)  
- Wait 10s

**Node Details:**

- **Wait 5s**  
  - Type: Wait node  
  - Role: Pauses workflow for 5 seconds before checking status.  
  - Configuration: Fixed 5 seconds delay.  
  - Input: From Log Request  
  - Output: To Check Status  
  - Edge Cases: Delay too short may cause unnecessary rapid polling.

- **Check Status**  
  - Type: HTTP Request node  
  - Role: Queries prediction status from Replicate API using prediction ID.  
  - Configuration:  
    - GET `https://api.replicate.com/v1/predictions/{{prediction_id}}`  
    - Authorization header with API token  
    - `neverError: true` for graceful error handling  
  - Input: From Wait 5s or Wait 10s  
  - Output: JSON status of prediction  
  - Edge Cases: Network issues, invalid IDs, or auth failures.

- **Is Complete?** (If Node)  
  - Type: If node  
  - Role: Checks if prediction status equals `"succeeded"`.  
  - Input: From Check Status  
  - Output:  
    - True branch: Success Response  
    - False branch: Has Failed? node  
  - Edge Cases: Status field missing or unexpected values.

- **Has Failed?** (If Node)  
  - Type: If node  
  - Role: Checks if prediction status equals `"failed"`.  
  - Input: From Is Complete? false branch  
  - Output:  
    - True branch: Error Response  
    - False branch: Wait 10s  
  - Edge Cases: Other statuses (e.g., "processing") cause another wait cycle.

- **Wait 10s**  
  - Type: Wait node  
  - Role: Pauses for 10 seconds before rechecking status to avoid rapid polling on failure or long processing.  
  - Configuration: Fixed 10 seconds delay.  
  - Input: From Has Failed? false branch  
  - Output: Back to Check Status (loop)  
  - Edge Cases: Longer wait increases total processing time but reduces API calls.

---

#### 1.5 Outcome Handling

**Overview:**  
Generates structured JSON responses for success or failure and prepares the final result for output.

**Nodes Involved:**  
- Success Response  
- Error Response  
- Display Result

**Node Details:**

- **Success Response**  
  - Type: Set node  
  - Role: Creates a JSON object indicating success with prediction details and output URL.  
  - Configuration: Sets a `response` object with keys: `success: true`, `result_url`, `prediction_id`, `status`, and message.  
  - Input: From Is Complete? true branch  
  - Output: To Display Result  
  - Edge Cases: Output URL missing or invalid.

- **Error Response**  
  - Type: Set node  
  - Role: Creates a JSON object indicating failure with error info and prediction details.  
  - Configuration: Sets a `response` object with keys: `success: false`, `error` text, `prediction_id`, `status`, and message.  
  - Input: From Has Failed? true branch  
  - Output: To Display Result  
  - Edge Cases: Error message may be generic if specific error info missing.

- **Display Result**  
  - Type: Set node  
  - Role: Prepares final output object labeled `final_result` containing the response from success or error branch.  
  - Configuration: Simply assigns the `response` object to `final_result`.  
  - Input: From Success Response or Error Response  
  - Output: Ends workflow with final structured output  
  - Edge Cases: None significant.

---

### 3. Summary Table

| Node Name             | Node Type          | Functional Role                                   | Input Node(s)            | Output Node(s)           | Sticky Note                                                                                             |
|-----------------------|--------------------|-------------------------------------------------|--------------------------|--------------------------|--------------------------------------------------------------------------------------------------------|
| Manual Trigger        | Trigger            | Initiates workflow manually                      | None                     | Set API Token             |                                                                                                        |
| Set API Token         | Set                | Assigns Replicate API token                       | Manual Trigger           | Set Other Parameters      |                                                                                                        |
| Set Other Parameters  | Set                | Sets input and optional parameters for prediction| Set API Token            | Create Other Prediction   |                                                                                                        |
| Create Other Prediction| HTTP Request       | Sends prediction creation request to Replicate  | Set Other Parameters     | Log Request               |                                                                                                        |
| Log Request           | Code               | Logs prediction request details                   | Create Other Prediction  | Wait 5s                   |                                                                                                        |
| Wait 5s               | Wait               | Waits 5 seconds before checking status            | Log Request              | Check Status              |                                                                                                        |
| Check Status          | HTTP Request       | Gets prediction status from Replicate API        | Wait 5s, Wait 10s        | Is Complete?              |                                                                                                        |
| Is Complete?          | If                 | Checks if prediction succeeded                    | Check Status             | Success Response, Has Failed? |                                                                                                    |
| Has Failed?           | If                 | Checks if prediction failed                        | Is Complete?             | Error Response, Wait 10s  |                                                                                                        |
| Wait 10s              | Wait               | Waits 10 seconds before next status check         | Has Failed?              | Check Status              |                                                                                                        |
| Success Response      | Set                | Constructs success JSON response                   | Is Complete? (true)      | Display Result            |                                                                                                        |
| Error Response        | Set                | Constructs error JSON response                     | Has Failed? (true)       | Display Result            |                                                                                                        |
| Display Result        | Set                | Prepares final output response                     | Success Response, Error Response | None             |                                                                                                        |
| Sticky Note4          | Sticky Note        | Detailed workflow and parameter documentation     | None                     | None                     | See section 5 for full content including links and detailed explanation                                |
| Sticky Note9          | Sticky Note        | Contact info and branding notes                    | None                     | None                     | See section 5 for full content including links and detailed explanation                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: Start workflow manually.

2. **Create Set Node for API Token**  
   - Name: Set API Token  
   - Assign variable `api_token` to `"YOUR_REPLICATE_API_TOKEN"` (replace with actual token).  
   - Connect output of Manual Trigger to this node.

3. **Create Set Node for Other Parameters**  
   - Name: Set Other Parameters  
   - Assign variables:  
     - `api_token`: Copy from previous node’s `api_token`.  
     - `input_file`: `"https://picsum.photos/512/512"` (default input image URL).  
     - `output_format`: `"webp"`  
     - `workflow_json`: `""` (empty string by default)  
     - `output_quality`: `95`  
     - `randomise_seeds`: `true`  
     - `force_reset_cache`: `false`  
     - `return_temp_files`: `false`  
   - Connect output of Set API Token to this node.

4. **Create HTTP Request Node to Create Prediction**  
   - Name: Create Other Prediction  
   - Method: POST  
   - URL: `https://api.replicate.com/v1/predictions`  
   - Headers:  
     - `Authorization`: `Bearer {{$json.api_token}}`  
     - `Prefer`: `wait`  
   - Body: JSON format with keys:  
     - `version`: `"fofr/any-comfyui-workflow:67ed4ba04ce0842446e16c428b1be131452815d01810861f71d171f63e8ba8f0"`  
     - `input`: object containing all parameters from Set Other Parameters (`input_file`, `output_format`, etc.)  
   - Response: JSON, set `neverError` to true  
   - Connect output of Set Other Parameters to this node.

5. **Create Code Node to Log Request**  
   - Name: Log Request  
   - Code: Logs timestamp, prediction ID, and model type from input JSON to console.  
   - Connect output of Create Other Prediction to this node.

6. **Create Wait Node for 5 seconds**  
   - Name: Wait 5s  
   - Duration: 5 seconds  
   - Connect output of Log Request to this node.

7. **Create HTTP Request Node to Check Status**  
   - Name: Check Status  
   - Method: GET  
   - URL: `https://api.replicate.com/v1/predictions/{{$json.id}}` (use prediction ID from Create Other Prediction)  
   - Header: `Authorization: Bearer {{ $json.api_token }}` (from Set API Token node)  
   - Response: JSON, `neverError` true  
   - Connect output of Wait 5s and Wait 10s to this node (for polling loop).

8. **Create If Node “Is Complete?”**  
   - Condition: `$json.status` equals `"succeeded"`  
   - True branch: Connect to Success Response node  
   - False branch: Connect to Has Failed? node  
   - Connect output of Check Status to this node.

9. **Create If Node “Has Failed?”**  
   - Condition: `$json.status` equals `"failed"`  
   - True branch: Connect to Error Response node  
   - False branch: Connect to Wait 10s node  
   - Connect false branch of Is Complete? to this node.

10. **Create Wait Node for 10 seconds**  
    - Name: Wait 10s  
    - Duration: 10 seconds  
    - Connect false branch of Has Failed? to this node  
    - Connect output of Wait 10s back to Check Status node to continue polling.

11. **Create Set Node for Success Response**  
    - Name: Success Response  
    - Assign variable `response` as an object:  
      ```json
      {
        "success": true,
        "result_url": $json.output,
        "prediction_id": $json.id,
        "status": $json.status,
        "message": "Other generated successfully"
      }
      ```  
    - Connect true branch of Is Complete? to this node.

12. **Create Set Node for Error Response**  
    - Name: Error Response  
    - Assign variable `response` as an object:  
      ```json
      {
        "success": false,
        "error": $json.error || "Other generation failed",
        "prediction_id": $json.id,
        "status": $json.status,
        "message": "Failed to generate other"
      }
      ```  
    - Connect true branch of Has Failed? to this node.

13. **Create Set Node for Display Result**  
    - Name: Display Result  
    - Assign variable `final_result` with the value of `response` variable from previous node.  
    - Connect outputs of Success Response and Error Response to this node.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | Context or Link                                                                                             |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| =======================================  ANY-COMFYUI-WORKFLOW GENERATOR ======================================== For any questions or support, please contact: Yaron@nofluff.online Explore more tips and tutorials here: - YouTube: https://www.youtube.com/@YaronBeen/videos - LinkedIn: https://www.linkedin.com/in/yaronbeen/ =======================================                                                                                                                                                                                                                                                                                                                                                                 | Contact info and branding details from Sticky Note9                                                        |
| ## 🤖 **FOFR/ANY-COMFYUI-WORKFLOW - OTHER GENERATION WORKFLOW** 🔥 Powered by Replicate API and n8n Automation --- 📝 **Model Overview** - Owner: fofr - Model: any-comfyui-workflow - Type: Other Generation - API Endpoint: https://api.replicate.com/v1/predictions 🎯 What This Model Does: Run any ComfyUI workflow. Guide: https://github.com/replicate/cog-comfyui --- 📋 **Parameter Reference** 🔴 Required Parameters: None 🔵 Optional Parameters: input_file, output_format, workflow_json, output_quality, randomise_seeds, force_reset_cache, return_temp_files 📖 Detailed Parameter Guide: - input_file (string): Input image, video, tar or zip file. See https://github.com/replicate/cog-comfyui for input guidance. - output_format (string): Output image format, default "webp" - workflow_json (string): ComfyUI workflow JSON string or URL - output_quality (integer): 0-100 quality for output images - randomise_seeds (boolean): Randomize seeds, default true - force_reset_cache (boolean): Reset ComfyUI cache before run - return_temp_files (boolean): Return temporary files for debug --- 🔧 **Workflow Components Explained** (see above sections) --- 🌟 **Key Benefits** - Instant generation, automation, error resilience, production ready, customizable, efficient processing --- 🚀 **Quick Start Instructions** 1. Get API key at https://replicate.com 2. Replace YOUR_REPLICATE_API_TOKEN 3. Adjust parameters as needed 4. Run workflow and get results --- 🔍 **Troubleshooting Guide** Common issues: invalid token, parameter validation, timeouts, output format Best practices: test defaults, monitor usage, keep keys secure --- 🔗 Additional Resources: - Model docs: https://replicate.com/fofr/any-comfyui-workflow - Replicate API docs: https://replicate.com/docs - n8n docs: https://docs.n8n.io --- | Detailed documentation and user instructions from Sticky Note4                                                   |

---

**Disclaimer:** The provided text is extracted exclusively from an automated workflow created using n8n, a tool for integration and automation. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.