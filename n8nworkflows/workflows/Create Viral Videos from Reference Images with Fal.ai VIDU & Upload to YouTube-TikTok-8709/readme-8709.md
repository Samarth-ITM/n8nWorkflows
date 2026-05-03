Create Viral Videos from Reference Images with Fal.ai VIDU & Upload to YouTube/TikTok

https://n8nworkflows.xyz/workflows/create-viral-videos-from-reference-images-with-fal-ai-vidu---upload-to-youtube-tiktok-8709


# Create Viral Videos from Reference Images with Fal.ai VIDU & Upload to YouTube/TikTok

### 1. Workflow Overview

This workflow automates the creation of short AI-generated videos from multiple user-provided reference images and a text prompt, then uploads the resulting videos to YouTube and TikTok. The main use case is content creators or marketers who want to quickly generate viral video clips based on visual references and natural language prompts, leveraging the Fal.ai VIDU video generation API.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Receives user input via a web form containing a text prompt and multiple image URLs.
- **1.2 Data Preparation:** Processes and formats the input data for the video generation API.
- **1.3 Video Generation Request:** Submits the video creation request to Fal.ai’s VIDU API.
- **1.4 Video Generation Polling:** Periodically checks the status of the video generation job until it completes.
- **1.5 Video Retrieval and Title Generation:** Once completed, retrieves the video file URL and generates an SEO-optimized video title using OpenAI.
- **1.6 Video Download and Upload:** Downloads the generated video, uploads it to Google Drive, then posts it to YouTube and TikTok via the Upload-Post API.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Captures user input from a web form to start the workflow.
- **Nodes Involved:** `On form submission`
- **Node Details:**
  - **Type:** Form Trigger
  - **Role:** Listens for form submissions with fields "Prompt" (text) and "Reference images" (textarea)
  - **Configuration:** 
    - Form title: "Reference to video"
    - Fields: "Prompt" (required), "Reference images" (required)
    - Description guides users to input multiple image URLs separated by commas or newlines
  - **Input:** HTTP webhook form submission
  - **Output:** JSON containing raw prompt and reference image URLs string
  - **Edge cases:** Malformed input (empty fields, invalid URLs) may cause downstream failures if not validated externally.

#### 2.2 Data Preparation

- **Overview:** Converts the raw reference images string into an array of URLs and bundles it with the prompt.
- **Nodes Involved:** `Array`, `Set data`
- **Node Details:**

  - **Array**
    - **Type:** Code
    - **Role:** Parses the "Reference images" string to an array by splitting on commas and newlines, filtering out empty strings.
    - **Key Expression:** Splits string with regex `/,\s*|\r\n+/`
    - **Input:** JSON from form submission
    - **Output:** JSON with `reference_image_urls` array
    - **Edge cases:** Excess whitespace, empty entries handled; invalid URLs not validated here.

  - **Set data**
    - **Type:** Set
    - **Role:** Assigns the prompt and reference image URLs array to new JSON properties for API call
    - **Key Expressions:** 
      - prompt = value from form submission field "Prompt"
      - reference_images = array from previous node
    - **Input:** Output of `Array`
    - **Output:** JSON with `prompt` and `reference_images`

#### 2.3 Video Generation Request

- **Overview:** Sends a request to Fal.ai VIDU API to generate a video based on the prompt and reference images.
- **Nodes Involved:** `Create Video`
- **Node Details:**
  - **Type:** HTTP Request
  - **Role:** POST to `https://queue.fal.run/fal-ai/vidu/q1/reference-to-video`
  - **Authentication:** HTTP header with API Key (Fal.run API credential)
  - **Payload:** JSON body containing:
    - prompt (user input)
    - reference_image_urls (array of URLs)
    - aspect_ratio: "16:9"
    - movement_amplitude: "auto"
  - **Output:** JSON containing a `request_id` for polling
  - **Edge cases:** API auth failure, invalid prompt/image URLs, malformed JSON

#### 2.4 Video Generation Polling

- **Overview:** Waits and checks repeatedly if the video generation job is completed.
- **Nodes Involved:** `Wait 60 sec.`, `Get status`, `Completed?`
- **Node Details:**

  - **Wait 60 sec.**
    - **Type:** Wait
    - **Role:** Delays workflow for 60 seconds before polling
    - **Input:** Output of `Create Video` or if not completed, loops
    - **Output:** Triggers `Get status`

  - **Get status**
    - **Type:** HTTP Request
    - **Role:** GET request to `https://queue.fal.run/fal-ai/vidu/requests/{{request_id}}/status`
    - **Authentication:** Fal.run API header
    - **Output:** JSON with status property (e.g., "COMPLETED", "PENDING")

  - **Completed?**
    - **Type:** If
    - **Role:** Checks if `status == "COMPLETED"`
    - **Output:** If true, proceeds; else loops back to wait

  - **Edge cases:** API downtime, long processing times, status other than expected, network timeouts

#### 2.5 Video Retrieval and Title Generation

- **Overview:** Retrieves the final video URL, generates a catchy SEO-friendly title using OpenAI.
- **Nodes Involved:** `Get Url Video`, `Generate title`, `Get File Video`
- **Node Details:**

  - **Get Url Video**
    - **Type:** HTTP Request
    - **Role:** GET `https://queue.fal.run/fal-ai/vidu/requests/{{request_id}}` to retrieve the video metadata including URL
    - **Authentication:** Fal.run API header
    - **Output:** JSON containing video URL and file name

  - **Generate title**
    - **Type:** OpenAI (Langchain)
    - **Role:** Uses GPT-5-MINI to create a YouTube SEO optimized title from the original user prompt
    - **Prompt:** System message instructs to generate catchy titles with max 60 characters, matching input language, no clickbait, includes keywords
    - **Input:** User prompt from `Set data` node
    - **Output:** JSON with generated title in `message.content`
    - **Version requirements:** OpenAI API access, model GPT-5-MINI configured
    - **Edge cases:** API quota limits, model unavailability, response formatting issues

  - **Get File Video**
    - **Type:** HTTP Request
    - **Role:** Downloads the video file from URL obtained in `Get Url Video`
    - **Input:** URL from previous node
    - **Output:** Binary video file data for upload

#### 2.6 Video Download and Upload

- **Overview:** Uploads the downloaded video to Google Drive, then posts it to YouTube and TikTok using Upload-Post API.
- **Nodes Involved:** `Upload Video`, `Upload to Youtube`, `Upload on TikTok`
- **Node Details:**

  - **Upload Video**
    - **Type:** Google Drive
    - **Role:** Uploads the video binary to a specific Google Drive folder
    - **Parameters:** 
      - File name: timestamp + original filename
      - Drive: "My Drive"
      - Folder ID: specified folder for Fal.run outputs
    - **Credential:** Google Drive OAuth2
    - **Edge cases:** Upload failure, auth expires

  - **Upload to Youtube**
    - **Type:** HTTP Request
    - **Role:** Uploads video file to YouTube via Upload-Post API
    - **Method:** POST multipart/form-data
    - **Body:** 
      - title: generated title from OpenAI node
      - user: placeholder "YOUR_USERNAME"
      - platform[]: "youtube"
      - video: binary from `Get File Video`
    - **Authentication:** API key in HTTP header
    - **Edge cases:** API rate limits, invalid credentials, video size limits

  - **Upload on TikTok**
    - **Type:** HTTP Request
    - **Role:** Same as YouTube upload but platform[] set to "tiktok"
    - **Note:** Free plan may not support TikTok uploads without paid upgrade
    - **Edge cases:** Same as above plus platform restrictions

---

### 3. Summary Table

| Node Name       | Node Type                | Functional Role                         | Input Node(s)            | Output Node(s)                           | Sticky Note                                                  |
|-----------------|--------------------------|---------------------------------------|--------------------------|-----------------------------------------|--------------------------------------------------------------|
| On form submission | Form Trigger             | Entry point: receives prompt & images | —                        | Array                                   | Guides user on form fields and input format                  |
| Array           | Code                     | Parses reference images string to array | On form submission       | Set data                               |                                                              |
| Set data        | Set                      | Prepares prompt and image URLs for API | Array                    | Create Video                           |                                                              |
| Create Video    | HTTP Request             | Requests video generation from Fal.ai  | Set data                 | Wait 60 sec.                          | API key must be set in HTTP Header                           |
| Wait 60 sec.    | Wait                     | Waits 60 seconds before checking status| Create Video / Completed? | Get status                            |                                                              |
| Get status      | HTTP Request             | Polls video generation status           | Wait 60 sec.             | Completed?                            | API key required                                              |
| Completed?      | If                       | Checks if video generation is finished | Get status               | Get Url Video (if yes), Wait 60 sec. (if no) |                                                              |
| Get Url Video   | HTTP Request             | Retrieves video metadata and URL        | Completed?               | Generate title                        | API key required                                              |
| Generate title  | OpenAI (Langchain)       | Generates SEO optimized video title    | Get Url Video            | Get File Video                       | Requires OpenAI API key, uses GPT-5-MINI                     |
| Get File Video  | HTTP Request             | Downloads video file                    | Generate title           | Upload Video, Upload to Youtube, Upload on TikTok |                                                              |
| Upload Video    | Google Drive             | Uploads video file to Google Drive      | Get File Video           | —                                   | OAuth2 credential required                                   |
| Upload to Youtube | HTTP Request            | Uploads video to YouTube via Upload-Post API | Get File Video           | —                                   | Requires Upload-Post API key, set YOUR_USERNAME              |
| Upload on TikTok | HTTP Request            | Uploads video to TikTok via Upload-Post API | Get File Video           | —                                   | Requires paid plan for TikTok uploads; set YOUR_USERNAME     |
| Sticky Note3    | Sticky Note              | Workflow description                   | —                        | —                                   | Workflow purpose, usage overview                             |
| Sticky Note5    | Sticky Note              | Instruction for scheduling trigger    | —                        | —                                   | Recommends 5-min intervals for scheduled runs                |
| Sticky Note6    | Sticky Note              | Instructions to obtain Fal.ai API key | —                        | —                                   | Link to Fal.ai and API key setup instructions                |
| Sticky Note7    | Sticky Note              | Reminds to set API key                 | —                        | —                                   |                                                              |
| Sticky Note8    | Sticky Note              | Instructions for Upload-Post API keys | —                        | —                                   | Link to Upload-Post, free plan limitations                   |
| Sticky Note1    | Sticky Note              | Example input and result               | —                        | —                                   | Example prompt, images, and output video URL                 |
| Sticky Note     | Sticky Note              | Reminder to set username for uploads  | —                        | —                                   |                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Form Trigger node**
   - Type: Form Trigger
   - Name: "On form submission"
   - Configure form with:
     - Title: "Reference to video"
     - Fields:
       - "Prompt" (text, required)
       - "Reference images" (textarea, required)
     - Description: instruct users to separate image URLs by commas or new lines
   - Save and note the webhook URL generated.

2. **Create a Code node**
   - Type: Code
   - Name: "Array"
   - JavaScript code:
     ```js
     const inputString = $json["Reference images"];
     const urlsArray = inputString
       .split(/,\s*|\r\n+/)
       .filter(url => url.trim() !== "");
     return [{ json: { reference_image_urls: urlsArray } }];
     ```
   - Connect "On form submission" to "Array"

3. **Create a Set node**
   - Type: Set
   - Name: "Set data"
   - Add assignments:
     - `prompt` = `={{ $('On form submission').item.json.Prompt }}`
     - `reference_images` = `={{ $json.reference_image_urls }}`
   - Connect "Array" to "Set data"

4. **Create HTTP Request node for video generation**
   - Type: HTTP Request
   - Name: "Create Video"
   - Method: POST
   - URL: `https://queue.fal.run/fal-ai/vidu/q1/reference-to-video`
   - Authentication: HTTP Header Auth with Fal.run API key
     - Header: `Authorization: Key YOURAPIKEY`
   - Body:
     ```json
     {
       "prompt": "{{ $json.prompt }}",
       "reference_image_urls": {{ $json.reference_images.toJsonString() }},
       "aspect_ratio": "16:9",
       "movement_amplitude": "auto"
     }
     ```
   - Content type: JSON
   - Connect "Set data" to "Create Video"

5. **Create Wait node**
   - Type: Wait
   - Name: "Wait 60 sec."
   - Duration: 60 seconds
   - Connect "Create Video" to "Wait 60 sec."

6. **Create HTTP Request node for status polling**
   - Type: HTTP Request
   - Name: "Get status"
   - Method: GET
   - URL: `https://queue.fal.run/fal-ai/vidu/requests/{{ $('Create Video').item.json.request_id }}/status`
   - Authentication: Fal.run API key (same as above)
   - Connect "Wait 60 sec." to "Get status"

7. **Create If node**
   - Type: If
   - Name: "Completed?"
   - Condition: Check if `{{$json.status}}` equals `"COMPLETED"`
   - Connect "Get status" to "Completed?"
     - True branch to next step
     - False branch loops back to "Wait 60 sec."

8. **Create HTTP Request node to get video URL**
   - Type: HTTP Request
   - Name: "Get Url Video"
   - Method: GET
   - URL: `https://queue.fal.run/fal-ai/vidu/requests/{{ $json.request_id }}`
   - Authentication: Fal.run API key
   - Connect "Completed?" true output to "Get Url Video"

9. **Create OpenAI node for title generation**
   - Type: OpenAI (Langchain)
   - Name: "Generate title"
   - Model: GPT-5-MINI or equivalent
   - Messages:
     - User input: prompt from `Set data`
     - System message with instructions for SEO-friendly title generation
   - Credentials: OpenAI API key
   - Connect "Get Url Video" to "Generate title"

10. **Create HTTP Request node to download video file**
    - Type: HTTP Request
    - Name: "Get File Video"
    - Method: GET
    - URL: `={{ $('Get Url Video').item.json.video.url }}`
    - No authentication required (public URL)
    - Connect "Generate title" to "Get File Video"

11. **Create Google Drive node to upload video**
    - Type: Google Drive
    - Name: "Upload Video"
    - Configure with OAuth2 Google Drive credentials
    - File name: `={{ $now.format('yyyyLLddHHmmss') }}-{{ $('Get Url Video').item.json.video.file_name }}`
    - Folder ID: target folder ID for uploads
    - Connect "Get File Video" to "Upload Video"

12. **Create HTTP Request node to upload to YouTube**
    - Type: HTTP Request
    - Name: "Upload to Youtube"
    - Method: POST
    - URL: `https://api.upload-post.com/api/upload`
    - Authentication: HTTP Header Auth with Upload-Post API key
    - Body (multipart/form-data):
      - title: `={{ $('Generate title').item.json.message.content }}`
      - user: `YOUR_USERNAME` (replace with your configured user)
      - platform[]: "youtube"
      - video: binary data from "Get File Video"
    - Connect "Get File Video" to "Upload to Youtube"

13. **Create HTTP Request node to upload to TikTok**
    - Type: HTTP Request
    - Name: "Upload on TikTok"
    - Same as YouTube upload but set platform[] to "tiktok"
    - Connect "Get File Video" to "Upload on TikTok"

14. **Ensure all credentials are configured:**
    - Fal.run API key in HTTP Header Auth credentials
    - OpenAI API credentials for title generation
    - Google Drive OAuth2 credentials
    - Upload-Post API keys for YouTube and TikTok uploads

15. **Optional: Add scheduling trigger**
    - Add Schedule Trigger node to start the workflow periodically (e.g., every 5 minutes) instead of only via form submission

---

### 5. General Notes & Resources

| Note Content                                                                                                                        | Context or Link                                                                                               |
|------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| Create a Fal.ai account and obtain your API key. Set HTTP Header Auth with "Authorization: Key YOURAPIKEY" in relevant nodes.     | https://fal.ai/                                                                                               |
| Upload-Post API allows 10 free uploads/month. TikTok upload requires paid plan upgrade.                                            | https://www.upload-post.com/?linkId=lp_144414&sourceId=n3witalia&tenantId=upload-post-app                      |
| Use the example prompt and image URLs from the sticky note to test the workflow.                                                   | Example prompt: "The girl is showing a pair of shoes to the monkey in the forest"                             |
| Recommended to run the workflow on schedule every 5 minutes for batch processing or manual runs via web form trigger.              | Sticky Note5                                                                                                |
| OpenAI GPT-5-MINI model is used for SEO title generation; ensure your API key supports this or select an alternative model.        | Requires OpenAI API access                                                                                   |
| Replace placeholder `YOUR_USERNAME` with your Upload-Post profile username to enable video uploads.                                | Sticky Note and Upload nodes                                                                                  |

---

**Disclaimer:**  
The provided text comes exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.