Transform Text to UGC Videos with OpenAI SORA-2 on Replicate

https://n8nworkflows.xyz/workflows/transform-text-to-ugc-videos-with-openai-sora-2-on-replicate-9362


# Transform Text to UGC Videos with OpenAI SORA-2 on Replicate

### 1. Workflow Overview

This n8n workflow automates the creation of short User-Generated Content (UGC) videos from text prompts using the OpenAI SORA-2 model hosted on Replicate. It enables users to input a descriptive text prompt and video length, then generates a video via the SORA-2 model API. The workflow includes handling of asynchronous video generation, status polling, and success/error response formatting.

The workflow is logically divided into the following blocks:

- **1.1 Input Initialization**  
  Receives manual trigger and sets up required API tokens and input parameters (prompt, seed image URL, video length).

- **1.2 Video Creation Request**  
  Sends a POST request to the Replicate API to start video generation with the given inputs.

- **1.3 Request Logging**  
  Logs the prediction request details for monitoring and debugging.

- **1.4 Status Polling Loop**  
  Periodically checks the status of the video generation until it either succeeds or fails.

- **1.5 Success and Error Handling**  
  Processes the final API response, formats success or error outputs, and presents the results.

- **1.6 Utility and Documentation Nodes**  
  Includes sticky notes for guidance, best practices, and optional batch processing tips.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Initialization

**Overview:**  
Initializes the workflow by manually triggering the process, setting API tokens for Replicate and OpenAI, and defining the video generation inputs such as prompt, seed image, and video length.

**Nodes Involved:**  
- Manual Trigger  
- Set API Token  
- Add Seed Image, Prompt and amount of seconds  
- Sticky Notes: Sticky Note, Sticky Note1, Sticky Note2

**Node Details:**

- **Manual Trigger**  
  - Type: Trigger node that manually starts the workflow.  
  - Config: No parameters; starts workflow on demand.  
  - Inputs: None  
  - Outputs: Set API Token  
  - Edge cases: None specific.

- **Set API Token**  
  - Type: Set node to assign API keys as workflow variables.  
  - Config: Stores `replicate_api_token` and `OPENAI_API_TOKE` as strings, placeholders for user to replace with actual keys.  
  - Expressions: Direct string assignment (e.g. `"=YOUR_REPLICATE_API_KEY"`).  
  - Inputs: Manual Trigger  
  - Outputs: Add Seed Image, Prompt and amount of seconds  
  - Edge cases: Missing or invalid API keys will cause authentication failures downstream.

- **Add Seed Image, Prompt and amount of seconds**  
  - Type: Set node to provide required inputs for video generation.  
  - Config:  
    - `Seed_image`: URL string to a seed image (1280x720 recommended).  
    - `Prompt`: Detailed text describing the video scene and action (JSON-safe, single line).  
    - `seconds (4,8 or 12)`: Number specifying video length.  
  - Inputs: Set API Token  
  - Outputs: Create Video  
  - Edge cases: Invalid URLs, prompts containing disallowed characters (quotes, backslashes, line breaks) can break JSON parsing.

- **Sticky Notes**  
  - Provide instructions for API key insertion, prompt writing guidelines, and seed image requirements.  
  - Contextual aid, no direct workflow logic.  
  - Important notes include avoiding double quotes, backslashes, and line breaks in prompts.

---

#### 1.2 Video Creation Request

**Overview:**  
Submits the video generation request to Replicate’s SORA-2 model API using the inputs and API tokens.

**Nodes Involved:**  
- Create Video

**Node Details:**

- **Create Video**  
  - Type: HTTP Request node (POST).  
  - Config:  
    - URL: `https://api.replicate.com/v1/models/openai/sora-2/predictions`  
    - Headers: Authorization with Bearer token from `replicate_api_token`, `Prefer: wait`.  
    - Body: JSON with keys: `prompt`, `seconds`, `aspect_ratio` (fixed `"landscape"`), and `openai_api_key` from token.  
  - Expressions reference previous node data for prompt, seconds, and API keys.  
  - Inputs: Add Seed Image, Prompt and amount of seconds  
  - Outputs: Log Request  
  - Edge cases: HTTP errors, authentication failures, invalid request body, API rate limits.

---

#### 1.3 Request Logging

**Overview:**  
Logs the prediction request details to console for monitoring and debugging purposes.

**Nodes Involved:**  
- Log Request

**Node Details:**

- **Log Request**  
  - Type: Code node (JavaScript).  
  - Config: Logs timestamp, prediction ID, and model type to console.  
  - Input: Response from Create Video containing prediction metadata.  
  - Output: Wait 5s  
  - Edge cases: None critical; logging failure does not halt workflow.

---

#### 1.4 Status Polling Loop

**Overview:**  
Periodically queries the prediction status from Replicate API until completion is confirmed or failure detected.

**Nodes Involved:**  
- Wait 5s  
- Check Status  
- Is Complete?  
- Has Failed?  
- Wait 90s

**Node Details:**

- **Wait 5s**  
  - Type: Wait node.  
  - Config: Pauses workflow for 5 seconds before status check.  
  - Input: Log Request  
  - Output: Check Status  
  - Edge cases: Workflow pause can be too short if API response delay is longer.

- **Check Status**  
  - Type: HTTP Request node (GET).  
  - Config:  
    - URL dynamically built using prediction ID from `Create Video` response.  
    - Authorization header with `replicate_api_token`.  
    - Response JSON parsed, error ignored to avoid workflow stop on HTTP errors.  
  - Input: Wait 5s and Wait 90s (loop back)  
  - Output: Is Complete?  
  - Edge cases: Network errors, API downtime, invalid prediction ID.

- **Is Complete?**  
  - Type: If node.  
  - Config: Checks if `status` field equals `"succeeded"`.  
  - Input: Check Status  
  - Output:  
    - True: Success Response  
    - False: Has Failed?  
  - Edge cases: Status field missing or unexpected value.

- **Has Failed?**  
  - Type: If node.  
  - Config: Checks if `status` equals `"failed"`.  
  - Input: Is Complete? (False branch)  
  - Output:  
    - True: Error Response  
    - False: Wait 90s  
  - Edge cases: Other statuses (e.g., `"processing"`) cause loop to continue.

- **Wait 90s**  
  - Type: Wait node.  
  - Config: Pauses 90 seconds before looping back to Check Status.  
  - Input: Has Failed? (False branch)  
  - Output: Check Status  
  - Edge cases: Long wait time may delay feedback; adjust as needed.

---

#### 1.5 Success and Error Handling

**Overview:**  
Formats final output object on success or failure for downstream use or presentation.

**Nodes Involved:**  
- Success Response  
- Error Response  
- Display Result

**Node Details:**

- **Success Response**  
  - Type: Set node.  
  - Config: Creates an object `response` with keys: `success: true`, `result_url` (video URL), `prediction_id`, `status`, and a success message.  
  - Input: Is Complete? (True branch)  
  - Output: Display Result  
  - Edge cases: Missing output URL or unexpected response structure.

- **Error Response**  
  - Type: Set node.  
  - Config: Creates an object `response` with keys: `success: false`, `error` (from API or generic message), `prediction_id`, `status`, and error message.  
  - Input: Has Failed? (True branch)  
  - Output: Display Result  
  - Edge cases: Missing error details in response.

- **Display Result**  
  - Type: Set node.  
  - Config: Assigns `final_result` to the `response` object created by Success or Error Response nodes.  
  - Input: Success Response and Error Response  
  - Output: None (terminal node)  
  - Edge cases: None.

---

#### 1.6 Utility and Documentation Nodes

**Overview:**  
Provides user guidance, batch processing tips, and optional nodes for scaling.

**Nodes Involved:**  
- Sticky Note9  
- Sticky Note4  
- Sticky Note3  
- Sticky Note5  
- Loop Over Items (optional batch processing node)

**Node Details:**

- **Sticky Note9, Sticky Note4, Sticky Note3, Sticky Note5**  
  - Type: Sticky Note nodes.  
  - Content: Detailed instructions about the workflow, model information, input requirements, prompt guidelines, batch processing options, and contact/support info.  
  - Position: Spread throughout the canvas for user reference.

- **Loop Over Items**  
  - Type: SplitInBatches node (not connected in main flow).  
  - Usage: Optionally processes multiple video generation requests in batches for bulk operations.  
  - Edge cases: Requires additional input structure for batch processing.

---

### 3. Summary Table

| Node Name                                    | Node Type           | Functional Role                          | Input Node(s)                        | Output Node(s)                  | Sticky Note                                                                                                                  |
|----------------------------------------------|---------------------|----------------------------------------|------------------------------------|--------------------------------|------------------------------------------------------------------------------------------------------------------------------|
| Manual Trigger                               | Manual Trigger      | Starts the workflow manually           | None                               | Set API Token                  |                                                                                                                              |
| Set API Token                                | Set                 | Stores API tokens                      | Manual Trigger                    | Add Seed Image, Prompt and amount of seconds | Sticky Note: "Add Your Replicate API and OpenAI api key"                                                                      |
| Add Seed Image, Prompt and amount of seconds | Set                 | Adds video inputs: seed image, prompt, seconds | Set API Token                    | Create Video                  | Sticky Note1: "Add a link to your Seed image. If you dont have one, you can generate it using OpenAI or Nano Banana..."        |
| Create Video                                 | HTTP Request        | Sends video generation request         | Add Seed Image, Prompt and amount of seconds | Log Request                  | Sticky Note4: "SORA-2 TEXT-TO-VIDEO WORKFLOW, Replicate API + n8n automation..."                                              |
| Log Request                                  | Code                | Logs prediction request info           | Create Video                     | Wait 5s                       |                                                                                                                              |
| Wait 5s                                      | Wait                | Pause 5 seconds before status check    | Log Request                     | Check Status                  |                                                                                                                              |
| Check Status                                 | HTTP Request        | Queries prediction status               | Wait 5s, Wait 90s                | Is Complete?                  |                                                                                                                              |
| Is Complete?                                 | If                  | Checks if prediction succeeded         | Check Status                    | Success Response, Has Failed? |                                                                                                                              |
| Has Failed?                                  | If                  | Checks if prediction failed            | Is Complete?                   | Error Response, Wait 90s      |                                                                                                                              |
| Wait 90s                                     | Wait                | Pause 90 seconds before next status check | Has Failed?                   | Check Status                 |                                                                                                                              |
| Success Response                             | Set                 | Formats success output object           | Is Complete? (true branch)       | Display Result               |                                                                                                                              |
| Error Response                               | Set                 | Formats error output object             | Has Failed? (true branch)         | Display Result               |                                                                                                                              |
| Display Result                               | Set                 | Final output assignment                  | Success Response, Error Response | None                        | Sticky Note3: "Connect this node to your storage in order to save the predictions"                                            |
| Loop Over Items                              | SplitInBatches      | Optional batch processing splitter      | None                           | Loop Over Items (loop)        | Sticky Note5: "Batch Processing Guide, Optional: Generate Multiple Videos..."                                                 |
| Sticky Note9                                 | Sticky Note         | Workflow branding and support info      | None                           | None                        | See content with YouTube and LinkedIn links                                                                                   |
| Sticky Note4                                 | Sticky Note         | Workflow overview and input/output details | None                           | None                        | Detailed workflow and model info                                                                                              |
| Sticky Note1                                 | Sticky Note         | Seed image instructions                 | None                           | None                        | Advises on seed image setup                                                                                                   |
| Sticky Note2                                 | Sticky Note         | Prompt writing guidelines               | None                           | None                        | Avoid double quotes, backslashes, line breaks                                                                                 |
| Sticky Note3                                 | Sticky Note         | Output saving instruction               | None                           | None                        | Connect Display Result node to storage                                                                                       |
| Sticky Note5                                 | Sticky Note         | Batch processing guidance               | None                           | None                        | Explains bulk video generation options                                                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Manual Trigger node**  
   - Name: `Manual Trigger`  
   - No configuration needed.

3. **Add a Set node for API tokens**  
   - Name: `Set API Token`  
   - Add two string fields:  
     - `replicate_api_token`: Set to your Replicate API key (e.g., `"YOUR_REPLICATE_API_KEY"`).  
     - `OPENAI_API_TOKE`: Set to your OpenAI API key (e.g., `"YOUR_OPENAI_API_KEY"`).  
   - Connect `Manual Trigger` output to this node.

4. **Add a Set node for inputs**  
   - Name: `Add Seed Image, Prompt and amount of seconds`  
   - Add fields:  
     - `Seed_image` (string): Provide a URL of seed image (recommend 1280x720).  
     - `Prompt` (string): Input your prompt describing the video scene (ensure JSON-safe, no quotes/backslashes/newlines).  
     - `seconds (4,8 or 12)` (number): Set video length (4, 8, or 12 seconds).  
   - Connect `Set API Token` output to this node.

5. **Add an HTTP Request node to create video**  
   - Name: `Create Video`  
   - Method: POST  
   - URL: `https://api.replicate.com/v1/models/openai/sora-2/predictions`  
   - Headers:  
     - `Authorization`: `Bearer {{ $json.replicate_api_token }}` (use expression to pull from `Set API Token`)  
     - `Prefer`: `wait`  
   - Body (JSON):  
     ```json
     {
       "input": {
         "prompt": "{{ $json.Prompt }}",
         "seconds": {{ $json["seconds (4,8 or 12)"] }},
         "aspect_ratio": "landscape",
         "openai_api_key": "{{ $json.OPENAI_API_TOKE }}"
       }
     }
     ```  
   - Connect `Add Seed Image, Prompt and amount of seconds` output to this node.

6. **Add a Code node to log the request**  
   - Name: `Log Request`  
   - JavaScript code:  
     ```javascript
     const data = $input.all()[0].json;
     console.log('digitalhera/heranathalie Request:', {
       timestamp: new Date().toISOString(),
       prediction_id: data.id,
       model_type: 'other'
     });
     return $input.all();
     ```  
   - Connect `Create Video` output to this node.

7. **Add a Wait node (5 seconds)**  
   - Name: `Wait 5s`  
   - Set to wait for 5 seconds.  
   - Connect `Log Request` output to this node.

8. **Add an HTTP Request node to check status**  
   - Name: `Check Status`  
   - Method: GET  
   - URL: `https://api.replicate.com/v1/predictions/{{ $json.id }}` (expression referencing the prediction ID from `Create Video`)  
   - Headers:  
     - `Authorization`: `Bearer {{ $json.replicate_api_token }}` (from `Set API Token`)  
   - Set to never error on response.  
   - Connect `Wait 5s` output to this node.

9. **Add an If node to check completion**  
   - Name: `Is Complete?`  
   - Condition: Check if `status` equals `"succeeded"`.  
   - Connect `Check Status` output to this node.

10. **Add an If node to check failure**  
    - Name: `Has Failed?`  
    - Condition: Check if `status` equals `"failed"`.  
    - Connect `Is Complete?` false output to this node.

11. **Add a Set node for success response**  
    - Name: `Success Response`  
    - Configure to set `response` object with:  
      - `success: true`  
      - `result_url`: `$json.output` (video URL)  
      - `prediction_id`: `$json.id`  
      - `status`: `$json.status`  
      - `message`: `"Video generated successfully"`  
    - Connect `Is Complete?` true output to this node.

12. **Add a Set node for error response**  
    - Name: `Error Response`  
    - Configure to set `response` object with:  
      - `success: false`  
      - `error`: `$json.error` or fallback string `"Video generation failed"`  
      - `prediction_id`: `$json.id`  
      - `status`: `$json.status`  
      - `message`: `"Failed to generate video"`  
    - Connect `Has Failed?` true output to this node.

13. **Add a Wait node (90 seconds)**  
    - Name: `Wait 90s`  
    - Set to wait for 90 seconds.  
    - Connect `Has Failed?` false output to this node.

14. **Connect Wait 90s output back to Check Status** to form a polling loop.

15. **Add a Set node to display final result**  
    - Name: `Display Result`  
    - Set `final_result` to the `response` object from either success or error.  
    - Connect `Success Response` and `Error Response` outputs to this node.

16. **Optional:** Add Sticky Note nodes with the content from corresponding sticky notes to provide instructions and guidance for users.

17. **Optional:** For batch processing, add a SplitInBatches node (`Loop Over Items`) after `Set API Token` and configure accordingly.

18. **Save and activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                             | Context or Link                                                                                                         |
|------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------|
| IMAGE TO VIDEO GENERATOR by Yaron Been. Support contact: Yaron@nofluff.online. Explore tips on YouTube and LinkedIn.                    | Sticky Note9 content with branding and support info.                                                                    |
| SORA-2 TEXT-TO-VIDEO MODEL details: OpenAI owner, endpoint, input requirements, and usage tips.                                         | Sticky Note4 content provides model info and workflow description.                                                     |
| Prompt writing best practices: Avoid double quotes, backslashes, and line breaks to ensure JSON compatibility.                         | Sticky Note2 content.                                                                                                   |
| Batch processing guide for generating multiple videos with same/different prompts and images.                                            | Sticky Note5 content explaining batch usage scenarios.                                                                 |
| Connect final Display Result node to storage or other systems to save generated videos or trigger further automation.                   | Sticky Note3 content advising on output usage.                                                                           |
| Monitor API usage and billing on Replicate platform to avoid unexpected costs.                                                          | General best practice implied from usage of Replicate API.                                                              |
| Use n8n credentials for storing API keys securely instead of plain text in production workflows.                                         | Security best practice to avoid exposing secrets.                                                                        |

---

This documentation provides a complete and detailed reference for understanding, reproducing, and extending the "Transform Text to UGC Videos with OpenAI SORA-2 on Replicate" n8n workflow. It ensures clarity for both human operators and automated systems.