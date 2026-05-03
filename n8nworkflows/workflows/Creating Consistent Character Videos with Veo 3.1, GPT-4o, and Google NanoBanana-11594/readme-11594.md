Creating Consistent Character Videos with Veo 3.1, GPT-4o, and Google NanoBanana

https://n8nworkflows.xyz/workflows/creating-consistent-character-videos-with-veo-3-1--gpt-4o--and-google-nanobanana-11594


# Creating Consistent Character Videos with Veo 3.1, GPT-4o, and Google NanoBanana

### 1. Workflow Overview

This workflow automates the creation of consistent and cinematic character videos using a combination of AI models and cloud services. It targets content creators, influencers, and digital artists seeking to generate photorealistic video sequences featuring a consistent character across multiple frames, styled with specific locations, moods, and outfits.

**Logical Blocks:**

- **1.1 Initial Setup and Trigger:** Periodic initiation of the workflow and random selection of story elements (location and poses).
- **1.2 Story Creation via GPT-4o:** Generating detailed cinematic prompts for the first frame, last frame, and video motion.
- **1.3 Start Frame Image Generation:** Creating the first frame image using Google Nano Banana Edit model with KIE.AI.
- **1.4 End Frame Image Generation:** Creating the last frame image with pose and expression changes but consistent character and environment.
- **1.5 Veo 3.1 Video Generation:** Producing a cinematic video transitioning between the start and end frames using KIE.AI Veo 3.1.
- **1.6 Social Media Content Generation and Publishing:** Generating influencer-style titles, descriptions, hashtags, and posting to TikTok and Instagram, with previews sent to Telegram.
- **1.7 Error Handling and Polling Loops:** Switch and wait nodes that manage asynchronous processing status checks for image and video generation.

---

### 2. Block-by-Block Analysis

#### 1.1 Initial Setup and Trigger

**Overview:**  
Initiates the workflow every 6 hours, randomly selecting a location and three unique poses (with constraints) to be used throughout the story and media generation.

**Nodes Involved:**  
- Schedule Trigger  
- Code in JavaScript  

**Node Details:**

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Starts workflow every 6 hours  
  - Config: Interval of 6 hours  
  - Inputs: None  
  - Outputs: 1 (to Code in JavaScript)  
  - Edge Cases: Missed triggers if system down or paused

- **Code in JavaScript**  
  - Type: Function Node  
  - Role: Randomly selects one location (out of 100) respecting a ≤5% chance for restricted locations, and selects 3 unique poses (from 15) with labels and guidance.  
  - Key Logic: Random selection with re-roll for restricted location IDs. Outputs structured JSON with location and pose details.  
  - Inputs: Trigger data  
  - Outputs: JSON with `locationId`, `location` object, `poseIds`, and `poses` array  
  - Edge Cases: Infinite loop prevented by design; random may repeat over multiple runs but no duplicates within single run.

#### 1.2 Story Creation via GPT-4o

**Overview:**  
Generates cinematic and photorealistic text prompts for the first frame, last frame, and a dynamic video motion prompt, ensuring character consistency and strict formatting for downstream processing.

**Nodes Involved:**  
- Story Creator Agent (LangChain LLM Chain)  
- Code1 (Function Node)  

**Node Details:**

- **Story Creator Agent**  
  - Type: LangChain Chain LLM  
  - Role: Uses GPT-4o (model `gpt-4.1-mini`) to create JSON prompts describing first frame, last frame, video motion, and outfit details.  
  - Configuration: Detailed prompt enforces strict JSON output format and consistent character identity.  
  - Inputs: JSON from previous JavaScript node (location, poses)  
  - Outputs: JSON with keys `first_frame`, `last_frame`, `video_prompt`, `dress`  
  - Edge Cases: Parsing errors if model outputs invalid JSON; retry enabled.

- **Code1**  
  - Type: Function Node  
  - Role: Parses and extracts text fields from Story Creator Agent output for further use.  
  - Inputs: JSON from Story Creator Agent  
  - Outputs: JSON containing `first_frame`, `last_frame`, `video_prompt`, and `dress` fields  
  - Edge Cases: JSON parse failures if input malformed.

#### 1.3 Start Frame Image Generation

**Overview:**  
Creates the first frame image using the Google Nano Banana Edit model on KIE.AI with fixed reference images of the character and the cinematic prompt generated earlier. The node submits the task and subsequent nodes poll for completion.

**Nodes Involved:**  
- Create Start Frame (HTTP Request)  
- Wait Start Frame (Wait)  
- Get Start Frame (HTTP Request)  
- Switch Start Frame (Switch)  
- Start Frame Image Url (Code)  
- Get Start Frame Binary (HTTP Request)  
- Send Start Frame (Telegram)  

**Node Details:**

- **Create Start Frame**  
  - Type: HTTP Request  
  - Role: Submits a task to KIE.AI to generate the first frame image  
  - Config: Uses model `google/nano-banana-edit`, prompt includes first frame description plus dress; includes 5 fixed reference image URLs for character consistency; output format JPEG 9:16 aspect ratio.  
  - Auth: HTTP Bearer with KIE.AI API key  
  - Inputs: JSON with cinematic prompt and dress  
  - Outputs: Task creation response with taskId  
  - Edge Cases: API failures, auth errors

- **Wait Start Frame**  
  - Type: Wait Node  
  - Role: Delays 5 seconds before polling for image generation status  
  - Inputs: From Create Start Frame  
  - Outputs: Passes control to Get Start Frame

- **Get Start Frame**  
  - Type: HTTP Request  
  - Role: Polls KIE.AI API for task status using taskId  
  - Inputs: taskId from previous node  
  - Outputs: JSON status with `data.state` field  
  - Edge Cases: API errors, timeout

- **Switch Start Frame**  
  - Type: Switch Node  
  - Role: Routes workflow based on task status: `fail`, `success`, `generating`, `queuing`, `waiting`  
  - Outputs:  
    - `success`: passes to Start Frame Image Url  
    - Other states: loops back to Wait Start Frame for retry  
  - Edge Cases: Unexpected states or missing fields

- **Start Frame Image Url**  
  - Type: Function Node  
  - Role: Extracts image result URLs from task response JSON  
  - Inputs: Task success JSON  
  - Outputs: `resultUrls` array with URLs of generated images  
  - Edge Cases: Malformed JSON or missing URLs

- **Get Start Frame Binary**  
  - Type: HTTP Request  
  - Role: Downloads binary image data from the first URL in `resultUrls`  
  - Inputs: URL from Start Frame Image Url  
  - Outputs: Binary image data  
  - Edge Cases: Network errors, 404

- **Send Start Frame**  
  - Type: Telegram Node  
  - Role: Sends the generated start frame image as a photo to a preconfigured Telegram chat for preview  
  - Inputs: Binary image data  
  - Config: Telegram chat ID fixed in parameters  
  - Edge Cases: Telegram API limits, invalid chat ID

#### 1.4 End Frame Image Generation

**Overview:**  
Generates the last frame image based on the last pose prompt, referencing the start frame image for visual consistency. The process includes task creation, polling for completion, extracting URLs, downloading binary, and sending previews.

**Nodes Involved:**  
- Create End Frame (HTTP Request)  
- Wait End Frame (Wait)  
- Get End Frame (HTTP Request)  
- Switch End Frame (Switch)  
- End Frame Image Url (Code)  
- Get End Frame Binary (HTTP Request)  
- Send End Frame (Telegram)  

**Node Details:**

- **Create End Frame**  
  - Type: HTTP Request  
  - Role: Submits task with last frame prompt and start frame image URL as reference  
  - Config: Model `google/nano-banana-edit`, output format JPEG 9:16  
  - Inputs: last frame prompt and image URL from Start Frame Image Url node  
  - Auth: HTTP Bearer KIE.AI  
  - Edge Cases: API or network errors

- **Wait End Frame**  
  - Type: Wait Node  
  - Role: Delays 5 seconds before polling status  
  - Inputs: From Create End Frame  
  - Outputs: Get End Frame

- **Get End Frame**  
  - Type: HTTP Request  
  - Role: Polls task status at KIE.AI API  
  - Inputs: taskId  
  - Outputs: Status JSON  
  - Edge Cases: API failure, timeouts

- **Switch End Frame**  
  - Type: Switch Node  
  - Role: Routes based on task status like Start Frame switch node  
  - Outputs:  
    - `success`: to End Frame Image Url  
    - Others: loops to Wait End Frame  
  - Edge Cases: Unexpected states

- **End Frame Image Url**  
  - Type: Function Node  
  - Role: Extracts result image URLs  
  - Inputs: Task success response  
  - Outputs: `resultUrls` array  
  - Edge Cases: Parsing errors

- **Get End Frame Binary**  
  - Type: HTTP Request  
  - Role: Downloads final frame image binary  
  - Inputs: URL from End Frame Image Url node  
  - Outputs: Binary image data  
  - Edge Cases: Network errors

- **Send End Frame**  
  - Type: Telegram Node  
  - Role: Sends final frame image preview to Telegram chat  
  - Inputs: Binary image data  
  - Edge Cases: Telegram API limits

#### 1.5 Veo 3.1 Video Generation

**Overview:**  
Creates a seamless, cinematic video transitioning from the start frame to the end frame using the Veo 3.1 fast model from KIE.AI. The process is asynchronous, involving task submission, webhook-based waiting, status polling, and final video URL extraction.

**Nodes Involved:**  
- Veo 3.1 (HTTP Request)  
- wait veo 3.1 (Wait)  
- Get veo 3.1 (HTTP Request)  
- Switch veo 3.1 (Switch)  
- Get Url veo 3.1 (Code)  
- Get veo 3.1 Binary (HTTP Request)  
- Send Video (Telegram)  

**Node Details:**

- **Veo 3.1**  
  - Type: HTTP Request  
  - Role: Submits video generation task to KIE.AI Veo 3.1 fast model  
  - Inputs: Video prompt, start and end frame image URLs, watermark text, aspect ratio 9:16  
  - Auth: HTTP Bearer KIE.AI  
  - Edge Cases: API limits, auth failures

- **wait veo 3.1**  
  - Type: Wait Node  
  - Role: Waits (webhook-based, but with fallback polling) for video processing  
  - Inputs: After Veo 3.1 task submission and polling loops  
  - Outputs: Triggers Get veo 3.1 node

- **Get veo 3.1**  
  - Type: HTTP Request  
  - Role: Polls KIE.AI API for video task status using taskId  
  - Outputs: Status JSON with `data.response.taskId` and `resultUrls`  
  - Edge Cases: API downtime

- **Switch veo 3.1**  
  - Type: Switch Node  
  - Role: Routes based on presence of `taskId` in response:  
    - `empty`: loops back to wait node for polling  
    - `success`: proceeds to extract video URL  
  - Edge Cases: Missing or malformed data

- **Get Url veo 3.1**  
  - Type: Function Node  
  - Role: Extracts first video URL from resultUrls array  
  - Outputs: JSON with `url` field  
  - Edge Cases: Missing URLs

- **Get veo 3.1 Binary**  
  - Type: HTTP Request  
  - Role: Downloads the video binary from extracted URL  
  - Outputs: Binary video data  
  - Edge Cases: Network errors

- **Send Video**  
  - Type: Telegram Node  
  - Role: Sends generated video as Telegram message to preview channel  
  - Inputs: Binary video data  
  - Edge Cases: Telegram video size limits

#### 1.6 Social Media Content Generation and Publishing

**Overview:**  
Generates catchy titles, descriptions, and hashtags for social media posts using GPT-4o, then publishes the video content to TikTok and Instagram using Blotato API, while sending previews via Telegram.

**Nodes Involved:**  
- Title Description (LangChain LLM Chain)  
- Json parser (Structured Output Parser)  
- Tiktok1 (Blotato Node)  
- Instagram (Blotato Node)  

**Node Details:**

- **Title Description**  
  - Type: LangChain Chain LLM  
  - Role: Generates a title, short description, and 5 hashtags in influencer style based on visual description input  
  - Inputs: Text prompt with visual description from video generation node  
  - Outputs: JSON with `title`, `description`, `hashTags`  
  - Edge Cases: Model output variability

- **Json parser**  
  - Type: LangChain Output Parser Structured  
  - Role: Parses the AI output into structured JSON for Title Description node  
  - Inputs: AI raw output  
  - Outputs: Structured JSON fields

- **Tiktok1**  
  - Type: Blotato Node  
  - Role: Posts the video to TikTok with AI-generated content label  
  - Config: Uses Blotato account with TikTok platform, video URL and generated text content  
  - Edge Cases: API limits, auth errors

- **Instagram**  
  - Type: Blotato Node  
  - Role: Posts the same video and content to Instagram  
  - Inputs: Video URL and title/hashtags from Title Description  
  - Edge Cases: API limits, auth errors

#### 1.7 Error Handling and Polling Loops

**Overview:**  
Switch nodes and wait nodes manage asynchronous API calls by polling task statuses and retrying or looping on intermediate states until success or failure.

**Nodes Involved:**  
- Switch Start Frame  
- Switch End Frame  
- Switch veo 3.1  
- Wait Start Frame  
- Wait End Frame  
- wait veo 3.1  

**Node Details:**

- Switch nodes check the `state` field in the JSON response to route workflow execution accordingly (success, fail, generating, etc.).  
- Wait nodes pause for a defined period (usually 5 seconds) before re-polling.  
- Edge Cases:  
  - Potential infinite loops if API never returns success or fail states (mitigate with timeouts).  
  - Handling unexpected or missing state values.  
  - Network or API failures during polling.

---

### 3. Summary Table

| Node Name           | Node Type                         | Functional Role                                     | Input Node(s)            | Output Node(s)           | Sticky Note                                                                                                                                                                                                                                                                                                                                                                                                                                             |
|---------------------|----------------------------------|----------------------------------------------------|--------------------------|--------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger    | Schedule Trigger                 | Initiates workflow every 6 hours                    | None                     | Code in JavaScript        | See Sticky Note2: Setup & Integration Guide                                                                                                                                                                                                                                                                                                                                                                                                            |
| Code in JavaScript  | Function Node                   | Randomly picks location and 3 unique poses          | Schedule Trigger          | Story Creator Agent       | See Sticky Note10: Story Creation Section                                                                                                                                                                                                                                                                                                                                                                                                               |
| Story Creator Agent | LangChain Chain LLM             | Generates cinematic prompt JSON for frames & video | Code in JavaScript        | Code1                     | See Sticky Note10: Story Creation Section                                                                                                                                                                                                                                                                                                                                                                                                               |
| Code1               | Function Node                   | Extracts cinematic prompt fields from JSON          | Story Creator Agent       | Create Start Frame        |                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| Create Start Frame  | HTTP Request                   | Creates start frame image task on KIE.AI            | Code1                     | Wait Start Frame          | See Sticky Note11: Start Frame Image Generation                                                                                                                                                                                                                                                                                                                                                                                                        |
| Wait Start Frame    | Wait Node                      | Waits 5 seconds before polling                       | Create Start Frame        | Get Start Frame           | See Sticky Note11: Start Frame Image Generation                                                                                                                                                                                                                                                                                                                                                                                                        |
| Get Start Frame     | HTTP Request                   | Polls KIE.AI API for start frame generation status  | Wait Start Frame          | Switch Start Frame        | See Sticky Note11: Start Frame Image Generation                                                                                                                                                                                                                                                                                                                                                                                                        |
| Switch Start Frame  | Switch Node                    | Routes based on start frame task status              | Get Start Frame           | Start Frame Image Url, Wait Start Frame | See Sticky Note11: Start Frame Image Generation                                                                                                                                                                                                                                                                                                                                                                                                        |
| Start Frame Image Url| Function Node                  | Extracts start frame image URLs                       | Switch Start Frame        | Get Start Frame Binary, Create End Frame | See Sticky Note11: Start Frame Image Generation                                                                                                                                                                                                                                                                                                                                                                                                        |
| Get Start Frame Binary| HTTP Request                  | Downloads binary start frame image                    | Start Frame Image Url     | Send Start Frame          | See Sticky Note11: Start Frame Image Generation                                                                                                                                                                                                                                                                                                                                                                                                        |
| Send Start Frame    | Telegram Node                  | Sends start frame image preview to Telegram          | Get Start Frame Binary    |                          | See Sticky Note14: Social Media Publishing                                                                                                                                                                                                                                                                                                                                                                                                              |
| Create End Frame    | HTTP Request                   | Creates end frame image task on KIE.AI                | Start Frame Image Url     | Wait End Frame            | See Sticky Note15: End Frame Image Generation                                                                                                                                                                                                                                                                                                                                                                                                          |
| Wait End Frame      | Wait Node                      | Waits 5 seconds before polling end frame status      | Create End Frame          | Get End Frame             | See Sticky Note15: End Frame Image Generation                                                                                                                                                                                                                                                                                                                                                                                                          |
| Get End Frame       | HTTP Request                   | Polls KIE.AI API for end frame generation status     | Wait End Frame            | Switch End Frame          | See Sticky Note15: End Frame Image Generation                                                                                                                                                                                                                                                                                                                                                                                                          |
| Switch End Frame    | Switch Node                    | Routes based on end frame task status                 | Get End Frame             | End Frame Image Url, Wait End Frame | See Sticky Note15: End Frame Image Generation                                                                                                                                                                                                                                                                                                                                                                                                          |
| End Frame Image Url | Function Node                  | Extracts end frame image URLs                          | Switch End Frame          | Get End Frame Binary, Veo 3.1 | See Sticky Note15: End Frame Image Generation                                                                                                                                                                                                                                                                                                                                                                                                          |
| Get End Frame Binary| HTTP Request                   | Downloads binary end frame image                       | End Frame Image Url       | Send End Frame            | See Sticky Note14: Social Media Publishing                                                                                                                                                                                                                                                                                                                                                                                                              |
| Send End Frame      | Telegram Node                  | Sends end frame image preview to Telegram             | Get End Frame Binary      |                          | See Sticky Note14: Social Media Publishing                                                                                                                                                                                                                                                                                                                                                                                                              |
| Veo 3.1             | HTTP Request                   | Submits cinematic video generation task to KIE.AI    | End Frame Image Url       | wait veo 3.1              | See Sticky Note13: Veo 3.1 Video Generation                                                                                                                                                                                                                                                                                                                                                                                                            |
| wait veo 3.1        | Wait Node                      | Waits for video processing (webhook/polling)         | Veo 3.1                   | Get veo 3.1               | See Sticky Note13: Veo 3.1 Video Generation                                                                                                                                                                                                                                                                                                                                                                                                            |
| Get veo 3.1         | HTTP Request                   | Polls KIE.AI video generation task status             | wait veo 3.1              | Switch veo 3.1            | See Sticky Note13: Veo 3.1 Video Generation                                                                                                                                                                                                                                                                                                                                                                                                            |
| Switch veo 3.1      | Switch Node                    | Routes based on video task status                      | Get veo 3.1               | wait veo 3.1, Get Url veo 3.1 | See Sticky Note13: Veo 3.1 Video Generation                                                                                                                                                                                                                                                                                                                                                                                                            |
| Get Url veo 3.1     | Function Node                  | Extracts video URL from response JSON                  | Switch veo 3.1            | Get veo 3.1 Binary, Title Description | See Sticky Note13, Sticky Note14                                                                                                                                                                                                                                                                                                                                                                                                                         |
| Get veo 3.1 Binary  | HTTP Request                   | Downloads video binary from extracted URL              | Get Url veo 3.1           | Send Video                | See Sticky Note14: Social Media Publishing                                                                                                                                                                                                                                                                                                                                                                                                              |
| Send Video          | Telegram Node                  | Sends video preview via Telegram                        | Get veo 3.1 Binary        |                          | See Sticky Note14: Social Media Publishing                                                                                                                                                                                                                                                                                                                                                                                                              |
| Title Description   | LangChain Chain LLM            | Generates title, description, and hashtags             | Get Url veo 3.1           | Tiktok1                   | See Sticky Note14: Social Media Publishing                                                                                                                                                                                                                                                                                                                                                                                                              |
| Json parser         | LangChain Output Parser        | Parses AI output into structured JSON                   | Title Description         | Title Description         | See Sticky Note14: Social Media Publishing                                                                                                                                                                                                                                                                                                                                                                                                              |
| Tiktok1             | Blotato Node                  | Posts video with AI label to TikTok                      | Title Description         | Instagram                 | See Sticky Note14: Social Media Publishing                                                                                                                                                                                                                                                                                                                                                                                                              |
| Instagram           | Blotato Node                  | Posts same video and content to Instagram                | Tiktok1                   |                          | See Sticky Note14: Social Media Publishing                                                                                                                                                                                                                                                                                                                                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Set interval: Every 6 hours  
   - No credentials needed  
   - Connect output to next node

2. **Create Function Node: Code in JavaScript**  
   - Paste provided JS code to randomly pick location and 3 unique poses with labels and guidance  
   - Output structured JSON with location and poses  
   - Connect input from Schedule Trigger  
   - Connect output to Story Creator Agent

3. **Create LangChain Chain LLM Node: Story Creator Agent**  
   - Model: GPT-4o (`gpt-4.1-mini`)  
   - Input: JSON from previous node with location and poses  
   - Prompt: Detailed, structured prompt enforcing JSON output with `first_frame`, `last_frame`, `video_prompt`, and `dress` fields (copy from provided prompt)  
   - Enable output parser for JSON  
   - Connect input from Code in JavaScript  
   - Connect output to Code1

4. **Create Function Node: Code1**  
   - JS code to parse LLM output and produce JSON with keys `first_frame`, `last_frame`, `video_prompt`, `dress`  
   - Connect input from Story Creator Agent  
   - Connect output to Create Start Frame

5. **Create HTTP Request Node: Create Start Frame**  
   - Method: POST  
   - URL: `https://api.kie.ai/api/v1/jobs/createTask`  
   - Body: JSON with model `google/nano-banana-edit`, prompt using `first_frame` plus dress, and 5 fixed reference image URLs for character consistency  
   - Output format: JPEG, aspect ratio 9:16  
   - Authentication: HTTP Bearer with KIE.AI API key  
   - Connect input from Code1  
   - Connect output to Wait Start Frame

6. **Create Wait Node: Wait Start Frame**  
   - Wait 5 seconds  
   - Connect input from Create Start Frame  
   - Connect output to Get Start Frame

7. **Create HTTP Request Node: Get Start Frame**  
   - Method: GET  
   - URL: `https://api.kie.ai/api/v1/jobs/recordInfo`  
   - Query Param: `taskId` from previous node response  
   - Authentication: HTTP Bearer KIE.AI  
   - Connect input from Wait Start Frame  
   - Connect output to Switch Start Frame

8. **Create Switch Node: Switch Start Frame**  
   - Routing based on `data.state` in API response:  
     - `success` → Start Frame Image Url  
     - `fail` → end or log error  
     - `generating`, `queuing`, `waiting` → loop back to Wait Start Frame  
   - Connect input from Get Start Frame

9. **Create Function Node: Start Frame Image Url**  
   - Extracts `resultUrls` array from task response JSON  
   - Connect input from Switch Start Frame `success` output  
   - Connect output to Get Start Frame Binary and Create End Frame

10. **Create HTTP Request Node: Get Start Frame Binary**  
    - Downloads image binary from first URL in `resultUrls`  
    - Connect input from Start Frame Image Url  
    - Connect output to Send Start Frame

11. **Create Telegram Node: Send Start Frame**  
    - Operation: sendPhoto  
    - Chat ID: configured Telegram chat  
    - File: binary from Get Start Frame Binary  
    - Credentials: Telegram Bot Token  
    - Connect input from Get Start Frame Binary  

12. **Create HTTP Request Node: Create End Frame**  
    - Method: POST  
    - URL: `https://api.kie.ai/api/v1/jobs/createTask`  
    - Body: JSON with model `google/nano-banana-edit`, prompt `last_frame`, image_urls referencing start frame image URL, output format JPEG 9:16  
    - Authentication: HTTP Bearer KIE.AI  
    - Connect input from Start Frame Image Url (to get URL for reference)  
    - Connect output to Wait End Frame

13. **Create Wait Node: Wait End Frame**  
    - Wait 5 seconds  
    - Connect input from Create End Frame  
    - Connect output to Get End Frame

14. **Create HTTP Request Node: Get End Frame**  
    - Same configuration as Get Start Frame but for end frame taskId  
    - Connect input from Wait End Frame  
    - Connect output to Switch End Frame

15. **Create Switch Node: Switch End Frame**  
    - Same logic as Switch Start Frame based on `data.state`  
    - `success` → End Frame Image Url  
    - Others → loop to Wait End Frame  
    - Connect input from Get End Frame

16. **Create Function Node: End Frame Image Url**  
    - Extract `resultUrls` from end frame task response  
    - Connect input from Switch End Frame `success` output  
    - Connect output to Get End Frame Binary and Veo 3.1

17. **Create HTTP Request Node: Get End Frame Binary**  
    - Download binary from first URL in `resultUrls`  
    - Connect input from End Frame Image Url  
    - Connect output to Send End Frame

18. **Create Telegram Node: Send End Frame**  
    - Operation: sendPhoto  
    - Chat ID: configured Telegram chat  
    - File: binary from Get End Frame Binary  
    - Credentials: Telegram Bot Token  
    - Connect input from Get End Frame Binary

19. **Create HTTP Request Node: Veo 3.1**  
    - Method: POST  
    - URL: `https://api.kie.ai/api/v1/veo/generate`  
    - Body: JSON including video prompt, start and end frame image URLs, model `veo3_fast`, watermark, aspect ratio 9:16, fixed seed, etc.  
    - Authentication: HTTP Bearer KIE.AI  
    - Connect input from End Frame Image Url  
    - Connect output to wait veo 3.1

20. **Create Wait Node: wait veo 3.1**  
    - No fixed wait time (webhook-based with fallback polling)  
    - Connect input from Veo 3.1  
    - Connect output to Get veo 3.1

21. **Create HTTP Request Node: Get veo 3.1**  
    - GET request to `https://api.kie.ai/api/v1/veo/record-info` with taskId  
    - Authentication: HTTP Bearer KIE.AI  
    - Connect input from wait veo 3.1  
    - Connect output to Switch veo 3.1

22. **Create Switch Node: Switch veo 3.1**  
    - Routes based on presence of `taskId` in response  
    - `empty` → loops to wait veo 3.1  
    - `success` → Get Url veo 3.1  
    - Connect input from Get veo 3.1

23. **Create Function Node: Get Url veo 3.1**  
    - Extracts video URL from response JSON  
    - Connect input from Switch veo 3.1 `success` output  
    - Connect output to Get veo 3.1 Binary and Title Description

24. **Create HTTP Request Node: Get veo 3.1 Binary**  
    - Downloads video binary from extracted URL  
    - Connect input from Get Url veo 3.1  
    - Connect output to Send Video

25. **Create Telegram Node: Send Video**  
    - Operation: sendVideo  
    - Chat ID: configured Telegram chat  
    - File: video binary  
    - Credentials: Telegram Bot Token  
    - Connect input from Get veo 3.1 Binary

26. **Create LangChain Chain LLM Node: Title Description**  
    - Model: GPT-4o `gpt-4.1-mini`  
    - Input: Visual description from video generation output  
    - Prompt: Generate catchy title, influencer style description, 5 hashtags  
    - Output parser enabled for structured JSON  
    - Connect input from Get Url veo 3.1

27. **Create LangChain Output Parser Node: Json parser**  
    - Parses AI output into structured title, description, hashtags  
    - Connect input from Title Description  
    - Connect output to Tiktok1

28. **Create Blotato Node: Tiktok1**  
    - Platform: TikTok  
    - Inputs: Video URL and content text (title + hashtags)  
    - Account: configured Blotato TikTok account  
    - AI-generated content label enabled  
    - Connect input from Title Description

29. **Create Blotato Node: Instagram**  
    - Platform: Instagram  
    - Inputs: Video URL and content text (title + hashtags)  
    - Account: configured Blotato Instagram account  
    - Connect input from Tiktok1

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | Context or Link                                                                                  |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Muhammad Farooq Iqbal - Automation Expert & n8n Creator. Contact for workflow customization and support: Email: mfarooqiqbal143@gmail.com, Phone: +923036991118, LinkedIn: https://linkedin.com/in/muhammadfarooqiqbal, Portfolio: https://mfarooqone.github.io/n8n/, UpWork: https://www.upwork.com/freelancers/~011aeba159896e2eba.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | Sticky Note7                                                                                   |
| Setup & Integration Guide: Requires API keys for OpenAI (https://platform.openai.com/api-keys), KIE.AI (https://kie.ai/), Blotato (https://blotato.com/), and Telegram Bot Token (via @BotFather). Reference images for the character must be configured in the Create Start Frame node. Telegram chat ID and Blotato accounts must be set.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | Sticky Note2                                                                                   |
| Story Creation Section: Random location and pose selection with 100 unique locations and 15 poses. GPT-4o generates cinematic prompts ensuring photorealistic style and character consistency.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | Sticky Note10                                                                                  |
| Start Frame Image Generation: Uses Google Nano Banana Edit model via KIE.AI to create the first frame with fixed character reference images, polling for completion, outputting 9:16 JPEG images.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | Sticky Note11                                                                                  |
| End Frame Image Generation: Similar to start frame but changes only pose/expression, maintains lighting, mood, and outfit consistency. Polling and extraction of final image URL.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | Sticky Note15                                                                                  |
| Veo 3.1 Video Generation: Uses KIE.AI Veo 3.1 Fast model to generate cinematic transition video with watermark, 9:16 aspect ratio. Webhook and polling for completion, extracting video URL.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Sticky Note13                                                                                  |
| Social Media Publishing: Generates catchy titles, descriptions, and hashtags via GPT-4o, posts to TikTok and Instagram via Blotato with AI-generated content label, and sends previews via Telegram.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | Sticky Note14                                                                                  |

---

**Disclaimer:** The content provided originates exclusively from an automated workflow created with n8n, a workflow automation tool. All data processed and generated complies strictly with current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.