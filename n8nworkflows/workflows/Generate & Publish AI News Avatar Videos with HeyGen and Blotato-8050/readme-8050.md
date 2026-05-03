Generate & Publish AI News Avatar Videos with HeyGen and Blotato

https://n8nworkflows.xyz/workflows/generate---publish-ai-news-avatar-videos-with-heygen-and-blotato-8050


# Generate & Publish AI News Avatar Videos with HeyGen and Blotato

### 1. Workflow Overview

This workflow automates the creation and publishing of vertical AI avatar videos featuring trending AI and LLM news. It targets content creators, marketers, and social media managers who want to streamline the production of short-form video content for platforms like TikTok, Instagram, and YouTube Shorts. The workflow logically divides into the following blocks:

- **1.1 Scheduling & News Collection:** Triggers the workflow daily and fetches recent AI/LLM news from RSS feeds.
- **1.2 AI Script Generation & Text Content Creation:** Uses AI to select the most viral news story, write a video script, generate a catchy title, and create both short and long social media captions.
- **1.3 Avatar Video Generation:** Configures HeyGen API parameters, submits the AI-generated script for avatar video creation, and waits for video processing to complete.
- **1.4 Publishing Preparation & Distribution:** Prepares necessary metadata and authentication for Blotato API, uploads the video, and optionally publishes to multiple social platforms.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduling & News Collection

**Overview:**  
This block runs the workflow on a fixed daily schedule and collects fresh AI/LLM news from multiple RSS feeds for analysis.

**Nodes Involved:**  
- Schedule Trigger1  
- Read AI News Feed  
- Read AI News Feed1

**Node Details:**

- **Schedule Trigger1**  
  - Type: Schedule Trigger  
  - Role: Starts workflow daily at 10:00 AM (configurable trigger time)  
  - Configuration: Interval set to trigger once daily at hour 10  
  - Inputs: None (trigger node)  
  - Outputs: Connected to AI Agent1  
  - Edge Cases: Misconfiguration may cause no trigger; time zone considerations may affect execution time.

- **Read AI News Feed** and **Read AI News Feed1**  
  - Type: RSS Feed Read Tool  
  - Role: Fetch news items from configured RSS feeds (multiple sources implied)  
  - Configuration: Uses default RSS feed reading options; feeds themselves are configured externally or in node parameters (not explicitly shown)  
  - Inputs: None  
  - Outputs: Feed news items sent to AI Agent1 (as ai_tool inputs)  
  - Edge Cases: RSS feed availability issues, network timeouts, or format changes may cause failures or empty data.

---

#### 2.2 AI Script Generation & Text Content Creation

**Overview:**  
Processes collected news, selects the most viral story, generates a 30-second video script in Russian, and creates supporting text content including a video title, a short caption, and a long caption with hashtags.

**Nodes Involved:**  
- AI Agent1  
- Write Long Caption1  
- Write Title1  
- Write Short Caption1  
- Write Script1 (connected as ai_languageModel input to AI Agent1)

**Node Details:**

- **AI Agent1**  
  - Type: LangChain Agent  
  - Role: Orchestrates multi-step AI prompt workflow; fetches top stories, selects viral news, fetches Hacker News comments, and drafts a detailed 30-second script in Russian with emotional hooks  
  - Configuration: Custom multi-step instruction prompt designed for Russian language output and specific script style  
  - Inputs: Trigger from Schedule Trigger1; AI tools inputs from RSS feeds  
  - Outputs: The finalized video script sent to Write Script1  
  - Edge Cases: AI prompt failures, insufficient or irrelevant news input, or API rate limits.

- **Write Script1**  
  - Type: LangChain LM ChatOpenAI  
  - Role: Completes the AI Agent’s scripted instructions to generate the video script text  
  - Configuration: Uses a smaller OpenAI model ("o3-mini") for resource efficiency  
  - Inputs: From AI Agent1  
  - Outputs: Script content forwarded to downstream nodes  
  - Edge Cases: Model response delays, partial outputs, or token limits.

- **Write Long Caption1**  
  - Type: LangChain OpenAI  
  - Role: Generates a 50-word social media caption in Russian with specific stylistic constraints and hashtags based on the video script content  
  - Configuration: Uses "gpt-4o-mini" model; detailed instructions for writing style, hashtag rules, and structure  
  - Inputs: AI Agent1 output (script text)  
  - Outputs: Long caption text sent to Write Title1  
  - Edge Cases: Style enforcement failures, output truncation.

- **Write Title1**  
  - Type: LangChain OpenAI  
  - Role: Creates a concise 2-3 word Russian title reflecting the video’s main idea  
  - Configuration: Uses "gpt-4o-mini"; input is the long caption content  
  - Inputs: From Write Long Caption1  
  - Outputs: Title forwarded to Write Short Caption1  
  - Edge Cases: Overly generic or irrelevant titles.

- **Write Short Caption1**  
  - Type: LangChain OpenAI  
  - Role: Produces a 2-sentence English summary caption of the video text at a 6th-grade reading level without emojis or personal names  
  - Configuration: Uses "gpt-4o-mini"; input is the long caption content  
  - Inputs: From Write Title1  
  - Outputs: Short caption passed to Setup Heygen1 node  
  - Edge Cases: Language style mismatch, loss of key information.

---

#### 2.3 Avatar Video Generation

**Overview:**  
Configures HeyGen API parameters, sends the video generation request with the AI script and metadata, then waits for processing to complete.

**Nodes Involved:**  
- Setup Heygen1  
- Create Avatar Video1  
- Wait6  
- Get Avatar Video1

**Node Details:**

- **Setup Heygen1**  
  - Type: Set  
  - Role: Holds user-configurable HeyGen API credentials and video options including `heygen_api_key`, `avatar_id`, `voice_id`, and optional `background_video_url`  
  - Configuration: Raw JSON with placeholders for sensitive data and default background video URL  
  - Inputs: From Write Short Caption1  
  - Outputs: Config data to Create Avatar Video1  
  - Edge Cases: Missing or invalid API keys or IDs cause authentication errors.

- **Create Avatar Video1**  
  - Type: HTTP Request  
  - Role: Sends POST request to HeyGen video generation API endpoint with video parameters including avatar, voice, background, script text, and title  
  - Configuration: Uses dynamic expressions to insert credentials and script text; generates vertical 9:16 video (720x1280)  
  - Inputs: From Setup Heygen1  
  - Outputs: HeyGen API response containing video ID for status check  
  - Edge Cases: API timeouts, invalid parameters, rate limits, or malformed requests.

- **Wait6**  
  - Type: Wait  
  - Role: Pauses workflow for 8 minutes to allow HeyGen to process the video  
  - Configuration: Fixed 8-minute wait period; no output data produced  
  - Inputs: From Create Avatar Video1  
  - Outputs: Trigger next node after delay  
  - Edge Cases: Insufficient wait time may cause premature status checks.

- **Get Avatar Video1**  
  - Type: HTTP Request  
  - Role: Polls HeyGen video status endpoint using video ID to retrieve the final video URL once processing completes  
  - Configuration: GET request with query parameter `video_id`; authenticates using `X-Api-Key` header from Setup Heygen1  
  - Inputs: From Wait6  
  - Outputs: Video data including downloadable `video_url`  
  - Edge Cases: Video still processing, 401/403 auth errors, network issues.

---

#### 2.4 Publishing Preparation & Distribution

**Overview:**  
Prepares credentials and platform IDs for Blotato API, uploads the generated video, and optionally publishes it to multiple social media platforms.

**Nodes Involved:**  
- Prepare for Publish1  
- Upload to Blotato1  
- [TikTOK] Publish via Blotato1 … [TikTOK] Publish via Blotato7 (disabled by default)

**Node Details:**

- **Prepare for Publish1**  
  - Type: Set  
  - Role: Holds Blotato API key and target platform IDs along with the final long and short captions extracted from previous nodes  
  - Configuration: JSON with placeholders for API key and platform/page/account IDs; dynamically inserts captions from Write Long Caption1 and Write Short Caption1  
  - Inputs: From Get Avatar Video1  
  - Outputs: Credentials and metadata to Upload to Blotato1  
  - Edge Cases: Missing API key or platform IDs result in failed uploads.

- **Upload to Blotato1**  
  - Type: HTTP Request  
  - Role: Uploads the generated video URL to Blotato media endpoint with authentication  
  - Configuration: POST request with JSON body containing video URL from Get Avatar Video1; Header includes Blotato API key from Prepare for Publish1  
  - Inputs: From Prepare for Publish1  
  - Outputs: Response forwarded to optional publish nodes  
  - Edge Cases: Upload failures, invalid URL, auth errors.

- **[TikTOK] Publish via Blotato1 … [TikTOK] Publish via Blotato7**  
  - Type: HTTP Request  
  - Role: Optional nodes (disabled by default) that post the video and captions to various social media platforms through Blotato API  
  - Configuration: Each posts to Blotato’s posts endpoint with platform-specific target info and media URL; all use Blotato API key from Prepare for Publish1  
  - Inputs: From Upload to Blotato1  
  - Outputs: None (end nodes)  
  - Edge Cases: Disabled by default; enabling requires correct platform IDs; potential API errors or rate limits.

---

### 3. Summary Table

| Node Name                | Node Type                             | Functional Role                                  | Input Node(s)                 | Output Node(s)                       | Sticky Note                                                                                                  |
|--------------------------|-------------------------------------|-------------------------------------------------|------------------------------|------------------------------------|--------------------------------------------------------------------------------------------------------------|
| Sticky • README (EN)      | Sticky Note                         | Workflow overview and instructions               | None                         | None                               | Describes full workflow purpose, usage, setup, and customization.                                           |
| Sticky • Setup (EN)       | Sticky Note                         | Credential and environment variable setup tips   | None                         | None                               | Lists credentials to create and environment variables for ease of use.                                      |
| Sticky • Troubleshooting (EN) | Sticky Note                     | Common issues and fixes                           | None                         | None                               | Covers common errors like authentication and processing delays.                                             |
| Schedule Trigger1         | Schedule Trigger                   | Daily trigger at 10:00 AM                         | None                         | AI Agent1                         |                                                                                                              |
| Read AI News Feed         | RSS Feed Read Tool                 | Fetches AI/LLM news from RSS feeds                | None                         | AI Agent1 (ai_tool input)           |                                                                                                              |
| Read AI News Feed1        | RSS Feed Read Tool                 | Additional news feed reader                        | None                         | AI Agent1 (ai_tool input)           |                                                                                                              |
| AI Agent1                | LangChain Agent                    | Selects viral story and drafts 30s video script  | Schedule Trigger1, Read AI News Feed(s) | Write Long Caption1, Write Script1 (ai_languageModel) |                                                                                                              |
| Write Script1            | LangChain LM ChatOpenAI            | Generates the final video script in Russian       | AI Agent1                    | AI Agent1 (ai_languageModel output) |                                                                                                              |
| Write Long Caption1      | LangChain OpenAI                   | Creates a long Russian caption with hashtags      | AI Agent1                    | Write Title1                      |                                                                                                              |
| Write Title1             | LangChain OpenAI                   | Generates a short Russian video title             | Write Long Caption1          | Write Short Caption1               |                                                                                                              |
| Write Short Caption1     | LangChain OpenAI                   | Creates a short English caption summary            | Write Title1                 | Setup Heygen1                     |                                                                                                              |
| Setup Heygen1            | Set                               | Stores HeyGen API credentials and video settings  | Write Short Caption1          | Create Avatar Video1              | Sticky • Fill: Setup Heygen (EN) — Fill `heygen_api_key`, `avatar_id`, `voice_id`, optional background video. |
| Create Avatar Video1     | HTTP Request                      | Sends video generation request to HeyGen API      | Setup Heygen1                 | Wait6                            |                                                                                                              |
| Wait6                    | Wait                             | Pauses workflow 8 minutes for video processing    | Create Avatar Video1          | Get Avatar Video1                 | Sticky • Troubleshooting (EN) — Increase wait time if processing status stuck.                               |
| Get Avatar Video1        | HTTP Request                      | Checks HeyGen for video processing status and URL | Wait6                        | Prepare for Publish1              |                                                                                                              |
| Prepare for Publish1     | Set                               | Stores Blotato API key, platform IDs, and captions | Get Avatar Video1             | Upload to Blotato1                | Sticky • Fill: Prepare for Publish (EN) — Fill Blotato key and platform IDs.                                |
| Upload to Blotato1       | HTTP Request                      | Uploads video URL to Blotato media endpoint        | Prepare for Publish1          | [TikTOK] Publish via Blotato*     |                                                                                                              |
| [TikTOK] Publish via Blotato1 - [TikTOK] Publish via Blotato7 | HTTP Request (disabled) | Optional posts to various social platforms via Blotato | Upload to Blotato1            | None                            | Disabled by default; enable only needed platforms.                                                          |

*Multiple nearly identical publish nodes for different social platforms, all disabled by default.

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Set to trigger daily at 10:00 AM (adjust as needed).

2. **Add Two RSS Feed Read Tool Nodes**  
   - Configure each to read from distinct AI/LLM news RSS feeds.  
   - No special parameters beyond feed URLs.

3. **Add a LangChain Agent Node (AI Agent1)**  
   - Configure prompt with detailed multi-step instructions to:  
     - Fetch and analyze top 10 AI/LLM news stories  
     - Choose the most viral  
     - Fetch Hacker News comments  
     - Write a 30-second Russian video script with emotional hooks  
   - Connect Schedule Trigger and both RSS nodes as inputs.

4. **Add a LangChain LM ChatOpenAI Node (Write Script1)**  
   - Use model "o3-mini"  
   - Connected as ai_languageModel input to AI Agent1.

5. **Add LangChain OpenAI Node for Long Caption (Write Long Caption1)**  
   - Use model "gpt-4o-mini"  
   - Configure prompt for a 50-word Russian caption with specific style and hashtag rules.  
   - Input from AI Agent1 output.

6. **Add LangChain OpenAI Node for Title (Write Title1)**  
   - Use model "gpt-4o-mini"  
   - Prompt to create 2-3 word Russian video title from long caption.  
   - Input from Write Long Caption1.

7. **Add LangChain OpenAI Node for Short Caption (Write Short Caption1)**  
   - Use model "gpt-4o-mini"  
   - Prompt to write a brief 2-sentence English summary without emojis or personal names.  
   - Input from Write Title1.

8. **Add Set Node for HeyGen Setup (Setup Heygen1)**  
   - Set JSON with keys: `heygen_api_key`, `avatar_id`, `voice_id`, and optional `background_video_url` (default example URL).  
   - Input from Write Short Caption1.

9. **Add HTTP Request Node to Create Avatar Video (Create Avatar Video1)**  
   - POST to `https://api.heygen.com/v2/video/generate`  
   - Body includes dynamic insertion of avatar_id, voice_id, background_video_url, script text (from AI Agent1 output), and title (from Write Title1 output)  
   - Headers include `X-Api-Key` from Setup Heygen1.  
   - Input from Setup Heygen1.

10. **Add Wait Node (Wait6)**  
    - Wait for 8 minutes to allow video processing.  
    - Input from Create Avatar Video1.

11. **Add HTTP Request Node to Get Avatar Video Status (Get Avatar Video1)**  
    - GET `https://api.heygen.com/v1/video_status.get` with query param `video_id` from Create Avatar Video1 response.  
    - Header `X-Api-Key` from Setup Heygen1.  
    - Input from Wait6.

12. **Add Set Node for Blotato Publish Preparation (Prepare for Publish1)**  
    - Set JSON with `blotato_api_key`, platform IDs (Instagram, TikTok, YouTube, Facebook, etc.), and captions from Write Long Caption1 and Write Short Caption1.  
    - Input from Get Avatar Video1.

13. **Add HTTP Request Node to Upload Video to Blotato (Upload to Blotato1)**  
    - POST to `https://backend.blotato.com/v2/media` with video URL from Get Avatar Video1.  
    - Header `blotato-api-key` from Prepare for Publish1.  
    - Input from Prepare for Publish1.

14. **Add Optional HTTP Request Nodes for Publishing to Social Platforms (e.g., [TikTOK] Publish via Blotato1, etc.)**  
    - POST to `https://backend.blotato.com/v2/posts`  
    - Body includes post content, platform target type, page/account IDs, text captions, and media URLs.  
    - Header includes `blotato-api-key` from Prepare for Publish1.  
    - Connect all from Upload to Blotato1.  
    - Disabled by default; enable and configure platform IDs as needed.

15. **Credentials Setup**  
    - Create credentials in n8n for:  
      - HeyGen API Key (used in HTTP headers)  
      - Blotato API Key (used in HTTP headers)  
    - Do not hardcode keys; use credentials or environment variables where possible.

16. **Adjust Schedule and Prompts**  
    - Modify Schedule Trigger timing as desired.  
    - Edit AI Agent prompt for different topics, languages, or tone.  
    - Update HeyGen avatar, voice, and background video URLs in Setup Heygen1.  
    - Provide your Blotato platform IDs in Prepare for Publish1.

---

### 5. General Notes & Resources

| Note Content                                                                                                                      | Context or Link                                                                                   |
|-----------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This workflow acts as a ‘newsroom in a box’ automating content creation from news curation to video publishing on social media.   | Sticky • README (EN) node content                                                                |
| For better reliability, increase the Wait node duration if HeyGen video status remains “processing” beyond expected time.        | Sticky • Troubleshooting (EN) node content                                                       |
| Credentials must be securely managed; avoid hardcoding API keys in nodes. Use n8n credentials or environment variables instead.  | Sticky • Setup (EN) node content                                                                 |
| To change the video topic or language, adjust the AI Agent prompt instructions accordingly.                                       | Sticky • README (EN) node content                                                                |
| Publishing nodes are disabled by default to prevent accidental posts; enable only those platforms you use and configure IDs.      | Sticky • Troubleshooting (EN) and README nodes                                                   |
| HeyGen API and Blotato API documentation should be consulted for detailed parameter options and rate limits.                     | User should refer to official HeyGen and Blotato API docs                                        |

---

**Disclaimer:**  
The provided text and workflow originate exclusively from an automated n8n workflow built with legal and public data sources. All integrations comply with content policies and do not contain illegal or offensive material.