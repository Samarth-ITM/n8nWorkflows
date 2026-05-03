Generate AI Images & Videos with KIE.AI Midjourney API

https://n8nworkflows.xyz/workflows/generate-ai-images---videos-with-kie-ai-midjourney-api-7677


# Generate AI Images & Videos with KIE.AI Midjourney API

### 1. Workflow Overview

This n8n workflow enables users to generate AI images and videos by interfacing with the KIE.AI Midjourney API. It provides an intuitive form-based interface for submitting generation requests in three modes: text-to-image, image-to-image, and image-to-video. Upon submission, the workflow sends the request to the API, waits and polls periodically for generation completion, then retrieves and displays the final results.

The workflow logic is organized into the following blocks:

- **1.1 User Input and Form Submission:** Captures user inputs through a form, including generation mode, prompt text, optional image URLs, and API key.
- **1.2 API Request Submission:** Sends the generation request to the KIE.AI Midjourney API with user-provided parameters.
- **1.3 Processing Wait and Polling:** Waits for a fixed interval, then polls the API to check the generation status repeatedly until completion.
- **1.4 Result Retrieval and Presentation:** Once generation is complete, formats and outputs the generated image/video URLs for user consumption.
- **1.5 Informational Guidance:** Sticky notes provide important instructions, setup steps, and best practices for using the workflow and API effectively.

---

### 2. Block-by-Block Analysis

#### 2.1 User Input and Form Submission

- **Overview:**  
  This block collects necessary user input via a form trigger node, providing fields for prompt, image URL(s), task type, and API key. It initiates the workflow execution upon form submission.

- **Nodes Involved:**  
  - `Submit Text Prompt for Video Generation`

- **Node Details:**

  - **Submit Text Prompt for Video Generation**  
    - Type: Form Trigger  
    - Role: Entry point to receive user inputs through a web form.  
    - Configuration:  
      - Form title: "AI video generator"  
      - Fields:  
        - `prompt` (text): User’s description of desired content.  
        - `img_url` (text): Optional image URL(s) for image-to-image or video generation modes.  
        - `task_type` (text): Generation mode selector (`mj_txt2img`, `mj_img2img`, `mj_video`).  
        - `api_key` (text): User’s API key for authentication.  
      - Form description instructs users how to fill the form.  
    - Inputs: None (trigger node)  
    - Outputs: JSON containing submitted form data.  
    - Edge Cases/Failures:  
      - Missing required fields (prompt, task_type, api_key) may cause downstream API errors.  
      - Malformed or inaccessible image URLs may result in generation failures.  
    - Version-specific: Uses formTrigger node version 2.2.

#### 2.2 API Request Submission

- **Overview:**  
  This block constructs and sends an HTTP POST request to the KIE.AI Midjourney API to start the AI content generation process based on user inputs.

- **Nodes Involved:**  
  - `Send Video Generation Request to KIE.AI API`

- **Node Details:**

  - **Send Video Generation Request to KIE.AI API**  
    - Type: HTTP Request  
    - Role: Sends generation request with parameters to KIE.AI API endpoint.  
    - Configuration:  
      - URL: `https://api.kie.ai/api/v1/mj/generate`  
      - Method: POST  
      - Headers:  
        - `Content-Type`: application/json  
        - `Authorization`: Bearer token from `api_key` form input  
      - Body (JSON): Includes taskType, prompt, fileUrls (array of image URLs or empty), aspectRatio (16:9), version (7), and other stylization parameters.  
      - Uses expressions to dynamically insert form data:  
        - `taskType` from `task_type`  
        - `prompt` text  
        - `fileUrls` array constructed conditionally from `img_url` (empty array if none)  
    - Inputs: JSON from form trigger node  
    - Outputs: JSON containing API response, including taskId used for polling status.  
    - Edge Cases/Failures:  
      - Invalid API key authorization errors.  
      - Improperly formatted or missing parameters causing API rejection.  
      - Network timeouts or connectivity issues.  
    - Version-specific: HTTP request node version 4.2.

#### 2.3 Processing Wait and Polling

- **Overview:**  
  After submitting the generation request, this block waits for 10 seconds, then polls the KIE.AI API repeatedly to check if the generation task is complete.

- **Nodes Involved:**  
  - `Wait for Video Processing Completion`  
  - `Obtain the generated status`  
  - `Check if Video Generation is Complete`

- **Node Details:**

  - **Wait for Video Processing Completion**  
    - Type: Wait  
    - Role: Delays execution for 10 seconds between polling attempts.  
    - Configuration: Wait time set to 10 seconds.  
    - Inputs: Output from API request or conditional node.  
    - Outputs: Triggers the next polling status request.  
    - Edge Cases: Excessive waiting may cause workflow delays; insufficient wait may cause too frequent polling.

  - **Obtain the generated status**  
    - Type: HTTP Request  
    - Role: Sends GET request to API to retrieve current status of generation task.  
    - Configuration:  
      - URL: `https://api.kie.ai/api/v1/mj/record-info`  
      - Method: GET  
      - Query Parameter: `taskId` extracted from previous API response (`data.taskId`)  
      - Headers: Authorization with Bearer token from original API key input  
      - Content-Type: application/json  
    - Inputs: From wait node, uses taskId from initial API response JSON.  
    - Outputs: JSON with status flags and result URLs if generation is complete.  
    - Edge Cases:  
      - API rate limits or failures could interrupt status checks.  
      - Missing or invalid taskId leading to errors.

  - **Check if Video Generation is Complete**  
    - Type: If Node  
    - Role: Evaluates whether generation succeeded by checking `successFlag == 1` in status JSON.  
    - Configuration:  
      - Condition: `data.successFlag == 1` (true)  
      - True branch: proceeds to format results.  
      - False branch: loops back to wait node to retry polling.  
    - Inputs: JSON from status request node.  
    - Outputs: Branches workflow based on completion status.  
    - Edge Cases:  
      - `successFlag` may be absent or different if generation failed; no explicit failure handling in workflow.

#### 2.4 Result Retrieval and Presentation

- **Overview:**  
  Upon successful completion, this block extracts and formats the generated media URLs for user display or further use.

- **Nodes Involved:**  
  - `Format and Display Video Results`

- **Node Details:**

  - **Format and Display Video Results**  
    - Type: Set  
    - Role: Assigns result URLs from API response to defined output fields.  
    - Configuration:  
      - Extracts up to four result URLs from `data.resultInfoJson.resultUrls` array.  
      - Assigns each URL to variables `img1` through `img4`.  
    - Inputs: JSON from the "Check if Video Generation is Complete" node’s true branch.  
    - Outputs: JSON with formatted result URLs.  
    - Edge Cases:  
      - If fewer than four URLs exist, some fields may be empty or undefined.  
      - Does not handle error cases or incomplete data gracefully.

#### 2.5 Informational Guidance

- **Overview:**  
  Sticky notes provide key instructions and context for users to properly configure and use the workflow and API.

- **Nodes Involved:**  
  - `Sticky Note6`  
  - `Sticky Note`  
  - `Sticky Note3`  
  - `Sticky Note1`

- **Node Details:**

  - **Sticky Note6**  
    - Content: Instructions to create a KIE.AI account and obtain the API key, emphasizing key safety and confidentiality.  
    - Helps users understand prerequisite setup.

  - **Sticky Note**  
    - Content: Step-by-step process usage instructions, including starting the workflow, filling and submitting the form, and waiting for processing.  
    - Guides users through workflow operation.

  - **Sticky Note3**  
    - Content: Detailed overview describing workflow purpose, supported generation modes, user prerequisites, customization tips, and troubleshooting advice.  
    - Provides comprehensive context and usage recommendations.

  - **Sticky Note1**  
    - Content: Detailed explanation of form parameters, their purpose, examples, and tips for better results.  
    - Clarifies form input expectations and best practices.

---

### 3. Summary Table

| Node Name                             | Node Type       | Functional Role                          | Input Node(s)                         | Output Node(s)                        | Sticky Note                                                                                                               |
|-------------------------------------|-----------------|----------------------------------------|-------------------------------------|-------------------------------------|---------------------------------------------------------------------------------------------------------------------------|
| Submit Text Prompt for Video Generation | Form Trigger    | Captures user input via form           | None                                | Send Video Generation Request to KIE.AI API |                                                                                                                           |
| Send Video Generation Request to KIE.AI API | HTTP Request   | Sends generation request to KIE.AI API | Submit Text Prompt for Video Generation | Wait for Video Processing Completion |                                                                                                                           |
| Wait for Video Processing Completion | Wait            | Delays workflow to await generation    | Send Video Generation Request to KIE.AI API / Check if Video Generation is Complete (false branch) | Obtain the generated status          |                                                                                                                           |
| Obtain the generated status          | HTTP Request    | Polls API for generation status        | Wait for Video Processing Completion | Check if Video Generation is Complete |                                                                                                                           |
| Check if Video Generation is Complete | If              | Checks if generation succeeded         | Obtain the generated status          | Format and Display Video Results (true branch), Wait for Video Processing Completion (false branch) |                                                                                                                           |
| Format and Display Video Results     | Set             | Extracts and formats result URLs       | Check if Video Generation is Complete | None                                |                                                                                                                           |
| Sticky Note6                        | Sticky Note     | Instruction: Obtain API key             | None                                | None                                | ## STEP 1 - GET API KEY (YOURAPIKEY) - Create an account [here](https://kie.ai/) and obtain API KEY. Keep it safe.          |
| Sticky Note                        | Sticky Note     | Instruction: Usage process overview     | None                                | None                                | ## STEP 2 - Usage Process - Detailed steps from starting workflow to obtaining results.                                    |
| Sticky Note3                       | Sticky Note     | Detailed workflow overview and tips    | None                                | None                                | ## Generate AI Images & Videos with KIE.AI Midjourney API - Comprehensive description, modes, setup, and troubleshooting.  |
| Sticky Note1                       | Sticky Note     | Explanation of form parameters          | None                                | None                                | ## STEP 3 - Form Parameters - Details on prompt, img_url, task_type, and api_key with examples and tips.                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node**  
   - Name: `Submit Text Prompt for Video Generation`  
   - Type: Form Trigger (version 2.2)  
   - Configure Form:  
     - Title: "AI video generator"  
     - Description: "Please fill in the following information to generate your video"  
     - Fields:  
       - `prompt` (text, required)  
       - `img_url` (text, optional)  
       - `task_type` (text, required), placeholder: "mj_txt2img"  
       - `api_key` (text, required)  
   - This node listens for form submissions and triggers the workflow.

2. **Create HTTP Request Node to Send Generation Request**  
   - Name: `Send Video Generation Request to KIE.AI API`  
   - Type: HTTP Request (version 4.2)  
   - Method: POST  
   - URL: `https://api.kie.ai/api/v1/mj/generate`  
   - Headers:  
     - `Content-Type`: application/json  
     - `Authorization`: `Bearer {{$json.api_key}}` (expression referencing form input)  
   - Body (JSON, raw mode):  
   ```json
   {
     "taskType": "{{$json.task_type}}",
     "speed": "fast",
     "prompt": "{{$json.prompt}}",
     "fileUrls": "{{$json.img_url ? [$json.img_url] : []}}",
     "aspectRatio": "16:9",
     "version": "7",
     "variety": 10,
     "stylization": 1,
     "weirdness": 1,
     "waterMark": "",
     "callBackUrl": "https://api.example.com/callback",
     "ow": 500,
     "videoBatchSize": 1
   }
   ```  
   - Connect output of Form Trigger node to this node.

3. **Create Wait Node**  
   - Name: `Wait for Video Processing Completion`  
   - Type: Wait (version 1.1)  
   - Configure to wait for 10 seconds.  
   - Connect output of HTTP Request node to this node.

4. **Create HTTP Request Node to Poll Status**  
   - Name: `Obtain the generated status`  
   - Type: HTTP Request (version 4.2)  
   - Method: GET  
   - URL: `https://api.kie.ai/api/v1/mj/record-info`  
   - Query Parameters:  
     - `taskId`: `={{$json.data.taskId}}` (expression from previous API response)  
   - Headers:  
     - `Content-Type`: application/json  
     - `Authorization`: `Bearer {{$node["Submit Text Prompt for Video Generation"].json["api_key"]}}` (reference original API key)  
   - Connect output of Wait node to this node.

5. **Create If Node to Check Completion**  
   - Name: `Check if Video Generation is Complete`  
   - Type: If (version 2.2)  
   - Condition: Check if `data.successFlag == 1` (Expression: `={{ $json.data.successFlag == 1 }}`)  
   - Connect output of status HTTP Request node to this node.

6. **Create Set Node to Format Results**  
   - Name: `Format and Display Video Results`  
   - Type: Set (version 3.4)  
   - Assign variables:  
     - `img1`: `={{ $json.data.resultInfoJson.resultUrls[0].resultUrl }}`  
     - `img2`: `={{ $json.data.resultInfoJson.resultUrls[1].resultUrl }}`  
     - `img3`: `={{ $json.data.resultInfoJson.resultUrls[2].resultUrl }}`  
     - `img4`: `={{ $json.data.resultInfoJson.resultUrls[3].resultUrl }}`  
   - Connect True output of If node to this node.

7. **Connect False Output of If Node Back to Wait Node**  
   - This creates a polling loop that retries every 10 seconds until generation completes.

8. **Add Sticky Notes for User Guidance (Optional but Recommended)**  
   - Create four Sticky Note nodes positioned logically around the workflow. Add content as follows:  
     - Sticky Note with instructions to get API key from https://kie.ai/  
     - Sticky Note explaining usage steps from starting workflow to receiving results  
     - Sticky Note with detailed overview, supported modes, customization tips, and troubleshooting  
     - Sticky Note describing form input parameters with examples and tips  

9. **Credentials Setup**  
   - No specific credentials node needed, as API key is supplied dynamically from form input.  
   - Ensure HTTP Request nodes do not use static credentials but instead use the API key from form data for Authorization header.

10. **Activate and Test Workflow**  
    - Execute workflow and open the form URL generated by the Form Trigger node.  
    - Fill in the fields according to usage instructions.  
    - Submit and observe workflow polling and final results output.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                   |
|---------------------------------------------------------------------------------------------------------------|--------------------------------------------------|
| KIE.AI Account registration and API key generation required to use this workflow.                              | https://kie.ai/                                   |
| Workflow supports three AI generation modes: text-to-image, image-to-image, image-to-video.                     | Workflow overview sticky note                     |
| Video generation can take several minutes; workflow polls every 10 seconds to check status.                    | Troubleshooting sticky note                        |
| Ensure images URLs supplied are publicly accessible and properly formatted for image-to-image/video modes.     | Form parameters sticky note                        |
| Example detailed prompt for improved quality: “Cinematic portrait of a cyberpunk character with neon blue lighting...” | Customization tips in overview sticky note        |
| Workflow is designed to be user-friendly and requires no coding skills from end users.                         | Usage process sticky note                          |

---

**Disclaimer:** The text provided is extracted exclusively from an automated workflow created with n8n, a workflow automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.