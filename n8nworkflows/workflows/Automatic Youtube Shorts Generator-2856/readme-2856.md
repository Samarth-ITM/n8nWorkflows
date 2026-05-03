Automatic Youtube Shorts Generator

https://n8nworkflows.xyz/workflows/automatic-youtube-shorts-generator-2856


# Automatic Youtube Shorts Generator

### 1. Workflow Overview

This workflow automates the creation of YouTube Shorts videos from trending Google News articles, specifically targeting the "Animal" category in this instance. It is designed for content creators, journalists, and digital marketers who want to quickly generate engaging short videos based on current news trends without manual editing.

The workflow is logically divided into the following blocks:

- **1.1 Trigger and Initial Data Fetch:** Starts the workflow manually or on schedule, fetches trending article transcripts.
- **1.2 Audio Processing:** Generates voiceover audio from transcripts using ElevenLabs TTS and processes it.
- **1.3 Image Generation and Processing:** Creates image prompts from transcripts, generates images via AI, converts and stores them.
- **1.4 Video Assembly and Monitoring:** Combines audio and images into a video, monitors video creation progress.
- **1.5 Metadata Generation and Upload:** Generates video titles and captions, uploads the video to YouTube, and logs metadata in Google Sheets.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger and Initial Data Fetch

**Overview:**  
This block initiates the workflow either manually or on a schedule and fetches the transcript of trending Google News articles in the specified video category.

**Nodes Involved:**  
- Schedule Trigger  
- When clicking ‘Test workflow’ (Manual Trigger)  
- Video Category (Set node)  
- Get Transcript By Deepseek  
- Fetch Elevenlabs  

**Node Details:**  

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Automatically triggers the workflow periodically (e.g., every 24 hours) to fetch new trending topics.  
  - Configuration: Default scheduling parameters (not detailed here).  
  - Inputs: None  
  - Outputs: Connects to "Grab Video" node.  
  - Edge Cases: Scheduling misconfiguration or downtime could delay updates.

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Allows manual initiation for testing or immediate runs.  
  - Inputs: None  
  - Outputs: Connects to "Video Category".  
  - Edge Cases: None significant.

- **Video Category**  
  - Type: Set  
  - Role: Defines the category filter for fetching transcripts (e.g., "Animal").  
  - Configuration: Sets static or dynamic category value.  
  - Inputs: From manual trigger.  
  - Outputs: Connects to "Get Transcript By Deepseek".  
  - Edge Cases: Incorrect category values may yield no results.

- **Get Transcript By Deepseek**  
  - Type: HTTP Request  
  - Role: Calls Deepseek API to retrieve article transcripts based on the category.  
  - Configuration: Uses API credentials, category parameter from previous node.  
  - Inputs: From "Video Category".  
  - Outputs: Connects to "Fetch Elevenlabs".  
  - Edge Cases: API rate limits, network errors, invalid responses.

- **Fetch Elevenlabs**  
  - Type: HTTP Request  
  - Role: Initiates voiceover generation request to ElevenLabs TTS service.  
  - Configuration: Uses ElevenLabs API key, sends transcript text.  
  - Inputs: From "Get Transcript By Deepseek".  
  - Outputs: Connects to "Wait".  
  - Edge Cases: Authentication errors, service downtime.

---

#### 2.2 Audio Processing

**Overview:**  
This block handles the waiting for audio generation, downloads the audio file, transcribes it with OpenAI Whisper for verification, and saves the audio to Google Cloud Storage.

**Nodes Involved:**  
- Wait  
- Save Audio (Google Cloud Storage)  
- Map Public Link (Set)  
- Download Audio (HTTP Request)  
- Open AI Whisper (HTTP Request)  

**Node Details:**  

- **Wait**  
  - Type: Wait  
  - Role: Pauses workflow to allow ElevenLabs to process audio generation.  
  - Configuration: Default wait time or webhook-based wait.  
  - Inputs: From "Fetch Elevenlabs".  
  - Outputs: Connects to "Save Audio".  
  - Edge Cases: Insufficient wait time may cause premature requests.

- **Save Audio**  
  - Type: Google Cloud Storage  
  - Role: Uploads generated audio file to Google Cloud Storage for persistent storage.  
  - Configuration: Uses GCS credentials, bucket name, and file path.  
  - Inputs: From "Wait".  
  - Outputs: Connects to "Map Public Link".  
  - Edge Cases: Storage permission errors, quota limits.

- **Map Public Link**  
  - Type: Set  
  - Role: Sets the public URL of the stored audio for downstream use.  
  - Configuration: Constructs URL from storage bucket and file path.  
  - Inputs: From "Save Audio".  
  - Outputs: Connects to "Download Audio".  
  - Edge Cases: Incorrect URL formatting.

- **Download Audio**  
  - Type: HTTP Request  
  - Role: Downloads the stored audio file for transcription.  
  - Configuration: Uses public URL from previous node.  
  - Inputs: From "Map Public Link".  
  - Outputs: Connects to "Open AI Whisper".  
  - Edge Cases: Network errors, file not found.

- **Open AI Whisper**  
  - Type: HTTP Request  
  - Role: Sends audio to OpenAI Whisper API for speech-to-text transcription.  
  - Configuration: Uses OpenAI API key, audio file input.  
  - Inputs: From "Download Audio".  
  - Outputs: Connects to "Create a list of Image Text".  
  - Edge Cases: API errors, transcription inaccuracies.

---

#### 2.3 Image Generation and Processing

**Overview:**  
This block extracts image prompt texts from the transcript, converts them into AI image generation prompts, generates images, converts images to files, and stores them in Google Cloud Storage.

**Nodes Involved:**  
- Create a list of Image Text (Code)  
- Split Out  
- Convert to Flux Prompt (HTTP Request)  
- Split Out - Image prompts  
- Get image Base 64 (HTTP Request)  
- Convert to File  
- Wait1  
- Save images to GC (Google Cloud Storage)  
- Limit  
- Image Link (Set)  

**Node Details:**  

- **Create a list of Image Text**  
  - Type: Code  
  - Role: Parses transcript text to extract or generate a list of descriptive texts for image prompts.  
  - Configuration: Custom JavaScript code processing transcript data.  
  - Inputs: From "Open AI Whisper".  
  - Outputs: Connects to "Split Out".  
  - Edge Cases: Parsing errors, empty or malformed transcript.

- **Split Out**  
  - Type: Split Out  
  - Role: Splits the list of image texts into individual items for parallel processing.  
  - Inputs: From "Create a list of Image Text".  
  - Outputs: Connects to "Convert to Flux Prompt".  
  - Edge Cases: Empty input arrays.

- **Convert to Flux Prompt**  
  - Type: HTTP Request  
  - Role: Converts each image text into a prompt format suitable for the AI image generation service.  
  - Inputs: From "Split Out".  
  - Outputs: Connects to "Split Out - Image prompts".  
  - Edge Cases: API errors, invalid prompt formats.

- **Split Out - Image prompts**  
  - Type: Split Out  
  - Role: Splits the AI-generated prompts for individual image generation requests.  
  - Inputs: From "Convert to Flux Prompt".  
  - Outputs: Connects to "Get image Base 64".  
  - Edge Cases: Empty or invalid prompts.

- **Get image Base 64**  
  - Type: HTTP Request  
  - Role: Requests AI image generation service to create images and returns them as Base64 strings.  
  - Inputs: From "Split Out - Image prompts".  
  - Outputs: Connects to "Convert to File".  
  - Edge Cases: API limits, generation failures.

- **Convert to File**  
  - Type: Convert To File  
  - Role: Converts Base64 image strings into file objects for storage.  
  - Inputs: From "Get image Base 64".  
  - Outputs: Connects to "Wait1".  
  - Edge Cases: Conversion errors.

- **Wait1**  
  - Type: Wait  
  - Role: Waits to ensure files are ready before uploading.  
  - Inputs: From "Convert to File".  
  - Outputs: Connects to "Save images to GC".  
  - Edge Cases: Timing issues.

- **Save images to GC**  
  - Type: Google Cloud Storage  
  - Role: Uploads image files to Google Cloud Storage.  
  - Inputs: From "Wait1".  
  - Outputs: Connects to "Limit".  
  - Edge Cases: Storage errors.

- **Limit**  
  - Type: Limit  
  - Role: Controls concurrency or limits the number of images processed simultaneously.  
  - Inputs: From "Save images to GC".  
  - Outputs: Connects to "Image Link".  
  - Edge Cases: Bottlenecks if limit too low.

- **Image Link**  
  - Type: Set  
  - Role: Sets the public URLs of stored images for video creation.  
  - Inputs: From "Limit".  
  - Outputs: Connects to "Combine Transcript".  
  - Edge Cases: URL formatting errors.

---

#### 2.4 Video Assembly and Monitoring

**Overview:**  
This block combines the transcript and images, maps background music, creates the video via an external service, waits for processing, and monitors video creation progress.

**Nodes Involved:**  
- Combine Transcript (Code)  
- Map Music (Set)  
- Code1 (Code)  
- Create Video (HTTP Request)  
- Wait2  
- Get Video Progress (HTTP Request)  
- Set Correct Video Url (Set)  

**Node Details:**  

- **Combine Transcript**  
  - Type: Code  
  - Role: Merges transcript text with image URLs and other metadata to prepare video content.  
  - Inputs: From "Image Link".  
  - Outputs: Connects to "Map Music".  
  - Edge Cases: Data mismatch or missing fields.

- **Map Music**  
  - Type: Set  
  - Role: Selects or assigns background music for the video.  
  - Inputs: From "Combine Transcript".  
  - Outputs: Connects to "Code1".  
  - Edge Cases: Missing or invalid music references.

- **Code1**  
  - Type: Code  
  - Role: Prepares final payload or parameters for video creation API.  
  - Inputs: From "Map Music".  
  - Outputs: Connects to "Create Video".  
  - Edge Cases: Code errors.

- **Create Video**  
  - Type: HTTP Request  
  - Role: Sends request to external video creation service to generate the video.  
  - Inputs: From "Code1".  
  - Outputs: Connects to "Wait2".  
  - Edge Cases: API errors, timeouts.

- **Wait2**  
  - Type: Wait  
  - Role: Pauses workflow to allow video processing.  
  - Inputs: From "Create Video".  
  - Outputs: Connects to "Get Video Progress".  
  - Edge Cases: Insufficient wait time.

- **Get Video Progress**  
  - Type: HTTP Request  
  - Role: Polls the video creation service to check processing status.  
  - Inputs: From "Wait2".  
  - Outputs: Connects to "Set Correct Video Url".  
  - Edge Cases: API errors, indefinite pending status.

- **Set Correct Video Url**  
  - Type: Set  
  - Role: Sets the final video URL once processing is complete.  
  - Inputs: From "Get Video Progress".  
  - Outputs: Connects to "Get Caption By Deepseek".  
  - Edge Cases: Missing or invalid URL.

---

#### 2.5 Metadata Generation and Upload

**Overview:**  
This block generates video captions and titles using Deepseek, uploads the video to YouTube, and updates a Google Sheet with video metadata for tracking.

**Nodes Involved:**  
- Get Caption By Deepseek (HTTP Request)  
- Get Title By Deepseek (HTTP Request)  
- Google Sheets (Write)  
- Grab Video (Google Sheets Read)  
- Download Video (HTTP Request)  
- YouTube (Upload)  
- Update Sheet (Google Sheets Write)  

**Node Details:**  

- **Get Caption By Deepseek**  
  - Type: HTTP Request  
  - Role: Retrieves a video caption based on the video content or transcript.  
  - Inputs: From "Set Correct Video Url".  
  - Outputs: Connects to "Get Title By Deepseek".  
  - Edge Cases: API failures.

- **Get Title By Deepseek**  
  - Type: HTTP Request  
  - Role: Retrieves a video title for YouTube upload.  
  - Inputs: From "Get Caption By Deepseek".  
  - Outputs: Connects to "Google Sheets".  
  - Edge Cases: API failures.

- **Google Sheets (Write)**  
  - Type: Google Sheets  
  - Role: Writes video metadata (title, description, URL) to a Google Sheet for record-keeping.  
  - Inputs: From "Get Title By Deepseek".  
  - Outputs: None (end of chain).  
  - Edge Cases: Sheet access permissions, quota limits.

- **Grab Video**  
  - Type: Google Sheets  
  - Role: Reads video data from Google Sheets to retrieve video URLs or metadata for processing.  
  - Inputs: From "Schedule Trigger".  
  - Outputs: Connects to "Download Video".  
  - Edge Cases: Empty or missing data.

- **Download Video**  
  - Type: HTTP Request  
  - Role: Downloads video file from storage or external source for upload.  
  - Inputs: From "Grab Video".  
  - Outputs: Connects to "YouTube".  
  - Edge Cases: Network errors.

- **YouTube**  
  - Type: YouTube Node  
  - Role: Uploads the final video to YouTube Shorts with metadata.  
  - Inputs: From "Download Video".  
  - Outputs: Connects to "Update Sheet".  
  - Edge Cases: Authentication errors, upload failures.

- **Update Sheet**  
  - Type: Google Sheets  
  - Role: Updates the Google Sheet with the YouTube video ID or status after upload.  
  - Inputs: From "YouTube".  
  - Outputs: None.  
  - Edge Cases: Sheet write errors.

---

### 3. Summary Table

| Node Name                 | Node Type              | Functional Role                              | Input Node(s)                  | Output Node(s)                  | Sticky Note                          |
|---------------------------|------------------------|----------------------------------------------|-------------------------------|--------------------------------|------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger         | Manual start trigger                         | None                          | Video Category                 |                                    |
| Video Category            | Set                    | Sets video category filter                    | When clicking ‘Test workflow’ | Get Transcript By Deepseek      |                                    |
| Get Transcript By Deepseek | HTTP Request           | Fetches article transcript from Deepseek API | Video Category                | Fetch Elevenlabs               |                                    |
| Fetch Elevenlabs          | HTTP Request           | Requests voiceover generation from ElevenLabs | Get Transcript By Deepseek    | Wait                          |                                    |
| Wait                     | Wait                   | Waits for audio generation completion         | Fetch Elevenlabs              | Save Audio                    |                                    |
| Save Audio               | Google Cloud Storage    | Uploads audio file to Google Cloud Storage    | Wait                         | Map Public Link               |                                    |
| Map Public Link          | Set                    | Sets public URL for stored audio              | Save Audio                   | Download Audio                |                                    |
| Download Audio           | HTTP Request           | Downloads audio file for transcription        | Map Public Link              | Open AI Whisper              |                                    |
| Open AI Whisper          | HTTP Request           | Transcribes audio to text using OpenAI Whisper | Download Audio               | Create a list of Image Text   |                                    |
| Create a list of Image Text | Code                   | Extracts image prompt texts from transcript   | Open AI Whisper              | Split Out                    |                                    |
| Split Out                | Split Out               | Splits image texts into individual prompts    | Create a list of Image Text   | Convert to Flux Prompt        |                                    |
| Convert to Flux Prompt   | HTTP Request           | Converts text to AI image generation prompts  | Split Out                   | Split Out - Image prompts     |                                    |
| Split Out - Image prompts | Split Out               | Splits image prompts for generation            | Convert to Flux Prompt        | Get image Base 64            |                                    |
| Get image Base 64        | HTTP Request           | Generates images and returns Base64 strings    | Split Out - Image prompts     | Convert to File              |                                    |
| Convert to File          | Convert To File         | Converts Base64 images to file objects          | Get image Base 64            | Wait1                       |                                    |
| Wait1                    | Wait                   | Waits before uploading images                    | Convert to File              | Save images to GC            |                                    |
| Save images to GC        | Google Cloud Storage    | Uploads images to Google Cloud Storage           | Wait1                       | Limit                       |                                    |
| Limit                    | Limit                  | Limits concurrency for image processing          | Save images to GC            | Image Link                  |                                    |
| Image Link               | Set                    | Sets public URLs for stored images                | Limit                       | Combine Transcript          |                                    |
| Combine Transcript       | Code                   | Combines transcript and image URLs for video     | Image Link                  | Map Music                  |                                    |
| Map Music                | Set                    | Assigns background music for video                | Combine Transcript           | Code1                      |                                    |
| Code1                    | Code                   | Prepares video creation payload                    | Map Music                   | Create Video               |                                    |
| Create Video             | HTTP Request           | Requests video creation from external service     | Code1                       | Wait2                      |                                    |
| Wait2                    | Wait                   | Waits for video processing completion              | Create Video                | Get Video Progress         |                                    |
| Get Video Progress       | HTTP Request           | Polls video creation status                          | Wait2                       | Set Correct Video Url      |                                    |
| Set Correct Video Url    | Set                    | Sets final video URL after processing                | Get Video Progress          | Get Caption By Deepseek    |                                    |
| Get Caption By Deepseek  | HTTP Request           | Retrieves video caption from Deepseek API            | Set Correct Video Url       | Get Title By Deepseek      |                                    |
| Get Title By Deepseek    | HTTP Request           | Retrieves video title from Deepseek API              | Get Caption By Deepseek     | Google Sheets              |                                    |
| Google Sheets            | Google Sheets           | Saves video metadata (title, description, URL)       | Get Title By Deepseek       | None                      |                                    |
| Schedule Trigger         | Schedule Trigger        | Scheduled start trigger for periodic runs            | None                       | Grab Video                 |                                    |
| Grab Video               | Google Sheets           | Reads video data for processing                        | Schedule Trigger            | Download Video             |                                    |
| Download Video           | HTTP Request           | Downloads video file for upload                         | Grab Video                  | YouTube                   |                                    |
| YouTube                  | YouTube Node            | Uploads video to YouTube Shorts                          | Download Video              | Update Sheet               |                                    |
| Update Sheet             | Google Sheets           | Updates Google Sheet with YouTube upload status          | YouTube                     | None                      |                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**  
   - Add a **Schedule Trigger** node to run the workflow periodically (e.g., every 24 hours).  
   - Add a **Manual Trigger** node named "When clicking ‘Test workflow’" for manual runs.

2. **Set Video Category:**  
   - Add a **Set** node named "Video Category" connected from the manual trigger.  
   - Configure to set the category (e.g., "Animal").

3. **Fetch Transcript:**  
   - Add an **HTTP Request** node named "Get Transcript By Deepseek".  
   - Configure with Deepseek API credentials and pass the category parameter.  
   - Connect from "Video Category".

4. **Request Voiceover Generation:**  
   - Add an **HTTP Request** node named "Fetch Elevenlabs".  
   - Configure with ElevenLabs API key, send transcript text.  
   - Connect from "Get Transcript By Deepseek".

5. **Wait for Audio Generation:**  
   - Add a **Wait** node named "Wait".  
   - Connect from "Fetch Elevenlabs".

6. **Save Audio to Cloud Storage:**  
   - Add a **Google Cloud Storage** node named "Save Audio".  
   - Configure with GCS credentials and target bucket.  
   - Connect from "Wait".

7. **Map Public Audio URL:**  
   - Add a **Set** node named "Map Public Link".  
   - Configure to construct public URL from GCS bucket and file path.  
   - Connect from "Save Audio".

8. **Download Audio File:**  
   - Add an **HTTP Request** node named "Download Audio".  
   - Configure to download audio from the public URL.  
   - Connect from "Map Public Link".

9. **Transcribe Audio:**  
   - Add an **HTTP Request** node named "Open AI Whisper".  
   - Configure with OpenAI API key to transcribe audio.  
   - Connect from "Download Audio".

10. **Extract Image Text Prompts:**  
    - Add a **Code** node named "Create a list of Image Text".  
    - Implement JavaScript to parse transcript and extract image prompts.  
    - Connect from "Open AI Whisper".

11. **Split Image Texts:**  
    - Add a **Split Out** node named "Split Out".  
    - Connect from "Create a list of Image Text".

12. **Convert to AI Image Prompts:**  
    - Add an **HTTP Request** node named "Convert to Flux Prompt".  
    - Configure to convert text prompts to AI image generation prompts.  
    - Connect from "Split Out".

13. **Split Image Prompts:**  
    - Add a **Split Out** node named "Split Out - Image prompts".  
    - Connect from "Convert to Flux Prompt".

14. **Generate Images (Base64):**  
    - Add an **HTTP Request** node named "Get image Base 64".  
    - Configure to call AI image generation API.  
    - Connect from "Split Out - Image prompts".

15. **Convert Base64 to File:**  
    - Add a **Convert To File** node named "Convert to File".  
    - Connect from "Get image Base 64".

16. **Wait Before Upload:**  
    - Add a **Wait** node named "Wait1".  
    - Connect from "Convert to File".

17. **Save Images to Cloud Storage:**  
    - Add a **Google Cloud Storage** node named "Save images to GC".  
    - Configure with GCS credentials and bucket.  
    - Connect from "Wait1".

18. **Limit Concurrency:**  
    - Add a **Limit** node named "Limit".  
    - Connect from "Save images to GC".

19. **Set Image Public URLs:**  
    - Add a **Set** node named "Image Link".  
    - Configure to create public URLs for images.  
    - Connect from "Limit".

20. **Combine Transcript and Images:**  
    - Add a **Code** node named "Combine Transcript".  
    - Implement logic to merge transcript and image URLs for video content.  
    - Connect from "Image Link".

21. **Map Background Music:**  
    - Add a **Set** node named "Map Music".  
    - Configure music selection parameters.  
    - Connect from "Combine Transcript".

22. **Prepare Video Creation Payload:**  
    - Add a **Code** node named "Code1".  
    - Prepare final payload for video creation API.  
    - Connect from "Map Music".

23. **Create Video:**  
    - Add an **HTTP Request** node named "Create Video".  
    - Configure with video creation service API.  
    - Connect from "Code1".

24. **Wait for Video Processing:**  
    - Add a **Wait** node named "Wait2".  
    - Connect from "Create Video".

25. **Poll Video Progress:**  
    - Add an **HTTP Request** node named "Get Video Progress".  
    - Configure to poll video creation status.  
    - Connect from "Wait2".

26. **Set Final Video URL:**  
    - Add a **Set** node named "Set Correct Video Url".  
    - Configure to store final video URL.  
    - Connect from "Get Video Progress".

27. **Get Video Caption:**  
    - Add an **HTTP Request** node named "Get Caption By Deepseek".  
    - Configure with Deepseek API to generate caption.  
    - Connect from "Set Correct Video Url".

28. **Get Video Title:**  
    - Add an **HTTP Request** node named "Get Title By Deepseek".  
    - Configure with Deepseek API to generate title.  
    - Connect from "Get Caption By Deepseek".

29. **Save Metadata to Google Sheets:**  
    - Add a **Google Sheets** node named "Google Sheets".  
    - Configure to write video metadata (title, caption, URL).  
    - Connect from "Get Title By Deepseek".

30. **Read Video Data for Upload:**  
    - Add a **Google Sheets** node named "Grab Video".  
    - Configure to read video data for upload.  
    - Connect from "Schedule Trigger".

31. **Download Video for Upload:**  
    - Add an **HTTP Request** node named "Download Video".  
    - Configure to download video file.  
    - Connect from "Grab Video".

32. **Upload Video to YouTube:**  
    - Add a **YouTube** node named "YouTube".  
    - Configure with YouTube OAuth2 credentials for upload.  
    - Connect from "Download Video".

33. **Update Google Sheet Post Upload:**  
    - Add a **Google Sheets** node named "Update Sheet".  
    - Configure to update sheet with upload status or video ID.  
    - Connect from "YouTube".

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                                                         |
|-----------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| Workflow automates YouTube Shorts creation from Google News trends using AI for transcript, images, and voiceover. | Workflow description and use case.                                                                     |
| Requires API keys for Google News (Deepseek), ElevenLabs TTS, OpenAI Whisper, Google Cloud Storage, and YouTube OAuth2. | Setup requirements.                                                                                    |
| Video creation uses external HTTP API services for AI image generation and video compilation.       | Integration details.                                                                                   |
| Google Sheets is used for metadata management and tracking video uploads.                            | Data organization and record-keeping.                                                                 |
| ElevenLabs TTS and OpenAI Whisper are used for audio generation and transcription verification.     | Audio processing details.                                                                              |
| YouTube node requires OAuth2 credentials with upload permissions.                                   | YouTube integration.                                                                                   |
| Workflow includes concurrency control and wait nodes to handle asynchronous API processing delays. | Reliability and timing considerations.                                                                |
| For more information on n8n nodes and API integrations, refer to official n8n documentation.        | https://docs.n8n.io/                                                                                   |

---

This document provides a comprehensive understanding of the "Automatic Youtube Shorts Generator" workflow, enabling reproduction, modification, and troubleshooting by advanced users and AI agents alike.