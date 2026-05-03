Convert 3-View Drawings to 360° Videos with GPT-4o-Image and Kling API

https://n8nworkflows.xyz/workflows/convert-3-view-drawings-to-360--videos-with-gpt-4o-image-and-kling-api-3716


# Convert 3-View Drawings to 360° Videos with GPT-4o-Image and Kling API

### 1. Workflow Overview

This workflow automates the conversion of orthographic three-view drawings (front and side views) into dynamic 360° rotation videos using PiAPI’s GPT-4o-Image and Kling APIs. It is designed to help designers, online shoppers, and content creators visualize 3D models efficiently by generating rotating videos from static 2D images.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Initialization:** Manual trigger and setting of basic parameters including API key and input image URL.
- **1.2 Front View Image Generation:** Uses GPT-4o-Image API to generate the front view image and extracts the image URL.
- **1.3 Side View Image Generation:** Uses GPT-4o-Image API to generate the side view image and extracts the image URL.
- **1.4 Video Generation Request:** Sends a video generation task to the Kling API using the generated images.
- **1.5 Video Generation Polling and Retrieval:** Periodically checks the status of the video generation task and retrieves the final video URL once completed.
- **1.6 Output Processing:** Extracts and formats the final video URLs for downstream use.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Initialization

- **Overview:** This block starts the workflow manually and sets the basic parameters required for API calls.
- **Nodes Involved:**  
  - When clicking ‘Test workflow’  
  - Basic Params

- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Entry point to manually start the workflow.  
    - Configuration: No parameters; triggers workflow on user action.  
    - Inputs: None  
    - Outputs: Connects to Basic Params node.  
    - Edge Cases: User must manually trigger; no automatic start.

  - **Basic Params**  
    - Type: Set  
    - Role: Holds user input parameters such as PiAPI X-API-Key and the 3-view image URL.  
    - Configuration: Raw JSON with keys `x-api-key` (empty string by default) and `image_url` (empty string by default).  
    - Key Expressions: None; user fills in values before running.  
    - Inputs: From manual trigger.  
    - Outputs: Connects to GPT-4o Generator: Front View.  
    - Edge Cases: Missing or invalid API key or image URL will cause downstream API failures.

---

#### 2.2 Front View Image Generation

- **Overview:** Generates the front view image from the input 3-view image using GPT-4o-Image API, then extracts the image URL from the streamed response.
- **Nodes Involved:**  
  - GPT-4o Generator: Front View  
  - Get Image URL of Front Image  
  - Verify Generation Status of Front View

- **Node Details:**

  - **GPT-4o Generator: Front View**  
    - Type: HTTP Request  
    - Role: Calls PiAPI GPT-4o-Image chat completions endpoint to generate front view image.  
    - Configuration: POST request to `https://api.piapi.ai/v1/chat/completions` with JSON body specifying model `gpt-4o-image-preview` and a message containing the input image URL and prompt "Capture front view of the image, then split them into two separate images for me."  
    - Authentication: HTTP Header Auth with Bearer token from `x-api-key` in Basic Params.  
    - Inputs: From Basic Params node.  
    - Outputs: Connects to Get Image URL of Front Image.  
    - Edge Cases: API key invalid or expired; streaming response parsing errors; network timeouts.

  - **Get Image URL of Front Image**  
    - Type: Code  
    - Role: Parses the streamed response from GPT-4o Generator to extract the first image URL embedded in Markdown image syntax.  
    - Configuration: Custom JavaScript code that splits the response by double newlines, iterates in reverse, parses JSON chunks, and extracts the URL from Markdown image syntax `![...](URL)`.  
    - Inputs: From GPT-4o Generator: Front View.  
    - Outputs: Connects to Verify Generation Status of Front View.  
    - Edge Cases: Malformed or incomplete response; no image URL found; JSON parse errors.

  - **Verify Generation Status of Front View**  
    - Type: If  
    - Role: Checks if the image URL extraction succeeded (`finish_reason` equals "not_found" means failure).  
    - Configuration: Condition checks if `finish_reason` is "not_found".  
    - Inputs: From Get Image URL of Front Image.  
    - Outputs:  
      - If true (not found): loops back to GPT-4o Generator: Front View to retry generation.  
      - If false (success): proceeds to GPT-4o Generator: Side View.  
    - Edge Cases: Infinite retry if image URL never found; no max retry limit configured.

---

#### 2.3 Side View Image Generation

- **Overview:** Similar to front view generation, this block generates the side view image and extracts its URL.
- **Nodes Involved:**  
  - GPT-4o Generator: Side View  
  - Get Image URL of Side Image  
  - Verify Generation Status of Side View

- **Node Details:**

  - **GPT-4o Generator: Side View**  
    - Type: HTTP Request  
    - Role: Calls PiAPI GPT-4o-Image API to generate side view image with prompt "Generate side view of the image".  
    - Configuration: POST request to same endpoint as front view with appropriate prompt and input image URL from Basic Params.  
    - Authentication: Bearer token from Basic Params.  
    - Inputs: From Verify Generation Status of Front View (on success).  
    - Outputs: Connects to Get Image URL of Side Image.  
    - Edge Cases: Same as front view generation.

  - **Get Image URL of Side Image**  
    - Type: Code  
    - Role: Parses streamed response to extract side view image URL similarly to front image extraction.  
    - Configuration: JavaScript code similar to front image extraction node.  
    - Inputs: From GPT-4o Generator: Side View.  
    - Outputs: Connects to Verify Generation Status of Side View.  
    - Edge Cases: Same as front image extraction.

  - **Verify Generation Status of Side View**  
    - Type: If  
    - Role: Checks if side image URL extraction succeeded.  
    - Configuration: Condition on `finish_reason` equals "not_found".  
    - Inputs: From Get Image URL of Side Image.  
    - Outputs:  
      - If true (not found): loops back to GPT-4o Generator: Side View to retry.  
      - If false (success): proceeds to Generate Kling Video.  
    - Edge Cases: Potential infinite retry loop.

---

#### 2.4 Video Generation Request

- **Overview:** Sends a video generation task to the Kling API using the front and side view image URLs.
- **Nodes Involved:**  
  - Generate Kling Video  
  - Get Kling Video

- **Node Details:**

  - **Generate Kling Video**  
    - Type: HTTP Request  
    - Role: Initiates a video generation task on PiAPI’s Kling API with parameters including front and side image URLs, rotation mode, duration, and prompt.  
    - Configuration: POST request to `https://api.piapi.ai/api/v1/task` with JSON body specifying model `kling`, task_type `video_generation`, version `1.6`, mode `pro`, front image URL from "Get Image URL of Front Image" node, side image URL from current JSON, duration 5 seconds, and prompt describing anticlockwise rotation with original facial expression.  
    - Headers: `x-api-key` from Basic Params.  
    - Inputs: From Verify Generation Status of Side View (on success).  
    - Outputs: Connects to Get Kling Video.  
    - Edge Cases: API key invalid; invalid or missing image URLs; API rate limits; network errors.

  - **Get Kling Video**  
    - Type: HTTP Request  
    - Role: Queries the status of the video generation task using the task ID returned from Generate Kling Video.  
    - Configuration: GET request to `https://api.piapi.ai/api/v1/task/{{task_id}}` with `x-api-key` header.  
    - Inputs: From Generate Kling Video and Wait for Video Generation nodes.  
    - Outputs: Connects to Verify Task Status.  
    - Edge Cases: Task ID invalid; API errors; network timeouts.

---

#### 2.5 Video Generation Polling and Retrieval

- **Overview:** Polls the Kling API to check if the video generation task is completed, waits if not, and retrieves the final video URLs.
- **Nodes Involved:**  
  - Verify Task Status  
  - Wait for Video Generation  
  - Get Final Video

- **Node Details:**

  - **Verify Task Status**  
    - Type: If  
    - Role: Checks if the video generation task status is "completed".  
    - Configuration: Condition comparing `$json.data.status` to "completed".  
    - Inputs: From Get Kling Video.  
    - Outputs:  
      - If true: proceeds to Get Final Video.  
      - If false: proceeds to Wait for Video Generation.  
    - Edge Cases: Task stuck in non-completed state; no timeout or max retry limit.

  - **Wait for Video Generation**  
    - Type: Wait  
    - Role: Pauses workflow for 20 seconds before re-querying task status.  
    - Configuration: Wait duration set to 20 seconds.  
    - Inputs: From Verify Task Status (if not completed).  
    - Outputs: Loops back to Get Kling Video.  
    - Edge Cases: Long wait times if video generation is slow; no max retry count.

  - **Get Final Video**  
    - Type: Code  
    - Role: Extracts the final video URL and watermark-free video URL from the completed task response.  
    - Configuration: JavaScript code accessing nested JSON fields for `video_url` and `watermark_free_url`.  
    - Inputs: From Verify Task Status (if completed).  
    - Outputs: Final output containing video URLs for downstream use.  
    - Edge Cases: Missing expected fields in response; malformed JSON.

---

### 3. Summary Table

| Node Name                     | Node Type          | Functional Role                          | Input Node(s)                      | Output Node(s)                     | Sticky Note                                                                                         |
|-------------------------------|--------------------|----------------------------------------|----------------------------------|----------------------------------|---------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’  | Manual Trigger     | Workflow entry point                    | None                             | Basic Params                     |                                                                                                   |
| Basic Params                  | Set                | Holds API key and input image URL      | When clicking ‘Test workflow’    | GPT-4o Generator: Front View     | Fill in your PiAPI X-API-Key and 3-View image URL before running.                                 |
| GPT-4o Generator: Front View  | HTTP Request       | Generate front view image via GPT-4o   | Basic Params                    | Get Image URL of Front Image      |                                                                                                   |
| Get Image URL of Front Image  | Code               | Extract front image URL from response  | GPT-4o Generator: Front View    | Verify Generation Status of Front View |                                                                                                   |
| Verify Generation Status of Front View | If          | Check if front image generation succeeded | Get Image URL of Front Image   | GPT-4o Generator: Front View (retry), GPT-4o Generator: Side View (on success) | Potential infinite retry if image URL not found.                                                  |
| GPT-4o Generator: Side View   | HTTP Request       | Generate side view image via GPT-4o    | Verify Generation Status of Front View | Get Image URL of Side Image     |                                                                                                   |
| Get Image URL of Side Image   | Code               | Extract side image URL from response   | GPT-4o Generator: Side View     | Verify Generation Status of Side View |                                                                                                   |
| Verify Generation Status of Side View | If           | Check if side image generation succeeded | Get Image URL of Side Image    | GPT-4o Generator: Side View (retry), Generate Kling Video (on success) | Potential infinite retry if image URL not found.                                                  |
| Generate Kling Video          | HTTP Request       | Request video generation task           | Verify Generation Status of Side View | Get Kling Video                |                                                                                                   |
| Get Kling Video               | HTTP Request       | Poll video generation task status       | Generate Kling Video, Wait for Video Generation | Verify Task Status           |                                                                                                   |
| Verify Task Status            | If                 | Check if video generation completed     | Get Kling Video                 | Get Final Video (if completed), Wait for Video Generation (if not) | No max retry limit; task may hang if never completed.                                            |
| Wait for Video Generation     | Wait               | Wait before polling task status again   | Verify Task Status (not completed) | Get Kling Video                | Waits 20 seconds between polls.                                                                   |
| Get Final Video               | Code               | Extract final video URLs                 | Verify Task Status (completed)  | None                            |                                                                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: When clicking ‘Test workflow’  
   - Purpose: Entry point for manual execution.

2. **Create Set Node for Basic Params**  
   - Name: Basic Params  
   - Parameters:  
     - `x-api-key`: empty string (to be filled by user)  
     - `image_url`: empty string (to be filled by user)  
   - Connect output of Manual Trigger to this node.

3. **Create HTTP Request Node for Front View Generation**  
   - Name: GPT-4o Generator: Front View  
   - Method: POST  
   - URL: `https://api.piapi.ai/v1/chat/completions`  
   - Authentication: HTTP Header Auth with Bearer token from Basic Params `x-api-key`  
   - Body (JSON):  
     ```json
     {
       "model": "gpt-4o-image-preview",
       "messages": [
         {
           "role": "user",
           "content": [
             {
               "type": "image_url",
               "image_url": { "url": "{{ $json.image_url }}" }
             },
             {
               "type": "text",
               "text": "Capture front view of the image, then split them into two separate images for me."
             }
           ]
         }
       ],
       "stream": true
     }
     ```
   - Connect output of Basic Params to this node.

4. **Create Code Node to Extract Front Image URL**  
   - Name: Get Image URL of Front Image  
   - JavaScript code: Parses streamed response to extract Markdown image URL (see node details above).  
   - Connect output of GPT-4o Generator: Front View to this node.

5. **Create If Node to Verify Front View Generation**  
   - Name: Verify Generation Status of Front View  
   - Condition: Check if `finish_reason` equals `"not_found"`  
   - True output: Connect back to GPT-4o Generator: Front View (retry)  
   - False output: Connect to GPT-4o Generator: Side View

6. **Create HTTP Request Node for Side View Generation**  
   - Name: GPT-4o Generator: Side View  
   - Method: POST  
   - URL: `https://api.piapi.ai/v1/chat/completions`  
   - Authentication: Bearer token from Basic Params `x-api-key`  
   - Body (JSON):  
     ```json
     {
       "model": "gpt-4o-image-preview",
       "messages": [
         {
           "role": "user",
           "content": [
             {
               "type": "image_url",
               "image_url": { "url": "{{ $('Basic Params').item.json.image_url }}" }
             },
             {
               "type": "text",
               "text": "Generate side view of the image"
             }
           ]
         }
       ],
       "stream": true
     }
     ```
   - Connect false output of Verify Generation Status of Front View to this node.

7. **Create Code Node to Extract Side Image URL**  
   - Name: Get Image URL of Side Image  
   - JavaScript code: Similar to front image extraction, parse streamed response for Markdown image URL.  
   - Connect output of GPT-4o Generator: Side View to this node.

8. **Create If Node to Verify Side View Generation**  
   - Name: Verify Generation Status of Side View  
   - Condition: Check if `finish_reason` equals `"not_found"`  
   - True output: Connect back to GPT-4o Generator: Side View (retry)  
   - False output: Connect to Generate Kling Video

9. **Create HTTP Request Node to Generate Kling Video**  
   - Name: Generate Kling Video  
   - Method: POST  
   - URL: `https://api.piapi.ai/api/v1/task`  
   - Headers: `x-api-key` from Basic Params  
   - Body (JSON):  
     ```json
     {
       "model": "kling",
       "task_type": "video_generation",
       "input": {
         "version": "1.6",
         "mode": "pro",
         "image_url": "{{ $('Get Image URL of Front Image').item.json.image_url }}",
         "image_tail_url": "{{ $json.image_url }}",
         "duration": 5,
         "prompt": "The character rotates smoothly, stay original facial expression. Apply anticlockwise rotation"
       }
     }
     ```
   - Connect false output of Verify Generation Status of Side View to this node.

10. **Create HTTP Request Node to Get Kling Video Status**  
    - Name: Get Kling Video  
    - Method: GET  
    - URL: `https://api.piapi.ai/api/v1/task/{{ $json.data.task_id }}`  
    - Headers: `x-api-key` from Basic Params  
    - Connect output of Generate Kling Video and Wait for Video Generation node to this node.

11. **Create If Node to Verify Task Status**  
    - Name: Verify Task Status  
    - Condition: Check if `$json.data.status` equals `"completed"`  
    - True output: Connect to Get Final Video  
    - False output: Connect to Wait for Video Generation

12. **Create Wait Node**  
    - Name: Wait for Video Generation  
    - Wait Time: 20 seconds  
    - Connect false output of Verify Task Status to this node.  
    - Connect output of this node back to Get Kling Video node to poll again.

13. **Create Code Node to Extract Final Video URLs**  
    - Name: Get Final Video  
    - JavaScript code: Extracts `video_url` and `watermark_free_url` from response JSON.  
    - Connect true output of Verify Task Status to this node.

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                                                      |
|-----------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| This workflow uses unofficial PiAPI endpoints for GPT-4o-Image and Kling video generation APIs.           | https://piapi.ai                                                                                   |
| The workflow can be paired with the "3D Figurine Orthographic Views" workflow for input image generation. | https://creators.n8n.io/workflows/3628                                                            |
| Example input image and output video demonstration available.                                             | Input: ![image](https://i.ibb.co/3ydMybrn/doll.png) Output: ![video](https://static.piapi.ai/n8n-instruction/3D-views2video/example1.mp4) |
| Users must provide their own PiAPI X-API-Key for authentication.                                          |                                                                                                    |
| Potential infinite retry loops exist if image generation fails; consider adding retry limits or error handling. |                                                                                                    |

---

This comprehensive documentation enables understanding, reproduction, and modification of the workflow, anticipating potential failure points and integration requirements.