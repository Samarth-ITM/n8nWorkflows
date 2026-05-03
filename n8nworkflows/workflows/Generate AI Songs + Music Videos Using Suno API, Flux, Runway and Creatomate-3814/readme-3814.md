Generate AI Songs + Music Videos Using Suno API, Flux, Runway and Creatomate

https://n8nworkflows.xyz/workflows/generate-ai-songs---music-videos-using-suno-api--flux--runway-and-creatomate-3814


# Generate AI Songs + Music Videos Using Suno API, Flux, Runway and Creatomate

### 1. Workflow Overview

This workflow automates the generation of AI-powered music tracks, cover art, and fully rendered music videos, triggered via Telegram chat and managed through Google Sheets. It is designed for creators who want to submit song ideas easily and receive polished multimedia outputs without manual intervention.

The workflow is logically divided into three modular blocks:

- **1.1 Input Reception & Logging:** Handles user input from Telegram (text or voice), transcribes voice notes, and logs song ideas into Google Sheets.
- **1.2 AI Content Generation:** Generates lyrics, music tracks, cover images, and video backgrounds using multiple AI models and APIs, while managing generation status and rate limits.
- **1.3 Final Rendering & Delivery:** Merges audio and video, uploads final assets to Google Drive, updates Google Sheets with URLs, and sends the final video link back to the Telegram user.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Logging

**Overview:**  
This block captures user song ideas from Telegram, supports both text and voice input, transcribes voice notes, and stores the ideas along with user chat IDs in Google Sheets for tracking and later processing.

**Nodes Involved:**  
- Track Ideas Agent Telegram Trigger  
- Switch Text or Voice Input  
- Download Voice Note  
- Transcribe Voice Note  
- Set Field  
- Set Field to "text"  
- Append Music Tracks to GSheets  

**Node Details:**

- **Track Ideas Agent Telegram Trigger**  
  - Type: Telegram Trigger  
  - Role: Entry point for receiving user messages (text or voice) from Telegram bot.  
  - Config: Listens for new messages in the bot’s chat.  
  - Inputs: Telegram user messages.  
  - Outputs: Routes messages to the switch node.  
  - Edge Cases: Telegram API rate limits, malformed messages.

- **Switch Text or Voice Input**  
  - Type: Switch  
  - Role: Determines if incoming message is text or voice note.  
  - Config: Checks message type field.  
  - Inputs: Telegram Trigger output.  
  - Outputs: Routes to either "Set Field to 'text'" or "Download Voice Note".  
  - Edge Cases: Unsupported message types, missing voice note URLs.

- **Download Voice Note**  
  - Type: Telegram node (Download)  
  - Role: Downloads voice note audio file from Telegram servers.  
  - Inputs: Voice note message metadata.  
  - Outputs: Audio file binary data.  
  - Edge Cases: Telegram file expiration, download failures.

- **Transcribe Voice Note**  
  - Type: OpenAI Node (Speech-to-Text)  
  - Role: Converts voice note audio to text transcription.  
  - Config: Uses OpenAI Whisper or equivalent model.  
  - Inputs: Audio file from Download Voice Note.  
  - Outputs: Transcribed text.  
  - Edge Cases: Audio quality issues, API timeouts.

- **Set Field**  
  - Type: Set  
  - Role: Normalizes data structure by setting transcribed text into a consistent field for downstream processing.  
  - Inputs: Transcribed text.  
  - Outputs: Structured text data.  
  - Edge Cases: Empty transcriptions.

- **Set Field to "text"**  
  - Type: Set  
  - Role: Prepares text input from Telegram for AI processing, ensuring consistent field naming.  
  - Inputs: Text messages from Telegram.  
  - Outputs: Structured text data.  
  - Edge Cases: Empty messages.

- **Append Music Tracks to GSheets**  
  - Type: Google Sheets Tool  
  - Role: Logs the user’s song idea and metadata into a Google Sheet for tracking.  
  - Inputs: Structured text data from Set nodes.  
  - Outputs: Confirmation of row append.  
  - Edge Cases: Google Sheets API quota limits, permission errors.

---

#### 2.2 AI Content Generation

**Overview:**  
This block processes logged song ideas to generate lyrics, music tracks, cover images, and video backgrounds using AI models and APIs. It manages asynchronous generation status checks, rate limiting, and uploads intermediate assets to Google Drive.

**Nodes Involved:**  
- Google Sheets Trigger: Start Processing Tracks  
- Get one Pending Row  
- Lyrics AI Agent  
- OpenAI Chat Model1  
- Structured Output Parser  
- Music Generation API Request  
- Wait  
- Get Music Generation Status  
- Confirm Generation Status  
- Download Audio Track  
- Upload Audio Track to Drive  
- Update Audio URL in GSheet  
- Cover Image and Video Prompts AI Agent  
- Google Gemini Chat Model  
- Structured Output Parser1  
- Generate Cover Image 1:1  
- Generate Cover Image 3:1  
- Wait1  
- Wait2  
- get_image_url1  
- Get Image  
- Upload Cover Image 1:1 to Drive  
- get_image_url  
- Get Image1  
- Upload to kraken to get url  
- Convert Cover Image 3:1 to Video  
- Get Video Generation Status  
- Confirm Generation Status1  
- Download Video  
- Upload Converted Video to Drive  
- Update Cover Photo & Video URLs  
- Merge  
- Send Status Updates on Telegram  
- Rate Limit Wait Node  
- Rate Limit Wait Node1  

**Node Details:**

- **Google Sheets Trigger: Start Processing Tracks**  
  - Type: Google Sheets Trigger  
  - Role: Detects new or updated rows indicating pending song ideas to process.  
  - Inputs: Changes in Google Sheet rows.  
  - Outputs: Triggers downstream AI generation.  
  - Edge Cases: Trigger delays, missed updates.

- **Get one Pending Row**  
  - Type: Google Sheets  
  - Role: Fetches one row marked as pending for processing.  
  - Inputs: Trigger event.  
  - Outputs: Data for AI generation.  
  - Edge Cases: Empty or malformed rows.

- **Lyrics AI Agent**  
  - Type: Langchain Agent  
  - Role: Generates song lyrics based on input idea.  
  - Inputs: Song idea text.  
  - Outputs: Generated lyrics.  
  - Integrates with OpenAI Chat Model1 and Structured Output Parser.  
  - Edge Cases: API failures, unexpected output format.

- **OpenAI Chat Model1**  
  - Type: OpenAI Chat Model  
  - Role: Language model for lyrics generation.  
  - Inputs: Prompts from Lyrics AI Agent.  
  - Outputs: Raw lyrics text.  
  - Edge Cases: Rate limits, model errors.

- **Structured Output Parser**  
  - Type: Langchain Output Parser  
  - Role: Parses AI raw output into structured data for workflow consumption.  
  - Inputs: Raw lyrics text.  
  - Outputs: Structured lyrics data.  
  - Edge Cases: Parsing errors, unexpected formats.

- **Music Generation API Request**  
  - Type: HTTP Request  
  - Role: Calls Suno or equivalent music generation API to create audio tracks from lyrics.  
  - Inputs: Structured lyrics and parameters.  
  - Outputs: Generation job ID or status.  
  - Edge Cases: API authentication errors, request failures.

- **Wait**  
  - Type: Wait  
  - Role: Delays workflow to allow asynchronous music generation to complete.  
  - Inputs: After API request.  
  - Outputs: Triggers status check.  
  - Edge Cases: Timeout if generation takes too long.

- **Get Music Generation Status**  
  - Type: HTTP Request  
  - Role: Polls music generation API for job completion status.  
  - Inputs: Job ID.  
  - Outputs: Status (pending, complete, failed).  
  - Edge Cases: API errors, job failures.

- **Confirm Generation Status**  
  - Type: Switch  
  - Role: Routes workflow based on generation status: proceed if complete, wait or retry if pending, handle errors if failed.  
  - Inputs: Status response.  
  - Outputs: Download audio or wait nodes.  
  - Edge Cases: Incorrect status codes.

- **Download Audio Track**  
  - Type: HTTP Request  
  - Role: Downloads generated audio file.  
  - Inputs: Completed job output URL.  
  - Outputs: Audio binary data.  
  - Edge Cases: Download failures, corrupted files.

- **Upload Audio Track to Drive**  
  - Type: Google Drive  
  - Role: Uploads audio file to Google Drive for storage and sharing.  
  - Inputs: Audio binary data.  
  - Outputs: Drive file metadata and URL.  
  - Edge Cases: Permission errors, quota limits.

- **Update Audio URL in GSheet**  
  - Type: Google Sheets  
  - Role: Updates Google Sheet row with audio file URL for tracking.  
  - Inputs: Drive file URL.  
  - Outputs: Confirmation.  
  - Edge Cases: Sheet update conflicts.

- **Cover Image and Video Prompts AI Agent**  
  - Type: Langchain Agent  
  - Role: Generates prompts for cover images and video backgrounds based on lyrics and song metadata.  
  - Inputs: Song data and lyrics.  
  - Outputs: Image and video prompt texts.  
  - Integrates with Google Gemini Chat Model and Structured Output Parser1.  
  - Edge Cases: API errors, prompt generation failures.

- **Google Gemini Chat Model**  
  - Type: Google Gemini Chat Model  
  - Role: Language model for generating creative image/video prompts.  
  - Inputs: Prompts from AI Agent.  
  - Outputs: Raw prompt text.  
  - Edge Cases: API rate limits.

- **Structured Output Parser1**  
  - Type: Langchain Output Parser  
  - Role: Parses Gemini model output into structured prompts.  
  - Inputs: Raw prompt text.  
  - Outputs: Structured image/video prompts.  
  - Edge Cases: Parsing errors.

- **Generate Cover Image 1:1** & **Generate Cover Image 3:1**  
  - Type: HTTP Request  
  - Role: Calls Flux or equivalent image generation API to create cover images in two aspect ratios.  
  - Inputs: Structured prompts.  
  - Outputs: Image generation job IDs or URLs.  
  - Edge Cases: API errors, generation failures.

- **Wait1** & **Wait2**  
  - Type: Wait  
  - Role: Delay to allow image generation to complete before fetching results.  
  - Edge Cases: Timeout risks.

- **get_image_url1** & **get_image_url**  
  - Type: HTTP Request  
  - Role: Retrieves generated image URLs from API.  
  - Inputs: Job IDs.  
  - Outputs: Image URLs.  
  - Edge Cases: API failures.

- **Get Image** & **Get Image1**  
  - Type: HTTP Request  
  - Role: Downloads generated images.  
  - Inputs: Image URLs.  
  - Outputs: Image binary data.  
  - Edge Cases: Download failures.

- **Upload Cover Image 1:1 to Drive**  
  - Type: Google Drive  
  - Role: Uploads 1:1 cover image to Google Drive.  
  - Inputs: Image binary data.  
  - Outputs: Drive file metadata and URL.  
  - Edge Cases: Permission issues.

- **Upload to kraken to get url**  
  - Type: HTTP Request  
  - Role: Uploads 3:1 cover image to Kraken for optimization and retrieves optimized URL.  
  - Inputs: Image binary data.  
  - Outputs: Optimized image URL.  
  - Edge Cases: Kraken API errors.

- **Convert Cover Image 3:1 to Video**  
  - Type: HTTP Request  
  - Role: Calls Creatomate or Runway API to convert 3:1 image into a looping video background.  
  - Inputs: Optimized image URL.  
  - Outputs: Video generation job ID.  
  - Edge Cases: API failures.

- **Get Video Generation Status**  
  - Type: HTTP Request  
  - Role: Polls video generation API for completion status.  
  - Inputs: Job ID.  
  - Outputs: Status.  
  - Edge Cases: Timeout, failures.

- **Confirm Generation Status1**  
  - Type: Switch  
  - Role: Routes workflow based on video generation status.  
  - Edge Cases: Unexpected statuses.

- **Download Video**  
  - Type: HTTP Request  
  - Role: Downloads generated video background.  
  - Edge Cases: Download errors.

- **Upload Converted Video to Drive**  
  - Type: Google Drive  
  - Role: Uploads video background to Google Drive.  
  - Edge Cases: Quota limits.

- **Update Cover Photo & Video URLs**  
  - Type: Google Sheets  
  - Role: Updates Google Sheet with URLs of cover images and video backgrounds.  
  - Edge Cases: Update conflicts.

- **Merge**  
  - Type: Merge  
  - Role: Combines outputs from image and video uploads before sending status updates.  
  - Edge Cases: Data mismatches.

- **Send Status Updates on Telegram**  
  - Type: Telegram  
  - Role: Notifies user of progress and completion of generation steps.  
  - Edge Cases: Telegram API limits.

- **Rate Limit Wait Node** & **Rate Limit Wait Node1**  
  - Type: Wait  
  - Role: Enforces delays to respect API rate limits.  
  - Edge Cases: Excessive delays.

---

#### 2.3 Final Rendering & Delivery

**Overview:**  
This block merges the generated audio and video, uploads the final music video to Google Drive, updates the Google Sheet with the final URLs and render status, and sends the final video link back to the Telegram user.

**Nodes Involved:**  
- Render Video Trigger  
- Get Pending Render Row  
- Render Audio Track + Video  
- Wait3  
- Send URL to GDrive Script and Upload  
- Rename Uploaded Video  
- Update Music Video URL + Render Status  
- Send Video URL to User  

**Node Details:**

- **Render Video Trigger**  
  - Type: Google Sheets Trigger  
  - Role: Detects rows marked for final rendering.  
  - Edge Cases: Missed triggers.

- **Get Pending Render Row**  
  - Type: Google Sheets  
  - Role: Fetches row pending final render.  
  - Edge Cases: Empty rows.

- **Render Audio Track + Video**  
  - Type: HTTP Request  
  - Role: Calls API (Creatomate/Runway) to merge audio and video into final music video.  
  - Inputs: Audio and video URLs.  
  - Outputs: Render job ID.  
  - Edge Cases: API failures.

- **Wait3**  
  - Type: Wait  
  - Role: Delay to allow rendering to complete.  
  - Edge Cases: Timeout.

- **Send URL to GDrive Script and Upload**  
  - Type: HTTP Request  
  - Role: Sends final video URL to Google Apps Script for upload and processing.  
  - Edge Cases: Script errors.

- **Rename Uploaded Video**  
  - Type: Google Drive  
  - Role: Renames uploaded video file for clarity and organization.  
  - Edge Cases: Permission issues.

- **Update Music Video URL + Render Status**  
  - Type: Google Sheets  
  - Role: Updates Google Sheet with final video URL and marks render as complete.  
  - Edge Cases: Update conflicts.

- **Send Video URL to User**  
  - Type: Telegram  
  - Role: Sends final music video link back to the Telegram user.  
  - Edge Cases: Telegram API errors.

---

### 3. Summary Table

| Node Name                          | Node Type                         | Functional Role                           | Input Node(s)                             | Output Node(s)                          | Sticky Note                         |
|-----------------------------------|----------------------------------|------------------------------------------|------------------------------------------|---------------------------------------|-----------------------------------|
| Track Ideas Agent Telegram Trigger| Telegram Trigger                 | Entry point for user song ideas          | -                                        | Switch Text or Voice Input             |                                   |
| Switch Text or Voice Input         | Switch                          | Routes text or voice input                | Track Ideas Agent Telegram Trigger       | Set Field to "text", Download Voice Note |                                   |
| Download Voice Note                | Telegram                        | Downloads voice note audio                | Switch Text or Voice Input                | Transcribe Voice Note                  |                                   |
| Transcribe Voice Note             | OpenAI (Speech-to-Text)         | Transcribes voice to text                 | Download Voice Note                       | Set Field                             |                                   |
| Set Field                        | Set                             | Normalizes transcribed text field        | Transcribe Voice Note                     | AI Music Agent                        |                                   |
| Set Field to "text"               | Set                             | Prepares text input for AI processing    | Switch Text or Voice Input                | AI Music Agent                        |                                   |
| Append Music Tracks to GSheets    | Google Sheets Tool              | Logs song ideas to Google Sheets         | AI Music Agent                           | AI Music Agent (tool output)          |                                   |
| AI Music Agent                   | Langchain Agent                 | Generates music-related AI outputs       | Set Field, Set Field to "text", Append Music Tracks to GSheets | Send Message to User                 |                                   |
| Send Message to User              | Telegram                        | Sends status messages to Telegram user   | AI Music Agent                           | -                                     |                                   |
| Google Sheets Trigger: Start Processing Tracks | Google Sheets Trigger          | Detects new song ideas to process        | -                                        | Get one Pending Row                   |                                   |
| Get one Pending Row               | Google Sheets                   | Fetches pending song idea row             | Google Sheets Trigger                    | Lyrics AI Agent                      |                                   |
| Lyrics AI Agent                  | Langchain Agent                 | Generates song lyrics                     | Get one Pending Row                      | Music Generation API Request          |                                   |
| OpenAI Chat Model1               | OpenAI Chat Model               | Language model for lyrics generation      | Lyrics AI Agent                         | Structured Output Parser              |                                   |
| Structured Output Parser         | Langchain Output Parser         | Parses lyrics output                      | OpenAI Chat Model1                      | Lyrics AI Agent                      |                                   |
| Music Generation API Request     | HTTP Request                   | Calls music generation API                | Lyrics AI Agent                        | Wait                                |                                   |
| Wait                            | Wait                           | Waits for music generation                | Music Generation API Request             | Get Music Generation Status           |                                   |
| Get Music Generation Status      | HTTP Request                   | Polls music generation status             | Wait                                   | Confirm Generation Status             |                                   |
| Confirm Generation Status        | Switch                        | Routes based on music generation status   | Get Music Generation Status             | Download Audio Track, Rate Limit Wait Node, Wait |                                   |
| Download Audio Track             | HTTP Request                   | Downloads generated audio                  | Confirm Generation Status               | Upload Audio Track to Drive, Wait     |                                   |
| Upload Audio Track to Drive      | Google Drive                  | Uploads audio to Google Drive              | Download Audio Track                    | Update Audio URL in GSheet            |                                   |
| Update Audio URL in GSheet       | Google Sheets                 | Updates sheet with audio URL               | Upload Audio Track to Drive             | Cover Image and Video Prompts AI Agent |                                   |
| Cover Image and Video Prompts AI Agent | Langchain Agent             | Generates image and video prompts          | Update Audio URL in GSheet              | Generate Cover Image 1:1, Generate Cover Image 3:1 |                                   |
| Google Gemini Chat Model         | Google Gemini Chat Model       | Language model for image/video prompts    | Cover Image and Video Prompts AI Agent  | Structured Output Parser1             |                                   |
| Structured Output Parser1        | Langchain Output Parser        | Parses image/video prompts                 | Google Gemini Chat Model                | Cover Image and Video Prompts AI Agent |                                   |
| Generate Cover Image 1:1         | HTTP Request                   | Generates 1:1 cover image                  | Cover Image and Video Prompts AI Agent  | Wait1                               |                                   |
| Generate Cover Image 3:1         | HTTP Request                   | Generates 3:1 cover image                  | Cover Image and Video Prompts AI Agent  | Wait2                               |                                   |
| Wait1                           | Wait                           | Waits for 1:1 image generation             | Generate Cover Image 1:1                | get_image_url1                      |                                   |
| get_image_url1                  | HTTP Request                   | Retrieves 1:1 image URL                     | Wait1                                 | Get Image                          |                                   |
| Get Image                      | HTTP Request                   | Downloads 1:1 image                         | get_image_url1                        | Upload Cover Image 1:1 to Drive      |                                   |
| Upload Cover Image 1:1 to Drive | Google Drive                  | Uploads 1:1 cover image                     | Get Image                            | Merge                              |                                   |
| Wait2                           | Wait                           | Waits for 3:1 image generation             | Generate Cover Image 3:1                | get_image_url                      |                                   |
| get_image_url                  | HTTP Request                   | Retrieves 3:1 image URL                     | Wait2                                 | Get Image1                         |                                   |
| Get Image1                     | HTTP Request                   | Downloads 3:1 image                         | get_image_url                        | Upload to kraken to get url          |                                   |
| Upload to kraken to get url    | HTTP Request                   | Optimizes 3:1 image via Kraken              | Get Image1                           | Convert Cover Image 3:1 to Video     |                                   |
| Convert Cover Image 3:1 to Video | HTTP Request                 | Converts 3:1 image to looping video         | Upload to kraken to get url           | 1 minute                          |                                   |
| 1 minute                       | Wait                           | Waits for video generation                   | Convert Cover Image 3:1 to Video      | Get Video Generation Status          |                                   |
| Get Video Generation Status    | HTTP Request                   | Polls video generation status                | 1 minute                             | Confirm Generation Status1           |                                   |
| Confirm Generation Status1     | Switch                        | Routes based on video generation status      | Get Video Generation Status           | Download Video, 1 minute, Rate Limit Wait Node1 |                                   |
| Download Video                | HTTP Request                   | Downloads generated video                      | Confirm Generation Status1            | Upload Converted Video to Drive      |                                   |
| Upload Converted Video to Drive | Google Drive                  | Uploads video background to Drive             | Download Video                      | Merge                              |                                   |
| Merge                         | Merge                         | Combines image and video upload outputs        | Upload Cover Image 1:1 to Drive, Upload Converted Video to Drive | Update Cover Photo & Video URLs    |                                   |
| Update Cover Photo & Video URLs | Google Sheets                 | Updates sheet with cover and video URLs         | Merge                              | Send Status Updates on Telegram      |                                   |
| Send Status Updates on Telegram | Telegram                      | Sends progress updates to user                  | Update Cover Photo & Video URLs      | -                                 |                                   |
| Rate Limit Wait Node           | Wait                           | Enforces API rate limits                        | Confirm Generation Status            | -                                 |                                   |
| Rate Limit Wait Node1          | Wait                           | Enforces API rate limits                        | Confirm Generation Status1           | -                                 |                                   |
| Render Video Trigger           | Google Sheets Trigger          | Detects rows ready for final video rendering     | -                                | Get Pending Render Row              |                                   |
| Get Pending Render Row         | Google Sheets                 | Fetches row pending final render                  | Render Video Trigger               | Render Audio Track + Video          |                                   |
| Render Audio Track + Video     | HTTP Request                 | Calls API to merge audio and video into final clip | Get Pending Render Row           | Wait3                             |                                   |
| Wait3                         | Wait                           | Waits for final rendering completion             | Render Audio Track + Video         | Send URL to GDrive Script and Upload |                                   |
| Send URL to GDrive Script and Upload | HTTP Request             | Sends final video URL to Google Apps Script       | Wait3                             | Rename Uploaded Video              |                                   |
| Rename Uploaded Video         | Google Drive                  | Renames final video file                           | Send URL to GDrive Script and Upload | Update Music Video URL + Render Status |                                   |
| Update Music Video URL + Render Status | Google Sheets           | Updates sheet with final video URL and status      | Rename Uploaded Video             | Send Video URL to User             |                                   |
| Send Video URL to User        | Telegram                      | Sends final video link back to Telegram user       | Update Music Video URL + Render Status | -                                 |                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Type: Telegram Trigger  
   - Configure with your Telegram Bot API credentials.  
   - Set to trigger on new messages.

2. **Add Switch Node to Detect Input Type**  
   - Type: Switch  
   - Condition: Check if incoming message contains voice note or text.  
   - Connect Telegram Trigger output to this node.

3. **Add Download Voice Note Node**  
   - Type: Telegram (Download)  
   - Configure to download voice note file from Telegram.  
   - Connect voice note output from Switch node here.

4. **Add Transcribe Voice Note Node**  
   - Type: OpenAI (Speech-to-Text)  
   - Configure with OpenAI API credentials.  
   - Connect Download Voice Note output here.

5. **Add Set Node to Normalize Transcription**  
   - Type: Set  
   - Configure to set transcription text into a consistent field (e.g., `text`).  
   - Connect Transcribe Voice Note output here.

6. **Add Set Node for Text Input**  
   - Type: Set  
   - Configure to set incoming text messages into the same consistent field (`text`).  
   - Connect text output from Switch node here.

7. **Add Append to Google Sheets Node**  
   - Type: Google Sheets Tool  
   - Configure with Google Sheets credentials and target sheet.  
   - Append new rows with user chat ID and song idea text.  
   - Connect both Set nodes (transcription and text) outputs here.

8. **Create Google Sheets Trigger Node**  
   - Type: Google Sheets Trigger  
   - Configure to watch for new rows indicating pending song ideas.

9. **Add Google Sheets Node to Get One Pending Row**  
   - Type: Google Sheets  
   - Configure to read one row marked as pending.

10. **Add Langchain Agent Node for Lyrics Generation**  
    - Type: Langchain Agent  
    - Configure with OpenAI Chat Model and Structured Output Parser.  
    - Connect Google Sheets row data here.

11. **Add OpenAI Chat Model Node**  
    - Configure with OpenAI API credentials.  
    - Connect to Langchain Agent for lyrics generation.

12. **Add Structured Output Parser Node**  
    - Configure to parse lyrics output into structured format.

13. **Add HTTP Request Node for Music Generation API**  
    - Configure to call Suno or equivalent music generation API.  
    - Use lyrics and parameters as payload.

14. **Add Wait Node**  
    - Configure a delay (e.g., 30-60 seconds) to allow music generation.

15. **Add HTTP Request Node to Poll Music Generation Status**  
    - Configure to check job status using job ID.

16. **Add Switch Node to Confirm Music Generation Status**  
    - Route based on status: proceed if complete, wait or retry if pending, handle errors.

17. **Add HTTP Request Node to Download Audio Track**  
    - Configure to download audio file from API.

18. **Add Google Drive Node to Upload Audio Track**  
    - Configure with Google Drive credentials and target folder.

19. **Add Google Sheets Node to Update Audio URL**  
    - Update the corresponding row with audio file URL.

20. **Add Langchain Agent Node for Cover Image and Video Prompts**  
    - Configure with Google Gemini Chat Model and Structured Output Parser1.

21. **Add Google Gemini Chat Model Node**  
    - Configure with Google API credentials.

22. **Add Structured Output Parser1 Node**  
    - Configure to parse image/video prompt outputs.

23. **Add HTTP Request Nodes to Generate Cover Images (1:1 and 3:1)**  
    - Configure calls to Flux or equivalent image generation API.

24. **Add Wait Nodes (Wait1 and Wait2)**  
    - Configure delays for image generation completion.

25. **Add HTTP Request Nodes to Retrieve Image URLs (get_image_url1 and get_image_url)**  
    - Configure to fetch generated image URLs.

26. **Add HTTP Request Nodes to Download Images (Get Image and Get Image1)**  
    - Configure to download image binaries.

27. **Add Google Drive Node to Upload 1:1 Cover Image**  
    - Configure upload to Drive folder.

28. **Add HTTP Request Node to Upload 3:1 Image to Kraken**  
    - Configure Kraken API credentials and upload.

29. **Add HTTP Request Node to Convert 3:1 Image to Video**  
    - Configure Creatomate or Runway API call.

30. **Add Wait Node (1 minute)**  
    - Delay to allow video generation.

31. **Add HTTP Request Node to Get Video Generation Status**  
    - Poll video generation job status.

32. **Add Switch Node to Confirm Video Generation Status**  
    - Route based on status.

33. **Add HTTP Request Node to Download Video**  
    - Download generated video background.

34. **Add Google Drive Node to Upload Converted Video**  
    - Upload video to Drive.

35. **Add Merge Node**  
    - Merge outputs from image and video uploads.

36. **Add Google Sheets Node to Update Cover Photo & Video URLs**  
    - Update sheet with URLs.

37. **Add Telegram Node to Send Status Updates**  
    - Notify user of progress.

38. **Add Google Sheets Trigger Node for Final Rendering**  
    - Detect rows marked for final video rendering.

39. **Add Google Sheets Node to Get Pending Render Row**  
    - Fetch row for rendering.

40. **Add HTTP Request Node to Render Audio + Video**  
    - Call Creatomate or Runway API to merge audio and video.

41. **Add Wait Node (Wait3)**  
    - Wait for rendering completion.

42. **Add HTTP Request Node to Send URL to Google Apps Script**  
    - Upload final video via Apps Script.

43. **Add Google Drive Node to Rename Uploaded Video**  
    - Rename file for clarity.

44. **Add Google Sheets Node to Update Music Video URL and Render Status**  
    - Mark render complete and update URL.

45. **Add Telegram Node to Send Final Video URL to User**  
    - Deliver final video link.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                              | Context or Link                                                                                      |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Sample audio generated by this workflow is available on YouTube: https://youtu.be/61MEf6MJEAM                                                              | Demonstrates output quality and style.                                                             |
| Full setup guide with detailed steps, API setup, code snippets, and integrations is provided as a PDF on the Gumroad download page.                        | Essential for configuring APIs, credentials, and Google Apps Script.                               |
| Song generation costs approx. $5 for 1000 credits; each song costs 12 credits; Flux image generation costs $0.04 per image; Runway video costs $0.16/10s.    | Important for budgeting and API usage planning.                                                    |
| Uses Google Apps Script for Google Drive uploads and file management.                                                                                      | Requires script deployment and permissions setup.                                                  |
| Integrates multiple AI APIs: OpenAI (Chat and Whisper), Google Gemini, Suno (music), Flux (images), Runway and Creatomate (video rendering).               | Ensure all API keys and credentials are correctly configured and tested.                            |
| Telegram bot requires webhook setup to receive messages.                                                                                                  | Telegram bot configuration must include webhook URL pointing to n8n instance.                      |

---

This structured documentation enables advanced users and AI agents to fully understand, reproduce, and modify the workflow, anticipate integration issues, and manage error handling effectively.