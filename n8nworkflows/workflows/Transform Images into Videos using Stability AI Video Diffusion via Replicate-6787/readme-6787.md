Transform Images into Videos using Stability AI Video Diffusion via Replicate

https://n8nworkflows.xyz/workflows/transform-images-into-videos-using-stability-ai-video-diffusion-via-replicate-6787


# Transform Images into Videos using Stability AI Video Diffusion via Replicate

---

### 1. Workflow Overview

This n8n workflow automates the process of transforming a single input image into a video using the Stability AI Stable-Video-Diffusion model via the Replicate API. It is designed for users needing automated video generation from images powered by advanced AI video diffusion models, suitable for creative content automation, marketing, or AI experimentation.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Initialization:** Starts the workflow manually and sets the Replicate API authentication token.
- **1.2 Parameter Configuration:** Defines all input parameters necessary for the video generation model, including defaults and user overrides.
- **1.3 Video Generation Request:** Submits the video generation request to the Replicate API, initiating the prediction.
- **1.4 Status Polling and Retry Logic:** Implements a loop that waits, checks the status of the prediction repeatedly, handling success, failure, or continuing to poll with backoff.
- **1.5 Result Handling and Logging:** Processes the final output or error, preparing structured responses and logging requests for monitoring.
- **1.6 User Guidance and Reference:** Provides extensive documentation and usage tips via sticky notes for users/operators.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Initialization

**Overview:**  
This block triggers the workflow manually and securely assigns the Replicate API token for authentication in subsequent API calls.

**Nodes Involved:**  
- Manual Trigger  
- Set API Token

**Node Details:**

- **Manual Trigger**  
  - *Type:* Manual trigger node  
  - *Role:* Starts workflow execution on-demand by user interaction.  
  - *Configuration:* Default, no parameters needed.  
  - *Connections:* Outputs to "Set API Token".  
  - *Edge Cases:* None typical; user must trigger manually.

- **Set API Token**  
  - *Type:* Set node  
  - *Role:* Stores the Replicate API token as a workflow variable for reuse.  
  - *Configuration:*  
    - `api_token` field is set to a placeholder string `"YOUR_REPLICATE_API_TOKEN"`. Users must replace with their actual token.  
  - *Connections:* Outputs to "Set Video Parameters".  
  - *Edge Cases:* Failure if token is invalid or missing in later API calls. No validation here.

---

#### 2.2 Parameter Configuration

**Overview:**  
Prepares and consolidates all parameters required by the Stability AI video diffusion model, allowing default values and dynamic referencing of the API token.

**Nodes Involved:**  
- Set Video Parameters

**Node Details:**

- **Set Video Parameters**  
  - *Type:* Set node  
  - *Role:* Defines all inputs for the video generation prediction, including both required and optional parameters.  
  - *Configuration:*  
    - Copies `api_token` from the previous node.  
    - Sets parameters with defaults:  
      - `seed`: -1 (indicates random seed)  
      - `cond_aug`: 0.02 (noise augmentation)  
      - `decoding_t`: 14 (frames per decoding batch)  
      - `input_image`: URL `"https://picsum.photos/512/512"` as a sample image  
      - `video_length`: `"14_frames_with_svd"` (video length preset)  
      - `sizing_strategy`: `"maintain_aspect_ratio"`  
      - `motion_bucket_id`: 127 (motion intensity)  
      - `frames_per_second`: 6  
  - *Connections:* Outputs to "Create Video Prediction".  
  - *Edge Cases:* Invalid parameter types or missing required `input_image` may cause API errors.

---

#### 2.3 Video Generation Request

**Overview:**  
Sends the configured parameters to the Replicate API to create a new video generation prediction. The node returns a prediction ID for tracking.

**Nodes Involved:**  
- Create Video Prediction  
- Log Request

**Node Details:**

- **Create Video Prediction**  
  - *Type:* HTTP Request node  
  - *Role:* POST request to `https://api.replicate.com/v1/predictions` to start video generation.  
  - *Configuration:*  
    - Uses POST method with JSON body including model version and input parameters dynamically injected via expressions.  
    - Sets HTTP headers:  
      - Authorization: Bearer token from `api_token`  
      - Prefer: wait (to wait for prediction)  
    - Response format: JSON, with error handling to never throw error on HTTP failure.  
  - *Connections:* Outputs to "Log Request".  
  - *Edge Cases:*  
    - Authentication failure (invalid API token)  
    - Network timeouts or API unavailability  
    - Invalid parameter combinations causing API rejection

- **Log Request**  
  - *Type:* Code node  
  - *Role:* Logs the prediction request details (timestamp, prediction ID, model type) for monitoring/debugging.  
  - *Configuration:* JavaScript logs to console, returns input unchanged.  
  - *Connections:* Outputs to "Wait 5s".  
  - *Edge Cases:* Logging failures are unlikely but do not block workflow.

---

#### 2.4 Status Polling and Retry Logic

**Overview:**  
Implements a polling loop to check the status of the video generation prediction every few seconds, handling both completion and failure states with appropriate delays and retries.

**Nodes Involved:**  
- Wait 5s  
- Check Status  
- Is Complete?  
- Has Failed?  
- Wait 10s

**Node Details:**

- **Wait 5s**  
  - *Type:* Wait node  
  - *Role:* Pauses workflow for 5 seconds before next status check.  
  - *Configuration:* Duration = 5 seconds.  
  - *Connections:* To "Check Status".  
  - *Edge Cases:* Delay could slow down responsiveness if prediction finishes sooner.

- **Check Status**  
  - *Type:* HTTP Request node  
  - *Role:* GET request to Replicate API with prediction ID to fetch current status.  
  - *Configuration:*  
    - URL dynamically constructed using prediction ID from "Create Video Prediction".  
    - Authorization header with API token.  
    - Accepts JSON response with no error throw on HTTP failure.  
  - *Connections:* To "Is Complete?".  
  - *Edge Cases:* Network failures or invalid prediction ID.

- **Is Complete?**  
  - *Type:* If node  
  - *Role:* Checks if prediction status is `"succeeded"`.  
  - *Configuration:* Condition: `$json.status === "succeeded"`  
  - *Connections:* On true, to "Success Response"; on false, to "Has Failed?".  
  - *Edge Cases:* Unexpected status values not handled explicitly.

- **Has Failed?**  
  - *Type:* If node  
  - *Role:* Checks if prediction status is `"failed"`.  
  - *Configuration:* Condition: `$json.status === "failed"`  
  - *Connections:* On true, to "Error Response"; on false, to "Wait 10s".  
  - *Edge Cases:* Other intermediate statuses cause loop continuation.

- **Wait 10s**  
  - *Type:* Wait node  
  - *Role:* Pauses workflow for 10 seconds before next status check retry after failure conditions are not met.  
  - *Configuration:* Duration = 10 seconds.  
  - *Connections:* Loops back to "Check Status".  
  - *Edge Cases:* Longer wait may delay failure detection.

---

#### 2.5 Result Handling and Logging

**Overview:**  
Processes the final prediction outcome, preparing structured success or error responses and displaying the results.

**Nodes Involved:**  
- Success Response  
- Error Response  
- Display Result

**Node Details:**

- **Success Response**  
  - *Type:* Set node  
  - *Role:* Constructs an object indicating successful video generation with relevant details including video URL and prediction ID.  
  - *Configuration:*  
    - Sets JSON field `response` with keys:  
      - `success: true`  
      - `video_url`: from prediction output  
      - `prediction_id`  
      - `status`  
      - `message`: "Video generated successfully"  
  - *Connections:* To "Display Result".  
  - *Edge Cases:* Missing or malformed output URL.

- **Error Response**  
  - *Type:* Set node  
  - *Role:* Constructs an error object with failure details.  
  - *Configuration:*  
    - Sets JSON field `response` with keys:  
      - `success: false`  
      - `error`: from error field or default message  
      - `prediction_id`  
      - `status`  
      - `message`: "Failed to generate video"  
  - *Connections:* To "Display Result".  
  - *Edge Cases:* May lack detailed error info if API response is incomplete.

- **Display Result**  
  - *Type:* Set node  
  - *Role:* Sets final output object under `final_result` for downstream use or interface display.  
  - *Configuration:* Copies `response` object from previous node.  
  - *Connections:* Terminal node.  
  - *Edge Cases:* None.

---

#### 2.6 User Guidance and Reference

**Overview:**  
Provides comprehensive documentation, instructions, parameter explanations, and contact information via sticky notes embedded in the workflow for operator reference.

**Nodes Involved:**  
- Sticky Note9  
- Sticky Note4

**Node Details:**

- **Sticky Note9**  
  - *Type:* Sticky Note node  
  - *Role:* Displays contact info and social media links of the workflow author.  
  - *Content:*  
    ```
    =======================================
            STABLE-VIDEO-DIFFUSION GENERATOR
    =======================================
    For any questions or support, please contact:
        Yaron@nofluff.online

    Explore more tips and tutorials here:
       - YouTube: https://www.youtube.com/@YaronBeen/videos
       - LinkedIn: https://www.linkedin.com/in/yaronbeen/
    =======================================
    ```
  - *Edge Cases:* None.

- **Sticky Note4**  
  - *Type:* Sticky Note node  
  - *Role:* Extensive documentation covering model overview, parameters, workflow explanation, benefits, quick start, and troubleshooting.  
  - *Content Highlights:*  
    - Model and API details  
    - Parameter reference and defaults  
    - Workflow node explanations  
    - Quick start and troubleshooting tips  
    - External resource links:  
      - https://replicate.com/stability-ai/stable-video-diffusion  
      - https://replicate.com/docs  
      - https://docs.n8n.io  
  - *Edge Cases:* None.

---

### 3. Summary Table

| Node Name            | Node Type          | Functional Role                  | Input Node(s)             | Output Node(s)              | Sticky Note                                                                                              |
|----------------------|--------------------|--------------------------------|---------------------------|-----------------------------|--------------------------------------------------------------------------------------------------------|
| Manual Trigger       | Manual Trigger     | Starts workflow execution       |                           | Set API Token               |                                                                                                        |
| Set API Token        | Set                | Stores API token for reuse      | Manual Trigger            | Set Video Parameters        |                                                                                                        |
| Set Video Parameters | Set                | Configures video generation parameters | Set API Token             | Create Video Prediction     |                                                                                                        |
| Create Video Prediction | HTTP Request      | Sends video generation request | Set Video Parameters      | Log Request                 |                                                                                                        |
| Log Request          | Code               | Logs request data for monitoring | Create Video Prediction   | Wait 5s                     |                                                                                                        |
| Wait 5s              | Wait               | Waits 5 seconds before status check | Log Request             | Check Status                |                                                                                                        |
| Check Status         | HTTP Request       | Checks prediction status        | Wait 5s, Wait 10s         | Is Complete?                |                                                                                                        |
| Is Complete?         | If                 | Checks if prediction succeeded  | Check Status              | Success Response, Has Failed? |                                                                                                        |
| Has Failed?          | If                 | Checks if prediction failed     | Is Complete?              | Error Response, Wait 10s    |                                                                                                        |
| Wait 10s             | Wait               | Waits 10 seconds before retry   | Has Failed?               | Check Status                |                                                                                                        |
| Success Response     | Set                | Formats successful response     | Is Complete?              | Display Result              |                                                                                                        |
| Error Response       | Set                | Formats error response          | Has Failed?               | Display Result              |                                                                                                        |
| Display Result       | Set                | Prepares final output           | Success Response, Error Response |                         |                                                                                                        |
| Sticky Note9         | Sticky Note        | Contact info and links          |                           |                             | Contact: Yaron@nofluff.online; YouTube: https://www.youtube.com/@YaronBeen/videos; LinkedIn: https://www.linkedin.com/in/yaronbeen/ |
| Sticky Note4         | Sticky Note        | Full workflow documentation    |                           |                             | Extensive model, parameter, workflow, troubleshooting info, and external links                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a Manual Trigger node named "Manual Trigger".  
   - No configuration needed. This node starts the workflow manually.

2. **Create Set Node for API Token**  
   - Add a Set node named "Set API Token".  
   - Add a string field `api_token` with value `"YOUR_REPLICATE_API_TOKEN"` (replace this with your actual Replicate API token).  
   - Connect output of "Manual Trigger" to this node.

3. **Create Set Node for Video Parameters**  
   - Add a Set node named "Set Video Parameters".  
   - Add fields:  
     - `api_token` (string): set via expression referencing `Set API Token` node's `api_token`  
     - `seed` (number): -1 (random seed)  
     - `cond_aug` (number): 0.02  
     - `decoding_t` (number): 14  
     - `input_image` (string): `"https://picsum.photos/512/512"` (default image URL)  
     - `video_length` (string): `"14_frames_with_svd"`  
     - `sizing_strategy` (string): `"maintain_aspect_ratio"`  
     - `motion_bucket_id` (number): 127  
     - `frames_per_second` (number): 6  
   - Connect output of "Set API Token" to this node.

4. **Create HTTP Request Node to Create Video Prediction**  
   - Add HTTP Request node named "Create Video Prediction".  
   - Method: POST  
   - URL: `https://api.replicate.com/v1/predictions`  
   - Headers:  
     - Authorization: Bearer token from `api_token` (expression referencing "Set Video Parameters")  
     - Prefer: wait  
   - Body (JSON):  
     ```json
     {
       "version": "stability-ai/stable-video-diffusion:3f0457e4619daac51203dedb472816fd4af51f3149fa7a9e0b5ffcf1b8172438",
       "input": {
         "seed": {{ $json.seed }},
         "cond_aug": {{ $json.cond_aug }},
         "decoding_t": {{ $json.decoding_t }},
         "input_image": "{{ $json.input_image }}",
         "video_length": "{{ $json.video_length }}",
         "sizing_strategy": "{{ $json.sizing_strategy }}",
         "motion_bucket_id": {{ $json.motion_bucket_id }},
         "frames_per_second": {{ $json.frames_per_second }}
       }
     }
     ```  
   - Enable JSON body and response as JSON.  
   - Connect output of "Set Video Parameters" to this node.

5. **Create Code Node to Log Request**  
   - Add Code node named "Log Request".  
   - JavaScript code:  
     ```javascript
     const data = $input.all()[0].json;
     console.log('stability-ai/stable-video-diffusion Request:', {
       timestamp: new Date().toISOString(),
       prediction_id: data.id,
       model_type: 'video'
     });
     return $input.all();
     ```  
   - Connect output of "Create Video Prediction" to this node.

6. **Create Wait Node for 5 Seconds**  
   - Add Wait node named "Wait 5s".  
   - Duration: 5 seconds.  
   - Connect output of "Log Request" to this node.

7. **Create HTTP Request Node to Check Status**  
   - Add HTTP Request node named "Check Status".  
   - Method: GET  
   - URL: `https://api.replicate.com/v1/predictions/{{ $('Create Video Prediction').item.json.id }}` (expression to insert prediction ID)  
   - Header: Authorization with Bearer token from "Set API Token".  
   - Response format: JSON with no error throw on failure.  
   - Connect output of "Wait 5s" and later "Wait 10s" nodes to this node (to enable polling loop).

8. **Create If Node to Check Completion**  
   - Add If node named "Is Complete?".  
   - Condition: Check if `$json.status === "succeeded"`.  
   - Connect output of "Check Status" to this node.

9. **Create If Node to Check Failure**  
   - Add If node named "Has Failed?".  
   - Condition: Check if `$json.status === "failed"`.  
   - Connect the "Is Complete?" false output to this node.

10. **Create Wait Node for 10 Seconds**  
    - Add Wait node named "Wait 10s".  
    - Duration: 10 seconds.  
    - Connect "Has Failed?" false output to "Wait 10s".  
    - Connect output of "Wait 10s" back to "Check Status" to continue polling.

11. **Create Set Node for Success Response**  
    - Add Set node named "Success Response".  
    - Assign a JSON object to `response` field:  
      ```json
      {
        "success": true,
        "video_url": $json.output,
        "prediction_id": $json.id,
        "status": $json.status,
        "message": "Video generated successfully"
      }
      ```  
    - Connect "Is Complete?" true output to this node.

12. **Create Set Node for Error Response**  
    - Add Set node named "Error Response".  
    - Assign a JSON object to `response` field:  
      ```json
      {
        "success": false,
        "error": $json.error || "Video generation failed",
        "prediction_id": $json.id,
        "status": $json.status,
        "message": "Failed to generate video"
      }
      ```  
    - Connect "Has Failed?" true output to this node.

13. **Create Set Node for Displaying Final Result**  
    - Add Set node named "Display Result".  
    - Assign `final_result` field to the `response` object from previous nodes.  
    - Connect outputs of both "Success Response" and "Error Response" to this node.

14. **Add Sticky Notes for Documentation and Contact**  
    - Add a sticky note with contact info and social media links (content from Sticky Note9).  
    - Add a sticky note with detailed workflow and parameter documentation (content from Sticky Note4).

---

### 5. General Notes & Resources

| Note Content                                                                                                                              | Context or Link                                                                                         |
|-------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| For any questions or support, contact Yaron@nofluff.online                                                                               | Contact email from Sticky Note9                                                                        |
| YouTube channel with tips and tutorials: https://www.youtube.com/@YaronBeen/videos                                                        | Sticky Note9 social media link                                                                         |
| LinkedIn profile: https://www.linkedin.com/in/yaronbeen/                                                                                 | Sticky Note9 social media link                                                                         |
| Model Documentation: https://replicate.com/stability-ai/stable-video-diffusion                                                           | Official Replicate model page                                                                          |
| Replicate API Documentation: https://replicate.com/docs                                                                                   | Official API documentation                                                                             |
| n8n Documentation: https://docs.n8n.io                                                                                                   | n8n official documentation                                                                             |
| Workflow includes robust retry logic with 5s and 10s wait nodes to handle asynchronous prediction completion                              | Design note                                                                                           |
| Important: Replace placeholder API token with your actual Replicate API token before running the workflow                                  | Critical configuration note                                                                            |
| Default parameter values provide a balanced starting point for video generation; adjust as needed for specific use cases                   | Operational guidance                                                                                   |
| The workflow logs prediction requests with timestamps and IDs to assist monitoring and troubleshooting                                     | Monitoring and debugging aid                                                                           |

---

**Disclaimer:**  
The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected material. All handled data is legal and publicly accessible.

---