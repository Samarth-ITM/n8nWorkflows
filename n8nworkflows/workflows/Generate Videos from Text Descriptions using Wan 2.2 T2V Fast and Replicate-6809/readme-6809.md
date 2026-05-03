Generate Videos from Text Descriptions using Wan 2.2 T2V Fast and Replicate

https://n8nworkflows.xyz/workflows/generate-videos-from-text-descriptions-using-wan-2-2-t2v-fast-and-replicate-6809


# Generate Videos from Text Descriptions using Wan 2.2 T2V Fast and Replicate

### 1. Workflow Overview

This workflow automates video generation from text descriptions using the "wan-2.2-t2v-fast" video generation model hosted on Replicate. It is designed for users who want to convert textual prompts into short AI-generated videos efficiently and reliably.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Initialization**: Starts the workflow manually and sets up authentication.
- **1.2 Parameter Configuration**: Defines the video generation parameters including the prompt, resolution, and other model-specific settings.
- **1.3 Video Generation Request**: Sends the generation request to Replicate API and logs the request.
- **1.4 Status Polling Loop**: Waits and repeatedly checks the status of the video generation until it completes successfully or fails.
- **1.5 Result Handling**: Routes successful outputs to a success response, and errors to an error response.
- **1.6 Final Output and Logging**: Prepares the final result output and logs information for monitoring.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Initialization

- **Overview:**  
  Begins the workflow execution manually and sets the Replicate API token required for authentication.

- **Nodes Involved:**  
  - Manual Trigger  
  - Set API Token

- **Node Details:**

  - **Manual Trigger**  
    - Type: Trigger (manual)  
    - Role: Starts the workflow when the user clicks to begin video generation.  
    - Configuration: No parameters; manual activation.  
    - Inputs: None  
    - Outputs: Connected to "Set API Token"  
    - Edge Cases: None (manual start)

  - **Set API Token**  
    - Type: Set  
    - Role: Stores the Replicate API token as a workflow variable for reuse.  
    - Configuration: Sets a string variable `api_token` with placeholder `"YOUR_REPLICATE_API_TOKEN"`. Users must replace this with their actual token for authentication.  
    - Inputs: From Manual Trigger  
    - Outputs: To "Set Video Parameters"  
    - Edge Cases: Failure if token is invalid or missing; must be replaced before use.

---

#### 2.2 Parameter Configuration

- **Overview:**  
  Defines all input parameters required by the video generation model, including the prompt and various optional settings.

- **Nodes Involved:**  
  - Set Video Parameters

- **Node Details:**

  - **Set Video Parameters**  
    - Type: Set  
    - Role: Prepares the payload parameters for the model request.  
    - Configuration:  
      - `api_token`: copied from "Set API Token" output for authorization.  
      - `seed`: set to -1 (special value to indicate random seed).  
      - `prompt`: default is `"A person walking through a magical forest with glowing particles"`.  
      - `go_fast`: boolean true to enable faster generation.  
      - `num_frames`: 81 frames for video length.  
      - `resolution`: "480p".  
      - `aspect_ratio`: "16:9".  
      - `sample_shift`: 12.  
      - `frames_per_second`: 16.  
    - Inputs: From "Set API Token"  
    - Outputs: To "Create Video Prediction"  
    - Edge Cases: Incorrect prompt or invalid parameter types may cause API errors.

---

#### 2.3 Video Generation Request

- **Overview:**  
  Sends an HTTP POST request to Replicate API to start video generation with configured parameters and logs the request details.

- **Nodes Involved:**  
  - Create Video Prediction  
  - Log Request

- **Node Details:**

  - **Create Video Prediction**  
    - Type: HTTP Request  
    - Role: Calls Replicate API endpoint `/v1/predictions` to submit the video generation job.  
    - Configuration:  
      - URL: `https://api.replicate.com/v1/predictions`  
      - Method: POST  
      - Headers: Authorization with Bearer token from `api_token`, Prefer header set to "wait" to wait for immediate response.  
      - Body: JSON containing model version and input parameters from "Set Video Parameters".  
    - Inputs: From "Set Video Parameters"  
    - Outputs: To "Log Request"  
    - Edge Cases: Authentication failure, network timeouts, invalid model parameters.

  - **Log Request**  
    - Type: Code  
    - Role: Logs the prediction request details for monitoring and debugging.  
    - Configuration: JavaScript code logs timestamp, prediction ID, and model type.  
    - Inputs: From "Create Video Prediction"  
    - Outputs: To "Wait 5s"  
    - Edge Cases: Logging failure (rare), ensure console is accessible.

---

#### 2.4 Status Polling Loop

- **Overview:**  
  Implements a polling mechanism to repeatedly check the status of the video generation prediction until completion or failure.

- **Nodes Involved:**  
  - Wait 5s  
  - Check Status  
  - Is Complete?  
  - Has Failed?  
  - Wait 10s

- **Node Details:**

  - **Wait 5s**  
    - Type: Wait  
    - Role: Delays 5 seconds between status requests to avoid overwhelming API.  
    - Inputs: From "Log Request" or "Has Failed?" (retry path)  
    - Outputs: To "Check Status"  
    - Edge Cases: Delay too short may cause API rate limiting; too long delays responsiveness.

  - **Check Status**  
    - Type: HTTP Request  
    - Role: Queries Replicate API `/v1/predictions/{id}` to get current status of prediction.  
    - Configuration:  
      - URL constructed with prediction ID from "Create Video Prediction".  
      - Method: GET  
      - Headers: Authorization with Bearer token.  
    - Inputs: From "Wait 5s" or "Wait 10s"  
    - Outputs: To "Is Complete?"  
    - Edge Cases: Network errors, invalid prediction ID, token expiration.

  - **Is Complete? (If node)**  
    - Type: If  
    - Role: Checks if prediction status equals `"succeeded"`.  
    - Inputs: From "Check Status"  
    - Outputs:  
      - True: To "Success Response"  
      - False: To "Has Failed?"  
    - Edge Cases: Status field missing or malformed.

  - **Has Failed? (If node)**  
    - Type: If  
    - Role: Checks if prediction status equals `"failed"`.  
    - Inputs: From "Is Complete?" (false branch)  
    - Outputs:  
      - True: To "Error Response"  
      - False: To "Wait 10s" (retry polling)  
    - Edge Cases: Status unknown or unexpected values.

  - **Wait 10s**  
    - Type: Wait  
    - Role: Delays 10 seconds before retrying status check after non-failure incomplete status.  
    - Inputs: From "Has Failed?" (false branch)  
    - Outputs: To "Check Status"  
    - Edge Cases: Similar to "Wait 5s".

---

#### 2.5 Result Handling

- **Overview:**  
  Based on prediction outcome, formats either success or error response objects for downstream consumption.

- **Nodes Involved:**  
  - Success Response  
  - Error Response

- **Node Details:**

  - **Success Response**  
    - Type: Set  
    - Role: Constructs a JSON object indicating success, including video URL, prediction ID, status, and message.  
    - Inputs: From "Is Complete?" (true branch)  
    - Outputs: To "Display Result"  
    - Edge Cases: Output missing expected fields.

  - **Error Response**  
    - Type: Set  
    - Role: Constructs a JSON object indicating failure, including error message, prediction ID, status, and message.  
    - Inputs: From "Has Failed?" (true branch)  
    - Outputs: To "Display Result"  
    - Edge Cases: Error message may be missing or generic.

---

#### 2.6 Final Output and Logging

- **Overview:**  
  Prepares the final structured output for consumption by users or other systems and serves as the last node in the workflow.

- **Nodes Involved:**  
  - Display Result

- **Node Details:**

  - **Display Result**  
    - Type: Set  
    - Role: Copies the response object (success or error) into a `final_result` field for output.  
    - Inputs: From "Success Response" or "Error Response"  
    - Outputs: None (end of workflow)  
    - Edge Cases: None significant.

---

### 3. Summary Table

| Node Name            | Node Type       | Functional Role                      | Input Node(s)            | Output Node(s)            | Sticky Note                                                                                                                 |
|----------------------|-----------------|-----------------------------------|--------------------------|---------------------------|-----------------------------------------------------------------------------------------------------------------------------|
| Manual Trigger       | manualTrigger   | Starts workflow manually           |                          | Set API Token              | See detailed workflow overview and support contacts in sticky notes                                                         |
| Set API Token        | set             | Stores Replicate API token         | Manual Trigger           | Set Video Parameters       | Remember to replace 'YOUR_REPLICATE_API_TOKEN' with your actual token                                                        |
| Set Video Parameters | set             | Defines video generation parameters| Set API Token            | Create Video Prediction    | Contains required and optional model parameters with sensible defaults                                                      |
| Create Video Prediction | httpRequest   | Sends generation request to Replicate API | Set Video Parameters | Log Request                | Uses Replicate API with "wait" preference for synchronous response                                                          |
| Log Request          | code            | Logs prediction request details    | Create Video Prediction  | Wait 5s                   | Logs timestamp and prediction ID for monitoring                                                                              |
| Wait 5s              | wait            | Delays 5 seconds before status check | Log Request, Has Failed? (retry) | Check Status             | Controls polling rate to avoid API overload                                                                                  |
| Check Status         | httpRequest     | Checks prediction status           | Wait 5s, Wait 10s        | Is Complete?               | Queries prediction status using prediction ID                                                                                |
| Is Complete?         | if              | Checks if prediction succeeded     | Check Status             | Success Response, Has Failed? | Branches workflow based on completion                                                                                        |
| Has Failed?          | if              | Checks if prediction failed        | Is Complete?             | Error Response, Wait 10s   | Implements retry logic and error detection                                                                                   |
| Wait 10s             | wait            | Waits 10 seconds for retry         | Has Failed?              | Check Status               | Longer delay after failure or incomplete status                                                                              |
| Success Response     | set             | Formats success JSON response      | Is Complete?             | Display Result             | Includes video URL and success message                                                                                       |
| Error Response       | set             | Formats error JSON response        | Has Failed?              | Display Result             | Includes error details and failure message                                                                                   |
| Display Result       | set             | Prepares final output to user/system | Success Response, Error Response |                       | Outputs final structured response                                                                                            |
| Sticky Note9         | stickyNote      | Displays contact and support info  |                          |                           | Contact: Yaron@nofluff.online; YouTube and LinkedIn links                                                                    |
| Sticky Note4         | stickyNote      | Detailed workflow and model overview|                          |                           | Extensive documentation including parameter guide and troubleshooting links                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "Manual Trigger" Node**  
   - Type: Manual Trigger  
   - No parameters required  
   - Position: Start of workflow

2. **Create "Set API Token" Node**  
   - Type: Set  
   - Add string variable named `api_token`  
   - Value: Replace `"YOUR_REPLICATE_API_TOKEN"` with your actual Replicate API token  
   - Connect "Manual Trigger" → "Set API Token"

3. **Create "Set Video Parameters" Node**  
   - Type: Set  
   - Assign following variables:  
     - `api_token` = expression: `{{$node["Set API Token"].json["api_token"]}}`  
     - `seed` = `-1` (number)  
     - `prompt` = `"A person walking through a magical forest with glowing particles"` (string)  
     - `go_fast` = `true` (boolean)  
     - `num_frames` = `81` (number)  
     - `resolution` = `"480p"` (string)  
     - `aspect_ratio` = `"16:9"` (string)  
     - `sample_shift` = `12` (number)  
     - `frames_per_second` = `16` (number)  
   - Connect "Set API Token" → "Set Video Parameters"

4. **Create "Create Video Prediction" Node**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.replicate.com/v1/predictions`  
   - Headers:  
     - `Authorization`: `Bearer {{$json["api_token"]}`  
     - `Prefer`: `wait`  
   - Body (JSON):  
     ```json
     {
       "version": "wan-video/wan-2.2-t2v-fast:17eacc978d91e78a88d808d66d3bc7cbc7f8a2248ace0c48cab8105ad1a68509",
       "input": {
         "seed": {{$json.seed}},
         "prompt": "{{$json.prompt}}",
         "go_fast": {{$json.go_fast}},
         "num_frames": {{$json.num_frames}},
         "resolution": "{{$json.resolution}}",
         "aspect_ratio": "{{$json.aspect_ratio}}",
         "sample_shift": {{$json.sample_shift}},
         "frames_per_second": {{$json.frames_per_second}}
       }
     }
     ```  
   - Response Format: JSON, Never error  
   - Connect "Set Video Parameters" → "Create Video Prediction"

5. **Create "Log Request" Node**  
   - Type: Code  
   - Language: JavaScript  
   - Code:  
     ```javascript
     const data = $input.all()[0].json;
     console.log('wan-video/wan-2.2-t2v-fast Request:', {
       timestamp: new Date().toISOString(),
       prediction_id: data.id,
       model_type: 'video'
     });
     return $input.all();
     ```  
   - Connect "Create Video Prediction" → "Log Request"

6. **Create "Wait 5s" Node**  
   - Type: Wait  
   - Parameters: 5 seconds delay  
   - Connect "Log Request" → "Wait 5s"

7. **Create "Check Status" Node**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://api.replicate.com/v1/predictions/{{$node["Create Video Prediction"].json["id"]}}`  
   - Headers:  
     - `Authorization`: `Bearer {{$node["Set API Token"].json["api_token"]}}`  
   - Response Format: JSON, Never error  
   - Connect "Wait 5s" → "Check Status"  
   - Also connect "Wait 10s" (to be created) → "Check Status" (for retry)

8. **Create "Is Complete?" Node**  
   - Type: If  
   - Condition: Check if `{{$json["status"]}}` equals `"succeeded"` (string equals, case insensitive false)  
   - Connect "Check Status" → "Is Complete?"

9. **Create "Has Failed?" Node**  
   - Type: If  
   - Condition: Check if `{{$json["status"]}}` equals `"failed"`  
   - Connect "Is Complete?" (false branch) → "Has Failed?"

10. **Create "Success Response" Node**  
    - Type: Set  
    - Assign object variable `response`:  
      ```json
      {
        "success": true,
        "video_url": "{{$json.output}}",
        "prediction_id": "{{$json.id}}",
        "status": "{{$json.status}}",
        "message": "Video generated successfully"
      }
      ```  
    - Connect "Is Complete?" (true branch) → "Success Response"

11. **Create "Error Response" Node**  
    - Type: Set  
    - Assign object variable `response`:  
      ```json
      {
        "success": false,
        "error": "{{$json.error || 'Video generation failed'}}",
        "prediction_id": "{{$json.id}}",
        "status": "{{$json.status}}",
        "message": "Failed to generate video"
      }
      ```  
    - Connect "Has Failed?" (true branch) → "Error Response"

12. **Create "Wait 10s" Node**  
    - Type: Wait  
    - Parameters: 10 seconds delay  
    - Connect "Has Failed?" (false branch) → "Wait 10s"

13. **Create "Display Result" Node**  
    - Type: Set  
    - Assign object variable `final_result` with value from `{{$json.response}}`  
    - Connect "Success Response" → "Display Result"  
    - Connect "Error Response" → "Display Result"

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                 | Context or Link                                                                                   |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Contact support at Yaron@nofluff.online for workflow assistance.                                                                                                                                                                            | Support email                                                                                     |
| Explore video generation tutorials and tips on YouTube: https://www.youtube.com/@YaronBeen/videos                                                                                                                                           | YouTube channel                                                                                   |
| LinkedIn profile for professional networking and updates: https://www.linkedin.com/in/yaronbeen/                                                                                                                                            | LinkedIn profile                                                                                  |
| Model documentation and API details available at https://replicate.com/wan-video/wan-2.2-t2v-fast                                                                                                                                             | Model documentation                                                                              |
| General Replicate API documentation at https://replicate.com/docs                                                                                                                                                                            | API reference                                                                                     |
| n8n official documentation for node and workflow configuration: https://docs.n8n.io                                                                                                                                                          | n8n docs                                                                                        |
| This workflow includes robust retry and error handling to deal with API timeouts, authentication errors, and generation failures. Default parameters are set for easy testing and can be customized as needed.                               | Workflow design note                                                                             |
| Pricing for video generation is based on video duration at 16 frames per second; adjust `frames_per_second` accordingly to manage cost and output quality.                                                                                  | Pricing consideration                                                                            |

---

This documentation fully describes the "wan-video/wan-2.2-t2v-fast - Video Generator" workflow to empower users and developers to understand, reproduce, and modify the workflow efficiently and reliably.