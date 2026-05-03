Transform Hairstyles in Photos with Flux Kontext Apps AI via Replicate API

https://n8nworkflows.xyz/workflows/transform-hairstyles-in-photos-with-flux-kontext-apps-ai-via-replicate-api-6874


# Transform Hairstyles in Photos with Flux Kontext Apps AI via Replicate API

### 1. Workflow Overview

This n8n workflow automates the process of transforming hairstyles in photos by leveraging the Flux Kontext Apps AI model "change-haircut" via the Replicate API. It is designed to accept an input image, apply AI-driven hairstyle and hair color modifications, and return a generated image URL or error message.

The workflow logically divides into the following blocks:

- **1.1 Input Reception & Parameter Setup:** Accepts manual trigger to start the workflow, sets API authentication token and image transformation parameters.
- **1.2 Image Generation Request:** Sends a structured prediction request to the Replicate API to generate the hairstyle transformation.
- **1.3 Status Polling Loop:** Waits and repeatedly checks the generation status until completion or failure.
- **1.4 Result Handling:** Routes the flow to success or error responses depending on prediction outcome.
- **1.5 Logging & Monitoring:** Records request details for auditing and troubleshooting.
- **1.6 Final Output Preparation:** Formats and outputs the final response object containing result URL or error.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Parameter Setup

**Overview:**  
This block initiates the workflow manually, sets the Replicate API token, and configures all input parameters required for the hairstyle transformation model.

**Nodes Involved:**  
- Manual Trigger  
- Set API Token  
- Set Image Parameters  

**Node Details:**

- **Manual Trigger**  
  - Type: Manual trigger node  
  - Role: Starts workflow execution on user command  
  - Configuration: Default, no parameters  
  - Inputs: None  
  - Outputs: Connects to "Set API Token"  
  - Edge Cases: User forgetting to trigger; no automated start mechanism

- **Set API Token**  
  - Type: Set node  
  - Role: Stores the Replicate API token as a workflow variable  
  - Configuration: Sets variable `api_token` with placeholder "YOUR_REPLICATE_API_TOKEN" (to be replaced by user)  
  - Input: From Manual Trigger  
  - Output: Connects to "Set Image Parameters"  
  - Edge Cases: Missing or invalid token will cause authentication errors downstream

- **Set Image Parameters**  
  - Type: Set node  
  - Role: Defines model input parameters with default values  
  - Configuration:  
    - Copies `api_token` from previous node  
    - Sets parameters: `seed` (-1), `gender` ("none"), `haircut` ("No change"), `hair_color` ("No change"), `input_image` (sample image URL), `aspect_ratio` ("match_input_image"), `output_format` ("png"), `safety_tolerance` (2)  
  - Input: From Set API Token  
  - Output: Connects to "Create Image Prediction"  
  - Edge Cases: Invalid or unsupported parameter values may cause API errors

---

#### 2.2 Image Generation Request

**Overview:**  
Sends a POST request to the Replicate API to initiate the hairstyle transformation prediction, passing all configured parameters and receiving a prediction ID for tracking.

**Nodes Involved:**  
- Create Image Prediction  

**Node Details:**

- **Create Image Prediction**  
  - Type: HTTP Request node  
  - Role: Calls Replicate API endpoint to create a prediction  
  - Configuration:  
    - URL: `https://api.replicate.com/v1/predictions`  
    - Method: POST  
    - Headers: Authorization Bearer token (dynamic), Prefer: wait  
    - Body: JSON containing model version ID and input parameters from previous node  
    - Response handling: JSON, never error to continue flow regardless of HTTP errors  
  - Input: From Set Image Parameters  
  - Output: Connects to "Log Request"  
  - Edge Cases: HTTP errors, invalid token, malformed parameters, API rate limits

---

#### 2.3 Status Polling Loop

**Overview:**  
Implements an intelligent polling mechanism by waiting, then checking the prediction status repeatedly until the job is completed or failed.

**Nodes Involved:**  
- Log Request  
- Wait 5s  
- Check Status  
- Is Complete?  
- Has Failed?  
- Wait 10s  

**Node Details:**

- **Log Request**  
  - Type: Code node  
  - Role: Logs prediction request details (timestamp, prediction ID, model type) for monitoring  
  - Configuration: JavaScript code logs to console and passes data through  
  - Input: From Create Image Prediction  
  - Output: Connects to Wait 5s  
  - Edge Cases: Logging failure does not stop workflow

- **Wait 5s**  
  - Type: Wait node  
  - Role: Introduces a 5 second delay before checking status  
  - Configuration: Wait for 5 seconds  
  - Input: From Log Request  
  - Output: Connects to Check Status  
  - Edge Cases: Delay too short or too long may affect responsiveness

- **Check Status**  
  - Type: HTTP Request node  
  - Role: Queries Replicate API for prediction status using prediction ID  
  - Configuration:  
    - URL dynamically constructed with prediction ID from Create Image Prediction  
    - Authorization Bearer token from Set API Token  
    - Response JSON expected  
  - Input: From Wait 5s and Wait 10s (retry)  
  - Output: Connects to Is Complete?  
  - Edge Cases: Network issues, token expiration, API downtime

- **Is Complete?**  
  - Type: If node  
  - Role: Checks if prediction status is "succeeded"  
  - Configuration: Condition: `$json.status == "succeeded"`  
  - Input: From Check Status  
  - Outputs:  
      - True: Connects to Success Response  
      - False: Connects to Has Failed?  
  - Edge Cases: Unexpected status values, expression failures

- **Has Failed?**  
  - Type: If node  
  - Role: Checks if prediction status is "failed"  
  - Configuration: Condition: `$json.status == "failed"`  
  - Input: From Is Complete? (False branch)  
  - Outputs:  
      - True: Connects to Error Response  
      - False: Connects to Wait 10s (retry)  
  - Edge Cases: Status other than succeeded/failed (e.g. "processing"), loop continues

- **Wait 10s**  
  - Type: Wait node  
  - Role: Delays 10 seconds before retrying status check if prediction not yet complete or failed  
  - Configuration: Wait 10 seconds  
  - Input: From Has Failed? (False branch)  
  - Output: Connects back to Check Status  
  - Edge Cases: Potential infinite loop if prediction never completes or fails

---

#### 2.4 Result Handling

**Overview:**  
Based on prediction outcome, prepares structured JSON responses indicating success or failure, including relevant data such as generated image URL or error details.

**Nodes Involved:**  
- Success Response  
- Error Response  
- Display Result  

**Node Details:**

- **Success Response**  
  - Type: Set node  
  - Role: Constructs success response object with data: success flag, image output URL, prediction ID, status, and message  
  - Configuration: Sets `response` JSON object  
  - Input: From Is Complete? (True branch)  
  - Output: Connects to Display Result  
  - Edge Cases: Missing output URL in prediction response

- **Error Response**  
  - Type: Set node  
  - Role: Constructs error response object with success false, error message, prediction ID, status, and message  
  - Configuration: Sets `response` JSON object, uses available error info or generic message  
  - Input: From Has Failed? (True branch)  
  - Output: Connects to Display Result  
  - Edge Cases: Missing error details in API response

- **Display Result**  
  - Type: Set node  
  - Role: Finalizes output by assigning the response object to `final_result` for downstream use or output  
  - Configuration: Sets `final_result` to the response from previous node  
  - Input: From Success Response or Error Response  
  - Output: None (end of flow)  
  - Edge Cases: None

---

#### 2.5 Logging & Monitoring

**Overview:**  
Logs prediction requests for tracking and debugging; integrated within the polling loop after prediction creation.

**Nodes Involved:**  
- Log Request (already detailed in 2.3)

---

#### 2.6 Sticky Notes & Documentation

**Overview:**  
Provides embedded documentation, usage instructions, parameter descriptions, contact info, and troubleshooting tips in visual sticky notes inside the workflow for user guidance.

**Nodes Involved:**  
- Sticky Note9  
- Sticky Note4  

**Node Details:**

- **Sticky Note9**  
  - Type: Sticky Note  
  - Role: Displays contact info and links for support and tutorials  
  - Content: Contact email and YouTube/LinkedIn links for author Yaron Been

- **Sticky Note4**  
  - Type: Sticky Note  
  - Role: Comprehensive documentation of model info, parameter guide, workflow explanation, benefits, setup instructions, troubleshooting, and resource links  
  - Content: Detailed markdown text with sections on model overview, parameters, workflow components, quick start, troubleshooting, and external links

---

### 3. Summary Table

| Node Name           | Node Type         | Functional Role                      | Input Node(s)                | Output Node(s)           | Sticky Note                                                                                                                          |
|---------------------|-------------------|-----------------------------------|-----------------------------|--------------------------|--------------------------------------------------------------------------------------------------------------------------------------|
| Manual Trigger      | Manual Trigger    | Starts workflow execution          | None                        | Set API Token             | See Sticky Note4 for full workflow documentation and Sticky Note9 for support contacts and resources                                  |
| Set API Token       | Set               | Stores Replicate API token         | Manual Trigger              | Set Image Parameters      | See Sticky Note4                                                                                                                     |
| Set Image Parameters| Set               | Sets all image generation parameters| Set API Token              | Create Image Prediction   | See Sticky Note4                                                                                                                     |
| Create Image Prediction | HTTP Request  | Sends image generation request     | Set Image Parameters        | Log Request               | See Sticky Note4                                                                                                                     |
| Log Request         | Code              | Logs prediction details            | Create Image Prediction     | Wait 5s                   | See Sticky Note4                                                                                                                     |
| Wait 5s             | Wait              | Delays 5 seconds before status check| Log Request               | Check Status              | See Sticky Note4                                                                                                                     |
| Check Status        | HTTP Request      | Queries prediction status          | Wait 5s, Wait 10s           | Is Complete?              | See Sticky Note4                                                                                                                     |
| Is Complete?        | If                | Checks if prediction succeeded     | Check Status                | Success Response, Has Failed? | See Sticky Note4                                                                                                                  |
| Has Failed?         | If                | Checks if prediction failed        | Is Complete?                | Error Response, Wait 10s  | See Sticky Note4                                                                                                                     |
| Wait 10s            | Wait              | Delay before retrying status check | Has Failed?                 | Check Status              | See Sticky Note4                                                                                                                     |
| Success Response    | Set               | Builds success JSON response       | Is Complete? (true)         | Display Result            | See Sticky Note4                                                                                                                     |
| Error Response      | Set               | Builds error JSON response         | Has Failed? (true)          | Display Result            | See Sticky Note4                                                                                                                     |
| Display Result      | Set               | Outputs final response              | Success Response, Error Response | None                  | See Sticky Note4                                                                                                                     |
| Sticky Note9        | Sticky Note       | Contact info and support links     | None                        | None                     | Contact: Yaron@nofluff.online; YouTube: https://www.youtube.com/@YaronBeen/videos; LinkedIn: https://www.linkedin.com/in/yaronbeen/ |
| Sticky Note4        | Sticky Note       | Full workflow documentation        | None                        | None                     | Comprehensive workflow and model documentation with setup and troubleshooting guide                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Node Type: Manual Trigger  
   - Purpose: Entry point to start the workflow manually  
   - No parameters needed

2. **Create Set Node ("Set API Token")**  
   - Node Type: Set  
   - Add a string variable `api_token` with value `"YOUR_REPLICATE_API_TOKEN"` (replace with your actual Replicate API token)  
   - Connect Manual Trigger output to this node

3. **Create Set Node ("Set Image Parameters")**  
   - Node Type: Set  
   - Assign variables:  
     - `api_token`: copy from previous node (`={{ $('Set API Token').item.json.api_token }}`)  
     - `seed`: -1 (integer)  
     - `gender`: "none" (string)  
     - `haircut`: "No change" (string)  
     - `hair_color`: "No change" (string)  
     - `input_image`: "https://picsum.photos/512/512" (string)  
     - `aspect_ratio`: "match_input_image" (string)  
     - `output_format`: "png" (string)  
     - `safety_tolerance`: 2 (integer)  
   - Connect "Set API Token" output to this node

4. **Create HTTP Request Node ("Create Image Prediction")**  
   - Node Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.replicate.com/v1/predictions`  
   - Headers:  
     - Authorization: `Bearer {{ $json.api_token }}`  
     - Prefer: `wait`  
   - Body (JSON):  
     ```json
     {
       "version": "flux-kontext-apps/change-haircut:735c13ba40448758e00e6bb2d764624508bed5a07bbb6a5f33e41482fed8ac88",
       "input": {
         "seed": {{ $json.seed }},
         "gender": "{{ $json.gender }}",
         "haircut": "{{ $json.haircut }}",
         "hair_color": "{{ $json.hair_color }}",
         "input_image": "{{ $json.input_image }}",
         "aspect_ratio": "{{ $json.aspect_ratio }}",
         "output_format": "{{ $json.output_format }}",
         "safety_tolerance": {{ $json.safety_tolerance }}
       }
     }
     ```  
   - Enable JSON body and send headers  
   - Connect "Set Image Parameters" output to this node

5. **Create Code Node ("Log Request")**  
   - Node Type: Code  
   - JavaScript code:  
     ```js
     const data = $input.all()[0].json;
     console.log('flux-kontext-apps/change-haircut Request:', {
       timestamp: new Date().toISOString(),
       prediction_id: data.id,
       model_type: 'image'
     });
     return $input.all();
     ```  
   - Connect "Create Image Prediction" output to this node

6. **Create Wait Node ("Wait 5s")**  
   - Node Type: Wait  
   - Wait time: 5 seconds  
   - Connect "Log Request" output to this node

7. **Create HTTP Request Node ("Check Status")**  
   - Node Type: HTTP Request  
   - Method: GET  
   - URL: `https://api.replicate.com/v1/predictions/{{ $('Create Image Prediction').item.json.id }}`  
   - Headers: Authorization: `Bearer {{ $('Set API Token').item.json.api_token }}`  
   - Connect "Wait 5s" output to this node

8. **Create If Node ("Is Complete?")**  
   - Node Type: If  
   - Condition: Check if `$json.status === "succeeded"`  
   - Connect "Check Status" output to this node

9. **Create Set Node ("Success Response")**  
   - Node Type: Set  
   - Assign variable `response` with object:  
     ```js
     {
       success: true,
       image_url: $json.output,
       prediction_id: $json.id,
       status: $json.status,
       message: "Image generated successfully"
     }
     ```  
   - Connect "Is Complete?" true output to this node

10. **Create Set Node ("Display Result")**  
    - Node Type: Set  
    - Assign variable `final_result` equal to the previous node's `response`  
    - Connect "Success Response" output to this node

11. **Create If Node ("Has Failed?")**  
    - Node Type: If  
    - Condition: Check if `$json.status === "failed"`  
    - Connect "Is Complete?" false output to this node

12. **Create Set Node ("Error Response")**  
    - Node Type: Set  
    - Assign variable `response` with object:  
      ```js
      {
        success: false,
        error: $json.error || "Image generation failed",
        prediction_id: $json.id,
        status: $json.status,
        message: "Failed to generate image"
      }
      ```  
    - Connect "Has Failed?" true output to this node

13. **Connect "Error Response" output to "Display Result"**

14. **Create Wait Node ("Wait 10s")**  
    - Node Type: Wait  
    - Wait time: 10 seconds  
    - Connect "Has Failed?" false output to this node

15. **Connect "Wait 10s" output back to "Check Status" to create polling loop**

16. **Verify all connections and parameter expressions for proper dynamic referencing**

17. **Test workflow with valid Replicate API token and accessible input image URL**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                           | Context or Link                                                                                     |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Workflow powered by Replicate API and n8n automation integrating the flux-kontext-apps/change-haircut AI model for image hairstyle transformation.                                                                                                                   | Model page: https://replicate.com/flux-kontext-apps/change-haircut                                |
| Contact for support: Yaron@nofluff.online. Video tutorials and additional tips available on YouTube channel https://www.youtube.com/@YaronBeen/videos and LinkedIn https://www.linkedin.com/in/yaronbeen/                                                                 | Support contacts and social media                                                                  |
| Recommended to replace the placeholder API token with a valid Replicate token before running. Monitor API usage and billing through Replicate platform.                                                                                                              | https://replicate.com/account                                                                     |
| Model parameters include required input_image and optional seed, gender, haircut, hair_color, aspect_ratio, output_format, and safety_tolerance with sensible defaults for ease of use.                                                                                | Parameter guide in Sticky Note4                                                                    |
| Workflow includes robust error handling and retry logic with status polling to accommodate asynchronous image generation times and potential API delays.                                                                                                              | Workflow design notes                                                                              |
| For troubleshooting, ensure correct token, parameter validation, and monitor logs for prediction IDs and timestamps.                                                                                                                                                | Troubleshooting section in Sticky Note4                                                           |
| n8n documentation valuable for understanding node functions and workflow design: https://docs.n8n.io                                                                                                                                                                  | n8n official docs                                                                                  |

---

**Disclaimer:** The text above is exclusively derived from an automated n8n workflow utilizing legal and publicly available data and content. It complies with all current content policies and contains no illegal, offensive, or protected material.