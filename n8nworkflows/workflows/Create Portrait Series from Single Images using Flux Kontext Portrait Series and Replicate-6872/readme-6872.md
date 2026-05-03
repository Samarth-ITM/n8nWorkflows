Create Portrait Series from Single Images using Flux Kontext Portrait Series and Replicate

https://n8nworkflows.xyz/workflows/create-portrait-series-from-single-images-using-flux-kontext-portrait-series-and-replicate-6872


# Create Portrait Series from Single Images using Flux Kontext Portrait Series and Replicate

### 1. Workflow Overview

This workflow automates the generation of a series of portrait-style images derived from a single input image using the Flux Kontext Portrait Series AI model hosted on Replicate. It is designed for users who want to create multiple poses or variations of a portrait from one photo, with configurable parameters such as background color, number of images, output format, and safety tolerance.

The workflow is logically divided into the following blocks:

- **1.1 Input Initialization and Authentication:** Receives manual trigger, sets the Replicate API token, and defines image generation parameters.
- **1.2 Image Generation Request:** Sends a prediction request to Replicate’s API with the configured parameters to start the portrait series generation.
- **1.3 Status Polling Loop:** Waits and polls the Replicate API for prediction status, repeating until the generation succeeds or fails.
- **1.4 Result Handling:** Processes successful or failed generation results, preparing structured JSON responses.
- **1.5 Logging and Monitoring:** Logs prediction requests for debugging and monitoring purposes.
- **1.6 User Feedback:** Outputs final results in a consistent format for downstream use or response.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Initialization and Authentication

**Overview:**  
This block sets up the initial trigger for the workflow, configures the API token for authentication with Replicate, and defines all input parameters necessary to generate the portrait series.

**Nodes Involved:**  
- Manual Trigger  
- Set API Token  
- Set Image Parameters  

**Node Details:**

- **Manual Trigger**  
  - **Type:** Manual trigger node  
  - **Role:** Initiates the workflow manually by user action  
  - **Config:** No parameters; simply starts execution  
  - **Inputs:** None  
  - **Outputs:** Connected to "Set API Token"  
  - **Edge Cases:** None; manual start only

- **Set API Token**  
  - **Type:** Set node  
  - **Role:** Stores the Replicate API token as a workflow variable  
  - **Config:** Assigns string variable `api_token` with placeholder `"YOUR_REPLICATE_API_TOKEN"` which must be replaced by the user with a valid token  
  - **Inputs:** Manual Trigger output  
  - **Outputs:** Connected to "Set Image Parameters"  
  - **Edge Cases:** Missing or invalid token will cause authentication failures downstream

- **Set Image Parameters**  
  - **Type:** Set node  
  - **Role:** Configures the input parameters for the portrait series generation  
  - **Config:** Copies `api_token` from previous node; sets:  
    - `background`: `"white"`  
    - `num_images`: `4` (number of portraits to generate)  
    - `input_image`: `"https://picsum.photos/512/512"` (default sample image URL)  
    - `output_format`: `"png"`  
    - `randomize_images`: `false` (whether to randomize poses)  
    - `safety_tolerance`: `2` (maximum permissive safety level)  
  - **Inputs:** From "Set API Token"  
  - **Outputs:** To "Create Image Prediction"  
  - **Edge Cases:** Input image URL must be valid and accessible; incorrect types or missing parameters may cause API errors

---

#### 2.2 Image Generation Request

**Overview:**  
This block sends the image generation request to the Replicate API with all configured parameters, triggering the AI model to start generating the portrait series.

**Nodes Involved:**  
- Create Image Prediction  
- Log Request  

**Node Details:**

- **Create Image Prediction**  
  - **Type:** HTTP Request node  
  - **Role:** Sends a POST request to Replicate API endpoint `/v1/predictions` to create a new prediction  
  - **Config:**  
    - URL: `https://api.replicate.com/v1/predictions`  
    - Method: POST  
    - JSON body containing the model version and inputs reflecting all parameters set previously  
    - Headers: Authorization using Bearer token from `api_token`, "Prefer: wait" to ask for synchronous response if possible  
    - Response: JSON expected, no error thrown on HTTP error to allow graceful handling  
  - **Inputs:** From "Set Image Parameters"  
  - **Outputs:** To "Log Request"  
  - **Expressions:** Parameter values interpolated from previous node using expressions  
  - **Edge Cases:**  
    - Network errors  
    - Authentication failures (invalid token)  
    - API rate limiting or timeouts  
    - Model version deprecation or unavailability

- **Log Request**  
  - **Type:** Code node  
  - **Role:** Logs prediction request details for monitoring and debugging  
  - **Config:** JavaScript code outputs timestamp, prediction ID, and model type to console  
  - **Inputs:** From "Create Image Prediction"  
  - **Outputs:** To "Wait 5s"  
  - **Edge Cases:** Console log failures do not affect workflow, but missing logs reduce traceability

---

#### 2.3 Status Polling Loop

**Overview:**  
After the initial request, this block waits and repeatedly checks the status of the image generation prediction until it either completes successfully or fails.

**Nodes Involved:**  
- Wait 5s  
- Check Status  
- Is Complete?  
- Has Failed?  
- Wait 10s  

**Node Details:**

- **Wait 5s**  
  - **Type:** Wait node  
  - **Role:** Pauses workflow 5 seconds before checking status  
  - **Config:** Wait 5 seconds  
  - **Inputs:** From "Log Request"  
  - **Outputs:** To "Check Status"  
  - **Edge Cases:** If system sleep or delay occurs, may add latency

- **Check Status**  
  - **Type:** HTTP Request node  
  - **Role:** Performs GET request to Replicate API to retrieve current prediction status by ID  
  - **Config:**  
    - URL dynamically built using prediction ID from "Create Image Prediction"  
    - Authorization header with Bearer token  
    - Returns JSON with current status  
  - **Inputs:** From "Wait 5s" and "Wait 10s"  
  - **Outputs:** To "Is Complete?"  
  - **Edge Cases:** Network failures, token expiry, invalid prediction ID

- **Is Complete?**  
  - **Type:** If node  
  - **Role:** Checks if prediction status equals `"succeeded"` to determine completion  
  - **Config:** Condition: `$json.status === "succeeded"`  
  - **Inputs:** From "Check Status"  
  - **Outputs:**  
    - True branch: to "Success Response"  
    - False branch: to "Has Failed?"  
  - **Edge Cases:** Status might be other unexpected values; careful to handle all

- **Has Failed?**  
  - **Type:** If node  
  - **Role:** Checks if prediction status equals `"failed"` to detect failure  
  - **Config:** Condition: `$json.status === "failed"`  
  - **Inputs:** From "Is Complete?" (false branch)  
  - **Outputs:**  
    - True branch: to "Error Response"  
    - False branch: to "Wait 10s" (retry polling)  
  - **Edge Cases:** Other statuses like "processing" or "starting" handled by retry loop

- **Wait 10s**  
  - **Type:** Wait node  
  - **Role:** Pauses workflow 10 seconds before next status check, implementing retry delay  
  - **Config:** Wait 10 seconds  
  - **Inputs:** From "Has Failed?" (false branch)  
  - **Outputs:** To "Check Status" (loop back)  
  - **Edge Cases:** Long wait may delay response; infinite loop possible if prediction hangs

---

#### 2.4 Result Handling

**Overview:**  
This block prepares structured JSON responses based on success or failure outcomes for downstream consumption or API response formatting.

**Nodes Involved:**  
- Success Response  
- Error Response  
- Display Result  

**Node Details:**

- **Success Response**  
  - **Type:** Set node  
  - **Role:** Constructs a success response object containing:  
    - success: `true`  
    - image_url: URL(s) from prediction output  
    - prediction_id and status  
    - message: "Image generated successfully"  
  - **Inputs:** From "Is Complete?" (true branch)  
  - **Outputs:** To "Display Result"  
  - **Edge Cases:** Missing output URLs could cause incomplete responses

- **Error Response**  
  - **Type:** Set node  
  - **Role:** Constructs an error response object containing:  
    - success: `false`  
    - error message from API or generic string  
    - prediction_id and status  
    - message: "Failed to generate image"  
  - **Inputs:** From "Has Failed?" (true branch)  
  - **Outputs:** To "Display Result"  
  - **Edge Cases:** Errors may be vague; ensure meaningful error messages

- **Display Result**  
  - **Type:** Set node  
  - **Role:** Finalizes the JSON response under `final_result` variable for output or further handling  
  - **Inputs:** From both Success and Error Response nodes  
  - **Outputs:** (No downstream node shown; this is terminal for the response)  
  - **Edge Cases:** None specific; acts as output wrapper

---

#### 2.5 Logging and Monitoring (Integrated in 2.2 block)

- See "Log Request" node details in block 2.2.

---

#### 2.6 User Feedback and Documentation

**Nodes Involved:**  
- Sticky Note9  
- Sticky Note4  

**Node Details:**

- **Sticky Note9**  
  - **Type:** Sticky note  
  - **Content:** Contains contact info and links for support and tutorials  
  - **Usage:** Reference for users seeking help or additional resources  

- **Sticky Note4**  
  - **Type:** Sticky note  
  - **Content:** Extensive documentation embedded in the workflow describing model overview, parameters, workflow components, quick start, troubleshooting, and external links  
  - **Usage:** Acts as self-contained documentation for users and developers  

---

### 3. Summary Table

| Node Name             | Node Type           | Functional Role                           | Input Node(s)           | Output Node(s)          | Sticky Note                                                |
|-----------------------|---------------------|-----------------------------------------|-------------------------|-------------------------|------------------------------------------------------------|
| Manual Trigger        | Manual Trigger      | Starts the workflow manually             | None                    | Set API Token            |                                                            |
| Set API Token         | Set                 | Stores API token for authentication     | Manual Trigger          | Set Image Parameters     |                                                            |
| Set Image Parameters  | Set                 | Defines image generation parameters      | Set API Token           | Create Image Prediction  |                                                            |
| Create Image Prediction| HTTP Request       | Sends generation request to Replicate    | Set Image Parameters    | Log Request              |                                                            |
| Log Request           | Code                | Logs prediction details for monitoring  | Create Image Prediction | Wait 5s                  |                                                            |
| Wait 5s               | Wait                | Waits 5 seconds before status check     | Log Request             | Check Status             |                                                            |
| Check Status          | HTTP Request        | Queries prediction status from API      | Wait 5s, Wait 10s       | Is Complete?             |                                                            |
| Is Complete?          | If                  | Checks if prediction succeeded          | Check Status            | Success Response, Has Failed? |                                                        |
| Has Failed?           | If                  | Checks if prediction failed              | Is Complete?            | Error Response, Wait 10s |                                                            |
| Wait 10s              | Wait                | Waits 10 seconds before retrying status | Has Failed?             | Check Status             |                                                            |
| Success Response      | Set                 | Builds success JSON response             | Is Complete? (true)     | Display Result           |                                                            |
| Error Response        | Set                 | Builds error JSON response               | Has Failed? (true)      | Display Result           |                                                            |
| Display Result        | Set                 | Finalizes response output                | Success Response, Error Response | None            |                                                            |
| Sticky Note9          | Sticky Note         | Support contact and resources            | None                    | None                    | Contact info and tutorial links                             |
| Sticky Note4          | Sticky Note         | Full workflow documentation              | None                    | None                    | Comprehensive documentation and parameter guide            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Manual Trigger node:**
   - Leave default settings.
   - This node starts the workflow when the user runs it manually.

3. **Add a Set node named "Set API Token":**
   - Add a string variable named `api_token`.
   - Set its value to your Replicate API token (replace `"YOUR_REPLICATE_API_TOKEN"`).
   - Connect the Manual Trigger node output to this node.

4. **Add a Set node named "Set Image Parameters":**
   - Assign the following variables:  
     - `api_token`: Expression referencing `Set API Token` node’s `api_token` (`={{ $('Set API Token').item.json.api_token }}`)  
     - `background`: `"white"` (string)  
     - `num_images`: `4` (number)  
     - `input_image`: `"https://picsum.photos/512/512"` (string; default image URL)  
     - `output_format`: `"png"` (string)  
     - `randomize_images`: `false` (boolean)  
     - `safety_tolerance`: `2` (number)  
   - Connect "Set API Token" output to this node.

5. **Add an HTTP Request node named "Create Image Prediction":**
   - Method: POST  
   - URL: `https://api.replicate.com/v1/predictions`  
   - Authentication: Not configured here; use header instead  
   - Headers:  
     - `Authorization`: `=Bearer {{ $json.api_token }}`  
     - `Prefer`: `wait`  
   - Body Content Type: JSON  
   - Body JSON (use expressions):  
     ```json
     {
       "version": "flux-kontext-apps/portrait-series:a7cb507cb19b970483f0b68f598d24b6d6a63e996fc972634a5b39b4d067e706",
       "input": {
         "background": "{{ $json.background }}",
         "num_images": {{ $json.num_images }},
         "input_image": "{{ $json.input_image }}",
         "output_format": "{{ $json.output_format }}",
         "randomize_images": {{ $json.randomize_images }},
         "safety_tolerance": {{ $json.safety_tolerance }}
       }
     }
     ```
   - Connect "Set Image Parameters" output to this node.

6. **Add a Code node named "Log Request":**
   - JavaScript code:  
     ```javascript
     const data = $input.all()[0].json;
     console.log('flux-kontext-apps/portrait-series Request:', {
       timestamp: new Date().toISOString(),
       prediction_id: data.id,
       model_type: 'image'
     });
     return $input.all();
     ```
   - Connect "Create Image Prediction" output to this node.

7. **Add a Wait node named "Wait 5s":**
   - Wait time: 5 seconds  
   - Connect "Log Request" output to this node.

8. **Add an HTTP Request node named "Check Status":**
   - Method: GET  
   - URL (expression): `=https://api.replicate.com/v1/predictions/{{ $('Create Image Prediction').item.json.id }}`  
   - Headers:  
     - `Authorization`: `=Bearer {{ $('Set API Token').item.json.api_token }}`  
   - Connect "Wait 5s" output to this node.

9. **Add an If node named "Is Complete?":**
   - Condition: Check if `$json.status === "succeeded"`  
   - Connect "Check Status" output to this node.

10. **Add an If node named "Has Failed?":**
    - Condition: Check if `$json.status === "failed"`  
    - Connect "Is Complete?" false output to this node.

11. **Add a Set node named "Success Response":**
    - Assign variable `response` (object) with:  
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

12. **Add a Set node named "Error Response":**
    - Assign variable `response` (object) with:  
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

13. **Add a Wait node named "Wait 10s":**
    - Wait time: 10 seconds  
    - Connect "Has Failed?" false output to this node.

14. **Connect "Wait 10s" output to "Check Status" node to form a polling loop.**

15. **Add a Set node named "Display Result":**
    - Assign variable `final_result` with the content of the `response` variable from either "Success Response" or "Error Response".  
    - Connect both "Success Response" and "Error Response" outputs to this node.

16. **(Optional) Add Sticky Note nodes:**
    - Add two sticky notes with the detailed documentation and support information as per the original workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                   | Context or Link                                                                                     |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Contact for questions or support: Yaron@nofluff.online. Explore tips and tutorials on YouTube and LinkedIn (links included in Sticky Note).                                                                                                                                                                                  | Sticky Note9                                                                                       |
| Comprehensive documentation embedded in workflow includes model overview, parameter explanations, workflow breakdown, quick start instructions, and troubleshooting tips.                                                                                                                                                        | Sticky Note4                                                                                       |
| Model Owner: flux-kontext-apps; Model Name: portrait-series; API Endpoint: https://api.replicate.com/v1/predictions.                                                                                                                                                                                                           | Workflow Metadata and Sticky Note4                                                                |
| Recommended to replace placeholder API token with a valid Replicate API token obtained at https://replicate.com, and to monitor API usage and billing.                                                                                                                                                                      | Sticky Note4 and general best practices                                                           |
| Parameters such as `input_image` must be a valid URL to an image in jpeg, png, gif, or webp format.                                                                                                                                                                                                                             | Parameter guide in Sticky Note4                                                                    |
| Workflow includes robust error handling and retry logic to accommodate API delays and failures without crashing.                                                                                                                                                                                                              | Workflow design notes                                                                              |
| The workflow is inactive by default (`active: false`); activate it before use.                                                                                                                                                                                                                                                 | Metadata                                                                                          |

---

**Disclaimer:**  
The provided text is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected material. All data handled is legal and public.