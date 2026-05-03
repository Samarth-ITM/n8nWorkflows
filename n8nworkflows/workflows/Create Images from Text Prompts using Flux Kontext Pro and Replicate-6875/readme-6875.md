Create Images from Text Prompts using Flux Kontext Pro and Replicate

https://n8nworkflows.xyz/workflows/create-images-from-text-prompts-using-flux-kontext-pro-and-replicate-6875


# Create Images from Text Prompts using Flux Kontext Pro and Replicate

### 1. Workflow Overview

This workflow is designed to generate images from text prompts using the Flux Kontext Pro model via the Replicate API. It is aimed at users who want to automate high-quality AI-driven image generation based on descriptive text inputs, optionally referencing an input image.

The workflow is divided into the following logical blocks:

**1.1 Input Reception and Initialization**  
Entry point and basic setup, including API token configuration.

**1.2 Parameter Preparation**  
Setting all required and optional parameters for the image generation request.

**1.3 Image Generation Request**  
Submitting the image generation job to the Replicate API and logging the request.

**1.4 Status Polling and Handling**  
Repeatedly checking the job status with wait intervals, handling success and failure cases.

**1.5 Response Construction and Output**  
Formatting the final success or error response and making it available downstream.

**1.6 Monitoring and Documentation**  
Embedded notes and logging for development, support, and user guidance.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Initialization

- **Overview:**  
Starts the workflow execution manually and sets the API token required for authentication with Replicate.

- **Nodes Involved:**  
  - Manual Trigger  
  - Set API Token

- **Node Details:**

1. **Manual Trigger**  
   - Type: Trigger Node  
   - Role: Initiates the workflow manually on user interaction.  
   - Config: Default manual trigger without parameters.  
   - Inputs: None  
   - Outputs: To "Set API Token"  
   - Edge Cases: None; manual start ensures control on execution.  

2. **Set API Token**  
   - Type: Set Node  
   - Role: Stores the Replicate API token for authorization.  
   - Config: A string variable `api_token` set to `"YOUR_REPLICATE_API_TOKEN"` (placeholder to be replaced).  
   - Inputs: From Manual Trigger  
   - Outputs: To "Set Image Parameters"  
   - Edge Cases: Missing or invalid token will cause authentication failures later.

---

#### 2.2 Parameter Preparation

- **Overview:**  
Sets all parameters needed for the image generation API call, including prompt, seed, input image, aspect ratio, output format, and safety tolerance.

- **Nodes Involved:**  
  - Set Image Parameters

- **Node Details:**

1. **Set Image Parameters**  
   - Type: Set Node  
   - Role: Prepares and consolidates all input parameters for the image generation request.  
   - Config:  
     - Copies `api_token` from the previous node.  
     - Sets defaults:  
       - `seed`: -1 (random seed)  
       - `prompt`: "A beautiful landscape with mountains and a lake at sunset"  
       - `input_image`: URL to a 512x512 placeholder image  
       - `aspect_ratio`: "match_input_image"  
       - `output_format`: "png"  
       - `safety_tolerance`: 2 (moderate safety tolerance)  
       - `prompt_upsampling`: false  
   - Inputs: From "Set API Token"  
   - Outputs: To "Create Image Prediction"  
   - Edge Cases: Improper prompt or invalid image URLs may cause generation errors.

---

#### 2.3 Image Generation Request

- **Overview:**  
Sends the image generation request to Replicate API, initiating the prediction process.

- **Nodes Involved:**  
  - Create Image Prediction  
  - Log Request

- **Node Details:**

1. **Create Image Prediction**  
   - Type: HTTP Request Node  
   - Role: Posts prediction request with parameters to Replicate API.  
   - Config:  
     - URL: `https://api.replicate.com/v1/predictions`  
     - Method: POST  
     - Headers: Authorization with Bearer token from `api_token`, Prefer header set to "wait" (wait for processing)  
     - Body: JSON including version ID, and all input parameters interpolated from previous node.  
     - Response: JSON, configured to never error (to handle gracefully)  
   - Inputs: From "Set Image Parameters"  
   - Outputs: To "Log Request"  
   - Edge Cases: Network errors, invalid API token, or API limits could fail request.

2. **Log Request**  
   - Type: Code Node  
   - Role: Logs the prediction request details for monitoring and debugging.  
   - Config: JavaScript code logs timestamp, prediction ID, and model type to console.  
   - Inputs: From "Create Image Prediction"  
   - Outputs: To "Wait 5s"  
   - Edge Cases: Logging failures won't affect workflow but may reduce monitoring visibility.

---

#### 2.4 Status Polling and Handling

- **Overview:**  
Waits fixed intervals, then checks the prediction status repeatedly until success or failure is confirmed.

- **Nodes Involved:**  
  - Wait 5s  
  - Check Status  
  - Is Complete? (If)  
  - Has Failed? (If)  
  - Wait 10s

- **Node Details:**

1. **Wait 5s**  
   - Type: Wait Node  
   - Role: Pauses workflow for 5 seconds before status check.  
   - Config: Wait 5 seconds  
   - Inputs: From "Log Request"  
   - Outputs: To "Check Status"  
   - Edge Cases: Minimal; waits are fixed.

2. **Check Status**  
   - Type: HTTP Request Node  
   - Role: Fetches current prediction status from Replicate API using prediction ID.  
   - Config:  
     - URL is dynamic: `https://api.replicate.com/v1/predictions/{{prediction_id}}`  
     - Method: GET  
     - Authorization header with API token  
     - Response JSON, never error  
   - Inputs: From "Wait 5s" and "Wait 10s" (for retries)  
   - Outputs: To "Is Complete?"  
   - Edge Cases: API limits, network issues, or invalid prediction IDs.

3. **Is Complete?**  
   - Type: If Node  
   - Role: Checks if prediction status equals "succeeded"  
   - Config: Condition `status == "succeeded"`  
   - Inputs: From "Check Status"  
   - Outputs:  
     - True: To "Success Response"  
     - False: To "Has Failed?"  
   - Edge Cases: Unexpected status values.

4. **Has Failed?**  
   - Type: If Node  
   - Role: Checks if prediction status equals "failed"  
   - Config: Condition `status == "failed"`  
   - Inputs: From "Is Complete?"  
   - Outputs:  
     - True: To "Error Response"  
     - False: To "Wait 10s" (retry)  
   - Edge Cases: Status stuck in pending or unknown state.

5. **Wait 10s**  
   - Type: Wait Node  
   - Role: Pauses 10 seconds before retrying status check after a failure condition check.  
   - Config: Wait 10 seconds  
   - Inputs: From "Has Failed?" false branch  
   - Outputs: To "Check Status"  
   - Edge Cases: Long-running jobs may cause delays.

---

#### 2.5 Response Construction and Output

- **Overview:**  
Builds structured JSON responses for success or error and makes them available as final output.

- **Nodes Involved:**  
  - Success Response  
  - Error Response  
  - Display Result

- **Node Details:**

1. **Success Response**  
   - Type: Set Node  
   - Role: Creates a success response object with image URL and metadata.  
   - Config:  
     - `response` object includes:  
       - `success: true`  
       - `image_url` (from prediction output)  
       - `prediction_id`  
       - `status`  
       - message string "Image generated successfully"  
   - Inputs: From "Is Complete?" true branch  
   - Outputs: To "Display Result"  
   - Edge Cases: Missing or malformed output URLs.

2. **Error Response**  
   - Type: Set Node  
   - Role: Creates an error response object with error details.  
   - Config:  
     - `response` object includes:  
       - `success: false`  
       - `error` (from JSON error or default message)  
       - `prediction_id`  
       - `status`  
       - message string "Failed to generate image"  
   - Inputs: From "Has Failed?" true branch  
   - Outputs: To "Display Result"  
   - Edge Cases: Absence of error message field.

3. **Display Result**  
   - Type: Set Node  
   - Role: Packages the final response (success or error) into `final_result`.  
   - Config: Sets `final_result` to the previous node’s `response`.  
   - Inputs: From both "Success Response" and "Error Response"  
   - Outputs: None (end of workflow)  
   - Edge Cases: None.

---

#### 2.6 Monitoring and Documentation

- **Overview:**  
Provides embedded sticky notes for user instructions, model details, and contact information.

- **Nodes Involved:**  
  - Sticky Note9  
  - Sticky Note4

- **Node Details:**

1. **Sticky Note9**  
   - Type: Sticky Note  
   - Role: Contact and support information for the workflow author.  
   - Content includes:  
     - Contact email: Yaron@nofluff.online  
     - YouTube and LinkedIn resource links  
   - Position: Top-left corner (visible at workflow start)  
   - Edge Cases: None.

2. **Sticky Note4**  
   - Type: Sticky Note  
   - Role: Comprehensive documentation embedded in the workflow.  
   - Content includes:  
     - Model overview  
     - Parameter reference and descriptions  
     - Workflow components explanation  
     - Benefits and quick start instructions  
     - Troubleshooting and external resource links  
   - Edge Cases: None.

---

### 3. Summary Table

| Node Name            | Node Type          | Functional Role                       | Input Node(s)         | Output Node(s)         | Sticky Note                                                                                                                       |
|----------------------|--------------------|------------------------------------|-----------------------|-----------------------|----------------------------------------------------------------------------------------------------------------------------------|
| Manual Trigger       | Manual Trigger     | Starts workflow execution           | None                  | Set API Token          | See Sticky Note4 for detailed model and workflow documentation.                                                                  |
| Set API Token        | Set                | Sets Replicate API token            | Manual Trigger        | Set Image Parameters   | See Sticky Note4 for detailed model and workflow documentation.                                                                  |
| Set Image Parameters | Set                | Prepares image generation parameters| Set API Token         | Create Image Prediction| See Sticky Note4 for detailed model and workflow documentation.                                                                  |
| Create Image Prediction | HTTP Request     | Sends image generation request     | Set Image Parameters  | Log Request            | See Sticky Note4 for detailed model and workflow documentation.                                                                  |
| Log Request          | Code               | Logs request details                | Create Image Prediction| Wait 5s                | See Sticky Note4 for detailed model and workflow documentation.                                                                  |
| Wait 5s              | Wait               | Waits 5 seconds before status check| Log Request           | Check Status           | See Sticky Note4 for detailed model and workflow documentation.                                                                  |
| Check Status         | HTTP Request       | Checks prediction status            | Wait 5s, Wait 10s     | Is Complete?           | See Sticky Note4 for detailed model and workflow documentation.                                                                  |
| Is Complete?         | If                 | Determines if prediction succeeded  | Check Status          | Success Response, Has Failed? | See Sticky Note4 for detailed model and workflow documentation.                                                                  |
| Has Failed?          | If                 | Determines if prediction failed     | Is Complete?          | Error Response, Wait 10s| See Sticky Note4 for detailed model and workflow documentation.                                                                  |
| Wait 10s             | Wait               | Waits 10 seconds before retry       | Has Failed?           | Check Status           | See Sticky Note4 for detailed model and workflow documentation.                                                                  |
| Success Response     | Set                | Builds success JSON response        | Is Complete?          | Display Result         | See Sticky Note4 for detailed model and workflow documentation.                                                                  |
| Error Response       | Set                | Builds error JSON response          | Has Failed?            | Display Result         | See Sticky Note4 for detailed model and workflow documentation.                                                                  |
| Display Result       | Set                | Final output packaging              | Success Response, Error Response | None            | See Sticky Note4 for detailed model and workflow documentation.                                                                  |
| Sticky Note9         | Sticky Note        | Contact info and support links      | None                  | None                   | Contains support contacts and resource links.                                                                                     |
| Sticky Note4         | Sticky Note        | Full workflow and model documentation| None                  | None                   | Contains detailed documentation, instructions, parameter references, and troubleshooting guidance.                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger node**  
   - No configuration needed. This will be your workflow entry point.

2. **Add Set node named "Set API Token"**  
   - Add string field `api_token` with value `"YOUR_REPLICATE_API_TOKEN"` (replace with your actual token).  
   - Connect Manual Trigger → Set API Token.

3. **Add Set node named "Set Image Parameters"**  
   - Copy `api_token` from previous node via expression: `={{ $('Set API Token').item.json.api_token }}`.  
   - Add parameters with these keys and values:  
     - `seed`: -1 (number)  
     - `prompt`: "A beautiful landscape with mountains and a lake at sunset" (string)  
     - `input_image`: "https://picsum.photos/512/512" (string)  
     - `aspect_ratio`: "match_input_image" (string)  
     - `output_format`: "png" (string)  
     - `safety_tolerance`: 2 (number)  
     - `prompt_upsampling`: false (boolean)  
   - Connect Set API Token → Set Image Parameters.

4. **Add HTTP Request node named "Create Image Prediction"**  
   - Method: POST  
   - URL: `https://api.replicate.com/v1/predictions`  
   - Headers:  
     - Authorization: `Bearer {{$json.api_token}}`  
     - Prefer: `wait`  
   - Body (JSON):  
     ```json
     {
       "version": "black-forest-labs/flux-kontext-pro:aa776ca45ce7f7d185418f700df8ec6ca6cb367bfd88e9cd225666c4c179d1d7",
       "input": {
         "seed": {{$json.seed}},
         "prompt": "{{$json.prompt}}",
         "input_image": "{{$json.input_image}}",
         "aspect_ratio": "{{$json.aspect_ratio}}",
         "output_format": "{{$json.output_format}}",
         "safety_tolerance": {{$json.safety_tolerance}},
         "prompt_upsampling": {{$json.prompt_upsampling}}
       }
     }
     ```
   - Send as JSON, expect JSON response, set "Never Error" true.  
   - Connect Set Image Parameters → Create Image Prediction.

5. **Add Code node named "Log Request"**  
   - JavaScript code:  
     ```javascript
     const data = $input.all()[0].json;
     console.log('black-forest-labs/flux-kontext-pro Request:', {
       timestamp: new Date().toISOString(),
       prediction_id: data.id,
       model_type: 'image'
     });
     return $input.all();
     ```  
   - Connect Create Image Prediction → Log Request.

6. **Add Wait node named "Wait 5s"**  
   - Set to wait 5 seconds.  
   - Connect Log Request → Wait 5s.

7. **Add HTTP Request node named "Check Status"**  
   - Method: GET  
   - URL: `https://api.replicate.com/v1/predictions/{{$('Create Image Prediction').item.json.id}}`  
   - Header: Authorization: `Bearer {{$('Set API Token').item.json.api_token}}`  
   - Expect JSON response, "Never Error" true.  
   - Connect Wait 5s → Check Status.  
   - Also connect Wait 10s → Check Status (for retries).

8. **Add If node named "Is Complete?"**  
   - Condition: `{{$json.status}} == 'succeeded'`  
   - Connect Check Status → Is Complete?.

9. **Add Set node named "Success Response"**  
   - Set a field `response` with object:  
     ```json
     {
       "success": true,
       "image_url": {{$json.output}},
       "prediction_id": {{$json.id}},
       "status": {{$json.status}},
       "message": "Image generated successfully"
     }
     ```  
   - Connect Is Complete? True → Success Response.

10. **Add Set node named "Display Result"**  
    - Set `final_result` = `{{$json.response}}` (from previous node).  
    - Connect Success Response → Display Result.

11. **Add If node named "Has Failed?"**  
    - Condition: `{{$json.status}} == 'failed'`  
    - Connect Is Complete? False → Has Failed?.

12. **Add Set node named "Error Response"**  
    - Set a field `response` with object:  
      ```json
      {
        "success": false,
        "error": {{$json.error || 'Image generation failed'}},
        "prediction_id": {{$json.id}},
        "status": {{$json.status}},
        "message": "Failed to generate image"
      }
      ```  
    - Connect Has Failed? True → Error Response.

13. **Connect Error Response → Display Result**  
    So both success and failure responses end at Display Result.

14. **Add Wait node named "Wait 10s"**  
    - Set to wait 10 seconds.  
    - Connect Has Failed? False → Wait 10s.  
    - Connect Wait 10s → Check Status (to retry polling).

15. **Add sticky notes** for:  
    - Contact/support info (Sticky Note9)  
    - Detailed workflow and model documentation (Sticky Note4)  
    Position these at top-left or sidebar for easy reference.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                   | Context or Link                                                                                           |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------|
| For any questions or support, contact: Yaron@nofluff.online. Explore video tutorials here: https://www.youtube.com/@YaronBeen/videos and LinkedIn profile: https://www.linkedin.com/in/yaronbeen/                                                                                                                                                                               | Sticky Note9 support information.                                                                         |
| Model info: black-forest-labs/flux-kontext-pro — state-of-the-art text-to-image editing with excellent prompt following and consistent results. Powered by Replicate API. Parameters include prompt, seed, input_image, aspect_ratio, output_format, safety_tolerance, prompt_upsampling. Detailed parameter guide and troubleshooting included.                                         | Sticky Note4 detailed workflow notes.                                                                     |
| API Documentation: https://replicate.com/docs ; Model Documentation: https://replicate.com/black-forest-labs/flux-kontext-pro ; n8n Documentation: https://docs.n8n.io                                                                                                                                                                                                          | Useful external references for API and workflow understanding.                                            |
| Replace placeholder API token with your actual Replicate API key. Monitor API usage and handle token security carefully. Test with default parameters first to ensure proper setup.                                                                                                                                                                                             | Workflow configuration best practices.                                                                    |
| The workflow includes retry logic with incremental wait times to handle API latency and long-running image generation jobs gracefully. Error handling nodes provide structured failure messages for downstream processing or user feedback.                                                                                                                                     | Robustness and error resilience design notes.                                                             |

---

This document enables users and automation agents to fully understand, reproduce, and maintain the image generation workflow using Flux Kontext Pro via Replicate API in n8n without needing the original JSON export.