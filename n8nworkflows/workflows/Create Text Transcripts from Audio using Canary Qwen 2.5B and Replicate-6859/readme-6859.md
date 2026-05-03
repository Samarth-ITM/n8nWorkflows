Create Text Transcripts from Audio using Canary Qwen 2.5B and Replicate

https://n8nworkflows.xyz/workflows/create-text-transcripts-from-audio-using-canary-qwen-2-5b-and-replicate-6859


# Create Text Transcripts from Audio using Canary Qwen 2.5B and Replicate

### 1. Workflow Overview

This workflow automates the process of creating text transcripts from audio files using the Canary Qwen 2.5B speech-to-text AI model hosted on the Replicate API. It is designed to accept an audio URL, send it to the AI model for transcription, and handle the asynchronous status checking until the transcription is complete or has failed. The workflow includes error handling, logging, and structured response formatting for integration with other systems or frontends.

The workflow can be logically divided into the following blocks:

- **1.1 Input Initialization and Parameter Setup**: Receives manual trigger and sets essential API credentials and transcription parameters.
- **1.2 Prediction Request Creation**: Sends transcription request to the Replicate API with configured parameters.
- **1.3 Asynchronous Status Polling Loop**: Waits and polls the prediction status until the transcription is finished or failed.
- **1.4 Result Handling and Response Formatting**: Processes success or failure results, formats responses, and displays output.
- **1.5 Logging and Monitoring**: Logs request details for debugging and operational monitoring.
- **1.6 Documentation and User Guidance**: Provides detailed documentation and usage instructions via sticky notes.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Initialization and Parameter Setup

**Overview:**  
This block starts the workflow manually and sets up all required parameters including the API token and transcription options such as the audio URL, prompt, confidence display, and timestamps.

**Nodes Involved:**  
- Manual Trigger  
- Set API Token  
- Set Text Parameters

**Node Details:**

- **Manual Trigger**  
  - Type: Manual trigger node  
  - Role: Initiates workflow execution on user command  
  - Config: No parameters; triggers manually  
  - Inputs: None  
  - Outputs: Connects to "Set API Token"  
  - Edge Cases: User forgetting to trigger manually; no automatic invocation

- **Set API Token**  
  - Type: Set node  
  - Role: Stores the Replicate API token securely within the workflow context  
  - Config: Hardcoded string assignment `"YOUR_REPLICATE_API_TOKEN"` (must be replaced by user)  
  - Inputs: Manual Trigger output  
  - Outputs: Connects to "Set Text Parameters"  
  - Edge Cases: Invalid or missing API token leading to authentication failures

- **Set Text Parameters**  
  - Type: Set node  
  - Role: Defines transcription input parameters for the API request  
  - Config:  
    - Copies API token from previous node  
    - Sets audio URL to a sample WAV file by default  
    - Defines empty LLM prompt  
    - Sets `show_confidence` to false (default)  
    - Sets `include_timestamps` to true (default)  
  - Inputs: From "Set API Token"  
  - Outputs: Connects to "Create Text Prediction" node  
  - Edge Cases: Incorrect audio URL or parameter types causing API errors

---

#### 2.2 Prediction Request Creation

**Overview:**  
Sends a POST request to the Replicate API to start the transcription prediction using the configured parameters.

**Nodes Involved:**  
- Create Text Prediction

**Node Details:**

- **Create Text Prediction**  
  - Type: HTTP Request node  
  - Role: Initiates the transcription job on Replicate API  
  - Config:  
    - Method: POST to `https://api.replicate.com/v1/predictions`  
    - JSON body includes:  
      - Model version ID and owner  
      - Input parameters: audio URL, LLM prompt, confidence toggle, timestamps toggle  
    - Headers: Authorization Bearer token using API token  
    - Header `Prefer: wait` to wait for synchronous response (but workflow implements async polling as well)  
    - Response format: JSON with error suppression (`neverError:true`)  
  - Inputs: From "Set Text Parameters"  
  - Outputs: Connects to "Log Request"  
  - Edge Cases: HTTP errors, invalid token, rate limiting, malformed request body

---

#### 2.3 Asynchronous Status Polling Loop

**Overview:**  
Since transcription is asynchronous, this block waits and polls the prediction status until completion or failure, implementing retry logic with delays.

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
  - Role: Logs prediction request details (timestamp, prediction ID, model type) for monitoring  
  - Config: JavaScript code logging to console  
  - Inputs: From "Create Text Prediction"  
  - Outputs: Connects to "Wait 5s"  
  - Edge Cases: Logging failures generally non-critical

- **Wait 5s**  
  - Type: Wait node  
  - Role: Delay 5 seconds before polling status  
  - Config: Fixed 5 seconds delay  
  - Inputs: From "Log Request" and re-entry from "Check Status" (retry loop)  
  - Outputs: Connects to "Check Status"  
  - Edge Cases: None significant; delay can be tuned

- **Check Status**  
  - Type: HTTP Request node  
  - Role: Queries Replicate API for the prediction's current status using prediction ID  
  - Config:  
    - GET request to `https://api.replicate.com/v1/predictions/{id}`  
    - Authorization header with API token  
    - Response format JSON, never error to continue workflow  
  - Inputs: From "Wait 5s" and "Wait 10s"  
  - Outputs: Connects to "Is Complete?" node  
  - Edge Cases: API downtime, network errors, invalid prediction ID

- **Is Complete?**  
  - Type: If node  
  - Role: Checks if prediction status is `"succeeded"`  
  - Config: Condition: `$json.status == "succeeded"`  
  - Inputs: From "Check Status"  
  - Outputs:  
    - True: "Success Response"  
    - False: "Has Failed?" node  
  - Edge Cases: Status field missing or unexpected values

- **Has Failed?**  
  - Type: If node  
  - Role: Checks if prediction status is `"failed"`  
  - Config: Condition: `$json.status == "failed"`  
  - Inputs: From "Is Complete?"  
  - Outputs:  
    - True: "Error Response"  
    - False: "Wait 10s" (retry delay)  
  - Edge Cases: Intermediate states like "processing" or "starting"

- **Wait 10s**  
  - Type: Wait node  
  - Role: Delays 10 seconds before retrying status check after a failure or non-completion state  
  - Config: Fixed 10 seconds delay  
  - Inputs: From "Has Failed?" (false branch)  
  - Outputs: Connects back to "Check Status" for retry  
  - Edge Cases: May cause longer-than-needed delays; tuning possible

---

#### 2.4 Result Handling and Response Formatting

**Overview:**  
Based on the prediction status, formats a structured JSON response indicating success with output URL or failure with error details, then outputs final result.

**Nodes Involved:**  
- Success Response  
- Error Response  
- Display Result

**Node Details:**

- **Success Response**  
  - Type: Set node  
  - Role: Constructs a success response object with details: success flag, result URL, prediction ID, status, and message  
  - Config: Uses expressions to extract from API response fields: `$json.output`, `$json.id`, `$json.status`  
  - Inputs: From "Is Complete?" (true branch)  
  - Outputs: Connects to "Display Result"  
  - Edge Cases: Missing or malformed output URL in success response

- **Error Response**  
  - Type: Set node  
  - Role: Constructs a failure response object with error message, prediction ID, status, and message  
  - Config: Extracts error from `$json.error` or defaults to string; includes prediction metadata  
  - Inputs: From "Has Failed?" (true branch)  
  - Outputs: Connects to "Display Result"  
  - Edge Cases: Missing error details in response

- **Display Result**  
  - Type: Set node  
  - Role: Sets final result payload accessible to downstream nodes or webhook responses  
  - Config: Assigns the response object stored in previous node’s `response` field to a new key `final_result`  
  - Inputs: From both "Success Response" and "Error Response"  
  - Outputs: Terminal output node (no further connections)  
  - Edge Cases: None significant

---

#### 2.5 Logging and Monitoring

**Overview:**  
Provides runtime logging of prediction requests for monitoring and debugging purposes.

**Nodes Involved:**  
- Log Request (covered in block 2.3)

**Node Details:**  
Described above in Asynchronous Status Polling Loop.

---

#### 2.6 Documentation and User Guidance

**Overview:**  
Contains sticky notes with detailed instructions, usage guide, parameter references, troubleshooting, and contact information for support.

**Nodes Involved:**  
- Sticky Note9  
- Sticky Note4

**Node Details:**

- **Sticky Note9**  
  - Type: Sticky Note  
  - Role: Brief branding and contact info for support  
  - Content: Contact email, YouTube and LinkedIn links for tutorials and support  
  - Placement: Top-left corner for visibility  
  - Edge Cases: None

- **Sticky Note4**  
  - Type: Sticky Note  
  - Role: Comprehensive workflow documentation including model overview, parameters, component explanations, quick start, troubleshooting, and external resource links  
  - Content: Extensive markdown-formatted guide embedded directly in the workflow canvas  
  - Placement: Large note on the left side for user reference  
  - Edge Cases: None

---

### 3. Summary Table

| Node Name           | Node Type        | Functional Role                       | Input Node(s)              | Output Node(s)             | Sticky Note                                                                                              |
|---------------------|------------------|------------------------------------|----------------------------|----------------------------|--------------------------------------------------------------------------------------------------------|
| Manual Trigger      | Manual Trigger   | Starts workflow execution           | None                       | Set API Token              |                                                                                                        |
| Set API Token       | Set              | Stores Replicate API token          | Manual Trigger             | Set Text Parameters        |                                                                                                        |
| Set Text Parameters | Set              | Sets transcription input parameters | Set API Token              | Create Text Prediction     |                                                                                                        |
| Create Text Prediction | HTTP Request    | Sends transcription request to Replicate | Set Text Parameters   | Log Request                |                                                                                                        |
| Log Request         | Code             | Logs request details for monitoring | Create Text Prediction     | Wait 5s                    |                                                                                                        |
| Wait 5s             | Wait             | Delays before status check          | Log Request, Wait 10s      | Check Status               |                                                                                                        |
| Check Status        | HTTP Request     | Polls transcription status          | Wait 5s, Wait 10s          | Is Complete?               |                                                                                                        |
| Is Complete?        | If               | Checks if transcription succeeded   | Check Status               | Success Response, Has Failed? |                                                                                                    |
| Has Failed?         | If               | Checks if transcription failed      | Is Complete?               | Error Response, Wait 10s   |                                                                                                        |
| Wait 10s            | Wait             | Delay before retrying status check  | Has Failed?                | Check Status               |                                                                                                        |
| Success Response    | Set              | Formats successful transcription response | Is Complete?           | Display Result             |                                                                                                        |
| Error Response      | Set              | Formats error response on failure   | Has Failed?                | Display Result             |                                                                                                        |
| Display Result      | Set              | Sets final structured output        | Success Response, Error Response | None                   |                                                                                                        |
| Sticky Note9        | Sticky Note      | Branding and support contact info   | None                       | None                      | =======================================\nCANARY-QWEN-2.5B GENERATOR\nFor questions/support:\nYaron@nofluff.online\nYouTube: https://www.youtube.com/@YaronBeen/videos\nLinkedIn: https://www.linkedin.com/in/yaronbeen/ |
| Sticky Note4        | Sticky Note      | Detailed workflow documentation and usage guide | None                 | None                      | https://replicate.com/zsxkib/canary-qwen-2.5b\nhttps://replicate.com/docs\nhttps://docs.n8n.io         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Manual Trigger node**  
   - Type: Manual Trigger  
   - No parameters  
   - This will start the workflow execution manually.

2. **Add a Set node named "Set API Token"**  
   - Assign a string value to a variable `api_token` with your Replicate API token (replace `"YOUR_REPLICATE_API_TOKEN"`).  
   - Connect Manual Trigger output to this node.

3. **Add a Set node named "Set Text Parameters"**  
   - Assign variables:  
     - `api_token`: copy from previous node (`{{$node["Set API Token"].json["api_token"]}}`)  
     - `audio`: set default test URL `"https://www2.cs.uic.edu/~i101/SoundFiles/BabyElephantWalk60.wav"`  
     - `llm_prompt`: empty string (optional prompt)  
     - `show_confidence`: boolean `false`  
     - `include_timestamps`: boolean `true`  
   - Connect output of "Set API Token" to this node.

4. **Add an HTTP Request node named "Create Text Prediction"**  
   - Method: POST  
   - URL: `https://api.replicate.com/v1/predictions`  
   - Headers:  
     - `Authorization`: `Bearer {{$json.api_token}}`  
     - `Prefer`: `wait`  
   - Body (JSON):  
     ```json
     {
       "version": "zsxkib/canary-qwen-2.5b:afba731fc7a4082730943a246233b09c7fa3dfb2c24b07fe199c1408a7c8cb2f",
       "input": {
         "audio": "{{$json.audio}}",
         "llm_prompt": "{{$json.llm_prompt}}",
         "show_confidence": {{$json.show_confidence}},
         "include_timestamps": {{$json.include_timestamps}}
       }
     }
     ```  
   - Response format: JSON, never error = true  
   - Connect "Set Text Parameters" output here.

5. **Add a Code node named "Log Request"**  
   - JavaScript code:  
     ```javascript
     const data = $input.all()[0].json;
     console.log('zsxkib/canary-qwen-2.5b Request:', {
       timestamp: new Date().toISOString(),
       prediction_id: data.id,
       model_type: 'text'
     });
     return $input.all();
     ```  
   - Connect "Create Text Prediction" output here.

6. **Add a Wait node named "Wait 5s"**  
   - Wait for 5 seconds  
   - Connect "Log Request" output to "Wait 5s"

7. **Add an HTTP Request node named "Check Status"**  
   - Method: GET  
   - URL: `https://api.replicate.com/v1/predictions/{{$node["Create Text Prediction"].json["id"]}}`  
   - Header: `Authorization: Bearer {{$node["Set API Token"].json["api_token"]}}`  
   - Response format: JSON, never error = true  
   - Connect "Wait 5s" output here (and also later from "Wait 10s" for retries).

8. **Add an If node named "Is Complete?"**  
   - Condition: check if `{{$json.status}}` equals `"succeeded"`  
   - Connect "Check Status" output here.

9. **Add a Set node named "Success Response"**  
   - Assign an object to `response` with keys:  
     - `success: true`  
     - `result_url: {{$json.output}}`  
     - `prediction_id: {{$json.id}}`  
     - `status: {{$json.status}}`  
     - `message: "Text generated successfully"`  
   - Connect "Is Complete?" true branch to this node.

10. **Add an If node named "Has Failed?"**  
    - Condition: check if `{{$json.status}}` equals `"failed"`  
    - Connect "Is Complete?" false branch to this node.

11. **Add a Set node named "Error Response"**  
    - Assign an object to `response` with keys:  
      - `success: false`  
      - `error: {{$json.error || "Text generation failed"}}`  
      - `prediction_id: {{$json.id}}`  
      - `status: {{$json.status}}`  
      - `message: "Failed to generate text"`  
    - Connect "Has Failed?" true branch here.

12. **Add a Wait node named "Wait 10s"**  
    - Wait for 10 seconds before retrying  
    - Connect "Has Failed?" false branch here.

13. **Connect "Wait 10s" output back to "Check Status"**  
    - This creates the polling retry loop.

14. **Add a Set node named "Display Result"**  
    - Assign `final_result` to the `response` object from either Success or Error node  
    - Connect both "Success Response" and "Error Response" outputs here.

15. **Add Sticky Notes for documentation and contact info** (optional but recommended)  
    - Add a sticky note with workflow overview, usage instructions, parameter reference, and links to resources (as per Sticky Note4 content).  
    - Add another sticky note with contact info and social links (as per Sticky Note9 content).

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | Context or Link                                                                                      |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| CANARY-QWEN-2.5B GENERATOR contact: Yaron@nofluff.online; YouTube tutorials: https://www.youtube.com/@YaronBeen/videos; LinkedIn: https://www.linkedin.com/in/yaronbeen/                                                                                                                                                                                                                                                                                                                                                                  | Support and tutorials for the Canary Qwen 2.5B workflow                                            |
| Model Documentation and API: https://replicate.com/zsxkib/canary-qwen-2.5b; Replicate API Docs: https://replicate.com/docs; n8n Documentation: https://docs.n8n.io                                                                                                                                                                                                                                                                                                                                                                       | Official documentation and API references                                                          |
| This model provides state-of-the-art speech-to-text transcription with low word error rate (~5.63%) and supports optional AI analysis prompts, confidence display, and timestamping features.                                                                                                                                                                                                                                                                                                                                           | Model capabilities and key features overview                                                       |
| Troubleshooting tips: Ensure valid API tokens; verify parameter types and audio URLs; monitor for timeouts or rate limits; validate output format.                                                                                                                                                                                                                                                                                                                                                                                      | Best practices for stable operation                                                                |

---

**Disclaimer:** The text provided is derived exclusively from an automated workflow created with n8n, respecting all applicable content policies and containing only legal, public data.