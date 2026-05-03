Process Audio Data with Aihilums Sehatsanjha AI via Replicate API

https://n8nworkflows.xyz/workflows/process-audio-data-with-aihilums-sehatsanjha-ai-via-replicate-api-6801


# Process Audio Data with Aihilums Sehatsanjha AI via Replicate API

---

### 1. Workflow Overview

This n8n workflow automates the process of generating audio-related outputs using the AI model "sehatsanjha" by aihilums via the Replicate API. It is designed for scenarios where users want to submit audio files and related session parameters to the AI model and receive processed results asynchronously, managing API interaction, retries, and error handling.

The workflow is structured into these logical blocks:

- **1.1 Input Initialization:** Receives manual trigger and sets the API token and all required and optional input parameters for the model.
- **1.2 Prediction Request:** Sends a request to the Replicate API to create a new prediction job.
- **1.3 Monitoring & Status Checking:** Waits and polls the prediction status until completion or failure, with built-in retry delays.
- **1.4 Response Handling:** Processes the prediction result or error, formatting structured success or error responses.
- **1.5 Logging & Output:** Logs the request details for monitoring and outputs the final structured result.
- **1.6 Documentation & User Guidance:** Sticky notes provide essential user instructions, parameter references, and support contacts.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Initialization

**Overview:**  
Sets up the workflow execution by triggering manually, defining the Replicate API token for authentication, and configuring all input parameters required or optional for invoking the AI model.

**Nodes Involved:**  
- Manual Trigger  
- Set API Token  
- Set Other Parameters  

**Node Details:**

- **Manual Trigger**  
  - Type: Trigger node  
  - Role: Starts the workflow manually on user command.  
  - Configuration: Default manual trigger, no parameters.  
  - Inputs: None  
  - Outputs: Connects to "Set API Token" node  
  - Edge cases: User must manually start; no auto-trigger or webhook configured.

- **Set API Token**  
  - Type: Set node  
  - Role: Assigns Replicate API token string for authentication.  
  - Configuration: Hardcoded string value `"YOUR_REPLICATE_API_TOKEN"` (to be replaced by user).  
  - Key expression: None (static value)  
  - Inputs: From Manual Trigger  
  - Outputs: Connects to "Set Other Parameters"  
  - Edge cases: Failure if token is invalid or missing; must be replaced with a valid token before running.

- **Set Other Parameters**  
  - Type: Set node  
  - Role: Defines all model input parameters including API token (passed from previous node), plus optional parameters like `cookie`, `user_id`, `audio_file`, `user_state`, `end_session`, and `new_session`.  
  - Configuration:  
    - `api_token`: references the output from "Set API Token"  
    - `cookie`: empty string by default  
    - `user_id`: empty string  
    - `audio_file`: default URL set to `https://picsum.photos/512/512` (placeholder image URL, likely for test)  
    - `user_state`: empty string  
    - `end_session`: false  
    - `new_session`: false  
  - Expressions: Dynamic reference to API token from previous node  
  - Inputs: From "Set API Token"  
  - Outputs: Connects to "Create Other Prediction" node  
  - Edge cases: Parameters must be validated for correctness before sending to API; missing or malformed URLs or invalid booleans could cause API errors.

---

#### 2.2 Prediction Request

**Overview:**  
Creates a prediction request on Replicate API with the configured input parameters, initiating asynchronous processing of the audio data.

**Nodes Involved:**  
- Create Other Prediction  

**Node Details:**

- **Create Other Prediction**  
  - Type: HTTP Request node  
  - Role: Sends POST request to Replicate API endpoint `/v1/predictions` to start model prediction.  
  - Configuration:  
    - Method: POST  
    - URL: `https://api.replicate.com/v1/predictions`  
    - Headers: Authorization `Bearer <api_token>`, `Prefer: wait` to wait for prediction completion if possible  
    - Body: JSON containing `version` (fixed model version ID), and `input` object with `cookie`, `user_id`, `audio_file`, `user_state`, `end_session`, and `new_session` from workflow data  
    - Response Handling: JSON, never error (to handle errors gracefully)  
  - Expressions: All inputs are dynamically mapped from previous node's JSON data  
  - Inputs: From "Set Other Parameters"  
  - Outputs: Connects to "Log Request" node  
  - Edge cases:  
    - Possible authentication errors if token invalid  
    - API rate limits or network failures  
    - Validation errors if inputs are malformed  
  - Version Requirements: n8n HTTP Request node v4.1 or newer recommended for JSON body and header dynamic expressions.

---

#### 2.3 Monitoring & Status Checking

**Overview:**  
Polls the Replicate API prediction status using a loop with wait nodes to delay between requests, handling both success and failure states with retries.

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
  - Role: Logs prediction ID, timestamp, and model type for monitoring/debugging.  
  - Configuration: Custom JS code to log to console: timestamp, prediction ID, model type `"other"`.  
  - Inputs: From "Create Other Prediction"  
  - Outputs: Connects to "Wait 5s"  
  - Edge cases: Console logging may fail if environment restricted; no direct workflow impact.

- **Wait 5s**  
  - Type: Wait node  
  - Role: Delays workflow execution by 5 seconds before next status check.  
  - Configuration: 5 seconds delay  
  - Inputs: From "Log Request"  
  - Outputs: Connects to "Check Status"  
  - Edge cases: None expected; delays ensure API is ready for status polling.

- **Check Status**  
  - Type: HTTP Request node  
  - Role: Requests current prediction status from Replicate API `/v1/predictions/{id}`.  
  - Configuration:  
    - GET method  
    - URL dynamically built with prediction ID from "Create Other Prediction" node  
    - Auth header with Bearer token from "Set API Token" node  
    - Response JSON, never error to handle gracefully  
  - Inputs: From "Wait 5s" and "Wait 10s" (looping)  
  - Outputs: Connects to "Is Complete?"  
  - Edge cases: Network failure, invalid prediction ID, API limits.

- **Is Complete?**  
  - Type: If node  
  - Role: Checks if prediction status equals `"succeeded"` indicating completion.  
  - Configuration: Condition: `$json.status == "succeeded"`  
  - Inputs: From "Check Status"  
  - Outputs:  
    - True: Connects to "Success Response"  
    - False: Connects to "Has Failed?"  
  - Edge cases: Status could be other values (e.g., `"processing"`, `"failed"`).

- **Has Failed?**  
  - Type: If node  
  - Role: Determines if prediction status equals `"failed"`.  
  - Configuration: Condition: `$json.status == "failed"`  
  - Inputs: From "Is Complete?" (false branch)  
  - Outputs:  
    - True: Connects to "Error Response"  
    - False: Connects to "Wait 10s" to retry status check  
  - Edge cases: Possible infinite loop if API hangs in unknown status; workflow relies on proper API response.

- **Wait 10s**  
  - Type: Wait node  
  - Role: Delays 10 seconds before retrying status check.  
  - Configuration: 10 seconds delay  
  - Inputs: From "Has Failed?" (false branch)  
  - Outputs: Connects back to "Check Status" (loop)  
  - Edge cases: Delays prevent spamming API; long waits if prediction slow.

---

#### 2.4 Response Handling

**Overview:**  
Creates structured JSON responses for both success and failure outcomes, preparing data for output or further processing.

**Nodes Involved:**  
- Success Response  
- Error Response  
- Display Result  

**Node Details:**

- **Success Response**  
  - Type: Set node  
  - Role: Constructs success response with prediction output URL, ID, status, and message.  
  - Configuration: Sets `response` object with  
    - `success: true`  
    - `result_url`: from prediction output  
    - `prediction_id`, `status`, `message` (static success message)  
  - Inputs: From "Is Complete?" (true branch)  
  - Outputs: Connects to "Display Result"  
  - Edge cases: Output URL must be valid; missing output would break downstream usage.

- **Error Response**  
  - Type: Set node  
  - Role: Constructs error response object including error message, prediction ID, and status.  
  - Configuration: Sets `response` object with  
    - `success: false`  
    - `error`: from JSON error or fallback string  
    - `prediction_id`, `status`, `message` (static failure message)  
  - Inputs: From "Has Failed?" (true branch)  
  - Outputs: Connects to "Display Result"  
  - Edge cases: Error message may be undefined; fallback string ensures clarity.

- **Display Result**  
  - Type: Set node  
  - Role: Exposes final structured response under `final_result` key for downstream consumption or output.  
  - Configuration: Sets `final_result` to the `response` object from previous node.  
  - Inputs: From both Success Response and Error Response  
  - Outputs: Terminal output of workflow  
  - Edge cases: None expected.

---

#### 2.5 Documentation & User Guidance

**Overview:**  
Provides embedded documentation, guidance, and contact information via sticky notes within the workflow canvas.

**Nodes Involved:**  
- Sticky Note9  
- Sticky Note4  

**Node Details:**

- **Sticky Note9**  
  - Type: Sticky Note  
  - Role: Displays concise contact and resource info for support and tutorials.  
  - Content Highlights: Contact email (Yaron@nofluff.online), YouTube and LinkedIn links for additional help.  
  - Position: Top-left of workflow  
  - Edge cases: None.

- **Sticky Note4**  
  - Type: Sticky Note  
  - Role: Detailed documentation on model overview, parameter reference, workflow explanation, benefits, quick start instructions, troubleshooting, and links.  
  - Content Highlights:  
    - Model info: Owner, type, API endpoint  
    - Parameter descriptions with defaults  
    - Stepwise explanation of workflow components  
    - Troubleshooting tips  
    - Useful external links:  
      - Model Docs: https://replicate.com/aihilums/sehatsanjha  
      - Replicate API Docs: https://replicate.com/docs  
      - n8n Docs: https://docs.n8n.io  
  - Edge cases: None.

---

### 3. Summary Table

| Node Name             | Node Type          | Functional Role                     | Input Node(s)                    | Output Node(s)                  | Sticky Note                                                                                          |
|-----------------------|--------------------|-----------------------------------|---------------------------------|--------------------------------|----------------------------------------------------------------------------------------------------|
| Manual Trigger        | Manual Trigger     | Starts workflow execution          | -                               | Set API Token                  | See Sticky Note4 for detailed workflow documentation and Sticky Note9 for contact and resources.    |
| Set API Token         | Set                | Defines Replicate API token        | Manual Trigger                  | Set Other Parameters           | See Sticky Note4 and Sticky Note9                                                                    |
| Set Other Parameters  | Set                | Configures model input parameters  | Set API Token                  | Create Other Prediction        | See Sticky Note4 and Sticky Note9                                                                    |
| Create Other Prediction | HTTP Request       | Sends prediction request to API    | Set Other Parameters            | Log Request                   | See Sticky Note4 and Sticky Note9                                                                    |
| Log Request           | Code               | Logs prediction details to console | Create Other Prediction         | Wait 5s                       | See Sticky Note4 and Sticky Note9                                                                    |
| Wait 5s               | Wait               | Waits 5 seconds before status check | Log Request                    | Check Status                  | See Sticky Note4 and Sticky Note9                                                                    |
| Check Status          | HTTP Request       | Polls prediction status from API   | Wait 5s, Wait 10s              | Is Complete?                  | See Sticky Note4 and Sticky Note9                                                                    |
| Is Complete?          | If                 | Checks if prediction succeeded     | Check Status                   | Success Response, Has Failed? | See Sticky Note4 and Sticky Note9                                                                    |
| Has Failed?           | If                 | Checks if prediction failed        | Is Complete?                   | Error Response, Wait 10s      | See Sticky Note4 and Sticky Note9                                                                    |
| Wait 10s              | Wait               | Waits 10 seconds before retry       | Has Failed?                    | Check Status                  | See Sticky Note4 and Sticky Note9                                                                    |
| Success Response      | Set                | Formats success response JSON      | Is Complete?                   | Display Result                | See Sticky Note4 and Sticky Note9                                                                    |
| Error Response        | Set                | Formats error response JSON        | Has Failed?                    | Display Result                | See Sticky Note4 and Sticky Note9                                                                    |
| Display Result        | Set                | Outputs final structured response  | Success Response, Error Response | -                            | See Sticky Note4 and Sticky Note9                                                                    |
| Sticky Note9          | Sticky Note        | Support contact and resource info  | -                             | -                            | Contact: Yaron@nofluff.online; YouTube: https://www.youtube.com/@YaronBeen/videos; LinkedIn: https://www.linkedin.com/in/yaronbeen/ |
| Sticky Note4          | Sticky Note        | Comprehensive workflow documentation | -                           | -                            | Extensive guide with parameter details, instructions, troubleshooting, and links.                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Add Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: Start workflow manually  
   - No parameters needed.

2. **Add Set Node "Set API Token"**  
   - Type: Set  
   - Assign string field `api_token` with your Replicate API token (replace `"YOUR_REPLICATE_API_TOKEN"`).  
   - Connect output of Manual Trigger to this node.

3. **Add Set Node "Set Other Parameters"**  
   - Type: Set  
   - Assign fields:  
     - `api_token`: Set expression to get from previous node (`{{$node["Set API Token"].json["api_token"]}}`)  
     - `cookie`: Empty string `""`  
     - `user_id`: Empty string `""`  
     - `audio_file`: Default `https://picsum.photos/512/512` (replace with actual audio file URL)  
     - `user_state`: Empty string `""`  
     - `end_session`: Boolean `false`  
     - `new_session`: Boolean `false`  
   - Connect output of "Set API Token" to this node.

4. **Add HTTP Request Node "Create Other Prediction"**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.replicate.com/v1/predictions`  
   - Headers:  
     - `Authorization`: `Bearer {{$json["api_token"]}}`  
     - `Prefer`: `wait`  
   - Body Type: JSON  
   - JSON Body:  
     ```json
     {
       "version": "aihilums/sehatsanjha:8d601ed5dfe3c91d7dacdc17a127497519acfdb4c4eefbec3db17b6d87734a69",
       "input": {
         "cookie": "{{$json.cookie}}",
         "user_id": "{{$json.user_id}}",
         "audio_file": "{{$json.audio_file}}",
         "user_state": "{{$json.user_state}}",
         "end_session": {{$json.end_session}},
         "new_session": {{$json.new_session}}
       }
     }
     ```
   - Connect output of "Set Other Parameters" to this node.

5. **Add Code Node "Log Request"**  
   - Type: Code (JavaScript)  
   - Code:  
     ```javascript
     const data = $input.all()[0].json;
     console.log('aihilums/sehatsanjha Request:', {
       timestamp: new Date().toISOString(),
       prediction_id: data.id,
       model_type: 'other'
     });
     return $input.all();
     ```  
   - Connect output of "Create Other Prediction" to this node.

6. **Add Wait Node "Wait 5s"**  
   - Type: Wait  
   - Parameters: 5 seconds  
   - Connect output of "Log Request" to this node.

7. **Add HTTP Request Node "Check Status"**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://api.replicate.com/v1/predictions/{{$node["Create Other Prediction"].json["id"]}}`  
   - Headers:  
     - `Authorization`: `Bearer {{$node["Set API Token"].json["api_token"]}}`  
   - Connect output of "Wait 5s" to this node. (Later, will also connect Wait 10s output here to loop.)

8. **Add If Node "Is Complete?"**  
   - Type: If  
   - Condition: Check if `$json.status === "succeeded"`  
   - Connect output of "Check Status" to this node.

9. **Add Set Node "Success Response"**  
   - Type: Set  
   - Assign a field `response` as an object with:  
     - `success: true`  
     - `result_url`: `$json.output`  
     - `prediction_id`: `$json.id`  
     - `status`: `$json.status`  
     - `message`: `"Other generated successfully"`  
   - Connect "Is Complete?" true branch to this node.

10. **Add Set Node "Display Result"**  
    - Type: Set  
    - Assign `final_result` field with `$json.response`  
    - Connect output of "Success Response" to this node.

11. **Add If Node "Has Failed?"**  
    - Type: If  
    - Condition: `$json.status === "failed"`  
    - Connect "Is Complete?" false branch to this node.

12. **Add Set Node "Error Response"**  
    - Type: Set  
    - Assign a field `response` as object with:  
      - `success: false`  
      - `error`: `$json.error` or fallback "Other generation failed"  
      - `prediction_id`: `$json.id`  
      - `status`: `$json.status`  
      - `message`: `"Failed to generate other"`  
    - Connect "Has Failed?" true branch to this node.

13. **Connect output of "Error Response" to "Display Result"**  
    - So both success and error responses go to final output.

14. **Add Wait Node "Wait 10s"**  
    - Type: Wait  
    - Parameters: 10 seconds  
    - Connect "Has Failed?" false branch to this node.

15. **Loop back Wait 10s output to "Check Status"**  
    - Creates polling retry loop until success or failure.

16. **Connect "Display Result" output as workflow output or further process.**

17. **Add Sticky Notes**  
    - One with contact/support info:  
      - Content:  
        ```
        =======================================
                SEHATSANJHA GENERATOR
        =======================================
        For any questions or support, please contact:
            Yaron@nofluff.online
        
        Explore more tips and tutorials here:
           - YouTube: https://www.youtube.com/@YaronBeen/videos
           - LinkedIn: https://www.linkedin.com/in/yaronbeen/
        =======================================
        ```
    - One with detailed workflow documentation and parameter references as described.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                | Context or Link                                                                                       |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| Contact for support: Yaron@nofluff.online                                                                                                                                                   | Support email                                                                                         |
| YouTube channel with tips and tutorials: https://www.youtube.com/@YaronBeen/videos                                                                                                           | Video resource                                                                                       |
| LinkedIn profile: https://www.linkedin.com/in/yaronbeen/                                                                                                                                     | Professional profile                                                                                 |
| Model Documentation: https://replicate.com/aihilums/sehatsanjha                                                                                                                             | Official model docs                                                                                  |
| Replicate API Documentation: https://replicate.com/docs                                                                                                                                      | API usage details                                                                                   |
| n8n Official Documentation: https://docs.n8n.io                                                                                                                                              | Workflow automation platform documentation                                                         |
| Troubleshooting tips include verifying API token, parameter types, generation timeout, and output format validation                                                                        | General best practices                                                                                |
| Replace placeholder audio file URL with actual audio file URL to process real data                                                                                                          | Important for production use                                                                         |
| Ensure API token is kept secure and not committed to source control                                                                                                                        | Security best practice                                                                                |

---

**Disclaimer:**  
The provided text is derived exclusively from an automated workflow created using n8n, a tool for integration and automation. This processing strictly complies with the applicable content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.

---