Create Professional Animated Videos with Bytedance Omni-Human AI via Replicate

https://n8nworkflows.xyz/workflows/create-professional-animated-videos-with-bytedance-omni-human-ai-via-replicate-6788


# Create Professional Animated Videos with Bytedance Omni-Human AI via Replicate

### 1. Workflow Overview

This workflow automates the creation of professional-quality animated videos by leveraging the Bytedance Omni-Human AI model via the Replicate API. It is designed for users who want to transform input audio and images into animated videos without manual intervention, providing automated status monitoring and robust error handling.

The workflow is logically divided into the following functional blocks:

- **1.1 Input Initialization**: Starts the workflow manually and sets required API authentication and video input parameters.
- **1.2 Video Generation Request**: Sends a video generation prediction request to the Replicate API with the specified inputs.
- **1.3 Status Monitoring Loop**: Periodically checks the status of the video generation prediction, with wait intervals and retry logic.
- **1.4 Outcome Handling**: Processes successful video generation responses or handles failure cases with structured feedback.
- **1.5 Logging and Result Display**: Logs request details for monitoring and sets the final response output.
- **1.6 Documentation and Support Notes**: Contains sticky notes with detailed workflow documentation, usage instructions, and support contacts.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Initialization

**Overview:**  
This block initializes the workflow by manually triggering execution and setting the Replicate API token and video input parameters (audio and image URLs).

**Nodes Involved:**  
- Manual Trigger  
- Set API Token  
- Set Video Parameters

**Node Details:**

- **Manual Trigger**  
  - Type: Manual trigger node  
  - Role: Starts the workflow manually  
  - Configuration: No parameters; user clicks to start  
  - Inputs: None  
  - Outputs: Connects to "Set API Token"  
  - Edge cases: None expected

- **Set API Token**  
  - Type: Set node  
  - Role: Assigns the Replicate API token for authentication  
  - Configuration: Static assignment of string variable `api_token` with placeholder `"YOUR_REPLICATE_API_TOKEN"` (to be replaced by the user)  
  - Inputs: From Manual Trigger  
  - Outputs: Connects to "Set Video Parameters"  
  - Edge cases: Missing or invalid API token causes authentication failures downstream

- **Set Video Parameters**  
  - Type: Set node  
  - Role: Defines input parameters for the video generation (audio and image URLs), also passes through the API token  
  - Configuration:  
    - `api_token`: references the token from "Set API Token" node  
    - `audio`: preset to `"https://www2.cs.uic.edu/~i101/SoundFiles/BabyElephantWalk60.wav"` (example audio)  
    - `image`: preset to `"https://picsum.photos/512/512"` (example image)  
  - Inputs: From "Set API Token"  
  - Outputs: Connects to "Create Video Prediction"  
  - Edge cases: Invalid or unreachable URLs may cause generation failures

#### 2.2 Video Generation Request

**Overview:**  
Sends a POST request to the Replicate API to start video generation, submitting the audio and image inputs for the Bytedance Omni-Human model.

**Nodes Involved:**  
- Create Video Prediction  
- Log Request

**Node Details:**

- **Create Video Prediction**  
  - Type: HTTP Request node  
  - Role: Issues API call to Replicate's prediction endpoint to start video generation  
  - Configuration:  
    - URL: `https://api.replicate.com/v1/predictions`  
    - Method: POST  
    - Headers: Authorization header with Bearer token from `api_token` variable; `Prefer: wait` to wait for synchronous response if possible  
    - Body: JSON including model version ID and inputs (`audio`, `image`) interpolated from previous node  
    - Response: JSON, with `neverError` set to true to avoid node errors on HTTP failures  
  - Inputs: From "Set Video Parameters"  
  - Outputs: Connects to "Log Request"  
  - Edge cases: API errors, invalid token, network issues; response may be delayed or incomplete

- **Log Request**  
  - Type: Code node  
  - Role: Logs prediction request details to console for monitoring  
  - Configuration: JavaScript code logs timestamp, prediction ID, and model type  
  - Inputs: From "Create Video Prediction"  
  - Outputs: Connects to "Wait 5s"  
  - Edge cases: Logging failures unlikely but should not block workflow

#### 2.3 Status Monitoring Loop

**Overview:**  
Implements polling to check the video generation status until completion or failure, including wait intervals and retry logic.

**Nodes Involved:**  
- Wait 5s  
- Check Status  
- Is Complete?  
- Has Failed?  
- Wait 10s

**Node Details:**

- **Wait 5s**  
  - Type: Wait node  
  - Role: Pauses execution for 5 seconds before checking status  
  - Configuration: Wait 5 seconds  
  - Inputs: From "Log Request"  
  - Outputs: Connects to "Check Status"  
  - Edge cases: Wait durations too short/long may affect responsiveness

- **Check Status**  
  - Type: HTTP Request node  
  - Role: Requests current prediction status from Replicate API  
  - Configuration:  
    - URL: Dynamic, includes prediction ID from "Create Video Prediction" node  
    - Method: GET  
    - Headers: Authorization with API token  
    - Response: JSON with `neverError` true  
  - Inputs: From "Wait 5s" and "Wait 10s" (retry)  
  - Outputs: Connects to "Is Complete?"  
  - Edge cases: API throttling, token expiration, network errors

- **Is Complete?**  
  - Type: If node  
  - Role: Tests if prediction status equals `"succeeded"`  
  - Configuration: Condition: `status == "succeeded"` (case-sensitive, loose type validation)  
  - Inputs: From "Check Status"  
  - Outputs:  
    - True: Connects to "Success Response"  
    - False: Connects to "Has Failed?"  
  - Edge cases: Unexpected status values; false negatives

- **Has Failed?**  
  - Type: If node  
  - Role: Tests if prediction status equals `"failed"`  
  - Configuration: Condition: `status == "failed"`  
  - Inputs: From "Is Complete?" (false branch)  
  - Outputs:  
    - True: Connects to "Error Response"  
    - False: Connects to "Wait 10s" (retry)  
  - Edge cases: Infinite loops if status stuck in non-terminal states

- **Wait 10s**  
  - Type: Wait node  
  - Role: Waits 10 seconds before retrying status check  
  - Configuration: 10 seconds pause  
  - Inputs: From "Has Failed?" (false branch)  
  - Outputs: Connects back to "Check Status"  
  - Edge cases: Retry intervals affect total wait time before failure

#### 2.4 Outcome Handling

**Overview:**  
Handles both successful and failed video generation outcomes by preparing structured responses for downstream use.

**Nodes Involved:**  
- Success Response  
- Error Response  
- Display Result

**Node Details:**

- **Success Response**  
  - Type: Set node  
  - Role: Creates a JSON object indicating success and containing video URL and metadata  
  - Configuration: Sets variable `response` to object with keys:  
    - `success: true`  
    - `video_url`: from prediction output URL  
    - `prediction_id`: prediction ID  
    - `status`: current status  
    - `message`: "Video generated successfully"  
  - Inputs: From "Is Complete?" (true branch)  
  - Outputs: Connects to "Display Result"  
  - Edge cases: Missing output URL if API response incomplete

- **Error Response**  
  - Type: Set node  
  - Role: Creates a JSON object indicating failure with error details  
  - Configuration: Sets variable `response` to object with keys:  
    - `success: false`  
    - `error`: from JSON error message or fallback string  
    - `prediction_id`  
    - `status`  
    - `message`: "Failed to generate video"  
  - Inputs: From "Has Failed?" (true branch)  
  - Outputs: Connects to "Display Result"  
  - Edge cases: Missing error details

- **Display Result**  
  - Type: Set node  
  - Role: Assigns the final response to variable `final_result` for output or further processing  
  - Configuration: Copies `response` variable to `final_result`  
  - Inputs: From "Success Response" and "Error Response"  
  - Outputs: None (end of workflow)  
  - Edge cases: None

#### 2.5 Logging and Result Display

**Overview:**  
This block logs requests for monitoring and sets the final output response that can be consumed by other systems or displayed.

**Nodes Involved:**  
- Log Request (also part of 2.2)  
- Display Result (also part of 2.4)

**Node Details:**  
Already described in previous blocks.

#### 2.6 Documentation and Support Notes

**Overview:**  
Provides embedded documentation, support contact information, and usage instructions inside sticky notes within the workflow.

**Nodes Involved:**  
- Sticky Note9  
- Sticky Note4

**Node Details:**

- Both sticky notes contain extensive textual content including:  
  - Contact info for support (Yaron@nofluff.online)  
  - Links to video tutorials and LinkedIn  
  - Detailed model overview, parameter explanation, and usage instructions  
  - Troubleshooting tips and best practices  
  - External documentation links for Replicate and n8n

- These notes do not affect workflow execution but serve as valuable references.

---

### 3. Summary Table

| Node Name             | Node Type           | Functional Role                      | Input Node(s)               | Output Node(s)                | Sticky Note                                                                                                      |
|-----------------------|---------------------|------------------------------------|-----------------------------|------------------------------|------------------------------------------------------------------------------------------------------------------|
| Manual Trigger        | Manual Trigger      | Starts workflow                    | None                        | Set API Token                | See Sticky Note4 for detailed workflow overview and instructions                                                |
| Set API Token         | Set                 | Sets Replicate API token           | Manual Trigger              | Set Video Parameters         | See Sticky Note4                                                                                                |
| Set Video Parameters  | Set                 | Sets video input parameters        | Set API Token               | Create Video Prediction      | See Sticky Note4                                                                                                |
| Create Video Prediction | HTTP Request       | Sends video generation request     | Set Video Parameters        | Log Request                 | See Sticky Note4                                                                                                |
| Log Request           | Code                | Logs request details               | Create Video Prediction     | Wait 5s                     | See Sticky Note4                                                                                                |
| Wait 5s               | Wait                | Waits 5 seconds before status check| Log Request                | Check Status                | See Sticky Note4                                                                                                |
| Check Status          | HTTP Request        | Checks prediction status           | Wait 5s, Wait 10s           | Is Complete?                | See Sticky Note4                                                                                                |
| Is Complete?          | If                  | Checks if prediction succeeded     | Check Status                | Success Response, Has Failed? | See Sticky Note4                                                                                                |
| Has Failed?           | If                  | Checks if prediction failed        | Is Complete?                | Error Response, Wait 10s    | See Sticky Note4                                                                                                |
| Wait 10s              | Wait                | Waits 10 seconds before retry      | Has Failed?                 | Check Status                | See Sticky Note4                                                                                                |
| Success Response      | Set                 | Creates success response object    | Is Complete?                | Display Result              | See Sticky Note4                                                                                                |
| Error Response        | Set                 | Creates error response object      | Has Failed?                 | Display Result              | See Sticky Note4                                                                                                |
| Display Result        | Set                 | Sets final output result           | Success Response, Error Response | None                    | See Sticky Note4                                                                                                |
| Sticky Note9          | Sticky Note         | Support and contact info           | None                       | None                       | Support contact: Yaron@nofluff.online; YouTube & LinkedIn links                                                |
| Sticky Note4          | Sticky Note         | Full workflow documentation       | None                       | None                       | Detailed workflow documentation, usage, troubleshooting, and external links                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create “Manual Trigger” Node**  
   - Node Type: Manual Trigger  
   - Purpose: Manually start the workflow  
   - No parameters needed  
   - Position: Leftmost start node

2. **Create “Set API Token” Node**  
   - Node Type: Set  
   - Purpose: Store API token for Replicate authentication  
   - Parameters:  
     - Variable: `api_token` (string)  
     - Value: Replace `"YOUR_REPLICATE_API_TOKEN"` with your actual Replicate API token  
   - Connect “Manual Trigger” output to this node’s input

3. **Create “Set Video Parameters” Node**  
   - Node Type: Set  
   - Purpose: Define video generation inputs and propagate API token  
   - Parameters:  
     - `api_token`: Expression referencing `api_token` from “Set API Token” node (`={{ $('Set API Token').item.json.api_token }}`)  
     - `audio`: URL string of input audio (default example: `https://www2.cs.uic.edu/~i101/SoundFiles/BabyElephantWalk60.wav`)  
     - `image`: URL string of input image (default example: `https://picsum.photos/512/512`)  
   - Connect from “Set API Token”

4. **Create “Create Video Prediction” Node**  
   - Node Type: HTTP Request  
   - Purpose: Start video generation prediction via Replicate API  
   - Configuration:  
     - URL: `https://api.replicate.com/v1/predictions`  
     - Method: POST  
     - Authentication: None here; use header manually  
     - Headers:  
       - `Authorization`: `Bearer {{ $json.api_token }}` (taken from input JSON)  
       - `Prefer`: `wait`  
     - Body Format: JSON  
     - Body Content:  
       ```json
       {
         "version": "bytedance/omni-human:7ec44f5140c7338b3496cbf99ee8ea391a4bc18ff5d1677a146dfc936a91f65b",
         "input": {
           "audio": "{{ $json.audio }}",
           "image": "{{ $json.image }}"
         }
       }
       ```  
     - Response Format: JSON, never throw error on failure  
   - Connect from “Set Video Parameters”

5. **Create “Log Request” Node**  
   - Node Type: Code  
   - Purpose: Log prediction request details to console  
   - Parameters: JavaScript code  
     ```javascript
     const data = $input.all()[0].json;
     console.log('bytedance/omni-human Request:', {
       timestamp: new Date().toISOString(),
       prediction_id: data.id,
       model_type: 'video'
     });
     return $input.all();
     ```  
   - Connect from “Create Video Prediction”

6. **Create “Wait 5s” Node**  
   - Node Type: Wait  
   - Purpose: Pause 5 seconds before status check  
   - Parameters: Unit: seconds, Amount: 5  
   - Connect from “Log Request”

7. **Create “Check Status” Node**  
   - Node Type: HTTP Request  
   - Purpose: Query status of prediction  
   - Configuration:  
     - URL: `https://api.replicate.com/v1/predictions/{{ $('Create Video Prediction').item.json.id }}`  
     - Method: GET  
     - Headers:  
       - `Authorization`: `Bearer {{ $('Set API Token').item.json.api_token }}`  
     - Response Format: JSON, never error  
   - Connect from “Wait 5s” and from “Wait 10s” (retry loop)

8. **Create “Is Complete?” Node**  
   - Node Type: If  
   - Purpose: Check if prediction status is “succeeded”  
   - Condition: `$json.status` equals `"succeeded"`  
   - Connect from “Check Status”  
   - True branch: Connect to “Success Response”  
   - False branch: Connect to “Has Failed?”

9. **Create “Has Failed?” Node**  
   - Node Type: If  
   - Purpose: Check if prediction status is “failed”  
   - Condition: `$json.status` equals `"failed"`  
   - Connect from “Is Complete?” (false branch)  
   - True branch: Connect to “Error Response”  
   - False branch: Connect to “Wait 10s”

10. **Create “Wait 10s” Node**  
    - Node Type: Wait  
    - Purpose: Pause 10 seconds before retrying status check  
    - Parameters: Unit: seconds, Amount: 10  
    - Connect from “Has Failed?” (false branch)  
    - Output connects back to “Check Status” (retry loop)

11. **Create “Success Response” Node**  
    - Node Type: Set  
    - Purpose: Prepare successful response object  
    - Parameters: Set variable `response` to:  
      ```json
      {
        "success": true,
        "video_url": "{{ $json.output }}",
        "prediction_id": "{{ $json.id }}",
        "status": "{{ $json.status }}",
        "message": "Video generated successfully"
      }
      ```  
    - Connect from “Is Complete?” (true branch)  
    - Output connects to “Display Result”

12. **Create “Error Response” Node**  
    - Node Type: Set  
    - Purpose: Prepare error response object  
    - Parameters: Set variable `response` to:  
      ```json
      {
        "success": false,
        "error": "{{ $json.error || 'Video generation failed' }}",
        "prediction_id": "{{ $json.id }}",
        "status": "{{ $json.status }}",
        "message": "Failed to generate video"
      }
      ```  
    - Connect from “Has Failed?” (true branch)  
    - Output connects to “Display Result”

13. **Create “Display Result” Node**  
    - Node Type: Set  
    - Purpose: Assign final result to output variable  
    - Parameters: Set `final_result` to `{{ $json.response }}`  
    - Connect from “Success Response” and “Error Response”  
    - This node ends the workflow execution

14. **Add Sticky Notes** (Optional for documentation)  
    - Create two Sticky Note nodes containing:  
      - Support contact, tutorial links, and credits  
      - Detailed workflow overview, parameter explanation, troubleshooting, and external documentation links  
    - Place them visually for user reference; these nodes do not connect to others

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | Context or Link                                                                                   |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| OMNI-HUMAN GENERATOR: For questions or support contact Yaron@nofluff.online. Explore tips and tutorials on YouTube (https://www.youtube.com/@YaronBeen/videos) and LinkedIn (https://www.linkedin.com/in/yaronbeen/).                                                                                                                                                                                                                                                                                                    | Support contact and tutorial resources                                                           |
| Powered by Replicate API and n8n Automation. Model documentation: https://replicate.com/bytedance/omni-human, Replicate API docs: https://replicate.com/docs, n8n docs: https://docs.n8n.io.                                                                                                                                                                                                                                                                                                                               | Official documentation links                                                                     |
| Troubleshooting tips: Verify API token validity and credits, validate input parameters, monitor generation timeouts, and ensure expected output formats. Best practices include starting with default parameters, securing API tokens, and monitoring usage.                                                                                                                                                                                                                                                           | Troubleshooting and best practices                                                               |
| The workflow includes automated retry logic with wait intervals (5s and 10s) to handle asynchronous video generation processing gracefully, avoiding premature failure reporting.                                                                                                                                                                                                                                                                                                                                         | Workflow design notes                                                                            |

---

**Disclaimer:**  
The provided text is extracted exclusively from an automated workflow created using n8n, an integration and automation tool. This process strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled are legal and publicly available.