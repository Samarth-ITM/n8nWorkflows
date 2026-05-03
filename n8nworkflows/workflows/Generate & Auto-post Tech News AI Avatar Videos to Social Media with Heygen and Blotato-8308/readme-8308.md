Generate & Auto-post Tech News AI Avatar Videos to Social Media with Heygen and Blotato

https://n8nworkflows.xyz/workflows/generate---auto-post-tech-news-ai-avatar-videos-to-social-media-with-heygen-and-blotato-8308


# Generate & Auto-post Tech News AI Avatar Videos to Social Media with Heygen and Blotato

### 1. Workflow Overview

This workflow automates the generation and posting of AI avatar videos featuring trending tech news sourced from Hacker News. It targets content creators and social media marketers focused on AI and technology news who want to produce engaging, viral-ready video content without manual filming or editing. The workflow orchestrates news retrieval, AI script generation, avatar video creation via Heygen API, video retrieval, media upload, and multi-platform social media posting through Blotato.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Trigger:** Initiates the workflow daily at a specified time.
- **1.2 AI News Research:** Fetches trending Hacker News articles related to AI and selects the top viral candidate.
- **1.3 AI Script and Caption Generation:** Uses OpenAI GPT-4 powered AI to generate a 30-second video script and social media captions (long and short).
- **1.4 Heygen Avatar Video Creation:** Prepares parameters and calls Heygen API to create avatar videos, optionally with background video.
- **1.5 Video Retrieval and Upload:** Waits for video processing completion, retrieves video, and uploads it to Blotato media storage.
- **1.6 Social Media Posting via Blotato:** Posts the generated video with captions to various social media platforms using Blotato nodes.
- **1.7 Workflow Configuration and Documentation:** Sticky notes provide instructions, tips, and links for setup and usage.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:** Starts the entire workflow automatically every day at 10 AM.
- **Nodes Involved:** Schedule Trigger
- **Node Details:**
  - **Type:** Schedule Trigger (n8n built-in)
  - **Configuration:** Set to trigger daily at hour 10 (10 AM)
  - **Inputs:** None (trigger node)
  - **Outputs:** Connects to AI Agent node
  - **Edge Cases:** If n8n instance is down or paused at trigger time, workflow won't run; ensure time zone is correct.
  - **Version:** 1.2

#### 1.2 AI News Research

- **Overview:** Retrieves top Hacker News stories tagged with AI, fetches the article and comments for the top viral story.
- **Nodes Involved:** AI Agent, Fetch HN Front Page, Fetch HN Article
- **Node Details:**

  - **AI Agent**
    - **Type:** Langchain Agent (AI orchestrator)
    - **Role:** Executes a multi-step instruction to:
      1. Fetch top 10 Hacker News AI/LLM stories from past 24h.
      2. Select highest viral potential story.
      3. Fetch article and comments.
      4. Generate a detailed 20-second monologue script (~30 seconds spoken).
      5. Modify first and last sentences for viral hook and call to action.
    - **Configuration:** Prompt text with detailed instructions and output format constraints (only exact video script).
    - **Inputs:** Trigger node output
    - **Outputs:** Connects to Write Long Caption node
    - **Version:** 1.8
    - **Edge Cases:** Possible API rate limits, failures in fetching Hacker News data, malformed article content, AI generation errors.
  
  - **Fetch HN Front Page**
    - **Type:** Hacker News Tool
    - **Role:** Fetches front page stories filtered by keyword "AI" and tagged "front_page".
    - **Inputs:** Invoked internally or by AI Agent
    - **Outputs:** Data fed to AI Agent
    - **Version:** 1
    - **Edge Cases:** Hacker News API availability, network errors.
  
  - **Fetch HN Article**
    - **Type:** Hacker News Tool
    - **Role:** Fetches full article and comments for the selected Hacker News story.
    - **Inputs:** Article ID dynamically provided by AI Agent output
    - **Outputs:** Data used by AI Agent
    - **Version:** 1
    - **Edge Cases:** Article ID missing or invalid, comments inaccessible.

#### 1.3 AI Script and Caption Generation

- **Overview:** Generates the video monologue script and two social media captions (long and short) using OpenAI GPT-4.
- **Nodes Involved:** Write Script, Write Long Caption, Write Short Caption
- **Node Details:**

  - **Write Script**
    - **Type:** Langchain LM Chat OpenAI (GPT-4.1)
    - **Role:** Generates initial video script based on AI Agent instructions.
    - **Credentials:** OpenAI API (brand account)
    - **Inputs:** From AI Agent node
    - **Outputs:** Feeds AI Agent node (loop for intermediate steps)
    - **Version:** 1.2
    - **Edge Cases:** API quota limits, prompt parsing errors.

  - **Write Long Caption**
    - **Type:** Langchain OpenAI GPT-4O
    - **Role:** Creates a detailed 50-word video caption with structured format (summary, bullet questions, hashtags).
    - **Credentials:** OpenAI API (brand account)
    - **Inputs:** AI Agent output (video script)
    - **Outputs:** Write Short Caption node
    - **Version:** 1.8
    - **Edge Cases:** Formatting inconsistencies, API failures.

  - **Write Short Caption**
    - **Type:** Langchain OpenAI GPT-4O
    - **Role:** Produces a viral, concise sentence (max 90 chars) summarizing the video.
    - **Credentials:** OpenAI API (brand account)
    - **Inputs:** Write Long Caption output
    - **Outputs:** Setup Heygen node
    - **Version:** 1.8
    - **Edge Cases:** Overly short or vague content, API rate limiting.

#### 1.4 Heygen Avatar Video Creation

- **Overview:** Configures video creation parameters and calls Heygen API to generate avatar videos with or without background video.
- **Nodes Involved:** Setup Heygen, If, Create Avatar Video WITH Background Video, Create Avatar Video WITHOUT Background Video, Merge
- **Node Details:**

  - **Setup Heygen**
    - **Type:** Set node
    - **Role:** Holds Heygen API key, avatar ID, voice ID, background video usage flag, and background video URL.
    - **Configuration:** Raw JSON with placeholders for:
      - heygen_api_key
      - avatar_id
      - voice_id
      - has_background_video (boolean)
      - background_video_url (default set to a public video URL)
    - **Outputs:** Connects to If node
    - **Version:** 3.4
    - **Edge Cases:** Missing keys or invalid IDs cause API errors.

  - **If**
    - **Type:** If node (boolean condition)
    - **Role:** Branches workflow based on has_background_video flag.
    - **Inputs:** Setup Heygen output
    - **Outputs:** Two branches:
      - True → Create Avatar Video WITH Background Video
      - False → Create Avatar Video WITHOUT Background Video
    - **Version:** 2.2

  - **Create Avatar Video WITH Background Video**
    - **Type:** HTTP Request
    - **Role:** Calls Heygen API endpoint to generate video with avatar + background video.
    - **Configuration:**
      - POST to https://api.heygen.com/v2/video/generate
      - JSON body includes avatar_id, voice_id, input_text from AI Agent output, background video URL and settings
      - Uses X-Api-Key header from Setup Heygen
      - Video dimension 720x1280, aspect ratio 9:16
      - Video title set from Write Short Caption content
    - **Inputs:** If node (true branch)
    - **Outputs:** Merge node
    - **Version:** 4.2
    - **Edge Cases:** API authentication failure, invalid avatar/voice IDs, network errors.

  - **Create Avatar Video WITHOUT Background Video**
    - **Type:** HTTP Request
    - **Role:** Similar to above but without background video section in JSON body.
    - **Inputs:** If node (false branch)
    - **Outputs:** Merge node
    - **Version:** 4.2
    - **Edge Cases:** Same as above.

  - **Merge**
    - **Type:** Merge node
    - **Role:** Merges output from both avatar video creation branches into a single stream.
    - **Inputs:** Both Create Avatar Video nodes
    - **Outputs:** Wait node
    - **Version:** 3.2

#### 1.5 Video Retrieval and Upload

- **Overview:** Waits for video processing, then retrieves the completed video from Heygen and uploads it to Blotato media storage.
- **Nodes Involved:** Wait, Get Avatar Video, Upload media
- **Node Details:**

  - **Wait**
    - **Type:** Wait node
    - **Role:** Pauses execution for 8 minutes to allow Heygen video rendering.
    - **Inputs:** Merge node output
    - **Outputs:** Get Avatar Video node
    - **Version:** 1.1
    - **Edge Cases:** Insufficient wait time causes incomplete video retrieval.

  - **Get Avatar Video**
    - **Type:** HTTP Request
    - **Role:** Fetches video processing status and video URL from Heygen API using video_id from avatar creation response.
    - **Configuration:**
      - GET to https://api.heygen.com/v1/video_status.get
      - Query param: video_id
      - Header: X-Api-Key from Setup Heygen
    - **Inputs:** Wait node
    - **Outputs:** Upload media node
    - **Version:** 4.2
    - **Edge Cases:** API errors, video not ready, invalid video_id.

  - **Upload media**
    - **Type:** Blotato node
    - **Role:** Uploads the finished video URL to Blotato media storage for later posting.
    - **Credentials:** Blotato API account configured
    - **Inputs:** Get Avatar Video node
    - **Outputs:** Multiple social media posting nodes
    - **Version:** 2
    - **Edge Cases:** Upload failures, invalid media URL.

#### 1.6 Social Media Posting via Blotato

- **Overview:** Posts the uploaded video with captions to various social platforms using Blotato nodes.
- **Nodes Involved:** Tiktok [BLOTATO], Linkedin [BLOTATO], Facebook [BLOTATO], Instagram [BLOTATO], Twitter [BLOTATO], Youtube [BLOTATO], Threads [BLOTATO], Bluesky [BLOTATO], Pinterest [BLOTATO] (disabled)
- **Node Details (common for all):**

  - **Type:** Blotato node for respective platform
  - **Role:** Posts video with text caption and media URL to the platform
  - **Configuration:**
    - Platform specified (e.g., tiktok, linkedin, facebook)
    - AccountId set per platform (cached from Blotato API)
    - Post content text from Write Long Caption or Write Short Caption depending on platform
    - Media URLs from Upload media node output
  - **Credentials:** Blotato API credentials required
  - **Inputs:** Upload media node
  - **Outputs:** None (end of workflow branches)
  - **Version:** 2
  - **Edge Cases:** API rate limits, authentication errors, platform-specific media requirements, duplicate content flags.
  - **Notes:** Pinterest node is disabled to prevent accidental posting.

#### 1.7 Workflow Configuration and Documentation

- **Overview:** Multiple sticky notes provide user instructions, setup tips, API documentation links, and best practices.
- **Nodes Involved:** Sticky Note, Sticky Note1, Sticky Note2, Sticky Note3, Sticky Note5, Sticky Note6
- **Node Details:**

  - **Sticky Note**
    - Explains Hackernews research approach and suggests SerpAPI alternative for other niches.
  - **Sticky Note1**
    - Details Heygen API requirements, background video usage, tutorial link.
  - **Sticky Note2**
    - Full tutorial link and detailed overview of the entire workflow steps.
  - **Sticky Note3**
    - Reminder to fill out "Setup Heygen" node parameters.
  - **Sticky Note5**
    - Blotato posting best practices, media requirements, support links.
  - **Sticky Note6**
    - Instructions on selecting social accounts in posting nodes.
  - **Edge Cases:** None (informational only)

---

### 3. Summary Table

| Node Name                         | Node Type                         | Functional Role                            | Input Node(s)                     | Output Node(s)                                           | Sticky Note                                                                                                                                                                                                                                                                             |
|----------------------------------|----------------------------------|--------------------------------------------|----------------------------------|----------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger                 | Schedule Trigger                 | Starts workflow daily at 10 AM             | None                             | AI Agent                                                 | # Research Tech News using Hackernews - This template uses the free Hackernews tool to fetch trending tech news - If your niche is outside of tech, I suggest replacing the Hackernews tool with SerpAPI tool, which allows your AI agent to use Google search to find recent industry news        |
| AI Agent                       | Langchain Agent                 | Orchestrates news fetching and script generation | Schedule Trigger                | Write Long Caption                                       | # Research Tech News using Hackernews - This template uses the free Hackernews tool to fetch trending tech news - If your niche is outside of tech, I suggest replacing the Hackernews tool with SerpAPI tool, which allows your AI agent to use Google search to find recent industry news        |
| Fetch HN Front Page             | Hacker News Tool                | Fetches front page AI-related stories      | Invoked by AI Agent              | AI Agent                                                 | # Research Tech News using Hackernews - This template uses the free Hackernews tool to fetch trending tech news - If your niche is outside of tech, I suggest replacing the Hackernews tool with SerpAPI tool, which allows your AI agent to use Google search to find recent industry news        |
| Fetch HN Article                | Hacker News Tool                | Fetches article and comments for top story | Invoked by AI Agent              | AI Agent                                                 | # Research Tech News using Hackernews - This template uses the free Hackernews tool to fetch trending tech news - If your niche is outside of tech, I suggest replacing the Hackernews tool with SerpAPI tool, which allows your AI agent to use Google search to find recent industry news        |
| Write Script                   | Langchain LM Chat OpenAI        | Generates video monologue script            | AI Agent                        | AI Agent                                                 |                                                                                                                                                                                                                                                                                         |
| Write Long Caption             | Langchain OpenAI                | Creates detailed video caption              | AI Agent                        | Write Short Caption                                      |                                                                                                                                                                                                                                                                                         |
| Write Short Caption            | Langchain OpenAI                | Creates viral short caption                  | Write Long Caption              | Setup Heygen                                            |                                                                                                                                                                                                                                                                                         |
| Setup Heygen                  | Set                            | Stores Heygen API keys and avatar config    | Write Short Caption             | If                                                      | # Create AI Clone Video Using HeyGen - This requires HeyGen API plan (paid); the free plan is insufficient: https://www.heygen.com/api-pricing - If you have a long script, you may need to increase the WAIT time. - You can use a custom Elevenlabs voice by integrating Elevenlabs within HeyGen web app. - Tutorial on how to create a high-quality avatar and voice clone: https://youtu.be/_jogmHuuKXk - If you want a background video playing behind your avatar: (1) ensure you have an avatar with background removed which requires a higher tier plan; (2) open SETUP HEYGEN node and set parameter 'has_background_video' to true; (3) open SETUP HEYGEN node and replace video URL in parameter 'background_video_url' |
| If                            | If                             | Branches on background video usage          | Setup Heygen                   | Create Avatar Video WITH Background Video / Create Avatar Video WITHOUT Background Video |                                                                                                                                                                                                                                                                                         |
| Create Avatar Video WITH Background Video | HTTP Request                  | Calls Heygen API to create avatar video with background | If (true branch)               | Merge                                                   |                                                                                                                                                                                                                                                                                         |
| Create Avatar Video WITHOUT Background Video | HTTP Request                  | Calls Heygen API to create avatar video without background | If (false branch)              | Merge                                                   |                                                                                                                                                                                                                                                                                         |
| Merge                         | Merge                          | Merges avatar video creation branches       | Create Avatar Video nodes       | Wait                                                    |                                                                                                                                                                                                                                                                                         |
| Wait                          | Wait                           | Delays workflow for video processing         | Merge                         | Get Avatar Video                                        |                                                                                                                                                                                                                                                                                         |
| Get Avatar Video              | HTTP Request                  | Retrieves processed video status and URL    | Wait                          | Upload media                                           |                                                                                                                                                                                                                                                                                         |
| Upload media                 | Blotato                       | Uploads video media to Blotato storage       | Get Avatar Video              | Multiple Blotato social posting nodes                  | # Post Everywhere Using Blotato - IMPORTANT: Do not post the same video over and over again, otherwise it will be flagged as spam. Make sure to disclose AI generated content if you're using avatars. - 👉  **Blotato API Docs**: https://help.blotato.com/api - ✅  **Troubleshoot Errors**: https://my.blotato.com/api-dashboard - 📷  **Media Requirements**: https://help.blotato.com/api/media - 🛠️  **Contact help**: log into blotato > click support button in bottom right corner |
| Tiktok [BLOTATO]             | Blotato                       | Posts video to TikTok                         | Upload media                  | None                                                   | # Step 2 1. Open each node 2. Select your social account 3. 👉 **You don't need to do anything else!**                                                                                                                                                                                    |
| Linkedin [BLOTATO]           | Blotato                       | Posts video to LinkedIn                       | Upload media                  | None                                                   | # Step 2 1. Open each node 2. Select your social account 3. 👉 **You don't need to do anything else!**                                                                                                                                                                                    |
| Facebook [BLOTATO]           | Blotato                       | Posts video to Facebook                       | Upload media                  | None                                                   | # Step 2 1. Open each node 2. Select your social account 3. 👉 **You don't need to do anything else!**                                                                                                                                                                                    |
| Instagram [BLOTATO]          | Blotato                       | Posts video to Instagram                      | Upload media                  | None                                                   | # Step 2 1. Open each node 2. Select your social account 3. 👉 **You don't need to do anything else!**                                                                                                                                                                                    |
| Twitter [BLOTATO]            | Blotato                       | Posts video to Twitter                        | Upload media                  | None                                                   | # Step 2 1. Open each node 2. Select your social account 3. 👉 **You don't need to do anything else!**                                                                                                                                                                                    |
| Youtube [BLOTATO]            | Blotato                       | Posts video to YouTube                        | Upload media                  | None                                                   | # Step 2 1. Open each node 2. Select your social account 3. 👉 **You don't need to do anything else!**                                                                                                                                                                                    |
| Threads [BLOTATO]            | Blotato                       | Posts video to Threads                        | Upload media                  | None                                                   | # Step 2 1. Open each node 2. Select your social account 3. 👉 **You don't need to do anything else!**                                                                                                                                                                                    |
| Bluesky [BLOTATO]            | Blotato                       | Posts video to Bluesky                        | Upload media                  | None                                                   | # Step 2 1. Open each node 2. Select your social account 3. 👉 **You don't need to do anything else!**                                                                                                                                                                                    |
| Pinterest [BLOTATO]          | Blotato (disabled)            | Posts video to Pinterest (disabled)          | Upload media                  | None                                                   | # Step 2 1. Open each node 2. Select your social account 3. 👉 **You don't need to do anything else!**                                                                                                                                                                                    |
| Sticky Note                  | Sticky Note                   | Instructions on Hackernews research          | None                         | None                                                   | # Research Tech News using Hackernews - This template uses the free Hackernews tool to fetch trending tech news - If your niche is outside of tech, I suggest replacing the Hackernews tool with SerpAPI tool, which allows your AI agent to use Google search to find recent industry news        |
| Sticky Note1                 | Sticky Note                   | Heygen setup notes and tutorial links        | None                         | None                                                   | # Create AI Clone Video Using HeyGen - This requires HeyGen API plan (paid); the free plan is insufficient: https://www.heygen.com/api-pricing - If you have a long script, you may need to increase the WAIT time. - You can use a custom Elevenlabs voice by integrating Elevenlabs within HeyGen web app. - Tutorial on how to create a high-quality avatar and voice clone: https://youtu.be/_jogmHuuKXk - If you want a background video playing behind your avatar: (1) ensure you have an avatar with background removed which requires a higher tier plan; (2) open SETUP HEYGEN node and set parameter 'has_background_video' to true; (3) open SETUP HEYGEN node and replace video URL in parameter 'background_video_url' |
| Sticky Note2                 | Sticky Note                   | Full workflow tutorial and overview          | None                         | None                                                   | # FULL TUTORIAL - https://help.blotato.com/api/templates/3-hackernews-to-ai-clone-videos - Describes the entire automatic process from news fetching to posting.                                                                                                                                                              |
| Sticky Note3                 | Sticky Note                   | Reminder to fill Heygen setup parameters     | None                         | None                                                   | # Step 1 - Fill out "Setup Heygen"                                                                                                                                                                                                                                                     |
| Sticky Note5                 | Sticky Note                   | Blotato posting best practices and docs      | None                         | None                                                   | # Post Everywhere Using Blotato - IMPORTANT: Do not post the same video over and over again, otherwise it will be flagged as spam. Make sure to disclose AI generated content if you're using avatars. - 👉  **Blotato API Docs**: https://help.blotato.com/api - ✅  **Troubleshoot Errors**: https://my.blotato.com/api-dashboard - 📷  **Media Requirements**: https://help.blotato.com/api/media - 🛠️  **Contact help**: log into blotato > click support button in bottom right corner |
| Sticky Note6                 | Sticky Note                   | Instructions for social account selection     | None                         | None                                                   | # Step 2 - 1. Open each node 2. Select your social account 3. 👉 **You don't need to do anything else!**                                                                                                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**
   - Type: Schedule Trigger
   - Parameters: Set trigger to daily at 10 AM
   - Connect output to AI Agent node

2. **Create AI Agent Node**
   - Type: Langchain Agent (@n8n/n8n-nodes-langchain.agent)
   - Parameters:
     - Text: Multi-step instructions to fetch Hacker News, select top viral story, fetch article and comments, create 30-second monologue script with viral hooks, and output only exact video script
     - Options: Enable returnIntermediateSteps = true
   - Connect input from Schedule Trigger
   - Connect output to Write Long Caption node

3. **Create Hacker News Tool Nodes**
   - Fetch HN Front Page
     - Type: Hacker News Tool
     - Parameters: resource "all", tags ["front_page"], keyword "AI"
     - Invoked internally or referenced in AI Agent
   - Fetch HN Article
     - Type: Hacker News Tool
     - Parameters: articleId dynamic from AI Agent output, includeComments = true
     - Invoked internally or referenced in AI Agent

4. **Create Write Script Node**
   - Type: Langchain LM Chat OpenAI
   - Parameters: model gpt-4.1
   - Credentials: OpenAI API (brand)
   - Connect input from AI Agent (if integrating separately)
   - Connect output to AI Agent (if applicable)

5. **Create Write Long Caption Node**
   - Type: Langchain OpenAI
   - Parameters:
     - modelId: GPT-4O
     - Messages: Prompt to write 50-word video caption with specific style and content constraints, using AI Agent output as source
   - Credentials: OpenAI API (brand)
   - Connect input from AI Agent output
   - Connect output to Write Short Caption

6. **Create Write Short Caption Node**
   - Type: Langchain OpenAI
   - Parameters:
     - modelId: GPT-4O
     - Messages: Prompt to write 1 viral sentence max 90 chars summarizing video, using Write Long Caption output
   - Credentials: OpenAI API (brand)
   - Connect input from Write Long Caption
   - Connect output to Setup Heygen

7. **Create Setup Heygen Node**
   - Type: Set
   - Parameters: Raw JSON including
     - heygen_api_key (your API key)
     - avatar_id (your avatar ID)
     - voice_id (your voice ID)
     - has_background_video (boolean flag)
     - background_video_url (default or custom URL)
   - Connect input from Write Short Caption
   - Connect output to If node

8. **Create If Node**
   - Type: If
   - Parameters: Condition on has_background_video boolean field (true/false)
   - Connect input from Setup Heygen
   - Connect true branch to Create Avatar Video WITH Background Video
   - Connect false branch to Create Avatar Video WITHOUT Background Video

9. **Create Create Avatar Video WITH Background Video Node**
   - Type: HTTP Request
   - Parameters:
     - Method: POST
     - URL: https://api.heygen.com/v2/video/generate
     - Body (JSON): includes avatar, voice, input_text from AI Agent output, background video URL and settings, video dimension 720x1280, aspect ratio 9:16, title from Write Short Caption
     - Headers: X-Api-Key with heygen_api_key from Setup Heygen
   - Connect input from If true branch
   - Connect output to Merge node

10. **Create Create Avatar Video WITHOUT Background Video Node**
    - Type: HTTP Request
    - Parameters: Similar to above but without background video in JSON body
    - Connect input from If false branch
    - Connect output to Merge node

11. **Create Merge Node**
    - Type: Merge
    - Parameters: default merge
    - Connect inputs from both Create Avatar Video nodes
    - Connect output to Wait node

12. **Create Wait Node**
    - Type: Wait
    - Parameters: Wait for 8 minutes
    - Connect input from Merge node
    - Connect output to Get Avatar Video node

13. **Create Get Avatar Video Node**
    - Type: HTTP Request
    - Parameters:
      - Method: GET
      - URL: https://api.heygen.com/v1/video_status.get
      - Query: video_id from video generation response
      - Headers: X-Api-Key from Setup Heygen
    - Connect input from Wait node
    - Connect output to Upload media node

14. **Create Upload media Node**
    - Type: Blotato
    - Parameters:
      - mediaUrl: video_url from Get Avatar Video output
      - resource: media
    - Credentials: Blotato API configured with your account
    - Connect input from Get Avatar Video
    - Connect output to all social media posting nodes

15. **Create Social Media Posting Nodes (one per platform)**
    - Platforms: TikTok, LinkedIn, Facebook, Instagram, Twitter, YouTube, Threads, Bluesky, Pinterest (optional/disabled)
    - Type: Blotato
    - Parameters:
      - platform: specific platform name
      - accountId: select from Blotato accounts list
      - postContentText: use Write Long Caption output for most platforms; Write Short Caption for Twitter, Threads, Bluesky
      - postContentMediaUrls: URL from Upload media node
      - Additional platform-specific parameters (e.g. Facebook page ID, YouTube video title from short caption slice)
    - Credentials: Blotato API configured
    - Connect input from Upload media node

16. **Add Sticky Notes for Documentation**
    - Add sticky notes with content covering:
      - Hackernews research explanation
      - Heygen setup instructions and tutorial links
      - Full workflow tutorial link
      - Reminder to fill Heygen setup parameters
      - Blotato posting best practices and support docs
      - Instructions for social account selection

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                         | Context or Link                                                                                                        |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------|
| This template uses free Hackernews tool to fetch trending tech news. For niches outside tech, replace Hackernews tool with SerpAPI to enable AI agent to search recent news on Google.                                                                                                                                                                                | Sticky Note (Research Tech News)                                                                                       |
| Heygen API requires a paid plan; free plan is insufficient. For long scripts, increase WAIT time. Custom Elevenlabs voices can be used via Heygen web app integration. Tutorial on creating high-quality avatar and voice clone: https://youtu.be/_jogmHuuKXk. Background video requires higher-tier avatar with background removed.                                     | Sticky Note1 (Heygen setup)                                                                                            |
| Full tutorial for this workflow and detailed description: https://help.blotato.com/api/templates/3-hackernews-to-ai-clone-videos                                                                                                                                                                                                                                    | Sticky Note2 (Full Workflow Tutorial)                                                                                   |
| Do not post the same video repeatedly; it may be flagged as spam. Disclose AI-generated content if using avatars. Blotato API docs: https://help.blotato.com/api, Troubleshooting: https://my.blotato.com/api-dashboard, Media Requirements: https://help.blotato.com/api/media, Contact support via Blotato dashboard.                                                   | Sticky Note5 (Blotato Best Practices)                                                                                   |
| After opening each social posting node, select your social account. No further action needed to post.                                                                                                                                                                                                                                                               | Sticky Note6 (Social Account Selection)                                                                                  |

---

This comprehensive reference document enables full understanding, reproduction, and modification of the workflow, highlighting configuration details, dependencies, edge cases, and integration specifics for Heygen and Blotato APIs alongside AI-powered content generation.