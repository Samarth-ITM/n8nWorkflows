Generate & Process Audio with Voxtral Small 24B AI using Replicate API

https://n8nworkflows.xyz/workflows/generate---process-audio-with-voxtral-small-24b-ai-using-replicate-api-6799


# Generate & Process Audio with Voxtral Small 24B AI using Replicate API

### 1. Workflow Overview

This workflow automates audio generation and processing using the Voxtral Small 24B AI model via the Replicate API. It is designed primarily for users who want to submit an audio file for transcription, translation, or audio understanding and receive a generated audio output automatically.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Initialization:** Starting point and API token setup.
- **1.2 Parameter Configuration:** Setting audio input parameters and preparing the API request.
- **1.3 Audio Prediction Creation:** Sending the generation request to Replicate API and logging.
- **1.4 Status Polling and Control Flow:** Periodic checking of prediction status with retry handling.
- **1.5 Result Processing:** Handling success and error responses, final output preparation.
- **1.6 Logging and Monitoring:** Capturing request details for debugging and operational insight.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Initialization

**Overview:**  
This block triggers the workflow manually and sets up the Replicate API token required for authentication.

**Nodes Involved:**  
- Manual Trigger  
- Set API Token

**Node Details:**

- **Manual Trigger**  
  - Type: Trigger node  
  - Role: Initiates the workflow manually when clicked.  
  - Configuration: No parameters set.  
  - Input: None (trigger start).  
  - Output: To "Set API Token".  
  - Failures: None expected.  
  - Notes: Entry point for user to start the workflow.

- **Set API Token**  
  - Type: Set node  
  - Role: Stores the Replicate API token for subsequent requests.  
  - Configuration: Sets a string variable `api_token` with placeholder `"YOUR_REPLICATE_API_TOKEN"`.  
  - Input: From Manual Trigger  
  - Output: To "Set Audio Parameters"  
  - Failure Modes: If token is invalid or missing, downstream API calls will fail authentication.  
  - Note: User must replace placeholder with actual API token to authenticate requests.

---

#### 1.2 Parameter Configuration

**Overview:**  
Sets up input parameters required by the Voxtral Small 24B AI model, including the audio file URL, language, and token limits.

**Nodes Involved:**  
- Set Audio Parameters

**Node Details:**

- **Set Audio Parameters**  
  - Type: Set node  
  - Role: Defines the input parameters for the audio generation API call, including required and optional inputs.  
  - Configuration:  
    - Copies `api_token` from previous node.  
    - Sets `audio` parameter to a sample audio file URL: `https://www2.cs.uic.edu/~i101/SoundFiles/BabyElephantWalk60.wav`.  
    - Sets `language` to `"en"` (English).  
    - Sets `max_new_tokens` to `1024` (max tokens to generate).  
  - Input: From "Set API Token"  
  - Output: To "Create Audio Prediction"  
  - Failure Modes: Incorrect parameter types or missing required fields may cause API errors.

---

#### 1.3 Audio Prediction Creation

**Overview:**  
Sends a POST request to the Replicate API to create an audio generation prediction job with the configured parameters.

**Nodes Involved:**  
- Create Audio Prediction  
- Log Request

**Node Details:**

- **Create Audio Prediction**  
  - Type: HTTP Request node  
  - Role: Issues the API call to Replicate to start audio generation.  
  - Configuration:  
    - POST to `https://api.replicate.com/v1/predictions` endpoint.  
    - Body JSON includes model version ID and input parameters (`audio`, `language`, `max_new_tokens`).  
    - Headers include Authorization Bearer token and `Prefer: wait` to wait for immediate processing.  
    - Response format set to JSON with `neverError` true to avoid workflow abortion on API error.  
  - Input: From "Set Audio Parameters"  
  - Output: To "Log Request"  
  - Failure Modes:  
    - Invalid token or API key errors (401).  
    - Network timeouts.  
    - Model version mismatch or deprecated version ID.  
    - API rate limits or quota exceeded.  

- **Log Request**  
  - Type: Code node  
  - Role: Logs request details for monitoring.  
  - Configuration: JavaScript code logs prediction ID, timestamp, and model type to console.  
  - Input: From "Create Audio Prediction"  
  - Output: To "Wait 5s"  
  - Failures: Expression errors if input JSON missing expected fields.

---

#### 1.4 Status Polling and Control Flow

**Overview:**  
Implements a polling mechanism to check the status of the prediction until it is either completed or failed, with wait intervals and retry logic.

**Nodes Involved:**  
- Wait 5s  
- Check Status  
- Is Complete?  
- Has Failed?  
- Wait 10s

**Node Details:**

- **Wait 5s**  
  - Type: Wait node  
  - Role: Pauses workflow for 5 seconds before status check.  
  - Configuration: 5 seconds wait.  
  - Input: From "Log Request"  
  - Output: To "Check Status"  
  - Failures: None expected.

- **Check Status**  
  - Type: HTTP Request node  
  - Role: Retrieves the current status of the audio prediction from Replicate API.  
  - Configuration:  
    - GET from `https://api.replicate.com/v1/predictions/{prediction_id}` where `prediction_id` is dynamically retrieved from "Create Audio Prediction" node.  
    - Includes Authorization header with Bearer token from "Set API Token".  
    - JSON response, never error mode enabled.  
  - Input: From "Wait 5s" and "Wait 10s" (retry loop)  
  - Output: To "Is Complete?"  
  - Failure Modes: Network errors, invalid token, 404 if prediction ID invalid.

- **Is Complete?**  
  - Type: If node  
  - Role: Checks if prediction status is "succeeded".  
  - Configuration: Condition checks if `$json.status == "succeeded"`.  
  - Input: From "Check Status"  
  - Output:  
    - True: To "Success Response"  
    - False: To "Has Failed?"  
  - Failures: If status field missing, expression may fail.

- **Has Failed?**  
  - Type: If node  
  - Role: Checks if prediction status is "failed".  
  - Configuration: Condition checks if `$json.status == "failed"`.  
  - Input: From "Is Complete?" (false branch)  
  - Output:  
    - True: To "Error Response"  
    - False: To "Wait 10s" (retry delay)  
  - Failures: Same as above.

- **Wait 10s**  
  - Type: Wait node  
  - Role: Waits 10 seconds before retrying status check to avoid API spamming.  
  - Configuration: 10 seconds wait.  
  - Input: From "Has Failed?" (false branch)  
  - Output: To "Check Status"  
  - Failures: None expected.

---

#### 1.5 Result Processing

**Overview:**  
Handles final workflow outputs by preparing structured success or error response objects and forwarding them for display.

**Nodes Involved:**  
- Success Response  
- Error Response  
- Display Result

**Node Details:**

- **Success Response**  
  - Type: Set node  
  - Role: Constructs a JSON object indicating success, including generated audio URL and metadata.  
  - Configuration: Sets `response` object with keys: `success: true`, `audio_url`, `prediction_id`, `status`, and a success message.  
  - Input: From "Is Complete?" (true branch)  
  - Output: To "Display Result"  
  - Failures: If expected fields missing in input, response may lack data.

- **Error Response**  
  - Type: Set node  
  - Role: Constructs an error JSON response including error message and prediction status.  
  - Configuration: Sets `response` object with keys: `success: false`, `error`, `prediction_id`, `status`, and failure message.  
  - Input: From "Has Failed?" (true branch)  
  - Output: To "Display Result"  
  - Failures: If error field missing, defaults to generic message.

- **Display Result**  
  - Type: Set node  
  - Role: Prepares the final output variable `final_result` for downstream consumption or user display.  
  - Configuration: Assigns `final_result` from the previous response object.  
  - Input: From both "Success Response" and "Error Response"  
  - Output: Workflow end output  
  - Failures: None expected.

---

#### 1.6 Logging and Monitoring

**Overview:**  
Provides operational transparency and debugging support via console logs and embedded documentation.

**Nodes Involved:**  
- Log Request  
- Sticky Note4  
- Sticky Note9

**Node Details:**

- **Log Request**  
  - See details in 1.3 block.  
  - Role: Key for monitoring requests and troubleshooting.

- **Sticky Note9**  
  - Type: Sticky Note  
  - Role: Provides contact info, support, and external resource links for the workflow author.  
  - Content: Contact email, YouTube and LinkedIn links.

- **Sticky Note4**  
  - Type: Sticky Note  
  - Role: Comprehensive documentation embedded inside n8n.  
  - Content: Model overview, parameter definitions, detailed workflow explanations, best practices, troubleshooting, and links to official docs and tutorials.

---

### 3. Summary Table

| Node Name           | Node Type          | Functional Role                      | Input Node(s)           | Output Node(s)          | Sticky Note                                                                                  |
|---------------------|--------------------|------------------------------------|------------------------|-------------------------|----------------------------------------------------------------------------------------------|
| Manual Trigger      | Trigger            | Starts workflow manually            | None                   | Set API Token            |                                                                                            |
| Set API Token       | Set                | Stores API token for authentication| Manual Trigger          | Set Audio Parameters     |                                                                                            |
| Set Audio Parameters| Set                | Configures input parameters         | Set API Token           | Create Audio Prediction  |                                                                                            |
| Create Audio Prediction | HTTP Request    | Sends audio generation request      | Set Audio Parameters    | Log Request              |                                                                                            |
| Log Request         | Code               | Logs request details for monitoring | Create Audio Prediction | Wait 5s                  |                                                                                            |
| Wait 5s             | Wait               | Pauses 5 seconds before status check| Log Request             | Check Status             |                                                                                            |
| Check Status        | HTTP Request       | Polls prediction status             | Wait 5s, Wait 10s       | Is Complete?             |                                                                                            |
| Is Complete?        | If                 | Checks if prediction succeeded      | Check Status            | Success Response, Has Failed? |                                                                                        |
| Has Failed?         | If                 | Checks if prediction failed         | Is Complete?            | Error Response, Wait 10s |                                                                                            |
| Wait 10s            | Wait               | Waits 10 seconds before retrying    | Has Failed?             | Check Status             |                                                                                            |
| Success Response    | Set                | Creates success response object     | Is Complete?            | Display Result           |                                                                                            |
| Error Response      | Set                | Creates error response object       | Has Failed?             | Display Result           |                                                                                            |
| Display Result      | Set                | Outputs final result                 | Success Response, Error Response | None              |                                                                                            |
| Sticky Note9        | Sticky Note        | Support and contact info            | None                   | None                    | Contact: Yaron@nofluff.online; YouTube: https://www.youtube.com/@YaronBeen/videos; LinkedIn: https://www.linkedin.com/in/yaronbeen/ |
| Sticky Note4        | Sticky Note        | Full workflow and model documentation| None                   | None                    | Extensive model overview and instructions embedded in workflow                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Manual Trigger node**  
   - Type: Manual Trigger  
   - Purpose: To start the workflow execution manually.

2. **Add Set node "Set API Token"**  
   - Connect from Manual Trigger.  
   - Set String variable `api_token` with value `"YOUR_REPLICATE_API_TOKEN"`.  
   - Replace the placeholder with your actual Replicate API key before running.

3. **Add Set node "Set Audio Parameters"**  
   - Connect from "Set API Token".  
   - Assign variables:  
     - `api_token`: Copy from previous node (`={{ $('Set API Token').item.json.api_token }}`)  
     - `audio`: URL of the audio file to process (default example: `https://www2.cs.uic.edu/~i101/SoundFiles/BabyElephantWalk60.wav`)  
     - `language`: Language code, default `"en"`  
     - `max_new_tokens`: Number, default `1024`

4. **Add HTTP Request node "Create Audio Prediction"**  
   - Connect from "Set Audio Parameters".  
   - Method: POST  
   - URL: `https://api.replicate.com/v1/predictions`  
   - Headers:  
     - `Authorization`: `Bearer {{$json.api_token}}`  
     - `Prefer`: `wait`  
   - Body (JSON):  
     ```json
     {
       "version": "notdaniel/voxtral-small-24b-2507:31b40d9229c3df8c13863fcdd8cc95e6c1c6eb2b6c653936934234eadcc23d41",
       "input": {
         "audio": "{{$json.audio}}",
         "language": "{{$json.language}}",
         "max_new_tokens": {{$json.max_new_tokens}}
       }
     }
     ```  
   - Set "Response Format" to JSON, enable "Never Error" to prevent workflow stop on API error.

5. **Add Code node "Log Request"**  
   - Connect from "Create Audio Prediction".  
   - JavaScript code to log:  
     ```js
     const data = $input.all()[0].json;
     console.log('notdaniel/voxtral-small-24b-2507 Request:', {
       timestamp: new Date().toISOString(),
       prediction_id: data.id,
       model_type: 'audio'
     });
     return $input.all();
     ```

6. **Add Wait node "Wait 5s"**  
   - Connect from "Log Request".  
   - Configure to wait 5 seconds.

7. **Add HTTP Request node "Check Status"**  
   - Connect from "Wait 5s" and also from "Wait 10s" (later).  
   - Method: GET  
   - URL: `https://api.replicate.com/v1/predictions/{{$('Create Audio Prediction').item.json.id}}`  
   - Headers: Authorization Bearer token from "Set API Token".  
   - Response Format: JSON, never error enabled.

8. **Add If node "Is Complete?"**  
   - Connect from "Check Status".  
   - Condition: `$json.status == "succeeded"`  
   - True output: Connect to "Success Response"  
   - False output: Connect to "Has Failed?"

9. **Add If node "Has Failed?"**  
   - Connect from "Is Complete?" (False output).  
   - Condition: `$json.status == "failed"`  
   - True output: Connect to "Error Response"  
   - False output: Connect to "Wait 10s"

10. **Add Wait node "Wait 10s"**  
    - Connect from "Has Failed?" (False output).  
    - Wait 10 seconds before retry.  
    - Connect output back to "Check Status" for retry loop.

11. **Add Set node "Success Response"**  
    - Connect from "Is Complete?" (True output).  
    - Set variable `response` as object:  
      ```json
      {
        "success": true,
        "audio_url": $json.output,
        "prediction_id": $json.id,
        "status": $json.status,
        "message": "Audio generated successfully"
      }
      ```

12. **Add Set node "Error Response"**  
    - Connect from "Has Failed?" (True output).  
    - Set variable `response` as object:  
      ```json
      {
        "success": false,
        "error": $json.error || "Audio generation failed",
        "prediction_id": $json.id,
        "status": $json.status,
        "message": "Failed to generate audio"
      }
      ```

13. **Add Set node "Display Result"**  
    - Connect from both "Success Response" and "Error Response".  
    - Assign `final_result` variable equal to the `response` object from upstream.

14. **Add Sticky Notes** (optional but recommended for documentation)  
    - Add Sticky Note with contact info and support links.  
    - Add Sticky Note with detailed workflow and parameter documentation for ease of maintenance.

---

### 5. General Notes & Resources

| Note Content                                                                                                                       | Context or Link                                                                                                   |
|-----------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| VOXTRAL-SMALL-24B-2507 GENERATOR - For questions/support contact Yaron@nofluff.online. Videos: https://www.youtube.com/@YaronBeen/videos LinkedIn: https://www.linkedin.com/in/yaronbeen/ | Contact and support information embedded as Sticky Note in workflow.                                             |
| Model Documentation: https://replicate.com/notdaniel/voxtral-small-24b-2507                                                        | Official model documentation for parameter details and usage.                                                    |
| Replicate API Docs: https://replicate.com/docs                                                                                     | API usage, authentication, error codes, and best practices.                                                      |
| n8n Documentation: https://docs.n8n.io                                                                                            | Comprehensive reference for n8n node usage, expressions, and workflow creation.                                   |
| Recommendations: Replace API token placeholder with a valid token before running. Monitor API quota and usage to avoid failures. | Best practices for operational reliability and security.                                                         |
| Troubleshooting tips include checking token validity, parameter correctness, and monitoring workflow logs for errors or timeouts.| Embedded in Sticky Note documentation for user guidance.                                                         |

---

This structured document provides a comprehensive understanding of the workflow, enabling users and AI agents to analyze, reproduce, and troubleshoot it effectively.