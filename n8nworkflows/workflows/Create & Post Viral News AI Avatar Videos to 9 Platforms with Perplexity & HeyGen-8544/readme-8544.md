Create & Post Viral News AI Avatar Videos to 9 Platforms with Perplexity & HeyGen

https://n8nworkflows.xyz/workflows/create---post-viral-news-ai-avatar-videos-to-9-platforms-with-perplexity---heygen-8544


# Create & Post Viral News AI Avatar Videos to 9 Platforms with Perplexity & HeyGen

### 1. Workflow Overview

This workflow automates the creation and posting of viral AI avatar news videos to nine social media platforms using AI research and content generation tools, combined with avatar video synthesis and multi-platform publishing. It is designed for content creators, marketers, and social media managers who want to generate engaging, data-driven video content automatically daily without manual filming or editing.

The workflow is organized into the following logical blocks:

- **1.1 Scheduled Trigger:** Initiates the workflow daily at 10 AM.
- **1.2 AI Research Block:** Uses Perplexity AI to research the top 10 trending news stories in a specified industry, selects the most viral story, and compiles a detailed factual report.
- **1.3 AI Content Generation:** Uses OpenAI (ChatGPT-5) to write a conversational AI avatar video script, SEO-optimized caption, and a short viral title based on the research.
- **1.4 HeyGen Video Creation:** Calls HeyGen’s API to create an AI avatar video either with or without a background video, according to configuration.
- **1.5 Video Processing and Retrieval:** Waits for HeyGen to complete video processing, then fetches the finished video.
- **1.6 Media Upload and Multi-Platform Posting:** Uploads the video to Blotato, then posts it to nine social media platforms (TikTok, LinkedIn, Facebook, Instagram, Twitter, YouTube, Threads, Bluesky, Pinterest) using the Blotato API.
- **1.7 Error Handling:** Captures errors from social media posting nodes for debugging.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger  
- **Overview:** Triggers the entire workflow once daily at 10 AM to start the automated process.  
- **Nodes Involved:**  
  - Schedule Trigger  
- **Node Details:**  
  - **Schedule Trigger**  
    - Type: Schedule trigger node  
    - Configured to trigger daily at hour 10 (10 AM)  
    - Outputs: Starts the workflow by triggering the AI Research - Top 10 node  
    - Edge cases: If node is disabled or system time zone misconfigured, trigger may fail or run at incorrect time.

#### 1.2 AI Research Block  
- **Overview:** Researches the top 10 trending news stories in the specified industry (default: real estate), selects the most viral story, and produces a detailed factual report.  
- **Nodes Involved:**  
  - AI Research - Top 10 (Perplexity)  
  - AI Research - Report (Perplexity)  
- **Node Details:**  
  - **AI Research - Top 10**  
    - Type: Perplexity node (model: sonar-pro)  
    - Task: Fetches top 10 trending news in past 24 hours for the configured industry (default "real estate")  
    - Input: Trigger from Schedule Trigger  
    - Output: JSON list of 10 news stories  
    - Credentials: Perplexity API key required  
    - Edge cases: API quota exceeded, no trending news found, network errors  
  - **AI Research - Report**  
    - Type: Perplexity node (model: sonar-pro)  
    - Task: Selects the single most viral story from the top 10, researches it in detail, outputs a factual report with statistics and social media appeal rationale  
    - Input: Output from AI Research - Top 10  
    - Output: Detailed news report (string)  
    - Credentials: Perplexity API key required  
    - Edge cases: Selection ambiguity, API errors, incomplete data  

#### 1.3 AI Content Generation  
- **Overview:** Generates a conversational video script, SEO-optimized caption, and short viral title for the AI avatar video using OpenAI GPT-5 based on the selected news report.  
- **Nodes Involved:**  
  - AI Writer (OpenAI)  
- **Node Details:**  
  - **AI Writer**  
    - Type: OpenAI API node (model GPT-5)  
    - Task: Receives detailed news report, writes:  
      1. ~30-second conversational monologue script for AI avatar video  
      2. SEO-optimized caption with hashtags  
      3. Short viral title (max 8 words, no emojis)  
    - Input: AI Research - Report output  
    - Output: JSON object with keys: `script`, `caption`, `title`  
    - Credentials: OpenAI API key required  
    - Expressions: Uses JSONPath expressions to inject news report and output JSON  
    - Edge cases: API rate limits, malformed JSON output, unexpected content generation  

#### 1.4 HeyGen Video Creation  
- **Overview:** Creates an AI avatar video using HeyGen API either with or without a background video depending on configuration.  
- **Nodes Involved:**  
  - Setup Heygen (Set node)  
  - If (conditional node)  
  - Create Avatar Video WITH Background Video (HTTP Request)  
  - Create Avatar Video WITHOUT Background Video (HTTP Request)  
  - Merge (Merge node)  
- **Node Details:**  
  - **Setup Heygen**  
    - Type: Set node  
    - Task: Stores HeyGen API key, avatar ID, voice ID, and background video URL configuration  
    - Output: JSON with HeyGen credentials and flags  
  - **If**  
    - Type: Conditional node  
    - Checks if `has_background_video` is true to branch video creation logic  
  - **Create Avatar Video WITH Background Video**  
    - Type: HTTP Request  
    - Calls HeyGen `/v2/video/generate` API endpoint with avatar, voice, and background video parameters  
    - Uses expressions to inject script text, avatar ID, voice ID, and background video URL  
    - Headers include HeyGen API key  
    - Output: Video generation response with video_id  
  - **Create Avatar Video WITHOUT Background Video**  
    - Type: HTTP Request  
    - Similar to above but without background video parameters  
  - **Merge**  
    - Type: Merge node  
    - Combines output from either video creation path for downstream processing  
  - Edge cases: Invalid HeyGen API key, avatar or voice ID errors, background video URL invalid, network timeouts  

#### 1.5 Video Processing and Retrieval  
- **Overview:** Waits for HeyGen to process the video, polls for video status until completion, then retrieves the final video URL.  
- **Nodes Involved:**  
  - Wait (1 minute)  
  - Get Avatar Video (HTTP Request)  
  - If Video Done (Conditional node)  
- **Node Details:**  
  - **Wait**  
    - Type: Wait node  
    - Waits 1 minute before next execution  
    - Used to allow HeyGen video processing time  
  - **Get Avatar Video**  
    - Type: HTTP Request  
    - Calls HeyGen `/v1/video_status.get` with video_id from previous step  
    - Checks video processing status and retrieves video URL if completed  
    - Headers include HeyGen API key  
  - **If Video Done**  
    - Type: Conditional node  
    - Checks if video status is `"completed"`  
    - If true, proceeds to upload; if false, loops back to Wait node for polling  
  - Edge cases: Video processing delays, video failure states, API errors, infinite loops if video never completes  

#### 1.6 Media Upload and Multi-Platform Posting  
- **Overview:** Uploads the completed video to Blotato media storage and posts it along with the AI-generated caption and title to nine social media platforms using Blotato API nodes.  
- **Nodes Involved:**  
  - Upload media (Blotato node)  
  - Tiktok [BLOTATO]  
  - Linkedin [BLOTATO]  
  - Facebook [BLOTATO]  
  - Instagram [BLOTATO]  
  - Twitter [BLOTATO]  
  - Youtube [BLOTATO]  
  - Threads [BLOTATO]  
  - Bluesky [BLOTATO]  
  - Pinterest [BLOTATO]  
  - Error Report (Merge node for errors)  
- **Node Details:**  
  - **Upload media**  
    - Type: Blotato media upload node  
    - Uploads video URL from HeyGen to Blotato media resource  
    - Credentials: Blotato API key required  
  - **Social media posting nodes** (TikTok, LinkedIn, Facebook, Instagram, Twitter, YouTube, Threads, Bluesky, Pinterest)  
    - Type: Blotato nodes specialized for each platform  
    - Parameters:  
      - Platform-specific account IDs  
      - Post content text: uses AI Writer’s generated caption or title depending on platform  
      - Media URLs: uses uploaded video URL from Blotato  
      - Optional platform-specific fields (e.g., Facebook Page ID, Pinterest Board ID)  
      - Flags indicating AI-generated content where supported  
    - Error handling: On error, continue execution and route to Error Report node  
    - Retry configured on Instagram and TikTok nodes with wait times  
  - **Error Report**  
    - Type: Merge node  
    - Merges error outputs from all social posting nodes for centralized error tracking  
  - Edge cases: API rate limits, invalid account credentials, media upload failure, platform-specific posting restrictions, warm-up requirements for new social accounts  

#### 1.7 Ancillary Notes and Setup Instructions  
- **Overview:** Several sticky note nodes provide setup instructions, troubleshooting tips, platform-specific posting advice, and API keys configuration hints. These are informational and do not affect workflow logic directly.  
- **Nodes Involved:** Sticky Note, Sticky Note2, Sticky Note4, Sticky Note7, Sticky Note8, Sticky Note9, Sticky Note10, Sticky Note11, Sticky Note12, Sticky Note13, Sticky Note14, Sticky Note15  
- **Node Details:**  
  - Contain links to documentation, tutorials, API key setup, warm-up guides, error troubleshooting, and platform-specific posting requirements  
  - Important notes about API plan requirements, avatar ID correctness, video background options, billing setup, and best practices  

---

### 3. Summary Table

| Node Name                           | Node Type                       | Functional Role                                | Input Node(s)                  | Output Node(s)                                         | Sticky Note                                                                                                                               |
|-----------------------------------|--------------------------------|-----------------------------------------------|-------------------------------|-------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger                  | Schedule Trigger                | Starts workflow daily at 10 AM                 | -                             | AI Research - Top 10                                  |                                                                                                                                           |
| AI Research - Top 10             | Perplexity                     | Fetch top 10 trending news in industry         | Schedule Trigger              | AI Research - Report                                  | Sticky Note, Sticky Note15                                                                                                                |
| AI Research - Report             | Perplexity                     | Select top viral story & create detailed report| AI Research - Top 10          | AI Writer                                            | Sticky Note, Sticky Note15                                                                                                                |
| AI Writer                       | OpenAI (ChatGPT)               | Generate video script, caption, and title      | AI Research - Report          | Setup Heygen                                         | Sticky Note2                                                                                                                             |
| Setup Heygen                    | Set                           | Store HeyGen API keys, avatar/voice & config  | AI Writer                    | If                                                  | Sticky Note12, Sticky Note13, Sticky Note14                                                                                              |
| If                             | If                            | Branch on background video usage flag          | Setup Heygen                 | Create Avatar Video WITH Background Video, WITHOUT Background Video |                                                                                                     |
| Create Avatar Video WITH Background Video | HTTP Request                 | Create avatar video with background video      | If                          | Merge                                               | Sticky Note12                                                                                                                             |
| Create Avatar Video WITHOUT Background Video | HTTP Request                 | Create avatar video without background video   | If                          | Merge                                               | Sticky Note12                                                                                                                             |
| Merge                          | Merge                         | Combine outputs from video creation branches   | Create Avatar Video nodes    | Wait                                                |                                                                                                                                           |
| Wait                           | Wait                          | Wait 1 minute before polling video status      | Merge                       | Get Avatar Video                                    |                                                                                                                                           |
| Get Avatar Video               | HTTP Request                 | Poll HeyGen API for video processing status     | Wait                        | If Video Done                                       |                                                                                                                                           |
| If Video Done                 | If                            | Check if video processing completed             | Get Avatar Video             | Upload media, Wait (loop if not done)                |                                                                                                                                           |
| Upload media                  | Blotato                       | Upload final video to Blotato media storage     | If Video Done               | Social media posting nodes                           | Sticky Note10                                                                                                                            |
| Tiktok [BLOTATO]              | Blotato                      | Post video to TikTok                             | Upload media                | Error Report (on error)                              | Sticky Note8                                                                                                                             |
| Linkedin [BLOTATO]            | Blotato                      | Post video to LinkedIn                           | Upload media                | Error Report (on error)                              | Sticky Note8                                                                                                                             |
| Facebook [BLOTATO]            | Blotato                      | Post video to Facebook                           | Upload media                | Error Report (on error)                              | Sticky Note8                                                                                                                             |
| Instagram [BLOTATO]           | Blotato                      | Post video to Instagram                          | Upload media                | Error Report (on error)                              | Sticky Note8                                                                                                                             |
| Twitter [BLOTATO]             | Blotato                      | Post video to Twitter                            | Upload media                | Error Report (on error)                              | Sticky Note8                                                                                                                             |
| Youtube [BLOTATO]             | Blotato                      | Post video to YouTube                            | Upload media                | Error Report (on error)                              | Sticky Note8                                                                                                                             |
| Threads [BLOTATO]             | Blotato                      | Post video to Threads                            | Upload media                | Error Report (on error)                              | Sticky Note8                                                                                                                             |
| Bluesky [BLOTATO]             | Blotato                      | Post video to Bluesky                            | Upload media                | Error Report (on error)                              | Sticky Note8                                                                                                                             |
| Pinterest [BLOTATO]           | Blotato                      | Post video to Pinterest                          | Upload media                | Error Report (on error)                              | Sticky Note8                                                                                                                             |
| Error Report                 | Merge                         | Collect errors from posting nodes               | All social posting nodes     | -                                                   | Sticky Note7                                                                                                                             |
| Sticky Notes (various)         | Sticky Note                  | Setup instructions, tips, troubleshooting       | -                           | -                                                   | See individual notes above                                                                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Set trigger to run daily at 10 AM  
   - Connect output to "AI Research - Top 10" node  

2. **Create "AI Research - Top 10" node**  
   - Type: Perplexity (API) node  
   - Model: sonar-pro  
   - Prompt: "Research the top 10 trending news items in my industry from the past 24 hours. Industry: real estate" (customize industry as needed)  
   - Credentials: Configure Perplexity API key  
   - Connect output to "AI Research - Report" node  

3. **Create "AI Research - Report" node**  
   - Type: Perplexity (API) node  
   - Model: sonar-pro  
   - Prompt: Tasks to select the most viral news story from top 10, research it, and compile detailed factual report with social media appeal reasons  
   - Credentials: Use same Perplexity API key  
   - Connect output to "AI Writer" node  

4. **Create "AI Writer" node**  
   - Type: OpenAI (ChatGPT)  
   - Model: GPT-5  
   - Input message: Pass detailed news report for script, caption, and title generation as per prompt instructions  
   - Output: JSON with `script`, `caption`, `title`  
   - Credentials: Configure OpenAI API key  
   - Connect output to "Setup Heygen" node  

5. **Create "Setup Heygen" node**  
   - Type: Set node  
   - Define JSON with keys: `heygen_api_key`, `avatar_id`, `voice_id`, `has_background_video` (boolean), `background_video_url` (default provided)  
   - Fill in your HeyGen API key, avatar ID, voice ID, and optionally background video URL  
   - Connect output to "If" node  

6. **Create "If" node**  
   - Condition: Check if `has_background_video` is true in Setup Heygen data  
   - True branch connects to "Create Avatar Video WITH Background Video" node  
   - False branch connects to "Create Avatar Video WITHOUT Background Video" node  

7. **Create "Create Avatar Video WITH Background Video" node**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.heygen.com/v2/video/generate`  
   - Headers: `X-Api-Key` set from Setup Heygen's API key  
   - Body (JSON): Include avatar info, voice info, background video URL, script text from AI Writer, dimension 720x1280, aspect ratio 9:16, title from AI Writer  
   - Connect output to "Merge" node  

8. **Create "Create Avatar Video WITHOUT Background Video" node**  
   - Same as above but omit background video info from JSON body  
   - Connect output to "Merge" node  

9. **Create "Merge" node**  
   - Merge inputs from both video creation branches (mode: default)  
   - Connect output to "Wait" node  

10. **Create "Wait" node**  
    - Wait 1 minute  
    - Connect output to "Get Avatar Video" node  

11. **Create "Get Avatar Video" node**  
    - Type: HTTP Request  
    - Method: GET  
    - URL: `https://api.heygen.com/v1/video_status.get`  
    - Query parameter: `video_id` from Merge node output  
    - Header: `X-Api-Key` from Setup Heygen  
    - Connect output to "If Video Done" node  

12. **Create "If Video Done" node**  
    - Condition: Check if `data.status` equals `"completed"`  
    - True: Connect to "Upload media" node  
    - False: Connect back to "Wait" node (loop)  

13. **Create "Upload media" node**  
    - Type: Blotato node (media upload)  
    - Media URL: Use video URL from "Get Avatar Video" response  
    - Credentials: Blotato API key configured  
    - Connect output to all social media posting nodes in parallel  

14. **Create social media posting nodes:**  
    - For each platform (TikTok, LinkedIn, Facebook, Instagram, Twitter, YouTube, Threads, Bluesky, Pinterest):  
      - Use Blotato nodes configured with platform, account ID, and necessary parameters  
      - Post content text: Use AI Writer’s caption or title as appropriate  
      - Media URLs: Use uploaded media URL from "Upload media" node  
      - Set flags for AI-generated content where supported  
      - On error: continue execution and route errors to "Error Report" node  
      - Configure retry for Instagram and TikTok nodes with delays  
    - Connect all error outputs to "Error Report" node  

15. **Create "Error Report" node**  
    - Type: Merge node  
    - Inputs: error outputs from all social media posting nodes  
    - Purpose: Aggregate errors for monitoring  

16. **Add Sticky Note nodes**  
    - Add notes for setup instructions, API key configuration, troubleshooting, platform-specific posting guidelines, and links as per original workflow  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Context or Link                                                                                             |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| Research Industry News using Perplexity: Sign up, billing, and API key setup at https://www.perplexity.ai/account/api/keys. Update your niche/industry in the research prompt accordingly.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | Sticky Note near AI Research nodes                                                                          |
| FULL TUTORIAL overview and detailed description of the workflow with links: https://help.blotato.com/api/templates/3-hackernews-to-ai-clone-videos. Explains the end-to-end process combining ChatGPT, Perplexity, HeyGen, and Blotato for daily viral news AI avatar video production and social posting.                                                                                                                                                                                                                                                                                                                                                                                                                | Sticky Note2                                                                                                |
| Setup instructions include creating API keys for Perplexity, Blotato, HeyGen; enabling Verified Community Nodes in n8n; installing Blotato node; and configuring avatar and voice IDs for HeyGen. Testing tips include enabling one social platform at a time and using shorter scripts to reduce processing time.                                                                                                                                                                                                                                                                                                                                                                                               | Sticky Note14                                                                                               |
| HeyGen video creation requires a paid API plan; free plan is insufficient. Longer AI scripts require longer wait times. Elevenlabs voice integration is possible through HeyGen web app. Tutorial for creating high-quality avatars and voices: https://youtu.be/_jogmHuuKXk. Background video requires higher tier avatar with background removed; configure 'has_background_video' and background video URL accordingly.                                                                                                                                                                                                                                                                                                   | Sticky Note12                                                                                               |
| Platform-specific posting notes: TikTok requires warmed-up accounts (warm-up guide: https://help.blotato.com/platforms/tiktok/brand-new-accounts). Each social platform has specific media support and posting best practices detailed here: https://help.blotato.com/platforms. YouTube shorts auto-post if video <2min and 9:16 aspect ratio; new YouTube accounts have API posting limits. Pinterest requires warmed-up boards and board IDs. Contact Blotato support via https://help.blotato.com or bottom-right support button on blotato.com.                                                                                                                           | Sticky Note8                                                                                                |
| Troubleshooting checklist for HeyGen: Must have paid API plan, correct avatar ID (not group avatar), and be aware that longer scripts take more video processing time.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | Sticky Note11                                                                                               |
| Troubleshooting checklist for Perplexity: API billing must be funded, update industry prompt, modify prompt for specific research needs.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Sticky Note15                                                                                               |
| Error reporting and logs can be viewed at Blotato API Dashboard: https://my.blotato.com/api-dashboard                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | Sticky Note7                                                                                                |
| Important posting guideline: Avoid posting the same image/video repeatedly to prevent spam flags. Disclose AI-generated content when using avatars. Links to Blotato API docs, media requirements, and troubleshooting: https://help.blotato.com/api, https://my.blotato.com/api-dashboard, https://help.blotato.com/api/media                                                                                                                                                                                                                                                                                                                                                                                      | Sticky Note10                                                                                               |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, a workflow automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected material. All manipulated data is legal and public.