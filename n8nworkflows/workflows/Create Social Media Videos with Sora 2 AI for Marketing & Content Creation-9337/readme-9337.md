Create Social Media Videos with Sora 2 AI for Marketing & Content Creation

https://n8nworkflows.xyz/workflows/create-social-media-videos-with-sora-2-ai-for-marketing---content-creation-9337


# Create Social Media Videos with Sora 2 AI for Marketing & Content Creation

### 1. Workflow Overview

This workflow automates the generation of viral social media videos using the Defapi API's **Sora 2 AI** video generation model. It targets content creators, marketers, and video producers seeking to rapidly produce engaging, professional-quality videos from textual prompts and optional reference images. The workflow handles user input collection, image processing, API request submission, status polling, and final video retrieval and formatting.

**Logical Blocks:**

- **1.1 Input Reception:** Captures user inputs via a form, including a required text prompt and an optional image file.
- **1.2 Input Preparation:** Converts uploaded images to base64 data URI format and packages the prompt for API consumption.
- **1.3 Video Generation Request:** Sends the prepared data to the Sora 2 AI video generation endpoint with authentication.
- **1.4 Processing Wait & Status Polling:** Waits for a fixed interval and polls the API until the video generation is complete.
- **1.5 Completion Check & Result Formatting:** Evaluates whether generation succeeded, then formats and outputs the final video URL.
- **1.6 Content Moderation Awareness:** Embedded notes and best practices inform users about content restrictions and filtering stages.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Collects user inputs through a web form trigger node. Users provide a text prompt describing the desired video scene and may optionally upload an image for contextual reference.

- **Nodes Involved:**  
  - On form submission

- **Node Details:**  

  - **On form submission**  
    - *Type & Role:* Form Trigger node, entry point for user inputs  
    - *Configuration:*  
      - Form title: "Upload Image"  
      - Fields:  
        - "Prompt" (text, required)  
        - "Image" (file upload, optional, accepts .jpg, .png, .webp)  
    - *Expressions/Variables:* Captures form fields as JSON data; image file is treated as binary data under field name 'Image'  
    - *Input:* External HTTP form submission  
    - *Output:* JSON including prompt text and optionally binary image data  
    - *Edge Cases:* User submits no image (acceptable), submits unsupported file type (rejected by form), or omits required prompt (blocked by form)  
    - *Version:* Form Trigger v2.3  

---

#### 2.2 Input Preparation

- **Overview:**  
  Converts the optionally uploaded image into a base64-encoded data URI and pairs it with the prompt text for API submission.

- **Nodes Involved:**  
  - Convert to JSON

- **Node Details:**  

  - **Convert to JSON**  
    - *Type & Role:* Code node (JavaScript) to transform inputs into API-ready format  
    - *Configuration:*  
      - Extracts binary image data under `"Image"` key, encodes it as `data:[mimeType];base64,[data]` string within an array under `args.images`  
      - Copies prompt text into `args.prompt`  
      - Returns an object with `args` property containing `prompt` and optional `images` array  
    - *Expressions/Variables:* Uses `$input.first()` to access first item, binary data accessed via `$input.first().binary['Image']`  
    - *Input:* JSON and binary data from form submission node  
    - *Output:* JSON object `{ args: { prompt: string, images?: [string] } }`  
    - *Edge Cases:* No image uploaded results in no `args.images` property; ensures proper base64 formatting; possible failure if binary data corrupted or missing  
    - *Version:* Code node v2  

---

#### 2.3 Video Generation Request

- **Overview:**  
  Sends a POST request to Defapi’s Sora 2 API endpoint to initiate video generation, authenticating using Bearer token.

- **Nodes Involved:**  
  - Send Sora 2 Generation Request to Defapi.org API

- **Node Details:**  

  - **Send Sora 2 Generation Request to Defapi.org API**  
    - *Type & Role:* HTTP Request node, performs the API call to start video generation  
    - *Configuration:*  
      - HTTP Method: POST  
      - URL: `https://api.defapi.org/api/sora2/gen`  
      - Request Body: JSON, stringified from `args` property of previous node's output  
      - Authentication: HTTP Bearer Auth using credentials named "Defapi account" containing API key  
      - Headers: Content-Type application/json (implied with JSON body)  
    - *Expressions/Variables:* Body is `={{ JSON.stringify($json.args, null, 2) }}`  
    - *Input:* JSON with prompt and optional images array from previous code node  
    - *Output:* JSON containing `task_id` for status polling  
    - *Edge Cases:* Authentication failures (invalid/missing API key), HTTP errors (timeout, rate limit), malformed request if prompt or images missing or invalid  
    - *Version:* HTTP Request v4.2  

---

#### 2.4 Processing Wait & Status Polling

- **Overview:**  
  Implements a wait period followed by polling the API to check the generation status until completion.

- **Nodes Involved:**  
  - Wait for Processing Completion  
  - Obtain the generated status

- **Node Details:**  

  - **Wait for Processing Completion**  
    - *Type & Role:* Wait node, pauses workflow for a fixed interval before next action  
    - *Configuration:* Wait for 10 seconds  
    - *Input:* Triggered after sending generation request or after failed completion check  
    - *Output:* Continues to status polling node  
    - *Edge Cases:* Excessive wait times can delay user feedback; insufficient wait may cause premature polling  

  - **Obtain the generated status**  
    - *Type & Role:* HTTP Request node, polls the API task query endpoint for current status  
    - *Configuration:*  
      - HTTP Method: GET  
      - URL: `https://api.defapi.org/api/task/query`  
      - Query Parameter: `task_id` extracted dynamically from previous API response (`{{$json.data.task_id}}`)  
      - Authentication: HTTP Bearer Auth with "Defapi account" credentials  
      - Headers: Content-Type application/json  
    - *Input:* Task ID from generation request response  
    - *Output:* JSON with status field (e.g., "success", "pending", "failed") and result data  
    - *Edge Cases:* Network errors, invalid task ID, authentication failures, API rate limits, unexpected response formats  
    - *Version:* HTTP Request v4.2  

---

#### 2.5 Completion Check & Result Formatting

- **Overview:**  
  Evaluates if video generation is complete. If successful, extracts and formats the video URL for output; if not, loops back to wait and poll again.

- **Nodes Involved:**  
  - Check if Generation is Complete  
  - Format and Display Results

- **Node Details:**  

  - **Check if Generation is Complete**  
    - *Type & Role:* If node, conditional branching based on generation status  
    - *Configuration:*  
      - Condition: Checks if `$json.data.status == 'success'`  
      - True branch: proceed to formatting results  
      - False branch: loop back to wait node for further polling  
    - *Input:* API response from status polling  
    - *Output:* Branching workflow flow  
    - *Edge Cases:* Status other than "success" or prolonged pending/failed states; malformed status field  

  - **Format and Display Results**  
    - *Type & Role:* Set node, formats output data for downstream use or display  
    - *Configuration:*  
      - Assigns `image_url` property with value extracted from `$json.data.result.video` (final video URL)  
    - *Input:* API response confirming success  
    - *Output:* JSON containing the final video URL for download or embedding  
    - *Edge Cases:* Missing or malformed `video` URL in API response; handle gracefully to avoid broken links  
    - *Version:* Set node v3.4  

---

#### 2.6 Content Moderation Awareness

- **Overview:**  
  Provides documentation and best practices within sticky notes about the API’s multi-stage content moderation, including restrictions on images and prompts, and advice on avoiding generation failures.

- **Nodes Involved:**  
  - Sticky Note3 (Workflow Overview & Instructions)  
  - Sticky Note (Example Prompt)  
  - Sticky Note1 (Content Moderation Details)  
  - Sticky Note4 (Result Visual Example)

- **Node Details:**  

  - **Sticky Note3**  
    - Large detailed note describing the workflow purpose, user instructions, API constraints, prompt examples, moderation rules, and use cases.  
    - Acts as an embedded manual for users and maintainers.  

  - **Sticky Note**  
    - Contains an example prompt illustrating a creative video description.  

  - **Sticky Note1**  
    - Details the three-stage content moderation process and best practices to avoid failures.  

  - **Sticky Note4**  
    - Visual example (animated gif) illustrating the creative results achievable with the workflow.  

---

### 3. Summary Table

| Node Name                                   | Node Type           | Functional Role                          | Input Node(s)               | Output Node(s)                      | Sticky Note                                                                                                                |
|---------------------------------------------|---------------------|----------------------------------------|-----------------------------|------------------------------------|----------------------------------------------------------------------------------------------------------------------------|
| On form submission                          | Form Trigger        | Collect user prompt and optional image | External HTTP form submit    | Convert to JSON                    |                                                                                                                            |
| Convert to JSON                            | Code                | Convert uploaded image to base64 URI and format prompt | On form submission           | Send Sora 2 Generation Request to Defapi.org API |                                                                                                                            |
| Send Sora 2 Generation Request to Defapi.org API | HTTP Request        | Submit video generation request with auth | Convert to JSON              | Wait for Processing Completion     |                                                                                                                            |
| Wait for Processing Completion             | Wait                | Pause before polling generation status | Send Sora 2 Generation Request to Defapi.org API / Check if Generation is Complete (false branch) | Obtain the generated status        |                                                                                                                            |
| Obtain the generated status                 | HTTP Request        | Poll API for generation status         | Wait for Processing Completion | Check if Generation is Complete    |                                                                                                                            |
| Check if Generation is Complete             | If                  | Branch based on whether generation succeeded | Obtain the generated status   | Format and Display Results / Wait for Processing Completion |                                                                                                                            |
| Format and Display Results                   | Set                 | Extract and output final video URL     | Check if Generation is Complete (true branch) | None                              |                                                                                                                            |
| Sticky Note3                               | Sticky Note         | Workflow overview, instructions, prerequisites | None                        | None                              | Contains detailed workflow overview, setup instructions, usage tips, and API notes with link to https://defapi.org         |
| Sticky Note                                | Sticky Note         | Example prompt                         | None                        | None                              | Example prompt: "A pack of dogs driving tiny cars in a high-speed chase..."                                                |
| Sticky Note1                               | Sticky Note         | Content moderation details and best practices | None                        | None                              | Describes API content moderation steps and advice to avoid failures                                                        |
| Sticky Note4                               | Sticky Note         | Visual example of creative result      | None                        | None                              | Shows an animated GIF of a generated creative video                                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node**  
   - Add node: *Form Trigger*  
   - Configure form: Title "Upload Image"  
   - Fields:  
     - "Prompt" (Text, required)  
     - "Image" (File upload, optional, accept `.jpg, .png, .webp`)  
   - Save and note the webhook URL for user access  

2. **Add Code Node: Convert to JSON**  
   - Add node: *Code* (JavaScript)  
   - Paste code to:  
     - Check if binary file `Image` exists  
     - If yes, encode as base64 data URI: `data:[mimeType];base64,[data]` inside array `args.images`  
     - Copy prompt string to `args.prompt`  
     - Return `{ args: { prompt: string, images?: [string] } }`  
   - Connect from Form Trigger output  

3. **Add HTTP Request Node: Send Sora 2 Generation Request**  
   - Add node: *HTTP Request*  
   - Method: POST  
   - URL: `https://api.defapi.org/api/sora2/gen`  
   - Body Content Type: JSON  
   - Body: Use expression to stringify input JSON from previous node (`={{ JSON.stringify($json.args, null, 2) }}`)  
   - Authentication: HTTP Bearer Auth with credentials named "Defapi account" (create credentials using your Defapi API key)  
   - Connect from Code node output  

4. **Add Wait Node: Wait for Processing Completion**  
   - Add node: *Wait*  
   - Set to wait 10 seconds  
   - Connect from HTTP Request node output  

5. **Add HTTP Request Node: Obtain the generated status**  
   - Add node: *HTTP Request*  
   - Method: GET  
   - URL: `https://api.defapi.org/api/task/query`  
   - Query Parameter: `task_id` with value `={{$json.data.task_id}}` from previous node output  
   - Authentication: HTTP Bearer Auth with "Defapi account" credentials  
   - Connect from Wait node output  

6. **Add If Node: Check if Generation is Complete**  
   - Add node: *If*  
   - Condition: Check if `{{$json.data.status}}` equals `'success'` (case sensitive)  
   - Connect from status HTTP Request node output  

7. **Add Set Node: Format and Display Results**  
   - Add node: *Set*  
   - Assign new field `image_url` with value `={{$json.data.result.video}}`  
   - Connect from If node *true* output branch  

8. **Connect If node *false* branch back to Wait node**  
   - This creates a polling loop waiting for success status  

9. **Create HTTP Bearer Auth Credentials**  
   - In Credentials section, create new HTTP Bearer Auth credentials named "Defapi account"  
   - Paste your Defapi API key as the token  

10. **Test the Workflow**  
    - Activate workflow  
    - Access webhook URL from form trigger node  
    - Submit prompt and optionally an image  
    - Confirm video URL is returned after generation completes  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                     | Context or Link                                     |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------|
| Sign up for a Defapi account and obtain an API key with Sora 2 access at https://defapi.org.                                                                                                     | Defapi registration and API keys                    |
| Use prompt prefix `(15s,hd)` to generate 15-second high-definition videos.                                                                                                                       | Prompt syntax for video length and quality          |
| Avoid uploading images with real people or realistic human faces due to content moderation rejection.                                                                                            | Content moderation best practice                     |
| The workflow uses a multi-stage moderation process: image review, prompt filtering, and output review; failures often happen near 90% completion.                                                | Content moderation details                           |
| Example prompt: "A pack of dogs driving tiny cars in a high-speed chase through a city, wearing sunglasses and honking their horns, with dramatic action music and slow-motion jumps over hydrants." | Creative prompt example                              |
| For more detailed workflow overview, setup instructions, and prompt tips, see the embedded sticky note content within the workflow.                                                             | Embedded workflow documentation in Sticky Note3     |
| The workflow is designed for viral social media video content, suitable for TikTok, Instagram Reels, YouTube Shorts, marketing campaigns, and creative projects.                                | Use case context                                    |

---

*Disclaimer:* The provided text stems exclusively from an automated workflow created with n8n, adhering strictly to content policies. It contains no illegal, offensive, or protected elements. All data processed is legal and public.