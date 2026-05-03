Create Realistic Portrait Images using CyberRealistic Pony v36 and Replicate

https://n8nworkflows.xyz/workflows/create-realistic-portrait-images-using-cyberrealistic-pony-v36-and-replicate-6853


# Create Realistic Portrait Images using CyberRealistic Pony v36 and Replicate

### 1. Workflow Overview

This n8n workflow automates the creation of realistic portrait images using the AI model **CyberRealistic Pony v36** via the **Replicate API**. It is designed for users seeking to generate high-quality, fashion-style portraits with detailed and cinematic effects by leveraging advanced AI image generation.

**Target Use Cases:**  
- Automated AI portrait generation for fashion, editorial, and creative projects  
- Batch or on-demand generation triggered manually or integrated into larger pipelines  
- Reliable status tracking and error handling for asynchronous AI processing  

**Logical Blocks:**

- **1.1 Input Reception and Initialization**  
  Manual trigger to start, setting API authentication and default parameters.

- **1.2 AI Model Prediction Request**  
  Submitting the generation request to the Replicate API with configured parameters.

- **1.3 Prediction Status Polling and Control Loop**  
  Waiting and repeatedly checking the status of the prediction until success or failure.

- **1.4 Result Handling and Response Construction**  
  Handling success or failure outcomes, formatting structured JSON responses.

- **1.5 Logging and Monitoring**  
  Logging request details for monitoring and debugging.

- **1.6 User Guidance and Documentation**  
  Sticky notes provide detailed instructions, parameter explanations, and troubleshooting.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Initialization

**Overview:**  
Starts the workflow via manual trigger and sets up necessary authentication and generation parameters.

**Nodes Involved:**  
- Manual Trigger  
- Set API Token  
- Set Other Parameters

**Node Details:**

- **Manual Trigger**  
  - Type: Trigger node  
  - Role: Initiates workflow execution manually.  
  - Configuration: No parameters; click to start the process.  
  - Input: None  
  - Output: Connected to "Set API Token"  
  - Edge Cases: None (manual start)  

- **Set API Token**  
  - Type: Set node  
  - Role: Stores the Replicate API token as a string variable `api_token`.  
  - Configuration: Hardcoded string placeholder `"YOUR_REPLICATE_API_TOKEN"`. User must replace with valid token.  
  - Input: From Manual Trigger  
  - Output: To "Set Other Parameters"  
  - Edge Cases: Missing or invalid token will cause API auth failures downstream.  
  - Notes: Secure handling of token recommended; avoid committing token in public repos.

- **Set Other Parameters**  
  - Type: Set node  
  - Role: Defines all model input parameters including prompt, image size, steps, etc.  
  - Configuration:  
    - Copies `api_token` from previous node  
    - Sets model-specific parameters with defaults:  
      - `cfg`: 4  
      - `seed`: 0 (random)  
      - `steps`: 40  
      - `width`: 768  
      - `height`: 1152  
      - `prompt`: Detailed fashion portrait description with positive keywords  
      - `denoise`: 0.98  
      - `scheduler`: "karras"  
      - `facerestore`: true  
      - `sampler_name`: "dpmpp_3m_sde"  
      - `negative_prompt`: Keywords to filter out low quality or errors  
      - `codeformer_fidelity`: 0.4  
  - Input: From "Set API Token"  
  - Output: To "Create Other Prediction"  
  - Edge Cases: Invalid parameter types or values may cause API request rejection.

---

#### 2.2 AI Model Prediction Request

**Overview:**  
Sends the generation request to the Replicate API and receives a prediction ID for asynchronous tracking.

**Nodes Involved:**  
- Create Other Prediction  
- Log Request

**Node Details:**

- **Create Other Prediction**  
  - Type: HTTP Request node  
  - Role: Posts generation request to Replicate API endpoint `/v1/predictions`  
  - Configuration:  
    - Method: POST  
    - URL: `https://api.replicate.com/v1/predictions`  
    - Headers: Authorization Bearer token from `api_token`, Prefer: wait (wait for immediate response)  
    - Body: JSON with model version ID and input parameters from "Set Other Parameters"  
    - Response: JSON parsed, never error on response (to allow handling)  
  - Input: From "Set Other Parameters"  
  - Output: To "Log Request"  
  - Edge Cases:  
    - API auth failure if token invalid  
    - Parameter validation errors from Replicate API  
    - Network issues causing request failures  

- **Log Request**  
  - Type: Code node  
  - Role: Logs prediction request metadata to console for monitoring  
  - Configuration: Custom JavaScript logging timestamp, prediction ID, and model type  
  - Input: From "Create Other Prediction"  
  - Output: To "Wait 5s"  
  - Edge Cases: Logging failure does not affect workflow execution; only diagnostic  

---

#### 2.3 Prediction Status Polling and Control Loop

**Overview:**  
Waits 5 seconds after request, then repeatedly checks prediction status until it completes or fails. Implements retry logic with 10-second waits on failure.

**Nodes Involved:**  
- Wait 5s  
- Check Status  
- Is Complete?  
- Has Failed?  
- Wait 10s  

**Node Details:**

- **Wait 5s**  
  - Type: Wait node  
  - Role: Pauses workflow 5 seconds before first status check  
  - Configuration: 5 seconds delay  
  - Input: From "Log Request"  
  - Output: To "Check Status"  
  - Edge Cases: None  

- **Check Status**  
  - Type: HTTP Request node  
  - Role: GET request to Replicate API to retrieve prediction status by ID  
  - Configuration:  
    - Method: GET  
    - URL: Dynamic `https://api.replicate.com/v1/predictions/{{prediction_id}}`  
    - Headers: Authorization Bearer token  
    - Response: JSON parsed, never error on response  
  - Input: From "Wait 5s" and "Wait 10s" (loop)  
  - Output: To "Is Complete?"  
  - Edge Cases:  
    - Network or auth errors  
    - Invalid prediction ID causing 404 errors  

- **Is Complete?**  
  - Type: If node  
  - Role: Checks if prediction status equals "succeeded"  
  - Configuration: Condition `$json.status == "succeeded"`  
  - Input: From "Check Status"  
  - Output:  
    - True: "Success Response"  
    - False: "Has Failed?"  
  - Edge Cases: Status field missing or unexpected values  

- **Has Failed?**  
  - Type: If node  
  - Role: Checks if prediction status equals "failed"  
  - Configuration: Condition `$json.status == "failed"`  
  - Input: From "Is Complete?" (False branch)  
  - Output:  
    - True: "Error Response"  
    - False: "Wait 10s" (retry loop)  
  - Edge Cases: Possible infinite loop if status stuck in unknown state  

- **Wait 10s**  
  - Type: Wait node  
  - Role: Pauses workflow 10 seconds before retrying status check  
  - Configuration: 10 seconds delay  
  - Input: From "Has Failed?" (False branch)  
  - Output: To "Check Status" (loop)  
  - Edge Cases: None  

---

#### 2.4 Result Handling and Response Construction

**Overview:**  
Formats structured responses for both successful and failed predictions, returning result URLs or error messages.

**Nodes Involved:**  
- Success Response  
- Error Response  
- Display Result

**Node Details:**

- **Success Response**  
  - Type: Set node  
  - Role: Creates JSON object with success status and relevant output fields  
  - Configuration:  
    - Sets `response` object with:  
      - `success: true`  
      - `result_url`: Image output URL from prediction output  
      - `prediction_id`, `status`, and message string  
  - Input: From "Is Complete?" (True branch)  
  - Output: To "Display Result"  
  - Edge Cases: Missing or malformed output URLs  

- **Error Response**  
  - Type: Set node  
  - Role: Creates JSON object with failure status and error details  
  - Configuration:  
    - Sets `response` object with:  
      - `success: false`  
      - `error`: from API error or default message  
      - `prediction_id`, `status`, and message string  
  - Input: From "Has Failed?" (True branch)  
  - Output: To "Display Result"  
  - Edge Cases: Missing error details  

- **Display Result**  
  - Type: Set node  
  - Role: Final output node that holds the JSON response for downstream use or output  
  - Configuration: Assigns `final_result` to the `response` object created by Success or Error Response  
  - Input: From both Success Response and Error Response  
  - Output: None (end of workflow)  
  - Edge Cases: None  

---

#### 2.5 Logging and Monitoring

**Overview:**  
Logs prediction request metadata for debugging and operational monitoring.

**Nodes Involved:**  
- Log Request (already described in 2.2)

---

#### 2.6 User Guidance and Documentation

**Overview:**  
Sticky notes provide detailed instructions, parameter explanations, troubleshooting tips, and contact information.

**Nodes Involved:**  
- Sticky Note9  
- Sticky Note4

**Node Details:**

- **Sticky Note9**  
  - Content: Contact info for support, Yaron@nofluff.online, plus YouTube and LinkedIn links  
  - Role: User support and branding  
  - Position: Top-left corner  

- **Sticky Note4**  
  - Content:  
    - Complete documentation of model usage and parameters  
    - Workflow explanation and benefits  
    - Quick start instructions  
    - Troubleshooting guide  
    - External links to Replicate and n8n docs  
  - Role: Comprehensive user guide embedded in workflow canvas  

---

### 3. Summary Table

| Node Name            | Node Type           | Functional Role                        | Input Node(s)             | Output Node(s)               | Sticky Note                                                                                                                                                                                                                                                              |
|----------------------|---------------------|-------------------------------------|---------------------------|-----------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Manual Trigger       | Manual Trigger      | Workflow start trigger               | None                      | Set API Token                |                                                                                                                                                                                                                                                                            |
| Set API Token        | Set                 | Sets Replicate API token             | Manual Trigger            | Set Other Parameters         |                                                                                                                                                                                                                                                                            |
| Set Other Parameters | Set                 | Defines all model input parameters   | Set API Token             | Create Other Prediction      |                                                                                                                                                                                                                                                                            |
| Create Other Prediction | HTTP Request       | Sends generation request to Replicate API | Set Other Parameters      | Log Request                 |                                                                                                                                                                                                                                                                            |
| Log Request          | Code                | Logs prediction details for monitoring | Create Other Prediction   | Wait 5s                     |                                                                                                                                                                                                                                                                            |
| Wait 5s              | Wait                | Initial wait before status check     | Log Request               | Check Status                 |                                                                                                                                                                                                                                                                            |
| Check Status         | HTTP Request        | Polls prediction status              | Wait 5s, Wait 10s         | Is Complete?                 |                                                                                                                                                                                                                                                                            |
| Is Complete?         | If                  | Checks if prediction succeeded       | Check Status              | Success Response (True), Has Failed? (False) |                                                                                                                                                                                                                                                                            |
| Success Response     | Set                 | Formats success response JSON        | Is Complete? (True)       | Display Result              |                                                                                                                                                                                                                                                                            |
| Has Failed?          | If                  | Checks if prediction failed           | Is Complete? (False)      | Error Response (True), Wait 10s (False) |                                                                                                                                                                                                                                                                            |
| Error Response       | Set                 | Formats error response JSON          | Has Failed? (True)        | Display Result              |                                                                                                                                                                                                                                                                            |
| Wait 10s             | Wait                | Wait before retrying status check    | Has Failed? (False)       | Check Status                |                                                                                                                                                                                                                                                                            |
| Display Result       | Set                 | Holds final JSON response             | Success Response, Error Response | None                      |                                                                                                                                                                                                                                                                            |
| Sticky Note9         | Sticky Note         | Support contact and external links  | None                      | None                        | =======================================<br>CYBERREALISTIC-PONY-SEMIREAL-V36 GENERATOR<br>For any questions or support, please contact:<br>Yaron@nofluff.online<br>Explore more tips and tutorials here:<br>- YouTube: https://www.youtube.com/@YaronBeen/videos<br>- LinkedIn: https://www.linkedin.com/in/yaronbeen/ |
| Sticky Note4         | Sticky Note         | Full workflow documentation and guidance | None                      | None                        | **0XDINO/CYBERREALISTIC-PONY-SEMIREAL-V36 - OTHER GENERATION WORKFLOW**<br>Powered by Replicate API and n8n Automation<br>Model details, parameter reference, workflow explanation, troubleshooting, and links to documentation and external resources.                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "Manual Trigger" node:**  
   - Type: Manual Trigger  
   - Purpose: Start workflow manually  
   - No additional configuration needed  

2. **Create "Set API Token" node:**  
   - Type: Set  
   - Add string field `api_token`  
   - Value: `"YOUR_REPLICATE_API_TOKEN"` (replace with your actual token)  
   - Connect "Manual Trigger" ➔ "Set API Token"  

3. **Create "Set Other Parameters" node:**  
   - Type: Set  
   - Copy `api_token` from "Set API Token" using expression: `={{ $('Set API Token').item.json.api_token }}`  
   - Add numerical and string fields with these default values:  
     - `cfg`: 4  
     - `seed`: 0  
     - `steps`: 40  
     - `width`: 768  
     - `height`: 1152  
     - `prompt`: `"score_9, score_8_up, score_7_up, super-detailed fashion portrait of a young woman in ripped denim shorts and ribbed tank top, colorful accessories, RAW photography style, soft cinematic lighting, dramatic shadows across her face and body, brown hair gently tousled, (fine-art editorial atmosphere), moody tone, high-resolution textures and rich natural detail, solo subject"`  
     - `denoise`: 0.98  
     - `scheduler`: `"karras"`  
     - `facerestore`: true (boolean)  
     - `sampler_name`: `"dpmpp_3m_sde"`  
     - `negative_prompt`: `"score 6, score 5, score 4, (worst quality:1.2), (low quality:1.2), (normal quality:1.2), lowres, bad anatomy, bad hands, signature, watermarks, ugly, imperfect eyes, skewed eyes, unnatural face, unnatural body, error, extra limb, missing limbs"`  
     - `codeformer_fidelity`: 0.4  
   - Connect "Set API Token" ➔ "Set Other Parameters"  

4. **Create "Create Other Prediction" node:**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.replicate.com/v1/predictions`  
   - Headers:  
     - Authorization: `Bearer {{ $json.api_token }}`  
     - Prefer: `wait`  
   - Body (JSON):  
   ```json
   {
     "version": "0xdino/cyberrealistic-pony-semireal-v36:37b3641b63dde3759bd28e8c3e43a6466993ac95f3eead034cabd36506f2749c",
     "input": {
       "cfg": {{ $json.cfg }},
       "seed": {{ $json.seed }},
       "steps": {{ $json.steps }},
       "width": {{ $json.width }},
       "height": {{ $json.height }},
       "prompt": "{{ $json.prompt }}",
       "denoise": {{ $json.denoise }},
       "scheduler": "{{ $json.scheduler }}",
       "facerestore": {{ $json.facerestore }},
       "sampler_name": "{{ $json.sampler_name }}",
       "negative_prompt": "{{ $json.negative_prompt }}",
       "codeformer_fidelity": {{ $json.codeformer_fidelity }}
     }
   }
   ```  
   - Enable JSON body and parse JSON response  
   - Connect "Set Other Parameters" ➔ "Create Other Prediction"  

5. **Create "Log Request" node:**  
   - Type: Code  
   - JavaScript code:  
   ```javascript
   const data = $input.all()[0].json;
   console.log('0xdino/cyberrealistic-pony-semireal-v36 Request:', {
     timestamp: new Date().toISOString(),
     prediction_id: data.id,
     model_type: 'other'
   });
   return $input.all();
   ```  
   - Connect "Create Other Prediction" ➔ "Log Request"  

6. **Create "Wait 5s" node:**  
   - Type: Wait  
   - Wait for 5 seconds  
   - Connect "Log Request" ➔ "Wait 5s"  

7. **Create "Check Status" node:**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://api.replicate.com/v1/predictions/{{ $('Create Other Prediction').item.json.id }}` (dynamic)  
   - Header Authorization: `Bearer {{ $('Set API Token').item.json.api_token }}`  
   - Parse JSON response, never fail on error  
   - Connect "Wait 5s" ➔ "Check Status" and "Wait 10s" ➔ "Check Status" (loop)  

8. **Create "Is Complete?" node:**  
   - Type: If  
   - Condition: `$json.status == "succeeded"`  
   - Connect "Check Status" ➔ "Is Complete?"  

9. **Create "Success Response" node:**  
   - Type: Set  
   - Set field `response` (object):  
     ```json
     {
       "success": true,
       "result_url": $json.output,
       "prediction_id": $json.id,
       "status": $json.status,
       "message": "Other generated successfully"
     }
     ```  
   - Connect "Is Complete?" (true) ➔ "Success Response"  

10. **Create "Has Failed?" node:**  
    - Type: If  
    - Condition: `$json.status == "failed"`  
    - Connect "Is Complete?" (false) ➔ "Has Failed?"  

11. **Create "Error Response" node:**  
    - Type: Set  
    - Set field `response` (object):  
      ```json
      {
        "success": false,
        "error": $json.error || "Other generation failed",
        "prediction_id": $json.id,
        "status": $json.status,
        "message": "Failed to generate other"
      }
      ```  
    - Connect "Has Failed?" (true) ➔ "Error Response"  

12. **Create "Wait 10s" node:**  
    - Type: Wait  
    - Wait for 10 seconds  
    - Connect "Has Failed?" (false) ➔ "Wait 10s"  

13. **Create "Display Result" node:**  
    - Type: Set  
    - Set field `final_result` to `={{ $json.response }}`  
    - Connect both "Success Response" and "Error Response" ➔ "Display Result"  

14. **Verify connections complete the loop properly:**  
    - "Wait 10s" ➔ "Check Status" (to continue polling)  

15. **Add Sticky Notes (optional but recommended):**  
    - Add detailed instructions, parameter descriptions, troubleshooting tips as per original workflow content  
    - Include contact info and external documentation links  

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| Contact Yaron at Yaron@nofluff.online for support and questions. | Support email (Sticky Note9) |
| YouTube tutorials and additional tips: https://www.youtube.com/@YaronBeen/videos | Video resource (Sticky Note9) |
| LinkedIn profile for updates: https://www.linkedin.com/in/yaronbeen/ | Professional networking (Sticky Note9) |
| Model documentation: https://replicate.com/0xdino/cyberrealistic-pony-semireal-v36 | Official model page |
| Replicate API documentation: https://replicate.com/docs | API reference |
| n8n official documentation: https://docs.n8n.io | Workflow automation platform docs |
| Key advice: Replace placeholder API token before use, monitor API usage and billing, test with defaults first. | Best practices (Sticky Note4) |

---

**Disclaimer:**  
The provided text is exclusively derived from an n8n automated workflow. It strictly complies with current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and publicly accessible.