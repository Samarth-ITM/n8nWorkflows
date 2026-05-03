Generate Custom Images from Prompts with Black Forest Labs Flux Dev via Replicate

https://n8nworkflows.xyz/workflows/generate-custom-images-from-prompts-with-black-forest-labs-flux-dev-via-replicate-6784


# Generate Custom Images from Prompts with Black Forest Labs Flux Dev via Replicate

### 1. Workflow Overview

This workflow automates the generation of custom images from textual prompts using the “flux-dev” model by Black Forest Labs via the Replicate API. It is designed for use cases that involve transforming descriptive text and optional image inputs into AI-generated images with configurable parameters. The workflow handles the entire process from initiation, parameter setup, API request submission, polling for completion, to response formatting and error handling.

The workflow’s logic is organized into the following functional blocks:

- **1.1 Input Reception and Initialization**: Starts the workflow manually and sets the API token.
- **1.2 Parameter Configuration**: Defines all parameters for the image generation request, including prompt, seed, image URL, and generation settings.
- **1.3 Image Generation Request**: Sends the configured parameters to the Replicate API to initiate image generation.
- **1.4 Status Polling and Control Loop**: Implements waiting and repeated status checks until the image generation completes successfully or fails.
- **1.5 Response Handling**: Processes success and failure cases, formats structured JSON responses.
- **1.6 Logging and Monitoring**: Records key request data for debugging and operational monitoring.
- **1.7 User Information and Documentation**: Provides embedded notes with instructions, parameter details, and support contacts.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Initialization

**Overview:**  
This block initiates the workflow manually and sets up the Replicate API authentication token.

**Nodes Involved:**  
- Manual Trigger  
- Set API Token

**Node Details:**

- **Manual Trigger**  
  - Type: Manual trigger node  
  - Role: Starts the workflow execution on user command  
  - Configuration: Default; no parameters  
  - Inputs: None  
  - Outputs: Connected to “Set API Token” node  
  - Edge Cases: None, but user must initiate the workflow manually

- **Set API Token**  
  - Type: Set node  
  - Role: Assigns the Replicate API token to a variable for use in downstream API calls  
  - Configuration: Contains a string assignment for `api_token` with placeholder “YOUR_REPLICATE_API_TOKEN” to be replaced by the user  
  - Inputs: From Manual Trigger  
  - Outputs: To “Set Image Parameters”  
  - Edge Cases: Failure if token is invalid or not replaced; could cause authorization errors downstream

---

#### 2.2 Parameter Configuration

**Overview:**  
Defines all parameters needed for the image generation request, combining required and optional inputs with sensible defaults.

**Nodes Involved:**  
- Set Image Parameters

**Node Details:**

- **Set Image Parameters**  
  - Type: Set node  
  - Role: Prepares all input parameters including prompt, seed, image URL, generation settings, and reuses the API token from previous node  
  - Configuration:  
    - `api_token`: copied from “Set API Token” output  
    - `seed`: -1 (random seed)  
    - `image`: default placeholder URL (https://picsum.photos/512/512)  
    - `prompt`: “A beautiful landscape with mountains and a lake at sunset”  
    - `go_fast`: true (optimized fast model)  
    - `guidance`: 3  
    - `megapixels`: "1"  
    - `num_outputs`: 1  
    - `aspect_ratio`: "1:1"  
    - `output_format`: "webp"  
    - `output_quality`: 80  
    - `prompt_strength`: 0.8  
    - `num_inference_steps`: 28  
    - `disable_safety_checker`: false  
  - Inputs: From “Set API Token”  
  - Outputs: To “Create Image Prediction” HTTP Request node  
  - Edge Cases: Improper parameter values can cause model failures or unexpected outputs; prompt must be valid string

---

#### 2.3 Image Generation Request

**Overview:**  
Sends a POST request to the Replicate API to start image generation using the configured parameters.

**Nodes Involved:**  
- Create Image Prediction

**Node Details:**

- **Create Image Prediction**  
  - Type: HTTP Request node  
  - Role: Calls Replicate’s prediction API endpoint to generate images  
  - Configuration:  
    - Method: POST  
    - URL: https://api.replicate.com/v1/predictions  
    - Headers: Authorization Bearer token from `api_token`  
    - Request Body: JSON including version ID and parameters mapped from prior node's JSON data  
    - Header “Prefer”: “wait” to request synchronous processing if possible  
    - Response: JSON, never error on HTTP errors to allow graceful handling  
  - Inputs: From “Set Image Parameters”  
  - Outputs: To “Log Request”  
  - Edge Cases:  
    - Authentication failure if API token invalid  
    - Network timeouts or API rate limiting errors  
    - API rejects invalid parameters or unsupported model version

---

#### 2.4 Status Polling and Control Loop

**Overview:**  
Implements a polling mechanism to repeatedly check the prediction status until the image generation is complete or has failed. It uses wait nodes and conditional branching for retries and time delays.

**Nodes Involved:**  
- Log Request  
- Wait 5s  
- Check Status  
- Is Complete? (If)  
- Has Failed? (If)  
- Wait 10s

**Node Details:**

- **Log Request**  
  - Type: Code node  
  - Role: Logs key generation details such as timestamp, prediction ID, and model type for monitoring  
  - Configuration: JavaScript code that logs to console and passes input unchanged  
  - Inputs: From “Create Image Prediction”  
  - Outputs: To “Wait 5s”  
  - Edge Cases: None significant; logging failure does not halt workflow

- **Wait 5s**  
  - Type: Wait node  
  - Role: Pauses workflow for 5 seconds before status check  
  - Configuration: 5 seconds delay  
  - Inputs: From “Log Request”  
  - Outputs: To “Check Status”  
  - Edge Cases: None

- **Check Status**  
  - Type: HTTP Request node  
  - Role: Queries Replicate API for current status of the prediction using prediction ID  
  - Configuration:  
    - Method: GET  
    - URL dynamically built with prediction ID from “Create Image Prediction” output  
    - Authorization header using API token from “Set API Token”  
    - Response format: JSON, never error  
  - Inputs: From “Wait 5s” and “Wait 10s” retry node  
  - Outputs: To “Is Complete?”  
  - Edge Cases: Network errors, token expiration, or prediction not found

- **Is Complete?**  
  - Type: If node  
  - Role: Checks if the status returned is “succeeded”  
  - Configuration: Compares JSON `status` to string “succeeded” (case-sensitive)  
  - Inputs: From “Check Status”  
  - Outputs:  
    - True: To “Success Response”  
    - False: To “Has Failed?”  
  - Edge Cases: Status field missing or unexpected status values

- **Has Failed?**  
  - Type: If node  
  - Role: Checks if the status is “failed”  
  - Configuration: Compares JSON `status` to string “failed” (case-sensitive)  
  - Inputs: From “Is Complete?” false branch  
  - Outputs:  
    - True: To “Error Response”  
    - False: To “Wait 10s” (retry after delay)  
  - Edge Cases: Unexpected status values or indefinite waiting if status stuck

- **Wait 10s**  
  - Type: Wait node  
  - Role: Delays 10 seconds before retrying “Check Status” to avoid excessive polling  
  - Configuration: 10 seconds delay  
  - Inputs: From “Has Failed?” false branch  
  - Outputs: To “Check Status”  
  - Edge Cases: None

---

#### 2.5 Response Handling

**Overview:**  
Prepares structured JSON outputs for both successful image generation and failure cases, then passes the result to a display node.

**Nodes Involved:**  
- Success Response  
- Error Response  
- Display Result

**Node Details:**

- **Success Response**  
  - Type: Set node  
  - Role: Constructs a JSON object indicating success with image URL, prediction ID, status, and message  
  - Configuration: Sets `response` object with fields: `success: true`, `image_url`, `prediction_id`, `status`, and a success message extracted from prediction output  
  - Inputs: From “Is Complete?” true branch  
  - Outputs: To “Display Result”  
  - Edge Cases: Missing output URLs or fields in API response

- **Error Response**  
  - Type: Set node  
  - Role: Constructs a JSON error response with failure details, prediction ID, and status  
  - Configuration: Sets `response` object with fields: `success: false`, `error` message (from API or default), `prediction_id`, `status`, and failure message  
  - Inputs: From “Has Failed?” true branch  
  - Outputs: To “Display Result”  
  - Edge Cases: Missing error details from API response

- **Display Result**  
  - Type: Set node  
  - Role: Stores the final response object (success or error) in a field `final_result` for downstream use or output  
  - Configuration: Assigns `final_result` from the incoming `response` field  
  - Inputs: From both “Success Response” and “Error Response”  
  - Outputs: None (end of workflow)  
  - Edge Cases: None

---

#### 2.6 Logging and Monitoring

**Overview:**  
Logs prediction requests for operational visibility in the console.

**Nodes Involved:**  
- Log Request (analyzed above in 2.4)

---

#### 2.7 User Information and Documentation

**Overview:**  
Contains embedded sticky notes providing workflow metadata, usage instructions, parameter references, and support info.

**Nodes Involved:**  
- Sticky Note9  
- Sticky Note4

**Node Details:**

- **Sticky Note9**  
  - Type: Sticky Note  
  - Role: Provides workflow branding and contact information for support  
  - Content: Contact email, YouTube and LinkedIn links for tutorial and support  
  - Position: Top-left of canvas  
  - Edge Cases: None

- **Sticky Note4**  
  - Type: Sticky Note  
  - Role: Comprehensive workflow documentation including model overview, parameter guide, workflow components explanation, benefits, quick start instructions, troubleshooting, and external resource links  
  - Content: Extensive markdown-formatted text with references to documentation and best practices  
  - Position: Left side of canvas  
  - Edge Cases: None

---

### 3. Summary Table

| Node Name           | Node Type          | Functional Role                       | Input Node(s)            | Output Node(s)             | Sticky Note                                                                                                                           |
|---------------------|--------------------|------------------------------------|--------------------------|----------------------------|-------------------------------------------------------------------------------------------------------------------------------------|
| Manual Trigger      | Manual Trigger     | Starts workflow execution           | None                     | Set API Token              |                                                                                                                                     |
| Set API Token       | Set                | Sets Replicate API token            | Manual Trigger           | Set Image Parameters       |                                                                                                                                     |
| Set Image Parameters| Set                | Configures image generation params  | Set API Token            | Create Image Prediction    |                                                                                                                                     |
| Create Image Prediction | HTTP Request    | Sends image generation request to API | Set Image Parameters    | Log Request                |                                                                                                                                     |
| Log Request         | Code               | Logs request details for monitoring | Create Image Prediction  | Wait 5s                    |                                                                                                                                     |
| Wait 5s             | Wait               | Waits 5 seconds before status check | Log Request              | Check Status               |                                                                                                                                     |
| Check Status        | HTTP Request       | Queries prediction status           | Wait 5s, Wait 10s        | Is Complete?               |                                                                                                                                     |
| Is Complete?        | If                 | Checks if prediction succeeded     | Check Status             | Success Response, Has Failed? |                                                                                                                                     |
| Has Failed?         | If                 | Checks if prediction failed        | Is Complete?             | Error Response, Wait 10s   |                                                                                                                                     |
| Wait 10s            | Wait               | Waits 10 seconds before retrying   | Has Failed?              | Check Status               |                                                                                                                                     |
| Success Response    | Set                | Prepares success JSON response     | Is Complete?             | Display Result             |                                                                                                                                     |
| Error Response      | Set                | Prepares error JSON response       | Has Failed?              | Display Result             |                                                                                                                                     |
| Display Result      | Set                | Stores final response object       | Success Response, Error Response | None                  |                                                                                                                                     |
| Sticky Note9        | Sticky Note        | Contact and branding information   | None                     | None                      | =======================================<br>FLUX-DEV GENERATOR<br>Contact: Yaron@nofluff.online<br>YouTube: https://www.youtube.com/@YaronBeen/videos<br>LinkedIn: https://www.linkedin.com/in/yaronbeen/ |
| Sticky Note4        | Sticky Note        | Detailed workflow documentation    | None                     | None                      | See detailed workflow overview, parameters, usage instructions, troubleshooting, and links in node content                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create “Manual Trigger” node**  
   - Type: Manual Trigger  
   - Purpose: Start workflow on demand  
   - No parameters to set  

2. **Create “Set API Token” node**  
   - Type: Set  
   - Connect input from “Manual Trigger”  
   - Add assignment: `api_token` (string) = `"YOUR_REPLICATE_API_TOKEN"` (replace with actual token)  

3. **Create “Set Image Parameters” node**  
   - Type: Set  
   - Connect input from “Set API Token”  
   - Assign variables as follows:  
     - `api_token`: Copy from `$('Set API Token').item.json.api_token`  
     - `seed`: -1 (number)  
     - `image`: `"https://picsum.photos/512/512"` (string)  
     - `prompt`: `"A beautiful landscape with mountains and a lake at sunset"` (string)  
     - `go_fast`: true (boolean)  
     - `guidance`: 3 (number)  
     - `megapixels`: `"1"` (string)  
     - `num_outputs`: 1 (number)  
     - `aspect_ratio`: `"1:1"` (string)  
     - `output_format`: `"webp"` (string)  
     - `output_quality`: 80 (number)  
     - `prompt_strength`: 0.8 (number)  
     - `num_inference_steps`: 28 (number)  
     - `disable_safety_checker`: false (boolean)  

4. **Create “Create Image Prediction” node**  
   - Type: HTTP Request  
   - Connect input from “Set Image Parameters”  
   - Method: POST  
   - URL: `https://api.replicate.com/v1/predictions`  
   - Headers:  
     - `Authorization`: `Bearer {{$json.api_token}}`  
     - `Prefer`: `wait`  
   - Body (JSON): Structure parameters as per the node’s configuration, mapping all fields from the input JSON (see parameters in step 3)  
   - Response format: JSON, set to never error  

5. **Create “Log Request” node**  
   - Type: Code  
   - Connect input from “Create Image Prediction”  
   - Add JavaScript code to log timestamp, prediction ID, model type, and return input unchanged  

6. **Create “Wait 5s” node**  
   - Type: Wait  
   - Connect input from “Log Request”  
   - Set delay to 5 seconds  

7. **Create “Check Status” node**  
   - Type: HTTP Request  
   - Connect input from “Wait 5s” (and from “Wait 10s” later)  
   - Method: GET  
   - URL: `https://api.replicate.com/v1/predictions/{{$('Create Image Prediction').item.json.id}}`  
   - Headers:  
     - `Authorization`: `Bearer {{$('Set API Token').item.json.api_token}}`  
   - Response format: JSON, set to never error  

8. **Create “Is Complete?” node**  
   - Type: If condition node  
   - Connect input from “Check Status”  
   - Condition: Check if `$json.status` equals `"succeeded"` (case-sensitive)  
   - True branch: Connect to “Success Response”  
   - False branch: Connect to “Has Failed?”  

9. **Create “Has Failed?” node**  
   - Type: If condition node  
   - Connect input from “Is Complete?” false branch  
   - Condition: Check if `$json.status` equals `"failed"` (case-sensitive)  
   - True branch: Connect to “Error Response”  
   - False branch: Connect to “Wait 10s”  

10. **Create “Wait 10s” node**  
    - Type: Wait  
    - Connect input from “Has Failed?” false branch  
    - Set delay to 10 seconds  
    - Connect output back to “Check Status” (loop)  

11. **Create “Success Response” node**  
    - Type: Set  
    - Connect input from “Is Complete?” true branch  
    - Assign:  
      ```json
      {
        "response": {
          "success": true,
          "image_url": $json.output,
          "prediction_id": $json.id,
          "status": $json.status,
          "message": "Image generated successfully"
        }
      }
      ```  

12. **Create “Error Response” node**  
    - Type: Set  
    - Connect input from “Has Failed?” true branch  
    - Assign:  
      ```json
      {
        "response": {
          "success": false,
          "error": $json.error || "Image generation failed",
          "prediction_id": $json.id,
          "status": $json.status,
          "message": "Failed to generate image"
        }
      }
      ```  

13. **Create “Display Result” node**  
    - Type: Set  
    - Connect input from both “Success Response” and “Error Response”  
    - Assign final field: `final_result` = `$json.response`  

14. **(Optional) Add Sticky Notes**  
    - Create two Sticky Note nodes with the provided detailed content for user guidance and support contacts  
    - Position them on the canvas for clarity  

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| For any questions or support, contact: Yaron@nofluff.online | Contact Email (Sticky Note9) |
| Explore tips and tutorials: YouTube: https://www.youtube.com/@YaronBeen/videos, LinkedIn: https://www.linkedin.com/in/yaronbeen/ | Social Media Links (Sticky Note9) |
| Model documentation: https://replicate.com/black-forest-labs/flux-dev | Model Docs |
| Replicate API docs: https://replicate.com/docs | API Documentation |
| n8n documentation: https://docs.n8n.io | Platform Documentation |
| Workflow includes parameter guide, quick start instructions, troubleshooting notes in embedded documentation | Sticky Note4 content |

---

**Disclaimer:** The text above is derived exclusively from an automated n8n workflow using the Replicate API. It complies strictly with content policies and contains no illegal or protected content. All data handled is legal and publicly available.