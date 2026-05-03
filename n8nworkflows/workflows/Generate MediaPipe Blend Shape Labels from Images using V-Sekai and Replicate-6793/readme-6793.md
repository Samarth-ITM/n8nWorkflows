Generate MediaPipe Blend Shape Labels from Images using V-Sekai and Replicate

https://n8nworkflows.xyz/workflows/generate-mediapipe-blend-shape-labels-from-images-using-v-sekai-and-replicate-6793


# Generate MediaPipe Blend Shape Labels from Images using V-Sekai and Replicate

### 1. Workflow Overview

This workflow automates the process of generating MediaPipe blend shape labels from images using the V-Sekai Mediapipe Labeler model hosted on Replicate. It is designed for users who want to analyze facial blend shapes from images or videos through an AI model, facilitating downstream applications such as animation, avatar creation, or facial expression analysis.

The workflow is structured into the following logical blocks:  
- **1.1 Input Initialization:** Manual trigger and setting up API authentication.  
- **1.2 Parameter Configuration:** Defining image input parameters and model options.  
- **1.3 Prediction Creation:** Submitting the image to the Replicate API for processing.  
- **1.4 Prediction Monitoring:** Polling the API to check prediction status until completion or failure.  
- **1.5 Results Handling:** Handling success and failure cases by formatting responses.  
- **1.6 Logging & Output:** Logging request details and preparing the final output for downstream use or display.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Initialization

- **Overview:** Starts the workflow and sets the required API authentication token for the Replicate API.  
- **Nodes Involved:** Manual Trigger, Set API Token  

**Manual Trigger**  
- Type: Trigger node to manually start the workflow.  
- Configuration: No parameters; simply triggers workflow execution.  
- Inputs: None  
- Outputs: Connects to "Set API Token".  
- Edge Cases: None specific; user must manually initiate.  

**Set API Token**  
- Type: Set node to store the Replicate API token as a variable.  
- Configuration: Assigns a string variable `api_token` with a placeholder value `YOUR_REPLICATE_API_TOKEN` (to be replaced by user).  
- Inputs: From Manual Trigger.  
- Outputs: Connects to "Set Image Parameters".  
- Edge Cases: Failure if token is invalid or missing during later API calls.  
- Notes: This token authorizes all subsequent API requests to Replicate.

#### 2.2 Parameter Configuration

- **Overview:** Defines all input parameters for the image prediction request, including required and optional fields with defaults.  
- **Nodes Involved:** Set Image Parameters  

**Set Image Parameters**  
- Type: Set node to define all model input parameters.  
- Configuration:  
  - Copies `api_token` from the "Set API Token" node.  
  - Sets parameters such as:  
    - `test_mode` (boolean, default false)  
    - `max_people` (number, default 100)  
    - `media_path` (string, default sample image URL)  
    - `export_train` (boolean, default true)  
    - `aligned_media` (string, default sample image URL)  
    - `frame_sample_rate` (number, default 1)  
- Inputs: From "Set API Token".  
- Outputs: Connects to "Create Image Prediction".  
- Edge Cases: Input validation errors if parameters are wrongly typed or missing.  

#### 2.3 Prediction Creation

- **Overview:** Sends a POST request to Replicate's API to create a new image prediction job using configured parameters.  
- **Nodes Involved:** Create Image Prediction  

**Create Image Prediction**  
- Type: HTTP Request node  
- Configuration:  
  - URL: `https://api.replicate.com/v1/predictions`  
  - Method: POST  
  - Headers: Authorization with Bearer token from `api_token`, Prefer header set to "wait" (wait for prediction to complete if possible).  
  - Body: JSON structured with the model version ID and input parameters from previous node.  
  - Response: Expects JSON, set to never error on HTTP errors to allow graceful handling.  
- Inputs: From "Set Image Parameters".  
- Outputs: Connects to "Log Request".  
- Edge Cases: Network errors, invalid token, API rate limits, or malformed request payload.

#### 2.4 Prediction Monitoring

- **Overview:** Implements polling to check the status of the prediction until it is either complete or fails.  
- **Nodes Involved:** Log Request, Wait 5s, Check Status, Is Complete?, Has Failed?, Wait 10s  

**Log Request**  
- Type: Code node  
- Configuration: Logs prediction ID, timestamp, and model type to console for monitoring.  
- Inputs: From "Create Image Prediction".  
- Outputs: Connects to "Wait 5s".  
- Edge Cases: Console logging failure does not block workflow.  

**Wait 5s**  
- Type: Wait node  
- Configuration: waits 5 seconds before next action.  
- Inputs: From "Log Request".  
- Outputs: Connects to "Check Status".  
- Edge Cases: None critical; delay to avoid API flooding.  

**Check Status**  
- Type: HTTP Request node  
- Configuration:  
  - URL dynamically set to `https://api.replicate.com/v1/predictions/{prediction_id}` where `prediction_id` comes from "Create Image Prediction" node output.  
  - Method: GET  
  - Authorization header with API token.  
  - Expects JSON response with prediction status.  
- Inputs: From "Wait 5s" and "Wait 10s".  
- Outputs: Connects to "Is Complete?".  
- Edge Cases: API downtime, token expiry, network errors.  

**Is Complete?**  
- Type: IF node  
- Configuration: Checks if `status` field in response equals `"succeeded"`.  
- Inputs: From "Check Status".  
- Outputs:  
  - True branch to "Success Response" node.  
  - False branch to "Has Failed?" node.  
- Edge Cases: Unexpected status values.  

**Has Failed?**  
- Type: IF node  
- Configuration: Checks if `status` equals `"failed"`.  
- Inputs: From "Is Complete?" false branch.  
- Outputs:  
  - True branch to "Error Response".  
  - False branch to "Wait 10s" (retry delay).  
- Edge Cases: Status neither succeeded nor failed (e.g., "processing").  

**Wait 10s**  
- Type: Wait node  
- Configuration: waits 10 seconds before retrying status check.  
- Inputs: From "Has Failed?" false branch.  
- Outputs: Connects back to "Check Status".  
- Edge Cases: Prevents rapid polling and API rate limits.

#### 2.5 Results Handling

- **Overview:** Processes the final outcome of the prediction, formatting success or error messages.  
- **Nodes Involved:** Success Response, Error Response, Display Result  

**Success Response**  
- Type: Set node  
- Configuration: Creates a JSON object with keys:  
  - `success: true`  
  - `image_url` (output URL from prediction)  
  - `prediction_id`  
  - `status`  
  - `message` ("Image generated successfully")  
- Inputs: From "Is Complete?" true branch.  
- Outputs: Connects to "Display Result".  
- Edge Cases: Missing output URL or incomplete data.  

**Error Response**  
- Type: Set node  
- Configuration: Creates a JSON object with keys:  
  - `success: false`  
  - `error` message from prediction or default message  
  - `prediction_id`  
  - `status`  
  - `message` ("Failed to generate image")  
- Inputs: From "Has Failed?" true branch.  
- Outputs: Connects to "Display Result".  
- Edge Cases: Missing error details from API response.  

**Display Result**  
- Type: Set node  
- Configuration: Sets the final output object `final_result` to the response from either success or error node.  
- Inputs: From "Success Response" or "Error Response".  
- Outputs: Terminal node output of the workflow.  
- Edge Cases: None.

#### 2.6 Logging & Notes

- **Overview:** Provides static sticky notes with documentation, instructions, and support information embedded in the workflow for user assistance.  
- **Nodes Involved:** Sticky Note4, Sticky Note9  
- These contain detailed user guidance, parameter explanations, and contact/support references.  
- No input/output connections; purely informational.

---

### 3. Summary Table

| Node Name             | Node Type          | Functional Role                          | Input Node(s)            | Output Node(s)          | Sticky Note                                                                                                    |
|-----------------------|--------------------|----------------------------------------|--------------------------|-------------------------|---------------------------------------------------------------------------------------------------------------|
| Manual Trigger        | Trigger            | Starts workflow manually                | -                        | Set API Token            |                                                                                                               |
| Set API Token         | Set                | Stores Replicate API token              | Manual Trigger           | Set Image Parameters     |                                                                                                               |
| Set Image Parameters  | Set                | Defines input parameters for prediction | Set API Token            | Create Image Prediction  |                                                                                                               |
| Create Image Prediction | HTTP Request      | Submits image for prediction            | Set Image Parameters     | Log Request              |                                                                                                               |
| Log Request           | Code               | Logs prediction for monitoring          | Create Image Prediction  | Wait 5s                  |                                                                                                               |
| Wait 5s               | Wait               | Delay before status check                | Log Request              | Check Status             |                                                                                                               |
| Check Status          | HTTP Request       | Checks prediction status                 | Wait 5s, Wait 10s        | Is Complete?             |                                                                                                               |
| Is Complete?          | IF                 | Checks if prediction succeeded          | Check Status             | Success Response, Has Failed? |                                                                                                           |
| Has Failed?           | IF                 | Checks if prediction failed              | Is Complete?             | Error Response, Wait 10s |                                                                                                               |
| Wait 10s              | Wait               | Delay before retrying status check      | Has Failed?              | Check Status             |                                                                                                               |
| Success Response      | Set                | Formats success output                   | Is Complete?             | Display Result           |                                                                                                               |
| Error Response        | Set                | Formats error output                     | Has Failed?              | Display Result           |                                                                                                               |
| Display Result        | Set                | Sets final output object                 | Success Response, Error Response | -                 |                                                                                                               |
| Sticky Note9          | Sticky Note        | Workflow branding and support info      | -                        | -                       | =======================================<br>V-SEKAI.MEDIAPIPE-LABELER GENERATOR<br>For support contact Yaron@nofluff.online |
| Sticky Note4          | Sticky Note        | Detailed workflow and model documentation | -                      | -                       | Contains detailed instructions, parameter descriptions, troubleshooting, and reference links                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Manual Trigger node:**  
   - Name: `Manual Trigger`  
   - No configuration needed. This node starts the workflow.

3. **Add a Set node:**  
   - Name: `Set API Token`  
   - Add one assignment:  
     - Name: `api_token`  
     - Type: String  
     - Value: Replace `"YOUR_REPLICATE_API_TOKEN"` with your actual Replicate API token.

4. **Connect Manual Trigger to Set API Token.**

5. **Add another Set node:**  
   - Name: `Set Image Parameters`  
   - Define the following assignments:  
     - `api_token`: Expression referencing `Set API Token` node's `api_token` value.  
     - `test_mode` (Boolean): false (default)  
     - `max_people` (Number): 100  
     - `media_path` (String): e.g., `"https://picsum.photos/512/512"` (replace with actual image/video URL)  
     - `export_train` (Boolean): true  
     - `aligned_media` (String): optional aligned media URL or same as media_path  
     - `frame_sample_rate` (Number): 1

6. **Connect Set API Token to Set Image Parameters.**

7. **Add an HTTP Request node:**  
   - Name: `Create Image Prediction`  
   - Method: POST  
   - URL: `https://api.replicate.com/v1/predictions`  
   - Authentication: None (use header for Bearer token)  
   - Headers:  
     - `Authorization`: `Bearer {{$json.api_token}}` (expression)  
     - `Prefer`: `wait`  
   - Body Content Type: JSON  
   - Body: JSON structured as:  
     ```json
     {
       "version": "fire/v-sekai.mediapipe-labeler:f6cda62f5bdf02558ef9f9d23512a296db9927b2b93b6c57295f3e9d6ae696fa",
       "input": {
         "test_mode": {{$json.test_mode}},
         "max_people": {{$json.max_people}},
         "media_path": "{{$json.media_path}}",
         "export_train": {{$json.export_train}},
         "aligned_media": "{{$json.aligned_media}}",
         "frame_sample_rate": {{$json.frame_sample_rate}}
       }
     }
     ```  
   - Set "Never Error" to true to allow graceful failure handling.  

8. **Connect Set Image Parameters to Create Image Prediction.**

9. **Add a Code node:**  
   - Name: `Log Request`  
   - Paste the following JavaScript to log prediction details:  
     ```javascript
     const data = $input.all()[0].json;
     console.log('fire/v-sekai.mediapipe-labeler Request:', {
       timestamp: new Date().toISOString(),
       prediction_id: data.id,
       model_type: 'image'
     });
     return $input.all();
     ```
10. **Connect Create Image Prediction to Log Request.**

11. **Add a Wait node:**  
    - Name: `Wait 5s`  
    - Wait for 5 seconds.  

12. **Connect Log Request to Wait 5s.**

13. **Add an HTTP Request node:**  
    - Name: `Check Status`  
    - Method: GET  
    - URL: `https://api.replicate.com/v1/predictions/{{$json.id}}` (use expression from Create Image Prediction output)  
    - Headers:  
      - `Authorization`: `Bearer {{$node["Set API Token"].json.api_token}}` (expression)  
    - Never Error: true  

14. **Connect Wait 5s to Check Status and also connect Wait 10s (see step 20) to Check Status to allow retries.**

15. **Add an IF node:**  
    - Name: `Is Complete?`  
    - Condition: Check if `$json.status === "succeeded"` (string equality).  
    - True output: connect to Success Response node (step 16).  
    - False output: connect to Has Failed? node (step 17).

16. **Add a Set node:**  
    - Name: `Success Response`  
    - Assignments:  
      - `response` (object):  
        ```json
        {
          "success": true,
          "image_url": {{$json.output}},
          "prediction_id": {{$json.id}},
          "status": {{$json.status}},
          "message": "Image generated successfully"
        }
        ```  

17. **Add an IF node:**  
    - Name: `Has Failed?`  
    - Condition: Check if `$json.status === "failed"`.  
    - True output: connect to Error Response node (step 18).  
    - False output: connect to Wait 10s node (step 19).

18. **Add a Set node:**  
    - Name: `Error Response`  
    - Assignments:  
      - `response` (object):  
        ```json
        {
          "success": false,
          "error": $json.error || "Image generation failed",
          "prediction_id": $json.id,
          "status": $json.status,
          "message": "Failed to generate image"
        }
        ```  

19. **Add a Wait node:**  
    - Name: `Wait 10s`  
    - Wait for 10 seconds before retrying.  

20. **Connect Has Failed? false output to Wait 10s and Wait 10s output back to Check Status.**

21. **Connect Success Response and Error Response nodes to a Set node:**  
    - Name: `Display Result`  
    - Assignments:  
      - `final_result`: set to the incoming `response` object.  

22. **This `Display Result` node is the final output of the workflow.**

23. **(Optional) Add Sticky Note nodes:**  
    - For documentation, support info, and parameter references, create two sticky notes with the following content summarized from the original workflow:  
      - Sticky Note9: Contact info and branding message.  
      - Sticky Note4: Detailed instructions, parameter descriptions, troubleshooting tips, and links to Replicate and n8n documentation.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | Context or Link                                                                                      |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| For any questions or support, please contact: Yaron@nofluff.online. Explore more tips and tutorials at YouTube (https://www.youtube.com/@YaronBeen/videos) and LinkedIn (https://www.linkedin.com/in/yaronbeen/).                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 | Support contact & branding                                                                          |
| Model documentation and API reference: https://replicate.com/fire/v-sekai.mediapipe-labeler and https://replicate.com/docs. The workflow is powered by Replicate API and integrates with n8n automation for seamless AI image prediction.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | Official documentation and resources                                                               |
| Parameter notes: `media_path` is required and must point to an accessible image or video URL. Optional parameters include `test_mode`, `max_people` (1-100), `export_train`, `aligned_media`, and `frame_sample_rate`. Ensure API token is valid and has sufficient quota.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | Parameter guidelines                                                                               |
| Troubleshooting tips: Check API token validity, network connectivity, and parameter formats. Handle longer processing times with sufficient polling delays. Monitor logs for prediction IDs and timestamps to diagnose issues.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Troubleshooting recommendations                                                                   |
| The workflow uses "Prefer: wait" header to attempt synchronous prediction completion, but still implements polling with retries to handle asynchronous responses reliably.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | API usage pattern explanation                                                                      |

---

**Disclaimer:**  
This documentation exclusively covers the workflow automated with n8n and respects content policies. All data processed is legal and public.