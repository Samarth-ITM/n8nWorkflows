Create Animated Illustrations from Text Prompts with Midjourney and Kling API

https://n8nworkflows.xyz/workflows/create-animated-illustrations-from-text-prompts-with-midjourney-and-kling-api-3626


# Create Animated Illustrations from Text Prompts with Midjourney and Kling API

### 1. Workflow Overview

This workflow automates the creation of animated illustrations by integrating Midjourney’s image generation and Kling’s video generation APIs, both accessed via the PiAPI platform. It is designed for content creators, social media professionals, and digital marketers who want to generate visually appealing animated artwork from textual prompts without manual intervention or complex configurations.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Manual trigger to start the workflow.
- **1.2 Image Generation with Midjourney:** Sends a text prompt to Midjourney API to generate a static illustration image.
- **1.3 Image Generation Status Polling:** Waits and checks for the completion of the image generation task.
- **1.4 Image Extraction:** Selects a random temporary image URL from the generated images.
- **1.5 Video Generation with Kling:** Uses the selected image and a motion prompt to generate an animated video.
- **1.6 Video Generation Status Polling:** Waits and checks for the completion of the video generation task.
- **1.7 Final Video URL Extraction:** Extracts the final video URL and a watermark-free version for output.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block initiates the workflow manually via a trigger node.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’

- **Node Details:**  
  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Starts the workflow on user command.  
    - Configuration: No parameters required.  
    - Inputs: None  
    - Outputs: Connects to Basic Prompt node.  
    - Edge Cases: None, but workflow will not run unless triggered manually.

---

#### 2.2 Image Generation with Midjourney

- **Overview:**  
  Sends a textual prompt to Midjourney API via PiAPI to generate an illustration image.

- **Nodes Involved:**  
  - Basic Prompt  
  - Midjourney Image Generator

- **Node Details:**  
  - **Basic Prompt**  
    - Type: HTTP Request  
    - Role: Sends a POST request to PiAPI Midjourney endpoint to start image generation.  
    - Configuration:  
      - URL: `https://api.piapi.ai/api/v1/task`  
      - Method: POST  
      - Body (JSON): Contains model "midjourney", task_type "imagine", and input prompt with aspect ratio and processing mode.  
      - Header: Requires `x-api-key` (user must provide their PiAPI API key).  
      - Prompt is customizable by the user.  
    - Inputs: From manual trigger  
    - Outputs: Task ID and status data to Midjourney Image Generator.  
    - Edge Cases: API key missing or invalid, prompt syntax errors, API downtime.

  - **Midjourney Image Generator**  
    - Type: HTTP Request  
    - Role: Polls the Midjourney task status using the task ID to check if image generation is complete.  
    - Configuration:  
      - URL: `https://api.piapi.ai/api/v1/task/{{ $json.data.task_id }}` (dynamic)  
      - Method: GET (default)  
      - Header: Requires `x-api-key`.  
    - Inputs: Task ID from Basic Prompt node.  
    - Outputs: Task status and output data to Get Data Status node.  
    - Edge Cases: API key issues, network timeouts, invalid task ID.

---

#### 2.3 Image Generation Status Polling

- **Overview:**  
  Waits and verifies if the image generation task is completed before proceeding.

- **Nodes Involved:**  
  - Get Data Status  
  - Wait for Image Generation

- **Node Details:**  
  - **Get Data Status**  
    - Type: If Node  
    - Role: Checks if the `data.status` field equals "completed".  
    - Configuration:  
      - Condition: `$json.data.status === "completed"`  
    - Inputs: From Midjourney Image Generator  
    - Outputs:  
      - True branch: proceeds to Get Image node.  
      - False branch: proceeds to Wait for Image Generation node.  
    - Edge Cases: Unexpected status values, missing data fields.

  - **Wait for Image Generation**  
    - Type: Wait Node  
    - Role: Pauses workflow execution until webhook is triggered or manually resumed.  
    - Configuration:  
      - Webhook ID assigned (used for external trigger to continue).  
    - Inputs: False branch from Get Data Status  
    - Outputs: Loops back to Midjourney Image Generator for re-check.  
    - Edge Cases: Workflow stuck if webhook never triggered, manual intervention needed.

---

#### 2.4 Image Extraction

- **Overview:**  
  Selects a random temporary image URL from the generated images to be used for video generation.

- **Nodes Involved:**  
  - Get Image

- **Node Details:**  
  - **Get Image**  
    - Type: Code (JavaScript)  
    - Role: Extracts a random image URL from the array of temporary image URLs returned by Midjourney.  
    - Configuration:  
      - JavaScript code picks a random index from `data.output.temporary_image_urls` array.  
      - Returns `{ random_temp_url: selected_url }`.  
    - Inputs: From Get Data Status (true branch)  
    - Outputs: Passes selected image URL to Kling Video Generator.  
    - Edge Cases: Empty or missing image URL array, code execution errors.

---

#### 2.5 Video Generation with Kling

- **Overview:**  
  Uses the selected image URL and a motion prompt to request animated video generation from Kling API via PiAPI.

- **Nodes Involved:**  
  - Kling Video Generator  
  - Get Video

- **Node Details:**  
  - **Kling Video Generator**  
    - Type: HTTP Request  
    - Role: Sends a POST request to PiAPI Kling endpoint to start video generation.  
    - Configuration:  
      - URL: `https://api.piapi.ai/api/v1/task`  
      - Method: POST  
      - Body (JSON): Includes model "kling", task_type "video_generation", input version "1.6", `image_url` from previous node, and a motion prompt.  
      - Header: Requires `x-api-key`.  
      - Motion prompt is customizable by the user.  
    - Inputs: From Get Image node (random_temp_url)  
    - Outputs: Task ID and status data to Get Video node.  
    - Edge Cases: API key missing/invalid, invalid image URL, prompt errors.

  - **Get Video**  
    - Type: HTTP Request  
    - Role: Polls the Kling video generation task status using task ID.  
    - Configuration:  
      - URL: `https://api.piapi.ai/api/v1/task/{{ $json.data.task_id }}` (dynamic)  
      - Header: Requires `x-api-key`.  
    - Inputs: From Kling Video Generator  
    - Outputs: Task status and output data to Verify Data Status node.  
    - Edge Cases: API key issues, network timeouts, invalid task ID.

---

#### 2.6 Video Generation Status Polling

- **Overview:**  
  Waits and verifies if the video generation task is completed before extracting the final video URL.

- **Nodes Involved:**  
  - Verify Data Status  
  - Wait for Video Generation

- **Node Details:**  
  - **Verify Data Status**  
    - Type: If Node  
    - Role: Checks if `data.status` equals "completed".  
    - Configuration:  
      - Condition: `$json.data.status === "completed"`  
    - Inputs: From Get Video node  
    - Outputs:  
      - True branch: proceeds to Get Final Video URL node.  
      - False branch: proceeds to Wait for Video Generation node.  
    - Edge Cases: Unexpected status values, missing data fields.

  - **Wait for Video Generation**  
    - Type: Wait Node  
    - Role: Pauses workflow execution for a fixed duration (20 seconds) before rechecking.  
    - Configuration:  
      - Amount: 20 seconds  
    - Inputs: False branch from Verify Data Status  
    - Outputs: Loops back to Get Video node for re-check.  
    - Edge Cases: Extended wait times if generation is slow, potential timeouts.

---

#### 2.7 Final Video URL Extraction

- **Overview:**  
  Extracts the final video URL and a watermark-free video URL from the completed video generation response.

- **Nodes Involved:**  
  - Get Final Video URL

- **Node Details:**  
  - **Get Final Video URL**  
    - Type: Code (JavaScript)  
    - Role: Extracts URLs for the generated video and a watermark-free version from the API response.  
    - Configuration:  
      - JavaScript code accesses `data.output.video_url` and `data.output.works[0].video.resource_without_watermark`.  
      - Returns `{ video_url, watermark_free_url }`.  
    - Inputs: True branch from Verify Data Status  
    - Outputs: Final output containing video URLs for downstream use or display.  
    - Edge Cases: Missing or malformed output fields, multiple video works array empty.

---

### 3. Summary Table

| Node Name                | Node Type         | Functional Role                          | Input Node(s)                  | Output Node(s)               | Sticky Note                                                                                                         |
|--------------------------|-------------------|----------------------------------------|-------------------------------|------------------------------|---------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger    | Starts the workflow manually            | None                          | Basic Prompt                 |                                                                                                                     |
| Basic Prompt             | HTTP Request      | Sends Midjourney image generation request | When clicking ‘Test workflow’ | Midjourney Image Generator    |                                                                                                                     |
| Midjourney Image Generator | HTTP Request      | Polls Midjourney image generation status | Basic Prompt                  | Get Data Status              |                                                                                                                     |
| Get Data Status          | If                | Checks if image generation is complete | Midjourney Image Generator    | Get Image (true), Wait for Image Generation (false) |                                                                                                                     |
| Wait for Image Generation | Wait              | Waits for image generation completion   | Get Data Status (false)        | Midjourney Image Generator   |                                                                                                                     |
| Get Image                | Code              | Extracts random image URL from results  | Get Data Status (true)         | Kling Video Generator        |                                                                                                                     |
| Kling Video Generator    | HTTP Request      | Sends Kling video generation request    | Get Image                     | Get Video                   |                                                                                                                     |
| Get Video                | HTTP Request      | Polls Kling video generation status     | Kling Video Generator          | Verify Data Status           |                                                                                                                     |
| Verify Data Status       | If                | Checks if video generation is complete  | Get Video                     | Get Final Video URL (true), Wait for Video Generation (false) |                                                                                                                     |
| Wait for Video Generation | Wait              | Waits 20 seconds before rechecking video status | Verify Data Status (false)     | Get Video                   |                                                                                                                     |
| Get Final Video URL      | Code              | Extracts final video URLs                | Verify Data Status (true)      | None                        |                                                                                                                     |
| Sticky Note              | Sticky Note       | Describes workflow purpose               | None                          | None                        | ## Motion-illustration\nThis workflow is primarily designed to generate dynamic illustrations for content creators and social media professionals with APIs provided by PiAPI. |
| Sticky Note1             | Sticky Note       | Step-by-step instructions                 | None                          | None                        | ## Step-by-step Instruction\n1. Fill in your x-api-key of your PiAPI account in the Midjourney Image Generator and Kling Video Generator nodes.\n2. Enter your desired image prompt in **Basic Prompt** node.\n3. Click Test workflow. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: `When clicking ‘Test workflow’`  
   - Type: Manual Trigger  
   - No parameters needed.

2. **Create HTTP Request Node for Midjourney Image Generation**  
   - Name: `Basic Prompt`  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.piapi.ai/api/v1/task`  
   - Authentication: None (API key passed in header)  
   - Headers: Add header `x-api-key` with your PiAPI API key (credential must be created in n8n).  
   - Body Content Type: JSON  
   - Body:  
     ```json
     {
       "model": "midjourney",
       "task_type": "imagine",
       "input": {
         "prompt": "A gentle girl and a fluffy rabbit explore a sunlit forest together, playing by a sparkling stream. Butterflies flutter around them as golden sunlight filters through green leaves. Warm and peaceful atmosphere, 4K nature documentary style. --s 500 --sref 4028286908  --niji 6",
         "aspect_ratio": "1:1",
         "process_mode": "turbo",
         "skip_prompt_check": false
       }
     }
     ```
   - Connect output of Manual Trigger to this node.

3. **Create HTTP Request Node to Poll Midjourney Task Status**  
   - Name: `Midjourney Image Generator`  
   - Type: HTTP Request  
   - Method: GET (default)  
   - URL: `https://api.piapi.ai/api/v1/task/{{ $json.data.task_id }}` (use expression to get task_id from previous node)  
   - Headers: Add header `x-api-key` with your PiAPI API key.  
   - Connect output of `Basic Prompt` to this node.

4. **Create If Node to Check Image Generation Completion**  
   - Name: `Get Data Status`  
   - Type: If  
   - Condition: Check if `$json.data.status === "completed"` (string equality, case sensitive)  
   - Connect output of `Midjourney Image Generator` to this node.

5. **Create Wait Node for Image Generation**  
   - Name: `Wait for Image Generation`  
   - Type: Wait  
   - Parameters: Use webhook wait (no fixed time) with a unique webhook ID (auto-generated or manually set).  
   - Connect false output of `Get Data Status` to this node.  
   - Connect output of this wait node back to `Midjourney Image Generator` to poll again.

6. **Create Code Node to Extract Random Image URL**  
   - Name: `Get Image`  
   - Type: Code (JavaScript)  
   - Code:  
     ```javascript
     return {
       random_temp_url: $input.all()[0].json.data.output.temporary_image_urls[
         Math.floor(Math.random() * $input.all()[0].json.data.output.temporary_image_urls.length)
       ]
     };
     ```  
   - Connect true output of `Get Data Status` to this node.

7. **Create HTTP Request Node for Kling Video Generation**  
   - Name: `Kling Video Generator`  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.piapi.ai/api/v1/task`  
   - Headers: Add header `x-api-key` with your PiAPI API key.  
   - Body Content Type: JSON  
   - Body (use expression to inject image URL):  
     ```json
     {
       "model": "kling",
       "task_type": "video_generation",
       "input": {
         "version": "1.6",
         "image_url": "{{ $json.random_temp_url }}",
         "prompt": "A young girl sits on a sunlit grassy meadow, gently petting a fluffy white rabbit"
       }
     }
     ```  
   - Connect output of `Get Image` to this node.

8. **Create HTTP Request Node to Poll Kling Video Task Status**  
   - Name: `Get Video`  
   - Type: HTTP Request  
   - Method: GET (default)  
   - URL: `https://api.piapi.ai/api/v1/task/{{ $json.data.task_id }}`  
   - Headers: Add header `x-api-key` with your PiAPI API key.  
   - Connect output of `Kling Video Generator` to this node.

9. **Create If Node to Check Video Generation Completion**  
   - Name: `Verify Data Status`  
   - Type: If  
   - Condition: Check if `$json.data.status === "completed"`  
   - Connect output of `Get Video` to this node.

10. **Create Wait Node for Video Generation**  
    - Name: `Wait for Video Generation`  
    - Type: Wait  
    - Parameters: Wait for 20 seconds (fixed delay)  
    - Connect false output of `Verify Data Status` to this node.  
    - Connect output of this wait node back to `Get Video` to poll again.

11. **Create Code Node to Extract Final Video URLs**  
    - Name: `Get Final Video URL`  
    - Type: Code (JavaScript)  
    - Code:  
      ```javascript
      return {
        video_url: $input.all()[0].json.data.output.video_url,
        watermark_free_url: $input.all()[0].json.data.output.works[0].video.resource_without_watermark
      };
      ```  
    - Connect true output of `Verify Data Status` to this node.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                     | Context or Link                                                                                   |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This workflow is primarily designed to generate dynamic illustrations for content creators and social media professionals with APIs provided by PiAPI.                                                                          | Sticky Note on workflow overview                                                               |
| Step-by-step Instruction: 1. Fill in your x-api-key of your PiAPI account in the Midjourney Image Generator and Kling Video Generator nodes. 2. Enter your desired image prompt in **Basic Prompt** node. 3. Click Test workflow. | Sticky Note with user instructions                                                             |
| PiAPI platform provides professional API services to simplify AI model usage for image and video generation.                                                                                                                   | https://piapi.ai                                                                                |
| Recommended video model for motion prompt: live-wallpaper LoRA of Wanx. See PiAPI docs for more video and image model use cases.                                                                                                | https://piapi.ai/docs/wanx-lora/use-case                                                       |
| Troubleshooting tips: 1. Verify that the X-API-Key is correctly filled in all required nodes. 2. Check task status in PiAPI Task History for detailed status and error information.                                              | https://piapi.ai and workflow description images                                               |
| Example input prompt and output video are provided to illustrate expected results.                                                                                                                                               | Example prompt and video links in workflow description                                         |

---

This document fully describes the workflow structure, node configurations, and operational logic to enable reproduction, modification, and troubleshooting by advanced users and automation agents.