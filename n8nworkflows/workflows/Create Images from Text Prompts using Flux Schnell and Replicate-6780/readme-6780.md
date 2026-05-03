Create Images from Text Prompts using Flux Schnell and Replicate

https://n8nworkflows.xyz/workflows/create-images-from-text-prompts-using-flux-schnell-and-replicate-6780


# Create Images from Text Prompts using Flux Schnell and Replicate

### 1. Workflow Overview

This workflow automates the generation of images from text prompts using the Flux Schnell model hosted on Replicate's API. It is designed for users who want to convert descriptive text prompts into AI-generated images efficiently and reliably.

The workflow is logically organized into the following blocks:

- **1.1 Input Initialization**: Receives manual trigger input and sets necessary authentication and image generation parameters.
- **1.2 Image Generation Request**: Sends a request to Replicate API to create an image prediction based on the input parameters.
- **1.3 Prediction Status Polling**: Continuously checks the status of the image generation until it completes or fails, employing wait cycles to manage API calls.
- **1.4 Result Handling**: Processes the final results, distinguishing between success and failure, formats the output, and presents it.
- **1.5 Logging and Monitoring**: Logs the prediction requests for monitoring and debugging purposes.
- **1.6 Documentation and Support**: Provides embedded notes with model details, usage instructions, and support contacts.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Initialization

- **Overview:**  
Collects and sets up essential inputs including the Replicate API token and all model parameters for image generation, preparing data for the prediction request.

- **Nodes Involved:**  
  - Manual Trigger  
  - Set API Token  
  - Set Image Parameters

- **Node Details:**

  - **Manual Trigger**  
    - *Type:* manualTrigger  
    - *Role:* Starts workflow execution manually.  
    - *Configuration:* No parameters, simply triggers the chain.  
    - *Connections:* Outputs to Set API Token.  
    - *Edge Cases:* User must manually trigger; no automatic execution.

  - **Set API Token**  
    - *Type:* set  
    - *Role:* Stores the Replicate API token securely for use in subsequent API calls.  
    - *Configuration:* Assigns string parameter `api_token` with placeholder value `"YOUR_REPLICATE_API_TOKEN"`. User must replace this with their actual token.  
    - *Connections:* Outputs to Set Image Parameters.  
    - *Edge Cases:* Failure if token is invalid or missing.

  - **Set Image Parameters**  
    - *Type:* set  
    - *Role:* Defines all input parameters for the Flux Schnell model, combining required and optional inputs.  
    - *Configuration:*  
      - Copies `api_token` from previous node  
      - Sets:  
        - `prompt`: Text prompt for image generation (default: "A beautiful landscape with mountains and a lake at sunset")  
        - `seed`: -1 (random seed)  
        - `go_fast`: true (enables faster prediction mode)  
        - `megapixels`: "1" (approximate image resolution)  
        - `num_outputs`: 1 (number of images to generate)  
        - `aspect_ratio`: "1:1"  
        - `output_format`: "webp"  
        - `output_quality`: 80 (quality scale from 0 to 100)  
        - `num_inference_steps`: 4 (steps for diffusion model inference)  
        - `disable_safety_checker`: false (enables safety checks)  
    - *Connections:* Outputs to Create Image Prediction.  
    - *Edge Cases:* Invalid values may cause API errors.

---

#### 2.2 Image Generation Request

- **Overview:**  
Sends a POST request to Replicate API to initiate the image generation with the prepared parameters and waits for immediate response.

- **Nodes Involved:**  
  - Create Image Prediction

- **Node Details:**

  - **Create Image Prediction**  
    - *Type:* httpRequest  
    - *Role:* Sends the generation request to Replicate's `/v1/predictions` endpoint.  
    - *Configuration:*  
      - Method: POST  
      - URL: `https://api.replicate.com/v1/predictions`  
      - Headers: Authorization bearer token from `api_token`, Prefer header set to `wait` for synchronous response  
      - Body (JSON): includes model version ID and input parameters interpolated via expressions  
      - Response format: JSON, with error suppression (neverError=true) to avoid workflow interruption on API errors  
    - *Connections:* Outputs to Log Request.  
    - *Edge Cases:*  
      - API token invalid or expired causing authentication failure  
      - API rate limits or network errors  
      - Model version mismatch or parameter validation errors  
    - *Version Requirements:* Supports n8n HTTP Request node v4.1 (for advanced headers and JSON body handling).

---

#### 2.3 Prediction Status Polling

- **Overview:**  
Implements a loop to check the prediction status periodically until the image generation completes successfully or fails.

- **Nodes Involved:**  
  - Log Request  
  - Wait 5s  
  - Check Status  
  - Is Complete?  
  - Has Failed?  
  - Wait 10s

- **Node Details:**

  - **Log Request**  
    - *Type:* code  
    - *Role:* Logs prediction details to console for monitoring purposes.  
    - *Configuration:* JavaScript code logs timestamp, prediction ID, and model type.  
    - *Connections:* Outputs to Wait 5s.  
    - *Edge Cases:* Console output may be lost if n8n instance logs are not retained.

  - **Wait 5s**  
    - *Type:* wait  
    - *Role:* Pauses workflow for 5 seconds before next status check.  
    - *Configuration:* Wait duration set to 5 seconds.  
    - *Connections:* Outputs to Check Status.  
    - *Edge Cases:* Excessive waiting may slow workflow; too short may cause frequent API calls.

  - **Check Status**  
    - *Type:* httpRequest  
    - *Role:* Queries the prediction status from Replicate API using prediction ID.  
    - *Configuration:*  
      - Method: GET  
      - URL dynamically constructed with prediction ID from Create Image Prediction node  
      - Header: Authorization with API token  
      - Response JSON, never error to continue workflow on failure  
    - *Connections:* Outputs to Is Complete?.  
    - *Edge Cases:*  
      - Network errors or token invalidity  
      - Prediction ID missing or malformed

  - **Is Complete?**  
    - *Type:* if  
    - *Role:* Checks if the prediction status equals "succeeded".  
    - *Configuration:* Expression comparing `$json.status` to "succeeded".  
    - *Connections:*  
      - True branch to Success Response  
      - False branch to Has Failed?  
    - *Edge Cases:* Status field missing or unexpected values.

  - **Has Failed?**  
    - *Type:* if  
    - *Role:* Checks if the prediction status equals "failed".  
    - *Configuration:* Expression comparing `$json.status` to "failed".  
    - *Connections:*  
      - True branch to Error Response  
      - False branch to Wait 10s (retry)  
    - *Edge Cases:* Other statuses like "processing" lead to retry loop.

  - **Wait 10s**  
    - *Type:* wait  
    - *Role:* Pauses workflow for 10 seconds before retrying status check.  
    - *Configuration:* Wait duration set to 10 seconds.  
    - *Connections:* Outputs back to Check Status.  
    - *Edge Cases:* Extended delays if prediction remains pending.

---

#### 2.4 Result Handling

- **Overview:**  
Handles the final output, formatting success or error responses and making results available for further use.

- **Nodes Involved:**  
  - Success Response  
  - Error Response  
  - Display Result

- **Node Details:**

  - **Success Response**  
    - *Type:* set  
    - *Role:* Constructs a structured JSON object indicating success with image URL and metadata.  
    - *Configuration:* Sets `response` object with fields:  
      - `success`: true  
      - `image_url`: from `$json.output` (output URLs from API)  
      - `prediction_id`: from `$json.id`  
      - `status`: from `$json.status`  
      - `message`: "Image generated successfully"  
    - *Connections:* Outputs to Display Result.  
    - *Edge Cases:* Missing output URL leads to incomplete results.

  - **Error Response**  
    - *Type:* set  
    - *Role:* Constructs error response JSON with failure details.  
    - *Configuration:* Sets `response` object with fields:  
      - `success`: false  
      - `error`: from `$json.error` or default message  
      - `prediction_id`: from `$json.id`  
      - `status`: from `$json.status`  
      - `message`: "Failed to generate image"  
    - *Connections:* Outputs to Display Result.  
    - *Edge Cases:* May not capture all failure reasons if API response is incomplete.

  - **Display Result**  
    - *Type:* set  
    - *Role:* Assigns final response object to `final_result` for output or downstream consumption.  
    - *Configuration:* Sets `final_result` to the `response` object from previous node.  
    - *Connections:* Terminal node; no further outputs.  
    - *Edge Cases:* None significant.

---

#### 2.5 Logging and Monitoring

- **Overview:**  
Provides console logging of prediction requests to assist with debugging, auditing, and operational monitoring.

- **Nodes Involved:**  
  - Log Request (described above)

- **Node Details:**  
Same as described in 2.3 Log Request node.

---

#### 2.6 Documentation and Support

- **Overview:**  
Contains embedded sticky notes within the workflow that provide detailed model information, parameter explanations, usage instructions, and support contact information.

- **Nodes Involved:**  
  - Sticky Note9  
  - Sticky Note4

- **Node Details:**

  - **Sticky Note9**  
    - *Type:* stickyNote  
    - *Role:* Displays contact info and links for support  
    - *Content Highlights:*  
      - Contact email: Yaron@nofluff.online  
      - YouTube: https://www.youtube.com/@YaronBeen/videos  
      - LinkedIn: https://www.linkedin.com/in/yaronbeen/  
    - *Connections:* Visual only, no connections.

  - **Sticky Note4**  
    - *Type:* stickyNote  
    - *Role:* Extensive documentation embedded in workflow for user convenience  
    - *Content Highlights:*  
      - Model overview, parameter details, workflow explanation  
      - Quick start instructions  
      - Troubleshooting guide  
      - Useful links:  
        - Model: https://replicate.com/black-forest-labs/flux-schnell  
        - Replicate API Docs: https://replicate.com/docs  
        - n8n Docs: https://docs.n8n.io  
    - *Connections:* Visual only, no connections.

---

### 3. Summary Table

| Node Name            | Node Type           | Functional Role                   | Input Node(s)           | Output Node(s)            | Sticky Note                                                              |
|----------------------|---------------------|---------------------------------|-------------------------|---------------------------|--------------------------------------------------------------------------|
| Manual Trigger       | manualTrigger       | Start workflow manually          | —                       | Set API Token             | See Sticky Note4 for workflow overview and instructions                  |
| Set API Token        | set                 | Set Replicate API token          | Manual Trigger          | Set Image Parameters       | See Sticky Note4                                                          |
| Set Image Parameters | set                 | Configure image generation params| Set API Token           | Create Image Prediction    | See Sticky Note4                                                          |
| Create Image Prediction| httpRequest        | Send image generation request    | Set Image Parameters    | Log Request                | See Sticky Note4                                                          |
| Log Request          | code                | Log prediction details           | Create Image Prediction | Wait 5s                    | See Sticky Note4                                                          |
| Wait 5s              | wait                | Pause before status check        | Log Request             | Check Status               | See Sticky Note4                                                          |
| Check Status         | httpRequest         | Get prediction status            | Wait 5s, Wait 10s       | Is Complete?               | See Sticky Note4                                                          |
| Is Complete?         | if                  | Check if prediction succeeded    | Check Status            | Success Response, Has Failed? | See Sticky Note4                                                        |
| Has Failed?          | if                  | Check if prediction failed       | Is Complete?            | Error Response, Wait 10s   | See Sticky Note4                                                          |
| Wait 10s             | wait                | Pause before retrying status check| Has Failed?            | Check Status               | See Sticky Note4                                                          |
| Success Response     | set                 | Format success response          | Is Complete?            | Display Result             | See Sticky Note4                                                          |
| Error Response       | set                 | Format error response            | Has Failed?             | Display Result             | See Sticky Note4                                                          |
| Display Result       | set                 | Output final result object       | Success Response, Error Response | —                   | See Sticky Note4                                                          |
| Sticky Note9         | stickyNote           | Support contact info             | —                       | —                         | Contact: Yaron@nofluff.online, YouTube, LinkedIn links                   |
| Sticky Note4         | stickyNote           | Detailed documentation           | —                       | —                         | Extensive model info, usage, troubleshooting, and support links          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - No parameters required. This node starts the workflow manually.

2. **Create Set API Token Node**  
   - Type: Set  
   - Add field: `api_token` (string)  
   - Value: Replace `"YOUR_REPLICATE_API_TOKEN"` with your actual Replicate API token.  
   - Connect Manual Trigger → Set API Token.

3. **Create Set Image Parameters Node**  
   - Type: Set  
   - Add fields with these values:  
     - `api_token`: Expression to copy from `Set API Token` node: `={{ $('Set API Token').item.json.api_token }}`  
     - `seed`: number, default `-1`  
     - `prompt`: string, e.g., `"A beautiful landscape with mountains and a lake at sunset"`  
     - `go_fast`: boolean, `true`  
     - `megapixels`: string, `"1"`  
     - `num_outputs`: number, `1`  
     - `aspect_ratio`: string, `"1:1"`  
     - `output_format`: string, `"webp"`  
     - `output_quality`: number, `80`  
     - `num_inference_steps`: number, `4`  
     - `disable_safety_checker`: boolean, `false`  
   - Connect Set API Token → Set Image Parameters.

4. **Create HTTP Request Node "Create Image Prediction"**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.replicate.com/v1/predictions`  
   - Headers:  
     - `Authorization`: `Bearer {{ $json.api_token }}`  
     - `Prefer`: `wait`  
   - Body (JSON):  
     ```json
     {
       "version": "black-forest-labs/flux-schnell:c846a69991daf4c0e5d016514849d14ee5b2e6846ce6b9d6f21369e564cfe51e",
       "input": {
         "seed": {{ $json.seed }},
         "prompt": "{{ $json.prompt }}",
         "go_fast": {{ $json.go_fast }},
         "megapixels": "{{ $json.megapixels }}",
         "num_outputs": {{ $json.num_outputs }},
         "aspect_ratio": "{{ $json.aspect_ratio }}",
         "output_format": "{{ $json.output_format }}",
         "output_quality": {{ $json.output_quality }},
         "num_inference_steps": {{ $json.num_inference_steps }},
         "disable_safety_checker": {{ $json.disable_safety_checker }}
       }
     }
     ```  
   - Response Format: JSON  
   - Connect Set Image Parameters → Create Image Prediction.

5. **Create Code Node "Log Request"**  
   - Type: Code (JavaScript)  
   - Code:
     ```javascript
     const data = $input.all()[0].json;
     console.log('black-forest-labs/flux-schnell Request:', {
       timestamp: new Date().toISOString(),
       prediction_id: data.id,
       model_type: 'image'
     });
     return $input.all();
     ```  
   - Connect Create Image Prediction → Log Request.

6. **Create Wait Node "Wait 5s"**  
   - Type: Wait  
   - Duration: 5 seconds  
   - Connect Log Request → Wait 5s.

7. **Create HTTP Request Node "Check Status"**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://api.replicate.com/v1/predictions/{{ $('Create Image Prediction').item.json.id }}`  
   - Headers:  
     - `Authorization`: `Bearer {{ $('Set API Token').item.json.api_token }}`  
   - Response Format: JSON  
   - Connect Wait 5s → Check Status.  
   - Also connect Wait 10s → Check Status (retry loop).

8. **Create If Node "Is Complete?"**  
   - Type: If  
   - Condition: `$json.status` EQUALS `succeeded`  
   - Connect Check Status → Is Complete?  

9. **Create Set Node "Success Response"**  
   - Type: Set  
   - Assign field `response` (object) with:  
     ```json
     {
       "success": true,
       "image_url": $json.output,
       "prediction_id": $json.id,
       "status": $json.status,
       "message": "Image generated successfully"
     }
     ```  
   - Connect Is Complete? True → Success Response.

10. **Create Set Node "Display Result"**  
    - Type: Set  
    - Assign field `final_result` to: `={{ $json.response }}`  
    - Connect Success Response → Display Result.

11. **Create If Node "Has Failed?"**  
    - Type: If  
    - Condition: `$json.status` EQUALS `failed`  
    - Connect Is Complete? False → Has Failed?

12. **Create Set Node "Error Response"**  
    - Type: Set  
    - Assign field `response` (object) with:  
      ```json
      {
        "success": false,
        "error": $json.error || "Image generation failed",
        "prediction_id": $json.id,
        "status": $json.status,
        "message": "Failed to generate image"
      }
      ```  
    - Connect Has Failed? True → Error Response.

13. **Connect Error Response → Display Result**

14. **Create Wait Node "Wait 10s"**  
    - Type: Wait  
    - Duration: 10 seconds  
    - Connect Has Failed? False → Wait 10s.  
    - Connect Wait 10s → Check Status (retry loop).

15. **Add Sticky Notes for Documentation and Support**  
    - Create two Sticky Note nodes with content as per the embedded detailed documentation and support contacts.  
    - No connections needed.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                      | Context or Link                                         |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------|
| =======================================<br>FLUX-SCHNELL GENERATOR<br>For any questions or support, please contact:<br>Yaron@nofluff.online<br>Explore more tips and tutorials here:<br>- YouTube: https://www.youtube.com/@YaronBeen/videos<br>- LinkedIn: https://www.linkedin.com/in/yaronbeen/<br>======================================= | Support contact info and tutorial links embedded in Sticky Note9 |
| BLACK-FOREST-LABS/FLUX-SCHNELL - IMAGE GENERATION WORKFLOW<br>Powered by Replicate API and n8n Automation<br>Model documentation: https://replicate.com/black-forest-labs/flux-schnell<br>Replicate API docs: https://replicate.com/docs<br>n8n docs: https://docs.n8n.io<br>Includes parameter reference, usage instructions, and troubleshooting guide | Detailed embedded documentation in Sticky Note4        |

---

**Disclaimer:** The provided text originates solely from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.