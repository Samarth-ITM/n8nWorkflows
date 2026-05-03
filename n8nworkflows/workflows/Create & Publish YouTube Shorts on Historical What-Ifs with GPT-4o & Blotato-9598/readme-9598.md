Create & Publish YouTube Shorts on Historical What-Ifs with GPT-4o & Blotato

https://n8nworkflows.xyz/workflows/create---publish-youtube-shorts-on-historical-what-ifs-with-gpt-4o---blotato-9598


# Create & Publish YouTube Shorts on Historical What-Ifs with GPT-4o & Blotato

### 1. Workflow Overview

This workflow automates the end-to-end process of creating and publishing daily YouTube Shorts focused on “What If” historical scenarios using AI and video generation services. It is designed for content creators and marketers who want to scale viral faceless video production with minimal manual intervention.

The workflow is logically divided into three main blocks:

- **1.1 Scheduled Trigger & AI Content Generation:**  
  Automatically triggers daily, invokes an AI agent (GPT-4o) to brainstorm 50 historical “What If” video ideas, randomly selects one, researches facts, and generates a 60-second video script along with a short caption and hook title.

- **1.2 Video Creation & Preparation:**  
  Takes the AI-generated script and parameters to request Blotato’s video creation API for producing a cinematic, faceless video. It waits for video generation completion, fetches the final video URL, and prepares publishing metadata.

- **1.3 Upload & Publish to YouTube:**  
  Uploads the finished video to Blotato’s media backend, then posts it automatically to YouTube Shorts using the generated video media URL, captions, and title.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger & AI Content Generation

- **Overview:**  
  This block triggers the workflow daily at 10 AM, initiates the AI brainstorming process, and structures the AI response into usable JSON for subsequent nodes.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Brainstorm Idea  
  - Structured Output Parser  
  - AI Agent1

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Initiates workflow daily at a fixed time (10:00 AM)  
    - Configuration: Interval trigger set to 10:00 AM daily  
    - Input: None  
    - Output: Triggers Brainstorm Idea node  
    - Edge cases: Missed trigger if n8n instance is down at trigger time

  - **Brainstorm Idea**  
    - Type: Langchain LLM Chat (OpenAI GPT-4o)  
    - Role: Generates 50 “What If history” video ideas as a prompt for the AI Agent  
    - Configuration: Model set to GPT-4o, no special options  
    - Credentials: Requires OpenAI API key  
    - Input: Trigger from Schedule Trigger  
    - Output: Passes generated ideas to AI Agent1  
    - Edge cases: API key invalid, quota exceeded, network issues

  - **Structured Output Parser**  
    - Type: Langchain Structured Output Parser  
    - Role: Parses AI Agent’s output into JSON with fields for script, caption, and title  
    - Configuration: JSON schema example provided for script, caption, title  
    - Input: AI Agent1 output  
    - Output: Structured JSON data for video preparation  
    - Edge cases: Parsing failures if AI output is malformed

  - **AI Agent1**  
    - Type: Langchain Agent (GPT-4o)  
    - Role: Core AI agent that:  
      1. Brainstorms 50 viral faceless video ideas on “What if history...”  
      2. Selects one idea randomly and researches it  
      3. Writes a 60-second viral faceless video script in simple language with a hook  
      4. Writes a 2-sentence caption with hashtags including #ai  
    - Configuration: Custom instructions in JSON format with output enforced as JSON  
    - Input: Brainstorm Idea output  
    - Output: Script, caption, and title JSON (passed to Structured Output Parser)  
    - Edge cases: API failures, incomplete output, or invalid JSON formatting

---

#### 2.2 Video Creation & Preparation

- **Overview:**  
  This block prepares video creation parameters using the AI-generated script, calls Blotato’s video creation API, waits for completion, and fetches the generated video data.

- **Nodes Involved:**  
  - Prepare Video  
  - Create Video  
  - Wait  
  - Get Video  
  - Prepare for Publish

- **Node Details:**

  - **Prepare Video**  
    - Type: Set Node  
    - Role: Constructs JSON payload with AI script and video generation options for Blotato API  
    - Configuration:  
      - `blotato_api_key`: (empty, to be filled)  
      - Video template: “empty” (default)  
      - Voice: ElevenLabs multilingual voice ID  
      - Caption position: bottom  
      - Script: populated from AI Agent1’s parsed output  
      - Style: cinematic  
      - Animation flags: animate first image true, animate all false  
      - Text-to-image and image-to-video models specified  
    - Input: JSON from AI Agent1  
    - Output: Payload for Create Video node  
    - Edge cases: Missing or invalid API key, invalid script JSON

  - **Create Video**  
    - Type: HTTP Request  
    - Role: Sends a POST request to Blotato’s video creation endpoint with parameters from Prepare Video  
    - Configuration:  
      - URL: Blotato video creation API  
      - Headers: API key from payload  
      - Body: JSON with template, voice, caption position, script, style, animation flags, and AI models  
    - Input: Prepare Video output  
    - Output: Response containing video creation task details (including task ID)  
    - Edge cases: API errors, network timeouts, quota limits

  - **Wait**  
    - Type: Wait Node  
    - Role: Pauses workflow for 10 minutes to allow video creation completion  
    - Configuration: 10 minutes delay  
    - Input: Create Video output  
    - Output: Triggers Get Video node after wait  
    - Edge cases: Wait duration too short or long might cause premature or delayed requests

  - **Get Video**  
    - Type: HTTP Request  
    - Role: Polls Blotato API for the created video status and retrieves video media URL  
    - Configuration:  
      - URL dynamically built using video creation ID from prior node  
      - Header includes Blotato API key  
    - Input: Wait output  
    - Output: Video metadata including mediaUrl  
    - Edge cases: Video not ready, API errors, invalid ID

  - **Prepare for Publish**  
    - Type: Set Node  
    - Role: Aggregates all necessary metadata and credentials for publishing  
    - Configuration:  
      - Copies Blotato API key from Prepare Video  
      - Sets YouTube account ID and other social media placeholders (empty except YouTube)  
      - Includes final text captions and script captions in JSON form  
    - Input: Get Video output  
    - Output: Payload for Upload to Blotato and YT Post nodes  
    - Edge cases: Missing YouTube ID, empty captions

---

#### 2.3 Upload & Publish to YouTube

- **Overview:**  
  This block uploads the generated video to Blotato’s media service and then posts it automatically to YouTube Shorts with captions and title.

- **Nodes Involved:**  
  - Upload to Blotato  
  - YT Post

- **Node Details:**

  - **Upload to Blotato**  
    - Type: HTTP Request  
    - Role: Uploads the final video media URL to Blotato’s media backend  
    - Configuration:  
      - URL: Blotato media endpoint  
      - Method: POST  
      - Body: JSON with `url` field from Get Video mediaUrl  
      - Header: Blotato API key from Prepare for Publish  
    - Input: Prepare for Publish output  
    - Output: Upload confirmation, triggers YT Post  
    - Edge cases: Invalid URL, upload failure, API key issues

  - **YT Post**  
    - Type: HTTP Request  
    - Role: Posts the video to YouTube with the generated title, caption, and visibility  
    - Configuration:  
      - URL: Blotato posts endpoint  
      - Method: POST  
      - Body JSON includes:  
        - YouTube account ID  
        - Caption text from Prepare for Publish  
        - Media URLs array with video mediaUrl  
        - Platform set to “youtube”  
        - Target with title from AI Agent1, public privacy, notification enabled, and not made for kids  
      - Header: Blotato API key  
    - Input: Upload to Blotato output  
    - Output: Confirmation of YouTube post  
    - Edge cases: Invalid account ID, API errors, network issues

---

### 3. Summary Table

| Node Name           | Node Type                          | Functional Role                            | Input Node(s)           | Output Node(s)          | Sticky Note                                                                                          |
|---------------------|----------------------------------|-------------------------------------------|------------------------|-------------------------|----------------------------------------------------------------------------------------------------|
| Schedule Trigger     | Schedule Trigger                  | Daily workflow trigger at 10:00 AM        | —                      | Brainstorm Idea         |                                                                                                    |
| Brainstorm Idea      | Langchain LLM Chat (OpenAI GPT)  | Generate 50 viral “What If history” ideas | Schedule Trigger        | AI Agent1               |                                                                                                    |
| AI Agent1           | Langchain Agent (GPT-4o)          | Select idea, research, write script & caption | Brainstorm Idea         | Prepare Video           |                                                                                                    |
| Structured Output Parser | Langchain Output Parser        | Parse AI Agent output to structured JSON  | AI Agent1               | Prepare Video           |                                                                                                    |
| Prepare Video        | Set Node                         | Prepare Blotato video creation payload    | AI Agent1 / Structured Output Parser | Create Video            |                                                                                                    |
| Create Video         | HTTP Request                    | Request Blotato to create video from script | Prepare Video           | Wait                    |                                                                                                    |
| Wait                 | Wait Node                        | Wait 10 minutes for video processing      | Create Video            | Get Video               |                                                                                                    |
| Get Video            | HTTP Request                    | Fetch video metadata including media URL  | Wait                    | Prepare for Publish     |                                                                                                    |
| Prepare for Publish  | Set Node                         | Prepare metadata for upload and publishing | Get Video               | Upload to Blotato       |                                                                                                    |
| Upload to Blotato    | HTTP Request                    | Upload final video to Blotato media backend | Prepare for Publish     | YT Post                 |                                                                                                    |
| YT Post              | HTTP Request                    | Post video to YouTube Shorts               | Upload to Blotato       | —                       |                                                                                                    |
| Sticky Note1         | Sticky Note                     | Marks “Video Generator” block               | —                      | —                       | # Video Generator                                                                                   |
| Sticky Note2         | Sticky Note                     | Marks “YouTube Post” block                   | —                      | —                       | # Youtube Post                                                                                     |
| Sticky Note          | Sticky Note                     | Workflow overview, problem and solution     | —                      | —                       | ## Automate YT Short Template Note ... (full detailed content on workflow purpose, scope, setup)  |
| Sticky Note3         | Sticky Note                     | Video tutorial link                          | —                      | —                       | @[youtube](yptHq2J8DmI)                                                                            |
| Sticky Note5         | Sticky Note                     | Marks “Script Generator” block               | —                      | —                       | # Script Generator                                                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Set interval to daily at 10:00 AM

2. **Create Brainstorm Idea Node**  
   - Type: Langchain LLM Chat (OpenAI)  
   - Model: GPT-4o  
   - Connect Schedule Trigger → Brainstorm Idea  
   - Configure OpenAI credentials with valid API key

3. **Create AI Agent1 Node**  
   - Type: Langchain Agent  
   - Prompt:  
     - INSTRUCTIONS: Brainstorm 50 “What if history...” video ideas, select one randomly, research facts, write 60-second script with hook, write 2-sentence caption with hashtags (#ai)  
     - Output format: JSON with fields `script`, `caption`, `title`  
   - Connect Brainstorm Idea → AI Agent1  
   - Set OpenAI credentials

4. **Create Structured Output Parser Node**  
   - Type: Langchain Structured Output Parser  
   - JSON Schema: `{ "script": "string", "caption": "string", "title": "string" }`  
   - Connect AI Agent1 → Structured Output Parser

5. **Create Prepare Video Node**  
   - Type: Set Node  
   - Parameters:  
     ```json
     {
       "blotato_api_key": "",
       "template": "empty",
       "voiceId": "elevenlabs/eleven_multilingual_v2/pqHfZKP75CvOlQylNhV4",
       "captionPosition": "bottom",
       "script": {{ structured_output_parser.script.toJsonString() }},
       "style": "cinematic",
       "animate_first_image": true,
       "animate_all": false,
       "text_to_image_model": "replicate/recraft-ai/recraft-v3",
       "image_to_video_model": "fal-ai/framepack"
     }
     ```  
   - Connect Structured Output Parser → Prepare Video

6. **Create Create Video Node**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://backend.blotato.com/v2/videos/creations`  
   - Headers: `blotato-api-key` from Prepare Video node  
   - JSON Body: Includes template, voiceId, captionPosition, script, style, animation flags, and AI models from Prepare Video  
   - Connect Prepare Video → Create Video

7. **Create Wait Node**  
   - Type: Wait Node  
   - Duration: 10 minutes  
   - Connect Create Video → Wait

8. **Create Get Video Node**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://backend.blotato.com/v2/videos/creations/{{ $json.item.id }}` (use ID from Create Video output)  
   - Headers: `blotato-api-key` from Prepare Video node  
   - Connect Wait → Get Video

9. **Create Prepare for Publish Node**  
   - Type: Set Node  
   - Parameters:  
     ```json
     {
       "blotato_api_key": {{ Prepare Video.blotato_api_key }},
       "instagram_id": "",
       "youtube_id": "",  // Fill with your YouTube Account ID
       "tiktok_id": "",
       "facebook_id": "",
       "facebook_page_id": "",
       "threads_id": "",
       "twitter_id": "",
       "linkedin_id": "",
       "pinterest_id": "",
       "pinterest_board_id": "",
       "bluesky_id": "",
       "final_text_long": {{ Prepare Video.script.caption.toJsonString() }},
       "final_text_short": {{ Prepare Video.script.caption.toJsonString() }}
     }
     ```  
   - Connect Get Video → Prepare for Publish

10. **Create Upload to Blotato Node**  
    - Type: HTTP Request  
    - Method: POST  
    - URL: `https://backend.blotato.com/v2/media`  
    - Headers: `blotato-api-key` from Prepare for Publish  
    - Body: JSON with `"url": {{ Get Video.item.mediaUrl }}`  
    - Connect Prepare for Publish → Upload to Blotato

11. **Create YT Post Node**  
    - Type: HTTP Request  
    - Method: POST  
    - URL: `https://backend.blotato.com/v2/posts`  
    - Headers: `blotato-api-key` from Prepare for Publish  
    - Body:  
      ```json
      {
        "post": {
          "accountId": {{ Prepare for Publish.youtube_id }},
          "content": {
            "text": {{ Prepare for Publish.final_text_short }},
            "mediaUrls": [{{ Get Video.item.mediaUrl }}],
            "platform": "youtube"
          },
          "target": {
            "targetType": "youtube",
            "title": {{ AI Agent1.output.title }},
            "privacyStatus": "public",
            "shouldNotifySubscribers": true,
            "isMadeForKids": false
          }
        }
      }
      ```  
    - Connect Upload to Blotato → YT Post

12. **Activate workflow**  
    - Ensure all credentials (OpenAI, Blotato API key) and account IDs (YouTube) are properly set  
    - Save and activate workflow

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Context or Link                                                                                                        |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------|
| ## Automate YT Short Template Note: This workflow automates manual YouTube Shorts creation, including idea brainstorming, script writing, video generation, and publishing. It uses GPT-4o and Blotato API to produce daily viral historical “What If” videos. Setup requires OpenAI and Blotato API keys plus YouTube account ID. Customize video style and voice in the Prepare Video node.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               | Included as a Sticky Note in the workflow overview node.                                                                |
| Video tutorial available: @[youtube](yptHq2J8DmI)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | Sticky Note linked video for further understanding.                                                                     |
| VoiceId used: `elevenlabs/eleven_multilingual_v2/pqHfZKP75CvOlQylNhV4` — a multilingual voice from ElevenLabs for natural narration.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | Prepare Video node voice configuration.                                                                                  |
| Text-to-image model `replicate/recraft-ai/recraft-v3` and image-to-video model `fal-ai/framepack` are used for visual content generation from script text. Adjust these in Prepare Video node for different styles.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | Prepare Video node video style configuration.                                                                            |
| The workflow depends on Blotato’s API for both video generation and media upload/posting. Ensure your API key has sufficient quota and permissions.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | Critical external dependency note.                                                                                       |
| The AI Agent prompt is carefully crafted to use simple language suitable for 6th grade and to avoid greetings to optimize viewer engagement for faceless short videos.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | AI Agent1 node prompt design rationale.                                                                                   |

---

**Disclaimer:** The provided content is exclusively generated from an n8n automation workflow. It complies fully with applicable content policies and contains no illegal, offensive, or protected elements. All data processed is legal and publicly available.