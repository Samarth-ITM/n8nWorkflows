Create & Auto-Post Instagram Reels with AI Clones: Script-to-Post (Heygen + Submagic + Blotato)

https://n8nworkflows.xyz/workflows/create---auto-post-instagram-reels-with-ai-clones--script-to-post--heygen---submagic---blotato--8918


# Create & Auto-Post Instagram Reels with AI Clones: Script-to-Post (Heygen + Submagic + Blotato)

### 1. Workflow Overview

This workflow automates the creation and publication of Instagram Reels using AI-driven content generation and video production services. It is designed for social media creators, agencies, and brands aiming to produce engaging short-form video content on autopilot. The workflow converts a user-provided topic or idea into a fully edited Instagram Reel featuring an AI avatar presenting a scripted message, stylized captions overlaid on video, and a tailored Instagram post caption with hashtags optimized for engagement.

Logical blocks:

- **1.1 Input Reception and Script Generation:** Trigger reception and AI script creation for Instagram Reel.
- **1.2 AI Avatar Video Creation (Heygen):** Generate a video of an AI avatar speaking the script.
- **1.3 Video Captioning (Submagic):** Apply dynamic overlaid captions to the AI avatar video.
- **1.4 Media Upload and Instagram Posting (Blotato):** Upload the captioned video and create the Instagram Reel post with AI-generated captions.
- **1.5 Instagram Caption Generation:** Produce a scroll-stopping Instagram caption with hashtags based on the script.
- **1.6 Control and Polling Logic:** Wait and poll mechanisms for asynchronous video processing steps.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Script Generation

- **Overview:**  
Receives a chat message (topic or idea) and generates a short, engaging Instagram Reel script tailored for a personal finance influencer persona.

- **Nodes Involved:**  
  - When chat message received  
  - Instagram Script Generator  
  - OpenAI Chat Model (linked to Instagram Script Generator)

- **Node Details:**

  - **When chat message received**  
    - Type: Chat trigger node  
    - Role: Entry point for workflow triggered by incoming chat messages containing the topic/idea  
    - Config: Uses webhook ID for incoming connections, no special options enabled  
    - Input: External chat message  
    - Output: Passes message content to Instagram Script Generator  
    - Failure Modes: Webhook disconnection or invalid payload  
    - Sub-workflow: None

  - **Instagram Script Generator**  
    - Type: AI Agent (LangChain agent node)  
    - Role: Transforms the input topic into a 25–30 second Instagram Reel script with hook, body, and close  
    - Config: Custom system prompt specifying style (conversational, authentic, slightly provocative), niche (personal finance), and output format constraints (single paragraph, <1500 characters)  
    - Input: User message from trigger  
    - Output: Script text passed to Heygen video generation  
    - Expressions: Uses LangChain system/user messages for prompt  
    - Failure Modes: API call failure, prompt misformat, rate limits  
    - Version: 2.2  
    - Sub-workflow: None

  - **OpenAI Chat Model**  
    - Type: Language model node (LangChain OpenAI chat)  
    - Role: Provides underlying language processing for Instagram Script Generator agent  
    - Config: Uses default or configured OpenAI-compatible model, no extra options specified  
    - Input: Agent prompts  
    - Output: Text completion for script generation  
    - Failure Modes: Authentication errors, rate limits, network timeouts  
    - Version: 1.2  
    - Sub-workflow: None

---

#### 2.2 AI Avatar Video Creation (Heygen)

- **Overview:**  
Takes the generated script and sends it to Heygen API to create a vertical video of an AI avatar presenting the script in a natural voice.

- **Nodes Involved:**  
  - Post to Heygen  
  - Wait 30 Secs  
  - GET Result  
  - If (status check)  
  - Wait Another 30 Secs (re-poll if not ready)

- **Node Details:**

  - **Post to Heygen**  
    - Type: HTTP Request  
    - Role: Sends a POST request to Heygen video generation API with avatar, voice, and video dimensions  
    - Config:  
      - URL: https://api.heygen.com/v2/video/generate  
      - JSON body includes:  
        - character: avatar type and ID  
        - voice: text input from script, voice ID, speed 1.1  
        - dimension: 720x1280 (vertical video)  
      - Auth: HTTP header with API key (generic credential)  
    - Input: Script text from Instagram Script Generator  
    - Output: Heygen job initiation response (video ID)  
    - Failure Modes: Authentication errors, malformed JSON, API downtime

  - **Wait 30 Secs**  
    - Type: Wait  
    - Role: Pause to allow Heygen video processing  
    - Config: 30 seconds delay  
    - Input: Triggered after Heygen POST  
    - Output: Triggers GET Result node

  - **GET Result**  
    - Type: HTTP Request  
    - Role: Poll Heygen API for video generation status  
    - Config:  
      - URL: https://api.heygen.com/v1/video_status.get  
      - Query parameters: video ID from previous response  
      - Auth: HTTP header with API key  
    - Input: After wait  
    - Output: Status JSON to If node  
    - Failure Modes: Network errors, invalid video ID, auth errors

  - **If**  
    - Type: Conditional node  
    - Role: Checks if Heygen video status is `"completed"`  
    - Config: Compares `$json.data.status` to `"completed"`  
    - Input: Status from GET Result  
    - Output:  
      - True: Proceeds to Post to Submagic  
      - False: Loops back to Wait Another 30 Secs for re-polling

  - **Wait Another 30 Secs**  
    - Type: Wait  
    - Role: Additional wait before re-polling Heygen status  
    - Config: 30 seconds delay  
    - Input: False path from If node  
    - Output: Back to GET Result

---

#### 2.3 Video Captioning (Submagic)

- **Overview:**  
Creates a Submagic project to add stylized captions to the Heygen avatar video and waits for captioning to complete.

- **Nodes Involved:**  
  - Post to Submagic  
  - Wait 15 Secs1  
  - Get Captioned Video from Submagic  
  - If1  
  - Wait 15 Secs

- **Node Details:**

  - **Post to Submagic**  
    - Type: HTTP Request  
    - Role: Creates a captioning project on Submagic with the Heygen video  
    - Config:  
      - URL: https://api.submagic.co/v1/projects  
      - Method: POST  
      - Body parameters: include video details and caption template (not fully detailed in JSON)  
      - Auth: HTTP header with API key  
    - Input: Heygen video ID or URL from previous block  
    - Output: Submagic project ID  
    - Failure Modes: Auth errors, bad body params, API downtime

  - **Wait 15 Secs1**  
    - Type: Wait  
    - Role: Pause to allow Submagic to begin processing  
    - Config: 15 seconds  
    - Input: After POST to Submagic  
    - Output: Triggers Get Captioned Video from Submagic

  - **Get Captioned Video from Submagic**  
    - Type: HTTP Request  
    - Role: Polls Submagic API for captioned video status and download URL  
    - Config:  
      - URL template: https://api.submagic.co/v1/projects/{{ $json.id }}  
      - Auth: HTTP header with API key  
    - Input: After wait or re-poll  
    - Output: Project status and video URL to If1 node  
    - Failure Modes: Network errors, invalid project ID, auth errors

  - **If1**  
    - Type: Conditional node  
    - Role: Checks if Submagic project status is `"completed"`  
    - Config: Compares `$json.status` to `"completed"`  
    - Input: Submagic status JSON  
    - Output:  
      - True: Passes video download URL to Upload media node  
      - False: Loops back to Wait 15 Secs for re-polling

  - **Wait 15 Secs**  
    - Type: Wait  
    - Role: Additional wait before re-polling Submagic status  
    - Config: 15 seconds  
    - Input: If1 false branch  
    - Output: Back to Get Captioned Video from Submagic

---

#### 2.4 Media Upload and Instagram Posting (Blotato)

- **Overview:**  
Uploads the finalized captioned video to Blotato and creates the Instagram Reel post with the AI-generated caption and media.

- **Nodes Involved:**  
  - Upload media  
  - Instagram Caption Agent  
  - Create post

- **Node Details:**

  - **Upload media**  
    - Type: Blotato node (media resource)  
    - Role: Uploads the captioned video URL to Blotato cloud storage  
    - Config: Reads mediaUrl from Submagic downloadUrl field  
    - Input: Submagic completed project with download URL  
    - Output: Returns uploaded media URL for post creation  
    - Failure Modes: API auth errors, upload failures, invalid URLs

  - **Instagram Caption Agent**  
    - Type: OpenAI node (LangChain OpenAI)  
    - Role: Generates an engaging Instagram caption with hooks, CTAs, and hashtags based on the original Reel script  
    - Config:  
      - System message defines style: conversational, authentic, provocative, personal finance hashtags, 80–150 words, includes emojis and CTAs  
      - Inputs original script from Instagram Script Generator  
      - Output: Caption text for Instagram post  
    - Input: After media upload  
    - Output: Caption text to Create post node  
    - Failure Modes: API errors, prompt misformat, rate limits

  - **Create post**  
    - Type: Blotato node (Instagram reel post creation)  
    - Role: Publishes the Reel to Instagram using Blotato integration  
    - Config:  
      - AccountId selected from credential list  
      - Media type: reel  
      - Post text: from Instagram Caption Agent output  
      - Media URLs: from Upload media output  
    - Input: Caption text and media URL  
    - Output: Instagram post creation response  
    - Failure Modes: Auth errors, invalid media URLs, API limits

---

#### 2.5 Control and Polling Logic

- **Overview:**  
Manages asynchronous API processing delays by waiting and polling until Heygen and Submagic processing completes before advancing.

- **Nodes Involved:**  
  - Wait 30 Secs  
  - Wait Another 30 Secs  
  - Wait 15 Secs  
  - Wait 15 Secs1  
  - If (Heygen status)  
  - If1 (Submagic status)

- **Node Details:**  
  - Wait nodes configured with fixed durations (15 or 30 seconds)  
  - If nodes check for `"completed"` status strings to gate progress  
  - Polling loops ensure downstream nodes process only when media is ready  
  - Potential Failures: Infinite loops if APIs never return completed status; no explicit timeout or error escalation present

---

### 3. Summary Table

| Node Name                  | Node Type                        | Functional Role                                   | Input Node(s)               | Output Node(s)            | Sticky Note                                                                 |
|----------------------------|---------------------------------|-------------------------------------------------|-----------------------------|---------------------------|----------------------------------------------------------------------------|
| When chat message received  | LangChain Chat Trigger          | Entry point, receives topic/idea for script     |                             | Instagram Script Generator | Trigger and Script                                                          |
| Instagram Script Generator  | LangChain Agent                 | Generates Instagram Reel script from topic      | When chat message received  | Post to Heygen             | Create IG Reel with Personal Avatar                                        |
| OpenAI Chat Model           | LangChain OpenAI Chat           | Underlying LM for Instagram Script Generator    | Instagram Script Generator  | Instagram Script Generator |                                                                            |
| Post to Heygen              | HTTP Request                   | Sends script to Heygen to generate AI avatar video | Instagram Script Generator  | Wait 30 Secs               | Create IG Reel with Personal Avatar                                        |
| Wait 30 Secs               | Wait                           | Wait for Heygen video processing                 | Post to Heygen              | GET Result                 | Create IG Reel with Personal Avatar                                        |
| GET Result                 | HTTP Request                   | Poll Heygen video generation status              | Wait 30 Secs                | If                        | Create IG Reel with Personal Avatar                                        |
| If                         | If                             | Check if Heygen video is completed               | GET Result                  | Post to Submagic, Wait Another 30 Secs | Create IG Reel with Personal Avatar                                        |
| Wait Another 30 Secs       | Wait                           | Additional wait before re-polling Heygen         | If (false)                  | GET Result                 | Create IG Reel with Personal Avatar                                        |
| Post to Submagic           | HTTP Request                   | Create Submagic captioning project                | If (true)                   | Wait 15 Secs1              | Text Overlay Agent                                                         |
| Wait 15 Secs1              | Wait                           | Wait for Submagic captioning to start             | Post to Submagic            | Get Captioned Video from Submagic | Text Overlay Agent                                                         |
| Get Captioned Video from Submagic | HTTP Request             | Poll Submagic for captioned video status          | Wait 15 Secs1, Wait 15 Secs | If1                       | Text Overlay Agent                                                         |
| If1                        | If                             | Check if Submagic video processing completed      | Get Captioned Video from Submagic | Upload media, Wait 15 Secs | Text Overlay Agent                                                         |
| Wait 15 Secs               | Wait                           | Additional wait before re-polling Submagic status | If1 (false)                 | Get Captioned Video from Submagic | Text Overlay Agent                                                         |
| Upload media               | Blotato                        | Uploads captioned video to Blotato                | If1 (true)                  | Instagram Caption Agent    | Post to Instagram                                                          |
| Instagram Caption Agent    | LangChain OpenAI               | Generates Instagram caption with hashtags         | Upload media                | Create post                | Post to Instagram                                                          |
| Create post                | Blotato                        | Creates Instagram Reel post with media and caption | Instagram Caption Agent     |                           | Post to Instagram                                                          |
| Sticky Note                | Sticky Note                   | Text Overlay Agent description                     |                             |                           | Text Overlay Agent                                                         |
| Sticky Note1               | Sticky Note                   | Post to Instagram description                       |                             |                           | Post to Instagram                                                          |
| Sticky Note2               | Sticky Note                   | Create IG Reel with Personal Avatar description    |                             |                           | Create IG Reel with Personal Avatar                                        |
| Sticky Note3               | Sticky Note                   | Trigger and Script description                      |                             |                           | Trigger and Script                                                          |
| Sticky Note4               | Sticky Note                   | Full workflow description and resources            |                             |                           | See section 5 below                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node:**  
   - Add a **LangChain Chat Trigger** node named `When chat message received`.  
   - Configure webhook for external chat input.

2. **Add Instagram Script Generator:**  
   - Add a **LangChain Agent** node named `Instagram Script Generator`.  
   - Connect input from `When chat message received`.  
   - Configure the system message with the detailed prompt specifying:  
     - Instagram Reel script generation  
     - Personal finance niche and podcast-style influencer tone  
     - Script structure: hook, body, CTA  
     - Output constraints: <1500 characters, one paragraph  
   - Ensure it uses the OpenAI Chat Model (LangChain OpenAI Chat) node for language generation.

3. **Add OpenAI Chat Model Node:**  
   - Add **LangChain OpenAI Chat** node named `OpenAI Chat Model`.  
   - Connect it as the language model for `Instagram Script Generator`.  
   - Set model credentials (OpenAI or compatible).  
   - Use default model or specify (e.g., GPT-4).

4. **Add Heygen Video Generation:**  
   - Add **HTTP Request** node named `Post to Heygen`.  
   - Connect from `Instagram Script Generator`.  
   - Configure POST request to `https://api.heygen.com/v2/video/generate`.  
   - Set JSON body: include avatar ID (`56b7bf60959448f789bff62bd7b5ef48`), voice ID (`0fcabc9607434b6385b4f5c6810c5b7a`), input_text from script output, video dimension 720x1280.  
   - Configure authentication with Heygen API key via HTTP Header auth credentials.

5. **Add Wait 30 Seconds:**  
   - Add **Wait** node `Wait 30 Secs`.  
   - Connect from `Post to Heygen`.  
   - Set wait time to 30 seconds.

6. **Add Heygen Status Polling:**  
   - Add **HTTP Request** node `GET Result`.  
   - Connect from `Wait 30 Secs`.  
   - Configure GET request to `https://api.heygen.com/v1/video_status.get`.  
   - Add query parameters for video ID from previous response.  
   - Use same Heygen HTTP header auth credentials.

7. **Add Heygen Completion Check:**  
   - Add **If** node named `If`.  
   - Connect from `GET Result`.  
   - Condition: `$json.data.status` equals `completed`.  
   - True path proceeds to Submagic, false path leads to another wait.

8. **Add Wait Another 30 Seconds:**  
   - Add **Wait** node `Wait Another 30 Secs`.  
   - Connect from `If` false path.  
   - Set wait time 30 seconds.  
   - Connect output back to `GET Result` for polling loop.

9. **Add Submagic Captioning:**  
   - Add **HTTP Request** node `Post to Submagic`.  
   - Connect from `If` true path.  
   - Configure POST request to `https://api.submagic.co/v1/projects`.  
   - Provide video info and caption template parameters as required by Submagic API.  
   - Use HTTP Header auth credentials for Submagic API.

10. **Add Wait 15 Seconds1:**  
    - Add **Wait** node `Wait 15 Secs1`.  
    - Connect from `Post to Submagic`.  
    - Set wait time 15 seconds.

11. **Add Get Captioned Video:**  
    - Add **HTTP Request** node `Get Captioned Video from Submagic`.  
    - Connect from `Wait 15 Secs1` and also from a secondary wait node (see step 14).  
    - Configure GET request to `https://api.submagic.co/v1/projects/{{ $json.id }}`.  
    - Use Submagic HTTP header auth.

12. **Add Submagic Completion Check:**  
    - Add **If** node `If1`.  
    - Connect from `Get Captioned Video from Submagic`.  
    - Condition: `$json.status` equals `completed`.  
    - True path proceeds to upload media, false path waits.

13. **Add Wait 15 Seconds:**  
    - Add **Wait** node `Wait 15 Secs`.  
    - Connect from `If1` false path.  
    - Set wait time 15 seconds.  
    - Connect output back to `Get Captioned Video from Submagic` for polling loop.

14. **Add Upload Media to Blotato:**  
    - Add **Blotato** node `Upload media`.  
    - Connect from `If1` true path.  
    - Configure to upload media from Submagic's download URL.  
    - Use Blotato credentials.

15. **Add Instagram Caption Agent:**  
    - Add **LangChain OpenAI** node `Instagram Caption Agent`.  
    - Connect from `Upload media`.  
    - Configure system prompt to generate engaging Instagram captions optimized for personal finance niche including hashtags and CTAs.  
    - Input original script from `Instagram Script Generator` node.

16. **Add Create Post on Instagram (Blotato):**  
    - Add **Blotato** node `Create post`.  
    - Connect from `Instagram Caption Agent`.  
    - Set Instagram media type to "reel".  
    - Use Blotato account ID.  
    - Post content text from `Instagram Caption Agent` output.  
    - Media URLs from `Upload media` output.

17. **Add Sticky Notes (Optional):**  
    - Add sticky notes for documentation and clarity:  
      - "Trigger and Script" near trigger and script nodes  
      - "Create IG Reel with Personal Avatar" near Heygen nodes  
      - "Text Overlay Agent" near Submagic nodes  
      - "Post to Instagram" near Blotato nodes  
      - Full workflow description with usage instructions

18. **Credentials Setup:**  
    - Create and link credentials for:  
      - OpenAI (or compatible) for LangChain nodes  
      - Heygen API key (HTTP Header Auth)  
      - Submagic API key (HTTP Header Auth)  
      - Blotato account and token

19. **Testing:**  
    - Trigger workflow with sample topics (e.g., "3 money habits for 2025").  
    - Confirm successful script generation, video creation, captioning, upload, and Instagram posting.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | Context or Link                                                                                      |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| AI Clone Instagram Reel Builder + Auto-Post (Heygen + Submagic + Blotato + n8n) template automates end-to-end Reel creation from idea to Instagram posting. Ideal for creators, agencies, and brands seeking consistent, scalable short-form content without manual editing.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | Full workflow description from Sticky Note4 (positioned top-left in workflow canvas)              |
| Watch step-by-step automation builds on YouTube: https://youtu.be/MmZxLuAkqig?si=DRfS89yQlSlbMbfZ                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | YouTube video link for visual guidance                                                             |
| Best practice: Store all secret keys and tokens in n8n Credentials manager instead of hardcoding in nodes. This improves security and maintainability.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | Security recommendation                                                                            |
| The workflow uses wait and polling loops with conditional checks to handle asynchronous video processing by Heygen and Submagic. Production usage should add error handling (e.g., retries, alert notifications via Slack or email) and safeguard against infinite loops.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | Operational notes                                                                                   |
| Customization tips include adjusting the LangChain prompt personas for script tone, swapping Heygen avatar or voice IDs, changing video resolution in Heygen request, selecting different Submagic caption templates, and extending Blotato posting targets beyond Instagram (TikTok, YouTube Shorts).                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | Workflow flexibility and customization ideas                                                     |

---

**Disclaimer:** The provided text is generated exclusively from an automated n8n workflow. It complies fully with content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.