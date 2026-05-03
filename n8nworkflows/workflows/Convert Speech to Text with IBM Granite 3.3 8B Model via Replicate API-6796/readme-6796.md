Convert Speech to Text with IBM Granite 3.3 8B Model via Replicate API

https://n8nworkflows.xyz/workflows/convert-speech-to-text-with-ibm-granite-3-3-8b-model-via-replicate-api-6796


# Convert Speech to Text with IBM Granite 3.3 8B Model via Replicate API

### 1. Workflow Overview

This workflow automates the process of converting speech audio into text using the IBM Granite 3.3 8B automatic speech recognition (ASR) model hosted on Replicate API. It is designed for users who want to transcribe audio files into text programmatically, leveraging a state-of-the-art AI model with minimal manual intervention.

The workflow logic is structured into the following functional blocks:

- **1.1 Input Initialization**: Receives manual trigger and sets API authentication and model parameters.
- **1.2 Prediction Request Creation**: Sends audio and parameters to Replicate API to initiate speech-to-text prediction.
- **1.3 Prediction Monitoring Loop**: Waits and polls the prediction status until completion or failure.
- **1.4 Result Handling**: Processes success or failure responses and formats output accordingly.
- **1.5 Logging and Monitoring**: Logs key request details for audit and debugging.
- **1.6 User Guidance and Documentation**: Sticky notes provide comprehensive instructions, parameter explanations, and contact information.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Initialization

**Overview:**  
This block starts the workflow execution, sets the required API token for authentication, and configures all parameters needed for the speech-to-text model call.

**Nodes Involved:**  
- Manual Trigger  
- Set API Token  
- Set Text Parameters

**Node Details:**

- **Manual Trigger**  
  - Type: Trigger node to start the workflow manually.  
  - Configuration: No parameters; simply fires the workflow on demand.  
  - Input: None (trigger).  
  - Output: Starts flow to "Set API Token".  
  - Failure modes: None, as it is a manual trigger.  

- **Set API Token**  
  - Type: Set node to define the Replicate API token.  
  - Configuration: Assigns a string variable `api_token` with placeholder "YOUR_REPLICATE_API_TOKEN". User must replace this with a valid token.  
  - Inputs: From Manual Trigger.  
  - Outputs: Passes `api_token` to next node.  
  - Edge cases: If token is invalid or not replaced, API calls will fail with authentication errors.  

- **Set Text Parameters**  
  - Type: Set node that configures all input parameters for the prediction request.  
  - Configuration:  
    - Copies `api_token` from previous node.  
    - Defines model parameters including:  
      - `seed` (number, default -1, which randomizes seed)  
      - `audio` (array, default URL to sample WAV file)  
      - `top_k` (number, 50)  
      - `top_p` (number, 0.9)  
      - `prompt`, `chat_template`, `system_prompt`, `stop_sequences` (strings, empty by default)  
      - `max_tokens` (512), `min_tokens` (0)  
      - `temperature` (0.6), `presence_penalty` and `frequency_penalty` (0)  
  - Inputs: From "Set API Token".  
  - Outputs: Prepared JSON parameters forwarded to prediction creation.  
  - Edge cases: Parameters must be validated as per API specs; invalid types or missing required fields may cause API rejection.  

---

#### 2.2 Prediction Request Creation

**Overview:**  
This block sends a POST request to Replicate API to start a speech-to-text prediction job with the configured parameters.

**Nodes Involved:**  
- Create Text Prediction

**Node Details:**

- **Create Text Prediction**  
  - Type: HTTP Request node.  
  - Configuration:  
    - URL: `https://api.replicate.com/v1/predictions`  
    - Method: POST  
    - Headers:  
      - `Authorization: Bearer <api_token>` from JSON parameter  
      - `Prefer: wait` to wait for synchronous response if possible  
    - Body: JSON constructed dynamically from parameters set previously, including model version ID and all input parameters.  
    - Response: JSON, never error flag set to true to avoid node failure on HTTP errors.  
  - Inputs: From "Set Text Parameters".  
  - Outputs: Prediction response with ID and initial status forwarded to logging.  
  - Edge cases:  
    - HTTP errors (e.g., 401 Unauthorized) if token invalid.  
    - API rate limits or quota exceeded.  
    - Network timeouts.  

---

#### 2.3 Prediction Monitoring Loop

**Overview:**  
Implements a polling mechanism to check the status of the prediction every few seconds until it either succeeds or fails.

**Nodes Involved:**  
- Log Request  
- Wait 5s  
- Check Status  
- Is Complete? (If node)  
- Has Failed? (If node)  
- Wait 10s  

**Node Details:**

- **Log Request**  
  - Type: Code node (JavaScript).  
  - Configuration: Logs prediction ID, timestamp, and model type to console for monitoring.  
  - Inputs: From "Create Text Prediction".  
  - Outputs: Proceeds to "Wait 5s".  
  - Edge cases: None, but console logs depend on n8n runtime environment.  

- **Wait 5s**  
  - Type: Wait node.  
  - Configuration: Waits 5 seconds before next action.  
  - Inputs: From "Log Request".  
  - Outputs: To "Check Status".  
  - Edge cases: Workflow delay may impact throughput but is necessary for API polling.  

- **Check Status**  
  - Type: HTTP Request node.  
  - Configuration:  
    - URL constructed dynamically: `https://api.replicate.com/v1/predictions/{{prediction_id}}`  
    - Method: GET (default)  
    - Headers: Authorization with `api_token` from "Set API Token".  
    - Response JSON expected, with never error flag enabled.  
  - Inputs: From "Wait 5s" and "Wait 10s" (retry path).  
  - Outputs: To "Is Complete?" node.  
  - Edge cases: API downtime, invalid prediction ID, auth errors.  

- **Is Complete?**  
  - Type: If node (conditional branching).  
  - Configuration: Checks if status equals "succeeded".  
  - Inputs: From "Check Status".  
  - Outputs:  
    - True branch: "Success Response" node.  
    - False branch: "Has Failed?" node.  

- **Has Failed?**  
  - Type: If node.  
  - Configuration: Checks if status equals "failed".  
  - Inputs: From "Is Complete?" false branch.  
  - Outputs:  
    - True branch: "Error Response".  
    - False branch: "Wait 10s" to retry status check.  

- **Wait 10s**  
  - Type: Wait node.  
  - Configuration: Waits 10 seconds before polling again.  
  - Inputs: From "Has Failed?" false branch.  
  - Outputs: Back to "Check Status" for polling loop.  
  - Edge cases: Long-running requests may cause workflow timeouts if too many retries.  

---

#### 2.4 Result Handling

**Overview:**  
Processes the final status and generates a structured JSON response indicating success or failure, passing results downstream.

**Nodes Involved:**  
- Success Response  
- Error Response  
- Display Result

**Node Details:**

- **Success Response**  
  - Type: Set node.  
  - Configuration: Sets a JSON object `response` including:  
    - `success: true`  
    - `result_url`: URL of generated text output from prediction output field  
    - `prediction_id`, `status`  
    - `message`: "Text generated successfully"  
  - Inputs: From "Is Complete?" true branch.  
  - Outputs: To "Display Result".  

- **Error Response**  
  - Type: Set node.  
  - Configuration: Sets a JSON object `response` including:  
    - `success: false`  
    - `error`: captures error message from API or generic failure message  
    - `prediction_id`, `status`  
    - `message`: "Failed to generate text"  
  - Inputs: From "Has Failed?" true branch.  
  - Outputs: To "Display Result".  

- **Display Result**  
  - Type: Set node.  
  - Configuration: Assigns the final `response` object to a field `final_result` for output or downstream use.  
  - Inputs: From both success and error response nodes.  
  - Outputs: End of workflow or further downstream processing.  

---

#### 2.5 Logging and Monitoring

**Overview:**  
Provides console logging of prediction requests for audit and debugging.

**Nodes Involved:**  
- Log Request (already described in 2.3)

**Details:**  
- Logs important metadata like timestamp, prediction ID, and model type to console for troubleshooting.

---

#### 2.6 User Guidance and Documentation

**Overview:**  
Sticky notes provide detailed contextual information about the workflow, model, parameters, usage instructions, and contact info.

**Nodes Involved:**  
- Sticky Note9  
- Sticky Note4

**Details:**

- **Sticky Note9**  
  - Contains contact email and links to YouTube and LinkedIn channels for support and tutorials.

- **Sticky Note4**  
  - Extensive documentation including:  
    - Model overview and API endpoint  
    - Parameter descriptions and defaults  
    - Workflow component explanations  
    - Quick start instructions with API token setup  
    - Troubleshooting tips  
    - Links to official Replicate and n8n documentation  

---

### 3. Summary Table

| Node Name           | Node Type           | Functional Role                 | Input Node(s)             | Output Node(s)              | Sticky Note                                                                                     |
|---------------------|---------------------|--------------------------------|---------------------------|-----------------------------|------------------------------------------------------------------------------------------------|
| Manual Trigger      | Trigger             | Initiates workflow             | -                         | Set API Token               |                                                                                                |
| Set API Token       | Set                 | Stores API token               | Manual Trigger            | Set Text Parameters          |                                                                                                |
| Set Text Parameters | Set                 | Configures text generation parameters | Set API Token             | Create Text Prediction       |                                                                                                |
| Create Text Prediction | HTTP Request      | Sends generation request to Replicate API | Set Text Parameters        | Log Request                 |                                                                                                |
| Log Request         | Code                | Logs prediction request details | Create Text Prediction    | Wait 5s                     |                                                                                                |
| Wait 5s             | Wait                | Delays 5 seconds before status check | Log Request               | Check Status                |                                                                                                |
| Check Status        | HTTP Request        | Polls prediction status       | Wait 5s, Wait 10s         | Is Complete?                |                                                                                                |
| Is Complete?        | If                  | Checks if prediction succeeded | Check Status              | Success Response, Has Failed? |                                                                                                |
| Has Failed?         | If                  | Checks if prediction failed    | Is Complete?              | Error Response, Wait 10s    |                                                                                                |
| Wait 10s            | Wait                | Delays 10 seconds before retry | Has Failed?               | Check Status                |                                                                                                |
| Success Response    | Set                 | Prepares success JSON response | Is Complete? (true branch) | Display Result              |                                                                                                |
| Error Response      | Set                 | Prepares error JSON response   | Has Failed? (true branch) | Display Result              |                                                                                                |
| Display Result      | Set                 | Outputs final response object  | Success Response, Error Response | -                      |                                                                                                |
| Sticky Note9        | Sticky Note         | Contact and support info       | -                         | -                           | Contains contact email and social links                                                        |
| Sticky Note4        | Sticky Note         | Comprehensive workflow documentation | -                         | -                           | Contains detailed usage instructions, parameter descriptions, and troubleshooting guide       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a "Manual Trigger" node to start the workflow manually. No configuration needed.

2. **Add "Set API Token" Node**  
   - Add a "Set" node named "Set API Token".  
   - Create a string variable `api_token` with the value `"YOUR_REPLICATE_API_TOKEN"`.  
   - Connect Manual Trigger → Set API Token.

3. **Add "Set Text Parameters" Node**  
   - Add another "Set" node named "Set Text Parameters".  
   - Copy the `api_token` from "Set API Token" node using expression: `={{ $('Set API Token').item.json.api_token }}`.  
   - Define parameters:  
     - seed: -1  
     - audio: `["https://www2.cs.uic.edu/~i101/SoundFiles/BabyElephantWalk60.wav"]` (array)  
     - top_k: 50  
     - top_p: 0.9  
     - prompt: "" (empty string)  
     - max_tokens: 512  
     - min_tokens: 0  
     - temperature: 0.6  
     - chat_template: ""  
     - system_prompt: ""  
     - stop_sequences: ""  
     - presence_penalty: 0  
     - frequency_penalty: 0  
   - Connect Set API Token → Set Text Parameters.

4. **Add "Create Text Prediction" HTTP Request Node**  
   - Add an "HTTP Request" node named "Create Text Prediction".  
   - Configure:  
     - Method: POST  
     - URL: `https://api.replicate.com/v1/predictions`  
     - Headers:  
       - Authorization: `Bearer {{ $json.api_token }}`  
       - Prefer: wait  
     - Body Content Type: JSON  
     - Body (raw JSON) using expression with all parameters:  
       ```json
       {
         "version": "ibm-granite/granite-speech-3.3-8b:688e7a943167401c310f0975cb68f1a35e0bddc3b65f60bde89c37860e07edf1",
         "input": {
           "seed": {{ $json.seed }},
           "audio": "{{ $json.audio }}",
           "top_k": {{ $json.top_k }},
           "top_p": {{ $json.top_p }},
           "prompt": "{{ $json.prompt }}",
           "max_tokens": {{ $json.max_tokens }},
           "min_tokens": {{ $json.min_tokens }},
           "temperature": {{ $json.temperature }},
           "chat_template": "{{ $json.chat_template }}",
           "system_prompt": "{{ $json.system_prompt }}",
           "stop_sequences": "{{ $json.stop_sequences }}",
           "presence_penalty": {{ $json.presence_penalty }},
           "frequency_penalty": {{ $json.frequency_penalty }}
         }
       }
       ```  
     - Enable "Never Error" in response options to prevent node failure on HTTP errors.  
   - Connect Set Text Parameters → Create Text Prediction.

5. **Add "Log Request" Code Node**  
   - Add a "Code" node named "Log Request".  
   - JavaScript code:  
     ```javascript
     const data = $input.all()[0].json;
     console.log('ibm-granite/granite-speech-3.3-8b Request:', {
       timestamp: new Date().toISOString(),
       prediction_id: data.id,
       model_type: 'text'
     });
     return $input.all();
     ```  
   - Connect Create Text Prediction → Log Request.

6. **Add "Wait 5s" Node**  
   - Add a "Wait" node named "Wait 5s".  
   - Set to wait 5 seconds.  
   - Connect Log Request → Wait 5s.

7. **Add "Check Status" HTTP Request Node**  
   - Add "HTTP Request" named "Check Status".  
   - Configure:  
     - URL: `https://api.replicate.com/v1/predictions/{{ $('Create Text Prediction').item.json.id }}`  
     - Headers: Authorization: `Bearer {{ $('Set API Token').item.json.api_token }}`  
     - Response type JSON, with "Never Error" enabled.  
   - Connect Wait 5s → Check Status.  

8. **Add "Is Complete?" If Node**  
   - Add an "If" node named "Is Complete?".  
   - Condition: `$json.status` equals `"succeeded"`.  
   - Connect Check Status → Is Complete?.

9. **Add "Has Failed?" If Node**  
   - Add another "If" node named "Has Failed?".  
   - Condition: `$json.status` equals `"failed"`.  
   - Connect Is Complete? (false branch) → Has Failed?.

10. **Add "Success Response" Set Node**  
    - Add "Set" node named "Success Response".  
    - Set variable `response` to object:  
      ```json
      {
        "success": true,
        "result_url": $json.output,
        "prediction_id": $json.id,
        "status": $json.status,
        "message": "Text generated successfully"
      }
      ```  
    - Connect Is Complete? (true branch) → Success Response.

11. **Add "Error Response" Set Node**  
    - Add "Set" node named "Error Response".  
    - Set variable `response` to object:  
      ```json
      {
        "success": false,
        "error": $json.error || "Text generation failed",
        "prediction_id": $json.id,
        "status": $json.status,
        "message": "Failed to generate text"
      }
      ```  
    - Connect Has Failed? (true branch) → Error Response.

12. **Add "Wait 10s" Node**  
    - Add a "Wait" node named "Wait 10s".  
    - Set to wait 10 seconds.  
    - Connect Has Failed? (false branch) → Wait 10s.  
    - Connect Wait 10s → Check Status (loop for polling).

13. **Add "Display Result" Set Node**  
    - Add "Set" node named "Display Result".  
    - Set variable `final_result` to the JSON `response` from either Success Response or Error Response node: `={{ $json.response }}`.  
    - Connect Success Response → Display Result.  
    - Connect Error Response → Display Result.

14. **Add Sticky Notes** (optional for documentation)  
    - Add two Sticky Note nodes with the content provided to assist users with context, parameter details, and support info.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | Context or Link                                                                                          |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| =======================================\n        GRANITE-SPEECH-3.3-8B GENERATOR\n=======================================\nFor any questions or support, please contact:\n    Yaron@nofluff.online\n\nExplore more tips and tutorials here:\n   - YouTube: https://www.youtube.com/@YaronBeen/videos\n   - LinkedIn: https://www.linkedin.com/in/yaronbeen/\n======================================= | Contact and support information including social media links.                                          |
| ## 🤖 **IBM-GRANITE/GRANITE-SPEECH-3.3-8B - TEXT GENERATION WORKFLOW**\n\n**🔥 Powered by Replicate API and n8n Automation**\n\n---\n\n### 📝 **Model Overview**\n\n- **Owner**: ibm-granite\n- **Model**: granite-speech-3.3-8b\n- **Type**: Text Generation\n- **API Endpoint**: https://api.replicate.com/v1/predictions\n\n**🎯 What This Model Does:**\nGranite-speech-3.3-8b is a compact and efficient speech-language model, specifically designed for automatic speech recognition (ASR) and automatic speech translation (AST).\n\n---\n\n### 📋 **Parameter Reference**\n\n**🔴 Required Parameters:** None\n**🔵 Optional Parameters:** seed, audio, top_k, top_p, prompt, max_tokens, min_tokens, temperature (and 5 more)\n\n... | Extensive documentation, parameter guide, instructions, and troubleshooting. See https://replicate.com/ibm-granite/granite-speech-3.3-8b and https://replicate.com/docs |

---

**Disclaimer:** The provided content originates solely from an automated n8n workflow. It adheres strictly to applicable content policies and contains no illegal, offensive, or protected material. All data processed is legal and publicly available.