Creating Images from Text Prompts with Flux Kontext Max on Replicate

https://n8nworkflows.xyz/workflows/creating-images-from-text-prompts-with-flux-kontext-max-on-replicate-6873


# Creating Images from Text Prompts with Flux Kontext Max on Replicate

### 1. Workflow Overview

This workflow automates the generation of images from text prompts using the Flux Kontext Max AI model via the Replicate API. It is designed for users who want to transform natural language descriptions into high-quality images with configurable parameters such as seed, input image, aspect ratio, and output format.

The workflow is logically divided into the following blocks:

- **1.1 Input Initialization:** Starts the workflow and sets up authentication and generation parameters.
- **1.2 Image Prediction Request:** Sends the image generation request to Replicate and initiates the prediction process.
- **1.3 Prediction Status Polling:** Waits and periodically checks the status of the image generation until completion or failure.
- **1.4 Response Handling:** Differentiates between success and failure, prepares structured responses accordingly.
- **1.5 Logging and Final Output:** Logs request details for monitoring and prepares the final output for consumption.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Initialization

**Overview:**  
This block starts the workflow execution manually and sets essential parameters including the Replicate API token and the input parameters for image generation.

**Nodes Involved:**  
- Manual Trigger  
- Set API Token  
- Set Image Parameters  

**Node Details:**

- **Manual Trigger**  
  - *Type:* Trigger node to manually start the workflow.  
  - *Configuration:* No parameters; user clicks to start the process.  
  - *Input/Output:* No input; outputs trigger to next node.  
  - *Failure Types:* None expected.  

- **Set API Token**  
  - *Type:* Set node to define the Replicate API token.  
  - *Configuration:* Contains a string variable `api_token` initialized to `"YOUR_REPLICATE_API_TOKEN"`. Users must replace this placeholder with their actual API token.  
  - *Input/Output:* Receives trigger from manual trigger; outputs `api_token` to next node.  
  - *Edge Cases:* Failure if token is invalid or missing.  

- **Set Image Parameters**  
  - *Type:* Set node to define all parameters needed for image generation.  
  - *Configuration:* Sets variables:  
    - `api_token` (passed from previous node)  
    - `seed` (default -1 meaning random)  
    - `prompt` (default example: "A beautiful landscape with mountains and a lake at sunset")  
    - `input_image` (default URL to a placeholder image)  
    - `aspect_ratio` (default "match_input_image")  
    - `output_format` (default "png")  
    - `safety_tolerance` (default 2, max allowed)  
    - `prompt_upsampling` (default false)  
  - *Input/Output:* Receives `api_token` from previous node; outputs all parameters to the prediction request node.  
  - *Edge Cases:* Incorrect parameter types or missing required `prompt` cause errors downstream.  

---

#### 2.2 Image Prediction Request

**Overview:**  
This block sends a POST request to Replicate API to create an image generation prediction based on the parameters defined earlier.

**Nodes Involved:**  
- Create Image Prediction  
- Log Request  
- Wait 5s  

**Node Details:**

- **Create Image Prediction**  
  - *Type:* HTTP Request node for POST method.  
  - *Configuration:*  
    - URL: `https://api.replicate.com/v1/predictions`  
    - Headers: Authorization Bearer token using `api_token` variable; `Prefer: wait` to wait for initial processing.  
    - Body: JSON including model version ID and all input parameters dynamically interpolated.  
    - Response: JSON, with error suppression enabled (neverError: true).  
  - *Input/Output:* Receives parameters from "Set Image Parameters"; outputs prediction response JSON (including prediction ID).  
  - *Edge Cases:* API authorization failure, network timeout, response errors from API.  

- **Log Request**  
  - *Type:* Code node (JavaScript) for logging.  
  - *Configuration:* Logs timestamp, prediction ID, and model type to console for monitoring/debugging.  
  - *Input/Output:* Receives prediction response; passes data forward.  
  - *Edge Cases:* Logging failures do not break workflow but may reduce observability.  

- **Wait 5s**  
  - *Type:* Wait node to pause execution for 5 seconds before status check.  
  - *Configuration:* Wait time fixed at 5 seconds.  
  - *Input/Output:* Receives data from Log Request; outputs to status check node.  
  - *Edge Cases:* None significant.  

---

#### 2.3 Prediction Status Polling

**Overview:**  
This block polls the Replicate API periodically to check if the image generation prediction has completed successfully, failed, or is still processing. It uses conditional logic to handle each case and includes retry delays.

**Nodes Involved:**  
- Check Status  
- Is Complete?  
- Has Failed?  
- Wait 10s  

**Node Details:**

- **Check Status**  
  - *Type:* HTTP Request node (GET) to fetch prediction status.  
  - *Configuration:*  
    - URL dynamically constructed using prediction ID from "Create Image Prediction" node.  
    - Authorization header with API token.  
    - Response JSON with error suppression enabled.  
  - *Input/Output:* Receives from "Wait 5s" or "Wait 10s"; outputs current prediction status JSON.  
  - *Edge Cases:* Network issues, invalid prediction ID, auth failure.  

- **Is Complete?**  
  - *Type:* If node to check if prediction status equals `"succeeded"`.  
  - *Configuration:* Compares `status` field in JSON response.  
  - *Input/Output:* Receives from "Check Status".  
    - If true: outputs to "Success Response".  
    - If false: outputs to "Has Failed?".  
  - *Edge Cases:* Unexpected status values may cause logic errors.  

- **Has Failed?**  
  - *Type:* If node to check if prediction status equals `"failed"`.  
  - *Configuration:* Compares `status` field.  
  - *Input/Output:* Receives from "Is Complete?" false branch.  
    - If true: outputs to "Error Response".  
    - If false: outputs to "Wait 10s" for retry.  
  - *Edge Cases:* Intermediate statuses like "processing" require looping.  

- **Wait 10s**  
  - *Type:* Wait node to delay 10 seconds before retrying status check.  
  - *Configuration:* Fixed 10-second wait.  
  - *Input/Output:* Receives from "Has Failed?" false branch; outputs back to "Check Status" for next poll.  
  - *Edge Cases:* Extended delays may be needed for slow predictions.  

---

#### 2.4 Response Handling

**Overview:**  
This block builds the final structured JSON responses for successful or failed image generation attempts, preparing data for downstream consumption.

**Nodes Involved:**  
- Success Response  
- Error Response  
- Display Result  

**Node Details:**

- **Success Response**  
  - *Type:* Set node to create a success response object.  
  - *Configuration:* Sets an object with keys:  
    - `success: true`  
    - `image_url`: URL from prediction output  
    - `prediction_id` and `status` from JSON  
    - `message`: "Image generated successfully"  
  - *Input/Output:* Receives from "Is Complete?" true branch; outputs to "Display Result".  
  - *Edge Cases:* Missing output URL would cause incomplete response.  

- **Error Response**  
  - *Type:* Set node to create an error response object.  
  - *Configuration:* Sets an object with keys:  
    - `success: false`  
    - `error`: from JSON error field or generic message  
    - `prediction_id` and `status`  
    - `message`: "Failed to generate image"  
  - *Input/Output:* Receives from "Has Failed?" true branch; outputs to "Display Result".  
  - *Edge Cases:* Missing error details handled by fallback message.  

- **Display Result**  
  - *Type:* Set node to assign the final response object to `final_result` for output.  
  - *Configuration:* Passes through the structured response from either Success or Error node.  
  - *Input/Output:* Receives from either "Success Response" or "Error Response".  
  - *Edge Cases:* None.  

---

#### 2.5 Logging and Documentation

**Overview:**  
This block includes sticky notes with detailed documentation, usage instructions, parameter definitions, and support contacts, aiding users and maintainers.

**Nodes Involved:**  
- Sticky Note4 (Documentation & Instructions)  
- Sticky Note9 (Contact & Resources)  

**Node Details:**

- **Sticky Note4**  
  - *Content:* Extensive documentation about the workflow, model parameters, usage instructions, troubleshooting, and links to resources.  
  - *Role:* Provides contextual help and operational guidance within the workflow editor.  

- **Sticky Note9**  
  - *Content:* Contact email for support and links to YouTube and LinkedIn channels for tutorials and tips.  
  - *Role:* Support and community engagement information.  

---

### 3. Summary Table

| Node Name            | Node Type            | Functional Role                       | Input Node(s)           | Output Node(s)          | Sticky Note                                                                                   |
|----------------------|----------------------|------------------------------------|------------------------|-------------------------|----------------------------------------------------------------------------------------------|
| Manual Trigger       | Trigger              | Start workflow manually              | —                      | Set API Token            |                                                                                              |
| Set API Token        | Set                  | Define Replicate API token           | Manual Trigger          | Set Image Parameters     |                                                                                              |
| Set Image Parameters | Set                  | Configure image generation parameters| Set API Token           | Create Image Prediction  |                                                                                              |
| Create Image Prediction | HTTP Request       | Send image generation request to Replicate API | Set Image Parameters   | Log Request              |                                                                                              |
| Log Request          | Code                 | Log prediction details for monitoring| Create Image Prediction | Wait 5s                  |                                                                                              |
| Wait 5s              | Wait                 | Wait before checking status          | Log Request             | Check Status             |                                                                                              |
| Check Status         | HTTP Request         | Poll prediction status from API      | Wait 5s, Wait 10s       | Is Complete?             |                                                                                              |
| Is Complete?         | If                   | Check if prediction succeeded        | Check Status            | Success Response, Has Failed? |                                                                                              |
| Has Failed?          | If                   | Check if prediction failed           | Is Complete?            | Error Response, Wait 10s |                                                                                              |
| Wait 10s             | Wait                 | Delay before retrying status check   | Has Failed?             | Check Status             |                                                                                              |
| Success Response     | Set                  | Build success response object        | Is Complete?            | Display Result           |                                                                                              |
| Error Response       | Set                  | Build error response object          | Has Failed?             | Display Result           |                                                                                              |
| Display Result       | Set                  | Output final response payload        | Success Response, Error Response | —                  |                                                                                              |
| Sticky Note4         | Sticky Note          | Documentation and instructions       | —                       | —                       | Extensive workflow & model documentation with usage, parameters, and troubleshooting guide. |
| Sticky Note9         | Sticky Note          | Support contacts and resource links  | —                       | —                       | Contact info and links to YouTube and LinkedIn for tips and tutorials.                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node:**  
   - No parameters; used to start the workflow manually.

2. **Add a Set node named "Set API Token":**  
   - Add string variable `api_token` with value `"YOUR_REPLICATE_API_TOKEN"`.  
   - Connect Manual Trigger's output to this node.

3. **Add a Set node named "Set Image Parameters":**  
   - Add the following variables:  
     - `api_token`: Set value to expression referencing `Set API Token` node’s `api_token`.  
     - `seed`: Number, default `-1` (random seed).  
     - `prompt`: String, e.g., `"A beautiful landscape with mountains and a lake at sunset"`.  
     - `input_image`: String URL, e.g., `"https://picsum.photos/512/512"`.  
     - `aspect_ratio`: String, `"match_input_image"`.  
     - `output_format`: String, `"png"`.  
     - `safety_tolerance`: Number, `2`.  
     - `prompt_upsampling`: Boolean, `false`.  
   - Connect "Set API Token" node output to this node.

4. **Add an HTTP Request node named "Create Image Prediction":**  
   - Method: POST  
   - URL: `https://api.replicate.com/v1/predictions`  
   - Headers:  
     - `Authorization`: `Bearer {{ $json.api_token }}` (dynamic from input)  
     - `Prefer`: `wait`  
   - Body (JSON):  
     ```json
     {
       "version": "black-forest-labs/flux-kontext-max:f72e27297d9c05a36b7fd8faff393d31e3b368543e0bc44bde521886700e166c",
       "input": {
         "seed": {{ $json.seed }},
         "prompt": "{{ $json.prompt }}",
         "input_image": "{{ $json.input_image }}",
         "aspect_ratio": "{{ $json.aspect_ratio }}",
         "output_format": "{{ $json.output_format }}",
         "safety_tolerance": {{ $json.safety_tolerance }},
         "prompt_upsampling": {{ $json.prompt_upsampling }}
       }
     }
     ```  
   - Connect "Set Image Parameters" node output to this node.

5. **Add a Code node named "Log Request":**  
   - JavaScript code to log timestamp, prediction ID, and model type:  
     ```js
     const data = $input.all()[0].json;
     console.log('black-forest-labs/flux-kontext-max Request:', {
       timestamp: new Date().toISOString(),
       prediction_id: data.id,
       model_type: 'image'
     });
     return $input.all();
     ```  
   - Connect "Create Image Prediction" output to this node.

6. **Add a Wait node named "Wait 5s":**  
   - Wait time: 5 seconds  
   - Connect "Log Request" output to this node.

7. **Add an HTTP Request node named "Check Status":**  
   - Method: GET  
   - URL: `https://api.replicate.com/v1/predictions/{{ $('Create Image Prediction').item.json.id }}` (dynamic from prediction)  
   - Header: `Authorization: Bearer {{ $('Set API Token').item.json.api_token }}`  
   - Connect "Wait 5s" output to this node.

8. **Add an If node named "Is Complete?":**  
   - Condition: Check if `status` field equals `"succeeded"` (case sensitive false).  
   - Connect "Check Status" output to this node.

9. **Add an If node named "Has Failed?":**  
   - Condition: Check if `status` field equals `"failed"`.  
   - Connect "Is Complete?" false output to this node.

10. **Add a Set node named "Success Response":**  
    - Assign `response` object:  
      ```json
      {
        "success": true,
        "image_url": $json.output,
        "prediction_id": $json.id,
        "status": $json.status,
        "message": "Image generated successfully"
      }
      ```  
    - Connect "Is Complete?" true output to this node.

11. **Add a Set node named "Error Response":**  
    - Assign `response` object:  
      ```json
      {
        "success": false,
        "error": $json.error || "Image generation failed",
        "prediction_id": $json.id,
        "status": $json.status,
        "message": "Failed to generate image"
      }
      ```  
    - Connect "Has Failed?" true output to this node.

12. **Add a Wait node named "Wait 10s":**  
    - Wait time: 10 seconds  
    - Connect "Has Failed?" false output to this node.

13. **Connect "Wait 10s" output back to "Check Status"** to implement retry loop.

14. **Add a Set node named "Display Result":**  
    - Assign `final_result` to the `response` object from either "Success Response" or "Error Response".  
    - Connect outputs of "Success Response" and "Error Response" to this node.

15. **(Optional) Add Sticky Note nodes for documentation and support:**  
    - Include detailed documentation about parameters, usage, troubleshooting, and contact info.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | Context or Link                                                                                                      |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------|
| =======================================<br>FLUX-KONTEXT-MAX GENERATOR<br>=======================================<br>For any questions or support, please contact:<br>Yaron@nofluff.online<br>Explore more tips and tutorials here:<br>- YouTube: https://www.youtube.com/@YaronBeen/videos<br>- LinkedIn: https://www.linkedin.com/in/yaronbeen/<br>======================================= | Contact info and video/link resources for user support and extended learning                                         |
| BLACK-FOREST-LABS/FLUX-KONTEXT-MAX IMAGE GENERATION WORKFLOW powered by Replicate API and n8n.<br>Model owner: black-forest-labs<br>Model type: Image Generation<br>API endpoint: https://api.replicate.com/v1/predictions<br>Required parameter: prompt<br>Optional: seed, input_image, aspect_ratio, output_format, safety_tolerance, prompt_upsampling.<br>Default safe values: safety_tolerance=2, output_format=png.<br>Instructions for API token setup and workflow running included.<br>Troubleshooting includes token validity, parameter validation, timeout monitoring.<br>Links:<br>- Model Docs: https://replicate.com/black-forest-labs/flux-kontext-max<br>- Replicate API Docs: https://replicate.com/docs<br>- n8n Docs: https://docs.n8n.io | Embedded workflow documentation with parameter explanations, usage instructions, and troubleshooting tips.            |

---

**Disclaimer:**  
The provided text is extracted exclusively from an automated n8n workflow. All data and API usage comply strictly with content policies and legal standards. No illegal, offensive, or protected content is included. All data manipulated is legal and publicly accessible.