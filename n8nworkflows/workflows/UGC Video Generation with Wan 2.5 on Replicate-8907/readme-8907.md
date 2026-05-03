UGC Video Generation with Wan 2.5 on Replicate

https://n8nworkflows.xyz/workflows/ugc-video-generation-with-wan-2-5-on-replicate-8907


# UGC Video Generation with Wan 2.5 on Replicate

### 1. Workflow Overview

This n8n workflow automates the generation of dynamic videos from static images using the "WAN-VIDEO 2.5" AI model hosted on Replicate. It is targeted at marketers, content creators, and social media managers seeking to transform product photos or illustrations into engaging promotional videos with minimal manual effort.

The workflow is logically organized into the following blocks:

- **1.1 Input Initialization:** Starts the process manually and sets essential parameters such as the Replicate API token, input image URL, and descriptive video prompt.
- **1.2 Video Creation Request:** Sends a generation request to the Replicate API for the WAN-VIDEO 2.5 model, initiating video synthesis.
- **1.3 Status Polling Loop:** Waits and repeatedly checks the prediction status until the video is successfully generated or a failure occurs.
- **1.4 Response Handling:** Routes results to success or error response nodes, formats outputs, and prepares the final result.
- **1.5 Logging and Monitoring:** Logs metadata about the requests for monitoring and debugging purposes.
- **1.6 Optional Batch Processing (Not fully connected):** Supports batch operations for generating multiple videos with different images or prompts.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Initialization

**Overview:**  
This block receives manual trigger input, assigns the Replicate API token, and sets the seed image URL along with the video prompt describing the desired animation.

**Nodes Involved:**  
- Manual Trigger  
- Set API Token  
- Add Seed Image and Prompt  
- Sticky Notes (guidance and instructions)

**Node Details:**  

- **Manual Trigger**  
  - Type: Trigger node  
  - Role: Initiates workflow execution manually  
  - Configuration: Default trigger, no parameters  
  - Connections: Output → Set API Token  
  - Failures: None expected

- **Set API Token**  
  - Type: Set node  
  - Role: Sets a string variable `api_token` for Replicate API authentication  
  - Configuration: Hardcoded placeholder `"=YOUR_REPLICATE_API"`; user must replace with valid token  
  - Connections: Input → Manual Trigger; Output → Add Seed Image and Prompt  
  - Failures: Missing or invalid API token leads to authentication errors downstream  

- **Add Seed Image and Prompt**  
  - Type: Set node  
  - Role: Defines two key parameters for video generation:  
    - `Seed_image`: URL of the input image to animate  
    - `Prompt`: A detailed textual description of desired video motion and scene setup  
  - Configuration: Pre-populated with a marketing video example (AG1 product) including exact timing and camera directions  
  - Connections: Input → Set API Token; Output → Create Video  
  - Expressions: Uses JSON properties to assign values to variables  
  - Failures: Invalid or inaccessible image URLs, improper prompt syntax (e.g., disallowed characters) can cause API rejection  

- **Sticky Notes (Input Guidance)**  
  - Provide instructions on API token setup, image URL requirements, and prompt writing guidelines (avoid double quotes, backslashes, line breaks)  
  - Contextual help to prevent common user errors

---

#### 2.2 Video Creation Request

**Overview:**  
Sends a POST request to Replicate’s WAN-VIDEO 2.5 model endpoint with the image and prompt to start video generation.

**Nodes Involved:**  
- Create Video  
- Sticky Note (API token reminder)

**Node Details:**  

- **Create Video**  
  - Type: HTTP Request  
  - Role: Calls the Replicate API to create a new video prediction job  
  - Configuration:  
    - URL: `https://api.replicate.com/v1/models/wan-video/wan-2.5-i2v/predictions`  
    - Method: POST  
    - Headers:  
      - Authorization: Bearer token from `Set API Token` node  
      - Prefer: wait (instructs API to wait for response but workflow manages polling anyway)  
    - Body: JSON including `input` object with `image` and `prompt` fields from `Add Seed Image and Prompt` node  
    - Response handling: JSON, never error mode (to prevent node failure on HTTP errors)  
  - Connections: Input → Add Seed Image and Prompt; Output → Log Request  
  - Failures: Auth errors, network timeouts, invalid payload errors, rate limits  

- **Sticky Note (API token reminder)**  
  - Content: “Add Your Replicate API”  
  - Purpose: Reminds user to configure API token  

---

#### 2.3 Status Polling Loop

**Overview:**  
Waits a short fixed interval and queries the Replicate API to check if the video generation prediction has completed or failed. If pending, loops with delays until resolved.

**Nodes Involved:**  
- Log Request  
- Wait 5s  
- Check Status  
- Is Complete? (If node)  
- Has Failed? (If node)  
- Wait 90s (retry delay in failure branch)  

**Node Details:**  

- **Log Request**  
  - Type: Code node  
  - Role: Logs prediction metadata for monitoring  
  - Configuration: JavaScript code logs timestamp, prediction ID, and model type to console for debugging  
  - Input: Output from Create Video (prediction JSON)  
  - Output: Passes data unchanged to Wait 5s  
  - Failures: Possible runtime errors if input data missing fields  

- **Wait 5s**  
  - Type: Wait node  
  - Role: Delays execution for 5 seconds before checking status to allow generation progress  
  - Configuration: 5 seconds delay  
  - Input: Log Request output  
  - Output: Check Status  
  - Failures: None expected, but network or system delays possible  

- **Check Status**  
  - Type: HTTP Request  
  - Role: Queries Replicate prediction endpoint by prediction ID to get current status and outputs  
  - Configuration:  
    - URL: `https://api.replicate.com/v1/predictions/{{ prediction_id }}` dynamically filled from "Create Video" node result  
    - Method: GET  
    - Authorization header with Bearer token  
    - JSON response, never error mode  
  - Input: Wait 5s output  
  - Output: Is Complete?  
  - Failures: Auth errors, network failures, invalid IDs  

- **Is Complete?**  
  - Type: If node  
  - Role: Checks if status equals "succeeded" indicating video ready  
  - Configuration: Checks `$json.status === "succeeded"`  
  - Input: Check Status  
  - Output:  
    - True branch → Success Response  
    - False branch → Has Failed?  

- **Has Failed?**  
  - Type: If node  
  - Role: Checks if status equals "failed" indicating generation failure  
  - Configuration: Checks `$json.status === "failed"`  
  - Input: Is Complete? false branch  
  - Output:  
    - True branch → Error Response  
    - False branch → Wait 90s (retry)  

- **Wait 90s**  
  - Type: Wait node  
  - Role: Longer delay before retrying status check to avoid excessive polling on failure or slow processing  
  - Configuration: 90 seconds delay  
  - Input: Has Failed? false branch  
  - Output: Loops back to Check Status  
  - Failures: None expected  

---

#### 2.4 Response Handling

**Overview:**  
Prepares structured JSON outputs for success or failure cases and consolidates final results.

**Nodes Involved:**  
- Success Response  
- Error Response  
- Display Result  
- Sticky Note (save predictions)

**Node Details:**  

- **Success Response**  
  - Type: Set node  
  - Role: Formats a success response object including:  
    - `success: true`  
    - `result_url`: URL from prediction output (video URL)  
    - `prediction_id`, `status`, and a message  
  - Input: Is Complete? true branch  
  - Output: Display Result  
  - Failures: Missing output fields cause incomplete response  

- **Error Response**  
  - Type: Set node  
  - Role: Formats an error response object including:  
    - `success: false`  
    - `error`: error text from JSON or default message  
    - `prediction_id`, `status`, and failure message  
  - Input: Has Failed? true branch  
  - Output: Display Result  
  - Failures: Missing error data could reduce clarity  

- **Display Result**  
  - Type: Set node  
  - Role: Assigns the final combined response to a variable `final_result` for downstream use or output  
  - Input: Success Response or Error Response  
  - Output: None (end node)  
  - Failures: None expected  

- **Sticky Note (save predictions)**  
  - Content: “Connect this node to your storage in order to save the predictions”  
  - Suggests extending workflow by attaching storage nodes for persistence  

---

#### 2.5 Logging and Monitoring

**Overview:**  
Logs each request with timestamp and prediction ID for monitoring and debugging.

**Nodes Involved:**  
- Log Request  

**Node Details:**  
See details in 2.3 Status Polling Loop section.

---

#### 2.6 Optional Batch Processing (Unconnected in this JSON)

**Overview:**  
Supports batch processing of multiple videos by looping over items, enabling use cases like generating multiple videos from varied images and prompts.

**Nodes Involved:**  
- Loop Over Items  
- Sticky Note (Batch Processing Guide)  

**Node Details:**  

- **Loop Over Items**  
  - Type: SplitInBatches  
  - Role: Processes multiple input items in batches (unconnected in this exported workflow)  
  - Configuration: Default batch size  
  - Potential use: For bulk video generation scenarios  
  - Failures: Unconnected, no impact in current workflow  

- **Sticky Note (Batch Processing Guide)**  
  - Provides detailed usage scenarios and options for batch workflows, including same image/different prompts, multiple images, etc.

---

### 3. Summary Table

| Node Name                | Node Type           | Functional Role                        | Input Node(s)             | Output Node(s)           | Sticky Note                                                                                       |
|--------------------------|---------------------|-------------------------------------|---------------------------|--------------------------|-------------------------------------------------------------------------------------------------|
| Manual Trigger           | Trigger             | Starts workflow manually             |                           | Set API Token             |                                                                                                 |
| Set API Token            | Set                 | Sets Replicate API token             | Manual Trigger            | Add Seed Image and Prompt | "Add Your Replicate API"                                                                         |
| Add Seed Image and Prompt| Set                 | Defines input image URL and prompt   | Set API Token             | Create Video              | "1- Add a link to your Seed image. If you dont have one, you can generate it using OpenAI or Nano Banana" <br> "PROMPT WRITING GUIDELINES: Avoid double quotes, backslashes, line breaks" |
| Create Video             | HTTP Request        | Sends video generation request       | Add Seed Image and Prompt | Log Request               |                                                                                                 |
| Log Request              | Code                | Logs prediction metadata             | Create Video              | Wait 5s                   |                                                                                                 |
| Wait 5s                  | Wait                | Delay before status check            | Log Request               | Check Status              |                                                                                                 |
| Check Status             | HTTP Request        | Checks prediction completion status  | Wait 5s                   | Is Complete?              |                                                                                                 |
| Is Complete?             | If                  | Checks if prediction succeeded       | Check Status              | Success Response, Has Failed? |                                                                                                 |
| Has Failed?              | If                  | Checks if prediction failed          | Is Complete? (false branch) | Error Response, Wait 90s |                                                                                                 |
| Wait 90s                 | Wait                | Delay before retrying status check   | Has Failed? (false branch) | Check Status              |                                                                                                 |
| Success Response         | Set                 | Formats success output                | Is Complete? (true branch) | Display Result           |                                                                                                 |
| Error Response           | Set                 | Formats error output                  | Has Failed? (true branch)  | Display Result            |                                                                                                 |
| Display Result           | Set                 | Assigns final response object        | Success Response, Error Response |                      | "Connect this node to your storage in order to save the predictions"                            |
| Loop Over Items          | SplitInBatches      | Batch processing (unconnected)       |                           |                          | "Batch Processing Guide: Options for multiple videos with same/different images and prompts"     |
| Sticky Notes (multiple)  | Sticky Note         | Provide guidance, instructions       |                           |                          | See detailed contents in section 2 above                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a Manual Trigger node to start the workflow.

2. **Add Set Node for API Token**  
   - Create a Set node named "Set API Token".  
   - Add a string variable `api_token` with value `"=YOUR_REPLICATE_API"`.  
   - Connect Manual Trigger output to this node.  
   - Replace `"YOUR_REPLICATE_API"` with your actual Replicate API token.

3. **Add Set Node for Seed Image and Prompt**  
   - Add another Set node named "Add Seed Image and Prompt".  
   - Define two string fields:  
     - `Seed_image`: set to a publicly accessible image URL (e.g., `"https://replicate.delivery/pbxt/NllgibhEDDtjRmPSLSDvXnkiobQCJDeoI4PwWdJ0bdXIeZbP/AG1_1.png"`)  
     - `Prompt`: set a detailed description of the video animation (avoid double quotes, backslashes, line breaks). Example prompt provided in original workflow.  
   - Connect "Set API Token" output to this node.

4. **Create HTTP Request Node for Video Creation**  
   - Add HTTP Request node called "Create Video".  
   - Set method to POST.  
   - URL: `https://api.replicate.com/v1/models/wan-video/wan-2.5-i2v/predictions`  
   - Headers:  
     - Authorization: `Bearer {{ $json.api_token }}` (fetch from "Set API Token" node)  
     - Prefer: `wait`  
   - Body type: JSON  
   - Body content:  
     ```json
     {
       "input": {
         "image": "{{ $json.Seed_image }}",
         "prompt": "{{ $json.Prompt }}"
       }
     }
     ```  
   - Set response to JSON, enable "never error" mode to avoid node failure on API errors.  
   - Connect "Add Seed Image and Prompt" output to this node.

5. **Add Code Node for Logging**  
   - Create a Code node named "Log Request".  
   - JavaScript code:  
     ```javascript
     const data = $input.all()[0].json;
     console.log('digitalhera/heranathalie Request:', {
       timestamp: new Date().toISOString(),
       prediction_id: data.id,
       model_type: 'other'
     });
     return $input.all();
     ```  
   - Connect "Create Video" output to this node.

6. **Add Wait Node for 5 seconds**  
   - Add a Wait node named "Wait 5s".  
   - Set delay to 5 seconds.  
   - Connect "Log Request" output to this node.

7. **Add HTTP Request Node for Status Check**  
   - Add HTTP Request node named "Check Status".  
   - Method: GET  
   - URL: `https://api.replicate.com/v1/predictions/{{ $json.id }}`  
   - Header: Authorization `Bearer {{ $json.api_token }}` (pass from "Set API Token")  
   - Response: JSON, "never error" enabled  
   - Connect "Wait 5s" output to this node.

8. **Add If Node "Is Complete?"**  
   - Add If node named "Is Complete?"  
   - Condition: `$json.status === "succeeded"` (string equals)  
   - Connect "Check Status" output to this node.

9. **Add If Node "Has Failed?"**  
   - Add If node named "Has Failed?"  
   - Condition: `$json.status === "failed"`  
   - Connect "Is Complete?" false branch to this node.

10. **Add Wait Node for 90 seconds**  
    - Add Wait node named "Wait 90s".  
    - Delay: 90 seconds.  
    - Connect "Has Failed?" false branch to this node.  
    - Connect "Wait 90s" output back to "Check Status" to create retry loop.

11. **Add Set Node "Success Response"**  
    - Add Set node named "Success Response".  
    - Create a JSON object variable `response` with:  
      ```json
      {
        "success": true,
        "result_url": $json.output,
        "prediction_id": $json.id,
        "status": $json.status,
        "message": "Other generated successfully"
      }
      ```  
    - Connect "Is Complete?" true branch to this node.

12. **Add Set Node "Error Response"**  
    - Add Set node named "Error Response".  
    - Create JSON object variable `response` with:  
      ```json
      {
        "success": false,
        "error": $json.error || "Other generation failed",
        "prediction_id": $json.id,
        "status": $json.status,
        "message": "Failed to generate other"
      }
      ```  
    - Connect "Has Failed?" true branch to this node.

13. **Add Set Node "Display Result"**  
    - Add Set node named "Display Result".  
    - Create variable `final_result` assigned to `$json.response` (incoming `response` object).  
    - Connect outputs from "Success Response" and "Error Response" to this node.

14. **Optional: Add Storage Node(s)**  
    - To persist prediction results, connect "Display Result" output to storage nodes (e.g., Google Sheets, database, or file storage).

15. **Add Sticky Notes**  
    - Add sticky notes at strategic positions with instructions about API token setup, prompt writing, batch processing, and saving results.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| IMAGE TO VIDEO GENERATOR - For support, contact Yaron@nofluff.online. Explore tutorials at YouTube: https://www.youtube.com/@YaronBeen/videos and LinkedIn: https://www.linkedin.com/in/yaronbeen/ | Sticky Note covering overall project branding and support |
| Model Documentation: https://replicate.com/wan-video/wan-2.5-i2v | Official model API docs for WAN-VIDEO 2.5 |
| Replicate API Docs: https://replicate.com/docs | General API documentation for authentication and usage |
| n8n Documentation: https://docs.n8n.io | Official n8n platform documentation |
| Prompt Writing Guidelines: Avoid double quotes, backslashes, and line breaks to prevent JSON parsing errors. | Sticky Note with prompt formatting instructions |
| Batch Processing Guide: Options for generating multiple videos with same/different images and prompts for bulk automation. | Sticky Note describing batch options (node present but unconnected) |

---

This document fully describes the workflow structure, node configurations, dependencies, and usage considerations. It supports recreating, modifying, and troubleshooting the video generation workflow using n8n and the Replicate API.