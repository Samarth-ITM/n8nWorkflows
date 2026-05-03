Bytedance Seedream 3: Text-to-Image Transformation Template

https://n8nworkflows.xyz/workflows/bytedance-seedream-3--text-to-image-transformation-template-6786


# Bytedance Seedream 3: Text-to-Image Transformation Template

### 1. Workflow Overview

This workflow, titled **Bytedance Seedream 3: Text-to-Image Transformation Template**, automates the generation of high-resolution images from textual prompts using the Bytedance Seedream-3 AI model via the Replicate API. It is designed for users who want to transform descriptive text into detailed images at 2K resolution (2048x2048 pixels).

The workflow includes the following logical blocks:

- **1.1 Input Reception and Initialization**  
  Starts the workflow via manual trigger and sets the required API token and image generation parameters.

- **1.2 Image Generation Request and Logging**  
  Submits the generation request to the Replicate API and logs the request metadata.

- **1.3 Status Polling and Retry Logic**  
  Implements a loop that waits and polls the API to check the image generation status until it is complete or failed, with retry waits on failure.

- **1.4 Success and Error Handling**  
  Processes the final output or error response, formats a structured JSON response, and sets the output for downstream consumption or display.

- **1.5 Documentation and Support Notes**  
  Provides extensive sticky notes with user guidance, parameter explanations, and contact/support information.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Initialization

- **Overview:**  
  This block initiates the workflow execution and configures the authentication token and image generation parameters with default values.

- **Nodes Involved:**  
  - Manual Trigger  
  - Set API Token  
  - Set Image Parameters

- **Node Details:**

  - **Manual Trigger**  
    - *Type:* Manual trigger node that starts the workflow execution manually.  
    - *Configuration:* No parameters; user clicks to start the workflow.  
    - *Connections:* Outputs to "Set API Token".  
    - *Potential Failures:* None expected.  

  - **Set API Token**  
    - *Type:* Set node to define the authentication token for Replicate API.  
    - *Configuration:* Sets a string variable `api_token` with placeholder `"YOUR_REPLICATE_API_TOKEN"`.  
    - *Key Variables:* `api_token` — must be replaced by the user with a valid Replicate API token.  
    - *Inputs:* From "Manual Trigger".  
    - *Outputs:* To "Set Image Parameters".  
    - *Potential Failures:* Using an invalid or missing token will cause authentication errors downstream.  

  - **Set Image Parameters**  
    - *Type:* Set node to define parameters for the image generation request.  
    - *Configuration:*  
      - Copies `api_token` from previous node.  
      - Sets defaults:  
        - `seed`: -1 (random seed)  
        - `size`: "regular"  
        - `width` and `height`: 2048 each (2K resolution)  
        - `prompt`: "A beautiful landscape with mountains and a lake at sunset"  
        - `aspect_ratio`: "16:9"  
        - `guidance_scale`: 2.5 (prompt adherence)  
    - *Inputs:* From "Set API Token".  
    - *Outputs:* To "Create Image Prediction".  
    - *Potential Failures:* Incorrect parameter types or missing required `prompt` will cause API errors.

---

#### 2.2 Image Generation Request and Logging

- **Overview:**  
  Sends an HTTP POST request to Replicate API to create a new image generation prediction and logs the request details.

- **Nodes Involved:**  
  - Create Image Prediction  
  - Log Request

- **Node Details:**

  - **Create Image Prediction**  
    - *Type:* HTTP Request node to POST prediction request to Replicate API.  
    - *Configuration:*  
      - URL: `https://api.replicate.com/v1/predictions`  
      - Method: POST  
      - Headers:  
        - `Authorization: Bearer <api_token>`  
        - `Prefer: wait` (asks API to wait for prediction completion but fallback handling implemented)  
      - Body (JSON): includes version ID and inputs (`seed`, `size`, `width`, `height`, `prompt`, `aspect_ratio`, `guidance_scale`) interpolated from previous node.  
      - Response is expected JSON and errors are suppressed (`neverError: true`).  
    - *Inputs:* From "Set Image Parameters".  
    - *Outputs:* To "Log Request".  
    - *Potential Failures:*  
      - Authentication errors if token invalid.  
      - API rate limiting or quota exceeded errors.  
      - Invalid parameter values or missing required fields.  

  - **Log Request**  
    - *Type:* Code node to log prediction request details for monitoring.  
    - *Configuration:* Logs timestamp, prediction ID, and model type ("image") to console.  
    - *Inputs:* From "Create Image Prediction".  
    - *Outputs:* To "Wait 5s".  
    - *Potential Failures:* Logging itself unlikely to fail; if prediction ID missing, log may be incomplete.

---

#### 2.3 Status Polling and Retry Logic

- **Overview:**  
  Implements a polling loop that waits and checks the status of the image generation prediction until it is completed successfully or failed. On failure, it implements a retry delay.

- **Nodes Involved:**  
  - Wait 5s  
  - Check Status  
  - Is Complete?  
  - Has Failed?  
  - Wait 10s

- **Node Details:**

  - **Wait 5s**  
    - *Type:* Wait node to pause workflow for 5 seconds.  
    - *Configuration:* Wait unit: seconds; amount: 5.  
    - *Inputs:* From "Log Request".  
    - *Outputs:* To "Check Status".  
    - *Potential Failures:* None expected.  

  - **Check Status**  
    - *Type:* HTTP Request node to GET prediction status from Replicate API.  
    - *Configuration:*  
      - URL: `https://api.replicate.com/v1/predictions/<prediction_id>` (prediction ID dynamically from "Create Image Prediction" node)  
      - Method: GET  
      - Header: `Authorization: Bearer <api_token>`  
      - Response expected JSON with `neverError: true`.  
    - *Inputs:* From "Wait 5s" and "Wait 10s" (retry loop).  
    - *Outputs:* To "Is Complete?".  
    - *Potential Failures:*  
      - Authentication errors if token invalid.  
      - Network errors/timeouts.  
      - Missing or invalid prediction ID.  

  - **Is Complete?**  
    - *Type:* If node to check if prediction status equals "succeeded".  
    - *Configuration:* Condition: `$json.status === 'succeeded'`  
    - *Inputs:* From "Check Status".  
    - *Outputs:*  
      - True: To "Success Response"  
      - False: To "Has Failed?"  
    - *Potential Failures:* Expression errors if `status` field missing.  

  - **Has Failed?**  
    - *Type:* If node to check if prediction status equals "failed".  
    - *Configuration:* Condition: `$json.status === 'failed'`  
    - *Inputs:* From "Is Complete?" (false branch).  
    - *Outputs:*  
      - True: To "Error Response"  
      - False: To "Wait 10s" (continue polling)  
    - *Potential Failures:* Expression errors if `status` missing.  

  - **Wait 10s**  
    - *Type:* Wait node delaying 10 seconds before next status check.  
    - *Configuration:* Wait unit: seconds; amount: 10.  
    - *Inputs:* From "Has Failed?" (false branch).  
    - *Outputs:* To "Check Status".  
    - *Potential Failures:* None expected.

---

#### 2.4 Success and Error Handling

- **Overview:**  
  Processes the final result after image generation completes or fails, formats a structured JSON response, and passes it to the output node.

- **Nodes Involved:**  
  - Success Response  
  - Error Response  
  - Display Result

- **Node Details:**

  - **Success Response**  
    - *Type:* Set node to build success JSON response object.  
    - *Configuration:* Creates a `response` object with:  
      - `success: true`  
      - `image_url`: from `$json.output` (image URL array)  
      - `prediction_id` and `status` from API response  
      - `message`: "Image generated successfully"  
    - *Inputs:* From "Is Complete?" (true branch).  
    - *Outputs:* To "Display Result".  
    - *Potential Failures:* If `output` field missing or malformed, image URL may be invalid.  

  - **Error Response**  
    - *Type:* Set node to build error JSON response object.  
    - *Configuration:* Creates a `response` object with:  
      - `success: false`  
      - `error`: from `$json.error` or string "Image generation failed"  
      - `prediction_id` and `status` from API response  
      - `message`: "Failed to generate image"  
    - *Inputs:* From "Has Failed?" (true branch).  
    - *Outputs:* To "Display Result".  
    - *Potential Failures:* `error` field may be empty or missing.  

  - **Display Result**  
    - *Type:* Set node to assign the final `response` object to a variable `final_result`.  
    - *Configuration:* Sets `final_result` to the `response` object passed from either success or error node.  
    - *Inputs:* From "Success Response" and "Error Response".  
    - *Outputs:* None (end of workflow).  
    - *Potential Failures:* None expected.  

---

#### 2.5 Documentation and Support Notes

- **Overview:**  
  Contains detailed sticky notes with instructions, parameter explanations, support contacts, and links to resources.

- **Nodes Involved:**  
  - Sticky Note9  
  - Sticky Note4

- **Node Details:**

  - **Sticky Note9**  
    - *Type:* Sticky Note node with contact information and resource links.  
    - *Content:*  
      - Title and contact email (Yaron@nofluff.online)  
      - YouTube and LinkedIn links for tutorials and support.  
    - *Position:* Top-left corner for visibility.  

  - **Sticky Note4**  
    - *Type:* Large Sticky Note with comprehensive workflow documentation.  
    - *Content:*  
      - Model overview, parameters, workflow components, benefits, quick start, troubleshooting, and resource links.  
      - URLs to Replicate model docs, API docs, and n8n docs.  
    - *Purpose:* Acts as embedded user manual within the workflow editor.

---

### 3. Summary Table

| Node Name            | Node Type        | Functional Role                      | Input Node(s)               | Output Node(s)               | Sticky Note                                                                                                                                                                              |
|----------------------|------------------|------------------------------------|-----------------------------|-----------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Manual Trigger       | Manual Trigger   | Starts workflow execution           |                             | Set API Token                | For any questions or support, please contact: Yaron@nofluff.online. Explore more tips and tutorials on YouTube and LinkedIn (see Sticky Note9).                                        |
| Set API Token        | Set              | Sets Replicate API token            | Manual Trigger              | Set Image Parameters         | Replace 'YOUR_REPLICATE_API_TOKEN' with your actual token. See parameter guidance in Sticky Note4.                                                                                      |
| Set Image Parameters | Set              | Defines input parameters for model  | Set API Token               | Create Image Prediction      | Pre-filled with defaults, editable prompt and image size parameters. See Sticky Note4 for parameter details.                                                                             |
| Create Image Prediction | HTTP Request   | Sends generation request to API     | Set Image Parameters        | Log Request                 | Uses Replicate API endpoint with Bearer token authorization. See Sticky Note4 for API details.                                                                                           |
| Log Request          | Code             | Logs prediction request details     | Create Image Prediction     | Wait 5s                     | Logs timestamp and prediction ID for monitoring.                                                                                                                                         |
| Wait 5s              | Wait             | Waits 5 seconds before status check | Log Request                 | Check Status                | Implements initial delay before polling.                                                                                                                                                 |
| Check Status         | HTTP Request     | Gets current status of prediction   | Wait 5s, Wait 10s           | Is Complete?                | Checks Replicate API prediction status with token auth.                                                                                                                                |
| Is Complete?         | If               | Checks if prediction succeeded      | Check Status                | Success Response, Has Failed? | Routes workflow based on success status.                                                                                                                                                  |
| Has Failed?          | If               | Checks if prediction failed         | Is Complete? (false branch) | Error Response, Wait 10s    | Routes workflow based on failure status or continues polling.                                                                                                                            |
| Wait 10s             | Wait             | Waits 10 seconds for retry delay    | Has Failed? (false branch)  | Check Status                | Implements retry wait delay on pending status.                                                                                                                                           |
| Success Response     | Set              | Formats successful result response  | Is Complete? (true branch) | Display Result              | Outputs structured JSON with image URL and success message.                                                                                                                              |
| Error Response       | Set              | Formats error response              | Has Failed? (true branch)   | Display Result              | Outputs structured JSON with error details and failure message.                                                                                                                          |
| Display Result       | Set              | Sets final result variable           | Success Response, Error Response |                             | Prepares final JSON response for output consumption.                                                                                                                                     |
| Sticky Note9         | Sticky Note      | Contact and support information     |                             |                             | Contains contact email and helpful links.                                                                                                                                                 |
| Sticky Note4         | Sticky Note      | Comprehensive workflow documentation |                             |                             | Detailed model, parameter, and workflow explanation plus support resources and troubleshooting tips.                                                                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: `Manual Trigger`  
   - Purpose: Manually start the workflow.

2. **Create Set Node for API Token**  
   - Name: `Set API Token`  
   - Assign variable: `api_token` (string)  
   - Value: `"YOUR_REPLICATE_API_TOKEN"` (replace with your actual token)  
   - Connect output of `Manual Trigger` to this node.

3. **Create Set Node for Image Parameters**  
   - Name: `Set Image Parameters`  
   - Assign variables:  
     - `api_token`: copy from previous node via expression  
     - `seed`: `-1` (number)  
     - `size`: `"regular"` (string)  
     - `width`: `2048` (number)  
     - `height`: `2048` (number)  
     - `prompt`: `"A beautiful landscape with mountains and a lake at sunset"` (string)  
     - `aspect_ratio`: `"16:9"` (string)  
     - `guidance_scale`: `2.5` (number)  
   - Connect output of `Set API Token` to this node.

4. **Create HTTP Request Node to Create Prediction**  
   - Name: `Create Image Prediction`  
   - HTTP Method: POST  
   - URL: `https://api.replicate.com/v1/predictions`  
   - Headers:  
     - `Authorization`: `Bearer {{ $json.api_token }}`  
     - `Prefer`: `wait`  
   - Body Type: JSON  
   - Body Content:  
     ```json
     {
       "version": "bytedance/seedream-3:e97385a576173b08a6a87546457582b01f65bf29a4dc00f1191e884894e0bc73",
       "input": {
         "seed": {{ $json.seed }},
         "size": "{{ $json.size }}",
         "width": {{ $json.width }},
         "height": {{ $json.height }},
         "prompt": "{{ $json.prompt }}",
         "aspect_ratio": "{{ $json.aspect_ratio }}",
         "guidance_scale": {{ $json.guidance_scale }}
       }
     }
     ```  
   - Response Format: JSON, with `neverError` enabled to avoid workflow failure on HTTP error.  
   - Connect output of `Set Image Parameters` to this node.

5. **Create Code Node for Logging**  
   - Name: `Log Request`  
   - Code:  
     ```javascript
     const data = $input.all()[0].json;
     console.log('bytedance/seedream-3 Request:', {
       timestamp: new Date().toISOString(),
       prediction_id: data.id,
       model_type: 'image'
     });
     return $input.all();
     ```  
   - Connect output of `Create Image Prediction` to this node.

6. **Create Wait Node for 5 Seconds**  
   - Name: `Wait 5s`  
   - Wait for: 5 seconds  
   - Connect output of `Log Request` to this node.

7. **Create HTTP Request Node to Check Status**  
   - Name: `Check Status`  
   - HTTP Method: GET  
   - URL: `https://api.replicate.com/v1/predictions/{{ $('Create Image Prediction').item.json.id }}`  
   - Header:  
     - `Authorization`: `Bearer {{ $('Set API Token').item.json.api_token }}`  
   - Response Format: JSON with `neverError` enabled.  
   - Connect output of `Wait 5s` to this node.

8. **Create If Node to Check Completion**  
   - Name: `Is Complete?`  
   - Condition: Check if `$json.status === "succeeded"`  
   - Connect output of `Check Status` to this node.

9. **Create If Node to Check Failure**  
   - Name: `Has Failed?`  
   - Condition: Check if `$json.status === "failed"`  
   - Connect "false" output of `Is Complete?` to this node.

10. **Create Wait Node for 10 Seconds (Retry Delay)**  
    - Name: `Wait 10s`  
    - Wait for: 10 seconds  
    - Connect "false" output of `Has Failed?` to this node.

11. **Connect Retry Loop**  
    - Connect output of `Wait 10s` back to `Check Status`.

12. **Create Set Node for Success Response**  
    - Name: `Success Response`  
    - Assign variable `response` (object):  
      ```json
      {
        "success": true,
        "image_url": $json.output,
        "prediction_id": $json.id,
        "status": $json.status,
        "message": "Image generated successfully"
      }
      ```  
    - Connect "true" output of `Is Complete?` to this node.

13. **Create Set Node for Error Response**  
    - Name: `Error Response`  
    - Assign variable `response` (object):  
      ```json
      {
        "success": false,
        "error": $json.error || "Image generation failed",
        "prediction_id": $json.id,
        "status": $json.status,
        "message": "Failed to generate image"
      }
      ```  
    - Connect "true" output of `Has Failed?` to this node.

14. **Create Set Node to Display Final Result**  
    - Name: `Display Result`  
    - Assign variable `final_result` to `$json.response`  
    - Connect outputs of both `Success Response` and `Error Response` to this node.

15. **Add Sticky Notes**  
    - Create two sticky note nodes with content as per the original workflow:  
      - Sticky Note9: Contact and support info.  
      - Sticky Note4: Full documentation, parameter guide, troubleshooting, and resource links.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                   | Context or Link                                      |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| SEEDREAM-3 GENERATOR contact: Yaron@nofluff.online. Video tutorials and tips available on YouTube: https://www.youtube.com/@YaronBeen/videos and LinkedIn: https://www.linkedin.com/in/yaronbeen/                              | Support contact and educational resources           |
| Model documentation and API references: https://replicate.com/bytedance/seedream-3 and https://replicate.com/docs. n8n documentation: https://docs.n8n.io                                                                | Official docs for model, API, and automation setup  |
| The workflow supports high-resolution (2K) text-to-image generation with retry logic and error handling to ensure robustness in production environments.                                                                       | Workflow design note                                 |
| Ensure to replace the placeholder API token with a valid Replicate API token to avoid authentication failures.                                                                                                                | Security and authentication reminder                 |
| The workflow uses "Prefer: wait" header but also implements polling to handle longer generation times gracefully.                                                                                                             | API usage best practice                               |
| Recommended to monitor API usage and billing as high-res image generation can consume significant API credits.                                                                                                                | Operational recommendation                           |

---

**Disclaimer:**  
The provided text is derived exclusively from an automated workflow created with n8n, respecting all current content policies and containing no illegal, offensive, or protected material. All manipulated data is legal and public.