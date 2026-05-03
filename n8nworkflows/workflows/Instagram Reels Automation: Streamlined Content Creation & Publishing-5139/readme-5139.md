Instagram Reels Automation: Streamlined Content Creation & Publishing

https://n8nworkflows.xyz/workflows/instagram-reels-automation--streamlined-content-creation---publishing-5139


# Instagram Reels Automation: Streamlined Content Creation & Publishing

### 1. Workflow Overview

This workflow automates the creation and publishing of Instagram Reels content by integrating Google Drive, AI-powered caption generation, Airtable for content management, and Facebook Graph API for Instagram publishing. It targets social media managers or content creators who want a streamlined process to upload videos, generate engaging captions automatically, schedule posts, and track content efficiently. The workflow consists of two main logical blocks:

- **1.1 File Acquisition and Preparation:** Detects new video files in a designated Google Drive folder or periodically selects random videos from a source folder, downloads them, processes metadata, and organizes files between folders.
- **1.2 Instagram Content Creation and Publishing:** Uses AI (Google Gemini and LangChain) to create captions, stores post metadata in Airtable, publishes the video as an Instagram Reel via Facebook Graph API, and cleans up files after posting.

Supporting nodes handle timing (waits), error prevention, and API credential management.

---

### 2. Block-by-Block Analysis

#### 1.1 File Acquisition and Preparation

**Overview:**  
This block detects new video files either by manual upload into a Google Drive folder or by scheduled random selection from a source folder. It downloads the selected video, extracts and processes metadata, and moves files between Google Drive folders to organize workflow states.

**Nodes Involved:**  
- Schedule Trigger1  
- HTTP Request  
- Code1  
- HTTP Request1  
- Post File Upload in Google Drive Folder Trigger  
- Post File Download in N8N (Google Drive Node)  
- Code  
- Post File Upload in Google Drive Folder Trigger  
- Google Drive1  
- Wait1  
- Sticky Notes (contextual)

**Node Details:**

- **Schedule Trigger1**  
  - Type: Schedule Trigger  
  - Role: Periodically triggers every 6 hours to start the random video selection path.  
  - Config: 6-hour interval.  
  - Inputs: None  
  - Outputs: HTTP Request  
  - Edge Cases: Misconfigured intervals or timezone issues may delay triggers.

- **HTTP Request**  
  - Type: HTTP Request (Google Drive API v3)  
  - Role: Queries Google Drive folder ('Random video source folder') for mp4 video files.  
  - Config: GET request with query parameters filtering mp4 files, including support for shared drives.  
  - Inputs: Schedule Trigger1  
  - Outputs: Code1  
  - Edge Cases: API rate limits, invalid folder ID, network failures.

- **Code1**  
  - Type: Code (JavaScript)  
  - Role: Selects one random video file from the API response array.  
  - Inputs: HTTP Request  
  - Outputs: HTTP Request1  
  - Edge Cases: Empty folder (returns no files), malformed API response.

- **HTTP Request1**  
  - Type: HTTP Request (Google Drive API PATCH)  
  - Role: Moves the randomly selected video file from the source folder to the target folder watched by the manual upload trigger.  
  - Config: PATCH request adding and removing parents (folders).  
  - Inputs: Code1  
  - Outputs: Wait2  
  - Edge Cases: Permission issues, invalid file or folder IDs.

- **Post File Upload in Google Drive Folder Trigger**  
  - Type: Google Drive Trigger  
  - Role: Watches the target folder for new video file uploads (manual or via the random mover).  
  - Config: Polls every 1 minute, triggers on fileCreated event in a specific folder.  
  - Inputs: None (trigger node)  
  - Outputs: Post File Download in N8N (Google Drive Node)  
  - Edge Cases: Folder ID misconfiguration, API auth failure.

- **Post File Download in N8N (Google Drive Node)**  
  - Type: Google Drive Node  
  - Role: Downloads the triggered video file for processing.  
  - Config: Downloads based on the file ID from trigger output.  
  - Inputs: Post File Upload in Google Drive Folder Trigger, Wait2  
  - Outputs: Code  
  - Edge Cases: Download failures, file not found, API errors.

- **Code**  
  - Type: Code (JavaScript)  
  - Role: Extracts and sanitizes key metadata from the downloaded file (file ID, name, content link). Handles missing or invalid data gracefully.  
  - Inputs: Post File Download in N8N (Google Drive Node)  
  - Outputs: AI Agent  
  - Edge Cases: Unexpected input structure, missing file metadata, null or empty file names.

- **Google Drive1**  
  - Type: Google Drive Node  
  - Role: Deletes the video file after successful publishing to keep the folder clean.  
  - Config: Deletes file by URL from earlier download metadata.  
  - Inputs: Wait1 (post publishing wait)  
  - Outputs: None  
  - Edge Cases: Deletion failures, file already deleted or missing.

- **Wait1**  
  - Type: Wait  
  - Role: Waits 20 seconds after posting to allow for processing before deleting the file.  
  - Inputs: Post to IG  
  - Outputs: Google Drive1  
  - Edge Cases: Timing mismatches, premature deletion.

- **Wait2**  
  - Type: Wait  
  - Role: Waits 20 seconds after moving the file before starting the download and caption generation process to ensure file availability.  
  - Inputs: HTTP Request1  
  - Outputs: Post File Download in N8N (Google Drive Node)  
  - Edge Cases: Insufficient wait time causing file access errors.

- **Sticky Notes**  
  - Provide contextual comments explaining the logic and timing, e.g., "wait 20 seconds then trigger the insta posting workflow," "manual upload trigger," and "random file mover."

---

#### 1.2 Instagram Content Creation and Publishing

**Overview:**  
This block generates AI-powered captions for the video, logs metadata in Airtable, uploads the video to Instagram Reels via the Facebook Graph API, publishes the post, and manages timing between steps.

**Nodes Involved:**  
- Code  
- AI Agent  
- Google Gemini Chat Model (used internally by AI Agent)  
- Airtable  
- container (Facebook Graph API media upload)  
- Wait  
- Post to IG  
- Wait1 (shared with Block 1.1)  
- Sticky Notes (contextual)

**Node Details:**

- **Code**  
  - (As described above) Prepares metadata for AI captioning.  
  - Inputs: Post File Download in N8N (Google Drive Node)  
  - Outputs: AI Agent

- **Google Gemini Chat Model**  
  - Type: Google Gemini LLM Chat Model (via LangChain)  
  - Role: Provides the AI language model backend for the AI Agent node.  
  - Config: Using model "models/gemini-2.0-flash-001".  
  - Inputs: AI Agent (ai_languageModel input)  
  - Outputs: AI Agent (ai_languageModel output)  
  - Edge Cases: API limits, authentication errors, model unavailability.

- **AI Agent**  
  - Type: LangChain Agent  
  - Role: Generates an Instagram caption with specific instructions: 2-3 sentences, emojis, hashtags, call-to-action, under 150 characters, tone tailored for Instagram.  
  - Key Prompt Template: Uses the processed file name from Code node as input.  
  - Inputs: Code  
  - Outputs: Airtable  
  - Edge Cases: Prompt interpretation failures, empty or malformed inputs.

- **Airtable**  
  - Type: Airtable Node  
  - Role: Creates a record in Airtable with the video URL, file name, and generated caption for content tracking.  
  - Config: Uses a specified Airtable base and table with defined columns: Name, Caption, URL.  
  - Inputs: AI Agent  
  - Outputs: container  
  - Edge Cases: API rate limits, invalid base/table IDs, authentication errors.

- **container (Facebook Graph API Media Upload)**  
  - Type: Facebook Graph API Node  
  - Role: Uploads the video to Instagram Reels (media_type: REELS) with the AI-generated caption.  
  - Config: Uses Facebook Page ID and passes video URL and caption as query parameters.  
  - Inputs: Airtable  
  - Outputs: Wait  
  - Edge Cases: Upload failures, invalid Facebook credentials, API limits, incorrect Page ID.

- **Wait**  
  - Type: Wait  
  - Role: Waits 90 seconds after uploading before publishing the post to ensure media processing.  
  - Inputs: container  
  - Outputs: Post to IG  
  - Edge Cases: Insufficient wait time causing publish failures.

- **Post to IG**  
  - Type: Facebook Graph API Node  
  - Role: Publishes the uploaded media to Instagram by referencing the creation ID received from upload.  
  - Config: Uses Facebook Page ID and creation ID from previous node.  
  - Inputs: Wait  
  - Outputs: Wait1  
  - Edge Cases: Publishing errors, invalid creation ID.

- **Wait1**  
  - (Described above) Waits 20 seconds before deleting the file.

- **Sticky Notes**  
  - Comments indicate this block as "insta uploader" and label the entire workflow as "Instagram automation workflow....."

---

### 3. Summary Table

| Node Name                             | Node Type                          | Functional Role                                  | Input Node(s)                                | Output Node(s)                               | Sticky Note                                                                                      |
|-------------------------------------|----------------------------------|-------------------------------------------------|----------------------------------------------|----------------------------------------------|-------------------------------------------------------------------------------------------------|
| Schedule Trigger1                   | Schedule Trigger                 | Periodically triggers random video selection     | None                                         | HTTP Request                                 | ## Random file mover **For every 100 minutes this path of workflow will select and move a single video file from the G-drive fil named: Random video mover** |
| HTTP Request                       | HTTP Request                    | Queries Google Drive for mp4 videos               | Schedule Trigger1                            | Code1                                        |                                                                                                 |
| Code1                             | Code                            | Picks one random video from drive files           | HTTP Request                                 | HTTP Request1                                |                                                                                                 |
| HTTP Request1                     | HTTP Request                    | Moves selected file to target folder               | Code1                                        | Wait2                                        |                                                                                                 |
| Wait2                             | Wait                            | Waits 20 seconds before download                   | HTTP Request1                                | Post File Download in N8N (Google Drive Node) |                                                                                                 |
| Post File Upload in Google Drive Folder Trigger | Google Drive Trigger            | Triggers on manual or moved file upload            | None                                         | Post File Download in N8N (Google Drive Node) | ## manual upload trigger **it will trigger the workflow whenever the video file as upload manually on the folder** |
| Post File Download in N8N (Google Drive Node) | Google Drive                    | Downloads the video file                            | Post File Upload in Google Drive Folder Trigger, Wait2 | Code                                         |                                                                                                 |
| Code                             | Code                            | Extracts metadata (file ID, name, content link)    | Post File Download in N8N (Google Drive Node) | AI Agent                                     |                                                                                                 |
| AI Agent                         | LangChain Agent                 | Generates Instagram caption using AI               | Code                                         | Airtable                                     |                                                                                                 |
| Google Gemini Chat Model          | Google Gemini LLM Chat Model    | Provides AI model backend for caption generation   | AI Agent (ai_languageModel input)            | AI Agent (ai_languageModel output)            |                                                                                                 |
| Airtable                        | Airtable                       | Creates record with video metadata and caption     | AI Agent                                     | container                                    |                                                                                                 |
| container                       | Facebook Graph API              | Uploads video to Instagram Reels                    | Airtable                                     | Wait                                         | ## insta uploader                                                                               |
| Wait                           | Wait                          | Waits 90 seconds for media processing               | container                                    | Post to IG                                   |                                                                                                 |
| Post to IG                     | Facebook Graph API              | Publishes the uploaded Instagram Reel               | Wait                                         | Wait1                                        |                                                                                                 |
| Wait1                          | Wait                          | Waits 20 seconds before deleting the video          | Post to IG                                   | Google Drive1                                | ## wait 20 seconds then trigger the insta posting workflow                                      |
| Google Drive1                  | Google Drive                    | Deletes video file after posting                     | Wait1                                        | None                                         |                                                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node:**  
   - Name: Schedule Trigger1  
   - Interval: Every 6 hours  
   - Purpose: Periodically initiate random video selection.

2. **Add an HTTP Request node (Google Drive API v3) to list videos:**  
   - Name: HTTP Request  
   - Method: GET  
   - URL: `https://www.googleapis.com/drive/v3/files`  
   - Query Parameters:  
     - `q`: `'YOUR_RANDOM_VIDEO_SOURCE_FOLDER_ID' in parents and mimeType='video/mp4' and trashed=false`  
     - `fields`: `files(id, name, mimeType, parents)`  
     - `supportsAllDrives`: `true`  
     - `includeItemsFromAllDrives`: `true`  
   - Credentials: Google OAuth2 with appropriate Drive access.

3. **Add a Code node (Code1) to select a random video:**  
   - Name: Code1  
   - JavaScript: Select a random file from `items[0].json.files`. Return a single item with the random file.

4. **Add an HTTP Request node (HTTP Request1) to move the selected file:**  
   - Method: PATCH  
   - URL: `https://www.googleapis.com/drive/v3/files/{{ $json.id }}`  
   - Query Parameters:  
     - `addParents`: `YOUR_FOLDER_ID` (target folder for processing)  
     - `removeParents`: `YOUR_RANDOM_VIDEO_SOURCE_FOLDER_ID` (source folder)  
     - `fields`: `id,name,parents,webContentLink,webViewLink,mimeType`  
   - Credentials: Google OAuth2.

5. **Add a Wait node (Wait2):**  
   - Duration: 20 seconds  
   - Purpose: Allow file move operation to complete before download.

6. **Add a Google Drive Trigger node:**  
   - Name: Post File Upload in Google Drive Folder Trigger  
   - Event: fileCreated  
   - Folder to watch: `YOUR_FOLDER_ID` (same as above)  
   - Polling interval: 1 minute  
   - Credential: Google Drive OAuth2.

7. **Add a Google Drive node (Post File Download in N8N):**  
   - Operation: Download  
   - File ID: `={{ $json.id }}` from the trigger or Wait2 output  
   - Credential: Google Drive OAuth2.

8. **Add a Code node to extract metadata (Code):**  
   - Extract file ID, file name, and web content link or web view link.  
   - Handle missing or null values gracefully.

9. **Add a LangChain Google Gemini Chat Model node:**  
   - Model: `models/gemini-2.0-flash-001`  
   - Credential: Google Gemini API.

10. **Add a LangChain Agent node (AI Agent):**  
    - Prompt: Generate an Instagram caption with:  
      - 2-3 engaging sentences + emojis  
      - 3-5 hashtags based on file name  
      - Call-to-action for comments  
      - Max 150 characters  
      - Tone: Relatable, Instagram audience  
    - Use output from Code node as input variable.  
    - Connect AI Agent's language model input to Google Gemini Chat Model.

11. **Add an Airtable node:**  
    - Operation: Create record  
    - Base ID and Table ID: Your Airtable  
    - Map columns:  
      - `Name`: processedFileName from Code node  
      - `URL`: processedWebContentLink from Code node  
      - `Caption`: AI Agent output  
    - Credential: Airtable Personal Access Token.

12. **Add a Facebook Graph API node (container):**  
    - Operation: POST to `/YOUR_FACEBOOK_PAGE_ID/media`  
    - Query Parameters:  
      - `video_url`: URL from Airtable  
      - `media_type`: `REELS`  
      - `caption`: AI Agent output caption  
    - Credential: Facebook Graph API OAuth2.

13. **Add a Wait node:**  
    - Duration: 90 seconds  
    - Purpose: Wait for media processing before publishing.

14. **Add a Facebook Graph API node (Post to IG):**  
    - Operation: POST to `/YOUR_FACEBOOK_PAGE_ID/media_publish`  
    - Query Parameter:  
      - `creation_id`: ID from previous upload response  
    - Credential: Facebook Graph API OAuth2.

15. **Add a Wait node (Wait1):**  
    - Duration: 20 seconds  
    - Purpose: Wait before deleting the video file.

16. **Add a Google Drive node (Google Drive1):**  
    - Operation: Delete file  
    - File ID or URL from Code node metadata.  
    - Credential: Google Drive OAuth2.

17. **Connect nodes as per the logical flow:**  
    - Schedule Trigger1 → HTTP Request → Code1 → HTTP Request1 → Wait2 → Post File Download in N8N → Code → AI Agent → Airtable → container → Wait → Post to IG → Wait1 → Google Drive1  
    - Post File Upload Trigger → Post File Download → Code (same path for manual uploads)

18. **Add Sticky Notes for documentation and clarity** (optional but recommended).

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                 |
|---------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Workflow enables automated Instagram Reels posting with AI-generated captions and content tracking in Airtable | Instagram Reels Automation Project                                                             |
| Uses Google Gemini (PaLM) via LangChain for caption generation                                                 | Requires Google Gemini API credentials                                                          |
| Facebook Graph API v22.0 used for media upload and publishing                                                  | Instagram Business account and Facebook Page with correct permissions required                   |
| Polling intervals and wait times tuned to avoid file access conflicts and API processing delays               | Adjust wait times based on API responsiveness and file size                                     |
| Airtable base and table IDs must be configured with appropriate schema                                        | Columns: Name (string), Caption (string), URL (string)                                          |
| Google Drive folders IDs must be set correctly for source, processing, and trigger folders                     | Ensure OAuth2 credentials have access to all relevant folders                                   |
| Sticky Notes in workflow provide key operational comments                                                     | Add or update for clarity during maintenance                                                   |

---

**Disclaimer:**  
The text provided is derived exclusively from an automated workflow created using n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.