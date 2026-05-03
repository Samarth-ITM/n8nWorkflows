Creating images from text prompts using CyberRealistic Pony v1.25 on Replicate

https://n8nworkflows.xyz/workflows/creating-images-from-text-prompts-using-cyberrealistic-pony-v1-25-on-replicate-6854


# Creating images from text prompts using CyberRealistic Pony v1.25 on Replicate

### 1. Workflow Overview

This n8n workflow automates the process of generating images from text prompts using the "CyberRealistic Pony v1.25" AI model hosted on Replicate. Its main purpose is to allow users to submit detailed descriptive prompts and retrieve high-quality AI-generated images with customizable parameters. The workflow is designed for AI enthusiasts, content creators, and developers who want to integrate AI-based image generation seamlessly into their automation pipelines.

The workflow logic is divided into the following functional blocks:

- **1.1 Input Reception:** Starts execution manually and sets authentication tokens.
- **1.2 Parameter Configuration:** Defines all generation parameters, combining required and optional inputs.
- **1.3 Image Generation Request:** Sends the configured request to the Replicate API and initiates the prediction.
- **1.4 Status Polling Loop:** Periodically checks the status of the prediction until it completes or fails.
- **1.5 Response Handling:** Processes the success or failure responses, formatting output accordingly.
- **1.6 Logging and Monitoring:** Logs each prediction request for debugging and monitoring purposes.
- **1.7 Result Presentation:** Prepares the final structured response for downstream consumption or user display.
- **1.8 Documentation & Notes:** Provides embedded documentation and usage guidance within sticky notes.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
Initial entry point for the workflow triggered manually by the user to start the image generation process.

**Nodes Involved:**  
- Manual Trigger  
- Set API Token

**Node Details:**

- **Manual Trigger**  
  - Type: Trigger node, manual activation  
  - Configuration: Default, no parameters  
  - Inputs: None (starting node)  
  - Outputs: Connects to "Set API Token"  
  - Edge Cases: None; user must trigger manually  
  - Version: 1

- **Set API Token**  
  - Type: Set node, assigns static credentials  
  - Configuration: Assigns a string variable `api_token` with placeholder `"YOUR_REPLICATE_API_TOKEN"`  
  - Inputs: From Manual Trigger  
  - Outputs: Connects to "Set Other Parameters"  
  - Edge Cases: Failure if token is invalid or missing in later API calls  
  - Version: 3.3

---

#### 1.2 Parameter Configuration

**Overview:**  
Sets all generation parameters including both mandatory and optional fields with preset defaults, enabling customization of the generation process.

**Nodes Involved:**  
- Set Other Parameters

**Node Details:**

- **Set Other Parameters**  
  - Type: Set node, assigns multiple variables  
  - Configuration:  
    - Copies `api_token` from previous node  
    - Sets:  
      - `cfg` (CFG scale): 4  
      - `seed`: 0 (random seed)  
      - `steps`: 40 (sampling steps)  
      - `width`: 768 pixels  
      - `height`: 1152 pixels  
      - `prompt`: Detailed positive prompt describing a fashion portrait  
      - `denoise`: 0.98 (denoise strength)  
      - `scheduler`: "karras"  
      - `facerestore`: true (boolean)  
      - `sampler_name`: "dpmpp_3m_sde"  
      - `negative_prompt`: Describes qualities to avoid (low quality, bad anatomy, etc.)  
      - `codeformer_fidelity`: 0.4  
  - Inputs: From "Set API Token"  
  - Outputs: Connects to "Create Other Prediction"  
  - Edge Cases: Misconfigured parameters may cause API rejection or poor output  
  - Version: 3.3

---

#### 1.3 Image Generation Request

**Overview:**  
Sends a POST request to the Replicate API to initiate image generation with the configured parameters. Receives a prediction ID for tracking.

**Nodes Involved:**  
- Create Other Prediction

**Node Details:**

- **Create Other Prediction**  
  - Type: HTTP Request node  
  - Configuration:  
    - URL: `https://api.replicate.com/v1/predictions`  
    - Method: POST  
    - Headers: Authorization bearer token from `api_token`, Prefer: wait (waits for response)  
    - Body: JSON with model version and all input parameters interpolated from previous node's JSON  
    - Response: Parsed JSON enabled, never error (handles API errors gracefully)  
  - Inputs: From "Set Other Parameters"  
  - Outputs: Connects to "Log Request"  
  - Edge Cases: Network issues, invalid token, invalid parameters, API rate limiting  
  - Version: 4.1

---

#### 1.6 Logging and Monitoring

**Overview:**  
Logs critical metadata about the generation request for monitoring and debugging.

**Nodes Involved:**  
- Log Request

**Node Details:**

- **Log Request**  
  - Type: Code node (JavaScript)  
  - Configuration: Logs timestamp, prediction ID, and model type to console  
  - Inputs: From "Create Other Prediction"  
  - Outputs: Connects to "Wait 5s"  
  - Edge Cases: Logging failures do not block workflow  
  - Version: 2

---

#### 1.4 Status Polling Loop

**Overview:**  
Waits and polls the prediction status repeatedly until completion or failure.

**Nodes Involved:**  
- Wait 5s  
- Check Status  
- Is Complete?  
- Has Failed?  
- Wait 10s

**Node Details:**

- **Wait 5s**  
  - Type: Wait node  
  - Configuration: Waits 5 seconds  
  - Inputs: From "Log Request"  
  - Outputs: Connects to "Check Status"  
  - Edge Cases: Timing can be adjusted for faster/slower polling  
  - Version: 1

- **Check Status**  
  - Type: HTTP Request node  
  - Configuration:  
    - URL dynamically constructed with prediction ID from "Create Other Prediction"  
    - Method: GET  
    - Headers: Authorization bearer token from "Set API Token"  
    - Response JSON parsed, never error  
  - Inputs: From "Wait 5s" and "Wait 10s"  
  - Outputs: Connects to "Is Complete?"  
  - Edge Cases: Network errors, API errors, invalid prediction ID  
  - Version: 4.1

- **Is Complete?**  
  - Type: If node  
  - Configuration: Checks if `status` field equals `"succeeded"`  
  - Inputs: From "Check Status"  
  - Outputs:  
    - True: To "Success Response"  
    - False: To "Has Failed?"  
  - Edge Cases: Unexpected status values  
  - Version: 2

- **Has Failed?**  
  - Type: If node  
  - Configuration: Checks if `status` field equals `"failed"`  
  - Inputs: From "Is Complete?"  
  - Outputs:  
    - True: To "Error Response"  
    - False: To "Wait 10s" (retry loop)  
  - Edge Cases: Infinite loops if status stuck in unknown state  
  - Version: 2

- **Wait 10s**  
  - Type: Wait node  
  - Configuration: Waits 10 seconds before retrying status check  
  - Inputs: From "Has Failed?" (false branch)  
  - Outputs: Connects back to "Check Status"  
  - Edge Cases: May delay workflow significantly if prediction takes long  
  - Version: 1

---

#### 1.5 Response Handling

**Overview:**  
Formats structured JSON responses for success or failure cases.

**Nodes Involved:**  
- Success Response  
- Error Response  
- Display Result

**Node Details:**

- **Success Response**  
  - Type: Set node  
  - Configuration: Sets `response` object with fields:  
    - `success`: true  
    - `result_url`: prediction output (URL to generated image)  
    - `prediction_id`  
    - `status`  
    - `message`: "Other generated successfully"  
  - Inputs: From "Is Complete?" (true branch)  
  - Outputs: Connects to "Display Result"  
  - Edge Cases: Missing output URL in JSON  
  - Version: 3.3

- **Error Response**  
  - Type: Set node  
  - Configuration: Sets `response` object with fields:  
    - `success`: false  
    - `error`: error message or generic "Other generation failed"  
    - `prediction_id`  
    - `status`  
    - `message`: "Failed to generate other"  
  - Inputs: From "Has Failed?" (true branch)  
  - Outputs: Connects to "Display Result"  
  - Edge Cases: Missing error details in JSON  
  - Version: 3.3

- **Display Result**  
  - Type: Set node  
  - Configuration: Assigns final `final_result` object equal to the previous node's `response`  
  - Inputs: From both "Success Response" and "Error Response"  
  - Outputs: None (end of workflow)  
  - Edge Cases: None  
  - Version: 3.3

---

#### 1.8 Documentation & Notes

**Overview:**  
Two sticky note nodes embed detailed documentation, contact info, parameter explanations, and usage instructions directly within the workflow for easy reference.

**Nodes Involved:**  
- Sticky Note9  
- Sticky Note4

**Node Details:**

- **Sticky Note9**  
  - Type: Sticky Note node  
  - Content: Contact info for support (Yaron@nofluff.online), and links to YouTube and LinkedIn for tutorials and tips  
  - Position: Top-left corner  
  - Edge Cases: None

- **Sticky Note4**  
  - Type: Sticky Note node  
  - Content: Extensive model overview, parameter reference, workflow component explanations, benefits, quick start instructions, troubleshooting guide, and resource links  
  - Position: Large block on left side  
  - Edge Cases: None

---

### 3. Summary Table

| Node Name               | Node Type            | Functional Role                 | Input Node(s)          | Output Node(s)            | Sticky Note                                                                                  |
|-------------------------|----------------------|--------------------------------|-----------------------|---------------------------|----------------------------------------------------------------------------------------------|
| Manual Trigger          | Manual Trigger       | Starts workflow manually       | None                  | Set API Token             |                                                                                              |
| Set API Token           | Set                  | Sets Replicate API token       | Manual Trigger        | Set Other Parameters      |                                                                                              |
| Set Other Parameters    | Set                  | Defines generation parameters  | Set API Token         | Create Other Prediction   |                                                                                              |
| Create Other Prediction | HTTP Request         | Sends generation request       | Set Other Parameters  | Log Request               |                                                                                              |
| Log Request             | Code                 | Logs request info              | Create Other Prediction | Wait 5s                  |                                                                                              |
| Wait 5s                 | Wait                 | Delays for 5 seconds           | Log Request           | Check Status              |                                                                                              |
| Check Status            | HTTP Request         | Polls prediction status        | Wait 5s, Wait 10s     | Is Complete?              |                                                                                              |
| Is Complete?            | If                   | Checks if prediction succeeded | Check Status          | Success Response, Has Failed? |                                                                                              |
| Has Failed?             | If                   | Checks if prediction failed    | Is Complete?          | Error Response, Wait 10s  |                                                                                              |
| Wait 10s                | Wait                 | Delays for 10 seconds          | Has Failed?           | Check Status              |                                                                                              |
| Success Response        | Set                  | Formats success response       | Is Complete?          | Display Result            |                                                                                              |
| Error Response          | Set                  | Formats error response         | Has Failed?            | Display Result            |                                                                                              |
| Display Result          | Set                  | Finalizes output object        | Success Response, Error Response | None                 |                                                                                              |
| Sticky Note9            | Sticky Note          | Support contact and links      | None                  | None                      | Contact: Yaron@nofluff.online; YouTube: https://www.youtube.com/@YaronBeen/videos; LinkedIn: https://www.linkedin.com/in/yaronbeen/ |
| Sticky Note4            | Sticky Note          | Full documentation and guide   | None                  | None                      | Detailed model overview, parameter guide, instructions, troubleshooting, resource links       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger node**  
   - Name: "Manual Trigger"  
   - Purpose: Manual start of the workflow  
   - No special configuration

2. **Create Set node for API Token**  
   - Name: "Set API Token"  
   - Connect input from "Manual Trigger"  
   - Add assignment:  
     - Key: `api_token`  
     - Type: String  
     - Value: `"YOUR_REPLICATE_API_TOKEN"` (replace with your actual token)

3. **Create Set node for Other Parameters**  
   - Name: "Set Other Parameters"  
   - Connect input from "Set API Token"  
   - Assign multiple parameters:  
     - `api_token`: Copy from previous node `$('Set API Token').item.json.api_token`  
     - `cfg`: Number 4  
     - `seed`: Number 0  
     - `steps`: Number 40  
     - `width`: Number 768  
     - `height`: Number 1152  
     - `prompt`: String with detailed fashion portrait prompt  
     - `denoise`: Number 0.98  
     - `scheduler`: String "karras"  
     - `facerestore`: Boolean true  
     - `sampler_name`: String "dpmpp_3m_sde"  
     - `negative_prompt`: String describing unwanted image features  
     - `codeformer_fidelity`: Number 0.4

4. **Create HTTP Request node to send prediction request**  
   - Name: "Create Other Prediction"  
   - Connect input from "Set Other Parameters"  
   - Method: POST  
   - URL: `https://api.replicate.com/v1/predictions`  
   - Headers:  
     - Authorization: `Bearer {{$json.api_token}}`  
     - Prefer: `wait`  
   - Body Type: JSON  
   - Body content:  
     ```json
     {
       "version": "0xdino/cyberrealistic-pony-v125:bf02541d927899ad63dae3abbfbd5185f5a00f09401ccbe5bfd0f66d6283ea65",
       "input": {
         "cfg": {{$json.cfg}},
         "seed": {{$json.seed}},
         "steps": {{$json.steps}},
         "width": {{$json.width}},
         "height": {{$json.height}},
         "prompt": "{{$json.prompt}}",
         "denoise": {{$json.denoise}},
         "scheduler": "{{$json.scheduler}}",
         "facerestore": {{$json.facerestore}},
         "sampler_name": "{{$json.sampler_name}}",
         "negative_prompt": "{{$json.negative_prompt}}",
         "codeformer_fidelity": {{$json.codeformer_fidelity}}
       }
     }
     ```
   - Enable JSON response parsing and never error option

5. **Create Code node to log request**  
   - Name: "Log Request"  
   - Connect input from "Create Other Prediction"  
   - JS Code:  
     ```js
     const data = $input.all()[0].json;
     console.log('0xdino/cyberrealistic-pony-v125 Request:', {
       timestamp: new Date().toISOString(),
       prediction_id: data.id,
       model_type: 'other'
     });
     return $input.all();
     ```

6. **Create Wait node to pause 5 seconds**  
   - Name: "Wait 5s"  
   - Connect input from "Log Request"  
   - Parameters: 5 seconds

7. **Create HTTP Request node to check prediction status**  
   - Name: "Check Status"  
   - Connect input from "Wait 5s" and "Wait 10s" (later)  
   - Method: GET  
   - URL: `https://api.replicate.com/v1/predictions/{{$('Create Other Prediction').item.json.id}}`  
   - Headers:  
     - Authorization: `Bearer {{$('Set API Token').item.json.api_token}}`  
   - Enable JSON response parsing and never error option

8. **Create If node to check if prediction succeeded**  
   - Name: "Is Complete?"  
   - Connect input from "Check Status"  
   - Condition: `$json.status === "succeeded"`  
   - True branch: to "Success Response"  
   - False branch: to "Has Failed?"

9. **Create If node to check if prediction failed**  
   - Name: "Has Failed?"  
   - Connect input from "Is Complete?" false branch  
   - Condition: `$json.status === "failed"`  
   - True branch: to "Error Response"  
   - False branch: to "Wait 10s"

10. **Create Wait node to pause 10 seconds**  
    - Name: "Wait 10s"  
    - Connect input from "Has Failed?" false branch  
    - Parameters: 10 seconds  
    - Output loops back to "Check Status"

11. **Create Set node for success response**  
    - Name: "Success Response"  
    - Connect input from "Is Complete?" true branch  
    - Assign `response` object with:  
      - success: true  
      - result_url: `$json.output` (image URL)  
      - prediction_id: `$json.id`  
      - status: `$json.status`  
      - message: "Other generated successfully"

12. **Create Set node for error response**  
    - Name: "Error Response"  
    - Connect input from "Has Failed?" true branch  
    - Assign `response` object with:  
      - success: false  
      - error: `$json.error` or "Other generation failed"  
      - prediction_id: `$json.id`  
      - status: `$json.status`  
      - message: "Failed to generate other"

13. **Create Set node to display final result**  
    - Name: "Display Result"  
    - Connect input from both "Success Response" and "Error Response"  
    - Assign `final_result` = `$json.response`

14. **Optionally, add Sticky Note nodes**  
    - Add one with support contact and links  
    - Add one with full documentation, parameter descriptions, troubleshooting, and resource links

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                      | Context or Link                                               |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------|
| CYBERREALISTIC-PONY-V125 GENERATOR: For questions or support contact Yaron@nofluff.online; YouTube channel with tutorials: https://www.youtube.com/@YaronBeen/videos; LinkedIn profile: https://www.linkedin.com/in/yaronbeen/                                                                                                                                                                                                                                   | Support contacts and tutorial links embedded in workflow     |
| Powered by Replicate API and n8n automation; includes detailed parameter reference, workflow explanation, quick start instructions, troubleshooting guide, and external resources such as model docs (https://replicate.com/0xdino/cyberrealistic-pony-v125), Replicate API docs (https://replicate.com/docs), and n8n documentation (https://docs.n8n.io)                                                                                                          | Embedded workflow documentation and resource links            |
| Best practices: Replace API token with your own, monitor usage and billing, use valid parameter values, avoid committing API tokens to code, test with defaults first, and monitor logs for issues.                                                                                                                                                                                                                                                                | Usage recommendations                                        |
| The workflow uses the latest n8n node versions (HTTP Request v4.1, Set v3.3, If v2), ensure your n8n instance supports these for compatibility.                                                                                                                                                                                                                                                                                                                  | Version requirements                                         |

---

**Disclaimer:**  
The content above is generated from an automated n8n workflow using publicly available AI models via the Replicate API. It complies with all applicable content policies and contains no illegal or protected content. All data processed is legal and publicly accessible.