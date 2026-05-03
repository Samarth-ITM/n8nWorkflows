Generate Funny AI Videos with Sora 2 and Auto-Publish to TikTok

https://n8nworkflows.xyz/workflows/generate-funny-ai-videos-with-sora-2-and-auto-publish-to-tiktok-10212


# Generate Funny AI Videos with Sora 2 and Auto-Publish to TikTok

### 1. Workflow Overview

This workflow automates the generation, processing, and publishing of AI-generated videos with audio using the OpenAI Sora 2 model, saving them to Google Drive, creating optimized video titles with GPT-5, and automatically uploading the final videos to TikTok via Postiz. It is designed to be triggered by a form submission, making it suitable for content creators or social media managers who want to streamline their video production and distribution process.

**Logical Blocks:**

- **1.1 Input Reception:** Captures user input (video prompt and duration) via a form trigger.
- **1.2 AI Video Generation:** Sends the prompt to Sora 2 API to generate a video, then polls for completion.
- **1.3 Video Retrieval:** Once the video is ready, fetches the video URL and downloads the video file.
- **1.4 Title Generation:** Uses GPT-5 to generate an SEO-optimized YouTube-style title based on the prompt.
- **1.5 Video Upload to TikTok:** Uploads the video to Postiz platform and schedules it for publishing on TikTok.
- **1.6 Supporting Notes and Configuration:** Contains sticky notes with instructions for API key setup and workflow usage.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** This block receives input data from a user form, specifically the video prompt and duration, serving as the workflow’s entry point.
- **Nodes Involved:** 
  - On form submission

- **Node Details:**

  - **On form submission**
    - Type: Form Trigger
    - Role: Entry point to receive user inputs "PROMPT" (text) and "DURATION" (number).
    - Configuration: 
      - Form title: "Generate AI Videos (with audio)"
      - Form fields: PROMPT (required), DURATION (required)
      - Webhook ID configured for external trigger
    - Inputs: External HTTP form submission
    - Outputs: JSON containing PROMPT and DURATION
    - Edge Cases: Missing required fields; malformed input; webhook unavailability.

---

#### 2.2 AI Video Generation

- **Overview:** This block sends the video prompt and duration to the Sora 2 API to create the video, then waits and polls the API for completion status.
- **Nodes Involved:** 
  - Create Video
  - Wait 60 sec.
  - Get status
  - Completed?

- **Node Details:**

  - **Create Video**
    - Type: HTTP Request
    - Role: Sends POST request to Sora 2 endpoint to initiate video generation.
    - Configuration:
      - URL: https://queue.fal.run/fal-ai/sora-2/text-to-video
      - Method: POST
      - Body: JSON with prompt and duration combined from form inputs
      - Authentication: Header Auth with API key ("Authorization: Key YOURAPIKEY")
      - Headers: Content-Type application/json
    - Inputs: JSON from form submission
    - Outputs: JSON containing request_id for video generation
    - Edge Cases: API key invalid, request limits, malformed prompt, network errors.

  - **Wait 60 sec.**
    - Type: Wait node
    - Role: Pauses workflow for 60 seconds before polling status.
    - Configuration: Fixed 60-second wait
    - Inputs: Triggered after Create Video
    - Outputs: Triggers Get status node
    - Edge Cases: Time delays may cause workflow timeouts in low-resource environments.

  - **Get status**
    - Type: HTTP Request
    - Role: Polls the Sora 2 API to check the current status of video generation using request_id.
    - Configuration:
      - URL template: https://queue.fal.run/fal-ai/sora-2/requests/{{request_id}}/status
      - Authentication: Header Auth with API key
    - Inputs: request_id from Create Video output
    - Outputs: JSON with status field
    - Edge Cases: API downtime, invalid request_id, response delays.

  - **Completed?**
    - Type: If node
    - Role: Checks if the video generation status equals "COMPLETED" to proceed or wait longer.
    - Configuration: Condition on the "status" field
    - Inputs: Output from Get status
    - Outputs: 
      - True: Continue to Get Url Video node
      - False: Loop back to Wait 60 sec. node to poll again
    - Edge Cases: Status values other than "COMPLETED" or unexpected API responses.

---

#### 2.3 Video Retrieval

- **Overview:** After video generation completes, this block obtains the video URL and downloads the video file.
- **Nodes Involved:** 
  - Get Url Video
  - Generate title
  - Get File Video

- **Node Details:**

  - **Get Url Video**
    - Type: HTTP Request
    - Role: Retrieves detailed video request information including video URL by request_id.
    - Configuration:
      - URL template: https://queue.fal.run/fal-ai/sora-2/requests/{{request_id}}
      - Authentication: Header Auth with separate credential (note: uses "Youtube Transcript Extractor API 1" credential, likely a generic API key)
    - Inputs: request_id from Completed? node
    - Outputs: JSON containing video metadata including video URL
    - Edge Cases: API access errors, credential mismatch, missing video URL.

  - **Generate title**
    - Type: OpenAI (LangChain) node
    - Role: Generates an SEO-optimized YouTube title using GPT-5 based on the original prompt.
    - Configuration:
      - Model: gpt-4o-mini (GPT-5 equivalent or advanced variant)
      - Prompt content: Input prompt text from original user input
      - System message: Instructions to create catchy, SEO-friendly titles up to 60 characters, in the prompt’s language.
    - Inputs: PROMPT from Get Url Video or original form submission
    - Outputs: Text response with generated title
    - Edge Cases: API rate limits, prompt truncation, off-topic title generation.

  - **Get File Video**
    - Type: HTTP Request
    - Role: Downloads the actual video file from the video URL obtained.
    - Configuration:
      - URL: Dynamic, taken from Get Url Video node output JSON path video.url
      - Method: GET
      - No authentication required
    - Inputs: video URL from Get Url Video
    - Outputs: Binary video data ready for upload
    - Edge Cases: Broken or inaccessible video URL, network timeouts, large file sizes.

---

#### 2.4 Video Upload to TikTok

- **Overview:** Uploads the downloaded video file to Postiz (a social media publishing platform) and schedules it for TikTok publishing with the generated title.
- **Nodes Involved:** 
  - Upload Video to Postiz
  - TikTok

- **Node Details:**

  - **Upload Video to Postiz**
    - Type: HTTP Request
    - Role: Uploads the binary video file to Postiz storage.
    - Configuration:
      - URL: https://api.postiz.com/public/v1/upload
      - Method: POST
      - Content-Type: multipart/form-data
      - Body: Binary data attached as "file" parameter
      - Authentication: Header Auth with Postiz API key
    - Inputs: Binary video from Get File Video
    - Outputs: JSON with uploaded file ID and path
    - Edge Cases: File size limits, authentication failure, network errors.

  - **TikTok**
    - Type: Postiz node (custom)
    - Role: Posts the uploaded video to TikTok with the generated title and scheduling.
    - Configuration:
      - Date: Current date/time (dynamic)
      - Post content: 
        - Image ID and path from Upload Video to Postiz response
        - Content: Generated title from Generate title node
      - IntegrationId: TikTok channel ID (to be set by user)
      - ShortLink: true (generate shortened links if applicable)
      - Credentials: Postiz account API key
    - Inputs: Uploaded video metadata and generated title
    - Outputs: Confirmation of post scheduling or publishing
    - Edge Cases: Invalid channel ID, API quota limits, scheduling conflicts.

---

#### 2.5 Supporting Notes and Configuration

- **Overview:** Provides user guidance and setup instructions through sticky notes.
- **Nodes Involved:** 
  - Sticky Note3
  - Sticky Note5
  - Sticky Note6
  - Sticky Note7
  - Sticky Note8
  - Sticky Note (unnamed)

- **Node Details:**

  - **Sticky Note3**
    - Content: Workflow description and overview, highlighting the use of Sora 2, Google Drive, GPT-5, TikTok, and form-based input.
    - Context: General introduction

  - **Sticky Note5**
    - Content: Instructions for workflow triggering (manual or scheduled every 5 minutes).
    - Context: Scheduling recommendations

  - **Sticky Note6**
    - Content: Instructions for obtaining the Fal.ai API key and setting HTTP header authentication in nodes.
    - Context: API key setup

  - **Sticky Note7**
    - Content: Reminder to set the API key in the respective node.
    - Context: API key application

  - **Sticky Note8**
    - Content: Postiz account creation, API key setup, channel connection, and obtaining ChannelId for TikTok.
    - Context: TikTok publishing setup

  - **Sticky Note (near TikTok node)**
    - Content: Reminder to set ChannelId in TikTok node.
    - Context: TikTok integration detail

---

### 3. Summary Table

| Node Name          | Node Type                | Functional Role                        | Input Node(s)                 | Output Node(s)                 | Sticky Note                                                                                       |
|--------------------|--------------------------|-------------------------------------|------------------------------|-------------------------------|-------------------------------------------------------------------------------------------------|
| On form submission  | Form Trigger             | Receives video prompt and duration  |                              | Create Video                  |                                                                                                 |
| Create Video        | HTTP Request             | Sends prompt/duration to Sora 2     | On form submission           | Wait 60 sec.                  | Set API Key created in Step 2                                                                   |
| Wait 60 sec.        | Wait                     | Waits before polling status          | Create Video                 | Get status                    |                                                                                                 |
| Get status          | HTTP Request             | Polls for video generation status    | Wait 60 sec.                 | Completed?                   |                                                                                                 |
| Completed?          | If                       | Checks if video generation is done   | Get status                   | Get Url Video / Wait 60 sec.  |                                                                                                 |
| Get Url Video       | HTTP Request             | Gets video URL and metadata          | Completed?                   | Generate title                |                                                                                                 |
| Generate title      | OpenAI (LangChain)       | Creates SEO-friendly video title     | Get Url Video                | Get File Video                |                                                                                                 |
| Get File Video      | HTTP Request             | Downloads video file                  | Generate title               | Upload Video to Postiz        |                                                                                                 |
| Upload Video to Postiz | HTTP Request           | Uploads video to Postiz storage      | Get File Video               | TikTok                       |                                                                                                 |
| TikTok             | Postiz                   | Posts video to TikTok channel        | Upload Video to Postiz       |                               | Set ChannelId Step 3                                                                             |
| Sticky Note3        | Sticky Note              | Workflow overview and description    |                              |                               | # Generate AI Videos (with audio), using Sora 2 and Upload to TikTok...                          |
| Sticky Note5        | Sticky Note              | Scheduling instructions              |                              |                               | ## STEP 3 - MAIN FLOW...                                                                          |
| Sticky Note6        | Sticky Note              | API key acquisition instructions    |                              |                               | ## STEP 1 - GET API KEY (YOURAPIKEY)...                                                        |
| Sticky Note7        | Sticky Note              | API key reminder                     |                              |                               | Set API Key created in Step 2                                                                   |
| Sticky Note8        | Sticky Note              | Postiz and TikTok integration guide |                              |                               | ## STEP 2 - Upload video on TikTok...                                                          |
| Sticky Note (unnamed near TikTok) | Sticky Note    | TikTok ChannelId reminder            |                              |                               | Set ChannelId Step 3                                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Form Trigger Node**
   - Type: Form Trigger
   - Name: On form submission
   - Configure form titled "Generate AI Videos (with audio)"
   - Add two required fields: "PROMPT" (text) and "DURATION" (number)
   - Save and note webhook URL for external trigger

2. **Create HTTP Request Node to Generate Video**
   - Type: HTTP Request
   - Name: Create Video
   - Method: POST
   - URL: https://queue.fal.run/fal-ai/sora-2/text-to-video
   - Authentication: HTTP Header Auth
     - Header Name: Authorization
     - Header Value: Key YOURAPIKEY (replace with actual Fal.ai API key)
   - Headers: Content-Type: application/json
   - Body (JSON): 
     ```json
     {
       "prompt": "{{ $json.PROMPT }}. Duration of the video: {{ $json.DURATION }}"
     }
     ```
   - Connect input from On form submission node

3. **Create Wait Node**
   - Type: Wait
   - Name: Wait 60 sec.
   - Time: 60 seconds
   - Connect input from Create Video node output

4. **Create HTTP Request Node to Get Status**
   - Type: HTTP Request
   - Name: Get status
   - Method: GET
   - URL: https://queue.fal.run/fal-ai/sora-2/requests/{{ $('Create Video').item.json.request_id }}/status
   - Authentication: HTTP Header Auth
     - Use the same Fal.ai API key credentials as Create Video node
   - Connect input from Wait 60 sec. node output

5. **Create If Node to Check Completion**
   - Type: If
   - Name: Completed?
   - Condition: Check if `{{ $json.status }}` equals "COMPLETED"
   - Connect input from Get status node output
   - True output connects to next block (Get Url Video)
   - False output loops back to Wait 60 sec. node (to poll again)

6. **Create HTTP Request Node to Get Video URL**
   - Type: HTTP Request
   - Name: Get Url Video
   - Method: GET
   - URL: https://queue.fal.run/fal-ai/sora-2/requests/{{ $json.request_id }}
   - Authentication: HTTP Header Auth
     - Use appropriate API key (can reuse Fal.ai key or corresponding credential)
   - Connect true output from Completed? node

7. **Create OpenAI LangChain Node to Generate Title**
   - Type: OpenAI (LangChain)
   - Name: Generate title
   - Model: gpt-4o-mini (or GPT-5 equivalent)
   - Messages:
     - User content: Input prompt from original prompt (`{{ $('Get new video').item.json.PROMPT }}`)
     - System content: Instructions for YouTube SEO title generation (as per workflow)
   - Connect input from Get Url Video node

8. **Create HTTP Request Node to Download Video File**
   - Type: HTTP Request
   - Name: Get File Video
   - Method: GET
   - URL: `={{ $('Get Url Video').item.json.video.url }}`
   - Connect input from Generate title node

9. **Create HTTP Request Node to Upload Video to Postiz**
   - Type: HTTP Request
   - Name: Upload Video to Postiz
   - Method: POST
   - URL: https://api.postiz.com/public/v1/upload
   - Authentication: HTTP Header Auth
     - Header Name: Authorization
     - Header Value: Postiz API key (from Postiz account)
   - Content-Type: multipart/form-data
   - Body: form binary data, parameter name "file", input from Get File Video binary data
   - Connect input from Get File Video node

10. **Create Postiz Node to Post on TikTok**
    - Type: Postiz (custom node)
    - Name: TikTok
    - Credentials: Postiz API key
    - Parameters:
      - Date: Current timestamp (use expression for now)
      - Post content:
        - Image: Use ID and path from Upload Video to Postiz response
        - Content: Generated title from Generate title node
      - IntegrationId: Set to your TikTok channel ID from Postiz
      - ShortLink: true
    - Connect input from Upload Video to Postiz node

11. **Add Sticky Notes as per need**
    - Add sticky notes to explain API key setup (Fal.ai and Postiz)
    - Add notes for scheduling triggers and TikTok integration steps

12. **Optional: Configure Scheduling**
    - Add a Schedule Trigger node if periodic automatic execution is needed, recommended every 5 minutes

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                                                          |
|-----------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| Create an account on [Fal.ai](https://fal.ai/) and obtain API Key for Sora 2 video generation.       | API key acquisition instructions                                                                         |
| Create an account on [Postiz](https://postiz.com/?ref=n3witalia) with a free 7-day trial for TikTok. | Postiz account and TikTok channel integration setup                                                      |
| Postiz allows connecting social accounts and obtaining ChannelId for scheduling posts.               | Postiz TikTok integration instructions                                                                   |
| Workflow requires self-hosted n8n due to community nodes compatibility.                              | Compatibility notice                                                                                      |
| Schedule trigger recommended at 5-minute intervals for batch processing or repeated runs.            | Scheduling advice                                                                                        |

---

**Disclaimer:** The text provided is generated strictly from an automated n8n workflow. It complies fully with content policies and contains no illegal, offensive, or protected content. All data handled is legal and public.