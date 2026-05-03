Generate 3D Structured Images with Fire Part Crafter AI via Replicate API

https://n8nworkflows.xyz/workflows/generate-3d-structured-images-with-fire-part-crafter-ai-via-replicate-api-6790


# Generate 3D Structured Images with Fire Part Crafter AI via Replicate API

### 1. Workflow Overview

This workflow automates the generation of structured 3D images using the Fire Part Crafter AI model via the Replicate API. It is designed to accept an input image, configure generation parameters, send a request to the API, monitor the prediction status until completion, handle success or failure responses, and provide a structured output with the generated image URL or error details.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception & Initialization:** Manual trigger to start the workflow and setting the Replicate API token.
- **1.2 Parameter Configuration:** Setting all input parameters required by the Fire Part Crafter model, including both mandatory and optional fields.
- **1.3 Prediction Request & Logging:** Sending the prediction request to the Replicate API and logging the request for monitoring.
- **1.4 Status Polling Loop:** Waiting and polling the API repeatedly to check the prediction status until it completes successfully or fails.
- **1.5 Result Handling:** Processing successful predictions to return the image URL or handling failures with an error response.
- **1.6 Final Output:** Consolidating the result into a final structured response for downstream use or output delivery.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Initialization

- **Overview:**  
  This block begins the workflow execution manually and securely sets the API token required for authentication with the Replicate API.

- **Nodes Involved:**  
  - Manual Trigger  
  - Set API Token

- **Node Details:**

  - **Manual Trigger**  
    - *Type:* Trigger node  
    - *Role:* Initiates the workflow manually by user interaction.  
    - *Configuration:* Default manual trigger with no parameters.  
    - *Connections:* Output connected to "Set API Token".  
    - *Failure Cases:* None; manual trigger only fails if manually aborted.

  - **Set API Token**  
    - *Type:* Set node  
    - *Role:* Stores the Replicate API token as a string variable for use in subsequent HTTP requests.  
    - *Configuration:* Contains a single assignment named `api_token` set as a string placeholder `"YOUR_REPLICATE_API_TOKEN"`. User must replace this with a valid token before execution.  
    - *Connections:* Input from "Manual Trigger", output to "Set Image Parameters".  
    - *Failure Cases:* Workflow failure if token is invalid or missing; no automatic validation here.

#### 2.2 Parameter Configuration

- **Overview:**  
  This block sets all parameters required for the Fire Part Crafter model, including image URL, seed, number of parts, and optional generation parameters.

- **Nodes Involved:**  
  - Set Image Parameters

- **Node Details:**

  - **Set Image Parameters**  
    - *Type:* Set node  
    - *Role:* Defines all input parameters for the prediction request.  
    - *Configuration:*  
      - Copies `api_token` from previous node.  
      - `seed`: integer, default 0 (random seed).  
      - `image`: string URL, default `"https://picsum.photos/512/512"`.  
      - `num_parts`: integer, default 16.  
      - `num_tokens`: string, default `"2048"`.  
      - `guidance_scale`: number, default 7.  
      - `remove_background`: boolean, default false.  
      - `use_flash_decoder`: boolean, default false.  
      - `num_inference_steps`: integer, default 50.  
    - *Expressions:* Uses expression syntax to inherit `api_token`.  
    - *Connections:* Input from "Set API Token", output to "Create Image Prediction".  
    - *Failure Cases:* Parameter type mismatches or missing required fields could cause API errors downstream.

#### 2.3 Prediction Request & Logging

- **Overview:**  
  This block sends the prediction request to the Replicate API and logs the request details for monitoring and debugging purposes.

- **Nodes Involved:**  
  - Create Image Prediction  
  - Log Request

- **Node Details:**

  - **Create Image Prediction**  
    - *Type:* HTTP Request node  
    - *Role:* Sends a POST request to Replicate API to start image generation using configured parameters.  
    - *Configuration:*  
      - URL: `https://api.replicate.com/v1/predictions`  
      - Method: POST  
      - Body: JSON with model version and input parameters dynamically mapped from previous node.  
      - Headers: Authorization Bearer token from `api_token`, Prefer header set to `wait` (wait for completion if possible).  
      - Response: JSON, never error option enabled to capture all responses.  
    - *Connections:* Input from "Set Image Parameters", output to "Log Request".  
    - *Failure Cases:* Authentication errors if token invalid; API rate limits; malformed request body; network timeouts.

  - **Log Request**  
    - *Type:* Code node  
    - *Role:* Logs prediction request details such as timestamp, prediction ID, and model type for monitoring.  
    - *Configuration:* JavaScript code logging to console the prediction ID and timestamp extracted from input JSON.  
    - *Connections:* Input from "Create Image Prediction", output to "Wait 5s".  
    - *Failure Cases:* Logging failures do not break the workflow; safe fallback.

#### 2.4 Status Polling Loop

- **Overview:**  
  This block implements a polling loop that waits and periodically queries the prediction status until it completes successfully or fails.

- **Nodes Involved:**  
  - Wait 5s  
  - Check Status  
  - Is Complete?  
  - Has Failed?  
  - Wait 10s

- **Node Details:**

  - **Wait 5s**  
    - *Type:* Wait node  
    - *Role:* Pauses workflow execution for 5 seconds before status check.  
    - *Configuration:* Wait time set to 5 seconds.  
    - *Connections:* Input from "Log Request", output to "Check Status".  
    - *Failure Cases:* None typical.

  - **Check Status**  
    - *Type:* HTTP Request node  
    - *Role:* Queries the Replicate API for current status of prediction using prediction ID.  
    - *Configuration:*  
      - URL dynamically constructed with prediction ID from `"Create Image Prediction"`.  
      - Method: GET  
      - Headers: Authorization Bearer token from "Set API Token".  
      - Response: JSON, never error enabled.  
    - *Connections:* Input from "Wait 5s" or "Wait 10s", output to "Is Complete?".  
    - *Failure Cases:* Network issues, invalid prediction ID, auth errors.

  - **Is Complete?**  
    - *Type:* If node  
    - *Role:* Checks if the prediction status equals `"succeeded"`.  
    - *Configuration:* Condition: `$json.status` equals `"succeeded"`.  
    - *Connections:*  
      - True branch to "Success Response".  
      - False branch to "Has Failed?".  
    - *Failure Cases:* Expression evaluation errors if status field missing.

  - **Has Failed?**  
    - *Type:* If node  
    - *Role:* Checks if prediction status equals `"failed"`.  
    - *Configuration:* Condition: `$json.status` equals `"failed"`.  
    - *Connections:*  
      - True branch to "Error Response".  
      - False branch to "Wait 10s" (retry polling).  
    - *Failure Cases:* Same as above, expression errors.

  - **Wait 10s**  
    - *Type:* Wait node  
    - *Role:* Waits 10 seconds before retrying status check.  
    - *Configuration:* Wait time set to 10 seconds.  
    - *Connections:* Input from "Has Failed?" (false), output to "Check Status".  
    - *Failure Cases:* None typical.

#### 2.5 Result Handling

- **Overview:**  
  This block prepares structured JSON responses for success or error outcomes, encapsulating relevant metadata and user-friendly messages.

- **Nodes Involved:**  
  - Success Response  
  - Error Response

- **Node Details:**

  - **Success Response**  
    - *Type:* Set node  
    - *Role:* Constructs a success response object containing the image URL, prediction ID, status, and message.  
    - *Configuration:*  
      - Creates an object field `response` with keys: `success: true`, `image_url` (from prediction output), `prediction_id`, `status`, and a success message.  
    - *Connections:* Input from "Is Complete?" (true), output to "Display Result".  
    - *Failure Cases:* Missing output data could cause undefined fields.

  - **Error Response**  
    - *Type:* Set node  
    - *Role:* Constructs an error response object with error details, prediction ID, status, and failure message.  
    - *Configuration:*  
      - Creates an object field `response` with keys: `success: false`, `error` (from error field or default message), `prediction_id`, `status`, and failure message.  
    - *Connections:* Input from "Has Failed?" (true), output to "Display Result".  
    - *Failure Cases:* Missing error details handled with fallback string.

#### 2.6 Final Output

- **Overview:**  
  This node consolidates the response object into a final result field for downstream usage or output.

- **Nodes Involved:**  
  - Display Result

- **Node Details:**

  - **Display Result**  
    - *Type:* Set node  
    - *Role:* Assigns the final response object to a field named `final_result` for output or further processing.  
    - *Configuration:* Sets `final_result` to the `response` object from success or error nodes.  
    - *Connections:* Input from both "Success Response" and "Error Response".  
    - *Failure Cases:* None typical.

---

### 3. Summary Table

| Node Name             | Node Type               | Functional Role                            | Input Node(s)               | Output Node(s)              | Sticky Note                                                                                 |
|-----------------------|-------------------------|--------------------------------------------|-----------------------------|-----------------------------|---------------------------------------------------------------------------------------------|
| Manual Trigger        | Trigger                 | Starts workflow execution                   |                             | Set API Token                |                                                                                             |
| Set API Token         | Set                     | Stores Replicate API token                   | Manual Trigger              | Set Image Parameters         |                                                                                             |
| Set Image Parameters  | Set                     | Defines model input parameters               | Set API Token               | Create Image Prediction      |                                                                                             |
| Create Image Prediction| HTTP Request            | Sends image generation request to Replicate| Set Image Parameters        | Log Request                 |                                                                                             |
| Log Request           | Code                    | Logs prediction request details              | Create Image Prediction     | Wait 5s                     |                                                                                             |
| Wait 5s               | Wait                    | Waits 5 seconds before status check          | Log Request                | Check Status                |                                                                                             |
| Check Status          | HTTP Request            | Queries prediction status                     | Wait 5s, Wait 10s          | Is Complete?                |                                                                                             |
| Is Complete?          | If                      | Checks if prediction status is "succeeded"  | Check Status               | Success Response, Has Failed? |                                                                                             |
| Has Failed?           | If                      | Checks if prediction status is "failed"     | Is Complete? (false)        | Error Response, Wait 10s    |                                                                                             |
| Wait 10s              | Wait                    | Waits 10 seconds for retry                     | Has Failed? (false)          | Check Status                |                                                                                             |
| Success Response      | Set                     | Prepares success response object             | Is Complete? (true)         | Display Result              |                                                                                             |
| Error Response        | Set                     | Prepares error response object               | Has Failed? (true)          | Display Result              |                                                                                             |
| Display Result        | Set                     | Consolidates final response for output       | Success Response, Error Response |                          |                                                                                             |
| Sticky Note9          | Sticky Note             | Displays branding and contact info            |                             |                             | =======================================<br>PART-CRAFTER GENERATOR<br>Contact: Yaron@nofluff.online<br>YouTube & LinkedIn links |
| Sticky Note4          | Sticky Note             | Comprehensive workflow and model documentation|                             |                             | Detailed workflow overview, parameter guide, troubleshooting, and resource links             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node:**  
   - Name: `Manual Trigger`  
   - Purpose: Start the workflow manually.

2. **Add a Set node to store API token:**  
   - Name: `Set API Token`  
   - Add a string field: `api_token` with value: `"YOUR_REPLICATE_API_TOKEN"` (replace with actual token).  
   - Connect `Manual Trigger` output to this node.

3. **Add a Set node to configure image generation parameters:**  
   - Name: `Set Image Parameters`  
   - Assign the following fields:  
     - `api_token`: set via expression from `Set API Token` node (`{{$node["Set API Token"].json.api_token}}`)  
     - `seed`: number, default `0`  
     - `image`: string URL, default `"https://picsum.photos/512/512"`  
     - `num_parts`: number, default `16`  
     - `num_tokens`: string, default `"2048"`  
     - `guidance_scale`: number, default `7`  
     - `remove_background`: boolean, default `false`  
     - `use_flash_decoder`: boolean, default `false`  
     - `num_inference_steps`: number, default `50`  
   - Connect output of `Set API Token` to this node.

4. **Add an HTTP Request node to create image prediction:**  
   - Name: `Create Image Prediction`  
   - Method: POST  
   - URL: `https://api.replicate.com/v1/predictions`  
   - Headers:  
     - `Authorization`: `Bearer {{$json.api_token}}`  
     - `Prefer`: `wait`  
   - Body (JSON):  
     ```json
     {
       "version": "fire/part-crafter:a8fe4d91050b7d6497dd6676506a4105a3903b7287353681f52f3be0bb01d67f",
       "input": {
         "seed": {{$json.seed}},
         "image": "{{$json.image}}",
         "num_parts": {{$json.num_parts}},
         "num_tokens": "{{$json.num_tokens}}",
         "guidance_scale": {{$json.guidance_scale}},
         "remove_background": {{$json.remove_background}},
         "use_flash_decoder": {{$json.use_flash_decoder}},
         "num_inference_steps": {{$json.num_inference_steps}}
       }
     }
     ```  
   - Send body as JSON.  
   - Connect output of `Set Image Parameters` to this node.

5. **Add a Code node to log the request:**  
   - Name: `Log Request`  
   - JavaScript code snippet:
     ```javascript
     const data = $input.all()[0].json;
     console.log('fire/part-crafter Request:', {
       timestamp: new Date().toISOString(),
       prediction_id: data.id,
       model_type: 'image'
     });
     return $input.all();
     ```  
   - Connect output of `Create Image Prediction` to this node.

6. **Add a Wait node to pause 5 seconds:**  
   - Name: `Wait 5s`  
   - Wait for 5 seconds.  
   - Connect output of `Log Request` to this node.

7. **Add an HTTP Request node to check prediction status:**  
   - Name: `Check Status`  
   - Method: GET  
   - URL: `https://api.replicate.com/v1/predictions/{{$node["Create Image Prediction"].json.id}}`  
   - Header:  
     - `Authorization`: `Bearer {{$node["Set API Token"].json.api_token}}`  
   - Response format: JSON  
   - Connect output of `Wait 5s` to this node.

8. **Add an If node to check if status is succeeded:**  
   - Name: `Is Complete?`  
   - Condition: `$json.status` equals `"succeeded"`  
   - True output to `Success Response` node (to be created)  
   - False output to `Has Failed?` node (to be created)  
   - Connect output of `Check Status` to this node.

9. **Add an If node to check if status is failed:**  
   - Name: `Has Failed?`  
   - Condition: `$json.status` equals `"failed"`  
   - True output to `Error Response` node (to be created)  
   - False output to `Wait 10s` node (to be created)  
   - Connect False output of `Is Complete?` to this node.

10. **Add a Wait node to pause 10 seconds before retry:**  
    - Name: `Wait 10s`  
    - Wait for 10 seconds.  
    - Connect False output of `Has Failed?` to this node.  
    - Connect output of `Wait 10s` back to `Check Status` node (loop).

11. **Add a Set node to create success response:**  
    - Name: `Success Response`  
    - Set field `response` as object with keys:  
      - `success`: `true`  
      - `image_url`: from `$json.output`  
      - `prediction_id`: from `$json.id`  
      - `status`: from `$json.status`  
      - `message`: `"Image generated successfully"`  
    - Connect True output of `Is Complete?` to this node.

12. **Add a Set node to create error response:**  
    - Name: `Error Response`  
    - Set field `response` as object with keys:  
      - `success`: `false`  
      - `error`: from `$json.error` or default `"Image generation failed"`  
      - `prediction_id`: from `$json.id`  
      - `status`: from `$json.status`  
      - `message`: `"Failed to generate image"`  
    - Connect True output of `Has Failed?` to this node.

13. **Add a Set node to consolidate and display result:**  
    - Name: `Display Result`  
    - Set field `final_result` to the `response` field from either Success or Error Response node.  
    - Connect outputs of both `Success Response` and `Error Response` to this node.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| For any questions or support, contact Yaron at Yaron@nofluff.online | Contact info in Sticky Note9 |
| Explore more tips and tutorials on YouTube: https://www.youtube.com/@YaronBeen/videos | Sticky Note9 |
| LinkedIn profile for updates and contact: https://www.linkedin.com/in/yaronbeen/ | Sticky Note9 |
| Model Documentation: https://replicate.com/fire/part-crafter | Model overview and API reference |
| Replicate API Documentation: https://replicate.com/docs | API usage guidelines |
| n8n Documentation: https://docs.n8n.io | Platform usage and node references |
| Workflow includes intelligent retry logic with wait nodes to handle API latency and transient errors gracefully | Design note |
| Replace placeholder API token with a valid one from https://replicate.com/account | Important setup step |
| Recommended to test with default parameters initially and adjust based on use case | Best practice |

---

**Disclaimer:** The content above is generated from an automated n8n workflow designed exclusively for legal and public data processing. It adheres strictly to content policies and contains no illegal, offensive, or protected elements.