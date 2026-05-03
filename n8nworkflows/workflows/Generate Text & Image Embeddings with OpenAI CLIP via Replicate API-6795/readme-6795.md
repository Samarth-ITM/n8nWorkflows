Generate Text & Image Embeddings with OpenAI CLIP via Replicate API

https://n8nworkflows.xyz/workflows/generate-text---image-embeddings-with-openai-clip-via-replicate-api-6795


# Generate Text & Image Embeddings with OpenAI CLIP via Replicate API

---

### 1. Workflow Overview

This workflow automates the generation of text and image embeddings using the OpenAI CLIP model through the Replicate API. It is designed for users who want to encode images and text into vector embeddings for applications such as similarity search, classification, or AI-driven content generation. The workflow handles the entire pipeline from manual trigger to API request, status polling, and result delivery with error management.

The workflow is logically divided into these functional blocks:

- **1.1 Input Initialization and Configuration**: Manual start, API token setup, and input parameter preparation.
- **1.2 Prediction Request**: Submission of the embedding generation request to the Replicate API.
- **1.3 Status Polling and Result Handling**: Repeated checking of the prediction status, branching on success or failure.
- **1.4 Response Formatting and Logging**: Structuring the output response and logging request details for monitoring.
- **1.5 User Guidance and Metadata**: Sticky notes providing detailed documentation, usage instructions, and contact information.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Initialization and Configuration

**Overview:**  
This block initializes the workflow execution via manual trigger, sets the Replicate API token, and configures the input parameters (text and image URL) to be sent to the model.

**Nodes Involved:**  
- Manual Trigger  
- Set API Token  
- Set Image Parameters

**Node Details:**

- **Manual Trigger**  
  - Type: `manualTrigger`  
  - Role: Starts the workflow manually on user demand.  
  - Configuration: No parameters required.  
  - Inputs: None  
  - Outputs: Connected to "Set API Token"  
  - Edge Cases: None.  
  - Version: 1

- **Set API Token**  
  - Type: `set`  
  - Role: Stores the Replicate API token as a workflow variable.  
  - Configuration: Assigns a string variable `api_token` with placeholder "YOUR_REPLICATE_API_TOKEN".  
  - Inputs: From Manual Trigger  
  - Outputs: To "Set Image Parameters"  
  - Edge Cases: Failure if token is invalid or missing during API calls.  
  - Version: 3.3

- **Set Image Parameters**  
  - Type: `set`  
  - Role: Prepares the input payload parameters for the API call, including text (empty by default) and a default sample image URL. It also passes through the API token from previous node.  
  - Configuration:
    - `api_token` copied from "Set API Token" node.  
    - `text` initialized as empty string (user can customize).  
    - `image` set to `"https://picsum.photos/512/512"` as a default test image.  
  - Inputs: From "Set API Token"  
  - Outputs: To "Create Image Prediction"  
  - Edge Cases: Empty text or invalid image URL may affect generation results.  
  - Version: 3.3

---

#### 2.2 Prediction Request

**Overview:**  
This block sends the image and optional text inputs to the Replicate API to create a new prediction job using the OpenAI CLIP model version specified. It includes setting authorization headers and waiting for the API response.

**Nodes Involved:**  
- Create Image Prediction  
- Log Request  
- Wait 5s

**Node Details:**

- **Create Image Prediction**  
  - Type: `httpRequest`  
  - Role: Sends POST request to `https://api.replicate.com/v1/predictions` to start the embedding generation.  
  - Configuration:
    - HTTP Method: POST  
    - URL: `https://api.replicate.com/v1/predictions`  
    - JSON Body:  
      ```json
      {
        "version": "openai/clip:fd95fe35085b5b9a63d830d3126311ee6b32a7a976c78eb5f210a3a007bcdda6",
        "input": {
          "text": "{{ $json.text }}",
          "image": "{{ $json.image }}"
        }
      }
      ```  
    - Headers: Authorization bearer token from `api_token` variable, and `Prefer: wait` to wait for synchronous response if possible.  
    - Response format: JSON, never error on HTTP errors to handle them in workflow logic.  
  - Inputs: From "Set Image Parameters"  
  - Outputs: To "Log Request"  
  - Edge Cases:  
    - Invalid token resulting in 401 Unauthorized  
    - Malformed input parameters causing API errors  
    - Network or timeout errors  
  - Version: 4.1

- **Log Request**  
  - Type: `code` (JavaScript)  
  - Role: Logs prediction request details (timestamp, prediction id, model type) for monitoring and debugging purposes.  
  - Configuration: Logs to console and passes input unchanged.  
  - Inputs: From "Create Image Prediction"  
  - Outputs: To "Wait 5s"  
  - Edge Cases: None  
  - Version: 2

- **Wait 5s**  
  - Type: `wait`  
  - Role: Pauses workflow for 5 seconds before checking prediction status.  
  - Configuration: Fixed 5 seconds delay  
  - Inputs: From "Log Request"  
  - Outputs: To "Check Status"  
  - Edge Cases: None  
  - Version: 1

---

#### 2.3 Status Polling and Result Handling

**Overview:**  
This block implements a polling mechanism to check the prediction status until it completes successfully or fails. It handles both outcomes and retries failure checks with delays.

**Nodes Involved:**  
- Check Status  
- Is Complete?  
- Has Failed?  
- Wait 10s  
- Success Response  
- Error Response  
- Display Result

**Node Details:**

- **Check Status**  
  - Type: `httpRequest`  
  - Role: GET request to fetch the current status of the prediction job using its unique `id`.  
  - Configuration:  
    - URL constructed dynamically: `https://api.replicate.com/v1/predictions/{{ prediction_id }}`  
    - Authorization header with bearer token from "Set API Token".  
    - Response JSON, never error on HTTP errors.  
  - Inputs: From "Wait 5s" and "Wait 10s"  
  - Outputs: To "Is Complete?"  
  - Edge Cases:  
    - API downtime or connectivity issues  
    - Invalid prediction id causing 404 errors  
  - Version: 4.1

- **Is Complete?**  
  - Type: `if`  
  - Role: Checks if the prediction status equals `"succeeded"`.  
  - Configuration: Condition: `$json.status === "succeeded"`  
  - Inputs: From "Check Status"  
  - Outputs:  
    - True branch to "Success Response"  
    - False branch to "Has Failed?"  
  - Edge Cases: None  
  - Version: 2

- **Has Failed?**  
  - Type: `if`  
  - Role: Checks if the prediction status equals `"failed"`.  
  - Configuration: Condition: `$json.status === "failed"`  
  - Inputs: From "Is Complete?" (false branch)  
  - Outputs:  
    - True branch to "Error Response"  
    - False branch to "Wait 10s" (retry)  
  - Edge Cases: None  
  - Version: 2

- **Wait 10s**  
  - Type: `wait`  
  - Role: Delay between status polling retries when job is not complete or failed.  
  - Configuration: Fixed 10 seconds  
  - Inputs: From "Has Failed?" (false branch)  
  - Outputs: To "Check Status" (loop)  
  - Edge Cases: None  
  - Version: 1

- **Success Response**  
  - Type: `set`  
  - Role: Formats a structured success response object containing the image URL, prediction ID, status, and a success message.  
  - Configuration:  
    - Create a JSON object:  
      ```json
      {
        "success": true,
        "image_url": $json.output,
        "prediction_id": $json.id,
        "status": $json.status,
        "message": "Image generated successfully"
      }
      ```  
  - Inputs: From "Is Complete?" (true branch)  
  - Outputs: To "Display Result"  
  - Edge Cases: None  
  - Version: 3.3

- **Error Response**  
  - Type: `set`  
  - Role: Formats a structured error response containing error message, prediction ID, status, and failure message.  
  - Configuration:  
    - Create a JSON object:  
      ```json
      {
        "success": false,
        "error": $json.error || "Image generation failed",
        "prediction_id": $json.id,
        "status": $json.status,
        "message": "Failed to generate image"
      }
      ```  
  - Inputs: From "Has Failed?" (true branch)  
  - Outputs: To "Display Result"  
  - Edge Cases: None  
  - Version: 3.3

- **Display Result**  
  - Type: `set`  
  - Role: Wraps the final response object into a single output property `final_result`.  
  - Configuration: Assigns `final_result` with the contents of the `response` field from previous node.  
  - Inputs: From "Success Response" and "Error Response"  
  - Outputs: Workflow end or further consumption  
  - Edge Cases: None  
  - Version: 3.3

---

#### 2.4 User Guidance and Metadata

**Overview:**  
Sticky notes provide extended documentation and support information for users operating or modifying the workflow.

**Nodes Involved:**  
- Sticky Note9  
- Sticky Note4

**Node Details:**

- **Sticky Note9**  
  - Type: `stickyNote`  
  - Role: Displays contact information and links for support and tutorials.  
  - Content Highlights:
    - Title: "CLIP GENERATOR"
    - Contact email: Yaron@nofluff.online
    - YouTube and LinkedIn links for tutorials and tips.  
  - Position: Top-left corner for visibility.

- **Sticky Note4**  
  - Type: `stickyNote`  
  - Role: Comprehensive documentation including:
    - Model overview (owner, type, API endpoint)
    - Parameter description (required and optional)
    - Workflow component explanation
    - Benefits, quick start, and troubleshooting
    - External resource links (Replicate API docs, n8n docs)  
  - Position: Left side, large and detailed.

---

### 3. Summary Table

| Node Name           | Node Type          | Functional Role                         | Input Node(s)              | Output Node(s)              | Sticky Note                                                                                          |
|---------------------|--------------------|---------------------------------------|----------------------------|-----------------------------|----------------------------------------------------------------------------------------------------|
| Manual Trigger      | manualTrigger      | Starts workflow execution             |                            | Set API Token               |                                                                                                |
| Set API Token       | set                | Stores API token                      | Manual Trigger             | Set Image Parameters        |                                                                                                |
| Set Image Parameters| set                | Prepares input parameters             | Set API Token              | Create Image Prediction     |                                                                                                |
| Create Image Prediction | httpRequest       | Sends prediction request to API       | Set Image Parameters       | Log Request                |                                                                                                |
| Log Request         | code               | Logs request details                  | Create Image Prediction    | Wait 5s                    |                                                                                                |
| Wait 5s             | wait               | Waits 5 seconds before status check  | Log Request                | Check Status               |                                                                                                |
| Check Status        | httpRequest        | Fetches prediction job status         | Wait 5s, Wait 10s          | Is Complete?               |                                                                                                |
| Is Complete?        | if                 | Checks if prediction succeeded        | Check Status               | Success Response, Has Failed?|                                                                                                |
| Has Failed?         | if                 | Checks if prediction failed            | Is Complete?               | Error Response, Wait 10s    |                                                                                                |
| Wait 10s            | wait               | Waits 10 seconds before retrying      | Has Failed?                | Check Status               |                                                                                                |
| Success Response    | set                | Builds success output JSON             | Is Complete?               | Display Result             |                                                                                                |
| Error Response      | set                | Builds error output JSON               | Has Failed?                | Display Result             |                                                                                                |
| Display Result      | set                | Wraps final output response            | Success Response, Error Response |                           |                                                                                                |
| Sticky Note9        | stickyNote         | Support contact and tutorial links    |                            |                             | Contact: Yaron@nofluff.online; YouTube: https://www.youtube.com/@YaronBeen/videos; LinkedIn: https://www.linkedin.com/in/yaronbeen/ |
| Sticky Note4        | stickyNote         | Detailed workflow and model documentation |                            |                             | Extensive usage guide, troubleshooting, and resource links including https://replicate.com/openai/clip |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Manual Trigger node:**  
   - Name: `Manual Trigger`  
   - Leave default settings.

3. **Add a Set node to store API Token:**  
   - Name: `Set API Token`  
   - Add a string variable `api_token` with value `"YOUR_REPLICATE_API_TOKEN"` (replace with your actual token).  
   - Connect `Manual Trigger` → `Set API Token`.

4. **Add a Set node to configure input parameters:**  
   - Name: `Set Image Parameters`  
   - Add three variables:  
     - `api_token` (string) set to `={{ $('Set API Token').item.json.api_token }}`  
     - `text` (string) set to `""` (empty string, user can customize)  
     - `image` (string) set to `"https://picsum.photos/512/512"` for testing  
   - Connect `Set API Token` → `Set Image Parameters`.

5. **Add an HTTP Request node to create a prediction:**  
   - Name: `Create Image Prediction`  
   - HTTP Method: POST  
   - URL: `https://api.replicate.com/v1/predictions`  
   - Headers:  
     - `Authorization`: `Bearer {{ $json.api_token }}`  
     - `Prefer`: `wait`  
   - Body Content Type: JSON  
   - JSON Body:  
     ```json
     {
       "version": "openai/clip:fd95fe35085b5b9a63d830d3126311ee6b32a7a976c78eb5f210a3a007bcdda6",
       "input": {
         "text": "{{ $json.text }}",
         "image": "{{ $json.image }}"
       }
     }
     ```  
   - Connect `Set Image Parameters` → `Create Image Prediction`.

6. **Add a Code node for logging:**  
   - Name: `Log Request`  
   - JavaScript Code:  
     ```javascript
     const data = $input.all()[0].json;
     console.log('openai/clip Request:', {
       timestamp: new Date().toISOString(),
       prediction_id: data.id,
       model_type: 'image'
     });
     return $input.all();
     ```  
   - Connect `Create Image Prediction` → `Log Request`.

7. **Add a Wait node:**  
   - Name: `Wait 5s`  
   - Wait time: 5 seconds  
   - Connect `Log Request` → `Wait 5s`.

8. **Add an HTTP Request node to check prediction status:**  
   - Name: `Check Status`  
   - HTTP Method: GET  
   - URL: `https://api.replicate.com/v1/predictions/{{ $('Create Image Prediction').item.json.id }}`  
   - Headers:  
     - `Authorization`: `Bearer {{ $('Set API Token').item.json.api_token }}`  
   - Connect `Wait 5s` → `Check Status`.  
   - Also connect `Wait 10s` (created later) → `Check Status` to enable polling loop.

9. **Add an If node to check if prediction is complete:**  
   - Name: `Is Complete?`  
   - Condition: `$json.status === "succeeded"`  
   - Connect `Check Status` → `Is Complete?`.

10. **Add an If node to check if prediction failed:**  
    - Name: `Has Failed?`  
    - Condition: `$json.status === "failed"`  
    - Connect `Is Complete?` (false output) → `Has Failed?`.

11. **Add a Set node for success response:**  
    - Name: `Success Response`  
    - Assign a JSON object as `response`:  
      ```json
      {
        "success": true,
        "image_url": $json.output,
        "prediction_id": $json.id,
        "status": $json.status,
        "message": "Image generated successfully"
      }
      ```  
    - Connect `Is Complete?` (true output) → `Success Response`.

12. **Add a Set node for error response:**  
    - Name: `Error Response`  
    - Assign a JSON object as `response`:  
      ```json
      {
        "success": false,
        "error": $json.error || "Image generation failed",
        "prediction_id": $json.id,
        "status": $json.status,
        "message": "Failed to generate image"
      }
      ```  
    - Connect `Has Failed?` (true output) → `Error Response`.

13. **Add a Wait node for retry delay:**  
    - Name: `Wait 10s`  
    - Wait time: 10 seconds  
    - Connect `Has Failed?` (false output) → `Wait 10s`.

14. **Connect `Wait 10s` output back to `Check Status`** to enable polling loop.

15. **Add a Set node to finalize output:**  
    - Name: `Display Result`  
    - Assign `final_result` with the value of the `response` field from previous nodes.  
    - Connect both `Success Response` and `Error Response` → `Display Result`.

16. **Optional: Add Sticky Notes for documentation and support:**  
    - Add one sticky note with contact info, YouTube, LinkedIn links.  
    - Add a second sticky note with detailed model info, usage guide, troubleshooting, and external resource links (Replicate API and n8n docs).  
    - Position them for visibility.

17. **Ensure all connections match described flow and test the workflow by triggering manually.**

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| Contact for support: Yaron@nofluff.online | Workflow Sticky Note9 |
| YouTube tutorials: https://www.youtube.com/@YaronBeen/videos | Workflow Sticky Note9 |
| LinkedIn profile: https://www.linkedin.com/in/yaronbeen/ | Workflow Sticky Note9 |
| Model info and API docs: https://replicate.com/openai/clip | Sticky Note4 and documentation |
| Replicate API documentation: https://replicate.com/docs | Sticky Note4 |
| n8n official documentation: https://docs.n8n.io | Sticky Note4 |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.

---