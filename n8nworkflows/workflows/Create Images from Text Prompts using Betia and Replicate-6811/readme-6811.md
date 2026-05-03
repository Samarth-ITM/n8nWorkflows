Create Images from Text Prompts using Betia and Replicate

https://n8nworkflows.xyz/workflows/create-images-from-text-prompts-using-betia-and-replicate-6811


# Create Images from Text Prompts using Betia and Replicate

### 1. Workflow Overview

This workflow automates the generation of images from text prompts using the "Betia" model hosted on the Replicate API. It is designed for users who want to create customized images by submitting descriptive prompts and optional parameters controlling the image style and generation process. The workflow covers authentication, parameter setup, prediction creation, status tracking, error handling, and result delivery.

**Logical Blocks:**

- **1.1 Input Reception and Initialization:** User-triggered start and initial API token setup.
- **1.2 Parameter Configuration:** Setting all required and optional parameters for the image generation.
- **1.3 Prediction Request:** Sending the generation request to the Replicate API.
- **1.4 Status Polling and Monitoring:** Waiting and repeatedly checking the prediction status until completion or failure.
- **1.5 Result Handling:** Differentiating success and failure cases, delivering structured responses.
- **1.6 Logging and Output Preparation:** Logging request details and preparing the final output for downstream use or display.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Initialization

**Overview:**  
Starts the workflow manually and sets the required API token for authentication with the Replicate API.

**Nodes Involved:**  
- Manual Trigger  
- Set API Token

**Node Details:**

- **Manual Trigger**  
  - Type: manualTrigger  
  - Role: Initiates workflow execution on user command.  
  - Configuration: Default; no parameters needed.  
  - Inputs: None  
  - Outputs: Connects to "Set API Token"  
  - Edge Cases: None (manual start)  

- **Set API Token**  
  - Type: set  
  - Role: Defines the Replicate API token as a workflow variable.  
  - Configuration: Hardcoded string "YOUR_REPLICATE_API_TOKEN" (placeholder to be replaced).  
  - Key Variable: `api_token`  
  - Inputs: From Manual Trigger  
  - Outputs: Connects to "Set Other Parameters"  
  - Edge Cases: Missing or invalid API token will cause authentication failures downstream.

---

#### 2.2 Parameter Configuration

**Overview:**  
Sets all necessary input parameters for the model prediction including prompt, image size, model options, and generation control variables.

**Nodes Involved:**  
- Set Other Parameters

**Node Details:**

- **Set Other Parameters**  
  - Type: set  
  - Role: Populates the input JSON for the Replicate prediction request.  
  - Configuration:  
    - Copies `api_token` from previous node.  
    - Sets default values for mask, seed, image URL, model, width, height, prompt, and multiple generation tuning parameters such as `go_fast`, `lora_scale`, `num_outputs`, `aspect_ratio`, `output_format`, `guidance_scale`, and others.  
    - Prompt default: "Create something amazing"  
    - Seed default: -1 (randomized)  
  - Inputs: From "Set API Token"  
  - Outputs: Connects to "Create Other Prediction"  
  - Edge Cases: Incorrect parameter types or invalid URLs may cause API errors.

---

#### 2.3 Prediction Request

**Overview:**  
Sends a POST request to the Replicate API to start the image generation prediction based on the configured parameters.

**Nodes Involved:**  
- Create Other Prediction

**Node Details:**

- **Create Other Prediction**  
  - Type: httpRequest  
  - Role: Issues POST request to Replicate API `/v1/predictions` endpoint.  
  - Configuration:  
    - Uses JSON body with model version and input parameters extracted from previous node.  
    - Sets headers including `Authorization` with Bearer token and `Prefer: wait` for synchronous response waiting.  
    - Response expected as JSON.  
  - Inputs: From "Set Other Parameters"  
  - Outputs: Connects to "Log Request"  
  - Edge Cases: HTTP errors, authentication failures, API rate limits, or invalid input data.  
  - Version-specific: Uses Replicate API v1 specification.

---

#### 2.4 Status Polling and Monitoring

**Overview:**  
Implements a polling loop to wait and re-check the prediction status until it completes successfully or fails.

**Nodes Involved:**  
- Log Request  
- Wait 5s  
- Check Status  
- Is Complete?  
- Has Failed?  
- Wait 10s

**Node Details:**

- **Log Request**  
  - Type: code  
  - Role: Logs the prediction ID, timestamp, and model type for debugging and monitoring.  
  - Configuration: JavaScript code outputs console logs.  
  - Inputs: From "Create Other Prediction"  
  - Outputs: Connects to "Wait 5s"  
  - Edge Cases: Logging failures generally non-critical.

- **Wait 5s**  
  - Type: wait  
  - Role: Pauses workflow for 5 seconds before status check.  
  - Configuration: 5 seconds delay.  
  - Inputs: From "Log Request"  
  - Outputs: Connects to "Check Status"

- **Check Status**  
  - Type: httpRequest  
  - Role: GET request to `/v1/predictions/{id}` to fetch current status.  
  - Configuration:  
    - URL dynamically constructed using prediction ID from "Create Other Prediction".  
    - Authorization header with API token.  
    - JSON response expected.  
  - Inputs: From "Wait 5s" and "Wait 10s" (retry path)  
  - Outputs: Connects to "Is Complete?"

- **Is Complete?**  
  - Type: if  
  - Role: Checks if prediction status equals "succeeded".  
  - Configuration: Condition `status === "succeeded"`.  
  - Inputs: From "Check Status"  
  - Outputs:  
    - True: "Success Response"  
    - False: "Has Failed?"

- **Has Failed?**  
  - Type: if  
  - Role: Checks if prediction status equals "failed".  
  - Configuration: Condition `status === "failed"`.  
  - Inputs: From "Is Complete?" (false branch)  
  - Outputs:  
    - True: "Error Response"  
    - False: "Wait 10s"

- **Wait 10s**  
  - Type: wait  
  - Role: Delay for 10 seconds before retrying status check.  
  - Configuration: 10 seconds delay.  
  - Inputs: From "Has Failed?" (false branch)  
  - Outputs: Connects back to "Check Status" (forming a polling loop)

---

#### 2.5 Result Handling

**Overview:**  
Prepares structured JSON responses for both success and failure cases, including essential metadata and URLs.

**Nodes Involved:**  
- Success Response  
- Error Response  
- Display Result

**Node Details:**

- **Success Response**  
  - Type: set  
  - Role: Constructs a success JSON object with result URL, prediction ID, status, and message.  
  - Configuration: Uses values from prediction JSON output fields.  
  - Inputs: From "Is Complete?" (true branch)  
  - Outputs: Connects to "Display Result"

- **Error Response**  
  - Type: set  
  - Role: Constructs an error JSON object with error details, prediction ID, status, and message.  
  - Configuration: Uses error messages or defaults if none present.  
  - Inputs: From "Has Failed?" (true branch)  
  - Outputs: Connects to "Display Result"

- **Display Result**  
  - Type: set  
  - Role: Sets the final output variable `final_result` to the response object prepared above.  
  - Configuration: Direct mapping from previous node’s `response` field.  
  - Inputs: From both "Success Response" and "Error Response"  
  - Outputs: Workflow termination or downstream use.

---

#### 2.6 Logging and Documentation

**Overview:**  
Provides embedded sticky notes for documentation, usage instructions, and contact information.

**Nodes Involved:**  
- Sticky Note9  
- Sticky Note4

**Node Details:**

- **Sticky Note9**  
  - Type: stickyNote  
  - Role: Displays contact info and resource links for support.  
  - Content: Contact email, YouTube channel, LinkedIn profile.

- **Sticky Note4**  
  - Type: stickyNote  
  - Role: Comprehensive documentation of the workflow, model parameters, benefits, troubleshooting, and quick start guide.  
  - Content: Detailed instructions and references for users.

---

### 3. Summary Table

| Node Name             | Node Type          | Functional Role                  | Input Node(s)               | Output Node(s)             | Sticky Note                                                                                                                |
|-----------------------|--------------------|--------------------------------|-----------------------------|----------------------------|----------------------------------------------------------------------------------------------------------------------------|
| Manual Trigger        | manualTrigger      | Workflow entry point            | -                           | Set API Token               |                                                                                                                            |
| Set API Token         | set                | Defines API token               | Manual Trigger              | Set Other Parameters        |                                                                                                                            |
| Set Other Parameters  | set                | Defines model input parameters  | Set API Token               | Create Other Prediction     |                                                                                                                            |
| Create Other Prediction| httpRequest        | Initiates image generation      | Set Other Parameters        | Log Request                |                                                                                                                            |
| Log Request           | code               | Logs prediction details         | Create Other Prediction     | Wait 5s                    |                                                                                                                            |
| Wait 5s               | wait               | Delay before checking status    | Log Request                 | Check Status               |                                                                                                                            |
| Check Status          | httpRequest        | Polls prediction status         | Wait 5s, Wait 10s           | Is Complete?               |                                                                                                                            |
| Is Complete?          | if                 | Checks if prediction succeeded  | Check Status                | Success Response, Has Failed? |                                                                                                                            |
| Has Failed?           | if                 | Checks if prediction failed     | Is Complete? (false branch) | Error Response, Wait 10s   |                                                                                                                            |
| Wait 10s              | wait               | Delay before retrying status    | Has Failed? (false branch)  | Check Status               |                                                                                                                            |
| Success Response      | set                | Prepares success output         | Is Complete? (true branch)  | Display Result             |                                                                                                                            |
| Error Response        | set                | Prepares error output           | Has Failed? (true branch)   | Display Result             |                                                                                                                            |
| Display Result        | set                | Final output assignment         | Success Response, Error Response | -                    |                                                                                                                            |
| Sticky Note9          | stickyNote         | Support contact information     | -                           | -                          | =======================================\nBETIA GENERATOR\nContact:\nYaron@nofluff.online\nYouTube: https://www.youtube.com/@YaronBeen/videos\nLinkedIn: https://www.linkedin.com/in/yaronbeen/ |
| Sticky Note4          | stickyNote         | Full workflow documentation     | -                           | -                          | Detailed doc with model info, parameters, usage, and troubleshooting.                                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a "Manual Trigger" node:**  
   - No specific configuration needed. This starts the workflow manually.

3. **Add a "Set" node named "Set API Token":**  
   - Add a string field named `api_token`.  
   - Set its value to your actual Replicate API token (replace "YOUR_REPLICATE_API_TOKEN").

4. **Connect "Manual Trigger" output to "Set API Token" input.**

5. **Add a "Set" node named "Set Other Parameters":**  
   - Add the following fields with default values:  
     - `api_token`: reference `{{$node["Set API Token"].json["api_token"]}}`  
     - `mask`: "https://via.placeholder.com/512x512/000000/FFFFFF.png"  
     - `seed`: -1  
     - `image`: "https://picsum.photos/512/512"  
     - `model`: "dev"  
     - `width`: 512  
     - `height`: 512  
     - `prompt`: "Create something amazing"  
     - `go_fast`: false  
     - `extra_lora`: ""  
     - `lora_scale`: 1  
     - `megapixels`: "1"  
     - `num_outputs`: 1  
     - `aspect_ratio`: "1:1"  
     - `output_format`: "webp"  
     - `guidance_scale`: 3  
     - `output_quality`: 80  
     - `prompt_strength`: 0.8  
     - `extra_lora_scale`: 1  
     - `replicate_weights`: ""  
     - `num_inference_steps`: 28  
     - `disable_safety_checker`: false

6. **Connect "Set API Token" output to "Set Other Parameters" input.**

7. **Add an "HTTP Request" node named "Create Other Prediction":**  
   - Method: POST  
   - URL: https://api.replicate.com/v1/predictions  
   - Authentication: None (token passed in header)  
   - Headers:  
     - Authorization: `Bearer {{$json.api_token}}`  
     - Prefer: wait  
   - Body Content Type: JSON  
   - JSON Body: Use expressions to map input parameters as follows:  
     ```json
     {
       "version": "izzaanel/betia:cf6ebd6365656db3e6f55ab66444a8c144e960298905dadebb976816ed62451b",
       "input": {
         "mask": "{{$json.mask}}",
         "seed": {{$json.seed}},
         "image": "{{$json.image}}",
         "model": "{{$json.model}}",
         "width": {{$json.width}},
         "height": {{$json.height}},
         "prompt": "{{$json.prompt}}",
         "go_fast": {{$json.go_fast}},
         "extra_lora": "{{$json.extra_lora}}",
         "lora_scale": {{$json.lora_scale}},
         "megapixels": "{{$json.megapixels}}",
         "num_outputs": {{$json.num_outputs}},
         "aspect_ratio": "{{$json.aspect_ratio}}",
         "output_format": "{{$json.output_format}}",
         "guidance_scale": {{$json.guidance_scale}},
         "output_quality": {{$json.output_quality}},
         "prompt_strength": {{$json.prompt_strength}},
         "extra_lora_scale": {{$json.extra_lora_scale}},
         "replicate_weights": "{{$json.replicate_weights}}",
         "num_inference_steps": {{$json.num_inference_steps}},
         "disable_safety_checker": {{$json.disable_safety_checker}}
       }
     }
     ```
   - Response Format: JSON  
   - Send body and headers enabled.

8. **Connect "Set Other Parameters" output to "Create Other Prediction" input.**

9. **Add a "Code" node named "Log Request":**  
   - JavaScript code:  
     ```javascript
     const data = $input.all()[0].json;
     console.log('izzaanel/betia Request:', {
       timestamp: new Date().toISOString(),
       prediction_id: data.id,
       model_type: 'other'
     });
     return $input.all();
     ```
10. **Connect "Create Other Prediction" output to "Log Request" input.**

11. **Add a "Wait" node named "Wait 5s":**  
    - Unit: seconds  
    - Amount: 5

12. **Connect "Log Request" output to "Wait 5s" input.**

13. **Add an "HTTP Request" node named "Check Status":**  
    - Method: GET  
    - URL: `https://api.replicate.com/v1/predictions/{{$node["Create Other Prediction"].json["id"]}}`  
    - Headers:  
      - Authorization: `Bearer {{$node["Set API Token"].json["api_token"]}}`  
    - Response Format: JSON

14. **Connect "Wait 5s" output to "Check Status" input.**

15. **Add an "IF" node named "Is Complete?":**  
    - Condition:  
      - Type: String  
      - Operation: Equals  
      - Value: "succeeded"  
      - Source: `{{$json.status}}`

16. **Connect "Check Status" output to "Is Complete?" input.**

17. **Add an "IF" node named "Has Failed?":**  
    - Condition:  
      - Type: String  
      - Operation: Equals  
      - Value: "failed"  
      - Source: `{{$json.status}}`

18. **Connect "Is Complete?" false output to "Has Failed?" input.**

19. **Add a "Set" node named "Success Response":**  
    - Assign field `response` with object:  
      ```json
      {
        "success": true,
        "result_url": "{{$json.output}}",
        "prediction_id": "{{$json.id}}",
        "status": "{{$json.status}}",
        "message": "Other generated successfully"
      }
      ```

20. **Connect "Is Complete?" true output to "Success Response" input.**

21. **Add a "Set" node named "Error Response":**  
    - Assign field `response` with object:  
      ```json
      {
        "success": false,
        "error": "{{$json.error || 'Other generation failed'}}",
        "prediction_id": "{{$json.id}}",
        "status": "{{$json.status}}",
        "message": "Failed to generate other"
      }
      ```

22. **Connect "Has Failed?" true output to "Error Response" input.**

23. **Add a "Wait" node named "Wait 10s":**  
    - Unit: seconds  
    - Amount: 10

24. **Connect "Has Failed?" false output to "Wait 10s" input.**

25. **Connect "Wait 10s" output back to "Check Status" input to continue polling loop.**

26. **Add a "Set" node named "Display Result":**  
    - Assign field `final_result` to `{{$json.response}}`

27. **Connect "Success Response" and "Error Response" outputs to "Display Result" input.**

28. **Add two "Sticky Note" nodes:**  
    - One with support contact info: email and social media links.  
    - One with detailed model, parameter, and workflow documentation as per original notes.

---

### 5. General Notes & Resources

| Note Content                                                                                                     | Context or Link                                                                                  |
|-----------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| For questions or support contact: Yaron@nofluff.online                                                         | Support email from Sticky Note9                                                                 |
| Explore tips and tutorials on YouTube: https://www.youtube.com/@YaronBeen/videos                                | Support video channel link                                                                       |
| Connect on LinkedIn: https://www.linkedin.com/in/yaronbeen/                                                    | Professional contact                                                                            |
| Model Documentation: https://replicate.com/izzaanel/betia                                                      | Official model docs                                                                             |
| Replicate API Documentation: https://replicate.com/docs                                                        | API reference                                                                                   |
| n8n Documentation: https://docs.n8n.io                                                                          | Workflow automation platform docs                                                               |

---

**Disclaimer:**  
The text and workflow described result exclusively from an automated n8n workflow using legal, publicly available data and comply with all content policies. No illegal or protected content is involved.