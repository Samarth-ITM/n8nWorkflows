Create & Approve POV Videos with AI, ElevenLabs & Multi-Posting (TikTok/IG/YT)

https://n8nworkflows.xyz/workflows/create---approve-pov-videos-with-ai--elevenlabs---multi-posting--tiktok-ig-yt--4029


# Create & Approve POV Videos with AI, ElevenLabs & Multi-Posting (TikTok/IG/YT)

---

### 1. Workflow Overview

This workflow automates the end-to-end creation, approval, and multi-platform posting of TikTok-style POV (Point of View) videos, primarily targeting Instagram with cross-posting to Facebook and YouTube. It leverages AI models (OpenAI and OpenRouter), media generation APIs (PIAPI.ai for images/videos, ElevenLabs for audio), and video rendering (Creatomate). It manages content approval through email and Google Sheets integration, ensuring a smooth pipeline from idea generation to final publishing.

The workflow is logically divided into the following blocks:

- **1.1 Idea & Script Generation**: Generates daily video ideas, expands them into scripts, and prepares prompts for media creation.
- **1.2 Media Creation**: Produces images, videos, and audio assets using AI services.
- **1.3 Video Rendering**: Combines the generated media assets into a final video using Creatomate.
- **1.4 Storage & Notification**: Uploads final media to Google Drive, updates Google Sheets, and sends email notifications.
- **1.5 Approval Process**: Sends approval requests via email and handles approval/rejection responses.
- **1.6 Social Media Posting**: Posts approved videos to Instagram, Facebook, TikTok, and YouTube.
- **1.7 Scheduling & Polling**: Triggers workflow runs and polls for approved videos to post.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Idea & Script Generation

**Overview:**  
Generates daily POV video ideas using OpenAI, formats them, stores them in Google Sheets, and creates detailed scripts for media production.

**Nodes Involved:**  
- Schedule Trigger  
- Get Title  
- Generate Ideas  
- Item List Output Parser  
- Generate Topics  
- Format Row  
- Insert new Prompt, Caption and Title/Topic  
- Generate Script  
- Set Topics  

**Node Details:**

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Initiates daily idea generation workflow.  
  - Configuration: Runs once daily (default or user-configured).  
  - Inputs: None  
  - Outputs: Triggers "Get Title" node.  
  - Edge Cases: Missed triggers if n8n instance downtime.  

- **Get Title**  
  - Type: Google Sheets  
  - Role: Reads pending video titles/ideas from the "POV Videos" Google Sheet.  
  - Configuration: Reads from "Instagram" tab, selecting rows with status indicating pending ideas.  
  - Inputs: Trigger from Schedule Trigger  
  - Outputs: Sends data to "Generate Ideas".  
  - Edge Cases: Sheet access permission errors, empty rows.  

- **Generate Ideas**  
  - Type: Langchain Chain LLM (OpenAI)  
  - Role: Uses OpenAI to generate 3 POV video sequences based on input titles.  
  - Configuration: Prompt engineered to produce list items; uses OpenAI API credentials.  
  - Inputs: Titles from "Get Title"  
  - Outputs: Raw AI-generated text sent to "Item List Output Parser".  
  - Edge Cases: API rate limits, malformed outputs.  

- **Item List Output Parser**  
  - Type: Langchain Output Parser (Item List)  
  - Role: Parses AI output into structured list items.  
  - Inputs: Raw AI text from "Generate Ideas"  
  - Outputs: Parsed list sent to "Generate Script".  
  - Edge Cases: Parsing failures if AI output format changes.  

- **Generate Topics**  
  - Type: OpenAI node via Langchain  
  - Role: Uses OpenAI to create a new video idea topic.  
  - Inputs/Outputs: Triggered separately by "Schedule Trigger1" (see scheduling block).  
  - Edge Cases: Similar to Generate Ideas.  

- **Format Row**  
  - Type: Code node  
  - Role: Structures new video idea data into a format compatible with Google Sheets.  
  - Inputs: Output from "Generate Topics"  
  - Outputs: Structured row sent to "Insert new Prompt, Caption and Title/Topic".  
  - Edge Cases: Code errors or malformed data.  

- **Insert new Prompt, Caption and Title/Topic**  
  - Type: Google Sheets  
  - Role: Inserts new video idea rows into the Google Sheet "POV Videos".  
  - Inputs: Structured data from "Format Row"  
  - Outputs: None (side effect on sheet).  
  - Edge Cases: Sheet write permissions, conflicts with concurrent writes.  

- **Generate Script**  
  - Type: OpenAI node via Langchain  
  - Role: Expands selected POV sequence into a detailed video script.  
  - Inputs: Parsed sequences from "Item List Output Parser"  
  - Outputs: Script text sent to "Text-to-Image".  
  - Edge Cases: API errors, incomplete scripts.  

- **Set Topics**  
  - Type: Set node  
  - Role: Stores the generated script and topic data for downstream media creation.  
  - Inputs: Script data from "Generate Script"  
  - Outputs: Merges with media assets in "Merge" node.  
  - Edge Cases: Misalignment of data fields.  

---

#### 1.2 Media Creation

**Overview:**  
Generates images, videos, and audio assets required to produce the POV video.

**Nodes Involved:**  
- Text-to-Image  
- Wait for 1 Min  
- Get Image  
- Generate Video Prompt  
- Generate Video  
- Wait for 5 Mins  
- Access Videos  
- Store Video  
- Generate Sound Prompt  
- Text-to-Sound  
- 2 Min Wait  
- Store Sound  
- Allow Access  
- Audio Output  
- Limit  

**Node Details:**

- **Text-to-Image**  
  - Type: HTTP Request (PIAPI.ai)  
  - Role: Sends a prompt to PIAPI.ai to generate images based on the script.  
  - Configuration: Uses PIAPI.ai API credentials and prompt text from "Generate Script".  
  - Inputs: Script text.  
  - Outputs: Queues image generation; triggers "Wait for 1 Min".  
  - Edge Cases: API errors, generation delays.  

- **Wait for 1 Min**  
  - Type: Wait node  
  - Role: Pauses workflow to allow image generation completion.  
  - Inputs: From "Text-to-Image"  
  - Outputs: Triggers "Get Image".  

- **Get Image**  
  - Type: HTTP Request  
  - Role: Fetches the generated image URL or file from PIAPI.ai after generation delay.  
  - Inputs: After wait.  
  - Outputs: Passes image URL to "Generate Video Prompt".  
  - Edge Cases: Image not ready, timeout errors.  

- **Generate Video Prompt**  
  - Type: OpenAI via Langchain  
  - Role: Creates a video generation prompt based on the image.  
  - Inputs: Image URL or metadata.  
  - Outputs: Prompt for video generation sent to "Generate Video".  
  - Edge Cases: API failures.  

- **Generate Video**  
  - Type: HTTP Request (PIAPI.ai)  
  - Role: Requests a 5-second video generation from PIAPI.ai using prompt.  
  - Inputs: Video prompt.  
  - Outputs: Triggers "Wait for 5 Mins" for rendering.  
  - Edge Cases: Video generation failures or timeouts.  

- **Wait for 5 Mins**  
  - Type: Wait node  
  - Role: Waits for video to be processed.  
  - Inputs: From "Generate Video".  
  - Outputs: Triggers "Access Videos".  

- **Access Videos**  
  - Type: HTTP Request  
  - Role: Retrieves the generated video URL.  
  - Inputs: After wait.  
  - Outputs: Passes video URL to "Store Video" and "Merge" node.  
  - Edge Cases: Video not ready, API errors.  

- **Store Video**  
  - Type: Google Sheets  
  - Role: Updates the Google Sheet with video URL and metadata.  
  - Inputs: Video URL.  
  - Outputs: Triggers "Generate Sound Prompt".  
  - Edge Cases: Sheet write errors.  

- **Generate Sound Prompt**  
  - Type: Langchain Agent (OpenRouter)  
  - Role: Generates an audio prompt for ElevenLabs based on the script.  
  - Inputs: Script and video metadata.  
  - Outputs: Audio prompt sent to "Audio Output".  
  - Edge Cases: API failures.  

- **Audio Output**  
  - Type: Set node  
  - Role: Sets audio prompt data for text-to-speech conversion.  
  - Inputs: From "Generate Sound Prompt".  
  - Outputs: Triggers "Limit".  

- **Limit**  
  - Type: Limit node  
  - Role: Controls concurrency for audio generation requests.  
  - Inputs: Audio prompt.  
  - Outputs: Triggers "Text-to-Sound".  

- **Text-to-Sound**  
  - Type: HTTP Request (ElevenLabs)  
  - Role: Sends audio prompt to ElevenLabs API to generate a 20-second audio clip.  
  - Inputs: Audio prompt.  
  - Outputs: Triggers "2 Min Wait".  
  - Edge Cases: API limits, audio generation errors.  

- **2 Min Wait**  
  - Type: Wait node  
  - Role: Waits for audio generation to complete.  
  - Inputs: From "Text-to-Sound".  
  - Outputs: Triggers "Store Sound".  

- **Store Sound**  
  - Type: Google Drive  
  - Role: Uploads generated audio clip to Google Drive in designated folder.  
  - Inputs: Audio file from ElevenLabs.  
  - Outputs: Triggers "Allow Access".  
  - Edge Cases: Upload failures, permission errors.  

- **Allow Access**  
  - Type: Google Drive  
  - Role: Sets sharing permissions on the uploaded audio file for workflow access.  
  - Inputs: Uploaded audio file metadata.  
  - Outputs: Triggers "Merge" node for final data assembly.  
  - Edge Cases: Permission errors.  

---

#### 1.3 Video Rendering

**Overview:**  
Combines script, video, and audio assets into a final video using Creatomate API, monitors render status, and manages success or failure paths.

**Nodes Involved:**  
- Merge  
- List Elements  
- Rendor with Creatomate  
- Wait for Render  
- Get Final Video  
- Check Video Status  
- New Render Video Alert  
- Failed Render  
- Google Drive  
- Render Video Link  

**Node Details:**

- **Merge**  
  - Type: Merge node  
  - Role: Combines script, video, and audio data into a single payload.  
  - Inputs: From "Set Topics", "Access Videos", and "Allow Access".  
  - Outputs: Passes merged data to "List Elements".  
  - Edge Cases: Data mismatch or missing fields.  

- **List Elements**  
  - Type: Code node  
  - Role: Formats merged data into Creatomate’s template JSON structure.  
  - Inputs: Merged media/script data.  
  - Outputs: JSON passed to "Rendor with Creatomate".  
  - Edge Cases: Formatting errors causing rendering failures.  

- **Rendor with Creatomate**  
  - Type: HTTP Request (Creatomate API)  
  - Role: Sends formatted JSON template to Creatomate to initiate video rendering.  
  - Inputs: JSON template from "List Elements".  
  - Outputs: Job ID received, triggers "Wait for Render".  
  - Edge Cases: API errors, authentication failures.  

- **Wait for Render**  
  - Type: Wait node with webhook  
  - Role: Waits for a callback or a set time before querying render status.  
  - Inputs: From "Rendor with Creatomate".  
  - Outputs: Triggers "Get Final Video".  

- **Get Final Video**  
  - Type: HTTP Request  
  - Role: Polls Creatomate API for render result status and video URL.  
  - Inputs: Job ID from render request.  
  - Outputs: Passes status to "Check Video Status".  
  - Edge Cases: Timeout, API errors.  

- **Check Video Status**  
  - Type: Switch node  
  - Role: Routes workflow based on render success, failure, or pending status.  
  - Inputs: Render status data.  
  - Outputs:  
    - On success: "New Render Video Alert" and "Google Drive" nodes.  
    - On failure: "Failed Render" node.  
    - On pending: Re-triggers "Wait for Render".  
  - Edge Cases: Unrecognized statuses.  

- **New Render Video Alert**  
  - Type: Gmail  
  - Role: Sends email notification on successful render.  
  - Inputs: Success path from "Check Video Status".  
  - Outputs: Triggers "Render Video Link".  

- **Failed Render**  
  - Type: Gmail  
  - Role: Sends failure notification email.  
  - Inputs: Failure path from "Check Video Status".  
  - Outputs: None.  

- **Google Drive**  
  - Type: Google Drive  
  - Role: Uploads the rendered final video to Google Drive folder.  
  - Inputs: Video URL from "Get Final Video".  
  - Outputs: None.  

- **Render Video Link**  
  - Type: Google Sheets  
  - Role: Updates the video’s final link and status in Google Sheet.  
  - Inputs: Success notification trigger.  
  - Outputs: Triggers "Approval Email" node.  

---

#### 1.4 Storage & Notification

**Overview:**  
Manages storage of final media assets and sends email notifications for approvals and status updates.

**Nodes Involved:**  
- Google Drive  
- New Render Video Alert  
- Failed Render  
- Render Video Link  
- Approval Email  

**Node Details:**

- **Google Drive**  
  - See above in rendering.  

- **New Render Video Alert**  
  - See above in rendering.  

- **Failed Render**  
  - See above in rendering.  

- **Render Video Link**  
  - See above in rendering.  

- **Approval Email**  
  - Type: Gmail  
  - Role: Sends an email containing approval and rejection links for the generated video.  
  - Configuration: Uses Gmail OAuth2 credentials, custom HTML with embedded webhook URLs for approval.  
  - Inputs: Triggered after video link update.  
  - Outputs: None.  
  - Edge Cases: Email sending failures, invalid links.  

---

#### 1.5 Approval Process

**Overview:**  
Processes approval or rejection responses received via webhook, updates Google Sheet accordingly.

**Nodes Involved:**  
- Handle Approval/Rejection1  
- Video Update1  
- Check Approval  
- Update Google Sheet  
- Mark Rejected  

**Node Details:**

- **Handle Approval/Rejection1**  
  - Type: Webhook  
  - Role: Receives approval/rejection events from email links.  
  - Configuration: Public webhook URL embedded in approval emails.  
  - Inputs: HTTP request with approval status.  
  - Outputs: Passes approval data to "Video Update1".  
  - Edge Cases: Invalid requests, security concerns.  

- **Video Update1**  
  - Type: Google Sheets  
  - Role: Updates approval status and metadata in Google Sheet.  
  - Inputs: Approval data from webhook.  
  - Outputs: None.  

- **Check Approval**  
  - Type: If node  
  - Role: Checks if video is approved or rejected based on sheet data.  
  - Inputs: Latest row data.  
  - Outputs:  
    - Approved path triggers posting nodes and sheet updates.  
    - Rejected path triggers "Mark Rejected".  

- **Update Google Sheet**  
  - Type: Google Sheets  
  - Role: Updates publish status and posting metadata after social media posting.  
  - Inputs: Post results.  

- **Mark Rejected**  
  - Type: Google Sheets  
  - Role: Marks videos as rejected in the sheet to stop further processing.  
  - Inputs: From rejected branch of "Check Approval".  

---

#### 1.6 Social Media Posting

**Overview:**  
Posts approved videos to Instagram, Facebook, TikTok, and YouTube using respective APIs.

**Nodes Involved:**  
- Get Latest Approved Video  
- Reverse Rows  
- Get Newest Row  
- Instagram Container  
- Post to Instagram  
- Facebook Posts  
- Download Video  
- Post YouTube  
- Get Bin Video  
- Tiktok Post  

**Node Details:**

- **Get Latest Approved Video**  
  - Type: Google Sheets Trigger  
  - Role: Polls Google Sheet for newly approved videos to post.  
  - Inputs: Scheduled trigger (7 PM daily).  
  - Outputs: Triggers "Reverse Rows".  

- **Reverse Rows**  
  - Type: Code node  
  - Role: Reverses order of rows to process newest first.  
  - Inputs: Sheet data.  
  - Outputs: Passes to "Get Newest Row".  

- **Get Newest Row**  
  - Type: Code node  
  - Role: Selects the newest approved row for posting.  
  - Inputs: Reversed rows.  
  - Outputs: Passes to "Check Approval".  

- **Instagram Container**  
  - Type: Facebook Graph API  
  - Role: Creates a media container for Instagram video upload.  
  - Configuration: Uses Instagram Business Account ID and access token.  
  - Inputs: Approved video metadata.  
  - Outputs: Container ID to "Post to Instagram".  
  - Edge Cases: API permission errors, token expiration.  

- **Post to Instagram**  
  - Type: Facebook Graph API  
  - Role: Publishes the video to Instagram feed.  
  - Inputs: Container ID from previous node.  
  - Outputs: Post ID for status tracking.  

- **Facebook Posts**  
  - Type: Facebook Graph API  
  - Role: Posts the video to Facebook page.  
  - Inputs: Video URL and metadata.  
  - Outputs: Confirmation status.  

- **Download Video**  
  - Type: HTTP Request  
  - Role: Downloads final video from Google Drive or Creatomate URL for YouTube upload.  
  - Inputs: Video URL.  
  - Outputs: Binary video data to "Post YouTube".  
  - Edge Cases: Download failures, slow speeds.  

- **Post YouTube**  
  - Type: YouTube node  
  - Role: Uploads video to YouTube channel.  
  - Inputs: Video file from "Download Video".  
  - Outputs: Video ID and status.  
  - Edge Cases: OAuth token expiry, API quota.  

- **Get Bin Video**  
  - Type: HTTP Request  
  - Role: Retrieves video binary for TikTok posting.  
  - Inputs: Video URL.  
  - Outputs: Binary video data to "Tiktok Post".  

- **Tiktok Post**  
  - Type: HTTP Request  
  - Role: Posts video to TikTok (via API or third-party).  
  - Inputs: Video binary from "Get Bin Video".  
  - Outputs: Confirmation response.  
  - Edge Cases: TikTok API limitations or authentication.  

---

#### 1.7 Scheduling & Polling

**Overview:**  
Schedules daily workflow runs and polls for approved videos to post.

**Nodes Involved:**  
- Schedule Trigger  
- Schedule Trigger1  
- Get Latest Approved Video  

**Node Details:**

- **Schedule Trigger**  
  - See 1.1 block. Initiates idea generation.  

- **Schedule Trigger1**  
  - Type: Schedule Trigger  
  - Role: Triggers "Generate Topics" for additional topic generation.  
  - Configuration: Runs daily or as configured.  

- **Get Latest Approved Video**  
  - See 1.6 block. Polls for posting.  

---

### 3. Summary Table

| Node Name                   | Node Type                             | Functional Role                          | Input Node(s)                           | Output Node(s)                              | Sticky Note                                                                                                         |
|-----------------------------|-------------------------------------|----------------------------------------|---------------------------------------|---------------------------------------------|---------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger            | Schedule Trigger                    | Starts daily workflow                   | None                                  | Get Title                                  |                                                                                                                     |
| Get Title                  | Google Sheets                      | Reads pending video ideas               | Schedule Trigger                      | Generate Ideas                              |                                                                                                                     |
| Generate Ideas             | Langchain Chain LLM (OpenAI)       | Generates 3 POV sequences               | Get Title                            | Item List Output Parser                      |                                                                                                                     |
| Item List Output Parser    | Langchain Output Parser (Item List) | Parses AI output                        | Generate Ideas                       | Generate Script                             |                                                                                                                     |
| Generate Topics            | Langchain OpenAI                   | Creates new video idea topic            | Schedule Trigger1                    | Format Row                                  |                                                                                                                     |
| Format Row                 | Code                              | Formats idea for Google Sheet           | Generate Topics                     | Insert new Prompt, Caption and Title/Topic |                                                                                                                     |
| Insert new Prompt, Caption and Title/Topic | Google Sheets            | Inserts new video idea row              | Format Row                         | None                                        |                                                                                                                     |
| Generate Script            | Langchain OpenAI                   | Expands sequence into detailed script  | Item List Output Parser              | Text-to-Image                               |                                                                                                                     |
| Set Topics                 | Set                               | Stores script and topic data            | Generate Script                    | Merge                                       |                                                                                                                     |
| Text-to-Image              | HTTP Request (PIAPI.ai)             | Generates image from script             | Generate Script                    | Wait for 1 Min                              |                                                                                                                     |
| Wait for 1 Min             | Wait                              | Waits for image generation              | Text-to-Image                     | Get Image                                   |                                                                                                                     |
| Get Image                  | HTTP Request                      | Retrieves generated image               | Wait for 1 Min                    | Generate Video Prompt                        |                                                                                                                     |
| Generate Video Prompt      | Langchain OpenAI                   | Creates video prompt from image         | Get Image                        | Generate Video                              |                                                                                                                     |
| Generate Video             | HTTP Request (PIAPI.ai)             | Generates 5-second video                | Generate Video Prompt             | Wait for 5 Mins                             |                                                                                                                     |
| Wait for 5 Mins            | Wait                              | Waits for video rendering               | Generate Video                   | Access Videos                               |                                                                                                                     |
| Access Videos              | HTTP Request                      | Retrieves generated video URL           | Wait for 5 Mins                 | Store Video, Merge                           |                                                                                                                     |
| Store Video                | Google Sheets                      | Updates sheet with video URL            | Access Videos                   | Generate Sound Prompt                        |                                                                                                                     |
| Generate Sound Prompt      | Langchain Agent (OpenRouter)        | Creates audio prompt                     | Store Video                     | Audio Output                                |                                                                                                                     |
| Audio Output               | Set                               | Prepares audio prompt data               | Generate Sound Prompt           | Limit                                       |                                                                                                                     |
| Limit                     | Limit                             | Controls concurrency for audio          | Audio Output                   | Text-to-Sound                               |                                                                                                                     |
| Text-to-Sound             | HTTP Request (ElevenLabs)            | Generates audio clip                     | Limit                         | 2 Min Wait                                  |                                                                                                                     |
| 2 Min Wait                | Wait                              | Waits for audio generation completion  | Text-to-Sound                 | Store Sound                                 |                                                                                                                     |
| Store Sound               | Google Drive                      | Uploads audio to Drive                   | 2 Min Wait                   | Allow Access                                |                                                                                                                     |
| Allow Access              | Google Drive                      | Sets Drive sharing permissions          | Store Sound                 | Merge                                       |                                                                                                                     |
| Merge                     | Merge                             | Combines script, video, audio data      | Set Topics, Access Videos, Allow Access | List Elements                              |                                                                                                                     |
| List Elements             | Code                              | Formats data for Creatomate              | Merge                        | Rendor with Creatomate                       |                                                                                                                     |
| Rendor with Creatomate    | HTTP Request (Creatomate)           | Sends video rendering request            | List Elements                | Wait for Render                             |                                                                                                                     |
| Wait for Render           | Wait                              | Waits for render completion             | Rendor with Creatomate        | Get Final Video                             |                                                                                                                     |
| Get Final Video           | HTTP Request                      | Polls render status and gets final URL  | Wait for Render              | Check Video Status                          |                                                                                                                     |
| Check Video Status        | Switch                           | Routes success/failure/pending paths    | Get Final Video              | New Render Video Alert, Google Drive, Failed Render, Wait for Render |                                                                                                                     |
| New Render Video Alert    | Gmail                            | Sends success email notification         | Check Video Status (success) | Render Video Link                           |                                                                                                                     |
| Failed Render             | Gmail                            | Sends failure email notification         | Check Video Status (failure) | None                                        |                                                                                                                     |
| Google Drive              | Google Drive                     | Uploads final video                      | Check Video Status (success) | None                                        |                                                                                                                     |
| Render Video Link         | Google Sheets                    | Updates Google Sheet with final video URL | New Render Video Alert       | Approval Email                              |                                                                                                                     |
| Approval Email            | Gmail                            | Sends video approval request email       | Render Video Link            | None                                        | Embedded approval/rejection links; uses Gmail OAuth2 credentials.                                                  |
| Handle Approval/Rejection1 | Webhook                         | Receives approval/rejection responses    | None                        | Video Update1                               | Webhook endpoint for email approval links.                                                                         |
| Video Update1             | Google Sheets                   | Updates approval status in sheet         | Handle Approval/Rejection1   | None                                        |                                                                                                                     |
| Check Approval            | If                              | Routes based on approval status          | Get Newest Row               | Update Google Sheet, Facebook Posts, Instagram Container, Download Video, Get Bin Video, Mark Rejected |                                                                                                                     |
| Update Google Sheet       | Google Sheets                   | Updates publish status after posting     | Check Approval (approved)    | None                                        |                                                                                                                     |
| Mark Rejected             | Google Sheets                   | Marks videos as rejected                  | Check Approval (rejected)    | None                                        |                                                                                                                     |
| Get Latest Approved Video | Google Sheets Trigger           | Polls for approved videos to post        | Schedule Trigger (via schedule) | Reverse Rows                              |                                                                                                                     |
| Reverse Rows              | Code                            | Reverses order of rows                    | Get Latest Approved Video    | Get Newest Row                              |                                                                                                                     |
| Get Newest Row            | Code                            | Selects newest approved video             | Reverse Rows                | Check Approval                              |                                                                                                                     |
| Instagram Container       | Facebook Graph API              | Creates Instagram media container         | Check Approval (approved)   | Post to Instagram                           | Requires Instagram Business Account ID and token.                                                                   |
| Post to Instagram         | Facebook Graph API              | Publishes video to Instagram feed         | Instagram Container         | None                                        |                                                                                                                     |
| Facebook Posts            | Facebook Graph API              | Posts video to Facebook page               | Check Approval (approved)   | None                                        |                                                                                                                     |
| Download Video            | HTTP Request                   | Downloads video for YouTube upload         | Check Approval (approved)   | Post YouTube                                |                                                                                                                     |
| Post YouTube              | YouTube                        | Uploads video to YouTube channel           | Download Video              | None                                        | Requires YouTube OAuth2 credentials.                                                                                 |
| Get Bin Video             | HTTP Request                   | Retrieves video binary for TikTok          | Check Approval (approved)   | Tiktok Post                                 |                                                                                                                     |
| Tiktok Post               | HTTP Request                   | Posts video to TikTok                       | Get Bin Video               | None                                        |                                                                                                                     |
| Schedule Trigger1         | Schedule Trigger              | Triggers new topic generation daily        | None                       | Generate Topics                             |                                                                                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node** named "Schedule Trigger" to run daily; no parameters needed.

2. **Add a Google Sheets node "Get Title"** configured to read rows from the "POV Videos" sheet, "Instagram" tab, filtering for pending video ideas.

3. **Add a Langchain Chain LLM node "Generate Ideas"** using OpenAI credentials to generate 3 POV sequences based on titles from "Get Title".

4. **Add an Output Parser node "Item List Output Parser"** to parse AI-generated text into list items.

5. **Add a Langchain OpenAI node "Generate Script"** to expand sequences into detailed scripts.

6. **Add a Set node "Set Topics"** to store and prepare script data for media creation.

7. **Insert an HTTP Request node "Text-to-Image"** calling PIAPI.ai API with the script prompt to generate images; use PIAPI.ai credentials.

8. **Add a Wait node "Wait for 1 Min"** to delay for image generation.

9. **Add an HTTP Request node "Get Image"** to fetch generated image URL from PIAPI.ai.

10. **Add a Langchain OpenAI node "Generate Video Prompt"** to create a video prompt based on the image.

11. **Add an HTTP Request node "Generate Video"** to request a 5-second video from PIAPI.ai using the prompt; use PIAPI.ai credentials.

12. **Add a Wait node "Wait for 5 Mins"** to allow video rendering.

13. **Add an HTTP Request node "Access Videos"** to retrieve video URL.

14. **Add a Google Sheets node "Store Video"** to update the video URL in the Google Sheet.

15. **Add a Langchain Agent node "Generate Sound Prompt"** using OpenRouter to create audio prompts from the script.

16. **Add a Set node "Audio Output"** to prepare the audio prompt data.

17. **Add a Limit node "Limit"** configured to control concurrency for audio generation.

18. **Add an HTTP Request node "Text-to-Sound"** to send prompts to ElevenLabs API; use ElevenLabs credentials.

19. **Add a Wait node "2 Min Wait"** to allow audio generation.

20. **Add a Google Drive node "Store Sound"** to upload audio to designated Drive folder; use Google Drive OAuth2 credentials.

21. **Add a Google Drive node "Allow Access"** to set sharing permissions on the audio file.

22. **Add a Merge node "Merge"** to combine script, video, and audio data.

23. **Add a Code node "List Elements"** to format merged data into Creatomate template JSON.

24. **Add an HTTP Request node "Rendor with Creatomate"** to send the template to Creatomate API; use Creatomate credentials.

25. **Add a Wait node "Wait for Render"** with webhook to pause until rendering completes.

26. **Add an HTTP Request node "Get Final Video"** to poll rendering status.

27. **Add a Switch node "Check Video Status"** to route on success, failure, or pending status.

28. **Add Gmail nodes "New Render Video Alert" and "Failed Render"** to send success/failure emails; configure Gmail OAuth2 credentials.

29. **Add a Google Drive node "Google Drive"** to upload rendered video.

30. **Add a Google Sheets node "Render Video Link"** to update sheet with final video URL.

31. **Add a Gmail node "Approval Email"** to send approval request emails containing webhook links; configure Gmail OAuth2.

32. **Add a Webhook node "Handle Approval/Rejection1"** receiving approval/rejection HTTP requests.

33. **Add a Google Sheets node "Video Update1"** to update approval status in sheet.

34. **Add a Google Sheets Trigger node "Get Latest Approved Video"** to poll approved videos daily at 7 PM.

35. **Add Code nodes "Reverse Rows" and "Get Newest Row"** to select newest approved video.

36. **Add an If node "Check Approval"** to route between approved and rejected paths.

37. **Add Google Sheets nodes "Update Google Sheet" and "Mark Rejected"** to update publish status or mark rejection.

38. **Add Facebook Graph API nodes "Instagram Container", "Post to Instagram", and "Facebook Posts"** with respective credentials and IDs.

39. **Add HTTP Request nodes "Download Video" and "Get Bin Video"** to retrieve video binary for YouTube and TikTok.

40. **Add YouTube node "Post YouTube"** for video upload; configure YouTube OAuth2.

41. **Add HTTP Request node "Tiktok Post"** to post video to TikTok.

42. **Add a second Schedule Trigger "Schedule Trigger1"** to trigger "Generate Topics" daily.

---

### 5. General Notes & Resources

| Note Content                                                                                                                       | Context or Link                                                                                                                             |
|-----------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------|
| Workflow automates TikTok-style Instagram POV video creation and multi-platform posting using AI and API integrations.           | Workflow description and purpose.                                                                                                          |
| Requires API credentials: OpenAI, PIAPI.ai, ElevenLabs, Creatomate, Google Sheets/Drive, Gmail, Instagram Graph, Facebook Graph, YouTube. | Credential setup instructions.                                                                                                             |
| Google Sheet "POV Videos" structure is critical: includes columns such as `Timestamp`, `POV_Status`, `Approval`, `Publish_Status`. | Data tracking and workflow state management.                                                                                              |
| Creatomate template must be pre-configured with placeholders for script, video, and audio elements.                              | Video rendering requires correctly formatted JSON input.                                                                                   |
| Approval emails contain webhook links for accept/reject functionality, enabling asynchronous quality control.                     | Approval process integration.                                                                                                              |
| Schedule Trigger nodes run daily to automate idea generation and approval polling.                                                | Scheduling and polling workflow triggers.                                                                                                 |
| Video posting nodes require valid and current access tokens with appropriate API permissions.                                    | OAuth2 and API permissions management is essential for social media posting.                                                               |
| Error handling includes email notifications on rendering failures and conditional routing on approval status.                   | Ensures workflow robustness and operator alerting.                                                                                        |
| For customization, users can add platforms, modify email templates, or adjust video/audio duration via node parameters.         | Workflow extensibility.                                                                                                                    |
| ![POV Videos Automation.png](fileId:1313)                                                                                       | Workflow architecture visual reference (if accessible).                                                                                   |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

---