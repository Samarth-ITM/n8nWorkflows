General 3D Presentation Workflow with Midjourney, GPT-4o-image and Kling APIs

https://n8nworkflows.xyz/workflows/general-3d-presentation-workflow-with-midjourney--gpt-4o-image-and-kling-apis-3625


# General 3D Presentation Workflow with Midjourney, GPT-4o-image and Kling APIs

### 1. Workflow Overview

This workflow automates the creation of 360° or 180° spinning videos of high-quality 3D models using the PiAPI platform. It is designed to serve designers, online shoppers, content creators, and 3D beginners by generating inspiring 3D model visuals and converting them into engaging spinning animations.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Manual trigger to start the workflow and initial prompt submission to PiAPI.
- **1.2 Midjourney Image Generation:** Request and monitor image generation using the Midjourney model via PiAPI.
- **1.3 Image Processing with GPT-4o-image:** Convert the generated image into a detailed 3D figurine image using GPT-4o-image API.
- **1.4 Video Generation with Kling API:** Generate a spinning video of the 3D figurine image using the Kling model.
- **1.5 Video Retrieval and Finalization:** Poll for video generation completion and extract final video URLs.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block initiates the workflow manually and sends the user’s 3D model prompt to PiAPI’s Midjourney model to start image generation.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’  
  - Prompt

- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Starts the workflow on user command.  
    - Configuration: Default manual trigger, no parameters.  
    - Inputs: None  
    - Outputs: Connected to Prompt node.  
    - Edge Cases: User forgetting to trigger; no input validation here.

  - **Prompt**  
    - Type: HTTP Request  
    - Role: Sends a POST request to PiAPI to create an image generation task using Midjourney model.  
    - Configuration:  
      - URL: `https://api.piapi.ai/api/v1/task`  
      - Method: POST  
      - Body: JSON with model "midjourney", task_type "imagine", and user prompt describing the 3D model.  
      - Headers: Requires `x-api-key` for authentication (user must fill).  
      - Prompt example included in JSON body for reference.  
    - Expressions: Static JSON body with embedded prompt string.  
    - Inputs: From manual trigger.  
    - Outputs: Task creation response with task_id.  
    - Edge Cases: API key missing or invalid, network errors, prompt formatting errors.  
    - Version: HTTP Request node v4.2.

---

#### 2.2 Midjourney Image Generation

- **Overview:**  
  This block polls the Midjourney task status until the image generation completes, then extracts a random temporary image URL from the output.

- **Nodes Involved:**  
  - Midjourney Generator  
  - Check Generation Status  
  - Wait for Image Generation  
  - Get Image URL of Midjourney

- **Node Details:**

  - **Midjourney Generator**  
    - Type: HTTP Request  
    - Role: Polls PiAPI for the status of the Midjourney image generation task.  
    - Configuration:  
      - URL dynamically set to `https://api.piapi.ai/api/v1/task/{{ $json.data.task_id }}` using the task_id from Prompt node.  
      - Headers: Requires `x-api-key`.  
    - Inputs: From Prompt or Wait for Image Generation node.  
    - Outputs: Current task status and output data.  
    - Edge Cases: API key issues, task_id missing, timeouts.  
    - Version: v4.2.

  - **Check Generation Status**  
    - Type: If node  
    - Role: Checks if the Midjourney task status equals "completed".  
    - Configuration: Condition on `$json.data.status === "completed"`.  
    - Inputs: From Midjourney Generator.  
    - Outputs:  
      - True branch: Proceed to extract image URL.  
      - False branch: Wait and retry.  
    - Edge Cases: Unexpected status values, JSON parsing errors.  
    - Version: v2.2.

  - **Wait for Image Generation**  
    - Type: Wait  
    - Role: Pauses workflow for a webhook-based wait before retrying status check.  
    - Configuration: Uses webhook ID for asynchronous waiting.  
    - Inputs: From Check Generation Status (false branch).  
    - Outputs: Loops back to Midjourney Generator.  
    - Edge Cases: Webhook failures, timeouts.  
    - Version: v1.1.

  - **Get Image URL of Midjourney**  
    - Type: Code (JavaScript)  
    - Role: Extracts a random temporary image URL from the Midjourney task output array.  
    - Configuration:  
      - Selects a random URL from `data.output.temporary_image_urls`.  
      - Outputs `random_temp_url`.  
    - Inputs: From Check Generation Status (true branch).  
    - Outputs: To GPT-4o Image Generator.  
    - Edge Cases: Empty or missing image URL array, JSON structure changes.  
    - Version: v2.

---

#### 2.3 Image Processing with GPT-4o-image

- **Overview:**  
  This block sends the selected Midjourney image URL to GPT-4o-image API to convert it into a detailed 3D figurine image with front and profile views.

- **Nodes Involved:**  
  - GPT-4o Image Generator  
  - Get Image URL of GPT-4o-image  
  - Check Generation Status of GPT-4o

- **Node Details:**

  - **GPT-4o Image Generator**  
    - Type: HTTP Request  
    - Role: Sends a chat completion request to PiAPI’s GPT-4o-image-preview model with the Midjourney image URL and a prompt to convert it.  
    - Configuration:  
      - URL: `https://api.piapi.ai/v1/chat/completions`  
      - Method: POST  
      - Body: JSON with model "gpt-4o-image-preview", messages including image URL and conversion prompt, streaming enabled.  
      - Headers: Requires `Authorization` header with Bearer token (user must fill).  
      - Authentication: HTTP Header Auth credential configured.  
    - Inputs: From Get Image URL of Midjourney node.  
    - Outputs: Streamed response with image data.  
    - Edge Cases: Auth failures, streaming errors, malformed responses.  
    - Version: v4.2.

  - **Get Image URL of GPT-4o-image**  
    - Type: Code (JavaScript)  
    - Role: Parses the streamed GPT-4o-image response to extract the generated image URL from markdown image syntax.  
    - Configuration:  
      - Splits response by double newlines, iterates backwards to find JSON data with image URL.  
      - Returns `image_url` and `finish_reason` ("success" or "not_found").  
    - Inputs: From GPT-4o Image Generator.  
    - Outputs: To Check Generation Status of GPT-4o.  
    - Edge Cases: Parsing errors, missing URL, incomplete stream.  
    - Version: v2.

  - **Check Generation Status of GPT-4o**  
    - Type: If node  
    - Role: Checks if the image URL extraction failed (`finish_reason === "not_found"`).  
    - Configuration: Condition on `$json.finish_reason === "not_found"`.  
    - Inputs: From Get Image URL of GPT-4o-image.  
    - Outputs:  
      - True branch: Retry GPT-4o Image Generator.  
      - False branch: Proceed to Generate Kling Video.  
    - Edge Cases: Infinite retry loops if image never generated.  
    - Version: v2.2.

---

#### 2.4 Video Generation with Kling API

- **Overview:**  
  This block sends the 3D figurine image URL to the Kling model to generate a spinning video showcasing 3D details.

- **Nodes Involved:**  
  - Generate Kling Video  
  - Get Video URL  
  - Check Video Generation Status  
  - Wait for Video Generation  
  - Fetch Final Video URL

- **Node Details:**

  - **Generate Kling Video**  
    - Type: HTTP Request  
    - Role: Creates a video generation task on PiAPI using the Kling model with the 3D figurine image URL.  
    - Configuration:  
      - URL: `https://api.piapi.ai/api/v1/task`  
      - Method: POST  
      - Body: JSON with model "kling", task_type "video_generation", input includes image_url, aspect_ratio "9:16", prompt describing anime character rotation.  
      - Headers: Requires `x-api-key`.  
    - Inputs: From Check Generation Status of GPT-4o (false branch).  
    - Outputs: Task creation response with task_id.  
    - Edge Cases: API key issues, malformed image URL, API errors.  
    - Version: v4.2.

  - **Get Video URL**  
    - Type: HTTP Request  
    - Role: Polls PiAPI for the status of the Kling video generation task.  
    - Configuration:  
      - URL dynamically set to `https://api.piapi.ai/api/v1/task/{{ $json.data.task_id }}`.  
      - Headers: `x-api-key` included (hardcoded in this node, user should replace with credential).  
    - Inputs: From Generate Kling Video or Wait for Video Generation.  
    - Outputs: Current task status and output data.  
    - Edge Cases: API key hardcoding may cause security issues; task_id missing; network errors.  
    - Version: v4.2.

  - **Check Video Generation Status**  
    - Type: If node  
    - Role: Checks if the Kling video generation task status equals "completed".  
    - Configuration: Condition on `$json.data.status === "completed"`.  
    - Inputs: From Get Video URL.  
    - Outputs:  
      - True branch: Proceed to fetch final video URLs.  
      - False branch: Wait and retry.  
    - Edge Cases: Unexpected status values, JSON parsing errors.  
    - Version: v2.2.

  - **Wait for Video Generation**  
    - Type: Wait  
    - Role: Pauses workflow for 20 seconds before retrying video status check.  
    - Configuration: Fixed 20 seconds delay.  
    - Inputs: From Check Video Generation Status (false branch).  
    - Outputs: Loops back to Get Video URL.  
    - Edge Cases: Long wait times, webhook failures.  
    - Version: v1.1.

  - **Fetch Final Video URL**  
    - Type: Code (JavaScript)  
    - Role: Extracts final video URLs from the completed Kling video generation task output.  
    - Configuration:  
      - Extracts `video_url` and `watermark_free_url` from nested JSON fields.  
    - Inputs: From Check Video Generation Status (true branch).  
    - Outputs: Final video URLs for user consumption.  
    - Edge Cases: Missing or changed JSON structure, null values.  
    - Version: v2.

---

### 3. Summary Table

| Node Name                    | Node Type           | Functional Role                               | Input Node(s)                   | Output Node(s)                  | Sticky Note                                                                                       |
|------------------------------|---------------------|-----------------------------------------------|--------------------------------|--------------------------------|-------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger      | Initiates the workflow manually                | None                           | Prompt                         | ## General 3D Presentation<br>This workflow creates 360° or 180° spinning videos of high-quality 3D models with [PiAPI](https://piapi.ai) API. 🙋<br>### Required Instruction: <br>1. Fill in params of Prompt node.<br>2. Fill in x-api-key in Mijdourney Generator node and Generate Kling Video node, fill in Header Parameters of GPT-4o Image Generator (e.g., Bearer + your X-API-Key) |
| Prompt                       | HTTP Request        | Sends initial Midjourney image generation task | When clicking ‘Test workflow’  | Midjourney Generator           |                                                                                                 |
| Midjourney Generator          | HTTP Request        | Polls Midjourney task status                    | Prompt, Wait for Image Generation | Check Generation Status         |                                                                                                 |
| Check Generation Status       | If                  | Checks if Midjourney image generation is done  | Midjourney Generator           | Get Image URL of Midjourney, Wait for Image Generation |                                                                                                 |
| Wait for Image Generation     | Wait                | Waits before retrying Midjourney status check  | Check Generation Status (false) | Midjourney Generator           |                                                                                                 |
| Get Image URL of Midjourney   | Code                | Extracts random temporary image URL             | Check Generation Status (true) | GPT-4o Image Generator         |                                                                                                 |
| GPT-4o Image Generator        | HTTP Request        | Converts image to 3D figurine image             | Get Image URL of Midjourney    | Get Image URL of GPT-4o-image  |                                                                                                 |
| Get Image URL of GPT-4o-image | Code                | Parses GPT-4o-image response to extract image URL | GPT-4o Image Generator         | Check Generation Status of GPT-4o |                                                                                                 |
| Check Generation Status of GPT-4o | If              | Checks if GPT-4o-image generation succeeded     | Get Image URL of GPT-4o-image  | GPT-4o Image Generator, Generate Kling Video |                                                                                                 |
| Generate Kling Video          | HTTP Request        | Starts Kling video generation task               | Check Generation Status of GPT-4o (false) | Get Video URL                  |                                                                                                 |
| Get Video URL                | HTTP Request        | Polls Kling video generation status              | Generate Kling Video, Wait for Video Generation | Check Video Generation Status   |                                                                                                 |
| Check Video Generation Status | If                  | Checks if Kling video generation is done         | Get Video URL                  | Fetch Final Video URL, Wait for Video Generation |                                                                                                 |
| Wait for Video Generation     | Wait                | Waits 20 seconds before retrying video status    | Check Video Generation Status (false) | Get Video URL                  |                                                                                                 |
| Fetch Final Video URL         | Code                | Extracts final video URLs from Kling output      | Check Video Generation Status (true) | None                          |                                                                                                 |
| Sticky Note                  | Sticky Note         | Provides workflow overview and instructions      | None                           | None                          | ## General 3D Presentation<br>This workflow creates 360° or 180° spinning videos of high-quality 3D models with [PiAPI](https://piapi.ai) API. 🙋<br>### Required Instruction: <br>1. Fill in params of Prompt node.<br>2. Fill in x-api-key in Mijdourney Generator node and Generate Kling Video node, fill in Header Parameters of GPT-4o Image Generator (e.g., Bearer + your X-API-Key) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: `When clicking ‘Test workflow’`  
   - Type: Manual Trigger  
   - No parameters needed.

2. **Create HTTP Request Node for Prompt**  
   - Name: `Prompt`  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.piapi.ai/api/v1/task`  
   - Body Content Type: JSON  
   - Body:  
     ```json
     {
       "model": "midjourney",
       "task_type": "imagine",
       "input": {
         "prompt": "A blind box character design, in the chibi style, a super cute little girl wearing a white long-sleeved dress and pearl earrings with her head bowed in a prayer pose, facing upwards, wearing an oversized off-white dress with large round pearls on the shoulders, minimalist simple dress with Ruffles, against a beige background, a full-body shot in a three-quarter profile view, with a black, blue, and gray color scheme, soft lighting, 3D rendering, clay material, high detail, in the Pixar style. Clean white skin, brown renaissance braided bun. --ar 1:1 --niji 6",
         "aspect_ratio": "2:3",
         "process_mode": "turbo",
         "skip_prompt_check": false
       }
     }
     ```  
   - Headers: Add `x-api-key` parameter (to be filled by user).  
   - Connect input from Manual Trigger.

3. **Create HTTP Request Node for Midjourney Generator**  
   - Name: `Midjourney Generator`  
   - Type: HTTP Request  
   - Method: GET (default)  
   - URL: Expression: `https://api.piapi.ai/api/v1/task/{{ $json.data.task_id }}`  
   - Headers: Add `x-api-key` parameter (same as Prompt node).  
   - Connect input from `Prompt` node and from `Wait for Image Generation` node (to enable polling).

4. **Create If Node for Check Generation Status**  
   - Name: `Check Generation Status`  
   - Type: If  
   - Condition: `$json.data.status === "completed"`  
   - Connect input from `Midjourney Generator` node.  
   - True branch connects to `Get Image URL of Midjourney`.  
   - False branch connects to `Wait for Image Generation`.

5. **Create Wait Node for Image Generation**  
   - Name: `Wait for Image Generation`  
   - Type: Wait  
   - Use webhook mode (configure webhook ID automatically).  
   - Connect input from `Check Generation Status` (false branch).  
   - Output connects back to `Midjourney Generator` for polling.

6. **Create Code Node for Get Image URL of Midjourney**  
   - Name: `Get Image URL of Midjourney`  
   - Type: Code  
   - Language: JavaScript  
   - Code:  
     ```js
     return {
       random_temp_url: $input.all()[0].json.data.output.temporary_image_urls[
         Math.floor(Math.random() * $input.all()[0].json.data.output.temporary_image_urls.length)
       ]
     };
     ```  
   - Connect input from `Check Generation Status` (true branch).  
   - Output connects to `GPT-4o Image Generator`.

7. **Create HTTP Request Node for GPT-4o Image Generator**  
   - Name: `GPT-4o Image Generator`  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.piapi.ai/v1/chat/completions`  
   - Body Content Type: JSON  
   - Body (expression):  
     ```json
     {
       "model": "gpt-4o-image-preview",
       "messages": [
         {
           "role": "user",
           "content": [
             {
               "type": "image_url",
               "image_url": {
                 "url": "{{ $json.random_temp_url }}"
               }
             },
             {
               "type": "text",
               "text": "Convert this image into a 3D figurine image, with front view, with full details, profile."
             }
           ]
         }
       ],
       "stream": true
     }
     ```  
   - Headers: Add `Authorization` header (Bearer token) via HTTP Header Auth credential.  
   - Connect input from `Get Image URL of Midjourney`.  
   - Output connects to `Get Image URL of GPT-4o-image`.

8. **Create Code Node for Get Image URL of GPT-4o-image**  
   - Name: `Get Image URL of GPT-4o-image`  
   - Type: Code  
   - Language: JavaScript  
   - Code:  
     ```js
     const chunks = $input.first().json.data.split('\n\n');

     let imageUrl = null;

     for (let i = chunks.length - 1; i >= 0; i--) {
         const chunk = chunks[i];
         
         if (!chunk.startsWith('data: ')) continue;
         
         try {
             const jsonStr = chunk.substring(6); 
             if (jsonStr.trim() === '[DONE]') continue;
             
             const data = JSON.parse(jsonStr);
             
         
             if (data.choices && data.choices[0].delta.content) {
                 const content = data.choices[0].delta.content;
                 const urlMatch = content.match(/!\[.*?\]\((https?:\/\/[^\s]+)\)/);
                 
                 if (urlMatch && urlMatch[1]) {
                     imageUrl = urlMatch[1];
                     break;
                 }
             }
         } catch (e) {
             continue;
         }
     }

     return {
         image_url: imageUrl,
         finish_reason: imageUrl ? "success" : "not_found"
     };
     ```  
   - Connect input from `GPT-4o Image Generator`.  
   - Output connects to `Check Generation Status of GPT-4o`.

9. **Create If Node for Check Generation Status of GPT-4o**  
   - Name: `Check Generation Status of GPT-4o`  
   - Type: If  
   - Condition: `$json.finish_reason === "not_found"`  
   - Connect input from `Get Image URL of GPT-4o-image`.  
   - True branch loops back to `GPT-4o Image Generator` (retry).  
   - False branch connects to `Generate Kling Video`.

10. **Create HTTP Request Node for Generate Kling Video**  
    - Name: `Generate Kling Video`  
    - Type: HTTP Request  
    - Method: POST  
    - URL: `https://api.piapi.ai/api/v1/task`  
    - Body Content Type: JSON  
    - Body (expression):  
      ```json
      {
        "model": "kling",
        "task_type": "video_generation",
        "input": {
          "version": "1.6",
          "aspect_ratio": "9:16",
          "image_url": "{{ $json.image_url }}",
          "prompt": "An anime character anchored mid-frame, gradually rotating to showcase 3D details"
        }
      }
      ```  
    - Headers: Add `x-api-key` parameter (user must fill).  
    - Connect input from `Check Generation Status of GPT-4o` (false branch).  
    - Output connects to `Get Video URL`.

11. **Create HTTP Request Node for Get Video URL**  
    - Name: `Get Video URL`  
    - Type: HTTP Request  
    - Method: GET (default)  
    - URL: Expression: `https://api.piapi.ai/api/v1/task/{{ $json.data.task_id }}`  
    - Headers: Add `x-api-key` parameter (replace hardcoded key with credential recommended).  
    - Connect input from `Generate Kling Video` and from `Wait for Video Generation` (for polling).  
    - Output connects to `Check Video Generation Status`.

12. **Create If Node for Check Video Generation Status**  
    - Name: `Check Video Generation Status`  
    - Type: If  
    - Condition: `$json.data.status === "completed"`  
    - Connect input from `Get Video URL`.  
    - True branch connects to `Fetch Final Video URL`.  
    - False branch connects to `Wait for Video Generation`.

13. **Create Wait Node for Video Generation**  
    - Name: `Wait for Video Generation`  
    - Type: Wait  
    - Duration: 20 seconds  
    - Connect input from `Check Video Generation Status` (false branch).  
    - Output connects back to `Get Video URL`.

14. **Create Code Node for Fetch Final Video URL**  
    - Name: `Fetch Final Video URL`  
    - Type: Code  
    - Language: JavaScript  
    - Code:  
      ```js
      return {
        video_url: $input.all()[0].json.data.output.video_url,
        watermark_free_url: $input.all()[0].json.data.output.works[0].video.resource_without_watermark
      };
      ```  
    - Connect input from `Check Video Generation Status` (true branch).  
    - Output: Final video URLs for user.

15. **Create Sticky Note**  
    - Content:  
      ```
      ## General 3D Presentation
      This workflow creates 360° or 180° spinning videos of high-quality 3D models with https://piapi.ai API. 🙋
      ### Required Instruction: 
      1. Fill in params of Prompt node.
      2. Fill in x-api-key in Mijdourney Generator node and Generate Kling Video node, fill in Header Parameters of GPT-4o Image Generator (e.g., Bearer + your X-API-Key)
      ```  
    - Position near the top for user reference.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                      | Context or Link                                         |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------|
| This workflow requires valid API keys for PiAPI endpoints: `x-api-key` for Midjourney and Kling API nodes, and `Authorization` header (Bearer token) for GPT-4o Image Generator.                                                    | API Authentication                                     |
| Example prompt included in the Prompt node demonstrates detailed 3D character description with style and rendering instructions.                                                                                                 | Prompt Example                                          |
| The workflow uses webhook-based wait nodes for polling Midjourney image generation and fixed delay wait for Kling video generation to handle asynchronous API processing times.                                                     | Asynchronous Polling Strategy                           |
| The GPT-4o Image Generator node uses streaming response parsing to extract image URLs from partial data chunks, which may require careful error handling if API response format changes.                                         | Streaming Response Parsing                              |
| Hardcoded API key in Get Video URL node should be replaced with a credential for security best practices.                                                                                                                        | Security Best Practice                                  |
| PiAPI official site: https://piapi.ai                                                                                                                                                                                            | Official API Documentation                              |
| Example output video and reference videos are linked in the workflow description for visual confirmation of expected results.                                                                                                    | Example Videos                                          |

---

This documentation provides a complete and detailed reference to understand, reproduce, and modify the "General 3D Presentation Workflow with Midjourney, GPT-4o-image and Kling APIs" in n8n. It anticipates potential failure points such as API authentication errors, asynchronous polling challenges, and response parsing issues, enabling robust workflow management and extension.