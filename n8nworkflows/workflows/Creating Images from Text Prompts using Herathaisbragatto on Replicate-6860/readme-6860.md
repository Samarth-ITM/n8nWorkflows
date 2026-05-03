Creating Images from Text Prompts using Herathaisbragatto on Replicate

https://n8nworkflows.xyz/workflows/creating-images-from-text-prompts-using-herathaisbragatto-on-replicate-6860


# Creating Images from Text Prompts using Herathaisbragatto on Replicate

### 1. Workflow Overview

This workflow automates the creation of images from text prompts using the "herathaisbragatto" model hosted on Replicate. It is designed for users who want to generate artwork or “other” visual content based on descriptive inputs, leveraging Replicate's AI model API via n8n automation.

**Target Use Cases:**  
- Automated generation of images or creative visuals from textual descriptions  
- Integration into content pipelines where instant image generation is needed  
- Experimentation with AI image generation parameters for research or prototyping  

**Logical Blocks:**  

- **1.1 Input Initialization and Configuration**  
  Starts the workflow manually, sets the API credentials and model parameters.

- **1.2 Prediction Request Creation**  
  Sends a HTTP request to Replicate API to initiate image generation with the configured inputs.

- **1.3 Prediction Status Polling and Retry Logic**  
  Waits and repeatedly polls the prediction status until the image generation completes or fails. Includes handling for retries with waits.

- **1.4 Success and Error Handling**  
  Processes successful results or errors, formatting output response objects accordingly.

- **1.5 Logging and Monitoring**  
  Logs each prediction request details for monitoring and debugging.

- **1.6 Final Output Delivery**  
  Prepares and outputs the final structured response object for downstream use.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Input Initialization and Configuration

**Overview:**  
Begins the workflow execution via manual trigger and prepares all necessary inputs including authentication and model parameters.

**Nodes Involved:**  
- Manual Trigger  
- Set API Token  
- Set Other Parameters  

**Node Details:**

- **Manual Trigger**  
  - Type: Trigger  
  - Role: Starts the workflow manually by user interaction.  
  - Configuration: Default, no parameters.  
  - Inputs: None  
  - Outputs: Connects to "Set API Token" node.  
  - Edge cases: None.

- **Set API Token**  
  - Type: Set  
  - Role: Stores the Replicate API token used for authentication.  
  - Configuration: Assigns a string variable `api_token` with placeholder value `"YOUR_REPLICATE_API_TOKEN"` which must be replaced by the user.  
  - Inputs: From Manual Trigger  
  - Outputs: Connects to "Set Other Parameters" node.  
  - Edge cases: Missing or invalid token will cause authentication failures downstream.

- **Set Other Parameters**  
  - Type: Set  
  - Role: Defines and assigns all input parameters required by the model.  
  - Configuration: Assigns multiple parameters such as `mask`, `seed`, `image`, `model`, `width`, `height`, `prompt`, and many others. Parameters are assigned with default values, e.g. `prompt` = "Create something amazing", `width` = 512.  
  - Key expressions: References the `api_token` from previous node via expression.  
  - Inputs: From "Set API Token"  
  - Outputs: Connects to "Create Other Prediction" node.  
  - Edge cases: Parameter value type mismatches or missing mandatory fields like `prompt` may cause API errors.

---

#### 2.2 Prediction Request Creation

**Overview:**  
Submits a POST request to Replicate's prediction API to start the image generation process using the configured parameters.

**Nodes Involved:**  
- Create Other Prediction  

**Node Details:**

- **Create Other Prediction**  
  - Type: HTTP Request  
  - Role: Sends a POST request to `https://api.replicate.com/v1/predictions` with the model version and input parameters in JSON format.  
  - Configuration:  
    - URL: Replicate API endpoint for predictions.  
    - Headers: Authorization Bearer token from `api_token`.  
    - Body: JSON including the model version ID and all input parameters dynamically populated from previous node.  
    - Options: Waits synchronously for response (`Prefer: wait` header).  
  - Inputs: From "Set Other Parameters"  
  - Outputs: Connects to "Log Request" node.  
  - Edge cases: Possible failures include HTTP errors, invalid parameters, authentication errors, or API rate limits.

---

#### 2.3 Prediction Status Polling and Retry Logic

**Overview:**  
Handles asynchronous waiting and polling of prediction status until completion or failure, with retry delays.

**Nodes Involved:**  
- Log Request  
- Wait 5s  
- Check Status  
- Is Complete? (If)  
- Has Failed? (If)  
- Wait 10s  

**Node Details:**

- **Log Request**  
  - Type: Code  
  - Role: Logs prediction request details including timestamp, prediction ID, and model type for monitoring.  
  - Configuration: JavaScript code outputs log to console.  
  - Inputs: From "Create Other Prediction"  
  - Outputs: Connects to "Wait 5s"  
  - Edge cases: Logging failure does not block workflow.

- **Wait 5s**  
  - Type: Wait  
  - Role: Pauses workflow execution for 5 seconds before polling.  
  - Configuration: Wait duration set to 5 seconds.  
  - Inputs: From "Log Request"  
  - Outputs: Connects to "Check Status"  
  - Edge cases: None.

- **Check Status**  
  - Type: HTTP Request  
  - Role: Queries the Replicate API for the current status of the prediction by ID.  
  - Configuration: GET request to `https://api.replicate.com/v1/predictions/{id}` with auth header.  
  - Inputs: From "Wait 5s" or "Wait 10s"  
  - Outputs: Connects to "Is Complete?"  
  - Edge cases: HTTP errors, invalid ID, network issues.

- **Is Complete?** (If Node)  
  - Type: If  
  - Role: Checks if the prediction status equals `"succeeded"`.  
  - Inputs: From "Check Status"  
  - Outputs:  
    - True: to "Success Response"  
    - False: to "Has Failed?"  
  - Edge cases: Unexpected status values.

- **Has Failed?** (If Node)  
  - Type: If  
  - Role: Checks if the prediction status equals `"failed"`.  
  - Inputs: From "Is Complete?" (False branch)  
  - Outputs:  
    - True: to "Error Response"  
    - False: to "Wait 10s" (retry polling)  
  - Edge cases: May loop indefinitely if status is unknown or stuck.

- **Wait 10s**  
  - Type: Wait  
  - Role: Waits 10 seconds before re-polling the status after a failure check.  
  - Inputs: From "Has Failed?" (False branch)  
  - Outputs: Connects back to "Check Status"  
  - Edge cases: None.

---

#### 2.4 Success and Error Handling

**Overview:**  
Handles the final output formatting for both success and failure cases.

**Nodes Involved:**  
- Success Response  
- Error Response  
- Display Result  

**Node Details:**

- **Success Response**  
  - Type: Set  
  - Role: Constructs a success JSON object containing the prediction ID, status, output URL(s), and a success message.  
  - Inputs: From "Is Complete?" (True branch)  
  - Outputs: Connects to "Display Result"  
  - Edge cases: Missing or malformed output URLs.

- **Error Response**  
  - Type: Set  
  - Role: Constructs an error JSON object containing error messages, prediction ID, status, and a failure message.  
  - Inputs: From "Has Failed?" (True branch)  
  - Outputs: Connects to "Display Result"  
  - Edge cases: Generic fallback if no error details are returned.

- **Display Result**  
  - Type: Set  
  - Role: Sets the final output object named `final_result` containing either success or error response for downstream usage.  
  - Inputs: From both Success and Error Response nodes  
  - Outputs: None (end node)  
  - Edge cases: None.

---

#### 2.5 Logging and Monitoring (Integrated in 2.3)

- The "Log Request" node logs essential details for each prediction request to the console, aiding in development and troubleshooting.

---

### 3. Summary Table

| Node Name            | Node Type          | Functional Role                        | Input Node(s)                | Output Node(s)              | Sticky Note                                                                                           |
|----------------------|--------------------|-------------------------------------|-----------------------------|-----------------------------|-----------------------------------------------------------------------------------------------------|
| Manual Trigger       | Manual Trigger      | Starts the workflow execution       |                             | Set API Token                |                                                                                                     |
| Set API Token        | Set                | Stores Replicate API Token           | Manual Trigger              | Set Other Parameters         |                                                                                                     |
| Set Other Parameters | Set                | Assigns model input parameters       | Set API Token               | Create Other Prediction      |                                                                                                     |
| Create Other Prediction | HTTP Request       | Sends generation request to Replicate | Set Other Parameters        | Log Request                 |                                                                                                     |
| Log Request          | Code               | Logs prediction details              | Create Other Prediction     | Wait 5s                     |                                                                                                     |
| Wait 5s              | Wait               | Waits 5 seconds before status check | Log Request                 | Check Status                |                                                                                                     |
| Check Status         | HTTP Request       | Polls prediction status              | Wait 5s, Wait 10s           | Is Complete?                |                                                                                                     |
| Is Complete?         | If                 | Checks if prediction succeeded      | Check Status                | Success Response, Has Failed? |                                                                                                     |
| Has Failed?          | If                 | Checks if prediction failed         | Is Complete?                | Error Response, Wait 10s    |                                                                                                     |
| Wait 10s             | Wait               | Waits 10 seconds for retry           | Has Failed?                 | Check Status                |                                                                                                     |
| Success Response     | Set                | Formats success JSON response        | Is Complete? (True branch)  | Display Result              |                                                                                                     |
| Error Response       | Set                | Formats error JSON response          | Has Failed? (True branch)   | Display Result              |                                                                                                     |
| Display Result       | Set                | Outputs final response object        | Success Response, Error Response |                           |                                                                                                     |
| Sticky Note9         | Sticky Note        | Workflow author and contact info     |                             |                             | =======================================<br> HERATHAISBRAGATTO GENERATOR<br> For any questions or support: Yaron@nofluff.online<br> YouTube: https://www.youtube.com/@YaronBeen/videos<br> LinkedIn: https://www.linkedin.com/in/yaronbeen/ |
| Sticky Note4         | Sticky Note        | Detailed workflow and model documentation |                             |                             | Detailed notes describing model, parameters, workflow logic, troubleshooting, and links:<br> https://replicate.com/digitalhera/herathaisbragatto<br> https://replicate.com/docs<br> https://docs.n8n.io |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node**  
   - Purpose: Start the workflow manually.  
   - No configuration needed.

2. **Add a Set node named "Set API Token"**  
   - Purpose: Store your Replicate API token.  
   - Add a string field named `api_token` with default value `"YOUR_REPLICATE_API_TOKEN"` (replace with your actual token).  
   - Connect Manual Trigger → Set API Token.

3. **Add a Set node named "Set Other Parameters"**  
   - Purpose: Define all input parameters for the model.  
   - Add fields (type and default values):  
     - api_token (string): use expression to copy from `Set API Token` node.  
     - mask (string): `"https://via.placeholder.com/512x512/000000/FFFFFF.png"`  
     - seed (number): `-1`  
     - image (string): `"https://picsum.photos/512/512"`  
     - model (string): `"dev"`  
     - width (number): `512`  
     - height (number): `512`  
     - prompt (string): `"Create something amazing"`  
     - go_fast (boolean): `false`  
     - extra_lora (string): `""`  
     - lora_scale (number): `1`  
     - megapixels (string): `"1"`  
     - num_outputs (number): `1`  
     - aspect_ratio (string): `"1:1"`  
     - output_format (string): `"webp"`  
     - guidance_scale (number): `3`  
     - output_quality (number): `80`  
     - prompt_strength (number): `0.8`  
     - extra_lora_scale (number): `1`  
     - replicate_weights (string): `""`  
     - num_inference_steps (number): `28`  
     - disable_safety_checker (boolean): `false`  
   - Connect Set API Token → Set Other Parameters.

4. **Add an HTTP Request node named "Create Other Prediction"**  
   - Purpose: Send POST request to Replicate to create prediction.  
   - Settings:  
     - Method: POST  
     - URL: `https://api.replicate.com/v1/predictions`  
     - Body Content Type: JSON  
     - Body JSON (use expression to map all parameters from "Set Other Parameters" JSON):  
       ```json
       {
         "version": "digitalhera/herathaisbragatto:68f10949d36128d85f64e189dc0d9efd112bb2593245daadde6616023cd134ad",
         "input": {
           "mask": "{{ $json.mask }}",
           "seed": {{ $json.seed }},
           "image": "{{ $json.image }}",
           "model": "{{ $json.model }}",
           "width": {{ $json.width }},
           "height": {{ $json.height }},
           "prompt": "{{ $json.prompt }}",
           "go_fast": {{ $json.go_fast }},
           "extra_lora": "{{ $json.extra_lora }}",
           "lora_scale": {{ $json.lora_scale }},
           "megapixels": "{{ $json.megapixels }}",
           "num_outputs": {{ $json.num_outputs }},
           "aspect_ratio": "{{ $json.aspect_ratio }}",
           "output_format": "{{ $json.output_format }}",
           "guidance_scale": {{ $json.guidance_scale }},
           "output_quality": {{ $json.output_quality }},
           "prompt_strength": {{ $json.prompt_strength }},
           "extra_lora_scale": {{ $json.extra_lora_scale }},
           "replicate_weights": "{{ $json.replicate_weights }}",
           "num_inference_steps": {{ $json.num_inference_steps }},
           "disable_safety_checker": {{ $json.disable_safety_checker }}
         }
       }
       ```
     - Headers:  
       - Authorization: `Bearer {{ $json.api_token }}`  
       - Prefer: `wait`  
     - Response Format: JSON  
   - Connect Set Other Parameters → Create Other Prediction.

5. **Add a Code node named "Log Request"**  
   - Purpose: Log prediction request details.  
   - JavaScript Code:  
     ```javascript
     const data = $input.all()[0].json;
     console.log('digitalhera/herathaisbragatto Request:', {
       timestamp: new Date().toISOString(),
       prediction_id: data.id,
       model_type: 'other'
     });
     return $input.all();
     ```  
   - Connect Create Other Prediction → Log Request.

6. **Add a Wait node named "Wait 5s"**  
   - Purpose: Wait 5 seconds before polling status.  
   - Configure to wait 5 seconds.  
   - Connect Log Request → Wait 5s.

7. **Add an HTTP Request node named "Check Status"**  
   - Purpose: Query the prediction status.  
   - Method: GET  
   - URL: `https://api.replicate.com/v1/predictions/{{ $('Create Other Prediction').item.json.id }}` (use expression to insert prediction ID)  
   - Headers: Authorization Bearer token from "Set API Token" node.  
   - Response Format: JSON  
   - Connect Wait 5s → Check Status and also Wait 10s (see step 11) → Check Status.

8. **Add an If node named "Is Complete?"**  
   - Purpose: Check if status is "succeeded".  
   - Condition: `$json.status == "succeeded"`  
   - Connect Check Status → Is Complete?.

9. **Add an If node named "Has Failed?"**  
   - Purpose: Check if status is "failed".  
   - Condition: `$json.status == "failed"`  
   - Connect Is Complete? (False output) → Has Failed?.

10. **Add a Set node named "Success Response"**  
    - Purpose: Format success response JSON.  
    - Assign an object `response` with fields:  
      - success: true  
      - result_url: `$json.output`  
      - prediction_id: `$json.id`  
      - status: `$json.status`  
      - message: "Other generated successfully"  
    - Connect Is Complete? (True output) → Success Response.

11. **Add a Set node named "Error Response"**  
    - Purpose: Format error response JSON.  
    - Assign an object `response` with fields:  
      - success: false  
      - error: `$json.error` or fallback "Other generation failed"  
      - prediction_id: `$json.id`  
      - status: `$json.status`  
      - message: "Failed to generate other"  
    - Connect Has Failed? (True output) → Error Response.

12. **Add a Wait node named "Wait 10s"**  
    - Purpose: Wait 10 seconds before retrying status check.  
    - Configure wait for 10 seconds.  
    - Connect Has Failed? (False output) → Wait 10s.

13. **Add a Set node named "Display Result"**  
    - Purpose: Set a final output object `final_result` with the response from either Success or Error nodes.  
    - Assign `final_result` to the incoming `response` object.  
    - Connect Success Response → Display Result and Error Response → Display Result.

14. **Verify all connections as follows:**  
    - Manual Trigger → Set API Token → Set Other Parameters → Create Other Prediction → Log Request → Wait 5s → Check Status → Is Complete?  
    - Is Complete? True → Success Response → Display Result  
    - Is Complete? False → Has Failed?  
    - Has Failed? True → Error Response → Display Result  
    - Has Failed? False → Wait 10s → Check Status (loop back)

15. **Configure credentials:**  
    - Ensure HTTP Request nodes use proper authentication headers with the Replicate API token.  
    - No other credentials needed.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | Context or Link                                                        |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------|
| =======================================<br> HERATHAISBRAGATTO GENERATOR<br>For any questions or support, please contact:<br>Yaron@nofluff.online<br>Explore more tips and tutorials here:<br>- YouTube: https://www.youtube.com/@YaronBeen/videos<br>- LinkedIn: https://www.linkedin.com/in/yaronbeen/<br>=======================================                                                                                                                                                                                                                                                                                                    | Sticky Note with author contact and social links                      |
| This workflow is powered by Replicate API and n8n Automation. Model documentation and API references can be found at:<br>- Model: https://replicate.com/digitalhera/herathaisbragatto<br>- Replicate API Docs: https://replicate.com/docs<br>- n8n Docs: https://docs.n8n.io<br><br>Recommended best practices include securing API tokens, testing default parameters first, monitoring usage and billing, and validating parameter types.<br><br>Common issues include invalid tokens, parameter validation errors, generation timeouts, and output format mismatches. | Sticky Note with detailed workflow overview, parameter guide, and links |

---

**Disclaimer:** The provided text and workflow originate exclusively from an automated workflow created using n8n, an integration and automation tool. The processing strictly complies with current content policies and does not contain any illegal, offensive, or protected content. All data handled is legal and public.