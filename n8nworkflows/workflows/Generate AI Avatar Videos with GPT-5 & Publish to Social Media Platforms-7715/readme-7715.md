Generate AI Avatar Videos with GPT-5 & Publish to Social Media Platforms

https://n8nworkflows.xyz/workflows/generate-ai-avatar-videos-with-gpt-5---publish-to-social-media-platforms-7715


# Generate AI Avatar Videos with GPT-5 & Publish to Social Media Platforms

### 1. Workflow Overview

This workflow automates the complete process of generating AI avatar videos from voice input, leveraging GPT-5 for content creation, and publishing the resulting videos across nine social media platforms. It is designed for content creators, marketers, or social media managers who want to transform simple voice notes into engaging videos and distribute them automatically.

The workflow is logically divided into four main blocks:

- **1.1 Input Capture and Transcription:** Receives a voice message via Telegram, downloads and stores the audio, transcribes it using OpenAI, and logs metadata.
- **1.2 AI Content Generation:** Uses GPT-5 via LangChain to generate a video title and caption from the transcription.
- **1.3 AI Avatar Video Creation:** Calls HeyGen API to create an AI avatar video based on the selected avatar ID and voice audio.
- **1.4 Auto-Publishing to Social Media:** Uploads the final video to Google Drive, then uses Blotato nodes to post it on YouTube, TikTok, Instagram, Facebook, LinkedIn, Twitter (X), Threads, Bluesky, and Pinterest, followed by notifications via Telegram.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Capture and Transcription

- **Overview:**  
  This block initiates the workflow by capturing voice messages sent via Telegram, downloading the voice file, uploading it to Google Drive for storage, and transcribing the audio content to text. It logs voice metadata in Google Sheets for tracking.

- **Nodes Involved:**  
  - Telegram Trigger: Receive Voice Message  
  - Telegram: Download Voice  
  - Google Drive: Upload Voice  
  - TmpFiles: Upload Voice (Public URL)  
  - OpenAI - Transcribe Video to Text  
  - Google Sheets: Log Voice Metadata

- **Node Details:**  
  - *Telegram Trigger: Receive Voice Message*  
    - Type: Trigger node  
    - Configured to listen for "message" updates in Telegram, specifically voice messages.  
    - Outputs the Telegram message JSON including voice file metadata.  
    - Potential failure: Telegram API downtime, webhook misconfiguration.

  - *Telegram: Download Voice*  
    - Type: Telegram node  
    - Downloads the voice file using the voice file ID from the trigger node.  
    - Outputs binary data of the voice file.  
    - Failure cases: File not found or download errors.

  - *Google Drive: Upload Voice*  
    - Type: Google Drive node  
    - Uploads the downloaded voice file to a specified Google Drive folder.  
    - Uses OAuth2 credentials for authentication.  
    - Stores file under the unique voice file ID name.  
    - Failure: Google Drive API rate limits, permission issues.

  - *TmpFiles: Upload Voice (Public URL)*  
    - Type: HTTP Request  
    - Uploads voice file binary to tmpfiles.org to generate a public URL with direct download link (modifies URL for direct access).  
    - This URL is later used for voice input in video generation.  
    - Failure: tmpfiles.org service unavailability or upload errors.

  - *OpenAI - Transcribe Video to Text*  
    - Type: OpenAI node via LangChain  
    - Transcribes the uploaded audio into text with language set to French and zero temperature for deterministic output.  
    - Failure: API quota exceeded, transcription errors.

  - *Google Sheets: Log Voice Metadata*  
    - Type: Google Sheets node  
    - Logs voice file metadata including unique ID and upload URLs to a Google Sheet for record-keeping.  
    - Uses OAuth2 credentials.  
    - Failure: Sheet access issues or incorrect sheet ID.

---

#### 2.2 AI Content Generation

- **Overview:**  
  This block generates a compelling video title and caption based solely on the transcribed text using GPT-5, ensuring language detection and strict adherence to transcription content. Updates the metadata sheet with the generated content.

- **Nodes Involved:**  
  - AI Agent - Generate Title & Caption  
  - OpenAI Model GPT-5  
  - LangChain - Think Tool  
  - Google Sheets - Update Title & Caption

- **Node Details:**  
  - *OpenAI Model GPT-5*  
    - Type: Language Model Chat Node (LangChain)  
    - Calls the GPT-5-mini model to generate natural language output.  
    - Acts as the AI engine feeding the Agent.  
    - Failure: API quota, model unavailability.

  - *LangChain - Think Tool*  
    - Type: AI Tool Node  
    - Used for intermediate logic or augmentation in LangChain prompt workflow.  
    - Passes output to the Agent node.

  - *AI Agent - Generate Title & Caption*  
    - Type: LangChain Agent Node  
    - Receives transcribed text, applies a system prompt instructing it to generate a JSON with a concise Title (≤70 chars) and Caption (≤200 chars), in the transcription language, no emojis or hashtags, no invented info.  
    - Output is strict single-line JSON for easy parsing.  
    - Failure: Prompt misformatting, AI hallucination, JSON parsing errors.

  - *Google Sheets - Update Title & Caption*  
    - Type: Google Sheets Tool Node  
    - Updates or appends the Google Sheet row corresponding to the voice ID with the new Title and Caption generated by the AI.  
    - Failure: Sheet permissions, incorrect matching on ID VOICE.

---

#### 2.3 AI Avatar Video Creation

- **Overview:**  
  Using the avatar ID and voice URL from prior steps, this block calls HeyGen API to generate an AI avatar video, waits for processing, checks status, and downloads the final video.

- **Nodes Involved:**  
  - Google Sheets: Read Avatar Config  
  - ID Avatar  
  - HeyGen: Generate Avatar Video  
  - Wait for Rendering  
  - HeyGen: Check Video Status  
  - If: Video Completed?  
  - HTTP: Download Final Video  
  - Google Drive: Upload Final Video  
  - Google Sheets: Save Final Video URL

- **Node Details:**  
  - *Google Sheets: Read Avatar Config*  
    - Type: Google Sheets node  
    - Reads avatar configuration including avatar IDs from a Google Sheet.  
    - Failure: Sheet access, misconfiguration.

  - *ID Avatar*  
    - Type: Set node  
    - Sets variables `id_avatar` from the avatar config and `voice_url` from the public tmpfiles URL, preparing inputs for HeyGen.  
    - Failure: Expression errors if referenced nodes have no output.

  - *HeyGen: Generate Avatar Video*  
    - Type: HTTP Request node  
    - Posts to HeyGen API with JSON body specifying avatar ID, voice audio URL, and video dimensions.  
    - Authenticated via HTTP header credentials.  
    - Failure: API errors, invalid avatar ID, voice URL unreachable.

  - *Wait for Rendering*  
    - Type: Wait node  
    - Pauses execution to allow video rendering to complete (exact wait time configurable).  
    - Failure: Too short wait causing premature status check.

  - *HeyGen: Check Video Status*  
    - Type: HTTP Request node  
    - Polls HeyGen API with video ID to check processing status.  
    - Failure: API downtime, rate limits.

  - *If: Video Completed?*  
    - Type: If node  
    - Checks if HeyGen video status is “completed”.  
    - Routes to download or back to wait depending on status.

  - *HTTP: Download Final Video*  
    - Type: HTTP Request node  
    - Downloads the final rendered video from HeyGen’s video URL.

  - *Google Drive: Upload Final Video*  
    - Type: Google Drive node  
    - Uploads the downloaded video to a designated Google Drive folder, naming it by the voice message ID.

  - *Google Sheets: Save Final Video URL*  
    - Type: Google Sheets node  
    - Updates the Google Sheet with the Google Drive URL of the final video, matching by voice ID.

---

#### 2.4 Auto-Publishing to Social Media

- **Overview:**  
  This block publishes the final video to nine social media platforms via Blotato nodes, each configured with the appropriate account and post content from Google Sheets. Finally, it sends confirmation back to Telegram.

- **Nodes Involved:**  
  - Get Google Drive ID  
  - Upload Video to BLOTATO  
  - Tiktok  
  - Linkedin  
  - Facebook  
  - Instagram  
  - Twitter (X)  
  - Youtube  
  - Threads  
  - Bluesky  
  - Pinterest  
  - Merge  
  - Google Sheets - Update Status  
  - Telegram: Send Post Confirmation  
  - Telegram: Send Final Video

- **Node Details:**  
  - *Get Google Drive ID*  
    - Type: Set node  
    - Extracts the Google Drive file ID from the stored video URL for Blotato upload.

  - *Upload Video to BLOTATO*  
    - Type: Blotato node  
    - Uploads the video from Google Drive URL to Blotato for distribution.

  - *Platform-specific Blotato nodes (Tiktok, Linkedin, Facebook, Instagram, Twitter (X), Youtube, Threads, Bluesky, Pinterest)*  
    - Each is a Blotato node configured with platform, account ID, post text from Google Sheets caption, and media URL from uploaded video.  
    - Credentials: Blotato API key.  
    - Failure: API limits, authentication errors, media URL invalid.

  - *Merge*  
    - Type: Merge node (chooseBranch mode)  
    - Combines outputs from all platform nodes to unify flow.

  - *Google Sheets - Update Status*  
    - Type: Google Sheets node  
    - Updates the post status to "Posted" for the voice message ID.

  - *Telegram: Send Post Confirmation*  
    - Type: Telegram node  
    - Sends a confirmation message "Posted" back to the Telegram chat.

  - *Telegram: Send Final Video*  
    - Type: Telegram node  
    - Sends the posted video back to the Telegram chat as a video message.

---

### 3. Summary Table

| Node Name                           | Node Type                          | Functional Role                             | Input Node(s)                       | Output Node(s)                             | Sticky Note                                  |
|-----------------------------------|----------------------------------|---------------------------------------------|-----------------------------------|--------------------------------------------|----------------------------------------------|
| Telegram Trigger: Receive Voice Message | Telegram Trigger                  | Entry point: receive voice message           | —                                 | Telegram: Download Voice                    | Step 1 — Capture & Transcribe Voice Input     |
| Telegram: Download Voice           | Telegram                         | Download voice audio file                     | Telegram Trigger                   | Google Drive: Upload Voice, TmpFiles, OpenAI Transcribe | Step 1 — Capture & Transcribe Voice Input     |
| Google Drive: Upload Voice         | Google Drive                     | Upload voice audio to Drive                    | Telegram: Download Voice           | Google Sheets: Log Voice Metadata           | Step 1 — Capture & Transcribe Voice Input     |
| TmpFiles: Upload Voice (Public URL) | HTTP Request                    | Upload voice for public direct URL            | Telegram: Download Voice           | Google Sheets: Read Avatar Config           | Step 1 — Capture & Transcribe Voice Input     |
| OpenAI - Transcribe Video to Text | OpenAI (LangChain)               | Transcribe audio to text                       | Telegram: Download Voice           | AI Agent - Generate Title & Caption         | Step 1 — Capture & Transcribe Voice Input     |
| Google Sheets: Log Voice Metadata  | Google Sheets                   | Log voice metadata                             | Google Drive: Upload Voice         | —                                          | Step 1 — Capture & Transcribe Voice Input     |
| Google Sheets: Read Avatar Config  | Google Sheets                   | Retrieve avatar configuration                  | TmpFiles: Upload Voice             | ID Avatar                                   | Step 3 — Create AI Avatar Video                 |
| ID Avatar                         | Set                             | Set avatar ID and voice URL                    | Google Sheets: Read Avatar Config  | HeyGen: Generate Avatar Video               | Step 3 — Create AI Avatar Video                 |
| HeyGen: Generate Avatar Video      | HTTP Request                    | Request HeyGen to create avatar video          | ID Avatar                        | Wait for Rendering                          | Step 3 — Create AI Avatar Video                 |
| Wait for Rendering                | Wait                            | Delay for video rendering                        | HeyGen: Generate Avatar Video      | HeyGen: Check Video Status                   | Step 3 — Create AI Avatar Video                 |
| HeyGen: Check Video Status         | HTTP Request                    | Poll HeyGen API for video render status          | Wait for Rendering                | If: Video Completed?                         | Step 3 — Create AI Avatar Video                 |
| If: Video Completed?               | If                              | Branch for completed or pending video           | HeyGen: Check Video Status         | HTTP: Download Final Video / Wait for Rendering | Step 3 — Create AI Avatar Video                 |
| HTTP: Download Final Video         | HTTP Request                    | Download the completed video                     | If: Video Completed?              | Google Drive: Upload Final Video             | Step 3 — Create AI Avatar Video                 |
| Google Drive: Upload Final Video    | Google Drive                   | Upload final video to Drive                       | HTTP: Download Final Video         | Google Sheets: Save Final Video URL           | Step 3 — Create AI Avatar Video                 |
| Google Sheets: Save Final Video URL | Google Sheets                  | Save final video URL to sheet                     | Google Drive: Upload Final Video   | Google Sheets - Read Post Data               | Step 3 — Create AI Avatar Video                 |
| Google Sheets - Read Post Data      | Google Sheets                  | Read post metadata including caption & title    | Google Sheets: Save Final Video URL | Get Google Drive ID                          | Step 4 — Auto-Publish to 9 Social Platforms     |
| Get Google Drive ID                | Set                             | Extract Google Drive file ID from URL            | Google Sheets - Read Post Data     | Upload Video to BLOTATO                      | Step 4 — Auto-Publish to 9 Social Platforms     |
| Upload Video to BLOTATO            | Blotato                        | Upload video to Blotato platform                   | Get Google Drive ID                | Social platform nodes (Tiktok, Linkedin, etc.) | Step 4 — Auto-Publish to 9 Social Platforms     |
| Tiktok                           | Blotato                        | Publish video to TikTok                            | Upload Video to BLOTATO            | Merge                                       | Step 4 — Auto-Publish to 9 Social Platforms     |
| Linkedin                         | Blotato                        | Publish video to LinkedIn                          | Upload Video to BLOTATO            | Merge                                       | Step 4 — Auto-Publish to 9 Social Platforms     |
| Facebook                        | Blotato                        | Publish video to Facebook                           | Upload Video to BLOTATO            | Merge                                       | Step 4 — Auto-Publish to 9 Social Platforms     |
| Instagram                       | Blotato                        | Publish video to Instagram (Reel)                  | Upload Video to BLOTATO            | Merge                                       | Step 4 — Auto-Publish to 9 Social Platforms     |
| Twitter (X)                     | Blotato                        | Publish video to Twitter (X)                        | Upload Video to BLOTATO            | Merge                                       | Step 4 — Auto-Publish to 9 Social Platforms     |
| Youtube                        | Blotato                        | Publish video to YouTube (private)                  | Upload Video to BLOTATO            | Merge                                       | Step 4 — Auto-Publish to 9 Social Platforms     |
| Threads                        | Blotato                        | Publish video to Threads                             | Upload Video to BLOTATO            | Merge                                       | Step 4 — Auto-Publish to 9 Social Platforms     |
| Bluesky                        | Blotato                        | Publish video to Bluesky                             | Upload Video to BLOTATO            | Merge                                       | Step 4 — Auto-Publish to 9 Social Platforms     |
| Pinterest                      | Blotato                        | Publish video to Pinterest                           | Upload Video to BLOTATO            | Merge                                       | Step 4 — Auto-Publish to 9 Social Platforms     |
| Merge                          | Merge                          | Combine all social platform outputs                  | Social platform nodes             | Google Sheets - Update Status                | Step 4 — Auto-Publish to 9 Social Platforms     |
| Google Sheets - Update Status     | Google Sheets                  | Mark post status as "Posted"                         | Merge                            | Telegram: Send Post Confirmation             | Step 4 — Auto-Publish to 9 Social Platforms     |
| Telegram: Send Post Confirmation   | Telegram                       | Notify Telegram user of successful posting           | Google Sheets - Update Status     | Telegram: Send Final Video                    | Step 4 — Auto-Publish to 9 Social Platforms     |
| Telegram: Send Final Video         | Telegram                       | Send the final video back to Telegram chat           | Telegram: Send Post Confirmation  | —                                            | Step 4 — Auto-Publish to 9 Social Platforms     |
| AI Agent - Generate Title & Caption | LangChain Agent               | Generate video title and caption from transcription | OpenAI Model GPT-5, LangChain - Think Tool | Google Sheets - Update Title & Caption      | Step 2 — Generate Title & Caption with GPT‑5    |
| OpenAI Model GPT-5              | LangChain LM Chat OpenAI         | GPT-5 language model for content generation          | —                                 | AI Agent - Generate Title & Caption          | Step 2 — Generate Title & Caption with GPT‑5    |
| LangChain - Think Tool          | LangChain Tool                   | Intermediate thought processing for LangChain agent  | —                                 | AI Agent - Generate Title & Caption          | Step 2 — Generate Title & Caption with GPT‑5    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Type: Telegram Trigger  
   - Listen for "message" updates specifically voice messages.  
   - Configure Telegram credentials with OAuth token.  

2. **Add Telegram Download Voice Node**  
   - Type: Telegram node  
   - Input: voice file ID from trigger message.  
   - Download voice audio as binary.  
   - Use same Telegram credentials.

3. **Add Google Drive Upload Voice Node**  
   - Type: Google Drive node  
   - Upload the downloaded audio file.  
   - Use OAuth2 Google Drive credentials.  
   - Set filename to Telegram voice message unique ID.  
   - Specify target Drive and Folder IDs.  

4. **Add TmpFiles Upload Voice Node**  
   - Type: HTTP Request (POST multipart-form-data)  
   - Upload the downloaded binary voice file to tmpfiles.org.  
   - Extract the returned public URL, modifying it to direct download link (replace 'org/' by 'org/dl/').  

5. **Add OpenAI Transcribe Node**  
   - Type: OpenAI transcribe via LangChain node  
   - Input: voice audio binary from Telegram download.  
   - Set language to French, temperature 0.  
   - Use OpenAI API credentials.

6. **Add Google Sheets Log Voice Metadata Node**  
   - Type: Google Sheets node (appendOrUpdate)  
   - Log voice file unique ID and URLs to a Google Sheet.  
   - Use Google Sheets OAuth2 credentials.  
   - Configure document and sheet IDs.

7. **Add Google Sheets Read Avatar Config Node**  
   - Type: Google Sheets node (read)  
   - Read avatar IDs and configuration from a dedicated sheet.

8. **Add Set Node (ID Avatar)**  
   - Set variables:  
     - `id_avatar` from avatar config sheet.  
     - `voice_url` from TmpFiles public URL.  

9. **Add HeyGen Generate Avatar Video Node**  
   - HTTP POST to HeyGen API `/v2/video/generate`.  
   - Body JSON specifying avatar ID, voice audio URL, video dimensions 1280x720.  
   - Use HTTP Header Auth credentials (HeyGen API key).  

10. **Add Wait Node**  
    - Pause to allow video rendering (configurable time).  

11. **Add HeyGen Check Video Status Node**  
    - HTTP GET request to HeyGen API `/v1/video_status.get?video_id=...`.  
    - Poll video status.  

12. **Add If Node (Video Completed?)**  
    - Condition: video status equals "completed".  
    - True branch: proceed to download video.  
    - False branch: loop back to Wait node.  

13. **Add HTTP Download Final Video Node**  
    - Download the completed video file from HeyGen URL.  

14. **Add Google Drive Upload Final Video Node**  
    - Upload the downloaded video to Google Drive folder.  

15. **Add Google Sheets Save Final Video URL Node**  
    - Update spreadsheet with Google Drive video URL linked to voice message ID.  

16. **Add Google Sheets Read Post Data Node**  
    - Read post metadata (Title, Caption) for the voice ID.  

17. **Add Set Node (Get Google Drive ID)**  
    - Extract Google Drive file ID from final video URL for Blotato upload.  

18. **Add Upload Video to Blotato Node**  
    - Upload video via Blotato API using Google Drive URL.  
    - Use Blotato API credentials.  

19. **Add Blotato Social Media Publish Nodes**  
    - One node each for TikTok, LinkedIn, Facebook, Instagram (Reel), Twitter (X), YouTube (private), Threads, Bluesky, Pinterest.  
    - Configure each with platform, account IDs, post caption, and video URL from Blotato upload.  

20. **Add Merge Node**  
    - Combine all social media nodes' outputs.  

21. **Add Google Sheets Update Status Node**  
    - Update post status in Google Sheets to "Posted" for the voice ID.  

22. **Add Telegram Send Post Confirmation Node**  
    - Send confirmation message "Posted" to Telegram chat.  

23. **Add Telegram Send Final Video Node**  
    - Send the final posted video back to Telegram chat.  

24. **AI Content Generation Subflow**  
    - Add OpenAI GPT-5 Model Node (LangChain).  
    - Add LangChain Think Tool Node.  
    - Add AI Agent Node with system prompt to generate Title and Caption JSON from transcription text.  
    - Add Google Sheets Update Title & Caption Node to save AI output.  
    - Connect nodes in order: GPT-5 Model → Think Tool → AI Agent → Google Sheets update.

25. **Connect all nodes respecting the data flow described in section 2**, ensuring correct input/output connections and data mapping.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                               | Context or Link                                                                                          |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| Turn a simple idea into a viral video and auto-publish it across platforms using GPT-5, HeyGen, Blotato, Google Sheets, and n8n. Full tutorial video: [Watch on YouTube](https://youtu.be/J13Mvv_qGd0) Documentation with setup instructions, API config, and platform guides: [Open the full documentation on Notion](https://automatisation.notion.site/Blotato-2473d6550fd980e19983f69611a80a0d?source=copy_link) | General project description and setup references.                                                      |
| Requirements: Create a Blotato Pro account, generate API key, enable “Verified Community Nodes” in n8n, install Blotato node, duplicate Google Sheet template, make Google Drive folder public, and complete workflow steps.                                                                                                                                                                   | Setup prerequisites for successful workflow execution.                                                 |
| This workflow uses multiple external APIs and services (Telegram, OpenAI, HeyGen, Blotato, Google Drive/Sheets). Ensure all API credentials are valid and have sufficient quota.                                                                                                                                                                                                          | Credential management and quota monitoring recommendation.                                            |
| The workflow includes retry logic for video rendering by polling HeyGen’s API and waiting if the video is not yet ready. This prevents premature failures and ensures video availability before upload.                                                                                                                                                                                       | Reliability improvement note.                                                                          |
| The AI Agent enforces strict JSON output formatting for Title and Caption to facilitate automated updates to Google Sheets without manual parsing.                                                                                                                                                                                                                                       | Data integrity and parsing optimization.                                                              |

---

**Disclaimer:**  
The provided text is generated exclusively from an automated workflow built with n8n, respecting content policies. It contains no illegal or offensive elements. All processed data is legal and public.